# Revise `partition` Method Signature

* Proposal: [SE-0120](0120-revise-partition-method.md)
* Authors: [Lorenzo Racca](https://github.com/lorenzoracca), [Jeff Hajewski](https://github.com/j-haj), [Nate Cook](https://github.com/natecook1000)
* Review Manager: [Chris Lattner](https://github.com/lattner)
* Status: **Implemented (Swift 3.0)**
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-se-0120-revise-partition-method-signature/3475)
* Bug: [SR-1965](https://bugs.swift.org/browse/SR-1965)
* Previous Revision: [1](https://github.com/swiftlang/swift-evolution/blob/1dcfd35856a6f9c86af2cf7c94a9ab76411739e3/proposals/0120-revise-partition-method.md)

## Introduction

This proposal revises the signature for the collection partition algorithm. Partitioning is a foundational API for sorting and for searching through sorted collections.

- Swift-evolution thread: [Feedback from standard library team](https://forums.swift.org/t/review-se-0074-implementation-of-binary-search-functions/2438/5)
- Swift Bug: [SR-1965](https://bugs.swift.org/browse/SR-1965)

## Motivation

Based on feedback during the review of proposal [SE-0074, Implementation of Binary Search Functions][se-74] and the [list of open issues affecting standard library API stability][list], this is a revised proposal focused only on the existing collection partition method.

The standard library's current `partition` methods, which partition a mutable collection using a binary predicate based on the value of the first element of a collection, are used by the standard library's sorting algorithm but don't offer more general partitioning functionality. A more general partition algorithm using a unary (single-argument) predicate would be more flexible and generally useful.

[se-74]: 0074-binary-search.md
[list]: https://gist.github.com/gribozavr/37e811f12b27c6365fc88e6f9645634d

## Proposed solution

The standard library should replace the two existing `partition` methods with a single method taking a unary predicate called `partition(by:)`. `partition(by:)` rearranges the elements of the collection according to the predicate, such that after partitioning there is a pivot index `p` where no element before `p` satisfies the predicate and every element at and after `p` *does* satisfy the predicate.

```swift
var n = [30, 40, 20, 30, 30, 60, 10]
let p = n.partition(by: { $0 > 30 })
// n == [30, 10, 20, 30, 30, 60, 40]
// p == 5
```

After partitioning is complete, the predicate returns `false` for every element in `n.prefix(upTo: p)` and `true` for every element in `n.suffix(from: p)`.

## Detailed design

`partition(by:)` should be added as a `MutableCollection` requirement with default implementations for mutable and bidirectional mutable collections. Any mutable collection can be partitioned, but the bidirectional algorithm generally performs far fewer assignments.

The proposed APIs are collected here:

```swift
protocol MutableCollection {
    // existing requirements
    
    /// Reorders the elements of the collection such that all the elements 
    /// that match the given predicate are after all the elements that do 
    /// not match the predicate.
    ///
    /// After partitioning a collection, there is a pivot index `p` where 
    /// no element before `p` satisfies the `belongsInSecondPartition` 
    /// predicate and every element at or after `p` satisfies 
    /// `belongsInSecondPartition`.
    /// 
    /// In the following example, an array of numbers is partitioned by a
    /// predicate that matches elements greater than 30.
    ///
    ///     var numbers = [30, 40, 20, 30, 30, 60, 10]
    ///     let p = numbers.partition(by: { $0 > 30 })
    ///     // p == 5
    ///     // numbers == [30, 10, 20, 30, 30, 60, 40]
    ///
    /// The `numbers` array is now arranged in two partitions. The first 
    /// partition, `numbers.prefix(upTo: p)`, is made up of the elements that 
    /// are not greater than 30. The second partition, `numbers.suffix(from: p)`, 
    /// is made up of the elements that *are* greater than 30.
    ///
    ///     let first = numbers.prefix(upTo: p)
    ///     // first == [30, 10, 20, 30, 30]
    ///     let second = numbers.suffix(from: p)
    ///     // second == [60, 40]
    ///
    /// - Parameter belongsInSecondPartition: A predicate used to partition
    ///   the collection. All elements satisfying this predicate are ordered 
    ///   after all elements not satisfying it.
    /// - Returns: The index of the first element in the reordered collection
    ///   that matches `belongsInSecondPartition`. If no elements in the
    ///   collection match `belongsInSecondPartition`, the returned index is
    ///   equal to the collection's `endIndex`.
    ///
    /// - Complexity: O(n)
    mutating func partition(
        by belongsInSecondPartition: @noescape (Iterator.Element) throws-> Bool
    ) rethrows -> Index
}
    
extension MutableCollection {
    mutating func partition(
        by belongsInSecondPartition: @noescape (Iterator.Element) throws-> Bool
    ) rethrows -> Index
}

extension MutableCollection where Self: BidirectionalCollection {
    mutating func partition(
        by belongsInSecondPartition: @noescape (Iterator.Element) throws-> Bool
    ) rethrows -> Index
}
```

A full implementation of the two default implementations can be found [in this gist][gist].

[gist]: https://gist.github.com/natecook1000/70f36608ecd6236552ce0e9f79b98cff

## Impact on existing code

A thorough, though not exhaustive, search of GitHub for the existing `partition` method found no real evidence of its use. The evident uses of a `partition` method were mainly either tests from the Swift project or third-party implementations similar to the one proposed.

Any existing uses of the existing `partition` methods could be flagged or replaced programmatically. The replacement code, on a mutable collection `c`, finding the pivot `p`:

```swift
// old
let p = c.partition()

// new
let p = c.first.flatMap({ first in
    c.partition(by: { $0 >= first })
}) ?? c.startIndex
```

## Alternatives considered

To more closely match the existing API, the `partition(by:)` method could be added only as a default implementation for mutable bidirectional collections. This would unnecessarily limit access to the algorithm for mutable forward collections.

The external parameter label could be `where` instead of `by`. However, using `where` implies that the method finds a pre-existing partition point within in the collection (as in `index(where:)`), rather than modifying the collection to be partitioned by the predicate (as in `sort(by:)`, assuming [SE-0118][] is accepted).

[SE-0118]: 0118-closure-parameter-names-and-labels.md
