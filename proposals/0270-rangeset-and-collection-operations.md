# Add Collection Operations on Noncontiguous Elements

* Proposal: [SE-0270](0270-rangeset-and-collection-operations.md)
* Authors: [Nate Cook](https://github.com/natecook1000), [Jeremy Schonfeld](https://github.com/jmschonfeld)
* Review Manager: [John McCall](https://github.com/rjmccall)
* Status: **Implemented (Swift 6.0)**
* Implementation: [apple/swift#69766](https://github.com/apple/swift/pull/69766)
* Previous Revisions: [1](https://github.com/swiftlang/swift-evolution/blob/9b5957c00e7483ab8664afe921f989ed1394a666/proposals/0270-rangeset-and-collection-operations.md), [2](https://github.com/swiftlang/swift-evolution/blob/b17d85fcaf38598fd2ea19641d0e9c26c96747ec/proposals/0270-rangeset-and-collection-operations.md), [3](https://github.com/swiftlang/swift-evolution/blob/54d85f65fefce924eb4d5bf10dd633e81f063d11/proposals/0270-rangeset-and-collection-operations.md)
* Review: ([pitch](https://forums.swift.org/t/pitch-add-rangeset-and-related-collection-operations/29961)) ([first review](https://forums.swift.org/t/se-0270-add-collection-operations-on-noncontiguous-elements/30691)) ([first revision](https://forums.swift.org/t/returned-for-revision-se-0270-add-collection-operations-on-noncontiguous-elements/31484)) ([second review](https://forums.swift.org/t/se-0270-review-2-add-collection-operations-on-noncontiguous-elements/31653)) ([second revision](https://forums.swift.org/t/revised-se-0270-add-collection-operations-on-noncontiguous-elements/32840)) ([third review](https://forums.swift.org/t/se-0270-review-3-add-collection-operations-on-noncontiguous-elements/32839)) ([acceptance into preview package](https://forums.swift.org/t/accepted-se-0270-add-collection-operations-on-noncontiguous-elements/33270)) ([fourth pitch](https://forums.swift.org/t/pitch-revision-4-add-collection-operations-on-noncontiguous-elements/68345)) ([fourth review](https://forums.swift.org/t/se-0270-fourth-review-add-collection-operations-on-noncontiguous-elements/68855)) ([acceptance](https://forums.swift.org/t/accepted-se-0270-add-collection-operations-on-noncontiguous-elements/69080))

## Introduction

We can use a `Range<Index>` to refer to a group of consecutive positions in a collection, but the standard library doesn't currently provide a way to refer to discontiguous positions in an arbitrary collection. I propose the addition of a `RangeSet` type that can represent any number of positions, along with collection algorithms that operate on those positions.

## Motivation

There are varied uses for tracking multiple elements in a collection, such as maintaining the selection in a list of items, or refining a filter or search result set after getting more input from a user.

The Foundation data type most suited for this purpose, `IndexSet`, uses integers only, which limits its usefulness to arrays and other random-access collection types. The standard library is missing a collection that can efficiently store ranges of indices, and is missing the operations that you might want to perform with such a collection. These operations themselves can be challenging to implement correctly, and have performance traps as well — see WWDC 2018's [Embracing Algorithms](https://developer.apple.com/videos/wwdc/2018/?id=223) talk for a demonstration. 

## Proposed solution

This proposal adds a `RangeSet` type for representing multiple, noncontiguous ranges, as well as a variety of collection operations for creating and working with range sets.

```swift
var numbers = Array(1...15)

// Find the indices of all the even numbers
let indicesOfEvens = numbers.indices(where: { $0.isMultiple(of: 2) })

// Perform an operation with just the even numbers
let sumOfEvens = numbers[indicesOfEvens].reduce(0, +)
// sumOfEvens == 56

// You can gather the even numbers at the beginning
let rangeOfEvens = numbers.moveSubranges(indicesOfEvens, to: numbers.startIndex)
// numbers == [2, 4, 6, 8, 10, 12, 14, 1, 3, 5, 7, 9, 11, 13, 15]
// numbers[rangeOfEvens] == [2, 4, 6, 8, 10, 12, 14]
```


## Detailed design

`RangeSet` is generic over any `Comparable` type, and supports fast containment checks for ranges and individual values, as well as adding and removing ranges of that type. 

```swift
/// A set of values of any comparable value, represented by ranges.
public struct RangeSet<Bound: Comparable>: Equatable, CustomStringConvertible {
    /// Creates an empty range set.
    public init() {}

    /// Creates a range set containing the values in the given range.
    public init(_ range: Range<Bound>)
    
    /// Creates a range set containing the values in the given ranges.
    public init<S: Sequence>(_ ranges: S) where S.Element == Range<Bound>
    
    /// A Boolean value indicating whether the range set is empty.
    public var isEmpty: Bool { get }
    
    /// Returns a Boolean value indicating whether the given value is
    /// contained by the range set.
    ///
    /// - Complexity: O(log *n*), where *n* is the number of ranges in the
    ///   range set.
    public func contains(_ value: Bound) -> Bool
        
    /// Adds the values represented by the given range to the range set.
    ///
    /// If `range` overlaps or adjoins any existing ranges in the set, the
    /// ranges are merged together. Empty ranges are ignored.
    public mutating func insert(contentsOf range: Range<Bound>)
        
    /// Removes the given range of values from the range set.
    ///
    /// The values represented by `range` are removed from this set. This may
    /// result in one or more ranges being truncated or removed, depending on
    /// the overlap between `range` and the set's existing ranges.
    public mutating func remove(contentsOf range: Range<Bound>)
}

extension RangeSet: Sendable where Bound: Sendable {}
extension RangeSet: Hashable where Bound: Hashable {}
```

#### Conveniences for working with collection indices

Although a range set can represent a set of values of any `Comparable` type, the primary intended use case is to maintain a set of indices into a collection. To streamline this workflow, `RangeSet` includes an additional initializer and methods for inserting and removing individual indices.

```swift
extension RangeSet {
    /// Creates a new range set containing ranges that contain only the
    /// specified indices in the given collection.
    ///
    /// - Parameters:
    ///   - index: The index to include in the range set. `index` must be a
    ///     valid index of `collection` that isn't the collection's `endIndex`.
    ///   - collection: The collection that contains `index`.
    public init<S, C>(_ indices: S, within collection: C)
        where S: Sequence, C: Collection, S.Element == C.Index, C.Index == Bound
      
    /// Inserts a range that contains only the specified index into the range
    /// set.
    ///
    /// - Parameters:
    ///   - index: The index to insert into the range set. `index` must be a
    ///     valid index of `collection` that isn't the collection's `endIndex`.
    ///   - collection: The collection that contains `index`.
    ///
    /// - Returns: `true` if the range set was modified, or `false` if
    ///   the given `index` was already in the range set.
    ///
    /// - Complexity: O(*n*), where *n* is the number of ranges in the range
    ///   set.
    @discardableResult
    public mutating func insert<C>(_ index: Bound, within collection: C) -> Bool
        where C: Collection, C.Index == Bound
    
    /// Removes the range that contains only the specified index from the range
    /// set.
    ///
    /// - Parameters:
    ///   - index: The index to remove from the range set. `index` must be a
    ///     valid index of `collection` that isn't the collection's `endIndex`.
    ///   - collection: The collection that contains `index`.
    ///
    /// - Complexity: O(*n*), where *n* is the number of ranges in the range
    ///   set.
    public mutating func remove<C>(_ index: Bound, within collection: C)
        where C: Collection, C.Index == Bound
}
```


#### Accessing underlying ranges

`RangeSet` provides access to its ranges as a random-access collection via the `ranges` property. 
You can access the individual indices represented by the range set by using it as a subscript parameter to a collection's `indices` property.

```swift
extension RangeSet {
    public struct Ranges: RandomAccessCollection, Equatable, CustomStringConvertible {
        public var startIndex: Int { get }
        public var endIndex: Int { get }
        public subscript(i: Int) -> Range<Bound>
    }
    
    /// A collection of the ranges that make up the range set.
    public var ranges: Ranges { get }
}

extension RangeSet.Ranges: Sendable where Bound: Sendable {}
extension RangeSet.Ranges: Hashable where Bound: Hashable {}
```

The ranges that are exposed through this collection are always in ascending order, are never empty, and never overlap or adjoin. For this reason, inserting an empty range or a range that is subsumed by the ranges already in the set has no effect on the range set. Inserting a range that adjoins an existing range simply extends that range.

```swift
var set = RangeSet([0..<5, 10..<15])
set.insert(contentsOf: 7..<7)
set.insert(contentsOf: 11.<14)
// Array(set.ranges) == [0..<5, 10..<15] 

set.insert(contentsOf: 5..<7)
// Array(set.ranges) == [0..<7, 10..<15]

set.insert(contentsOf: 7..<10)
// Array(set.ranges) == [0..<15]
```

#### `SetAlgebra`-like methods

`RangeSet` implements a subset of the `SetAlgebra` protocol,
for working with more than one `RangeSet`.

```swift
extension RangeSet {
    public func union(_ other: RangeSet<Bound>) -> RangeSet<Bound>
    public mutating func formUnion(_ other: RangeSet<Bound>)

    public func intersection(_ other: RangeSet<Bound>) -> RangeSet<Bound>
    public mutating func formIntersection(_ other: RangeSet<Bound>)

    public func symmetricDifference(_ other: RangeSet<Bound>) -> RangeSet<Bound>
    public mutating func formSymmetricDifference(_ other: RangeSet<Bound>)
    
    public func subtracting(_ other: RangeSet<Bound>) -> RangeSet<Bound>
    public mutating func subtract(_ other: RangeSet<Bound>)
    
    public func isSubset(of other: RangeSet<Bound>) -> Bool
    public func isSuperset(of other: RangeSet<Bound>) -> Bool
    public func isStrictSubset(of other: RangeSet<Bound>) -> Bool
    public func isStrictSuperset(of other: RangeSet<Bound>) -> Bool
    public func isDisjoint(with other: RangeSet<Bound>) -> Bool
}
```

### New `Collection` APIs

#### Finding multiple elements

Akin to the `firstIndex(...)` and `lastIndex(...)` methods, this proposal introduces `indices(where:)` and `indices(of:)` methods that return a range set with the indices of all matching elements in a collection.

```swift
extension Collection {
    /// Returns the indices of all the elements that match the given predicate.
    ///
    /// For example, you can use this method to find all the places that a
    /// vowel occurs in a string.
    ///
    ///     let str = "Fresh cheese in a breeze"
    ///     let vowels: Set<Character> = ["a", "e", "i", "o", "u"]
    ///     let allTheVowels = str.indices(where: { vowels.contains($0) })
    ///     // str[allTheVowels].count == 9
    ///
    /// - Complexity: O(*n*), where *n* is the length of the collection.
    public func indices(where predicate: (Element) throws -> Bool) rethrows
        -> RangeSet<Index>
}

extension Collection where Element: Equatable {
    /// Returns the indices of all the elements that are equal to the given
    /// element.
    ///
    /// For example, you can use this method to find all the places that a
    /// particular letter occurs in a string.
    ///
    ///     let str = "Fresh cheese in a breeze"
    ///     let allTheEs = str.indices(of: "e")
    ///     // str[allTheEs].count == 7
    ///
    /// - Complexity: O(*n*), where *n* is the length of the collection.
    public func indices(of element: Element) -> RangeSet<Index>
}
```

#### Accessing elements via `RangeSet`

When you have a `RangeSet` describing a group of indices for a collection, you can access those elements via subscript. This subscript returns a new `DiscontiguousSlice` type, which couples the collection and range set to provide access.

```swift
extension Collection {
    /// Accesses a view of this collection with the elements at the given
    /// indices.
    ///
    /// - Complexity: O(1)
    public subscript(subranges: RangeSet<Index>) -> DiscontiguousSlice<Self> { get }
}

extension MutableCollection {
    /// Accesses a mutable view of this collection with the elements at the
    /// given indices.
    ///
    /// - Complexity: O(1) to access the elements, O(*m*) to mutate the
    ///   elements at the positions in `subranges`, where *m* is the number of
    ///   elements indicated by `subranges`.
    public subscript(subranges: RangeSet<Index>) -> DiscontiguousSlice<Self> { get set }
}

/// A collection wrapper that provides access to the elements of a collection,
/// indexed by a set of indices.
public struct DiscontiguousSlice<Base: Collection>: Collection, CustomStringConvertible {
    /// The collection that the indexed collection wraps.
    public var base: Base { get set }

    /// The set of index ranges that are available through this indexing
    /// collection.
    public var subranges: RangeSet<Base.Index> { get set }
    
    public typealias SubSequence = Self
    
    /// A position in an `DiscontiguousSlice`.
    public struct Index: Comparable {
        public let base: Base.Index
    }
    
    public var startIndex: Index { get }
    public var endIndex: Index { set }
    public subscript(i: Index) -> Base.Element { get }
    public subscript(bounds: Range<Index>) -> Self { get }
}

extension DiscontiguousSlice: BidirectionalCollection where Base: BidirectionalCollection {}
extension DiscontiguousSlice: Sendable where Base: Sendable, Base.Index: Sendable {}
extension DiscontiguousSlice: Equatable where Base.Element: Equatable {}
extension DiscontiguousSlice: Hashable where Base.Element: Hashable {}

extension DiscontiguousSlice.Index: Sendable where Base.Index: Sendable {}
extension DiscontiguousSlice.Index: Hashable where Base.Index: Hashable {}
```

#### Moving elements

Within a mutable collection, you can move the elements represented by a range set to be in a contiguous range before the element at a specific index, while otherwise preserving element order. When moving elements, other elements slide over to fill gaps left by the elements that move. For that reason, this method returns the new range of the elements that were previously located at the indices within the provided `RangeSet`.

```swift
extension MutableCollection {
    /// Collects the elements at the given indices just before the element at 
    /// the specified index.
    ///
    /// This example finds all the uppercase letters in the array and gathers
    /// them between `"i"` and `"j"`.
    ///
    ///     var letters = Array("ABCdeFGhijkLMNOp")
    ///     let uppercaseRanges = letters.indices(where: { $0.isUppercase })
    ///     let rangeOfUppercase = letters.moveSubranges(uppercaseRanges, to: 10)
    ///     // String(letters) == "dehiABCFGLMNOjkp"
    ///     // rangeOfUppercase == 4..<13
    ///
    /// - Parameters:
    ///   - subranges: The indices of the elements to move.
    ///   - insertionPoint: The index to use as the destination of the elements.
    /// - Returns: The new bounds of the moved elements that were previously located at
    ///	the indices provided by the RangeSet.
    ///
    /// - Complexity: O(*n* log *n*) where *n* is the length of the collection.
    @discardableResult
    public mutating func moveSubranges(
        _ subranges: RangeSet<Index>, to insertionPoint: Index
    ) -> Range<Index>
}
```

#### Removing elements

Within a range-replaceable collection, you can remove the elements represented by a range set. `removeSubranges(_:)` is a new `RangeReplaceableCollection` requirement with a default implementation, along with an overload for collections that also conform to `MutableCollection`.

```swift
extension RangeReplaceableCollection {
    /// Removes the elements at the given indices.
    ///
    /// For example, this code sample finds the indices of all the vowel
    /// characters in the string, and then removes those characters.
    ///
    ///     var str = "The rain in Spain stays mainly in the plain."
    ///     let vowels: Set<Character> = ["a", "e", "i", "o", "u"]
    ///     let vowelIndices = str.indices(where: { vowels.contains($0) })
    ///
    ///     str.removeSubranges(vowelIndices)
    ///     // str == "Th rn n Spn stys mnly n th pln."
    ///
    /// - Parameter subranges: A range set representing the elements to remove.
    ///
    /// - Complexity: O(*n*), where *n* is the length of the collection.
    public mutating func removeSubranges(_ subranges: RangeSet<Index>)
}

extension Collection {
    /// Returns a collection of the elements in this collection that are not
    /// represented by the given range set.
    ///
    /// For example, this code sample finds the indices of all the vowel
    /// characters in the string, and then retrieves a collection that omits
    /// those characters.
    ///
    ///     let str = "The rain in Spain stays mainly in the plain."
    ///     let vowels: Set<Character> = ["a", "e", "i", "o", "u"]
    ///     let vowelIndices = str.indices(where: { vowels.contains($0) })
    ///
    ///     let disemvoweled = str.removingSubranges(vowelIndices)
    ///     print(String(disemvoweled))
    ///     // Prints "Th rn n Spn stys mnly n th pln."
    ///
    /// - Parameter subranges: A range set representing the elements to remove.
    /// - Returns: A collection of the elements that are not in `indices`.
    public func removingSubranges(_ subranges: RangeSet<Index>) -> DiscontiguousSlice<Self>
}
```

## Other considerations

### Source compatibility

These additions preserve source compatibility.

### Effect on ABI stability

This proposal only makes additive changes to the existing ABI.

### Effect on API resilience

All the proposed additions are versioned.

## Alternatives considered

### `SetAlgebra` Conformance

An earlier version of this proposal included `SetAlgebra` conformance when the `Bound` type was `Stridable` with an integer `Stride`. The idea behind this constraint was that with an integer-strideable bound, `RangeSet` could translate an individual value into a `Range` which contained just that value. This translation enabled the implementation of `SetAlgebra` methods like insertion and removal of individual elements.

However, when working with collection indices, there is no guarantee that the correct stride distance is the same for all integer-stridable types. For example, when working with a collection `C` that uses even integers as its indices, removing a single integer from a `RangeSet<C.Index>` could leave an odd number as the start of one of the constitutive ranges:

```swift
let numbers = (0..<20).lazy.filter { $0.isMultiple(of: 2) }
var set = RangeSet(0..<10)
set.remove(4)
// set.ranges == [0..<4, 5..<10]
```

Since `5` is not a valid index of `numbers`, it's an error to use it when subscripting the collection.

One way of avoiding this issue would be to change the internal representation of `RangeSet` to store an enum of single values, ranges, and fully open ranges:

```swift
enum _RangeSetValue {
    case single(Bound)
    case halfOpen(Range<Bound>)
    case fullyOpen(lower: Bound, upper: Bound)
}
```

This implementation would allow correct `SetAlgebra` conformance, but lose the ability to always have a maximally efficient representation of ranges and a single canonical empty state. Through additions and removals, you could end up with a series of a fully-open ranges, none of which actually contain any values in the `Bound` type.

```swift
var set: RangeSet = [1..<4]
set.remove(2)
set.remove(3)
set.remove(4)
// set.ranges == [.fullyOpen(1, 2), .fullyOpen(2, 3), .fullyOpen(3, 4)]
```

### `Elements` view

In an earlier version of the proposal, `RangeSet` included a collection view of the indices, conditionally available when the `Bound` type was strideable with an integer stride. This collection view was removed for the same reasons as covered in the section above. Users can access these indices by using a `RangeSet` as a subscript for the source collection's `indices` property.


### More helpers for working with individual `Collection` indices

In an earlier version of this proposal, `RangeSet` included several methods that accepted range expressions as a parameter, along with the matching collection, so that the `RangeSet` could convert the range expression into a concrete range. These methods have been removed; instead, convert range expressions to concrete ranges before using `RangeSet` methods.

There has been some concern that any APIs that take both an index and a collection represent a violation of concerns. However, these kinds of methods are already well-represented in the standard library (such as `RangeExpression.relative(to:)`), and are necessary for building readable interfaces that work with the design of Swift's collections and indices.


### A predicate-based `gather`

This proposal originally included a predicate-based `gather` method (at a time when `moveSubranges(_:to:)` was named `gather(in:at:)`). This predicate-based method has been removed to allow design work to continue on the larger issue of predicate-based mutating collection methods.

In particular, the issue with this method stems from the fact that a collection user may sometimes want to use a predicate that operates on collection elements (e.g. to check for even elements), and may sometimes want to use a predicate that operates on collection indices (e.g. to test for indices that are part of a known group). For non-mutating methods, this poses no problem, as one can call the predicate-based method on the collection's `indices` property instead. However, with mutating methods, this kind of access poses issues with copy-on-write and exclusivity.

One potential solution is to offer an `(Index, Element) -> Bool` predicate for mutating methods instead of the currently standard `(Element) -> Bool` predicate. This kind of change should be considered for existing mutating methods, like `removeAll(where:)` and `partition(by:)`, as well as any future additions.

### `DiscontiguousSlice` conformance to `MutableCollection`

Previously, this proposal included a `MutableCollection` conformance for `DiscontiguousSlice` where the `Base` collection conformed to `MutableCollection`. However, after further consideration, this conformance was removed. The semantics of how this conformance should behave could be potentially unexpected and in addition to being slightly misleading the conformance would likely be hard to use in the first place. For these reasons, the conformance was removed.

### Other bikeshedding

The review garnered several alternative names for the `RangeSet` type. Some were too tied to the index use case (such as `IndexRangeSet`, `DiscontiguousIndices`, `SomeIndices`), while others didn't represent enough of an obvious improvement to supplant the proposed name (such as `SparseRange`, `DiscontiguousRange`, or `RangeBasedSet`).

There were also a suggestion that the methods for inserting and removing ranges of values should be aligned with the `SetAlgebra` methods `formUnion` and `subtract`. Instead, to keep the `RangeSet` API aligned with user expectations, these operations will keep the names `insert(contentsOf:)` and `remove(contentsOf:)`.

As a result of other feedback, some of the collection operations have been renamed. Instead of `removeAll(in:)` to match `removeAll(where:)`, removing the elements represented by a `RangeSet` is now `removeSubranges(_:)`, as a partner to `removeSubrange(_:)`. Similarly, the `gather(in:at:)` method has been renamed to `moveSubranges(_:to:)`. These names do better at continuing the naming scheme set up by `RangeReplaceableCollection`.

Additionally, we have also chosen the names `indices(of:)` and `indices(where:)` as opposed to the previously suggested names of `ranges(of:)` and `ranges(where:)`. Since this proposal's last revision, `Collection` now has a `ranges(of:)` API that accepts a `RegexComponent` and returns a `[Range]`, so to avoid ambiguity we use the term "indices" here to provide a `RangeSet` value. The compiler does not find these functions ambiguous with the existing `indices` property on `Collection`.

