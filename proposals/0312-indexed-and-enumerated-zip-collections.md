# Add `indexed()` and `Collection` conformances for `enumerated()` and `zip(_:_:)`

* Proposal: [SE-0312](0312-indexed-and-enumerated-zip-collections.md)
* Author: [Tim Vermeulen](https://github.com/timvermeulen)
* Review Manager: [Ben Cohen](https://github.com/airspeedswift)
* Status: **Returned for revision**
* Implementation: [apple/swift#36851](https://github.com/apple/swift/pull/36851)

## Introduction
This proposal aims to fix the lack of `Collection` conformance of the sequences returned by `zip(_:_:)` and `enumerated()`, preventing them from being used in a context that requires a `Collection`. Also included is the addition of the `indexed()` method on `Collection` as a more ergonomic, efficient, and correct alternative to `c.enumerated()` and `zip(c.indices, c)`.

Swift-evolution thread: [Pitch](https://forums.swift.org/t/pitch-add-indexed-and-collection-conformances-for-enumerated-and-zip/47288)

## Motivation
Currently, the `Zip2Sequence` and `EnumeratedSequence` types conform to `Sequence`, but not to any of the collection protocols. Adding these conformances was impossible before [SE-0234 Remove `Sequence.SubSequence`](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0234-remove-sequence-subsequence.md), and would have been an ABI breaking change before the language allowed `@available` annotations on protocol conformances ([PR](https://github.com/apple/swift/pull/34651)). Now we can add them!

Conformance to the collection protocols can be beneficial in a variety of ways, for example:
* `(1000..<2000).enumerated().dropFirst(500)` becomes a constant time operation.
* `zip("abc", [1, 2, 3]).reversed()` will return a `ReversedCollection` rather than allocating a new array.
* SwiftUI’s `List` and `ForEach` views will be able to directly take an enumerated or zipped collection as their data.

This proposal also includes the addition of the `indexed()` method (which can already be found in the [Swift Algorithms](https://github.com/apple/swift-algorithms) package) as an alternative for many use cases of `zip(_:_:)` and `enumerated()`. When the goal is to iterate over a collection’s elements and indices at the same time, `enumerated()` is often inadequate because it provides an offset, not a true index. For many collections this integer offset is different from the `Index` type, and in the case of `ArraySlice` in particular this offset is a common source of bugs when the slice’s `startIndex` isn’t `0`. `zip(c.indices, c)` solves these problems, but it is less ergonomic than `indexed()` and potentially less performant when traversing the indices of a collection is computationally expensive.

## Detailed design
Conditionally conform `Zip2Sequence` to `Collection` and `BidirectionalCollection`.

> **Note**: OS version 9999 is a placeholder and will be replaced with whatever actual OS versions this functionality will be introduced in.

```swift
@available(macOS 9999, iOS 9999, watchOS 9999, tvOS 9999, *)
extension Zip2Sequence: Collection
  where Sequence1: Collection, Sequence2: Collection
{
  // ...
}

@available(macOS 9999, iOS 9999, watchOS 9999, tvOS 9999, *)
extension Zip2Sequence: BidirectionalCollection
  where Sequence1: BidirectionalCollection, Sequence2: BidirectionalCollection
{
  // ...
}
```

Add a `zip(_:_:)` overload that returns a random-access collection when given two random-access collections.

```swift
@available(macOS 9999, iOS 9999, watchOS 9999, tvOS 9999, *)
public func zip<Base1: RandomAccessCollection, Base2: RandomAccessCollection>(
  _ base1: Base1, _ base2: Base2
) -> Zip2RandomAccessCollection<Base1, Base2> {
  Zip2RandomAccessCollection(base1, base2)
}

@available(macOS 9999, iOS 9999, watchOS 9999, tvOS 9999, *)
public struct Zip2RandomAccessCollection<Base1, Base2>
  where Base1: RandomAccessCollection, Base2: RandomAccessCollection
{
  // ...
}

@available(macOS 9999, iOS 9999, watchOS 9999, tvOS 9999, *)
extension Zip2RandomAccessCollection: RandomAccessCollection {
  // ...
}
```

Conditionally conform `EnumeratedSequence` to `Collection`, `BidirectionalCollection`, `RandomAccessCollection`, and `LazyCollectionProtocol`.

```swift
@available(macOS 9999, iOS 9999, watchOS 9999, tvOS 9999, *)
extension EnumeratedSequence: Collection where Base: Collection {
  // ...
}

@available(macOS 9999, iOS 9999, watchOS 9999, tvOS 9999, *)
extension EnumeratedSequence: BidirectionalCollection
  where Base: BidirectionalCollection
{
  // ...
}

@available(macOS 9999, iOS 9999, watchOS 9999, tvOS 9999, *)
extension EnumeratedSequence: RandomAccessCollection
  where Base: RandomAccessCollection {}

@available(macOS 9999, iOS 9999, watchOS 9999, tvOS 9999, *)
extension EnumeratedSequence: LazySequenceProtocol
  where Base: LazySequenceProtocol {}

@available(macOS 9999, iOS 9999, watchOS 9999, tvOS 9999, *)
extension EnumeratedSequence: LazyCollectionProtocol
  where Base: LazyCollectionProtocol {}
```

Add an `indexed()` method to `Collection` that returns a collection over (index, element) pairs of the original collection.

```swift
extension Collection {
  @available(macOS 9999, iOS 9999, watchOS 9999, tvOS 9999, *)
  public func indexed() -> IndexedCollection<Self> {
    Indexed(_base: self)
  }
}

@available(macOS 9999, iOS 9999, watchOS 9999, tvOS 9999, *)
public struct IndexedCollection<Base: Collection> {
  // ...
}

@available(macOS 9999, iOS 9999, watchOS 9999, tvOS 9999, *)
extension IndexedCollection: Collection {
  // ...
}

@available(macOS 9999, iOS 9999, watchOS 9999, tvOS 9999, *)
extension IndexedCollection: BidirectionalCollection where Base: BidirectionalCollection {
  // ...
}

@available(macOS 9999, iOS 9999, watchOS 9999, tvOS 9999, *)
extension IndexedCollection: RandomAccessCollection where Base: RandomAccessCollection {}

@available(macOS 9999, iOS 9999, watchOS 9999, tvOS 9999, *)
extension IndexedCollection: LazySequenceProtocol where Base: LazySequenceProtocol {}

@available(macOS 9999, iOS 9999, watchOS 9999, tvOS 9999, *)
extension IndexedCollection: LazyCollectionProtocol where Base: LazyCollectionProtocol {}
```

## Source compatibility
Adding `LazySequenceProtocol` conformance for `EnumeratedSequence` is a breaking change for code that relies on the `enumerated()` method currently not propagating `LazySequenceProtocol` conformance in a lazy chain:

```swift
extension Sequence {
  func everyOther_v1() -> [Element] {
    let x = self.lazy
      .enumerated()
      .filter { $0.offset.isMultiple(of: 2) }
      .map(\.element)
    
    // error: Cannot convert return expression of type 'LazyMapSequence<...>' to return type '[Self.Element]'
    return x
  }
  
  func everyOther_v2() -> [Element] {
    // will keep working, the eager overload of `map` is picked
    return self.lazy
      .enumerated()
      .filter { $0.offset.isMultiple(of: 2) }
      .map(\.element)
  }
}
```

All protocol conformances of an existing type to an existing protocol are potentially source breaking because users could have added the exact same conformances themselves. However, given that `Zip2Sequence` and `EnumeratedSequence` do not expose their underlying sequences, there is no reasonable way anyone could have conformed either type to `Collection` themselves. The only sensible conformance that could conflict with one of the conformances added in this proposal is the conformance of `EnumeratedSequence` to `LazySequenceProtocol`.

## Effect on ABI stability
This proposal does not affect ABI stability.

## Alternatives considered
#### Don’t add `LazyCollectionProtocol` conformance for `EnumeratedSequence` for the sake of source compatibility.
We consider it a bug that `enumerated()` currently does not propagate laziness in a lazy chain.

#### Keep `EnumeratedSequence` the way it is and add an `enumerated()` overload to `Collection` that returns a `Zip2Sequence<Range<Int>, Self>`.
This is tempting because `enumerated()` is little more than `zip(0..., self)`, but this would cause an unacceptable amount of source breakage due to the lack of `offset` and `element` tuple labels that `EnumeratedSequence` provides.

#### Add conditional conformance to `RandomAccessCollection` for `Zip2Sequence` rather than overloading `zip`.
It isn’t possible to conditionally conform `Zip2Sequence` to `RandomAccessCollection` in a way that has optimal performance in all cases.
Consider implementing `count`. Having it return `Swift.min(self._sequence1.count, self._sequence2.count)` works fine for random-access collections but is unexpectedly slow for collections that don’t support random-access:
```swift
let evenNumbers = (0 ..< 1_000_000).lazy.filter { $0.isMultiple(of: 2) }
let zipped = zip(evenNumbers, ["lorum", "ipsum", "dolor"])
// This would traverse the entire `0 ..< 1_000_000` range, even though the
// zipped collection only has 3 elements!
_ = zipped.count
```
But if `count` instead naively iterated over each pair of elements and counted them along the way, then this operation would always be O(n) and no longer meet the performance requirements of the `RandomAccessCollection` protocol.
The underlying issue is that the same implementation of `count` needs to work for random-access collections as well as non-random-access collections, meeting both of their individual performance needs.
The initial version of this proposal attempted to work around this problem by adding a `_hasFastCount` customisation point to the `Collection` protocol that can be checked at runtime inside the implementation of `count`:
```swift
protocol Collection: Sequence {
  // ...
  var _hasFastCount: Bool { get }
}
extension Collection {
  var _hasFastCount: Bool { false }
}
extension RandomAccessCollection {
  var _hasFastCount: Bool { true }
}
extension Zip2Sequence: Collection
  where Sequence1.Collection, Sequence2.Collection
{
  // ...
  var count: Int {
    if self._sequence1._hasFastCount && self._sequence2._hasFastCount {
      // It's fine to access each collection's `count` here.
      return Swift.min(self._sequence1.count, self._sequence2.count)
    } else {
      // Use some other strategy that finds the number of pairs in O(n)
      // without accessing the `count` property on the underlying collections.
      // ...
    }
  }
}
```
However, this didn't always work as intended. When a type conditionally conforms to `RandomAccessCollection`, accessing the value’s `_hasFastCount` property in a context where it is only statically known to be a `Collection` does not invoke the default implementation defined in the `RandomAccessCollection` extension:
```swift
// `ReversedCollection` conditionally conforms to `RandomAccessCollection`
// when the base collection does.
let reversedNumbers = (0 ..< 1_000_000).reversed()
let zipped = zip(reversedNumbers, ["lorum", "ipsum", "dolor"])
// Accidentally an O(n) operation.
_ = zipped.count
```
In this case, the `_hasFastCount` entry in the witness table of the `Collection` conformance of `reversedNumbers` would contain the default implementation defined in the extension on `Collection` (returning `false`) rather than the one on `RandomAccessCollection` (returning `true`), due to `ReversedCollection`’s conditional conformance to `RandomAccessCollection`. As a result, `self._sequence1._hasFastCount` inside `zipped.count` would evaluate to `false`, incorrectly triggering the code path meant for non-random-access collection.
A separate `Zip2RandomAccessCollection` type does not have this problem because the underlying collections are statically known to be random-access, and therefore `Swift.min(self._sequence1.count, self._sequence2.count)` suffices.

#### Only conform `Zip2Sequence` and `EnumeratedSequence` to `BidirectionalCollection` when the base collections conform to `RandomAccessCollection` rather than `BidirectionalCollection`.
`EnumeratedSequence` is simpler, the trade-off will be presented in terms of that type, but all of the below applies to both types equally.
 
Here’s what the `Collection` conformance could look like:
 
```swift
extension EnumeratedSequence: Collection where Base: Collection {
    struct Index {
        let base: Base.Index
        let offset: Int
    }
    var startIndex: Index {
        Index(base: _base.startIndex, offset: 0)
    }
    var endIndex: Index {
        Index(base: _base.endIndex, offset: 0)
    }
    func index(after index: Index) -> Index {
        Index(base: _base.index(after: index.base), offset: index.offset + 1)
    }
    subscript(index: Index) -> (offset: Int, element: Base.Element) {
        (index.offset, _base[index.base])
    }
}

extension EnumeratedSequence.Index: Comparable {
    static func == (lhs: Self, rhs: Self) -> Bool {
        return lhs.base == rhs.base
    }
    static func < (lhs: Self, rhs: Self) -> Bool {
        return lhs.base < rhs.base
    }
}
```

Here’s what the `Bidirectional` conformance could look like. The question is: should `Base` be required to conform to `BidirectionalCollection` or `RandomAccessCollection`?
 
```swift
extension EnumeratedSequence: BidirectionalCollection where Base: ??? {
    func index(before index: Index) -> Index {
        let currentOffset = index.base == _base.endIndex ? _base.count : index.offset
        return Index(base: _base.index(before: index.base), offset: currentOffset - 1)
    }
}
```

Notice that calling `index(before:)` with the end index requires computing the `count` of the base collection. This is an O(1) operation if the base collection is `RandomAccessCollection`, but O(n) if it's `BidirectionalCollection`.

##### Option 1: `where Base: BidirectionalCollection`

A direct consequence of `index(before:)` being O(n) when passed the end index is that some operations like `last` are also O(n):

```swift
extension BidirectionalCollection {
    var last: Element? {
        isEmpty ? nil : self[index(before: endIndex)]
    }
}

// A bidirectional collection that is not random-access.
let evenNumbers = (0 ... 1_000_000).lazy.filter { $0.isMultiple(of: 2) }
let enumerated = evenNumbers.enumerated()

// This is still O(1), ...
let endIndex = enumerated.endIndex

// ...but this is O(n).
let lastElement = enumerated.last!
print(lastElement) // (offset: 500000, element: 1000000)
```

However, since this performance pitfall only applies to the end index, iterating over a reversed enumerated collection stays O(n):

```swift
// A bidirectional collection that is not random-access.
let evenNumbers = (0 ... 1_000_000).lazy.filter { $0.isMultiple(of: 2) }

// Reaching the last element is O(n), and reaching every other element is another combined O(n).
for (offset, element) in evenNumbers.enumerated().reversed() {
    // ...
}
```

In other words, this could make some operations unexpectedly O(n), but it’s not likely to make operations unexpectedly O(n²).

##### Option 2: `where Base: RandomAccessCollection`

If `EnumeratedSequence`’s conditional conformance to `BidirectionalCollection` is restricted to when `Base: RandomAccessCollection`, then operations like `last` and `last(where:)` will only be available when they’re guaranteed to be O(1):

```swift
// A bidirectional collection that is not random-access.
let str = "Hello"

let lastElement = str.enumerated().last! // error: value of type 'EnumeratedSequence<String>' has no member 'last'
```

That said, some algorithms that can benefit from bidirectionality such as `reversed()` and `suffix(_:)` are also available on regular collections, but with a less efficient implementation. That means that the code would still compile if the enumerated sequence is not bidirectional, it would just perform worse — the most general version of `reversed()` on `Sequence` allocates an array and adds every element to that array before reversing it:

```swift
// A bidirectional collection that is not random-access.
let str = "Hello"

// This no longer conforms to `BidirectionalCollection`.
let enumerated = str.enumerated()

// As a result, this now returns a `[(offset: Int, element: Character)]` instead
// of a more efficient `ReversedCollection<EnumeratedSequence<String>>`.
let reversedElements = enumerated.reversed()
```

The base collection needs to be traversed twice either way, but the defensive approach of giving the `BidirectionalCollection` conformance a stricter bound ultimately results in an extra allocation.

Taking all of this into account, we've gone with option 1 for the sake of giving collections access to more algorithms and more efficient overloads of some algorithms. Conforming these collections to `BidirectionalCollection` when the base collection conforms to the same protocol is less surprising. We don’t think the possible performance pitfalls pose a large enough risk in practice to negate these benefits.
