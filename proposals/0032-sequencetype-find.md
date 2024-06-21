# Add `first(where:)` method to `Sequence`

* Proposal: [SE-0032](0032-sequencetype-find.md)
* Author: [Lily Ballard](https://github.com/lilyball)
* Review Manager: [Chris Lattner](https://github.com/lattner)
* Status: **Implemented (Swift 3.0)**
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-se-0032-add-find-method-to-sequence/2462)
* Bug: [SR-1519](https://bugs.swift.org/browse/SR-1519)
* Previous Revisions: [1](https://github.com/swiftlang/swift-evolution/blob/d709546002e1636a10350d14da84eb9e554c3aac/proposals/0032-sequencetype-find.md)

## Introduction

Add a new extension method to `Sequence` called `first(where:)` that returns the
found element.

Discussion on swift-evolution started with a proposal with title **Add find method to SequenceType**

Swift-evolution thread: [Proposal: Add function SequenceType.find()](https://forums.swift.org/t/proposal-add-function-sequencetype-find/825)

[Review](https://forums.swift.org/t/review-se-0032-add-find-method-to-sequencetype/2381)

## Motivation

It's often useful to find the first element of a sequence that passes some given
predicate. For `Collection`s you can call `index(of:)` or `index(where:)` and pass the resulting
index back into the `subscript`, but this is a bit awkward. For `Sequence`s,
there's no easy way to do this besides a manual loop that doesn't require
filtering the entire sequence and producing an array.

I have seen people write code like `seq.lazy.filter(predicate).first`, but this
doesn't actually work lazily because `.first` is only a method on
`Collection`, which means the call to `filter()` ends up resolving to the
`Sequence.filter()` that returns an Array instead of to
`LazySequenceProtocol.filter()` that returns a lazy sequence. Users typically aren't
aware of this, which means they end up doing a lot more work than expected.

## Proposed solution

Extend `Sequence` with a method called `first(where:)` that takes a predicate and
returns an optional value of the first element that passes the predicate, if
any.

## Detailed design

Add the following extension to `Sequence`:

```swift
extension Sequence {
  /// Returns the first element where `predicate` returns `true`, or `nil`
  /// if such value is not found.
  public func first(where predicate: @noescape (Self.Iterator.Element) throws -> Bool) rethrows -> Self.Iterator.Element? {
    for elt in self {
      if try predicate(elt) {
        return elt
      }
    }
    return nil
  }
}
```

## Impact on existing code

None, this feature is purely additive.

In theory, we might provide an automatic conversion from
`seq.filter(predicate).first` or `seq.lazy.filter(predicate).first` to
`seq.first(where: predicate)`, although the existing code would continue to
compile just fine.

## Alternatives considered

None
