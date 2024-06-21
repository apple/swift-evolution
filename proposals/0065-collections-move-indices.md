# A New Model for Collections and Indices

* Proposal: [SE-0065](0065-collections-move-indices.md)
* Authors: [Dmitri Gribenko](https://github.com/gribozavr), [Dave Abrahams](https://github.com/dabrahams), [Maxim Moiseev](https://github.com/moiseev)
* Review Manager: [Chris Lattner](https://github.com/lattner)
* Status: **Implemented (Swift 3.0)**
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-se-0065-a-new-model-for-collections/2371), [Swift-evolution thread](https://forums.swift.org/t/rfc-new-collections-model-collections-advance-indices/1643)
* Implementation: [apple/swift#2108](https://github.com/apple/swift/pull/2108)
* Previous Revisions: [1](https://github.com/swiftlang/swift-evolution/blob/21fac2e8034e79e4f44c1c8799808fc8cba83395/proposals/0065-collections-move-indices.md), [2](https://github.com/swiftlang/swift-evolution/blob/1a821cf7ccbdf1d7566e9ce2e991bdd835ba3b7d/proposals/0065-collections-move-indices.md), [3](https://github.com/swiftlang/swift-evolution/blob/d44c3e7c189ba39ddf8a914ae8b78b71f88fdcdf/proposals/0065-collections-move-indices.md), [4](https://github.com/swiftlang/swift-evolution/blob/57639040dc08d2f0b16d9bda527db069589b58d1/proposals/0065-collections-move-indices.md)

## Summary

We propose a new model for `Collection`s wherein responsibility for
index traversal is moved from the index to the collection itself.  For
example, instead of writing `i.successor()`, one would write
`c.index(after: i)`.  We also propose the following changes as a
consequence of the new model:

* A collection's `Index` can be any `Comparable` type.
* The distinction between intervals and ranges disappears, leaving
  only ranges.
* A closed range that includes the maximal value of its `Bound` type
  is now representable and does not trap.
* Existing “private” in-place index traversal methods are now available
  publicly.

## Motivation

In collections that don't support random access, (string views, sets,
dictionaries, trees, etc.) it's very common that deriving one index
value from another requires somehow inspecting the collection's data.
For example, you could represent an index into a hash table as an
offset into the underlying storage, except that one needs to actually
look at *structure* of the hash table to reach the next bucket.  In
the current model, supporting `i.successor()` means that the index
must additionally store not just an offset, but a reference to the
collection's structure.

The consequences for performance aren't pretty:

* Code that handles indices has to perform atomic reference counting,
  which has significant overhead and can prevent the optimizer from
  making other improvements.

* Additional references to a collection's storage block the
  library-level copy-on-write optimization: in-place mutation of
  uniquely-referenced data.  A live index makes underlying storage
  non-uniquely referenced, forcing unnecessary copies when the
  collection is mutated.  In the standard library, `Dictionary` and
  `Set` use a double-indirection trick to work around this issue.
  Unfortunately, even this trick is not a solution, because (as we
  have recently realized) it isn't threadsafe. [^1]

By giving responsibility for traversal to the collection, we ensure
that operations that need the collection's structure always have it,
without the costs of holding references in indices.

## Other Benefits

Although this change is primarily motivated by performance, it has
other significant benefits:

* Simplifies implementation of non-trivial indices.
* Allows us to eliminate the `Range`/`Interval` distinction.
* Making traversal a direct property of the `Collection` protocol,
  rather than its associated `Index` type, is closer to most peoples'
  mental model for collections, and simplifies the writing of many
  generic constraints.
* Makes it feasible to fix existing concurrency issues in `Set` and
  `Dictionary` indices.
* Allows `String` views to share a single index type, letting us
  eliminate the need for cumbersome index conversion functions (not
  part of this proposal, but planned).

## Out of Scope

This proposal intentionally does not:

* Expand the set of concrete collections provided by the standard
  library.
* Expand the set of collection protocols to provide functionality
  beyond what is already provided (for example, protocols for sorted
  collections, queues etc.)  Discussing how other concrete collections
  fit into the current protocol hierarchy is in scope, though.

## Limitations of the Model

Ideally, our collection model would allow us to implement every
interesting data structure with memory safety, optimal performance,
value semantics, and a variety of other useful properties such as
minimal invalidation of indexes upon mutation.  In practice, these
goals and the Swift language model interact in complicated ways,
preventing some designs altogether, and suggesting a variety of
implementation strategies for others that can be selected based on
one's priorities.  We've done some in-depth investigation of these
implications, but presenting and explaining them is well beyond the
scope of this proposal.

We can, however, be fairly sure that this change does not regress our
ability to build any Collections that could have been built in Swift
2.2.  After all, it is still *possible* to implement indices that store
references and have the old traversal methods (the collection's
traversal methods would simply forward to those of the index), so we
haven't lost the ability to express anything.

## Overview of Type And Protocol Changes

This section covers the proposed structural changes to the library at
a high level.  Details such as protocols introduced purely to work
around compiler limitations (e.g. `Indexable` or `IndexableBase`) have
been omitted.  For a complete view of the code
and documentation changes implementing this proposal, please see this
[pull request](https://github.com/apple/swift/pull/2108).

### Collection Protocol Hierarchy

In the proposed model, indices don't have any requirements beyond
`Comparable`, so the `ForwardIndex`, `BidirectionalIndex`, and
`RandomAccessIndex` protocols are eliminated.  Instead, we introduce
`BidirectionalCollection` and `RandomAccessCollection` to provide the
same traversal distinctions, as shown here:

```
                     +--------+
                     |Sequence|
                     +---+----+
                         |
                    +----+-----+
                    |Collection|
                    +----+-----+
                         |
          +--------------+-------------+
          |              |             |
          |     +--------+--------+    |
          |     |MutableCollection|    |
          |     +-----------------+    |
          |                            |
+---------+-------------+    +---------+----------------+
|BidirectionalCollection|    |RangeReplaceableCollection|
+---------+-------------+    +--------------------------+
          |
 +--------+-------------+
 |RandomAccessCollection|
 +----------------------+
```

These protocols compose naturally with the existing protocols
`MutableCollection` and `RangeReplaceableCollection` to describe a
collection's capabilities, e.g.

```swift
struct Array<Element>
  : RandomAccessCollection,
    MutableCollection,
    RangeReplaceableCollection { ... }

struct UnicodeScalarView : BidirectionalCollection { ... }
```

### Range Types

The proposal adds several new types to support ranges:

* The old `Range<T>`, `ClosedInterval<T>`, and
  `OpenInterval<T>` are replaced with four new generic range types:

  * Two for general ranges (whose bounds are `Comparable`): `Range<T>`
    and `ClosedRange<T>`.  Having a separate `ClosedRange` type allows
    us to address the vexing inability of the old `Range` to represent
    a range containing the maximal value of its bound.

  * Two for ranges that additionally conform to
    `RandomAccessCollection` (requiring bounds that are `Strideable`
    with `Stride` conforming to `Integer`): `CountableRange<T>` and
    `CountableClosedRange<T>`. These types can be folded into
    `Range` and `ClosedRange` when Swift acquires conditional
    conformance capability.

### The Associated `Indices` Type

The following code iterates over the indices of all elements in
`collection`:

```swift
for index in collection.indices { ... }
```

In Swift 2, `collection.indices` returned a `Range<Index>`, but
because a range is a simple pair of indices and indices can no longer
be advanced on their own, `Range<Index>` is no longer iterable.

In order to keep code like the above working, `Collection` has
acquired an associated `Indices` type that is always iterable, and
three generic types were introduced to provide a default `Indices` for
each `Collection` traversal category: `DefaultIndices<C>`,
`DefaultBidirectionalIndices<C>`, and `DefaultRandomAccessIndices<C>`.
These types store the underlying collection as a means of traversal.
Collections like `Array` whose `Indices` don't need the collection
simply use `typealias Indices = CountableRange<Index>`.

### Expanded Default Slice Types

Because Swift doesn't support conditional protocol conformances and
the three traversal distinctions have been moved into the `Collection`
hierarchy, the four generic types `Slice`, `MutableSlice`,
`RangeReplaceableSlice`, and `MutableRangeReplaceableSlice` have
become twelve, with the addition of variations such as
`RangeReplaceableBidirectionalSlice`.

### The `Comparable` Requirement on Indices

In this model indices store the minimal amount of information required
to describe an element's position.  Usually an index can be
represented with one or two `Int`s that efficiently encode the path to
the element from the root of a data structure.  Since one is free to
choose the encoding of the “path”, we think it is possible to choose
it in such a way that indices are cheaply comparable.  That has been
the case for all of the indices required to implement the standard
library, and a few others we investigated while researching this
change.

It's worth noting that this requirement isn't strictly
necessary. Without it, though, indices would have no requirements
beyond `Equatable`, and creation of a `Range<T>` would have to be
allowed for any `T` conforming to `Equatable`. As a consequence, most
interesting range operations, such as containment checks, would be
unavailable unless `T` were also `Comparable`, and we'd be unable to
provide bounds-checking in the general case.

That said, the requirement has real benefits.  For example, it allows
us to support distance measurement between arbitrary indices, even in
collections without random access traversal.  In the old model,
`x.distance(to: y)` for these collections had the undetectable
precondition that `x` precede `y`, with unpredictable consequences for
violation in the general case.

## Detailed API Changes

This section describes changes to methods, properties, and associated
types at a high level.  Details related to working around compiler
limitations have been omitted.  For a complete view of the code
and documentation changes implementing this proposal, please see this
[pull request](https://github.com/apple/swift/pull/2108).

### `Collection`s

The following APIs were added:

```swift
protocol Collection {
  ...
  /// A type that can represent the number of steps between pairs of
  /// `Index` values where one value is reachable from the other.
  ///
  /// Reachability is defined by the ability to produce one value from
  /// the other via zero or more applications of `index(after: _)`.
  associatedtype IndexDistance : SignedInteger = Int

  /// A collection type whose elements are the indices of `self` that
  /// are valid for subscripting, in ascending order.
  associatedtype Indices : Collection = DefaultIndices<Self>

  /// The indices that are valid for subscripting `self`, in ascending order.
  ///
  /// - Note: `indices` can hold a strong reference to the collection itself,
  ///   causing the collection to be non-uniquely referenced.  If you need to
  ///   mutate the collection while iterating over its indices, use the
  ///   `index(after: _)` method starting with `startIndex` to produce indices
  ///   instead.
  /// 
  ///   ```
  ///   var c = [10, 20, 30, 40, 50]
  ///   var i = c.startIndex
  ///   while i != c.endIndex {
  ///       c[i] /= 5
  ///       i = c.index(after: i)
  ///   }
  ///   // c == [2, 4, 6, 8, 10]
  ///   ```
  var indices: Indices { get }

  /// Returns the position immediately after `i`.
  ///
  /// - Precondition: `(startIndex..<endIndex).contains(i)`
  @warn_unused_result
  func index(after i: Index) -> Index

  /// Replaces `i` with its successor.
  func formIndex(after i: inout Index)

  /// Returns the result of advancing `i` by `n` positions.
  ///
  /// - Returns:
  ///   - If `n > 0`, the `n`th index after `i`.
  ///   - If `n < 0`, the `n`th index before `i`.
  ///   - Otherwise, `i` unmodified.
  ///
  /// - Precondition: `n >= 0` unless `Self` conforms to
  ///   `BidirectionalCollection`.
  /// - Precondition:
  ///   - If `n > 0`, `n <= self.distance(from: i, to: self.endIndex)`
  ///   - If `n < 0`, `n >= self.distance(from: i, to: self.startIndex)`
  ///
  /// - Complexity:
  ///   - O(1) if `Self` conforms to `RandomAccessCollection`.
  ///   - O(`abs(n)`) otherwise.
  func index(_ i: Index, offsetBy n: IndexDistance) -> Index

  /// Returns the result of advancing `i` by `n` positions, or until it
  /// equals `limit`.
  ///
  /// - Returns:
  ///   - If `n > 0`, the `n`th index after `i` or `limit`, whichever
  ///     is reached first.
  ///   - If `n < 0`, the `n`th index before `i` or `limit`, whichever
  ///     is reached first.
  ///   - Otherwise, `i` unmodified.
  ///
  /// - Precondition: `n >= 0` unless `Self` conforms to
  ///   `BidirectionalCollection`.
  ///
  /// - Complexity:
  ///   - O(1) if `Self` conforms to `RandomAccessCollection`.
  ///   - O(`abs(n)`) otherwise.
  func index(
    _ i: Index, offsetBy n: IndexDistance, limitedBy limit: Index) -> Index

  /// Advances `i` by `n` positions.
  ///
  /// - Precondition: `n >= 0` unless `Self` conforms to
  ///   `BidirectionalCollection`.
  /// - Precondition:
  ///   - If `n > 0`, `n <= self.distance(from: i, to: self.endIndex)`
  ///   - If `n < 0`, `n >= self.distance(from: i, to: self.startIndex)`
  ///
  /// - Complexity:
  ///   - O(1) if `Self` conforms to `RandomAccessCollection`.
  ///   - O(`abs(n)`) otherwise.
  func formIndex(_ i: inout Index, offsetBy n: IndexDistance)

  /// Advances `i` by `n` positions, or until it equals `limit`.
  ///
  /// - Precondition: `n >= 0` unless `Self` conforms to
  ///   `BidirectionalCollection`.
  ///
  /// - Complexity:
  ///   - O(1) if `Self` conforms to `RandomAccessCollection`.
  ///   - O(`abs(n)`) otherwise.
  func formIndex(
    _ i: inout Index, offsetBy n: IndexDistance, limitedBy limit: Index)

  /// Returns the distance between `start` and `end`.
  ///
  /// - Precondition: `start <= end` unless `Self` conforms to
  ///   `BidirectionalCollection`.
  /// - Complexity:
  ///   - O(1) if `Self` conforms to `RandomAccessCollection`.
  ///   - O(`n`) otherwise, where `n` is the method's result.
  func distance(from start: Index, to end: Index) -> IndexDistance
}

protocol BidirectionalCollection {
  /// Returns the position immediately preceding `i`.
  ///
  /// - Precondition: `i > startIndex && i <= endIndex` 
  func index(before i: Index) -> Index

  /// Replaces `i` with its predecessor.
  ///
  /// - Precondition: `i > startIndex && i <= endIndex`
  func formIndex(before i: inout Index)
}
```

Note:

* The `formIndex` overloads essentially enshrine the previously-hidden
  `_successorInPlace` et al., which can be important for performance
  when handling the rare heavyweight index type such as `AnyIndex`.

* `RandomAccessCollection` does not add any *syntactic* requirements
  beyond those of `BidirectionalCollection`.  Instead, it places
  tighter performance bounds on operations such as `c.index(i,
  offsetBy: n)` (O(1) instead of O(`n`)).

## `Range`s

The four range [Range Types](#range-types) share the common interface
shown below:

```swift
public struct Range<Bound: Comparable> : Equatable {
  
  /// Creates an instance with the given bounds.
  ///
  /// - Note: As this initializer does not check its precondition, it
  ///   should be used as an optimization only, when one is absolutely
  ///   certain that `lower <= upper`.  In general, the `..<` and `...`
  ///   operators are to be preferred for forming ranges.
  ///
  /// - Precondition: `lower <= upper`
  init(uncheckedBounds: (lower: Bound, upper: Bound))

  /// Returns `true` if the range contains the `value`.
  func contains(_ value: Bound) -> Bool
  
  /// Returns `true` iff `self` and `other` contain a value in common.
  func overlaps(_ other: Self) -> Bool

  /// Returns `true` iff `self.contains(x)` is `false` for all values of `x`.
  var isEmpty: Bool { get }
  
  /// The range's lower bound.
  var lowerBound: Bound { get }
  
  /// The range's upper bound.
  var upperBound: Bound { get }
  
  /// Returns `self` clamped to `limits`.
  ///
  /// The bounds of the result, even if it is empty, are always
  /// limited to the bounds of `limits`.
  func clamped(to limits: Self) -> Self
}
```

In addition, every implementable lossless conversion between range
types is provided as a label-less `init` with one argument:

```swift
let a = 1..<10
let b = ClosedRange(a) // <=== Here
```

Note in particular:

* In `Range<T>`, `T` is `Comparable` rather than an index
  type that can be advanced, so a generalized range is no longer a
  `Collection`, and `startIndex`/`endIndex` have become
  `lowerBound`/`upperBound`.
* The semantic order of `Interval`'s `clamp` method, which was
  unclear at its use-site, has been inverted and updated with a
  preposition for clarity.
  
## Downsides

The proposed approach has several disadvantages, which we explore here
in the interest of full disclosure:

* In Swift 2, `RandomAccessIndex` has operations like `+` that provide
  easy access to arbitrary position offsets in some collections.  That
  could also be seen as discouragement from trying to do random access
  operations with less-refined index protocols, because in those cases
  one has to resort to constructs like `i.advancedBy(n)`.  In this
  proposal, there is only `c.index(i, offsetBy: n)`, which makes
  random access equally (in)convenient for all collections, and there
  is no particular syntactic penalty for doing things that might turn
  out to be inefficient.

* Index movement is more complex in principle, since it now involves
  not only the index, but the collection as well. The impact of this
  complexity is limited somewhat because it's very common that code
  moving indices occurs in a method of the collection type, where
  “implicit `self`” kicks in.  The net result is that index
  manipulations end up looking like free function calls:

  ```swift
  let j = index(after: i)           // self.index(after: i)
  let k = index(j, offsetBy: 5)     // self.index(j, offsetBy: 5)
  ```

* The
  [new index manipulation methods](https://github.com/apple/swift/blob/swift-3-indexing-model/stdlib/public/core/Collection.swift#L135)
  increase the API surface area of `Collection`, which is already
  quite large since algorithms are implemented as extensions.

* Because Swift is unable to express conditional protocol
  conformances, implementing this change has required us to create a
  great deal of complexity in the standard library API.  Aside from
  the two excess “`Countable`” range types, there are new overloads
  for slicing and twelve distinct slice types that capture all the
  combinations of traversal, mutability, and range-replaceability.
  While these costs are probably temporary, they are very real in the
  meantime.

* The API complexity mentioned above stresses the type checker,
  requiring
  [several](https://github.com/apple/swift/commit/1a875cb922fa0c98d51689002df8e202993db2d3)
  [changes](https://github.com/apple/swift/commit/6c56af5c1bc319825872a25041ec33ab0092db05)
  just to get our test code to type-check in reasonable time.  Again,
  an ostensibly temporary—but still real—cost.

## Impact on existing code

Code that **does not need to change**:

* Code that works with `Array`, `ArraySlice`, `ContiguousArray`, and
  their indices.

* Code that operates on arbitrary collections and indices (on concrete
  instances or in generic context), but does no index traversal.

* Iteration over collections' indices with `c.indices` does not change.

* APIs of high-level collection algorithms don't change, even for
  algorithms that accept indices as parameters or return indices (e.g.,
  `index(of:)`, `min()`, `sort()`, `prefix()`, `prefix(upTo:)` etc.)

Code that **needs to change**:

* Code that advances indices (`i.successor()`, `i.predecessor()`,
  `i.advanced(by:)` etc.) or calculates distances between indices
  (`i.distance(to:)`) now needs to call a method on the collection
  instead.

  ```swift
  // Before:
  var i = c.index { $0 % 2 == 0 }
  let j = i.successor()
  print(c[j])

  // After:
  var i = c.index { $0 % 2 == 0 }   // No change in algorithm API.
  let j = c.index(after: i)         // Advancing an index requires a collection instance.
  print(c[j])                       // No change in subscripting.
  ```

  The transformation from `i.successor()` to `c.index(after: i)` is
  non-trivial.  Performing it correctly requires knowing how to get
  the corresponding collection.  In general, it is not possible to
  perform this migration automatically.  A very sophisticated migrator
  could handle some easy cases.

* Custom collection implementations need to change.  A simple fix would
  be to just move the methods from indices to collections to satisfy
  new protocol requirements.  This is a more or less mechanical fix that
  does not require design work.  This fix would allow the code to
  compile and run.

  In order to take advantage of performance improvements in
  the new model, and remove reference-counted stored properties from
  indices, the representation of the index might need to be redesigned.

  Implementing custom collections, as compared to using collections, is
  a niche case.  We believe that for custom collection types it is
  sufficient to provide clear steps for manual migration that don't
  require a redesign.  Implementing this in an automated migrator might
  be possible, but would be a heroic migration for a rare case.

## Implementation Status

[This pull request](https://github.com/apple/swift/pull/2108) contains
a complete implementation.

## Alternatives considered

We considered index-less models, for example, [D's
std.range](https://dlang.org/library/std/range.html) (see also [On
Iteration by Andrei
Alexandrescu](http://www.informit.com/articles/printerfriendly/1407357)).
Ranges work well for reference-typed collections, but it is not clear
how to adjust the concept of D's range (similar to `Slice` in Swift) for
mutable value-typed collections.  In D, you process a collection by
repeatedly slicing off elements.  Once you have found an element that
you would like to mutate, it is not clear how to actually change the
original collection, if the collection and its slice are value types.

----

[^1]: `Dictionary` and `Set` use a double-indirection trick to avoid
disturbing the reference count of the storage with indices.

```
    +--+    class                       struct
    |RC|---------+          +-----------------+
    +--+ Storage |<---------| DictionaryIndex |
      |          |          | value           |
      +----------+          +-----------------+
          ^
    +--+  |     class                struct
    |RC|-------------+        +------------+
    +--+ Indirection |<-------| Dictionary |
      |  ("owner")   |        | value      |
      +--------------+        +------------+
```

Instances of `Dictionary` point to an indirection, while
instances of `DictionaryIndex` point to the storage itself.
This allows us to have two separate reference counts.  One of
the refcounts tracks just the live `Dictionary` instances, which
allows us to perform precise uniqueness checks.

The issue that we were previously unaware of is that this scheme
is not thread-safe.  When uniquely-referenced storage is being
mutated in place, indices can be concurrently being incremented
(on a different thread).  This would be a read/write data race.

Fixing this data race (to provide memory safety) would require
locking dictionary storage on every access, which would be an
unacceptable performance penalty.
