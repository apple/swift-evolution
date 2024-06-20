# Add a lazy split for Sequence and Collection

* Proposal: [SE-0411]
* Authors: [JUNSHIN](https://github.com/greenthings)
* Review Manager: 
* Status: **Awaiting review**
* Bug: [apple/swift#49240](https://github.com/apple/swift/issues/49240)
* Review:([pitch](https://forums.swift.org/t/pitch-add-a-lazy-split-function-for-sequence/67848))


## Introduction

Swift is a high-performance language known for its efficient operations. When applying methods like map and filter, Swift leverages lazy evaluation. For instance, we can use array.lazy.map() as an example. However, it's worth nothing that the split function currently lacks support for this feature.

## Motivation

As a user of Swift, I believe that lazy evaluation to functions is a concept that enhances Swift's strength by enabling on-demand computation. I was motivated by a desire to contribute to the ecosystem, making Swift an even more powerful and versatile language.

## Proposed solution

In a manner consistent with Swift's conventions, it would be beneficial to consider the possibility of introducing the string.lazy.split() format for string manipulation, aligning it with the established patterns of array.lazy.map() and array.lazy.filter(). This approach would contribute to maintaining consistency and enhancing the comprehensiveness of string handling in the Swift language.

```swift

"a.b.c".lazy.split(separator: ".")

```


## Detailed design


To conform to the LazySequenceProtocol, it is necessary to implement the makeIterator function along with the Iterator type. we can find detailed documentation and information on this protocol at the following link: https://developer.apple.com/documentation/swift/lazysequenceprotocol

```swift
//===----------------------------------------------------------------------===//
// SplitSequence
//===----------------------------------------------------------------------===//

/// A sequence that lazily splits a base sequence into subsequences separated by
/// elements that satisfy the given `whereSeparator` predicate.
///
/// - Note: This type is the result of
///
///     x.split(maxSplits:omittingEmptySubsequences:whereSeparator)
///     x.split(separator:maxSplits:omittingEmptySubsequences)
///
///   where `x` conforms to `LazySequenceProtocol`.

public struct SplitSequence<Base: Sequence> {
  @usableFromInline
  internal let base: Base

  @usableFromInline
  internal let isSeparator: (Base.Element) -> Bool

  @usableFromInline
  internal let maxSplits: Int

  @usableFromInline
  internal let omittingEmptySubsequences: Bool

  @inlinable
  internal init(
    base: Base,
    isSeparator: @escaping (Base.Element) -> Bool,
    maxSplits: Int,
    omittingEmptySubsequences: Bool
  ) {
    self.base = base
    self.isSeparator = isSeparator
    self.maxSplits = maxSplits
    self.omittingEmptySubsequences = omittingEmptySubsequences
  }
}

extension SplitSequence: Sequence {
  public struct Iterator {
    public typealias Element = [Base.Element]

    @usableFromInline
    internal var base: Base.Iterator

    @usableFromInline
    internal let isSeparator: (Base.Element) -> Bool

    @usableFromInline
    internal let maxSplits: Int

    @usableFromInline
    internal let omittingEmptySubsequences: Bool

    /// The number of splits performed.
    @usableFromInline
    internal var splitCount = 0

    /// The number of subsequences returned.
    @usableFromInline
    internal var sequenceLength = 0

    @inlinable
    internal init(
      base: Base.Iterator,
      whereSeparator: @escaping (Base.Element) -> Bool,
      maxSplits: Int,
      omittingEmptySubsequences: Bool
    ) {
      self.base = base
      self.isSeparator = whereSeparator
      self.maxSplits = maxSplits
      self.omittingEmptySubsequences = omittingEmptySubsequences
    }
  }
  
  @inlinable
  public func makeIterator() -> Iterator {
    Iterator(
      base: base.makeIterator(),
      whereSeparator: self.isSeparator,
      maxSplits: self.maxSplits,
      omittingEmptySubsequences: self.omittingEmptySubsequences
    )
  }
}


extension LazySequenceProtocol {

  @inlinable
  public func split(
    maxSplits: Int = Int.max,
    omittingEmptySubsequences: Bool = true,
    whereSeparator isSeparator: @escaping (Element) -> Bool
  ) -> SplitSequence<Elements> {
    precondition(maxSplits >= 0, "Must take zero or more splits")

    return SplitSequence(
      base: elements,
      isSeparator: isSeparator,
      maxSplits: maxSplits,
      omittingEmptySubsequences: omittingEmptySubsequences
    )
  }
}
```
