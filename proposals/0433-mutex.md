# Synchronous Mutual Exclusion Lock 🔒

* Proposal: [SE-0433](0433-mutex.md)
* Author: [Alejandro Alonso](https://github.com/Azoy)
* Review Manager: [Stephen Canon](https://github.com/stephentyrone)
* Implementation: [apple/swift#71383](https://github.com/apple/swift/pull/71383)
* Status: **Accepted**
* Review: ([pitch](https://forums.swift.org/t/pitch-synchronous-mutual-exclusion-lock/69889)), ([review](https://forums.swift.org/t/se-0433-synchronous-mutual-exclusion-lock/71174)), ([acceptance](https://forums.swift.org/t/accepted-se-0433-synchronous-mutual-exclusion-lock/71463))

## Introduction

This proposal introduces a mutual exclusion lock, or a mutex, to the standard library. `Mutex` will be a new synchronization primitive in the synchronization module.

## Motivation

In concurrent programs, protecting shared mutable state is one of the core fundamental problems to ensuring reading and writing data is done in an explainable fashion. Synchronizing access to shared mutable state is not a new problem in Swift. We've introduced many features to help protect mutable data. Actors are a good default go-to solution for protecting mutable state because it isolates the stored data in its own domain. At any given point in time, only one task will be executing "on" the actor, and have exclusive access to it. Multiple tasks cannot access state protected by the actor at the same time, although they may interleave execution at potential suspension points (indicated by `await`). In general, the actor approach also lends itself well to code organization, since the actor's state, and operations on this state are logically declared in the same place: inside the actor.

Not all code may be able (or want) to adopt actors. Reasons for this can be very varied, for example code may have to execute synchronously without any potential for other tasks interleaving with it. Or the `async` effect introduced on methods may prevent legacy code which cannot use Swift Concurrency from interacting with the protected state.

Whatever the reason may be, it may not be feasible to use an actor. In such cases, Swift currently is missing standard tools for developers to ensure proper synchronization in their concurrent data-structures. Many Swift programs opt to use ad-hoc implementations of a mutual exclusion lock, or a mutex. A mutex is a simple to use synchronization primitive to help protect shared mutable data by ensuring that a single execution context has exclusive access to the related data. The main issue is that there isn't a single standardized implementation for this synchronization primitive resulting in everyone needing to roll their own.

## Proposed solution

We propose a new type in the Standard Library Synchronization module: `Mutex`. This type will be a wrapper over a platform-specific mutex primitive, along with a user-defined mutable state to protect. Below is an example use of `Mutex` protecting some internal data in a class usable simultaneously by many threads:

```swift
class FancyManagerOfSorts {
  let cache = Mutex<[String: Resource]>([:])
  
  func save(_ resource: Resource, as key: String) {
    cache.withLock {
      $0[key] = resource
    }
  }
}
```

Use cases for such a synchronized type are common. Another common need is a global cache, such as a dictionary:

```swift
let globalCache = Mutex<[MyKey: MyValue]>([:])
```

### Toolchains

You can try out `Mutex` using one of the following toolchains:

macOS: https://ci.swift.org/job/swift-PR-toolchain-macos/1207/artifact/branch-main/swift-PR-71383-1207-osx.tar.gz

Linux (x86_64): https://download.swift.org/tmp/pull-request/71383/779/ubuntu2004/PR-ubuntu2004.tar.gz

Windows: `https://ci-external.swift.org/job/swift-PR-build-toolchain-windows/1200/artifact/*zip*/archive.zip`

Note that these toolchains don't currently have the `transferring inout` implemented, but the functions are marked `@Sendable` to at least enforce sendability.

## Detailed design

### Underlying System Mutex Implementation

The `Mutex` type proposed is a wrapper around a platform's implementation.

* macOS, iOS, watchOS, tvOS, visionOS:
  * `os_unfair_lock`
* Linux:
  * `futex`
* Windows:
  * `SRWLOCK`

These mutex implementations all have different capabilities and guarantee different levels of fairness. Our proposed `Mutex` type does not guarantee fairness, and therefore it's okay to have different behavior from platform to platform. We only guarantee that only one execution context at a time will have access to the critical section, via mutual exclusion.

### API Design

Below is the complete API design for the new `Mutex` type:

```swift
/// A synchronization primitive that protects shared mutable state via
/// mutual exclusion.
///
/// The `Mutex` type offers non-recursive exclusive access to the state
/// it is protecting by blocking threads attempting to acquire the lock.
/// At any one time, only one execution context at a time has access to
/// the value stored within the `Mutex` allowing for exclusive access.
///
/// An example use of `Mutex` in a class used simultaneously by many
/// threads protecting a `Dictionary` value:
///
///     class Manager {
///       let cache = Mutex<[Key: Resource]>([:])
///
///       func saveResouce(_ resource: Resouce, as key: Key) {
///         cache.withLock {
///           $0[key] = resource
///         }
///       }
///     }
///
/// - Warning: Instances of this type are not recursive. Calls
///   to `withLock(_:)` (and related functions) within their
///   closure parameters will have platform-dependent behavior.
///   Some platforms may choose to panic the process, deadlock,
///   or leave this behavior unspecified.
///
public struct Mutex<State: ~Copyable>: ~Copyable {
  /// Initializes an instance of this mutex with the given initial state.
  ///
  /// - Parameter state: The initial state to give to the mutex.
  public init(_ state: transferring consuming State)
}
  
extension Mutex: Sendable where State: ~Copyable {}
  
extension Mutex where State: ~Copyable {
  /// Calls the given closure after acquring the lock and then releases
  /// ownership.
  ///
  /// This method is equivalent to the following sequence of code:
  ///
  ///     mutex.lock()
  ///     defer {
  ///       mutex.unlock()
  ///     }
  ///     return try body(&value)
  ///
  /// - Warning: Recursive calls to `withLock` within the
  ///   closure parameter has behavior that is platform dependent.
  ///   Some platforms may choose to panic the process, deadlock,
  ///   or leave this behavior unspecified. This will never
  ///   reacquire the lock however.
  ///
  /// - Parameter body: A closure with a parameter of `State`
  ///   that has exclusive access to the value being stored within
  ///   this mutex. This closure is considered the critical section
  ///   as it will only be executed once the calling thread has
  ///   acquired the lock.
  ///
  /// - Returns: The return value, if any, of the `body` closure parameter.
  public borrowing func withLock<Result: ~Copyable, E: Error>(
    _ body: (transferring inout State) throws(E) -> transferring Result
  ) throws(E) -> transferring Result
  
  /// Attempts to acquire the lock and then calls the given closure if
  /// successful.
  ///
  /// If the calling thread was successful in acquiring the lock, the
  /// closure will be executed and then immediately after it will
  /// release ownership of the lock. If we were unable to acquire the
  /// lock, this will return `nil`.
  ///
  /// This method is equivalent to the following sequence of code:
  ///
  ///     guard mutex.tryLock() else {
  ///       return nil
  ///     }
  ///     defer {
  ///       mutex.unlock()
  ///     }
  ///     return try body(&value)
  ///
  /// - Warning: Recursive calls to `withLockIfAvailable` within the
  ///   closure parameter has behavior that is platform dependent.
  ///   Some platforms may choose to panic the process, deadlock,
  ///   or leave this behavior unspecified. This will never
  ///   reacquire the lock however.
  ///
  /// - Parameter body: A closure with a parameter of `State`
  ///   that has exclusive access to the value being stored within
  ///   this mutex. This closure is considered the critical section
  ///   as it will only be executed if the calling thread acquires
  ///   the lock.
  ///
  /// - Returns: The return value, if any, of the `body` closure parameter
  ///   or nil if the lock couldn't be acquired.
  public borrowing func withLockIfAvailable<Result: ~Copyable, E: Error>(
    _ body: (transferring inout State) throws(E) -> transferring Result?
  ) throws(E) -> transferring Result?
}
```

## Interaction with Existing Language Features

`Mutex` will be decorated with the `@_staticExclusiveOnly` attribute, meaning you will not be able to declare a variable of type `Mutex` as `var`. These are the same restrictions imposed on the recently accepted `Atomic` and `AtomicLazyReference` types. Please refer to the [Atomics proposal](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0410-atomics.md) for a more in-depth discussion on what is allowed and not allowed. These restrictions are enabled for `Mutex` for all of the same reasons why it was resticted for `Atomic`. We do not want to introduce dynamic exclusivity checking when accessing a value of `Mutex` as a class stored property for instance.

### Interactions with Swift Concurrency

`Mutex` is unconditionally `Sendable` regardless of the value it's protecting. We can ensure the safetyness of this value due to the `transferring` marked parameters of both the initializer and the closure `inout` argument. (Please refer to [SE-0430 `transferring` isolation regions of parameter and result values](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0430-transferring-parameters-and-results.md)) This allows us to statically determine that the non-sendable value we're initializing the mutex with will have no other uses after initialization. Within the closure body, it ensures that if we tried to escape the protected non-sendable value, it would require us to replace the stored value for the notion of transferring out and then transferring back in.

Consider the following example of a mutex to a non-sendable class.

```swift
class NonSendableReference {
  var prop: UnsafeMutablePointer<Int>
}

// Some non-sendable class reference somewhere, perhaps a global.
let nonSendableRef = NonSendableReference(...)

let lockedPointer = Mutex<UnsafeMutablePointer<Int>>(...)

func something() {
  lockedPointer.withLock {
    // error: isolated parameter transferred out
    //        but hasn't had a value transferred back in.
    nonSendableRef.prop = $0
  }
}
```

Had this closure not been marked `transferring inout` or perhaps `@Sendable`, then `Mutex` would not have protected this class references or any underlying memory referenced by pointers. Transferring inout allows us to plug this safety hole of shared mutable state by statically requiring that if we need to escape the closure or `Mutex` isolation domain as a whole, that we transfer in a new value in this domain (or it could be the same value perhaps).

```swift
func something() {
  lockedPointer.withLock {
    // OK because we assign '$0' to a new value
    nonSendableRef.prop = $0
    
    // OK because we're transferring a new value in
    $0 = ...
  }
}
```

By marking the closure as such, we've effectively declared that the mutex is in itself its own isolation domain. We must not let non-sendable values it holds onto be unsafely sent across isolation domains to prevent these holes of shared mutable state.

### Differences between mutexes and actors

The mutex type we're proposing is a synchronous lock. This means when other participants want to acquire the lock to access the protected shared data, they will halt execution until they are able to do so. Threads that are waiting to acquire the lock will not be able to make forward progress until their request to acquire the lock has completed. This can lead to thread contention if the acquired thread's critical section is not able to be executed relatively quickly exhausting resources for the rest of the system to continue making forward progress. Synchronous locks are also prone to deadlocks (which Swift's actors cannot currently encounter due to their re-entrant nature) and live-locks which can leave a process in an unrecoverable state. These scenarios can occur when there is a complex hierarchy of different locks that manage to depend on the acquisition of each other.

Actors work very differently. Typical use of an actor doesn't request access to underlying shared data, but rather instruct the actor to perform some operation or service that has exclusive access to that data. An execution context making this request may need to await on the return value of that operation, but with Swift's `async`/`await` model it can immediately start doing other work allowing it to make forward progress on other tasks. The actor executes requests in a serial fashion in the order they are made. This ensures that the shared mutable state is only accessed by the actor. Deadlocks are not possible with the actor model. Asynchronous code that is dependent on a specific operation and resouce from an actor can be later resumed once the actor has serviced that request. While deadlocking is not possible, there are other problems actors have such as the actor reentrancy problem where the state of the actor has changed when the executing operation got resumed after a suspension point.

Mutexes and actors are very different synchronization tools that help protect shared mutable state. While they can both achieve synchronization of that data access, they do so in varying ways that may be desirable for some and undesirable for others. The proposed `Mutex` is yet another primitive that Swift should expose to help those achieve concurrency safe programs in cases where actors aren't suitable.

## Source compatibility

Source compatibility is preserved with the proposed API design as it is all additive as well as being hidden behind an explicit `import Synchronization`. Users who have not already imported the Synchronization module will not see this type, so there's no possibility of potential name conflicts with existing `Mutex` named types for instance. Of course, the standard library already has the rule that any type names that collide will disfavor the standard library's variant in favor of the user's defined type anyway.

## ABI compatibility

The API proposed here is fully addative and does not change or alter any of the existing ABI.

`Mutex` as proposed will be a new `@frozen` struct which means we cannot change its layout in the future on ABI stable platforms, namely the Darwin family. Because we cannot change the layout, we will most likely not be able to change to a hypothetical new and improved system mutex implementation on those platforms. If said new system mutex were to share the layout of the currently proposed underlying implementation, then we _may_ be able to migrate over to that implementation. Keep in mind that Linux and Windows are non-ABI stable platforms, so we can freely change the underlying implementation if the platform ever supports something better.

## Future directions

There are quite a few potential future directions this new type can take as well as new future similar types.

### Mutex Guard API

A token based approach for locking and unlocking may also be highly desirable for mutex API. This is similar to C++'s `std::lock_guard` or Rust's `MutexGuard`:

```swift
extension Mutex {
  public struct Guard: ~Copyable, ~Escapable {
    // Hand waving some syntax to borrow Mutex, or perhaps
    // we just store a pointer to it.
    let mutex: borrow Mutex<State>
    
    public var value: Value {...}
    
    deinit {
      mutex.unlock()
    }
  }
}

extension Mutex {
  public borrowing func lock() -> borrow(self) Guard {...}
  
  public borrowing func tryLock() -> borrow(self) Guard? {...}
}

func add(_ i: Int, to mutex: Mutex<Int>) {
  // This acquires the lock by calling the platform's
  // underlying 'lock()' primitive.
  let mGuard = mutex.lock()
  
  mGuard.value += 1
  
  // At the end of the scope, mGuard is immediately deinitialized
  // and releases the mutex by calling the platform's
  // 'unlock()' primitive.
}
```

The above example shows an API similar to Rust's `MutexGuard` which allows access to the protected state in the mutex. C++'s guard on the other hand just performs `lock()` and `unlock()` for the user (because `std::mutex` doesn't protect any state). Of course the immediate issue with this approach right now is that we don't have access to non-escapable types. When one were to call `lock()`, there's nothing preventing the user from taking the guard value and escaping it from the scope that the caller is in. Rust resolves this issue with lifetimes, but C++ doesn't solve this at all and just declares:

> The behavior is undefined if m is destroyed before the `lock_guard` object is.

Which is not something we want to introduce in Swift if it's something we can eventually prevent. If we had this feature today, the primitive `lock()`/`unlock()` operations would be better suited in the form of the guard API. I don't believe we'd have those methods if we had guards.

### Reader-Writer Locks, Recursive Locks, etc.

Another interesting future direction is the introduction of new kinds of locks to be added to the standard library, such as a reader-writer lock. One of the core issues with the proposal mutual exclusion lock is that anyone who takes the lock, either a reader and/or writer, must be the only person with exclusive access to the protected state. This is somewhat unfortunate for models where there are infinitely more readers than there will be writers to the state. A reader-writer lock resolves this issue by allowing multiple readers to take the lock and enforces that writers who need to mutate the state have exclusive access to the value. Another potential lock is a recursive lock who allows the lock to be acquired multiple times by the acquired thread. In the same vein, the acquired thread needs to be the one to release the lock and needs to release X amount of times equal to the number of times it acquired it.

## Alternatives considered

### Implement `lock()`, `unlock()`, and `tryLock()`

Seemingly missing from the `Mutex` type are the primitive locking and unlocking functions. These functions are fraught with peril in both Swift's concurrency model and in its ownership model.

In the face of `async`/`await`, these primitives are very dangerous. The example below highlights incorrect usage of these operations in an `async` function:

```swift
func test() async {
  mutex.lock()             // Called on Thread A
  await downloadImage(...) // <--- Potential suspension point
  mutex.unlock()           // May be called on Thread A, B, C, etc.
}
```

The potential suspension point may cause the proceeding code to be called on a different thread than the one that initiated the `await` call. We can make these primitives safe in asynchronous contexts though by disallowing their use altogether by marking them `@available(*, noasync)`. Calling `withLock` in an asynchronous function is \_okay\_ because the same thread that calls `lock()` will be the same one that calls `unlock()` because there will not be any suspension points between the calls.

Another bigger issue is how these functions interact with the ownership model.

```swift
// borrow access begins
mutex.lock()
// borrow access ends
...
// borrow access begins
mutex.unlock()
// borrow access ends
```

In the above, I've modeled where the borrowing accesses occur when calling these functions. This is an important distinction to make because unlike C++'s similar synchronization primitives, `Mutex` (and similarly `Atomic`) can be _moved_. These types guarantee a stable address "for the duration of a borrow access", but as you can see there's nothing guaranteeing that the unlock is occurring on the same address as the call to lock. As opposed to the closure based API (and hopefully in the future the guard based API):

```swift
// borrow access begins
mutex.withLock {
  ...
}
// borrow access ends

do {
  // borrow access ends
  let locked = mutex.lock()
  ...
  
  // borrow access ends when 'locked' gets deinitialized
}
```

In the first example with the closure, we syntactically define our borrow access with the closure because the entire closure will be executed during the borrow access of the mutex. In the second example, the guarded value we get back from a guard based `lock()` will extend the duration of the borrow access for as long as the `locked` binding is available.

Providing these APIs on `Mutex` would be incredibly unsafe. We feel that the proposed `withLock` closure based API is much safer and sufficient for most use cases of a mutex. A guard based `lock()` API should cover most of the remaining use cases of needing these bare primitive operations in a much more safer fashion.

### Rename to `Lock` or similar

A very common name for this type in various codebases is simply `Lock`. This is a decent name because many people know immediately what the purpose of the type is, but the issue is that it doesn't describe _how_ it's implemented. I believe this is a very important aspect of not only this specific type, but of synchronization primitives in general. Understanding that this particular lock is implemented via mutual exclusion conveys to developers who have used something similar in other languages that they cannot have multiple readers and cannot call `lock()` again on the acquired thread for instance. Many languages similar to Swift have opted into a similar design, naming this type mutex instead of a vague lock. In C++ we have `std::mutex` and in Rust we have `Mutex`.

### Include `Mutex` in the default Swift namespace (either in `Swift` or in `_Concurrency`)

This is another intriguing idea because on one hand misusing this type is significantly harder than misusing something like `Atomic`. Generally speaking, we do want folks to reach for this when they just need a simple traditional lock. However, by including it in the default namespace we also unintentionally discouraging folks from reaching for the language features and APIs they we've already built like `async/await`, `actors`, and so much more in this space. Gating the presence of this type behind `import Synchronization` is also an important marker for anyone reading code that the file deals with managing their own synchronization through the use of synchronization primitives such as `Atomic` and `Mutex`.
