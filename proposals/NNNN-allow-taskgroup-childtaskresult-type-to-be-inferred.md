# Allow TaskGroup's ChildTaskResult Type To Be Inferred

* Proposal: [SE-NNNN](NNNN-allow-taskgroup-childtaskresult-type-to-be-inferred.md)
* Author: [Richard L Zarth III](https://github.com/rlziii)
* Review Manager: TBD
* Status: **Awaiting review**
* Implementation: [apple/swift#74517](https://github.com/apple/swift/pull/74517)
* Review: ([pitch](https://forums.swift.org/t/allow-taskgroups-childtaskresult-type-to-be-inferred/72175))

## Introduction

`TaskGroup` and `ThrowingTaskGroup` currently require that one of their two generics (`ChildTaskResult`) always be specified upon creation. Due to improvements in closure parameter/result type inference introduced by [SE-0326](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0326-extending-multi-statement-closure-inference.md) this can be simplified by allowing the compiler to infer both of the generics in most cases.

## Motivation

Currently to create a new task group, there are two generics involved: `ChildTaskResult` and `GroupResult`.  The latter can often be inferred in many cases, but the former must always be supplied as part of either the `withTaskGroup(of:returning:body:)` or `withThrowingTaskGroup(of:returning:body:)` function.  For example:

```swift
let s = withTaskGroup(of: Void.self) { group in
  group.addTask { /* ... */ }
  group.addTask { /* ... */ }
  group.addTask { /* ... */ }

  return "Hello, world!"
}
```

The type of `s` (which is the `GroupResult` type) is correctly inferred as `String`.  However, the return value of the `addTask` closures is not inferred and currently must be supplied to the `of:` parameter of the `withTaskGroup(of:returning:body:)` function (e.g. `Void`).

Note that `withDiscardingTaskGroup(returning:body:)` and `withThrowingDiscardingTaskGroup(returning:body:)` do not have `ChildTaskResult` generics since their child tasks must always be of type `Void`.

## Proposed solution

Adding a default `ChildTaskResult.self` argument for `of childTaskResultType: ChildTaskResult.Type` will allow `withTaskGroup(of:returning:body:)` to infer the type of `ChildTaskResult` in most cases.  The currently signature of `withTaskGroup(of:returning:body:)` looks like:

```swift
public func withTaskGroup<ChildTaskResult, GroupResult>(
    of childTaskResultType: ChildTaskResult.Type,
    returning returnType: GroupResult.Type = GroupResult.self,
    body: (inout TaskGroup<ChildTaskResult>) async -> GroupResult
) async -> GroupResult where ChildTaskResult : Sendable
```

The function signature of `withThrowingTaskGroup(of:returning:body:)` is nearly identical, so only `withTaskGroup(of:returning:body:)` will be used as an example throughout this proposal.

Note that the `GroupResult` generic is inferrable via the `= GroupResult.self` default argument.  This can also be applied to `ChildTaskResult` as of [SE-0326](0326-extending-multi-statement-closure-inference.md).  As in:

```swift
public func withTaskGroup<ChildTaskResult, GroupResult>(
    of childTaskResultType: ChildTaskResult.Type = ChildTaskResult.self, // <- Updated.
    returning returnType: GroupResult.Type = GroupResult.self,
    body: (inout TaskGroup<ChildTaskResult>) async -> GroupResult
) async -> GroupResult where ChildTaskResult : Sendable
```

This allows the original example above to be simplified:

```swift
let s = withTaskGroup { group in
  group.addTask { /* ... */ }
  group.addTask { /* ... */ }
  group.addTask { /* ... */ }

  return "Hello, world!"
}
```

In the above snippet, `ChildTaskResult` is inferred as `Void` and `GroupResult` is inferred as `String`.  Not needing to specify the generics explicitly will simplify the API design for these functions and make it easier for new users of these APIs, as it can currently be confusing to understand the differences between `ChildTaskResult` and `GroupResult` (especially when one or both of those is `Void`).

## Detailed design

Because [SE-0326](0326-extending-multi-statement-closure-inference.md) will look at the first statement to determine the generic for `ChildTaskResult`, it is possible to get a compiler error by creating a task group like so:

```swift
// Expect `ChildTaskResult` to be `Void`...
withTaskGroup { group in // generic parameter 'ChildTaskResult' could not be inferred
    // Since `addTask` wasn't the first statement, this fails to compile.
    group.cancelAll()
    group.addTask { /* ... */ }
}
```

This can be fixed by going back to specifying the generic like before:

```swift
// Expect `ChildTaskResult` to be `Void`...
withTaskGroup(of: Void.self) { group in
    group.cancelAll()
    group.addTask { /* ... */ }
}
```

However, this is a rare case in general since `addTask` is generally the first statement in a task group body.

It is also possible to create a compiler error by returning two different values from a `addTask` closure:

```swift
withTaskGroup { group in
    group.addTask { "Hello, world!" }
    group.addTask { 42 } // Cannot convert value of type 'Int' to closure result type 'String'
}
```

The compiler will already give a good error message here, since the first `addTask` statement is what determined (in this case) that the `ChildTaskResult` generic was set to `String`.  If this needs to be made more clear (instead of being inferred), the user can always specify the generic directly as before:

```swift
withTaskGroup(of: Int.self) { group in
    // Now the error has moved here since the generic was specified up front...
    group.addTask { "Hello, world!" } // Cannot convert value of type 'String' to closure result type 'Int'
    group.addTask { 42 }
}
```

## Source compatibility

Omitting the `of childTaskResultType: ChildTaskResult.Type` parameter for both `withTaskGroup(of:returning:body:)` and `withThrowingTaskGroup(of:returning:body:)` is new, and therefore the inference of `ChildTaskResult` is opt-in and does not break source compatibility.

## ABI compatibility

No ABI impact since adding a default argument value is binary compatible change.

## Implications on adoption

This feature can be freely adopted and un-adopted in source
code with no deployment constraints and without affecting source or ABI
compatibility.

## Future directions

### TaskGroup APIs Without the "with..." Closures

While not possible without more compiler features to enforce the safety of a task group not escaping a context, and having to await all of its results at the end of a "scope..." it is an interesting future direction to explore a `TaskGroup` API that does not need to resort to "with..." methods, like this:

```swift
// Potential long-term direction that might be possible:
func test() async { 
  let group = TaskGroup<Int>()
  group.addTask { ... }
  
  // Going out of scope would have to imply group.awaitAll()...
}
```

If we were to explore such API, the type inference rules would be somewhat different, and a `TaskGroup` would likely be initialized more similarly to a collection: `TaskGroup<Int>`.

This proposal has no impact on this future direction, and can be accepted as is, without precluding future developments in API ergonomics like this.

## Alternatives considered

The main alternative is to do nothing; as in, leave the `withTaskGroup(of:returning:body:)` and `withThrowingTaskGroup(of:returning:body:)` APIs like they are and require the `ChildTaskResult` generic to always be specified.

## Acknowledgments

Thank you to both Konrad Malawski ([@ktoso](https://github.com/ktoso)) and Pavel Yaskevich ([@xedin](https://github.com/xedin)) for confirming the viability of this proposal/idea, and for Konrad Malawski ([@ktoso](https://github.com/ktoso)) for helping to review the proposal and implementation.
