# Distinguish between single-tuple and multiple-argument function types

* Proposal: [SE-0110](0110-distinguish-single-tuple-arg.md)
* Authors: Vladimir S., [Austin Zheng](https://github.com/austinzheng)
* Review Manager: [Chris Lattner](https://github.com/lattner)
* Status: **Implemented**
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-se-0110-distinguish-between-single-tuple-and-multiple-argument-function-types/3305), [Additional Commentary](https://forums.swift.org/t/core-team-addressing-the-se-0110-usability-regression-in-swift-4/6147)
* Bug: [SR-2008](https://bugs.swift.org/browse/SR-2008)
* Previous Revision: [Originally Accepted Proposal](https://github.com/swiftlang/swift-evolution/blob/9e44932452e1daead98f2bc2e58711eb489e9751/proposals/0110-distingish-single-tuple-arg.md)

## Introduction

Swift's type system should properly distinguish between functions that take one tuple argument, and functions that take multiple arguments.

Discussion: [pre-proposal](https://forums.swift.org/t/partial-list-of-open-swift-3-design-topics/3094)

## Motivation

Right now, the following is possible:

```swift
let fn1 : (Int, Int) -> Void = { x in
	// The type of x is the tuple (Int, Int).
	// ...
}

let fn2 : (Int, Int) -> Void = { x, y in
	// The type of x is Int, the type of y is Int.
	// ...
}
```

A variable of function type where there exist _n_ parameters (where _n_ > 1) can be assigned a value (whether it be a named function, a closure literal, or other acceptable value) which either takes in _n_ parameters, or one tuple containing _n_ elements. This seems to be an artifact of the tuple splat behavior removed in [SE-0029](0029-remove-implicit-tuple-splat.md).

The current behavior violates the principle of least surprise and weakens type safety, and should be changed.

## Proposed solution

We propose that this behavior should be fixed in the following ways:

* A function type declared with _n_ parameters (_n_ > 1) can only be satisfied by a function value which takes in _n_ parameters. In the above example, only the `fn2` expression would be considered valid.

* To declare a function type with one tuple parameter containing _n_ elements (where _n_ > 1), the function type's argument list must be enclosed by double parentheses:

	```swift
	let a : ((Int, Int, Int)) -> Int = { x in return x.0 + x.1 + x.2 }
	```

	We understand that this may be a departure from the current convention that a set of parentheses enclosing a single object are considered semantically meaningless, but it is the most natural way to differentiate between the two situations described above and would be a clearly-delineated one-time-only exception.

Existing Swift code widely takes advantage of the ability to pass a multi-parameter closure or function value to a higher-order function that operates on tuples, particularly with collection operations:

```
zip([1, 2, 3], [3, 2, 1]).filter(<) // => [(1, 3)]
zip([1, 2, 3], [3, 2, 1]).map(+) // => [4, 4, 4]
```

Without the implicit conversion, this requires invasive changes to explicitly destructure the tuple argument. In order to gain most of the type system benefits of distinguishing single-tuple-argument functions from multiple-argument functions, while maintaining the fluidity of functional code like the above, arguments of type `(T, U, ...) -> V` in call expressions are allowed to be converted to parameters of the corresponding single-tuple parameter type `((T, U, ...)) -> V`, so the two examples above will continue to be accepted.

## Impact on existing code

Minor changes to user code may be required if this proposal is accepted.

## Alternatives considered

Don't make this change.

## Revision history

The [original proposal as reviewed](https://github.com/swiftlang/swift-evolution/blob/9e44932452e1daead98f2bc2e58711eb489e9751/proposals/0110-distingish-single-tuple-arg.md) did not include the special-case conversion from `(T, U, ...) -> V` to `((T, U, ...)) -> V` for function arguments. In response to community feedback, [this conversion was added](https://forums.swift.org/t/core-team-addressing-the-se-0110-usability-regression-in-swift-4/6147) as part of the Core Team's acceptance of the proposal.
