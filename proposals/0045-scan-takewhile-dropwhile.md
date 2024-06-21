# Add prefix(while:) and drop(while:) to the stdlib

* Proposal: [SE-0045](0045-scan-takewhile-dropwhile.md)
* Author: [Lily Ballard](https://github.com/lilyball)
* Review Manager: [Chris Lattner](https://github.com/lattner)
* Status: **Implemented (Swift 3.1)**
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-with-modifications-se-0045-add-scan-prefix-while-drop-while-and-unfold-to-the-stdlib/2466)
* Bug: [SR-1516](https://bugs.swift.org/browse/SR-1516)
* Previous Revisions: [1](https://github.com/swiftlang/swift-evolution/blob/b39d653f7e3d5e982b562664343f26c826652291/proposals/0045-scan-takewhile-dropwhile.md), [2](https://github.com/swiftlang/swift-evolution/blob/baec22a8a5ddaa0407086380da32b5cad2144800/proposals/0045-scan-takewhile-dropwhile.md), [3](https://github.com/swiftlang/swift-evolution/blob/d709546002e1636a10350d14da84eb9e554c3aac/proposals/0045-scan-takewhile-dropwhile.md)

## Introduction

Add 2 new `Sequence` functions `prefix(while:)` and `drop(while:)`, with
overrides as appropriate on `Collection`, `LazySequenceProtocol`, and
`LazyCollectionProtocol`.

Swift-evolution thread:
[Proposal: Add scan, takeWhile, dropWhile, and iterate to the stdlib](https://forums.swift.org/t/proposal-add-scan-takewhile-dropwhile-and-iterate-to-the-stdlib/806)

[Review](https://forums.swift.org/t/review-se-0045-add-scan-prefix-while-drop-while-and-iterate-to-the-stdlib/2382)

## Motivation

The Swift standard library provides many useful sequence manipulators like
`dropFirst(_:)`, `filter(_:)`, etc. but it's missing a few common methods that
are quite useful.

## Proposed solution

Modify the declaration of `Sequence` with two new members:

```swift
protocol Sequence {
  // ...
  /// Returns a subsequence by skipping elements while `predicate` returns
  /// `true` and returning the remainder.
  func drop(while predicate: (Self.Iterator.Element) throws -> Bool) rethrows -> Self.SubSequence
  /// Returns a subsequence containing the initial elements until `predicate`
  /// returns `false` and skipping the remainder.
  func prefix(while predicate: (Self.Iterator.Element) throws -> Bool) rethrows -> Self.SubSequence
}
```

Also provide default implementations on `Sequence` that return `AnySequence`,
and default implementations on `Collection` that return a slice.

`LazySequenceProtocol` and `LazyCollectionProtocol` will also be extended with
implementations of `drop(while:)` and `prefix(while:)` that return lazy
sequence/collection types. Like the lazy `filter(_:)`, `drop(while:)` will
perform the filtering when `startIndex` is accessed.

## Detailed design

In addition to the above declarations, provide default implementations based on
`AnySequence`, similarly to how functions like `dropFirst(_:)` and `prefix(_:)`
are handled:

```swift
extension Sequence where
  SubSequence : Sequence,
  SubSequence.Iterator.Element == Iterator.Element,
  SubSequence.SubSequence == SubSequence {

  public func drop(while predicate: (Self.Iterator.Element) throws -> Bool) rethrows -> AnySequence<Self.Iterator.Element>
  public func prefix(while predicate: (Self.Iterator.Element) throws -> Bool) rethrows -> AnySequence<Self.Iterator.Element>
}
```

These default implementations produce an `AnySequence` that wraps an `Array`
(as the functions must be implemented eagerly so as preserve the convention of
not holding onto a user-provided closure past the function call without the
explicit appearance of `.lazy`).

Provide default implementations on `Collection` as well:

```swift
extension Collection {
  func drop(while predicate: (Self.Iterator.Element) throws -> Bool) rethrows -> Self.SubSequence
  func prefix(while predicate: (Self.Iterator.Element) throws -> Bool) rethrows -> Self.SubSequence
}
```

Also provide overrides as needed on `AnySequence`, `AnyCollection`,
`AnyBidirectionalCollection`, and `AnyRandomAccessCollection`.

Extend `LazySequenceProtocol` with lazy versions of the functions:

```swift
extension LazySequenceProtocol {
  func drop(while predicate: (Self.Iterator.Element) -> Bool) -> LazyDropWhileSequence<Self.Elements>
  func prefix(while predicate: (Self.Iterator.Element) -> Bool) -> LazyPrefixWhileSequence<Self.Elements>
}
```

The types `LazyDropWhileSequence` and `LazyPrefixWhileSequence` conform to
`LazySequenceProtocol`.

Extend `LazyCollectionProtocol` with collection variants for the functions:

```swift
extension LazyCollectionProtocol {
  func drop(while predicate: (Self.Iterator.Element) -> Bool) -> LazyDropWhileCollection<Self.Elements>
  func prefix(while predicate: (Self.Iterator.Element) -> Bool) -> LazyPrefixWhileCollection<Self.Elements>
}
```

The types `LazyDropWhileCollection` and `LazyPrefixWhileCollection` conform to
`LazyCollectionProtocol`.

## Impact on existing code

None, this feature is purely additive.

## Alternatives considered

#### Naming

The names here are likely to cause some bikeshedding. Here are some alternatives
I've heard proposed:

* `suffixFrom(firstElementSatisfying:)` instead of `drop(while:)` – Not only is
  it rather long, it's also focusing on taking a suffix while the actual
  expected usage of the function is focused around skipping elements at the
  start. There's also the potential confusion around whether it's the first
  element from the beginning or the first element from the end (since the term
  "suffix" implies working from the end backwards).
* `skip(while:)` instead of `drop(while:)` – I'm actually partial to this one,
  but we'd need to rename `dropFirst(_:)` as well. The benefit of this is it
  removes the potential confusion around whether the method is mutating.
* `take(while:)` instead of `prefix(while:)` – This was actually the original
  name proposed, and it matches precedent from other languages, but I eventually
  decided that consistency with `prefix(_:)` was desired. However, there is an
  argument to be made that `prefix(while:)` is using the term "prefix" like a
  verb instead of a noun, and the verb form means something different.
* `prefix(to:)` instead of `prefix(while:)` – The name here doesn't make it
  obvious that the argument is a predicate, and this also requires inverting the
  meaning of the predicate which I don't like. The focus of this function is on
  retaining the initial elements that have a desired characteristic, which
  suggests that the predicate should describe the characteristic the desired
  elements have, not the inverse.
* `prefix(having:)` instead of `prefix(while:)` – Reasonable. I chose
  `prefix(while:)` for consistency with `drop(while:)` but `prefix(having:)`
  makes more grammatical sense (since we're using the noun meaning of prefix
  rather than the verb meaning).

## Previous versions

Previous versions of this proposal included global functions `scan(_:combine:)`
and `unfold(_:applying:)` (see [revision 3][rev-3]). This proposal was partially
accepted, with `scan(_:combine:)` rejected on grounds of low utility and
`unfold(_:applying:)` rejected on grounds of poor naming (see [rationale][]).

[rationale]: https://forums.swift.org/t/accepted-with-modifications-se-0045-add-scan-prefix-while-drop-while-and-unfold-to-the-stdlib/2466
