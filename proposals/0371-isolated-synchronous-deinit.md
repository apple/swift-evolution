# Isolated synchronous deinit

* Proposal: [SE-0371](0371-isolated-synchronous-deinit.md)
* Author: [Mykola Pokhylets](https://github.com/nickolas-pohilets)
* Review Manager: [Frederick Kellison-Linn](https://github.com/jumhyn)
* Status: **Returned for revision**
* Implementation: [apple/swift#60057](https://github.com/apple/swift/pull/60057)
* Review: ([pitch](https://forums.swift.org/t/isolated-synchronous-deinit/58177)) ([review](https://forums.swift.org/t/se-0371-isolated-synchronous-deinit/59754)) ([returned for revision](https://forums.swift.org/t/returned-for-revision-se-0371-isolated-synchronous-deinit/60060))

## Introduction

This feature allows `deinit`'s of actors and global-actor isolated types (GAITs) to access non-sendable isolated state, lifting restrictions imposed by [SE-0327](https://github.com/apple/swift-evolution/blob/main/proposals/0327-actor-initializers.md). This is achieved by providing runtime support for hopping onto executors in `__deallocating_deinit()`'s.

## Motivation

Restrictions imposed by [SE-0327](https://github.com/apple/swift-evolution/blob/main/proposals/0327-actor-initializers.md) reduce the usefulness of explicit `deinit`s in actors and GAITs. Workarounds for these limitations may involve creation of `close()`-like methods, or even manual reference counting if the API should be able to serve several clients.

In cases when `deinit` belongs to a subclass of `UIView` or `UIViewController` which are known to call `dealloc` on the main thread, developers may be tempted to silence the diagnostic by adopting `@unchecked Sendable` in types that are not actually  sendable. This undermines concurrency checking by the compiler, and may lead to data races when using incorrectly marked types in other places.

While this proposal introduces additional control over deinit execution semantics that are necessary in some situations, it also introduces a certain amount of non-determinism to deinitializer execution. Because isolated deinits must potentially be enqueued and executed "later" rather than directly inline, this can cause subtle timing issues in resource reclamation. For example, APIs which require quick and predictable resource cleanup, such as scarce resources such as e.g. file descriptors or connections, should not be managed using isolated deinitializers, as the exact timing of when the resource would be released is non-deterministic, which can lead to subtle timing and resource starvation issues. Instead, for types which require tight control over lifetime and cleanup, one should still prefer using "with-style" APIs (`await withResource { resource }`), explicit `await resource.close()` or non-copyable & non-escapable types.

## Proposed solution

Allow execution of `deinit` and object deallocation to be the scheduled on the executor (either that of the actor itself or that of the relevant global actor), if needed.

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

  func click(_ times: Int) {
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

Note that since Swift 5.8 escaping `self` from `deinit` [reliably triggers fatal error](https://github.com/apple/swift/commit/108f780).

If needed, it is still possible to manually start a task from `deinit`. But any data needed by the task should be copied to avoiding capturing `self`:

```swift
actor Clicker {
  var count: Int = 0

  isolated deinit {
    Task { [count] in 
      await logClicks(count)
    }
  }
}
```

## Detailed design

### Runtime

This proposal introduces new runtime function:

```swift
@_silgen_name("swift_task_deinitOnExecutor")
@usableFromInline
internal func _deinitOnExecutor(_ object: __owned AnyObject,
                                _ work: @convention(thin) (__owned AnyObject) -> Void,
                                _ executor: UnownedSerialExecutor)
```

`swift_task_deinitOnExecutor` provides support for isolated synchronous `deinit`. If ensures that `deinit`'s code is running on the correct executor, by wrapping it into a task-less ad hoc job, if needed. It does not do any reference counting and can be safely used even with references that were released for the last time but not deallocated yet.

If no switching is needed, then `deinit` is executed immediately on the current thread. Otherwise, the task-less job is scheduled with the same priority as the task/thread that released the last strong reference to the object.

Note that `object` and `work` are imported in Swift as two separate arguments, because `work` consumes it's argument, which is currently not supported as a calling convention of Swift closures.

If `deinit` is isolated, code that normally is emitted into `__deallocating_init` gets emitted into a new entity (`__isolated_deallocating_init`), and `__deallocating_init` is emitted as a thunk that reads the executor (from `self` for actors and from the global actor for GAITs) and calls corresponding runtime function passing `self`, `__isolated_deallocating_init` and the desired executor.

Destroying `deinit` is not affected by this proposal.

### Rules for computing isolation

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

For consistency, `nonisolated` attribute also can be used, but for synchronous `deinit` it has no effect, because deinitializers are nonisolated by default.

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

Types that don't perform custom actions in `deinit` and only need to release references don't need isolated `deinit`. Releasing child objects can be done from any thread. If those objects are concerned about isolation, they should adopt isolation themselves. Implicit deinitializers cannot opt-in into isolation, so they are nonisolated by default.

```swift
class Foo {
  var bar: Bar

  // implicit deinit is nonisolated
  // release of Bar can happen on any thread/task
  // Bar is responsible to its own isolation.
}

class Bar {
  @MainActor deinit {}
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

### Exporting to Objective-C

`deinit` isolation is relevant only when subclassing. Since Objective-C code cannot subclass Swift classes, the generated  `*-Swift.h` files contain no addition information about new `deinit` features.

### Interaction with ObjC runtime.

All Objective-C-compatible Swift classes have `dealloc` method synthesized, which acts as a thunk to `__deallocating_deinit`. Normally when calling `super.deinit` from `__deallocating_deinit`, it is done by sending Objective-C `dealloc` message using `objc_msgSuper`.

This ensures maximum possible compatibility with Objective-C, but comes with some runtime cost.

For isolated synchronous `deinit`, this cost increases. For each class in the hierarchy `swift_task_deinitOnExecutor()` will be called, but slow path can be taken only for the most derived class. Other classes in the hierarchy would be doing redundant actor checks. To avoid this, isolated deinit bypasses Objective-C runtime when calling isolated `deinit` of Swift base classes, and calls `__isolated_deallocating_deinit` directly. Compatibility with Objective-C is preserved only on Swift<->Objective-C boundary in either direction.

This is still sufficient to create ObjC subclasses in runtime, as KVO does. Or swizzle `dealloc` of the most derived class of an instance. But swizzling `dealloc` of intermediate Swift classes might not work as expected.

### Isolated synchronous deinit of default actors

When deinitializing an instance of default actor, `swift_task_deinitOnExecutor()` attempts to take actor's lock and execute deinit on the current thread. If previous executor was another default actor, it remains locked. So potentially multiple actors can be locked at the same time. This does not lead to deadlocks, because (1) lock is acquired conditionally, without waiting; and (2) object cannot be deinitializer twice, so graph of the deinit calls has no cycles.

### Interaction with distributed actors

`deinit` declared in the code of the distributed actor applies only to the local actor and can be isolated as described above. Remote proxy has an implicit compiler-generated synchronous `deinit` which is never isolated.

### Interaction with task executor preference

Ad-hoc job created for isolated synchronous deinit is executed outside a task, so task executor preference does not apply.

### Task-local values

This proposal does not define how Swift runtime should behave when running isolated deinit. It may use task-local values as seen at the point of the last release, reset them to default values, or use some other set of values. Behavior is allowed to change without notice. But future proposals may change specification by defining a specific behavior.

Client code should not depend on behavior of particular implementation of the Swift runtime. Inside isolated `deinit` it is safe to read only the task-local values that were also set inside the `deinit`.

Note that any existing hopping in overridden `retain`/`release` for UIKit classes is unlikely to be aware of task-local values.

## Source compatibility

This proposal makes previously invalid code valid.

## Effect on ABI stability

This proposal does not change the ABI of existing language features, but does introduce new runtime functions.

## Effect on API resilience

Isolation attributes of the `deinit` become part of the public API, but they matter only when inheriting from the class.

Any changes to the isolation of `deinit` of non-open classes are allowed.

For open non-@objc classes, it is allowed to change synchronous `deinit` from isolated to nonisolated. Any non-recompiled subclasses will keep calling `deinit` of the superclass on the original actor. Changing `deinit` from nonisolated to isolated or changing identity of the isolating actor is a breaking change.

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
    <td>remove isolation</td><td>|</td><td>breaking</td><td>ok</td><td>ok</td>
  </tr>
  <tr>
    <td>add isolation</td><td>|</td><td>breaking</td><td>breaking</td><td>ok</td>
  </tr>
  <tr>
    <td>change actor</td><td>|</td><td>breaking</td><td>breaking</td><td>ok</td>
  </tr>
</table>

Adding an isolation annotation to a `dealloc` method of an imported Objective-C class is a breaking change. Existing Swift subclasses may have `deinit` isolated on different actor, and without recompilation will be calling `[super deinit]` on that actor. When recompiled, subclasses isolating on a different actor will produce a compilation error. Subclasses that had a non-isolated `deinit` (or a `deinit` isolated on the same actor) remain ABI compatible. It is possible to add isolation annotation to UIKit classes now, because currently all their subclasses have nonisolated `deinit`s.

Removing an isolation annotation from the `dealloc` method (together with `retain`/`release` overrides) is a breaking change. Any existing subclasses would be type-checked as isolated but compiled without isolation thunks. After changes in the base class, subclass `deinit`s could be called on the wrong executor.

If isolated deinit need to be suppressed in `.swiftinterface` for compatibility with older compilers, then `open` classes are emitted as `public` to prevent subclassing.

## Future Directions

### Implicit asynchronous `deinit`

Currently, if users need to initiate an asynchronous operation from `deinit`, they need to manually start a task. This requires copying all the needed data from the object, which can be tedious and error-prone. If some data is not copied explicitly, `self` will be captured implicitly, leading to a fatal error in runtime.

```swift
actor Service {
  func shutdown() {}
}

@MainActor
class ViewModel {
  let service: Service

  deinit {
    // Incorrect:
    _ = Task { await service.shutdown() }

    // Corrected version:
    _ = Task { [service] in await service.shutdown() }
  }
}
```

If almost every instance property is copied, then it would be more efficient to reuse original object as a task closure context and make `deinit` asynchronous:

```swift
  ...
  deinit async {
    await service.shutdown()

    // Destroy stored properties and deallocate memory
    // after asynchronous shutdown is complete
  }
}
```

Similarly to this proposal, `__deallocating_deinit` can be used as a thunk that starts an unstructured task for executing async deinit. But this is out of scope of this proposal.

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

Developers who put breakpoints in the isolated deinit might want to see the call stack that led to the last release of the object. Currently, if switching of executors was involved, the release call stack won't be shown in the debugger.

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

### Copy task-local values when hopping by default

This comes with a performance cost, which is unlikely to be beneficial to most of the users.
Leaving behavior of the task-locals undefined allows to potentially change it in the future, after getting more feedback from the users.

### Implicitly propagate isolation to synchronous `deinit`.

This would be a source-breaking change.

Majority of the `deinit`'s are implicitly synthesized by the compiler and only release stored properties. Global open source search in Sourcegraph, gives is 77.5k deinit declarations for 2.2m classes - 3.5%. Release can happen from any executor/thread and does not need isolation. Isolating implicit `deinit`s would come with a major performance cost. Providing special rules for propagating isolation to synchronous `deinit` unless it is implicit, would complicate propagation rules.
