# Implement a rotate algorithm, equivalent to std::rotate() in C++

* Proposal: [SE-0078](0078-rotate-algorithm.md)
* Authors: [Nate Cook](https://github.com/natecook1000), [Sergey Bolshedvorsky](https://github.com/bolshedvorsky)
* Review Manager: [Chris Lattner](https://github.com/lattner)
* Status:  **Returned for revision**
* Previous Revisions: [1](https://github.com/swiftlang/swift-evolution/blob/f5936651da1a08e2335a4991831db61da29aba15/proposals/0078-rotate-algorithm.md), [2](https://github.com/swiftlang/swift-evolution/blob/8d45024ed7baacce94e22080d74f136bebc5c075/proposals/0078-rotate-algorithm.md)
* Review: ([pitch](https://forums.swift.org/t/proposal-implement-a-rotate-algorithm-equivalent-to-std-rotate-in-c/491)) ([review](https://forums.swift.org/t/review-se-0078-implement-a-rotate-algorithm-equivalent-to-std-rotate-in-c/2440)) ([return for revision](https://forums.swift.org/t/review-se-0078-implement-a-rotate-algorithm-equivalent-to-std-rotate-in-c/2440/3)) ([immediate deferral](https://forums.swift.org/t/deferred-se-0078-implement-a-rotate-algorithm-equivalent-to-std-rotate-in-c/2744)) ([return for revision #2](https://forums.swift.org/t/returning-or-rejecting-all-the-deferred-evolution-proposals/60724))

## Introduction

This proposal is to add rotation and in-place reversing methods to Swift's standard library collections.

[Swift-evolution thread](https://forums.swift.org/t/proposal-implement-a-rotate-algorithm-equivalent-to-std-rotate-in-c/491), [Proposal review feedback](https://forums.swift.org/t/review-se-0078-implement-a-rotate-algorithm-equivalent-to-std-rotate-in-c/2440/3)

## Motivation

Rotation is one of the most important algorithms. It is a fundamental tool used in many other algorithms with applications even in GUI programming.

The "rotate" algorithm performs a left rotation on a range of elements. Specifically, it swaps the elements in the range `startIndex..<endIndex` according to a `middle` index in such a way that the element at `middle` becomes the first element of the new range and `middle - 1` becomes the last element. The result of the algorithm is the new index of the element that was originally first in the collection.

```swift
var a = [10, 20, 30, 40, 50, 60, 70]
let i = a.rotate(shiftingToStart: 2)
// a == [30, 40, 50, 60, 70, 10, 20]
// i == 5
```

The index returned from a rotation can be used as the `middle` argument in a second rotation to return the collection to its original state, like this:

```swift
a.rotate(shiftingToStart: i)
// a == [10, 20, 30, 40, 50, 60, 70]
```

There are three different versions of the rotate algorithm, optimized for collections with forward, bidirectional, and random access indices. The complexity of the implementation of these algorithms makes the generic rotate algorithm a perfect candidate for the standard library.

<details> <summary>**Example C++ Implementations**</summary>

**Forward indices** are the simplest and most general type of index and support only one-directional traversal.

The C++ implementation of the rotate algorithm for the `ForwardIterator` (`ForwardIndex` in Swift) may look like this:

```c++
template <ForwardIterator I>
I rotate(I f, I m, I l, std::forward_iterator_tag) {
    if (f == m) return l;
    if (m == l) return f;
    pair<I, I> p = swap_ranges(f, m, m, l);
    while (p.first != m || p.second != l) {
        if (p.second == l) {
            rotate_unguarded(p.first, m, l);
            return p.first;
        }
        f = m;
        m = p.second;
        p = swap_ranges(f, m, m, l);
    }
    return m;
}
```

**Bidirectional indices** are a refinement of forward indices that additionally support reverse traversal.

The C++ implementation of the rotate algorithm for the `BidirectionalIterator` (`BidirectionalIndex` in Swift) may look like this:

```c++
template <BidirectionalIterator I>
I rotate(I f, I m, I l, bidirectional_iterator_tag) {
    reverse(f, m);
    reverse(m, l);
    pair<I, I> p = reverse_until(f, m, l);
    reverse(p.first, p.second);
    if (m == p.first) return p.second;
    return p.first;
}
```

**Random access indices** access to any element in constant time (both far and fast).

The C++ implementation of the rotate algorithm for the `RandomAccessIterator` (`RandomAccessIndex` in Swift) may look like this:

```c++
template <RandomAccessIterator I>
I rotate(I f, I m, I l, std::random_access_iterator_tag) {
    if (f == m) return l;
    if (m == l) return f;
    DifferenceType<I> cycles = gcd(m - f, l - m);
    rotate_transform<I> rotator(f, m, l);
    while (cycles-- > 0) rotate_cycle_from(f + cycles, rotator);
    return rotator.m1;
}
```

</details>


## Proposed solution

The Swift standard library should provide generic implementations of the "rotate" algorithm for all three index types, in both mutating and nonmutating forms. The mutating form is called `rotate(shiftingToStart:)` and rotates the elements of a collection in-place. The nonmutating form of the "rotate" algorithm is called `rotated(shiftingToStart:)` and returns views onto the original collection with the elements rotated, preserving the level of the original collection's index type.

In addition, since the rotate algorithm for bidirectional collections depends on reversing the collection's elements in-place, the standard library should also provide an in-place `reverse()` method to complement the existing nonmutating `reversed()` collection method.

## Detailed design

#### `rotate(shiftingToStart:)` and `rotated(shiftingToStart:)`

The mutating rotation method will be added to the `MutableCollection` protocol requirements, with traversal-specific default implementations. This will allow the correct algorithm to be selected even in a generic context. These methods will have the following declarations:

```swift
protocol MutableCollection {
    // existing declarations
    
    /// Rotates the elements of the collection so that the element
    /// at `middle` ends up first.
    ///
    /// - Returns: The new index of the element that was first
    ///   pre-rotation.
    /// - Complexity: O(*n*)
    @discardableResult
    public mutating func rotate(shiftingToStart middle: Index) -> Index
}

extension MutableCollection {
    /// Rotates the elements of the collection so that the element
    /// at `middle` ends up first.
    ///
    /// - Returns: The new index of the element that was first
    ///   pre-rotation.
    /// - Complexity: O(*n*)
    @discardableResult
    public mutating func rotate(shiftingToStart middle: Index) -> Index
}

extension MutableCollection where Self: BidirectionalCollection {
    /// Rotates the elements of the collection so that the element
    /// at `middle` ends up first.
    ///
    /// - Returns: The new index of the element that was first
    ///   pre-rotation.
    /// - Complexity: O(*n*)
    @discardableResult
    public mutating func rotate(shiftingToStart middle: Index) -> Index
}

extension MutableCollection where Self: RandomAccessCollection {
    /// Reverses the elements of the collection in-place.
    public mutating func reverse()

    /// Rotates the elements of the collection so that the element
    /// at `middle` ends up first.
    ///
    /// - Returns: The new index of the element that was first
    ///   pre-rotation.
    /// - Complexity: O(*n*)
    @discardableResult
    public mutating func rotate(shiftingToStart middle: Index) -> Index
}
```

The nonmutating methods depend on three new specialized types: `RotatedCollection`, `RotatedBidirectionalCollection`, and `RotatedRandomAccessCollection`. These collections present a rotated view onto the elements of a collection without reallocating storage, and thus are able to do so in O(1) time. 

In addition to the standard `Collection` requirements, the rotated collections also define a `shiftedStartIndex` property that holds the rotated position of the base collection's `startIndex`. The three collections can share a single index type, `RotatedCollectionIndex`.

```swift
/// A rotated view of an underlying collection.
public struct RotatedCollection<Base: Collection>: Collection {
    // standard collection innards
    
    /// The shifted position of the base collection's `startIndex`.
    public var shiftedStartIndex: RotatedCollectionIndex<Self>
}

/// A rotated view of an underlying bidirectional collection.
public struct RotatedBidirectionalCollection<Base: BidirectionalCollection>: BidirectionalCollection {
    // standard collection innards
    
    /// The shifted position of the base collection's `startIndex`.
    public var shiftedStartIndex: RotatedCollectionIndex<Self>
}

/// A rotated view of an underlying random-access collection.
public struct RotatedRandomAccessCollection<Base: RandomAccessCollection>: RandomAccessCollection {
    // standard collection innards
    
    /// The shifted position of the base collection's `startIndex`.
    public var shiftedStartIndex: RotatedCollectionIndex<Self>
}

/// The index type for a `RotatedCollection`.
public struct RotatedCollectionIndex<Base: Comparable>: Comparable {
    // standard index innards
}

extension Collection {
    /// Returns a rotated view of the elements of the collection, where the
    /// element at `middle` ends up first.
    ///
    /// - Complexity: O(1)
    func rotated(shiftingToStart middle: Index) -> 
        RotatedCollection<Self>
}

extension BidirectionalCollection {
    /// Returns a rotated view of the elements of the collection, where the
    /// element at `middle` ends up first.
    ///
    /// - Complexity: O(1)
    func rotated(shiftingToStart middle: Index) -> 
        RotatedBidirectionalCollection<Self>
}

extension RandomAccessCollection {
    /// Returns a rotated view of the elements of the collection, where the
    /// element at `middle` ends up first.
    ///
    /// - Complexity: O(1)
    func rotated(shiftingToStart middle: Index) -> 
        RotatedRandomAccessCollection<Self>
}
```

Lazy collections will also be extended with rotate methods that provide lazy rotation:

```swift
extension LazyCollectionProtocol where Index == Elements.Index {
    /// Returns a lazy rotated view of the elements of the collection, where the
    /// element at `middle` ends up first.
    public func rotated(shiftingToStart middle: Elements.Index) ->
        LazyCollection<RotatedCollection<Elements>>
}

extension LazyCollectionProtocol where Index == Elements.Index,
    Self: BidirectionalCollection, Elements: BidirectionalCollection {
    /// Returns a lazy rotated view of the elements of the collection, where the
    /// element at `middle` ends up first.
    public func rotated(shiftingToStart middle: Elements.Index) ->
        LazyBidirectionalCollection<RotatedBidirectionalCollection<Elements>>
}

extension LazyCollectionProtocol where Index == Elements.Index,
    Self: RandomAccessCollection, Elements: RandomAccessCollection {
    /// Returns a lazy rotated view of the elements of the collection, where the
    /// element at `middle` ends up first.
    public func rotated(shiftingToStart middle: Elements.Index) ->
        LazyRandomAccessCollection<RotatedRandomAccessCollection<Elements>>
}
```

Rotation algorithms will be implemented in `stdlib/public/core/CollectionAlgorithms.swift`. The three rotated collection types and collection extensions will be implemented in `stdlib/public/core/Rotate.swift`. Tests will be implemented in `test/1_stdlib/Rotate.swift`.

## `reverse()`

The new mutating `reverse()` method is added in an extension to `MutableCollection where Self: BidirectionalCollection`.

```swift
extension MutableCollection where Self: BidirectionalCollection {
    /// Reverses the elements of the collection in place.
    ///
    /// - Complexity: O(*n*)
    public mutating func reverse()
}
```

## Usage examples

*In-place rotation:*

```swift
var numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9]
numbers.rotate(shiftingToStart: 3)
expectEqual(numbers, [4, 5, 6, 7, 8, 9, 1, 2, 3])

var toMerge = [2, 4, 6, 8, 10, 3, 5, 7, 9]
let i = toMerge[2..<7].rotate(shiftingToStart: 5)
expectEqual(toMerge, [2, 4, 3, 5, 6, 8, 10, 7, 9])
expectEqual(i, 4)
```

*Nonmutating rotation:*

```swift
let numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9]
let r = numbers.rotated(shiftingToStart: 3)
expectEqual(Array(r), [4, 5, 6, 7, 8, 9, 1, 2, 3])
expectEqual(r[r.shiftedStartIndex], 1)
```

*Lazy rotation:*

```swift
let numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9]
let r = numbers.lazy.rotated(shiftingToStart: 3)
expectEqual(r.first!, 4)
```

*Reversing in place:*

```swift
var numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9]
numbers.reverse()
expectEqual(numbers, [9, 8, 7, 6, 5, 4, 3, 2, 1])
numbers[0..<5].reverse()
expectEqual(numbers, [5, 6, 7, 8, 9, 4, 3, 2, 1])
```

## Impact on existing code

The rotation methods are an additive feature that doesn’t impact existing code.

The addition of the mutating `reverse()` method makes it slightly more challenging to migrate from Swift 2, where `reverse()` is the nonmutating method. The renaming of the `sort()`/`sorted()`/`sortInPlace()` methods presents a similar challenge, and the compiler responses in that case (warning when assigning the result of a `Void` function, preventing mutating method calls on immutable instances) will help here as well.

## Alternatives considered

The primary alternative is to not include these methods in the standard library, but the user will need to develop their custom implementation of the rotate algorithms tailored for their needs.

The [first revision of this proposal][rev-1] used `firstFrom` as the parameter name for the `rotate` method and didn't add either `rotate` or `reverse` as protocol requirements. In addition, the `RotatedCollection` type was only used for random-access collections—other collections used the existing `FlattenCollection` instead.

[Another version][rev-2] also made the `reverse()` method choose different algorithms for bidirectional and random-access collections. Without evidence that this would offer a significant performance benefit, this aspect of the proposal has been removed. 


