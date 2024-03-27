# Isolated synchronous deinit and asynchronous deinit

* Proposal: [SE-0371](0371-isolated-synchronous-deinit.md)
* Author: [Mykola Pokhylets](https://github.com/nickolas-pohilets)
* Review Manager: [Frederick Kellison-Linn](https://github.com/jumhyn)
* Status: **Returned for revision**
* Implementation: [apple/swift#60057](https://github.com/apple/swift/pull/60057)
* Review: ([pitch](https://forums.swift.org/t/isolated-synchronous-deinit/58177)) ([review](https://forums.swift.org/t/se-0371-isolated-synchronous-deinit/59754)) ([returned for revision](https://forums.swift.org/t/returned-for-revision-se-0371-isolated-synchronous-deinit/60060))

## Introduction

This feature allows `deinit`'s of actors and global-actor isolated types (GAITs) to access non-sendable isolated state, lifting restrictions imposed by [SE-0327](https://github.com/apple/swift-evolution/blob/main/proposals/0327-actor-initializers.md). This is achieved by providing runtime support for hopping onto executors in `__deallocating_deinit()`'s.

## Motivation

The combination of automatic reference counting and deterministic deinitialization makes `deinit` in Swift a powerful tool for resource management. It greatly reduces need for `close()`-like methods (`unsubscribe()`, `cancel()`, `shutdown()`, etc.) in the public API. Such methods not only clutter the public API, but also introduce a state where object is already unusable but is still able to be referenced.

Restrictions imposed by [SE-0327](https://github.com/apple/swift-evolution/blob/main/proposals/0327-actor-initializers.md) reduce the usefulness of explicit `deinit`s in actors and GAITs. Workarounds for these limitations may involve creation of `close()`-like methods, or even manual reference counting if the API should be able to serve several clients.

In cases when `deinit` belongs to a subclass of `UIView` or `UIViewController` which are known to call `dealloc` on the main thread, developers may be tempted to silence the diagnostic by adopting `@unchecked Sendable` in types that are not actually  sendable. This undermines concurrency checking by the compiler, and may lead to data races when using incorrectly marked types in other places.

If deinit is used to start asynchronous shutdown work, developers need be careful to explicitly capture all the required stored properties. Failure to do so, may lead to escaping`self` and creation of dangling reference.

## Proposed solution

Allow execution of `deinit` and object deallocation to be the scheduled on the executor of the containing type (either that of the actor itself or that of the relevant global actor), if needed.

Let's consider [examples from SE-0327](https://github.com/apple/swift-evolution/blob/main/proposals/0327-actor-initializers.md#data-races-in-deinitializers):

In the case of several instances with shared data isolated on a common actor, the problem is completely eliminated:

```swift
class NonSendableAhmed { 
  var state: Int = 0
}

@MainActor
class Maria {
  let friend: NonSendableAhmed

  init() {
    self.friend = NonSendableAhmed()
  }

  init(sharingFriendOf otherMaria: Maria) {
    // While the friend is non-Sendable, this initializer and
    // and the otherMaria are isolated to the MainActor. That is,
    // they share the same executor. So, it's OK for the non-Sendable value
    // to cross between otherMaria and self.
    self.friend = otherMaria.friend
  }

  isolated deinit {
    // Used to be a potential data race. Now, deinit is also
    // isolated on the MainActor, so this code is perfectly 
    // correct.
    friend.state += 1
  }
}

func example() async {
  let m1 = await Maria()
  let m2 = await Maria(sharingFriendOf: m1)
  doSomething(m1, m2)
} 
```

In the case of escaping `self`, the race condition is eliminated but the problem of dangling reference remains.

```swift
actor Clicker {
  var count: Int = 0

  func click(_ times: Int) async {
    for _ in 0..<times {
      self.count += 1 
    }
  }

  isolated deinit {
    let old = count
    let moreClicks = 10000
    
    Task { await self.click(moreClicks) } // ❌ This WILL keep `self` alive after the `deinit`!

    for _ in 0..<moreClicks {
        // No data race.
        // Actor job created by the task is either
        // not created yet or is waiting in the queue.
      self.count += 1 
    }

    assert(count == old + moreClicks) // Always works
    assert(count == old + 2 * moreClicks) // Always fails
  }
}
```

This can be improved further by using async `deinit`:

```swift
actor Clicker {
  ...

  deinit async {
    let old = count
    let moreClicks = 10000
    
    // All stored properties remain valid for the duration of this async call
    // Deallocation will happen only after body of the async deinit completes
    await self.click(moreClicks)

    for _ in 0..<moreClicks {
      self.count += 1 
    }

    assert(count == old + 2 * moreClicks)
  }
}
```

Asynchronous `deinit` always executes concurrently with the code that triggered last release.

If it is desired to perform async cleanup sequentially, recommended approach is to use a regular named method in combination with a runtime check:

```swift
class Connection {
  private var isClosed = false

  public consuming func close() async {
    ...
    isClosed = true
  }

  deinit {
    assert(isClosed, "Connection was not closed")
  }
}

func communicate() {
    let c = Connection(...)
    ...
    await c.close()
}
```

If Swift gets support for linear types in the future, it will be possible to replace runtime checks with compile-time checks.

## Detailed design

### Runtime

This proposal introduces two new runtime functions:

```swift
// Flags:
// * copyTaskLocalsOnHop    = 0x1
// * resetTaskLocalsOnNoHop = 0x2

@_silgen_name("swift_task_deinitOnExecutor")
@usableFromInline
internal func _deinitOnExecutor(_ object: __owned AnyObject,
                                _ work: @convention(thin) (__owned AnyObject) -> Void,
                                _ executor: UnownedSerialExecutor,
                                _ flags: Builtin.Word)

@_silgen_name("swift_task_deinitAsync")
@usableFromInline
internal func _deinitAsync(_ object: __owned AnyObject,
                           _ work: @convention(thin) (__owned AnyObject) async -> Void,
                           _ executor: Builtin.Executor?,
                           _ flags: Builtin.Word)
```

`swift_task_deinitOnExecutor` provides support for isolated synchronous `deinit`. If ensures that `deinit`'s code is running on the correct executor, by wrapping it into a task-less ad hoc job, if needed. It does not do any reference counting and can be safely used even with references that were released for the last time but not deallocated yet.

If no switching is needed, then `deinit` is executed immediately on the current thread. Otherwise, the task-less job copies priority and task-local values from the task/thread that released the last strong reference to the object.

Function takes flags that control how task-locals are copied:
* `copyTaskLocalsOnHop` - if task-locals should be copied in `O(n)` if job needs to be created.
* `resetTaskLocalsOnNoHop` - if task-locals should be blocked by a barrier in `O(1)` when deinit is executed immediately.

Passing no flags performs least work, but creates inconsistency between cases of calling deinit immediately and creating a job. Passing only `copyTaskLocalsOnHop` ensures that task-locals are consistently available in both cases. Passing only `resetTaskLocalsOnNoHop` ensures that task-locals are consistently reset in both cases. Passing both flags technically is possible, but creates inverted inconsistent behavior with runtime cost in all cases. Value of the `flags` parameter is controlled by [attributes](#task-local-values) applied to the deinit declaration.

`swift_task_deinitAsync` provides support for asynchronous `deinit` regardless of it's isolation. It always creates a task, regardless of current executor and sync/async context. Asynchronous deinit is always executed concurrently with the code that triggered last release. Code that triggered last release is neither blocked nor awaits for deinit completion.

Created task copies priority and task-local values from the task/thread that released the last strong reference to the object. The same flags are accepted, but since there is no "no-hop" case for async deinit, `resetTaskLocalsOnNoHop` is ignored.

As an optimization, task created by the `swift_task_deinitAsync` immediately starts execution on the correct executor.

Note that `object` and `work` are imported in Swift as two separate arguments, and not a closure for two reasons:

1. Normally task is responsible for releasing closure context after completion, but `work` consumes it's argument, so no extra release is needed.
2. `thin` function taking a single explicit argument has a different ABI from closure function taking closure context

If `deinit` is isolated or asynchronous, code that normally is emitted into `__deallocating_init` gets emitted into a new entity (`__isolated_deallocating_init`), and `__deallocating_init` is emitted as a thunk that reads the executor (from `self` for actors and from the global actor for GAITs) and calls corresponding runtime function passing `self`, `__isolated_deallocating_init` and the desired executor.

This proposal enables async destroying `deinit`, but otherwise destroying `deinit` is not affected.

### Rules for computing isolation

#### Synchronous isolated `deinit`

For backwards compatibility, isolation of the class is not propagated to the synchronous `deinit` by default:

```swift
@MainActor
class Foo {
  deinit {} // not isolated
}
```

To opt-in into isolation propagation, proposal allows `isolated` attribute to be applied to `deinit` declaration. This is an existing attribute, which currently can be applied only to function arguments, but has a different meaning there. When applied to a function argument it indicates a parameter which contains an instance of the actor which function should be isolated to.

```swift
@MainActor
class Foo {
  isolated deinit {} // Isolated on MainActor.shared
}

actor Bar {
  isolated deinit {} // Isolated on self
}
```

It is an error to use `isolated` attribute on a `deinit`, if containing class has no isolation.

```swift
class Foo {
  // error: deinit is marked isolated, but containing class Foo is not isolated to an actor
  isolated deinit {}
}
```

If containing class is not isolated, it is still possible to use global actor attribute on `deinit`.

```swift
class Foo {
  @MainActor deinit {}
}
```

It is also possible to use explicit global actor attribute, to override isolation of the containing class. Using global actor attribute on deinit to specify the same isolation as in the containing class, may be seen as a violation of DRY, but technically is valid and does not produce any warnings.

```swift
@MainActor
class Foo {
  // Allowed, but 'isolated' is still the recommended approach
  @MainActor deinit {}
}

@MainActor
class Bar {
  // Exotic, but will work
  @AnotherActor deinit {}
}

actor Baz {
  // Also possible
  @MainActor deinit {}
}
```

For consistency, `nonisolated` attribute also can be used, but for synchronous `deinit` it has no effect, because synchronous deinitializers are nonisolated by default.

```swift
@MainActor
class Foo {
  // Same as no attributes
  nonisolated deinit {}
}
```

Once isolation of the `deinit` is computed, it is then validated to be compatible with isolation of the `deinit` of the base class. Classes can add isolation to the non-isolated `deinit` of the base class, but they cannot change (remove or change actor) existing isolation. If base class has isolated `deinit`, all derived classes must have `deinit` with the same isolation.

Synthesized deinit inherits isolation of the superclass `deinit` automatically, but explicit `deinit` needs to marked with `isolated` or global actor attribute.

```swift
@MainActor
class Base {
  isolated deinit {}
}

class Derived: Base {
  // ok, isolation matches
  isolated deinit {}
}

class Removed: Base {
  // error: nonisolated deinitializer 'deinit' has different actor isolation from global actor 'MainActor'-isolated overridden declaration
  deinit {}
}

class Changed: Base {
  // error: global actor 'AnotherActor'-isolated deinitializer 'deinit' has different actor isolation from global actor 'MainActor'-isolated overridden declaration
  @AnotherActor deinit {}
}

class Implicit: Base {
  // ok, implicit deinit inherits isolation automatically
}
```

Note that type-checking of overridden `deinit` is inverted compared to regular functions. When type-checking regular function, compiler analyzes if overriding function can be called through the vtable slot of the overridden one. When type-checking `deinit`, compiler analyzes if `super.deinit()` can be called from the body of the `deinit`.

```swift
class Base {
  // Will be called only on MainActor
  @MainActor func foo() {}

  // Can be called from any executor
  nonisolated deinit {}
}

class Derived: Base {
  // Can be called from any executor, including MainActor
  nonisolated override func foo() {}

  @MainActor deinit {
    // Can we call super.deinit()?
  }
}

let x: Base = Derived()
x.foo() // Can we call Derived.foo()?
```

Types that don't perform custom actions in `deinit` and only need to release references don't need isolated or async `deinit`. Releasing child objects can be done from any thread. If those objects are concerned about isolation, they should adopt isolation themselves. Implicit deinitializers cannot opt-in into isolation, so they are synchronous and nonisolated by default.

```swift
class Foo {
  var bar: Bar

  // implicit deinit is synchronous and nonisolated
  // release of Bar can happen on any thread/task
  // Bar is responsible to its own isolation.
}

class Bar {
  @MainActor deinit {}
}
```

#### Asynchronous `deinit`

Asynchronous `deinit` comes with higher costs compared to isolated synchronous deinit. For asynchronous `deinit` new task is created every time. There is no fast path that would execute deinit code immediately. And task has higher memory and performance cost compared to a single ad-hoc job.

Because of that, when isolated synchronous `deinit` is sufficient, usage of asynchronous `deinit` is discouraged by a warning. Fix-it suggests to remove `async`, and insert `isolated` if there are no isolation-related attributes.

```swift
class Foo {
  // warning: async deinit contains no await statements; consider using isolated sync deinit instead
  deinit async {}
}
```

Asynchronous `deinit` can be nonisolated or have isolation which does not match with isolation of the stored properties. In this case it is possible to access properties of sendable type by `await`-ing. Properties of the non-sendable types cannot be accessed directly, but can be used inside isolated operations, which in turn can be `await`ed.

```swift
class NonSendable {
    var state: Int = 0
}

@MainActor
class WithNonSendable {
    var sendable: Int = 0
    var nonSendable: NonSendable = NonSendable()
    
    func getNonSendable() -> Int { nonSendable.state }
    
    nonisolated deinit async {
        // ok
        print(await self.sendable)
        // error: non-sendable type 'NonSendable' in implicitly asynchronous access to global actor 'FirstActor'-isolated property 'nonSendable' cannot cross actor boundary
        print(await self.nonSendable.state)
        // ok
        print(await getNonSendable())
    }
}
```

Using asynchronous `deinit` like this does solve the issue of data races, but this solution is sub-optimal. Generated code has a lot of executor hops, which could be avoided by isolating entire `deinit` on the correct executor. Also this masks the fact that `async` is not needed in this case. Developer will not get a suggestion to use isolated synchronous deinit instead.

To help guide developers to correct usage, isolation of the class is propagated to asynchronous deinit by default, similar to regular methods.

```swift
class NonSendable {
    var state: Int = 0
}

@MainActor
class WithNonSendable {
    var sendable: Int = 0
    var nonSendable: NonSendable = NonSendable()
    
    func getNonSendable() -> Int { nonSendable.state }
    
    // Isolated on MainActor
    // warning: async deinit contains no await statements; consider using isolated sync deinit instead
    deinit async {
        // ok
        print(self.sendable)
        // ok
        print(self.nonSendable.state)
        // ok
        print(getNonSendable())
    }
}

actor MyActor {
   // Isolated on self
  deinit async {
    ...
  }
}
```

Asynchronous deinit can be marked `nonisolated` to opt-out from class isolation.

```swift
@MainActor
class Foo {
  // Executes on generic executor
  nonisolated deinit async {
    ...
  }
}

actor MyActor {
  // Executes on generic executor
  nonisolated deinit async {
    ...
  }
}
```

For consistency, explicit `isolated` attribute is also allowed on asynchronous `deinit`. If class has no isolation, it will produce an error, and otherwise will have no effect.

```swift
@MainActor
class Foo {
  // ok
  isolated deinit async {
    ...
  }
}

class Bar {
  // error: deinit is marked isolated, but containing class Bar is not isolated to an actor
  isolated deinit async {
    ...
  }
}
```

If base class has asynchronous `deinit`, deinitializer in the derived class must be async too. Implicit `deinit` is synthesized as `async` automatically, if base class has async `deinit`. But explicit `deinit` must be marked as `async` explicitly.

```swift
class Base {
  deinit async {
    ...
  }
}

class Derived: Base {
  // error: deinit must be 'async' because parent class has 'async' deinit}
  deinit {
    ...
  }
}

class Implicit: Base {
  // ok, implicit deinit is automatically synthesized as 'async'
}
```

If base class has synchronous `deinit` (isolated or not), derived class may have async `deinit`.

```swift
@MainActor
class Base {
  isolated deinit {
    ...
  }
}

class Derived: Base {
  deinit async {
    ...
    // calls synchronous super.deinit in the end
  }
}
```

Async deinit can have isolation different from the isolation of the `super.deinit`. If `super.deinit` is asynchronous, then it is responsible for switching to the correct executor (including generic executor).

If `super.deinit` is synchronous and isolated, then async `deinit` in the derived class will hop to the correct executor before calling `super.deinit`. If `super.deinit` is synchronous and not isolated, then it will be called on whatever executor `deinit` of the derived class executes. This is similar to how calling synchronous functions from async functions works.

```swift
@MainActor
class FooBase {
  isolated deinit {}
}

class FooDerived: FooBase {
  nonisolated deinit async {
    // Executes on generic executor
    ...
    // Hops to MainActor before calling super.deinit
  }
}

class BarBase {
  deinit {}
}

class BarDerived: BarBase {
  @MainActor deinit async {
    // Executes on MainActor
    ...
    // Calls super.deinit while being on MainActor
    // Does not switch to generic executor
  }
}

class BazBase {
  @MainActor deinit async {
    // Can be called on any executor
    // Will hop to MainActor
  }
}

class BazDerived: BazBase {
  @AnotherActor deinit async {
    // Executes on AnotherActor
    // Calls super.deinit on AnotherActor
    // super.deinit then hops to MainActor
  }
}
```

### Importing Objective-C code

Objective-C compiler does not generate any code to make `dealloc` isolated and marking Objective-C classes as isolated on global actor using `__attribute__((swift_attr(..)))` has no effect on behavior of the ObjC code. Such classes are imported into Swift as having non-isolated `deinit`.

However if `__attribute__((swift_attr(..)))` is used in the class' `@interface` to explicitly mark the `dealloc` method as isolated on a global actor, then it is imported as an isolated `deinit`. Marking `dealloc` as isolated means that `dealloc` must be called only on that executor. It is assumed that the Objective-C implementation of such class ensures this by overriding `retain`/`release`. The `deinit` of Swift subclasses of the Objective-C class is generated as an override of the `dealloc` method. Any pre-conditions which hold for the base class will be true for Swift subclasses as well. In this case `deinit` of the Swift subclass is type-checked as isolated, but an isolation thunk is not generated for code size and performance optimization.

If `deinit` isolation was introduced into the hierarchy of the `@objc` Swift classes by a class implemented in Swift, then `retain`/`release` are not overridden and `dealloc` can be called from any thread, but isolation happens inside `dealloc` implementation. In this case, isolation thunks will be generated for each isolated `deinit` in the hierarchy. Only the `deinit` of the subclass furthest down the hierarchy does the actual switching. The rest will be called when already on the correct executor, and will follow the fast path in `swift_task_performOnExecutor()`.

Objective-C classes that isolate `dealloc` by overriding `retain/release` should mark `dealloc` as isolated. This not only allows Swift subclasses to fully benefit from isolation, but also prevents them from isolating their `deinit/dealloc` (including the call to `[super dealloc]`) on a different actor.

On the other hand, if Objective-C classes implement isolation by switching executors inside `dealloc`, they should not mark `dealloc` as isolated. Such `dealloc` can be called from any thread, and does not prevent Swift subclasses from isolating on different actor. And skipping isolation thunk in the Swift subclasses would be incorrect.

```objc
// Non-ARC code

// Executes on main queue/thread
- (void)dealloc_impl {
    ...
    [super dealloc];
}

static void dealloc_impl_helper(void *ctx) {
    [(MyClass*)ctx dealloc_impl];
}

// SHOULD NOT be marked as isolated!
// Executes on any thread
- (void)dealloc {
    dispatch_async_f(dispatch_get_main_queue(), self, dealloc_impl_helper);
}
```

There is no support for importing `dealloc` methods as asynchronous deinit. 

### Exporting to Objective-C

`deinit` isolation and asynchrony is relevant only when subclassing. Since Objective-C code cannot subclass Swift classes, the generated  `*-Swift.h` files contain no addition information about new `deinit` features.


### Interaction with ObjC runtime.

All Objective-C-compatible Swift classes have `dealloc` method synthesized, which acts as a thunk to `__deallocating_deinit`. Normally when calling `super.deinit` from `__deallocating_deinit`, it is done by sending Objective-C `dealloc` message using `objc_msgSuper`.

This ensures maximum possible compatibility with Objective-C, but comes with some runtime cost.

For isolated synchronous `deinit`, this cost increases, but not much. For each class in the hierarchy `swift_task_deinitOnExecutor()` will be called, but slow math can be taken only for the most derived class.

But for asynchronous `deinit`, cost of calling `[super dealloc]` becomes unacceptable. Every call to `[super dealloc]` would be creating new task.

To avoid this, async deinit bypasses Objective-C runtime when calling isolated or async `deinit` of Swift base classes, and calls `__isolated_deallocating_deinit` directly. Compatibility with Objective-C is preserved only on Swift<->Objective-C boundary in either direction.

This is still sufficient to create ObjC subclasses in runtime, as KVO does. Or swizzle `dealloc` of the most derived class of an instance. But swizzling `dealloc` of intermediate Swift classes might not work as expected.

Since asynchronous `deinit` introduces a precedent for loosening Objective-C compatibility, isolated synchronous `deinit` also makes use of it, bypassing Objective-C runtime and executor checks for performance.

### Isolated synchronous deinit of default actors

When deinitializing an instance of default actor, `swift_task_deinitOnExecutor()` attempts to take actor's lock and execute deinit on the current thread. If previous executor was another default actor, it remains locked. So potentially multiple actors can be locked at the same time. This does not lead to deadlocks, because (1) lock is acquired conditionally, without waiting; and (2) object cannot be deinitializer twice, so graph of the deinit calls has no cycles.

### Interaction with distributed actors

`deinit` declared in the code of the distributed actor applies only to the local actor and can be isolated or marked async as described above. Remote proxy has an implicit compiler-generated synchronous `deinit` which is never isolated.

### Interaction with task executor preference

Ad-hoc job created for isolated synchronous deinit is executed outside a task, so task executor preference does not apply.

Task created for async deinit is unstructed, and thus does not inherit task executor preference.

### Task-local values

Copying task-local values is a safe default choice for handling task-local values in isolated or asynchronous `deinit`. It allows code in the deinit to access task-local values at the point of last release, the same way they are available in synchronous nonisolated `deinit`.

It can be hard to predict where the point of the last release will be, but as long it can be constrained to some broad scope, code in the `deinit` can assume that all task-locals set for the entire scope are available.

It might be more reliable to instead read task-locals in the initializer, and store them as properties of the object. But it might be not possible if task-locals are tunneled through call stack without being directly accessible.

```swift
// MyLib
private class Logger {
  @TaskLocal
  static var instance: Logger?
}

public func withLogger(operation: () async -> Void) {
  let logger = Logger()
  Logger.$instance.withValue(logger, operation: operation)
}

public func log(_ message: String) {
  Logger.instance?.log(message)
}

// MyApp
import MyLib

class MyService {
  init() {
    // Cannot capture Logger.instance because it private
  }

  func getData() async -> Data { ... }
  func doStuff() async { ... }

  deinit async {
    do {
      try await shutdown()
    } catch {
      log("\(error)")
    }
  }
}

withLogger {
  let service = MyService()
  async let data = service.getData()
  await service.doStuff()
  // We don't know if last release happens in async let or in the main task
  // But we know that it happens inside the closure
  // So we can assume that all task locals set by the withLogger are available,
  // even if don't know them
}
```

But copying task-locals comes with a cost. For some performance-critical cases this cost might be unacceptable.

It is possible to disable this behavior using new attribute - `@resetTaskLocals`. It affects which flags will be passed to runtime functions.

By default `copyTaskLocalsOnHop` is passed, making task-locals from the point of last release available in the deinit on all code paths.

If `@resetTaskLocals` is specified, `resetTaskLocalsOnNoHop` is passed, making task-locals consistently reset in the deinit.

Inserting a barrier for fast path of the `swift_task_deinitOnExecutor()` has small runtime cost, but consistent behavior simplifies writing unit-tests and debugging, and smoothens learning curve.

Currently there are no attributes that would emit combination of flags that would neither insert a barrier nor copy task-local values. But they can be added in the future without changes in the language runtime.

Attribute is not inherited. Attribute needs to be applied to the most derived class to have effect:

```swift
private class TL {
  @TaskLocal
  static var value: Int = 0
}

class A {
  deinit async {
    print("A: \(TL.value)")
  }
}
class B: A {
  @resetTaskLocals
  deinit async {
    print("B: \(TL.value)")
  }
}
class C: B {
  deinit async {
    print("C: \(TL.value)")
  }
}

TL.$value.withValue(42) {
  // prints A: 42
  _ = A()
  // prints B: 0, A: 0
  _ = B()
  // prints C: 42, B: 42, A: 42
  _ = C()
}
```

## Source compatibility

This proposal makes previously invalid code valid.

## Effect on ABI stability

This proposal does not change the ABI of existing language features, but does introduce new runtime functions.

## Effect on API resilience

Isolation attributes and asynchrony of the `deinit` become part of the public API, but they matter only when inheriting from the class.

Any changes to the isolation or asynchrony of `deinit` of non-open classes are allowed.

For open classes, it is allowed to make any changes to the isolation of the asynchronous `deinit`.

For open non-@objc classes, it is allowed to change synchronous `deinit` from isolated to nonisolated. Any non-recompiled subclasses will keep calling `deinit` of the superclass on the original actor. Changing `deinit` from nonisolated to isolated or for changing identity of the isolating actor is a breaking change.

For open @objc classes, any change in isolation of the synchronous `deinit` is a breaking change, even changing from isolated to nonisolated. This removes symbol for `__isolated_deallocating_deinit` and clients will fail to link with new framework version. See also [Interaction with ObjC runtime](#interaction-with-objc-runtime).

<table>
  <tr>
    <td rowspan="2">Change</td>
    <td>|</td>
    <td colspan="2">open</td>
    <td rowspan="2">non-open</td>
  </tr>
  <tr>
  <td>|</td>
  <td>
  @objc
  </td>
  <td>
  non-@objc
  </td>
  </tr>
  <tr>
    <td>async <-> sync</td><td>|</td><td>breaking</td><td>breaking</td><td>ok</td>
  </tr>
  <tr>
    <td>sync: remove isolation</td><td>|</td><td>breaking</td><td>ok</td><td>ok</td>
  </tr>
  <tr>
    <td>sync: add isolation</td><td>|</td><td>breaking</td><td>breaking</td><td>ok</td>
  </tr>
  <tr>
    <td>sync: change actor</td><td>|</td><td>breaking</td><td>breaking</td><td>ok</td>
  </tr>
  <tr>
    <td>async: remove isolation</td><td>|</td><td>ok</td><td>ok</td><td>ok</td>
  </tr>
  <tr>
    <td>async: add isolation</td><td>|</td><td>ok</td><td>ok</td><td>ok</td>
  </tr>
  <tr>
    <td>async: change actor</td><td>|</td><td>ok</td><td>ok</td><td>ok</td>
  </tr>
</table>

Adding an isolation annotation to a `dealloc` method of an imported Objective-C class is a breaking change. Existing Swift subclasses may have `deinit` isolated on different actor, and without recompilation will be calling `[super deinit]` on that actor. When recompiled, subclasses isolating on a different actor will produce a compilation error. Subclasses that had a non-isolated `deinit` (or a `deinit` isolated on the same actor) remain ABI compatible. It is possible to add isolation annotation to UIKit classes now, because currently all their subclasses have nonisolated `deinit`s.

Removing an isolation annotation from the `dealloc` method (together with `retain`/`release` overrides) is a breaking change. Any existing subclasses would be type-checked as isolated but compiled without isolation thunks. After changes in the base class, subclass `deinit`s could be called on the wrong executor.

If isolated and async deinit need to be suppressed in `.swiftinterface` for compatibility with older compilers, then `open` classes are emitted as `public` to prevent subclassing.

## Future Directions

### Linear types

Invoking sequential async cleanup is a suspension point, and needs to be marked with `await`. Explicit method calls fit this role better than implicitly invoked `deinit`. But using such methods can be error-prone without compiler checks that cleanup method is called exactly once on all code paths. Move-only types help to ensure that cleanup method is called **at most once**. Linear types help to ensure that cleanup method is called **exactly once**.

```swift
@linear // like @moveonly, but consumption is mandatory
struct Connection {
  // acts as a named explicit async deinit
  consuming func close() async {
    ...
  }
}

func communicate() async {
  let c = Connection(...)
  // error: value of linear type is not consumed
}
```

### Improving de-virtualization and inlining of the executor access.

Consider the following example:

```swift
import Foundation

@_silgen_name("do_it")
@MainActor func doIt()

public class Foo {
  @MainActor
  public func foo() async {
    doIt()
  }
  @MainActor
  deinit {}
}
```

Currently both the `foo()` and `deinit` entry points produce two calls to access the `MainActor.shared.unownedExecutor`, with the second one even using dynamic dispatch. These two calls could be replaced with a single call to the statically referenced `swift_task_getMainExecutor()`.

```llvm
%1 = tail call swiftcc %swift.metadata_response @"type metadata accessor for Swift.MainActor"(i64 0) #6
%2 = extractvalue %swift.metadata_response %1, 0
%3 = tail call swiftcc %TScM* @"static Swift.MainActor.shared.getter : Swift.MainActor"(%swift.type* swiftself %2)
%4 = tail call i8** @"lazy protocol witness table accessor for type Swift.MainActor and conformance Swift.MainActor : Swift.Actor in Swift"() #6
%5 = bitcast %TScM* %3 to %objc_object*
%6 = tail call swiftcc { i64, i64 } @"dispatch thunk of Swift.Actor.unownedExecutor.getter : Swift.UnownedSerialExecutor"(%objc_object* swiftself %5, %swift.type* %2, i8** %4)
```

### Improving extended stack trace support

Developers who put breakpoints in the isolated or async deinit might want to see the call stack that lead to the last release of the object. Currently, if switching of executors was involved, the release call stack won't be shown in the debugger.

### Implementing API for synchronously scheduling arbitrary work on the actor

`swift_task_deinitOnExecutor()` has calling convention optimized for the `deinit` use case, but using a similar runtime function with a slightly different signature, one could implement an API for synchronously scheduling arbitrary work on the actor:

```swift
extension Actor {
  /// Adds a job to the actor queue that calls `work` passing `self` as an argument.
  nonisolated func enqueue(_ work: __owned @Sendable @escaping (isolated Self) -> Void)

  /// If actor's executor is already the current one - executes work immediately
  /// Otherwise adds a job to the actor's queue.
  nonisolated func executeOrEnqueue(_ work: __owned @Sendable @escaping (isolated Self) -> Void)
}

actor MyActor {
    var k: Int = 0
    func inc() { k += 1 }
}

let a = MyActor()
a.enqueue { aIsolated in
    aIsolated.inc() // no await
}
```

## Alternatives considered

### Placing hopping logic in `swift_release()` instead.

`UIView` and `UIViewController` implement hopping to the main thread by overriding the `release` method. But in Swift there are no vtable/wvtable slots for releasing, and adding them would also affect a lot of code that does not need isolated deinit.

### Don't copy task-local values when hopping by default

When switching executors, the current implementation copies priority and task-local values from the task/thread where the last release happened. This minimizes differences after code changes between async/isolated and nonisolated synchronous `deinit`s, and runtime differences between fast and slow paths of isolated deinit. Copying task-locals by default and providing an option to opt-out, follows the precedent of `Task.init` vs `Task.detached` and adheres to the principle of progressive disclosure.

There are no compile-time checks for availability of the non-default task-local values.
Checking correctness of the code that uses task-local values relies on testing.
Having different behavior depending on the point of the last release can lead to bugs,
which are easy to miss during the testing and hard to reproduce once reported.

One can argue that since point of the last release can be racy or simply hard to predict exactly, task-local values have little value in deinit. But even when point of the last release cannot be known exactly, often it can be bound to a scope. If task-local values are set for the entire scope, developers can expect them to be reliably available in the `deinit`.

In some cases it might be a better practice to capture task-local values in the `init()`, store them as object properties and re-inject them into `deinit`. But it is not always possible to know which task-local values are used inside the `deinit`. For example, if it calls functions from closed-source frameworks or hidden behind type-erasure.

It can be easy to miss during testing that task-local values are not injected into code running inside `deinit` if task-local values are accessed only in exceptional situations -  e.g. for logging assertion failures or runtime errors. But those are exactly the use cases where it is critically important for logging to work.

Consistently resetting task-locals is another alternative which gives consistent behavior. But it comes with it's own runtime cost, which needs to be paid on the fast-path of the isolated deinit per object. Assuming that trees of objects are likely to be isolated to the same actor. Cost of copying task-local values needs to be paid only when deinitiazing root object and is proportional to the number of task-locals. While cost of resetting needs to be paid per object and is proportional to the tree size. Assuming that number of task-locals is smaller than number of objects for a typical use case, copying task-local values is actually faster then resetting them.

Note that copying task-local values does not prevent future optimizations for reusing current task for async deinit of the child objects.
Release of the child objects happens from complier-generated code which does interfere with task-local values.
And any task-local values added inside the body of the deinit, would be removed inside the body as well.
So if last release of the child object starts a new async deinit, it would need to be executed with the same set of task-local values as parent deinit,
and thus can be executed in the same task.

### Implicitly propagate isolation to synchronous `deinit`.

This would be a source-breaking change.

Majority of the `deinit`'s are implicitly synthesized by the compiler and only release stored properties. Global open source search in Sourcegraph, gives is 77.5k deinit declarations for 2.2m classes - 3.5%. Release can happen from any executor/thread and does not need isolation. Isolating implicit `deinit`s would come with a major performance cost. Providing special rules for propagating isolation to synchronous `deinit` unless it is implicit, would complicate propagation rules.

### Use asynchronous deinit as the only tool for `deinit` isolation

Synchronous `deinit` has an efficient fast path that jumps right into the `deinit` implementation without context switching, or any memory allocations. Fast path is expected to be taken in majority of cases. And even slow path of isolated synchronous deinit is faster and has smaller memory footprint that asynchronous `deinit`.

### Execute asynchronous `deinit` sequentially if last release happens from async code

That would mean that behavior of asynchronous deinit changes when synchronous function is inlined into asynchronous caller, or a part of asynchronous function is extracted into a synchronous helper. This can happen due to optimizations, monomorphing generic code or manual refactoring. In either case, the change in behavior is likely to be unexpected to the developer. Changes caused by optimizations can lead to issues which reproduce only in release build.

Sequential execution of asynchronous deinit with the last release from sync code could be approximated
by adding such objects to the task-owned queue, which would be drained at the end of async scope. Calling runtime function at the end of each async scope can be too high performance cost.

It could be possible to implement this using explicit API:

```swift
func test() async {
  // Installs pool as a task-local value
  await withAsyncDeinitPool {
    let c = Foo()
    ...
    // If pool is installed, swift_task_deinitAsync adds object to the pool
    // instead of creating new task
  }
}
```
