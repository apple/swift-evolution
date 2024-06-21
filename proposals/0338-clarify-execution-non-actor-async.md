# Clarify the Execution of Non-Actor-Isolated Async Functions

* Proposal: [SE-0338](0338-clarify-execution-non-actor-async.md)
* Author: [John McCall](https://github.com/rjmccall)
* Review Manager: [Doug Gregor](https://github.com/DougGregor)
* Status: **Implemented (Swift 5.7)** ([Decision notes](https://forums.swift.org/t/accepted-se-0338-clarify-the-execution-of-non-actor-isolated-async-functions/54929))

## Introduction

[SE-0306](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0306-actors.md), which introduced actors to Swift, states that `async` functions may be actor-isolated, meaning that they formally run on some actor's executor.  Nothing in either SE-0306 or [SE-0296](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0296-async-await.md) (`async`/`await`) ever specifies where asynchronous functions that *aren't* actor-isolated run.  This proposal clarifies that they do not run on any actor's executor, and it tightens up the rules for [sendability checking](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md) to avoid a potential data race.

## Motivation

It is sometimes important that programmers be able to understand which executor is formally responsible for running a particular piece of code.  A function that does a large amount of computation on an actor's executor will prevent other tasks from making progress on that actor.  The proper isolation of a value may also depend on only accessing it from a particular executor (see note below).  Furthermore, in some situations the current executor has other semantic impacts, such as being "inherited" by tasks created with the `Task` initializer.  Therefore, Swift needs to provide an intuitive and comprehensible rule for which executors are responsible for running which code.

> Note: Swift will enforce the correct isolation of data by default with `Sendable` checking.  However, this will not be fully enabled until code adopts a future language mode (probably Swift 6).  Even under that mode, it will be possible to opt out using `@unsafe Sendable`, making safety the programmer's responsibility.  And even if neither of those caveats were true, it would still be important for programmers to be able to understand the execution rules in order to understand how best to fix an isolation error.

In the current implementation of Swift, `async` functions that aren't actor-isolated never intentionally change the current executor.  That is, whenever execution enters such an `async` function, it will continue on whatever the current executor is, with no specific preference for any particular executor.

To be slightly more precise, we can identify three principle ways that execution can enter an `async` function:

- It's called by some other `async` function.
- It calls some other `async` function which then returns, resuming the caller.
- It needs to suspend for internal reasons (perhaps it uses `withContinuation` or calls a runtime function that suspends), and it is resumed after that suspension.

In the current implementation, calls and returns from actor-isolated functions will continue running on that actor's executor.  As a result, actors are effectively "sticky": once a task switches to an actor's executor, they will remain there until either the task suspends or it needs to run on a different actor.  But if a task suspends within a non-actor-isolated function for a different reason than a call or return, it will generally resume on a non-actor executor.

This rule perhaps makes sense from the perspective of minimizing switches between executors, but it has several unfortunate consequences.  It can lead to unexpected "overhang", where an actor's executor continues to be tied up long after it was last truly needed.  An actor's executor can be surprisingly inherited by tasks created during this overhang, leading to unnecessary serialization and contention for the actor.  It also becomes unclear how to properly isolate data in such a function: some data accesses may be safe because of the executor the function happens to run on dynamically, but it is unlikely that this is guaranteed by the system.  All told, it is a very dynamic rule which interacts poorly with how the rest of concurrency is generally understood, both by Swift programmers and statically by the Swift implementation.

## Proposed solution

`async` functions that are not actor-isolated should formally run on a generic executor associated with no actor.  Such functions will formally switch executors exactly like an actor-isolated function would: on any entry to the function, including calls, returns from calls, and resumption from suspension, they will switch to a generic, non-actor executor.  If they were previously running on some actor's executor, that executor will become free to execute other tasks.

```swift
extension MyActor {
  func update() async {
    // This function is actor-isolated, so formally we switch to the actor.
    // as soon as it is called.

    // Here we call a function which is not actor-isolated.
    let update = await session.readConsistentUpdate()

    // Now we resume executing the function, so formally we switch back to
    // the actor.
    name = update.name
    age = update.age
  }
}

extension MyNetworkSession {
  func readConsistentUpdate() async -> Update {
    // This function is not actor-isolated, so formally we switch to a
    // generic executor when it's called.  So if we happen to be called
    // from an actor-isolated function, we will immediately switch off the
    // actor here.

    // This code runs without any special isolation.

    // Keep calling readUpdate until it returns the same thing twice in a
    // row.  If that never happens in 1000 different calls, just return the
    // last update.  This code is just for explanatory purposes; please don't
    // expect too much from it.
    var update: Update?
    for i in 0..<1000 {
      // Here we make an async call.
      let newUpdate = await readUpdateOnce()

      // Formally, we will switch back to the generic executor after the
      // call, so if we happen to have called an actor-isolated function,
      // we will immediately switch off of the actor here.

      if update == newUpdate { break }
      update = newUpdate
    }
    return update!
  }
}
```

## Detailed design

This proposal changes the semantics of non-actor-isolated `async` functions by specifying that they behave as if they were running on a generic executor not associated with any actor.  Technically, the current rule was never written down, so you could say that this proposal *sets* the semantics of these functions; in practice, though, this is an observable change in behavior.

As a result of this change, the formal executor of an `async` function is always known statically:
- actor-isolated `async` functions always formally run on the actor's executor
- non-actor-isolated `async` functions never formally run on any actor's executor

This change calls for tasks to switch executors at certain points:
- when the function is called
- when a call made by the function returns
- when the function returns from an internal suspension (e.g. due to a continuation)
As usual, these switches are subject to static and dynamic optimization.  These optimizations are the same as are already done with switches to actor executors.

Statically, if a non-actor-isolated async function doesn't do any significant work before returning, suspending, or making an async call, it can simply remain on the current executor and allow its caller, resumer, or callee to make whatever switches it feels are advisable.  This is why this proposal is careful to talk about what executor is *formally* running the task: the actual executor is permitted to be different.  Typically, this difference will not be observable, but there are some exceptions.  For example, if a function makes two consecutive calls to the same actor, it's possible (but not guaranteed) that the actor will not be given up between them, preventing other work from interleaving.  It is outside the scope of this proposal to define what work is "significant".

Dynamically, a switch will not suspend the task if the task is already on an appropriate executor.  Furthermore, some executor changes can be done cheaply without fully suspending the task by giving up the current thread.

### Sendability

The `Sendable` rule for calls to non-actor-isolated `async` functions is currently broken.  This rule is closely tied to the execution semantics of these functions because of the role of sendability checking in proving the absence of data races.  The `Sendable` rule is broken even under the current semantics, but it's arguably even more broken under the proposed rule, so we really do need to fix it as part of this proposal.  (There is an alternative which would make the current rule correct, but it doesn't seem advisable; see "Alternatives Considered".)

It is a basic goal of Swift concurrency that programs should be free of basic data races.  In order to achieve this, we must be able to prove that all uses of certain values and memory are totally ordered.  All of the code that runs on a particular task is totally ordered with respect to itself.  Similarly, all of the code that runs on a particular actor is totally ordered with respect to itself.  So, if we can restrict a value/memory to only be used by a single task or actor, we've proven that all of its uses are totally ordered.  This is the immediate goal of sendability checking: it prevents non-`Sendable` values from being shared between different concurrent contexts and thus potentially being accessed in non-totally-ordered ways.

For the purposes of sendability, the concurrent context of an actor-isolated `async` function is the actor.  An actor can have non-`Sendable` values in its actor-isolated storage.  Actor-isolated functions can read values from that storage into their local state, and similarly they can write values from their local state into actor-isolated storage.  Therefore, such functions must strictly separate their "internal" local state from the "external" local state of the task.  (It would be possible to be more lenient here, but that is outside the scope of this proposal.)

The current sendability rule for `async` calls is that the arguments and results of calls to actor-isolated `async` functions must be `Sendable` unless the callee is known to be isolated to the same actor as the caller.  Unfortunately, no such restriction is placed on calls to non-isolated `async` functions.  That is incorrect under both the current and the proposed execution semantics of such functions because the local state of such functions is not strictly isolated to the actor.

As a result, the following is allowed:

```swift
actor MyActor {
  var isolated: NonSendableValue

  // Imagine that there are two different tasks calling these two
  // functions, and the actor runs the task for `inside_one()` first.

  func inside_one() async {
    await outside(argument: isolated)
  }

  func inside_two() async {
    isolated.operate()
  }
}

// This is a non-actor-isolated async function.
func outside(argument: NonSendableValue) async {
  // Under the current execution semantics, when we resume from this
  // sleep, we will not be on the actor's executor anymore.
  // Under the proposed execution semantics, we will leave the actor's
  // executor even before sleeping.
  await Task.sleep(nanoseconds: 1_000)

  // In either case, this use of the non-Sendable value can now happen
  // concurrently with a use of it on the actor.
  argument.operate()
}
```

The sendability rule for `async` calls must be changed: the arguments and results of *all* `async` calls must be `Sendable` unless:
- the caller and callee are both known to be isolated to the same actor, or
- the caller and callee are both known to be non-actor-isolated.

## Source compatibility

The change to the execution semantics will not break source compatibility.  However, it's possible that recompiling code under this proposal will introduce a data race if that code was previously relying on an actor-isolated value passed as an argument to a non-actor-isolation function only being accessed on the actor's executor.  There should at least be a warning in this case.

The change to the sendability rule may break source compatibility for code that has already adopted concurrency.

In both cases, since Swift's current behavior is clearly undesirable, these seem like necessary changes.  There will not be any attempt to maintain compatibility for existing code.

## Effect on ABI stability

The change in execution semantics does not require additional runtime support; the compiler will simply emit a different pattern of calls.

The change in the sendability rule is compile-time and has no ABI impact.

## Effect on API resilience

This proposal does not introduce a new feature.

It may become more difficult to use `async` APIs that take non-`Sendable` arguments.  Such APIs are rare and usually aren't a good idea.

## Alternatives considered

### Full inheritance of the caller's executor

One alternative to this would be for `async` functions that aren't actor-isolated to "inherit" the executors of their callers.  Essentially, they would record the current executor when they are called, and they would return to that executor whenever they're resumed.

There are several benefits to this approach:

- It can be seen as consistent with the behavior of calls to synchronous functions, which of course "inherit" their executor because they have no ability to change it.

- It significantly improves the overhang problem relative to the current execution semantics.  Overhang would be bounded by the end of the call to an actor function, since upon return the caller would resume its own executor.

- It is the only alternative which would make the current sendability rule for calls to `async` functions correct.

However, it has three significant drawbacks:

- While the overhang would be bounded, it would still cover potentially a large amount of code.  Everything called by an actor-isolated async function would resume to the actor, which could include a large amount of work that really doesn't need to be actor-isolated.  Actors could become heavily contended for artificial and perhaps surprising reasons.

- It would make it difficult to write code that does leave the actor, since the inheritance would be implicit and recursive.  There could be an attribute which avoids the inheritance, but programmers would have to explicitly remember to use it.  This is the opposite of Swift's usual language design approach (e.g. with `mutating` methods); it's better to be less permissive by default so that the places which need stronger guarantees are explicit about it.

- It would substantially impede optimization.  Since the current executor would be semantically observable by inheritance, optimizations that remove executor switches would still have to dynamically record the correct executor that should be inherited.  Since they currently do not do this, and since there is no efficient support in the runtime for doing this, this would come at a substantial runtime cost.

### Initial inheritance of the caller's executor

Another alternative would be to only inherit the executor of the caller for the initial period of execution, from the call to the first suspension.  Later resumptions would resume to a generic, non-actor executor.

This would permit the current sendability rule for arguments, but only if we enforce that the parameters are not used after a suspension in the callee.  This is more flexible, but in ways that are highly likely to prove extremely brittle and limiting; a programmer relying on this flexibility is likely to come to regret it.  It would also still not permit return values to be non-`Sendable`, so the rule would still need changing.

The overhang problem would be further improved relative to full inheritance.  The only real overhang risk would be a function that does a lot of synchronous work before returning or suspending.

Sophisticated programmers might be able to use these semantics to avoid some needless switching.  It is common for `async` functions to begin with an `async` call, but if Swift has trouble analyzing the code that sets up that call, then under the proposed semantics, Swift might be unable to avoid the initial switch.  However, this optimization deficiency equally affects actor-isolated `async` functions, and arguably it ought to have a consistent solution.

This would still significantly inhibit optimization prior to `async` calls, since the current executor would be observable when (e.g.) creating new tasks with the `Task` initializer.  Other situations would be able to optimize freely.

Using a sendability rule that's sensitive to both data flow and control flow seems like a non-starter; it is far too complex for its rather weak benefits.  However, using such a rule is unnecessary, and these execution semantics could instead be combined with the proposed sendability rule.  Non-`Sendable` values that are isolated to the actor would not be shareable with the non-actor-isolated function, and uses of non-`Sendable` values created during the initial segment would be totally ordered by virtue of being isolated to the task.

Overall, while this approach has some benefits over the proposal, it seems better to go with a consistent and wholly static rule for which executor is running any particular `async` function.  Allowing a certain amount of inheritance of executors is an interesting future direction.

## Future directions

### Explicit inheritance of executors

There is still room under this proposal for `async` functions to dynamically inherit their executor from their caller.  It simply needs to be opt-in rather than opt-out.  This does not seem like such an urgent need that it needs to be part of this proposal.

While `reasync` functions have not yet been proposed, it would probably be reasonable for them to inherit executors, since they deliberately blur the lines between synchronous and asynchronous operation.

To allow the caller to use a stronger sendability rule, to avoid over-constraining static optimization of switching, and to support a more efficient ABI, this kind of inheritance should be part of the function signature of the callee.

### Control over executor-switching optimization

By adding potential switches in non-actor-isolated `async` functions, this proposal puts more pressure on Swift's optimizer to eliminate unnecessary switches.  It may be valuable to add a way for programmers to explicitly inform the optimizer that none of the code prior to a suspension is sensitive to the current executor.

### Distinguishing actor-isolated from task-isolated values

As discussed above, uses of a non-`Sendable` value may be totally ordered by being restricted to either a consistent task or a consistent actor.  The current sendability rules do not distinguish between these cases; instead, all non-`Sendable` values in a function are subject to uniform restrictions.  This forces the creation of hard walls between actor-isolated functions and other functions on the same task.  A more expressive sendability rule would distinguish these in actor-isolated `async` functions.  This would significantly decrease the degree to which this proposal infringes on reasonable expressivity in such functions.

The default for parameters and return values should probably be task-isolation rather than actor-isolation, so if we're going to consider this, we need to do it soon for optimal results.

## Acknowledgments

Many people contributed to the development of this proposal, but I'd like to especially thank Kavon Farvardin for his part in the investigation.
