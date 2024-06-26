# DiscardingTaskGroups

* Proposal: [SE-0381](0381-task-group-discard-results.md)
* Authors: [Cory Benfield](https://github.com/Lukasa), [Konrad Malawski](https://github.com/ktoso)
* Review Manager: [Doug Gregor](https://github.com/DougGregor)
* Status: **Implemented (Swift 5.9)**
* Implementation: [apple/swift#62361](https://github.com/apple/swift/pull/62361)
* Review: ([pitch](https://forums.swift.org/t/pitch-task-pools/61703)) ([review](https://forums.swift.org/t/se-0381-discardresults-for-taskgroups/62072)) ([acceptance](https://forums.swift.org/t/accepted-se-0381-discardingtaskgroups/62615))

### Introduction

We propose to introduce a new type of structured concurrency task group:  `Discarding[Throwing]TaskGroup`. This type of group is similar to `TaskGroup` however it discards results of its child tasks immediately. It is specialized for potentially never-ending task groups, such as top-level loops of http or other kinds of rpc servers.

## Motivation

Task groups are the building block of structured concurrency, allowing for the Swift runtime to relate groups of tasks together. This enables powerful features such as automatic cancellation propagation, correctly propagating errors, and ensuring well-defined lifetimes, as well as providing diagnostic information to programming tools.

The version of Task Groups introduced in [SE-0304](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md) provides all of these features. However, it also provides the ability to propagate return values to the user of the task group. This capability provides an unexpected limitation in some use-cases.

As users of Task Groups are able to retrieve the return values of child tasks, it implicitly follows that the Task Group preserves at least the `Result` of any completed child task. As a practical matter, the task group actually preseves the entire `Task` object. This data is preserved until the user consumes it via one of the Task Group consumption APIs, whether that is `next()` or by iterating the Task Group.

The result of this is that Task Groups are ill-suited to running for a potentially unbounded amount of time. An example of such a use-case is managing connections accepted from a listening socket. A simplified example of such a workload might be:

```swift
try await withThrowingTaskGroup(of: Void.self) { group in
    while let newConnection = try await listeningSocket.accept() {
        group.addTask {
            handleConnection(newConnection)
        }
    }
}
```

As written, this task group will leak all the child `Task` objects until the listening socket either terminates or throws. If this was written for a long-running server, it is entirely possible for this Task Group to survive for a period of days, leaking thousands of Task objects. For stable servers, this will eventually drive the process into memory exhaustion, forcing it to be killed by the OS.

The current implementation of Task Groups do not provide a practical way to avoid this issue. Task Groups are (correctly) not `Sendable`, so neither the consumption of completed `Task` results nor the submission of new work can be moved to a separate `Task`.

The most natural attempt to avoid this unbounded memory consumption would be to attempt to occasionally purge the completed task results. An example might be:

```swift
try await withThrowingTaskGroup(of: Void.self) { group in
    while let newConnection = try await listeningSocket.accept() {
        group.addTask {
            handleConnection(newConnection)
        }
        try await group.next()
    }
}
```

Unfortunately, all of the methods for attempting to pop the queue of completed `Task`s will suspend if all currently live child `Task`s are executing. This means that the above pattern (or any similar pattern) is at risk of occasional livelocks, where pending connections could be accepted, but the `Task` is blocked waiting for existing work to complete.

There is only one design pattern to avoid this issue, which involves forcibly bounding the maximum concurrency of the Task Group. This pattern looks something like the below:

```swift
try await withThrowingTaskGroup(of: Void.self) { group in
    // Fill the task group up to maxConcurrency
    for _ in 0..<maxConcurrency {
	    guard let newConnection = try await listeningSocket.accept() else {
	        break
	    }
	    
	    group.addTask { handleConnection(newConnection) }
	}
	
	// Now follow a one-in-one-out pattern
	while true {
	    _ = try await group.next()
	    guard let newConnection = try await listeningSocket.accept() else {
	        break
	    }
	    group.addTask { handleConnection(newConnection) }
	}
}
```

While this is workable, it forces users to determine a value for `maxConcurrency`. This is frequently very hard to decide _a priori_. Practically users tend to guess, and either get a value far too large (causing memory to be wasted) or far too low (causing the system to be underutilized). While there is value in developing a strategy for bounding the maximum concurrency of a `TaskGroup`, that problem is sufficiently complex to be worth its own separate discussion.

## Proposed Solution

We propose adding new `DiscardingTaskGroup` and `ThrowingDiscardingTaskGroup` group types (obtained by `withDiscardingTaskGroup` and `withThrowingDiscardingTaskGroup`). These groups, are somewhat similar to the normal `TaskGroups` implementations, however they differ in the following important ways:

1. `[Throwing]DiscardingTaskGroup` automatically cleans up its child `Task`s when those `Task`s complete.
2. `[Throwing]DiscardingTaskGroup` do not have a `next()` method, nor do they conform to `AsyncSequence`.

These group types are _not_ parameterized with the `ChildTaskResult`, and it is assumed to be `Void`, because as the name implies, they are always _discarding the results_ of their child tasks.

Cancellation and error propagation of `[Throwing]DiscardingTaskGroup` works the same way one comes to expect a task group to behave, however due to the inability to explicitly use `next()` to "re-throw" a child task error, the discarding task group types must handle this behavior implicitly by re-throwing the _first_ encountered error and cancelling the group.

`[Throwing]DiscardingTaskGroup` is a structured concurrency primitive, the same way as `[Throwing]TaskGroup` and _must_ automatically await all submitted tasks before the body of the `[try] await with[Throwing]DiscardingTaskGroup { body }` returns.

### API Surface

```swift
public func withDiscardingTaskGroup<GroupResult>(
  returning returnType: GroupResult.Type = GroupResult.self,
  body: (inout DiscardingTaskGroup) async -> GroupResult
) async -> GroupResult { ... } 

public func withThrowingDiscardingTaskGroup<GroupResult>(
  returning returnType: GroupResult.Type = GroupResult.self,
  body: (inout ThrowingDiscardingTaskGroup<Error>) async throws -> GroupResult
) async throws -> GroupResult { ... }
```

And the types themselfes, mostly mirroring the APIs of `TaskGroup`, except that they're missing `next()` and related functionality:

```swift
public struct DiscardingTaskGroup {
  
  public mutating func addTask(
    priority: TaskPriority? = nil,
    operation: @Sendable @escaping () async -> Void
  )

  public mutating func addTaskUnlessCancelled(
    priority: TaskPriority? = nil,
    operation: @Sendable @escaping () async -> Void
  ) -> Bool 

  public var isEmpty: Bool
  
  public func cancelAll()
  public var isCancelled: Bool
}
@available(*, unavailable)
extension DiscardingTaskGroup: Sendable { }

public struct ThrowingDiscardingTaskGroup<Failure: Error> {
  
  public mutating func addTask(
    priority: TaskPriority? = nil,
    operation: @Sendable @escaping () async throws -> Void
  )

  public mutating func addTaskUnlessCancelled(
    priority: TaskPriority? = nil,
    operation: @Sendable @escaping () async throws -> Void
  ) -> Bool 

  public var isEmpty: Bool
  
  public func cancelAll()
  public var isCancelled: Bool
}
@available(*, unavailable)
extension DiscardingThrowingTaskGroup: Sendable { }
```

## Detailed Design

### Discarding results

As indicated by the name a `[Throwing]DiscardingTaskGroup` will discard results of its child tasks _immediately_ and release the child task that produced the result. This allows for efficient and "running forever" request accepting loops such as HTTP or RPC servers.

Specifically, the first example shown in the Motivation section of this proposal, _is_ safe to be expressed using a discarding task group, as follows:

```swift
// GOOD, no leaks!
try await withThrowingDiscardingTaskGroup() { group in
    while let newConnection = try await listeningSocket.accept() {
        group.addTask {
            handleConnection(newConnection)
        }
    }
}
```

This code–unlike the `withThrowingTaskGroup` version shown earlier–does not leak tasks and therefore is safe and the recommended way to express such handler loops. 

### Error propagation and group cancellation

Throwing task groups rely on the `next()` (or `waitForAll()`) being throwing and end users consuming the child tasks this way in order to surface any error that the child tasks may have thrown. It is possible for a `ThrowingTaskGroup` to explicitly collect results (and failures), and react to them, like this:

```swift
try await withThrowingTaskGroup(of: Void.self) { group in 
  group.addTask { try boom() }
  group.addTask { try boom() }
  group.addTask { try boom() }
  
  try await group.next() // re-throws whichever error happened first
} // since body threw, the group and remaining tasks are immediately cancelled 
```

The above snippet illustrates a simple case of the error propagation out of a child task, through `try await next()` (or `try await group.waitForAll()`) out of the `withThrowingTaskGroup` closure body. As soon as an error is thrown out of the closure body, the group cancels itself and all remaining tasks implicitly, finally proceeding to await all the pending tasks.

This pattern is not possible with `ThrowingDiscardingTaskGroup` because the the results collecting methods are not available on discarding groups. In order to properly support the common use-case of discarding groups, the failure of a single task, should implicitly and _immediately_ cancel the group and all of its siblings.

This can be seen as the implicit immediate consumption of the child tasks inspecting the task for failures, and "re-throwing" the failure automatically. The error is then also re-thrown out of the `withThrowingDiscardingTaskGroup` method, like this:

```swift
try await withThrowingDiscardingTaskGroup() { group in 
  group.addTask { try boom(1) }
  group.addTask { try boom(2) }
  group.addTask { try boom(3) }
  // whichever failure happened first, is collected, stored, and re-thrown out of the method when exiting.
}
```

In other words, discarding task groups follow the "one for all, and all for one" pattern for failure handling. A failure of a single child task, _immediately_ causes cancellation of the group and its siblings. 

Preventing this behavior can be done in two ways:

- using `withDiscardingTaskGroup`, since the child tasks won't be allowed to throw, and must handle their errors in some other way,
- including normal `do {} catch {}` error handling logic inside the child-tasks, which only re-throws.

We feel this is the right approach for this structured concurrency primitive, as we should be leaning on normal swift code patterns, rather than introduce special one-off ways to handle and deal with errors. Although, if it were necessary, we could introduce a "failure reducer" in the future.

## Alternatives Considered

### Introducing new "TaskPool" type (initial pitch)

The [original pitch](https://forums.swift.org/t/pitch-task-pools/61703) introduced two new types, `TaskPool` and `ThrowingTaskPool`. These types were introduced in order to expose at the type system level the inability to iterate the pool for new tasks. This would avoid the `next()` behaviour introduced in this pitch, where `next()` always returns `nil`. This was judged a worthwhile change to justify introducing new types.

Several reviewers of the pitch felt that this was not a sufficiently useful capability to justify the introduction of the new types, and that the pitched behaviour more properly belonged as a "mode" of operation on `TaskGroup`. In line with that feedback, this proposal has moved to using the `discardResults` option.

### Extending [Throwing]TaskGroup with discardResults flag

After feedback on the the initial pitch, we attempted to avoid introducing a new type, and instead handle it using a `discardResults: Bool` flag on `with[Throwing]TaskGroup()` this was fairly problematic because:

- the group would have the `next()` method as well as `AsyncSequence` conformance present, but non-functional, i.e. always returning `nil` from `next()` which could lead to subtle bugs and confusion.
- we'd end up constraining this new option only to child task result types of `Void`, making access to this functionality a bit hard to discover

The group would also have very different implicit cancellation behavior, ultimately leading us to conclude during the Swift Evolution review that these two behaviors should not be conflated into one type.

### Alternate Error throwing behaviour

The pitch proposes that `ThrowingDiscardingTaskGroup` will throw only the _first_ error thrown by a child `Task`. This means that all subsequent errors will be discarded, which is an unfortunate loss of information. Two alternative behaviours could be chosen: we could not provide `ThrowingDiscardingTaskGroup` at all, or we could throw an aggregate error that contains *all* errors thrown by the child `Task`s.

Not allowing offering `ThrowingDiscardingTaskGroup` at all is a substantial ergonomic headache. Automatic error propagation is one of the great features of structured concurrency, and not being able to use it in servers or other long-running processes is an unnecessary limitation, especially as it's not particularly technically challenging to propagate errors. For this reason, we do not think it's wise to omit `discardResults` on `ThrowingDiscardingTaskGroup`.

The other alternative is to throw an aggregate error. This would require that `ThrowingDiscardingTaskGroup` persist all (or almost all) errors thrown by child tasks and merge them together into a single error `struct` that is thrown. This idea is a mixed bag.

The main advantage of throwing an aggregate error is that no information is lost. Programs can compute on all errors that were thrown, and at the very least can log or provide other metrics based on those errors. Avoiding data loss in this way is valuable, and gives programmers more flexibility.

Throwing an aggregate error has two principal disadvantages. The first is that aggregate errors do not behave gracefully in `catch` statements. If a child task has thrown `MyModuleError`, programmers would like to write `catch MyModuleError` in order to handle it. Aggregate errors break this situation, even if only one error is thrown: programmers have to write `catch let error = error as? MyAggregateError where error.baseErrors.contains(where: { $0 is MyModuleError })`, or something else equally painful.

The other main disadvantage is the storage bloat from `CancellationError`. The first thrown error will auto-cancel all child `Task`s. This is great, but that cancellation will likely manifest in as series of `CancellationError`s, which will presumably bubble to the top and be handled by the `ThrowingDiscardingTaskGroup`. This means that a `ThrowingDiscardingTaskGroup` will likely store a substantial collection of errors, where all but the first are `CancellationError`. This is a substantial regression in convenience for the mainline case, with additional costs in storage, without providing any more meaningful information.

For these reasons we've chosen the middle behaviour, where only one error is thrown. We think there is merit in throwing an aggregate error, however, and we'd like community feedback on this alternative.

### Child Task for reaping

An alternative would be to have Task Group spin up a child `Task` that can be used to consume tasks from the group. The API surface would look something like this:

```swift
withTaskGroupWithChildTask(of: Void.self) { group in
    group.addTask {
        handleConnection(newConnection)
    }
}
consumer: { group in
    for task in group { }
}
```

The advantage of this variant is that it is substantially more flexible, and allows non-`Void`-returning tasks. The downside of this variant is that it muddies the water on the question of whether Task Groups are `Sendable` (requiring a specific-exemption for this use-case) and forces users to understand the lifetime of a pair of different closures.

## Future Directions

### Error Handling

A number of concerns were raised during the pitch process that the "throw the first error only" pattern may be insufficiently flexible. Community members were particularly interested in having some sort of error filter function that could be used to filter, accumulate, or discard errors as needed.

The proposers feel that introducing this API surface in the first version of this feature adds significant complexity to this type. This requires us to be confident that the API surface proposed is going to serve the necessary use-cases, without adding unnecessary cognitive load. It's also not entirely clear where the line is between features that can be handled using `try`/`catch` and features that require a new error filter function.

As a result, the proposal authors have elected to defer implementing anything here until there are real-world examples to generalise from. Having some sort of error filter is likely to be valuable, and the implementation will preserve the capability to implement such a function, but for now the proposal is going to be kept relatively small.
