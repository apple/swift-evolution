# Implement Rotation Algorithms and Types, Part 2

* Proposal: [SE-NNNN](NNNN-rotate-redux.md)
* Authors: [Daryle Walker](https://github.com/CTMacUser),
  [Nate Cook](https://github.com/natecook1000),
  [Sergey Bolshedvorsky](https://github.com/bolshedvorsky)
* Review Manager: TBD
* Status: **Awaiting implementation**

*During the review process, add the following fields as needed:*

* Implementation: [apple/swift#23259](https://github.com/apple/swift/pull/23259)
* Decision Notes: [Rationale](https://forums.swift.org/),
  [Additional Commentary](https://forums.swift.org/)
* Bugs: [SR-125](https://bugs.swift.org/browse/SR-125)
* Previous Revision: [1](https://github.com/apple/swift-evolution/blob/...commit-ID.../proposals/NNNN-filename.md)
* Previous Proposal: [SE-0078](0078-rotate-algorithm.md)

## Introduction

This proposal is an update to SE-0078, providing rotation methods to the
Standard Library's sequences and collections.

Swift-evolution thread: [Discussion thread topic for the renewed pitch](https://forums.swift.org/t/a-new-call-to-rotate-an-update-to-se-0078/21250)

From SE-0078: [Swift-evolution thread](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20151214/002213.html),
[Proposal review feedback](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20160502/016642.html),
and [Decision rationale](https://lists.swift.org/pipermail/swift-evolution-announce/2016-May/000165.html)

## Motivation

**[This section is adapted from SE-0078's Motivation.]**

Rotation is one of the most important algorithms. It is a fundamental tool used
in many other algorithms with applications even in GUI programming.

A rotation algorithm takes a linear sequence along with an offset and direction,
producing a new sequence such that:

1. The new sequence has the same elements in same relative order except at the
   wrap-around point.
2. At the wrap-around point, the last element of the old sequence is followed by
   the first element of the old sequence.  (This doesn't apply when the offset
   was zero.)

If the offset is at least of the length of the sequence, then that offset is
effectively reduced modulo the length.  Since there are a finite number of
(rotation-conforming) rearrangements each rotation amount in one direction has
an equivalent in the other.  A sequence is *left rotated* when any element that
isn't wrapped around goes toward the start of the sequence during rotation when
said rotation broken down to single steps.  And a *right rotation* would then
send non-wrapped-around elements towards the end of the sequence.  (**To-Do**:
Add origins of the rotation directions of "left" and "right.")

Besides an offset and direction, a rotation on a sequence can be described by a
target element and where that element should be after the rotation.

In a more implementation-oriented view, a left rotation algorithm on a range of
collected elements along `startIndex..<endIndex` with a given `middle` index in
that range swaps elements such that the element at `middle` becomes the first
element of the new range and `middle - 1` becomes the last.  The result of the
algorithm is the new index of the element that was originally first in the
collection.

```swift
var a = [10, 20, 30, 40, 50, 60, 70]
let i = a.rotate(toFirst: 2)
// a == [30, 40, 50, 60, 70, 10, 20]
// i == 5
```

The index returned from a left rotation can be used as the `middle` argument in
a second rotation to return the collection to its original state, like this:

```swift
a.rotate(toFirst: i)
// a == [10, 20, 30, 40, 50, 60, 70]
```

There are three different versions of the rotate algorithm, optimized for
collections with forward, bidirectional, and random access indices. The
complexity of the implementation of these algorithms makes the generic rotate
algorithm a perfect candidate for the standard library.

<details> <summary>**Example C++ Implementations**</summary>

**Forward indices** are the simplest and most general type of index and support
only one-directional traversal.

The C++ implementation of the rotate algorithm for the `ForwardIterator`
(corresponding to a `Collection.Index` associated type in Swift) may look like
this:

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

**Bidirectional indices** are a refinement of forward indices that additionally
support reverse traversal.

The C++ implementation of the rotate algorithm for the `BidirectionalIterator`
(corresponding to a `BidirectionalCollection.Index` associated type in Swift)
may look like this:

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

**Random access indices** access to any element in constant time (both far and
fast).

The C++ implementation of the rotate algorithm for the `RandomAccessIterator`
(corresponding to a `RandomAccessCollection.Index` associated type in Swift) may
look like this:

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

In implementations, a right rotation algorithm on a range of collected elements
along `startIndex..<endIndex` with a given `middle` index in that range swaps
elements such that the element at `middle` becomes the last element of the new
range and `middle + 1` becomes the first.  The result of the algorithm is the
new index of the element that was originally first in the collection.

```swift
var b = [10, 20, 30, 40, 50, 60, 70]
let j = b.rotate(toLast: 2)
// b == [40, 50, 60, 70, 10, 20, 30]
// j == 4
```

The index returned from a right rotation can be used as the `middle` argument in
a *left* rotation to return the collection to its original state, like this:

```swift
b.rotate(toFirst: j)
// b == [10, 20, 30, 40, 50, 60, 70]
```

Note that a right rotation on a given middle pivot to be last is just a left
rotation on the immediately-following-the-middle pivot to be first.  If right
rotations are implemented separately from left ones, then right rotations are
equally worthy of being in the Standard Library.  If they are implemented off of
left rotations, then the subtleties involved still make them a candidate for the
Standard Library over letting the users translate themselves.

## Proposed solution

The Swift Standard Library should provide generic implementations of left- and
right-rotations for collections.  Further, collections should have in-place
rotation methods when possible.

The `rotated` methods on collections take an index to a pivot element and return
new collections that are immutable views to the receiver.  Left-rotations use
the pivot element as the returned view's first element in `rotated(toFirst:)`.
Right-rotations use the pivot element as the returned view's last element in
`rotated(toLast:)`.

Also, for collections that allow mutable per-element state, in-place rotation
methods are provided, still taking a given index to the pivot element, but now
returning the index where the original first element got shifted to.  Left
rotations take the pivot element as the new first one in `rotate(toFirst:)`,
while right rotations take the pivot element as the new last one in
`rotate(toLast:)`.

## Detailed design

### Collections, Rotated-View Copies

Like reversed collections, immutable rotated collections can be implemented
with a lazy wrapper `struct`.  (This pattern makes a private copy of the source
collection, so mutations on the wrapped collection won't carry over and stick.
Letting the wrapper use mutating operations could end up missing data for
inexperienced users.)

```swift
/// A collection that presents the elements of its base collection in a
/// shifted (*i.e.* rotated) order.
public struct RotatedCollection<Base: Collection>: Collection {
    public struct Index: Comparable {
        /// The position of the target element, or `nil` as a past-the-end
        /// marker.
        public let base: Base.Index?

        public static func == (lhs: Index, rhs: Index) -> Bool
        public static func < (lhs: Index, rhs: Index) -> Bool
    }

    public var startIndex: Index
    public var endIndex: Index

    public subscript(position: Index) -> Base.Element
    public func index(after i: Index) -> Index
}

extension RotatedCollection.Index: Hashable where Base.Index: Hashable {}

extension RotatedCollection: LazySequenceProtocol where Base: LazySequenceProtocol {}
```

The rotated view allows bi-directional or random-access traversal when the
wrapped collection type does, using conditional conformance.

```swift
extension RotatedCollection: BidirectionalCollection where Base: BidirectionalCollection {
    public func index(before i: Index) -> Index
}

extension RotatedCollection: RandomAccessCollection where Base: RandomAccessCollection {
    public func index(_ i: Index, offsetBy distance: Int) -> Index
    public func distance(from start: Index, to end: Index) -> Int
}
```

Besides using rotations through dereference, users may want to translate
rotation at the index level.  Going from a rotated index to the original
collection's index can be done through the `RotatedCollection.Index.base`
property.  Rotated collections provide a conversion method to get an index for
the rotated collection from a given index to the original collection.

```swift
extension RotatedCollection {
    /// Returns the translation of the given index in `base` to the
    /// corresponding rotated element's index in `self`.
    public func translate(baseIndex: Base.Index) -> Index
}
```

Extension methods on `Collection` vends rotated views of a given receiver and
an index to a given pivot element.

```swift
extension Collection {

    /// Returns the elements of this collection rotated left.
    public func rotated(toFirst middle: Index) -> RotatedCollection<Self>

    /// Returns the elements of this collection rotated right.
    public func rotated(toLast middle: Index) -> RotatedCollection<Self>

}
```

There are overloads for `RotatedCollection` to flatten rotating an
already-rotated collection.

```swift
extension RotatedCollection {

    /// Returns the elements of this collection rotated left, with an
    /// optimized type.
    public func rotated(toFirst middle: Index) -> RotatedCollection {
        // Sample implementation
        return base.rotated(toFirst: middle.base!)
    }

    /// Returns the elements of this collection rotated right, with an
    /// optimized type.
    public func rotated(toLast middle: Index) -> RotatedCollection {
        // Sample implementation
        return base.rotated(toLast: middle.base!)
    }

}
```

For the four methods above, `middle` must be a valid index of its collection
that is **not** `endIndex`.  Using the past-the-end marker is a precondition
failure.

Since `RotatedCollection` doesn't conform to `MutableCollection`, it cannot make
use of that protocol's in-place rotation methods (described in the next
section).  The structure of the type still allows internal rotation, now through
two custom methods.

```swift
extension RotatedCollection {

    /// Rerotates this collection to start at the given index.  If not
    /// `startIndex`, invalidates all outstanding `Index` values.
    public mutating func reseat(asFirst middle: Index)

    /// Rerotates this collection to end at the given index.  If not the
    /// last index before `endIndex`, invalidates all outstanding `Index`
    /// values.
    public mutating func reseat(asLast middle: Index)

}
```

Like the `rotated` methods, `middle` must be some valid index value besides
`endIndex`.  Unlike all `MutableCollection` methods, `reseat` will (usually)
invalidate all outstanding `Index` values.  As a workaround, store the desired
index's `base` value then `translate` after reseating.

### Collections, In-Place

Like the methods in the previous sections, in-place left- and right-rotations
could be implemented solely through extension methods.  However, this would not
permit optimizations for different traversal styles, let alone per-type
optimizations.  To permit optimizations, at least one of the rotation algorithms
needs to be a protocol requirement.

```swift
protocol MutableCollection {

    // Other requirements....

    /// Rotates the elements such that the value at the given index is
    /// now at `startIndex`.
    ///
    /// Passing `startIndex` as `middle` has no effect.
    ///
    /// The method applies a left-rotation, bringing the target element's
    /// value to `startIndex`.
    ///
    ///     var letters = ["A", "B", "C", "D", "E", "F"]
    ///     letters.rotate(toFirst: letters.index(after: letters.startIndex))
    ///     print(String(letters))
    ///     // Prints "BCDEFA"
    ///
    /// - Precondition: `middle` must be a valid index of this collection
    ///   and not equal to `endIndex`.
    ///
    /// - Parameter middle: The index of the element whose value will move
    ///   to `startIndex`.
    ///
    /// - Returns: The index of the element where the value originally at
    ///   `startIndex` went.
    ///
    /// - Postcondition: The new value is a left-rotation of the old;
    ///   `newValue == oldValue[middle...] + oldValue[..<middle]`.
    ///
    /// - Complexity: O(*n*), where *n* is the collection's length.
    @discardableResult
    mutating func rotate(toFirst middle: Index) -> Index

}
```

Since each rotation direction can be implemented in terms of the other, only one
algorithm needs to be a requirement.

```swift
extension MutableCollection {
    /// Rotates the elements such that the value at the given index is now
    /// at the last valid index before `endIndex`.
    ///
    /// Passing the index value before `endIndex` as `middle` has no
    /// effect.
    ///
    /// The method applies a right-rotation, bringing the target element's
    /// value towards `endIndex`.
    ///
    ///     var letters = ["A", "B", "C", "D", "E", "F"]
    ///     letters.rotate(toLast: letters.index(after: letters.startIndex))
    ///     print(String(letters))
    ///     // Prints "CDEFAB"
    ///
    /// - Precondition: `middle` must be a valid index of this collection
    ///   and not equal to `endIndex`.
    ///
    /// - Parameter middle: The index of the element whose value will move
    ///   to the last valid element.
    ///
    /// - Returns: The index of where the value originally at `startIndex`
    ///   went.
    ///
    /// - Postcondition: The new value is a right-rotation of the old;
    ///   `newValue == oldValue[middle>..] + oldValue[...middle]`.
    ///
    /// - Complexity: O(*n*), where *n* is the collection's length.
    @discardableResult
    public mutating func rotate(toLast middle: Index) -> Index
}
```

To preserve API resilience, `rotate(toFirst:)` must have a default
implementation, which is given for forward-only traversal (*i.e.* a collection
that doesn't conform to either `BidirectionalCollection` or
`RandomAccessCollection`).

```swift
extension MutableCollection {
    /// Rotates the elements such that the value at the given index is
    /// now at `startIndex`.
    ///
    /// Passing `startIndex` as `middle` has no effect.
    ///
    /// The method applies a left-rotation, bringing the target element's
    /// value to `startIndex`.
    ///
    ///     var letters = ["A", "B", "C", "D", "E", "F"]
    ///     letters.rotate(toFirst: letters.index(after: letters.startIndex))
    ///     print(String(letters))
    ///     // Prints "BCDEFA"
    ///
    /// - Precondition: `middle` must be a valid index of this collection
    ///   and not equal to `endIndex`.
    ///
    /// - Parameter middle: The index of the element whose value will move
    ///   to `startIndex`.
    ///
    /// - Returns: The index of the element where the value originally at
    ///   `startIndex` went.
    ///
    /// - Postcondition: The new value is a left-rotation of the old;
    ///   `newValue == oldValue[middle...] + oldValue[..<middle]`.
    ///
    /// - Complexity: O(*n*), where *n* is the collection's length.
    @discardableResult
    mutating func rotate(toFirst middle: Index) -> Index
}
```

Since there are optimized rotation algorithms for bi-directional and
random-access collections, there shall be default reimplementations for
`MutableCollection` conforming types that also conform to
`BidirectionalCollection` or `RandomAccessCollection` via extensions.  The
Standard Library may provide similar extensions for any of its (generic)
types that can benefit for further optimizations.

## Source compatibility

The rotation system is an additive feature, so it would only impact existing
code only if the user added members to Standard Library protocols and types with
the same names via extensions.

## Effect on ABI stability

The proposed feature is additive, so it only extends the ABI and should affect
stability.

## Effect on API resilience

Like stability, the resilience impact should be minimal since the changes are
additive.  One change, method `MutableCollection.rotate(toFirst:)`, is a new
protocol requirement.  It has a default implementation, though.

## Alternatives considered

The primary alternative is to not include this feature set into the Standard
Library.  This would require users to supply collection rotation themselves.

This proposal renames some methods from SE-0078 and adds other capabilities.  If
the additional features are cut, then users would have to supply their own code
for right rotations.
