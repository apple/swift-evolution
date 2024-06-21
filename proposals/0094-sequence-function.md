# Add sequence(first:next:) and sequence(state:next:) to the stdlib

* Proposal: [SE-0094](0094-sequence-function.md)
* Authors: [Lily Ballard](https://github.com/lilyball), [Erica Sadun](https://github.com/erica)
* Review Manager: [Chris Lattner](https://github.com/lattner)
* Status: **Implemented (Swift 3.0)**
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-with-revision-se-0094-add-sequence-initial-next-and-sequence-state-next-to-the-stdlib/2775)
* Bug: [SR-1622](https://bugs.swift.org/browse/SR-1622)
* Previous Revision: [1](https://github.com/swiftlang/swift-evolution/blob/7d220a152a681e28761493c7d9781dd867a04cf7/proposals/0094-sequence-function.md)
* Previous Proposal: [SE-0045](0045-scan-takewhile-dropwhile.md)

## Introduction

This proposal introduces `sequence(first:next:)` and `sequence(state:next:)`,
a pair of global functions that return (potentially-infinite) sequences of lazy
applications of a closure to an initial value or a mutable state.

Swift-evolution thread: 

[Discussion thread topic for SE-0045](https://forums.swift.org/t/proposal-add-scan-takewhile-dropwhile-and-iterate-to-the-stdlib/806/6)

[Initial Discussion](https://forums.swift.org/t/proposal-add-scan-takewhile-dropwhile-and-iterate-to-the-stdlib/806/15)

## Motivation

[SE-0045](0045-scan-takewhile-dropwhile.md), originally proposed `iterate(_:apply:)` (see [SE-0045r1](https://github.com/swiftlang/swift-evolution/blob/dd0a39dd051b11e4460accad5af0e74223533e95/proposals/0045-scan-takewhile-dropwhile.md)), a method that
was subsequently changed to `unfold(_:applying:)`. The proposal was
accepted with modifications. The core team rejected `unfold` based on its naming. As its core utility remains unquestionably high, this proposal re-introduces the method with better, more Swift-appropriate naming.

This function provides the natural counterpart to `reduce` as well as a
replacement for non-striding C-style `for` loops that were removed by the
acceptance of [SE-0007](0007-remove-c-style-for-loops.md), `sequence` can be used to apply generation steps that use non-linear math or
apply non-mathematical operations, as in the following examples:

```swift
for x in sequence(first: 0.1, next: { $0 * 2 }).prefix(while: { $0 < 4 }) {
    // 0.1, 0.2, 0.4, 0.8, ...
}
```

and

```swift
for view in sequence(first: someView, next: { $0.superview }) {
    // someView, someView.superview, someView.superview.superview, ...
}
```

See also: 

* [SE-0007 Remove C-style For Loops](0007-remove-c-style-for-loops.md), 
* [SE-0045](0045-scan-takewhile-dropwhile.md), 
* [SE-0045r1](https://github.com/swiftlang/swift-evolution/blob/b39d653f7e3d5e982b562664343f26c826652291/proposals/0045-scan-takewhile-dropwhile.md), 
* [SE-0045r3](https://github.com/swiftlang/swift-evolution/blob/d709546002e1636a10350d14da84eb9e554c3aac/proposals/0045-scan-takewhile-dropwhile.md)

## Detailed design

The declarations for the proposed functions look like:

```swift
public func sequence<T>(first: T, next: T -> T?) -> UnfoldSequence<T>
public func sequence<T, State>(state: State, next: (inout State) -> T?) -> UnfoldSequence<T>
```

Both functions return potentially-infinite sequences of lazy repeated
applications of a function to an initial value or a state.

The first function, `sequence(first:next:)`, yields the `first` value, followed
by a series of values derived from invoking `next` using the previous value.
The yielded sequence looks like `[first, next(first), next(next(first)), ...` .
This sequence terminates when the `next` function returns `nil`. If the
function never returns `nil` the sequence is infinite. This function is
equivalent to Haskell's [`iterate`][haskell-iterate], however the Swift version
is not always infinite and may terminate.

[haskell-iterate]: http://hackage.haskell.org/package/base-4.8.2.0/docs/Prelude.html#v:iterate

The second function, `sequence(state:next:)`, passes the `state` value as an
`inout` parameter to `next` and yields each subsequent return value. This
function is equivalent to Haskell's [`unfoldr`][haskell-unfoldr], though we've
chosen to make the state an `inout` parameter instead of returning a new state
as this is less likely to produce unwanted Copy on Write (COW) copies of data
structures.

[haskell-unfoldr]: http://hackage.haskell.org/package/base-4.8.2.0/docs/Data-List.html#v:unfoldr

Both functions return a sequence type named `UnfoldSequence`. Existing Swift
naming conventions would call this `SequenceSequence`. Using `UnfoldSequence`
instead resolves the unwarranted redundancy and provides a meaningful reference
to developers familiar with functional programming languages.

## Impact on existing code

None, this change is purely additive.

## Alternatives considered

The natural name for `sequence(state:next:)` is `unfold`. Functional
languages that offer `unfold` pair it with `fold`, which has already been
established in Swift as `reduce`. Renaming `reduce` has already been rejected.
The name `sequence` best describes this function in Swift. `unfold` on its own
is not descriptive and has no meaning to developers not familiar with functional
programming languages.

The function `sequence(first:next:)` can be expressed using
`sequence(state:next:)`. We include it in this proposal due to this form's high
utility. Correctly reimplementing this form in terms of `sequence(state:next:)`
is non-trivial; the simple solution is more eager than it should be.
