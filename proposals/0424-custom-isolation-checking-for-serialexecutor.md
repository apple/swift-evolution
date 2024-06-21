# Custom isolation checking for SerialExecutor

* Proposal: [SE-0424](0424-custom-isolation-checking-for-serialexecutor.md)
* Author: [Konrad 'ktoso' Malawski](https://github.com/ktoso)
* Review Manager: [John McCall](https://github.com/rjmccall)
* Status: **Implemented (Swift 6.0)**
* Review: ([pitch](https://forums.swift.org/t/pitch-custom-isolation-checking-for-serialexecutor/69786)) ([review](https://forums.swift.org/t/se-0424-custom-isolation-checking-for-serialexecutor/70195)) ([acceptance](https://forums.swift.org/t/accepted-se-0424-custom-isolation-checking-for-serialexecutor/70480))

## Introduction

[SE-0392 (Custom Actor Executors)](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0392-custom-actor-executors.md) added support for custom actor executors, but its support is incomplete. Safety checks like [`Actor.assumeIsolated`](https://developer.apple.com/documentation/swift/actor/assumeisolated(_:file:line:)) work correctly when code is running on the actor through a task, but they don't work when code is scheduled to run on the actor's executor through some other mechanism. For example, if an actor uses a serial `DispatchQueue` as its executor, a function dispatched _directly_ to the queue with DispatchQueue.async cannot use `assumeIsolated` to assert that the actor is currently isolated. This proposal fixes this by allowing custom actor executors to provide their own logic for these safety checks.

## Motivation

The Swift concurrency runtime dynamically tracks the current executor of a running task in thread-local storage. To run code on behalf of a task, an executor must call into the runtime, and the runtime will set up the tracking appropriately. APIs like `assertIsolated` and `assumeIsolated` are built on top of that functionality and perform their checks by comparing the expected executor with the current executor tracked by the runtime. If the current thread is not running a task, the runtime treats it as if it were running a non-isolated function, and the comparison will fail.

This logic is not sufficient to handle the situation in which code is running on an actor's serial executor, but the code is not associated with a task. Swift's default actor executors currently do not provide any way to enqueue work on them that is not associated with a task, so this situation does not apply to them. However, many custom executors do provide other APIs for enqueuing work, such as the `async` method on `DispatchSerialQueue`. These APIs are not required to inform the Swift concurrency runtime before running the code.  As a result, the runtime will be unaware that the current thread is associated with an actor's executor, and checks like `assumeIsolated` will fail.  This is undesirable because, as long as the executor still acts like a serial executor for any non-task code it runs this way, the code will still be effectively actor-isolated: no code that accesses the actor's isolated state can run concurrently with it.

The following example demonstrates such a situation:

```swift
import Dispatch

actor Caplin {
  let queue: DispatchSerialQueue(label: "CoolQueue")

  var num: Int // actor isolated state

  // use the queue as this actor's `SerialExecutor`
  nonisolated var unownedExecutor: UnownedSerialExecutor {
    queue.asUnownedSerialExecutor()
  }
  
  nonisolated func connect() {
    queue.async {
      // guaranteed to execute on `queue`
      // which is the same as self's serial executor
      queue.assertIsolated() // CRASH: Incorrect actor executor assumption
      self.assumeIsolated {  // CRASH: Incorrect actor executor assumption
        num += 1
      }
    }
  }
}
```

Even though the code is executing on the correct Dispatch**Serial**Queue, the assertions trigger and we're left unable to access the actor's state, even though isolation-wise it would be safe and correct to do so.

Being able to assert isolation for non-task code this way is important enough that the Swift runtime actually already has a special case for it: even if the current thread is not running a task, isolation checking will succeed if the target actor is the `MainActor` and the current thread is the *main thread*. This problem is more general than the main actor, however; it exists for all kinds of threads which may be used as actor executors. The most important example of this is `DispatchSerialQueue`, especially because it is so commonly used in pre-concurrency code bases to provide actor-like isolation.  Allowing types like `DispatchSerialQueue` to hook into isolation checking makes it much easier to gradually migrate code to actors: if an actor uses a queue as its executor, existing code that uses the queue don't have to be completely rewritten in order to access the actor's state.

One way to think of this proposal is that gives all `SerialExecutor`s the power to provide a "fallback" check like this, rather than keeping it special-cased to `MainActor`.

## Proposed solution

We propose to add a new last-resort mechanism to executor comparison, which will be used by all the isolation-checking APIs in the concurrency library.

This will be done by providing a new `checkIsolation()` protocol requirement on `SerialExecutor`:

```swift
protocol SerialExecutor: Executor {
  // ...
 
  /// Invoked as last resort when the Swift concurrency runtime is performing an isolation
  /// assertion and could not confirm that the current execution context belongs to the
  /// expected executor.
  ///
  /// This function MUST crash the program with a fatal error if it is unable 
  /// to prove that this thread can currently be safely treated as isolated
  /// to this ``SerialExecutor``.  That is, if a synchronous function calls
  /// this method, and the method does not crash with a fatal error, 
  /// then the execution of the entire function must be well-ordered
  /// with any other job enqueued on this executor, as if it were part of
  /// a job itself.
  ///
  /// A default implementation is provided that unconditionally causes a fatal error.
  func checkIsolation()
}

extension SerialExecutor {
  public func checkIsolation() {
    fatalError("Incorrect actor executor assumption, expected: \(self)")
  }
}
```

## Detailed design

This proposal adds another customization point to the Swift concurrency runtime that hooks into isolation context comparison mechanisms used by `assertIsolated`, `preconditionIsolated`, and `assumeIsolated`, as well as any implicitly injected assertions used in `@preconcurrency` code.

### Extended executor comparison mechanism

With this proposal, the logic for checking if the current executor is the same as an expected executor changes, and can be expressed using the following pseudo-code:

```swift
// !!!! PSEUDO-CODE !!!! Simplified for readability.

let current = Task.current.executor

guard let current else {
  // no current executor, last effort check performed by the expected executor:
  expected.checkIsolated()

  // e.g. MainActor:
  // MainActorExecutor.checkIsolated() {
  //   guard Thread.isMain else { fatalError("Expected main thread!")
  //   return // ok!
  // }
}

if isSameSerialExecutor(current, expected) {
  // comparison takes into account "complex equality" as introduced by 'SE-0392
  return // ok!
} else {
  // executor comparisons failed...

  // give the expected executor a last chance to check isolation by itself:
  expected.checkIsolated()

  // as the default implementation of checkIsolated is to unconditionally crash,
  // this call usually will result in crashing -- as expected.
}

return // ok, it seems the expected executor was able to prove isolation
```

This pseudo code snippet explains the flow of the executor comparisons. There are two situations in which the new `checkIsolated` method may be invoked: when there is no current executor present, or if all other comparisons have failed.
For more details on the executor comparison logic, you can refer to [SE-0392: Custom Actor Executors](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0392-custom-actor-executors.md).

Specific use-cases of this API include `DispatchSerialQueue`, which would be able to implement the requirement as follows:

```swift
// Dispatch 

extension DispatchSerialQueue { 
  public func checkIsolated(message: String) {
    dispatchPrecondition(condition: .onQueue(self)) // existing Dispatch API
  }
}
```

An executor that wishes to take advantage of this proposal will need to have some mechanism to identity its active worker thread.  If that's not possible or desired, the executor should leave the default implementation (that unconditionally crashes) in place.

### Impact on async code and isolation assumtions

The `assumeIsolated(_:file:line:)` APIs purposefully only accept a **synchronous** closure. This is correct, and it remains correct with these proposed additions.  An isolation check on an executor ensures that any actor using the executor is synchronously isolated, and the closure provided to `assumeIsolated` will execute prior to any possible async suspension.  This is what makes it safe to access actor-isolated state within the closure.

This means that the following code snippet, while a bit unusual remains correct isolation-wise:

```swift
actor Worker {
  var number: Int 
  
  nonisolated func canOnlyCallMeWhileIsolatedOnThisInstance() -> Int {
    self.preconditionIsolated("This method must be called while isolated to \(self)")

    return self.assumeIsolated { // () throws -> Int
      // suspensions are not allowed in this closure.
      
      self.number // we are guaranteed to be isolated on this actor; read is safe
    }
  }
      
```

As such, there is no negative impact on the correctness of these APIs.

Asynchronous functions should not use dynamic isolation checking.  Isolation checking is useful in synchronous functions because they naturally inherit execution properties like their caller's isolation without disturbing it.  A synchronous function may be formally non-isolated and yet actually run in an isolated context dynamically.  This is not true for asynchronous functions, which switch to their formal isolation on entry without regard to their caller's isolation.  If an asynchronous function is not formally isolated to an actor, its execution will never be dynamically in an isolated context, so there's no point in checking for it.

## Future directions

### Introduce `globalMainExecutor` global property and utilize `checkIsolated` on it

This proposal also paves the way to clean up this hard-coded aspect of the runtime, and it would be possible to change these heurystics to instead invoke the `checkIsolation()` method on a "main actor executor" SerialExecutor reference if it were available.

This proposal does not introduce a `globalMainActorExecutor`, however, similar how how [SE-0417: Task ExecutorPreference](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0417-task-executor-preference.md) introduced a:

```swift
nonisolated(unsafe)
public var globalConcurrentExecutor: any TaskExecutor { get }
```

the same could be done to the MainActor's executor:

```swift
nonisolated(unsafe)
public var globalMainExecutor: any SerialExecutor { get }
```

The custom heurystics that are today part of the Swift Concurrency runtime to detect the "main thread" and "main actor executor", could instead be delegated to this global property, and function correctly even if the MainActor's executor is NOT using the main thread (which can happen on some platforms):

```swift
// concurrency runtime pseudo-code
if expectedExecutor.isMainActor() {
  expectedExecutor.checkIsolated(message: message)
}
```

This would allow the isolation model to support different kinds of main executor and properly assert their isolation, using custom logic, rather than hardcoding the main thread assumptions into the Swift runtime.

## Alternatives considered

### Do not provide customization points, and just hardcode DispatchQueue handling

Alternatively, we could hardcode detecting dispatch queues and triggering `dispatchPrecondition` from within the Swift runtime.

This is not a good direction though, as our goal is to have the concurrency runtime be less attached to Dispatch and allow Swift to handle each and every execution environment equally well. As such, introducing necessary hooks as official and public API is the way to go here.
