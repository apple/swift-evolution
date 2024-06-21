# Removing `var` from Function Parameters

* Proposal: [SE-0003](0003-remove-var-parameters.md)
* Author: [Ashley Garland](https://github.com/bitjammer)
* Review Manager: [Joe Pamer](https://github.com/jopamer)
* Status: **Implemented (Swift 3.0)**
* Decision Notes: [Rationale](https://forums.swift.org/t/se-0003-removing-var-from-function-parameters-and-pattern-matching/1230)
* Implementation: [apple/swift@8a5ed40](https://github.com/apple/swift/commit/8a5ed405bf1f92ec3fc97fa46e52528d2e8d67d9)

## Note

This proposal underwent some major changes from its original form. See
the end of this document for historical information and why this
proposal changed.

## Introduction

There has been some confusion of semantics when a function parameter is
marked as `inout` compared to `var`. Both give a mutable local copy of a
value but parameters marked `inout` are automatically written back.

Function parameters are immutable by default:

```swift
func foo(i: Int) {
  i += 1 // illegal
}

func foo(var i: Int) {
  i += 1 // OK, but the caller cannot observe this mutation.
}
```

Here, the *local copy* of `x` mutates but the write does not propagate back to
the original value that was passed, so the caller can never observe the change
directly.  For that to happen to value types, you have to mark the parameter
with `inout`:

```swift
func doSomethingWithVar(var i: Int) {
  i = 2 // This will NOT have an effect on the caller's Int that was passed, but i can be modified locally
}

func doSomethingWithInout(inout i: Int) {
  i = 2 // This will have an effect on the caller's Int that was passed.
}

var x = 1
print(x) // 1

doSomethingWithVar(x)
print(x) // 1

doSomethingWithInout(&x)
print(x) // 2
```

## Motivation

Using `var` annotations on function parameters have limited utility,
optimizing for a line of code at the cost of confusion with `inout`,
which has the semantics most people expect. To emphasize the fact these
values are unique copies and don't have the write-back semantics of
`inout`, we should not allow `var` here.

In summary, the problems that motivate this change are:

- `var` is often confused with `inout` in function parameters.
- `var` is often confused to make value types have reference semantics.
- Function parameters are not refutable patterns like in *if-*,
  *while-*, *guard-*, *for-in-*, and *case* statements.

## Design

This is a trivial change to the parser. In Swift 2.2, a deprecation
warning will be emitted while in Swift 3 it will become an error.

## Impact on existing code

As a purely mechanical migration away from these uses of `var`, a temporary
variable can be immediately introduced that shadows the immutable copy in all of
the above uses. For example:

```swift
func foo(i: Int) {
  var i = i
}
```

However, shadowing is not necessarily an ideal fix and may indicate an
anti-pattern. We expect users of Swift to rethink some of their existing
code where these are used but it is not strictly necessary to react to
this language change.

## Alternatives considered

This proposal originally included removal of `var` bindings for all
refutable patterns as well as function parameters.

[Original SE-0003 Proposal](https://github.com/swiftlang/swift-evolution/blob/8cd734260bc60d6d49dbfb48de5632e63bf200cc/proposals/0003-remove-var-parameters-patterns.md)

Removal of `var` from refutable patterns was reconsidered due to the
burden it placed on valid mutation patterns already in use in Swift 2
code. You can view the discussion on the swift-evolution mailing list
here:

[Initial Discussion of Reconsideration](https://forums.swift.org/t/reconsidering-se-0003-removing-var-from-function-parameters-and-pattern-matching/1159)

The rationale for a final conclusion was also sent to the
swift-evolution list, which you can view here:

[Note on Revision of the Proposal](https://forums.swift.org/t/se-0003-removing-var-from-function-parameters-and-pattern-matching/1230)
