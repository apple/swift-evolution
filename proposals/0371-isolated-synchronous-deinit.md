# Isolated synchronous deinit

* Proposal: [SE-0371](0371-isolated-synchronous-deinit.md)
* Author: [Mykola Pokhylets](https://github.com/nickolas-pohilets)
* Review Manager: [Frederick Kellison-Linn](https://github.com/jumhyn)
* Status: **Returned for revision**
* Implementation: [apple/swift#60057](https://github.com/apple/swift/pull/60057)
* Review: ([pitch](https://forums.swift.org/t/isolated-synchronous-deinit/58177)) ([review](https://forums.swift.org/t/se-0371-isolated-synchronous-deinit/59754)) ([returned for revision](https://forums.swift.org/t/returned-for-revision-se-0371-isolated-synchronous-deinit/60060))

## Introduction

This feature allows `deinit`'s of actors and global-actor isolated types (GAITs) to access non-sendable isolated state, lifting restrictions imposed imposed by [SE-0327](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0327-actor-initializers.md). This is achieved by providing runtime support for hopping onto executors in `__deallocating_deinit()`'s.

## Motivation

The combination of automatic reference counting and deterministic deinitialization makes `deinit` in Swift a powerful tool for resource management. It greatly reduces need for `close()`-like methods (`unsubscribe()`, `cancel()`, `shutdown()`, etc.) in the public API. Such methods not only clutter the public API, but also introduce a state where object is already unusable but is still able to be referenced.

Restrictions imposed by [SE-0327](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0327-actor-initializers.md) reduce the usefulness of explicit `deinit`s in actors and GAITs. Workarounds for these limitations may involve creation of `close()`-like methods, or even manual reference counting if the API should be able to serve several clients.

In cases when `deinit` belongs to a subclass of `UIView` or `UIViewController` which are known to call `dealloc` on the main thread, developers may be tempted to silence the diagnostic by adopting `@unchecked Sendable` in types that are not actually  sendable. This undermines concurrency checking by the compiler, and may lead to data races when using incorrectly marked types in other places.

## Proposed solution

Allow execution of `deinit` and object deallocation to be the scheduled on the executor of the containing type (either that of the actor itself or that of the relevant global actor), if needed.

Let's consider [examples from SE-0327](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0327-actor-initializers.md#data-races-in-deinitializers):

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

  deinit {
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

In the case of escaping `self`, the race condition is eliminated but the problem of dangling reference remains. This problem exists for synchronous code as well and is orthogonal to the concurrency features.

```swift
actor Clicker {
  var count: Int = 0

  func click(_ times: Int) {
    for _ in 0..<times {
      self.count += 1 
    }
  }

  deinit {
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

## Detailed design

### Runtime

This proposal introduces a new runtime function that is used to schedule `deinit`'s code on an executor by wrapping it into a task-less ad hoc job. It does not do any reference counting and can be safely used even with references that were released for the last time but not deallocated yet.

If no switching is needed, then `deinit` is executed immediately on the current thread. Otherwise, the task-less job copies priority and task-local values from the task/thread that released the last strong reference to the object.

```cpp
using DeinitWorkFunction = SWIFT_CC(swift) void (void *);
  
SWIFT_EXPORT_FROM(swift_Concurrency) SWIFT_CC(swift)
void swift_task_deinitOnExecutor(void *object, DeinitWorkFunction *work, ExecutorRef newExecutor);
```

```swift
@_silgen_name("swift_task_deinitOnExecutor")
@usableFromInline
internal func _deinitOnExecutor(_ object: __owned AnyObject,
                                _ work: @convention(thin) (__owned AnyObject) -> Void,
                                _ executor: UnownedSerialExecutor)
```

If `deinit` is isolated, code that normally is emitted into `__deallocating_init` gets emitted into a new entity (`__isolated_deallocating_init`), and `__deallocating_init` is emitted as a thunk that reads the executor (from `self` for actors and from the global actor for GAITs) and calls `swift_task_performOnExecutor` passing `self`, `__isolated_deallocating_init` and the desired executor.

Non-deallocating `deinit` is not affected by this proposal.

### Rules for computing isolation

Isolation of `deinit` comes with runtime and code size cost. Types that don't perform custom actions in `deinit` and only need to release references don't need isolated `deinit`. Releasing child objects can be done from any thread. If those objects are concerned about isolation, they should adopt isolation themselves.

```swift
@MainActor
class Foo {
    let bar: Bar
    
    // No isolated deinit generated.
    // Reference to Bar can be released from any thread.
    // Class Bar is responsible for correctly isolating its own deinit.
}

actor MyActor {
    let bar: Bar
    
    // Similar
}
```

If there is an explicit `deinit`, then isolation is computed following usual rules for isolation of class instance members as defined by [SE-0313](https://github.com/gottesmm/swift-evolution/blob/move-function-pitch-v1/proposals/0313-actor-isolation-control.md) and [SE-0316](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0316-global-actors.md). It takes into account isolation attributes on the `deinit` itself, isolation of the `deinit` in the superclass, and isolation attributes on the containing class. If deinit belongs to an actor or GAIT, but isolation of the `deinit` is undesired, it can be suppressed using `nonisolated` attribute:

```swift
@MainActor
class Foo {
    deinit {} // Isolated on MainActor
}

@FirstActor
class Bar {
    @SecondActor
    deinit {} // Isolated on SecondActor
}

@MainActor
class Baz {
    nonisolated deinit {} // Not isolated
}

actor MyActor {
    deinit {} // Isolated on self
}

actor AnotherActor {
    nonisolated deinit {} // Not isolated
}
```

When inheritance is involved, computed isolation is then validated to be compatible with isolation of the `deinit` of the base class.
Classes can add isolation to the non-isolated `deinit` of the base class, but they cannot change (remove or change actor) existing isolation.

```swift
@FirstActor
class Base {} // deinit is not isolated

class Derived: Base {
    @SecondActor deinit { // Isolated on SecondActor
    }
}

class IsolatedBase {
    @FirstActor deinit {} // Isolated on FirstActor
}

class Derived1: IsolatedBase {
    // deinit is still isolated on FirstActor
}

class Derived2: IsolatedBase {
    nonisolated deinit {} // ERROR
}

class Derived3: IsolatedBase {
    @SecondActor deinit {} // ERROR
}
```

Note that this is opposite from the validation rules for overriding functions. That's because in case of `deinit`, isolation applies to the non-deallocating `deinit`, while overriding happens for `__deallocating_deinit`. `__deallocating_deinit` is always non-isolated, and is responsible for switching to correct executor before calling non-deallocating `deinit`. Non-deallocating `deinit` of the subclass calls non-deallocating `deinit` of the superclass. And it is allowed to call nonisolated function from isolated one.

### Importing Objective-C code

Objective-C compiler does not generate any code to make `dealloc` isolated and marking Objective-C classes as isolated on global actor using `__attribute__((swift_attr(..)))` has no effect on behavior of the ObjC code.
Such classes are imported into Swift as having non-isolated `deinit`.

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

`deinit` isolation is relevant only when subclassing since Objective-C code cannot subclass Swift classes. The generated  `*-Swift.h` files contain no information about `deinit` isolation.

### Isolated deinit of default actors

When deinitializing an instance of default actor, `swift_task_deinitOnExecutor()` attempts to take actor's lock and execute deinit on the current thread. If previous executor was another default actor, it remains locked. So potentially multiple actors can be locked at the same time. This does not lead to deadlocks, because (1) lock is acquired conditionally, without waiting; and (2) object cannot be deinitializer twice, so graph of the deinit calls has no cycles.

### Interaction with distributed actors

`deinit` declared in the code of the distributed actor applies only to the local actor and can be isolated as described above. Remote proxy has an implicit compiler-generated `deinit` which is never isolated.

## Source compatibility

This proposal makes previously invalid code valid.

This proposal may alter behavior of existing GAIT and actors when the last release happens on a different actor. But it is very unlikely that `deinit` of GAIT or an actor would have a synchronous externally-observable side effect that can be safely executed on a different actor.

## Effect on ABI stability

This proposal does not change the ABI of existing language features, but does introduce a new runtime function.

## Effect on API resilience

Isolation attributes of the `deinit` become part of the public API, but it matters only when inheriting from the class.

Changing a `deinit` from nonisolated to isolated is allowed on final Swift classes. But for non-final classes it is not allowed, because this effects how `deinit`s of the subclasses are generated.

The same is true for changing identity of the isolating actor.

Changing a `deinit` from isolated to nonisolated does not break ABI for Swift classes, including `@objc`-classes. Any non-recompiled subclasses will keep calling `deinit` of the superclass on the original actor.

Adding an isolation annotation to a `dealloc` method of an imported Objective-C class is a breaking change. Existing Swift subclasses may have `deinit` isolated on different actor, and without recompilation will be calling `[super deinit]` on that actor. When recompiled, subclasses isolating on a different actor will produce a compilation error. Subclasses that had a non-isolated `deinit` (or a `deinit` isolated on the same actor) remain ABI compatible. It is possible to add isolation annotation to UIKit classes, because currently all their subclasses have nonisolated `deinit`s.

Removing an isolation annotation from the `dealloc` method (together with `retain`/`release` overrides) is a breaking change. Any existing subclasses would be type-checked as isolated but compiled without isolation thunks. After changes in the base class, subclass `deinit`s could be called on the wrong executor.

## Future Directions

### Asynchronous `deinit`

Currently, if users need to initiate an asynchronous operation from `deinit`, they need to manually start a task. It is easy to accidentally capture `self` inside task's closure. Such `self` would become a dangling reference which may cause a crash, but is not guaranteed to reproduce during debugging.

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

A more developer-friendly approach would be to allow asynchronous deinit:

```swift
  ...
  deinit async {
    await service.shutdown()

    // Destroy stored properties and deallocate memory
    // after asynchronous shutdown is complete
  }
}
```

Similarly to this proposal, `__deallocating_deinit` can be used as a thunk that starts an unstructured task for executing async deinit. But such naïve approach can flood the task scheduler when deallocating large data structure with many async `deinit`s. More research is needed to understand typical usage of asynchronous operations in `deinit`, and applicable optimization methods. This is out of scope of this proposal.

### Asserting that `self` does not escape from `deinit`.

It is not legal for `self` to escape from `deinit`. Any strong references remaining after `swift_deallocObject()` is executed cannot be even released safely. Dereferencing dangling references can manifest itself as a crash, reading garbage data or corruption of unrelated memory. It can be hard to connect symptoms with the origin of the problem. A better solution would be to deterministically produce `fatalError()` in `swift_deallocObject()` if there are additional strong references. The ideal solution would be to detect escaping `self` using compile-time analysis.

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

Currently both the `foo()` and `deinit` entry points produce two calls to access the `MainActor.shared.unownedExecutor`, with the second one even using dynamic dispatch.
These two calls could be replaced with a single call to the statically referenced `swift_task_getMainExecutor()`.

```llvm
%1 = tail call swiftcc %swift.metadata_response @"type metadata accessor for Swift.MainActor"(i64 0) #6
%2 = extractvalue %swift.metadata_response %1, 0
%3 = tail call swiftcc %TScM* @"static Swift.MainActor.shared.getter : Swift.MainActor"(%swift.type* swiftself %2)
%4 = tail call i8** @"lazy protocol witness table accessor for type Swift.MainActor and conformance Swift.MainActor : Swift.Actor in Swift"() #6
%5 = bitcast %TScM* %3 to %objc_object*
%6 = tail call swiftcc { i64, i64 } @"dispatch thunk of Swift.Actor.unownedExecutor.getter : Swift.UnownedSerialExecutor"(%objc_object* swiftself %5, %swift.type* %2, i8** %4)
```

### Making fast path inlinable

For this to be useful the compiler first needs to be able to reason about the value of the current executor based on the isolation of the surrounding function. Then, an inlinable pre-check for `swift_task_isCurrentExecutor()` can be inserted allowing the isolated deallocating `deinit` to be inlined into the non-isolated one.

### Improving extended stack trace support

Developers who put breakpoints in the isolated deinit might want to see the call stack that lead to the last release of the object. Currently, if switching of executors was involved, the release call stack won't be shown in the debugger.

### Implementing API for synchronously scheduling arbitrary work on the actor

The introduced runtime function has calling convention optimized for the `deinit` use case, but using a similar runtime function with a slightly different signature, one could implement an API for synchronously scheduling arbitrary work on the actor:

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

### Isolated deinit for move-only types without isolation thunk

> [Joe Groff](https://forums.swift.org/t/isolated-synchronous-deinit/58177/17):
>
> Classes with shared ownership are ultimately the wrong tool for the job—they may be the least bad tool today, though. Ultimately, when we have move-only types, then since those have unique ownership, we'd be able to reason more strongly about what context their deinit executes in, since it would either happen at the end of the value's original lifetime, or if it's moved to a different owner, when that new owner consumes it. Within move-only types, we could also have a unique modifier for class types, to indicate statically that an object reference is the only one to the object, and that its release definitely executes deinit.

For move-only types as a tool for explicit resource management, it may be desired to have switching or not switching actor to execute isolating `deinit` to be explicit as well. Such types could have isolated `deinit` without isolating thunk, and instead compiler would check that value is dropped on the correct actor.

```swift
@moveonly
struct Resource {
    @MainActor
    deinit {
        ...
    }
}

@MainActor func foo() {
    let r: Resource = ...
    Task.detached { [move r] in
        // error: expression is 'async' but is not marked with 'await'
        // note: dropping move-only value with isolated deinit from outside of its actor context is implicitly asynchronous
        drop(r)
    }
}
```

## Alternatives considered

### Placing hopping logic in `swift_release()` instead.

`UIView` and `UIViewController` implement hopping to the main thread by overriding the `release` method. But in Swift there are no vtable/wvtable slots for releasing, and adding them would also affect a lot of code that does not need isolated deinit.

### Deterministic task local values

When switching executors, the current implementation copies priority and task-local values from the task/thread where the last release happened. This minimizes differences between isolated and nonisolated `deinit`s.

If there are references from different tasks/threads, the values of the priority and task-local values observed by the `deinit` are racy, both for isolated and nonsiolated `deinit`s.

One way of making task-local values predictable would be to clear them for the duration of the `deinit` execution.
This can be implemented efficiently, but would be too restrictive.
If the object is referenced in several tasks which all have a common parent, then `deinit` can reliably use task-local values which are known to be set in the parent task and not overridden in the child tasks.

If there is a demand for resetting task-local values, it can be implemented separately as an API:

```swift
// Temporary resets all task-local values to their defaults
// by appending a stop-node to the linked-list of task-locals 
func withoutTaskLocalValues(operation: () throws -> Void) rethrows
```

### Don't isolate empty explicit `deinit`

Empty explicit `deinit`s are also eligible to be nonisolated as an optimization. But that would mean that the interface of the declaration depends on its implementation. Currently, Swift infers function signature only for closure literals, but never for named functions.

### Explicit opt-in into `deinit` isolation

This would eliminate any possibility of unexpected changes in behavior of the existing code, but would penalize future users, creating inconsistency between isolation inference rules for `deinit` and regular methods. Currently there is syntax for opt-out from isolation for actor instance members, but there is no syntax for opt-in. Having opt-in for `deinit`s would require introducing such syntax, and it would be used only for actor `deinit`s. There is already the `isolated` keyword, but it is applied to function arguments, not to function declarations.

Classes whose `deinit`s have nonisolated synchronous externally-visible side effects, like `AnyCancellable`, are unlikely to be isolated on global actors or be implemented as an actor.

This proposal preserves behavior of `deinit`s that have synchronous externally-visible side effects only under assumption that they are always released on the isolating actor.

`deinit`s that explicitly notify about completion of their side effect continue to satisfy their contract even if proposal changes their behavior.

### Use asynchronous deinit as the only tool for `deinit` isolation

Synchronous `deinit` has an efficient fast path that jumps right into the `deinit` implementation without context switching, or any memory allocations. But asynchronous `deinit` would require creation of task in all cases.
