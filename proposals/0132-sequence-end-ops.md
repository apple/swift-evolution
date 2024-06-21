# Rationalizing Sequence end-operation names

* Proposal: [SE-0132](0132-sequence-end-ops.md)
* Authors: [Becca Royal-Gordon](https://github.com/beccadax), [Dave Abrahams](https://github.com/dabrahams)
* Review Manager: [Chris Lattner](https://github.com/lattner)
* Status: **Rejected**
* Decision Notes: [Rationale](https://forums.swift.org/t/deferred-se-0132-rationalizing-sequence-end-operation-names/3577)

## Introduction

Sequence and Collection offer many special operations which access or 
manipulate its first or last elements, but they are plagued by 
inconsistent naming which can make it difficult to find inverses or 
remember what the standard library offers. We propose that we standardize 
these names so they follow consistent, predictable patterns.

Swift-evolution thread: [[Draft] Rationalizing Sequence end-operation names](https://forums.swift.org/t/draft-rationalizing-sequence-end-operation-names/3103)

### Scope

**This proposal does not aim to add or remove any functionality**; 
it merely renames and redesigns existing operations. **Adding new 
operations is out of scope for this proposal** unless it's incidental 
to the new designs.

Nonetheless, we *do* want the new designs to support adding more 
operations in the future. The names we've chosen are informed by some of 
the speculative APIs discussed in "Future directions", although we think 
they are perfectly sensible names even if nothing else changes.

## Motivation

The `Sequence` and `Collection` protocols offer a wide variety of APIs 
which are defined to operate on, or from, one end of the sequence:

|                                  | Get                           | Index                        | Exclude                     | Remove (1)       | Pop (1)      | Equate (2)                                     |
| -------------------------------- | ----------------------------- | ---------------------------- | --------------------------- | ---------------- | ------------ | ---------------------------------------------- |
| **Fixed Size**                   |
| First 1                          | C.first                       | -                            | S.dropFirst()               | C.removeFirst()  | C.popFirst() | -                                              |
| Last 1                           | C.last                        | -                            | S.dropLast()                | C.removeLast()   | C.popLast()  | -                                              |
| First (n: Int)                   | S.prefix(3)                   | -                            | S.dropFirst(3)              | C.removeFirst(3) | -            | S.starts(with:&nbsp;[x,y,z])                   |
| &nbsp;&nbsp;...with closure      | S.prefix(while:&nbsp;isPrime) | -                            | S.drop(while:&nbsp;isPrime) | -                | -            | S.starts(with:&nbsp;[x,y,z],&nbsp;by:&nbsp;==) |
| Last (n: Int)                    | S.suffix(3)                   | -                            | S.dropLast(3)               | C.removeLast(3)  | -            | -                                              |
| &nbsp;&nbsp;...with closure      | -                             | -                            | -                           | -                | -            | -                                              |
| **Searching From End**           |
| First&nbsp;matching&nbsp;element | -                             | C.index(of:&nbsp;x)          | -                           | -                | -            | -                                              |
| &nbsp;&nbsp;...with closure      | S.first(where:&nbsp;isPrime)  | C.index(where:&nbsp;isPrime) | -                           | -                | -            | -                                              |
| Last matching element            | -                             | -                            | -                           | -                | -            | -                                              |
| &nbsp;&nbsp;...with closure      | -                             | -                            | -                           | -                | -            | -                                              |
| **Based on Index**               |
| startIndex ..< (i: Index)        | C.prefix(upTo:&nbsp;i)        | -                            | -                           | -                | -            | -                                              |
| startIndex ... (i: Index)        | C.prefix(through:&nbsp;i)     | -                            | -                           | -                | -            | -                                              |
| (i: Index) ..< endIndex          | C.suffix(from:&nbsp;i)        | -                            | -                           | -                | -            | -                                              |

> We have included several blank rows for operands which fit the APIs'
> patterns, even if they don't happen to have any operations currently.
> 
> **Type abbreviations:**
> 
> * S = Sequence
> * C = Collection (or a sub-protocol like BidirectionalCollection)
> 
> **Notes:**
> 
> 1. `remove` and `pop` both mutate the array to delete the indicated 
>    element(s), but `remove` assumes as a precondition that the 
>    indicated elements exist, while `pop` checks whether or not they 
>    exist.
>
> 2. `String` and `NSString` have bespoke versions of *first n* and 
>    *last n* Equate operations, in the form of their `hasPrefix` and 
>    `hasSuffix` methods.

Leaving aside the question of whether any gaps in these tables ought to 
be filled, we see a number of issues with existing terminology.

### Inconsistent use of `prefix` and `suffix`

Some APIs which operate on a variable number of elements anchored at 
one end or the other use the terms `prefix` or `suffix`:

* `Sequence.prefix(_:)` and `Sequence.suffix(_:)`
* `Sequence.prefix(while:)`
* `String.hasPrefix(_:)` and `String.hasSuffix(_:)`

Others, however, use `first` or `last`:

* `Sequence.dropFirst(_:)` and `Sequence.dropLast(_:)`
* `Sequence.removeFirst(_:)` and `Sequence.removeLast(_:)`

Still others use neither:

* `Sequence.starts(with:)`
* `Sequence.drop(while:)`

These methods are all closely related, but because of this inconsistent 
terminology, they fail to form predictable method families.

### `first` has multiple meanings

The word `first` can mean three different things in these APIs:

1. Just the very first element of the sequence.

2. A subsequence of elements anchored at the beginning of the sequence, 
   as mentioned in the last point.

3. The first element encountered in the sequence which matches a given 
   criterion when walking from the beginning of the sequence towards the 
   end.

It would be nice to have more clarity here—particularly around #2, which 
implies different return value behavior.

### `drop` is misleading and scary

In a Swift context, we believe the `drop` methods are actively confusing:

* `drop` does not have the -ing or -ed suffix normally used for a 
  nonmutating method.

* `drop` has strong associations with destructive operations; it's the 
  term used, for instance, for deleting whole tables in SQL. Even 
  `dropping` would probably sound more like a mutating operation than 
  alternatives.

* As previously mentioned, the use of `dropFirst` and `dropLast` for 
  single-drop operations and multiple-drop operations breaks up method 
  families.

`drop`, `dropFirst`, and `dropLast` are terms of art, so we allow them 
a certain amount of leeway. However, we believe the `drop` functions 
go well beyond what we should permit. They are relatively 
uncommon operations, associated primarily with functional languages 
rather than mainstream object-oriented or imperative languages, and 
their violation of the normal Swift naming guidelines is especially 
misleading.

The term-of-art exception is not a suicide pact; it is meant to aid 
understanding by importing common terminology, not bind us to follow 
every decision made by any language that came before us. In this case, 
we think we should ignore precedent and forge our own path.

### Unstated direction of operation

Several APIs could theoretically be implemented by working from either 
end of the sequence, and would return different results depending on 
the direction, but do not indicate the direction in their names:

* `Sequence.drop(while:)`
* `Collection.index(of:)`

Adding a direction to these APIs would make their behavior clearer and 
permit us to offer opposite-end equivalents in the future. (Unmerged 
[swift-evolution pull request 329](https://github.com/swiftlang/swift-evolution/pull/329) 
would add `lastIndex` methods.)

### Operations taking an index are really slicing

`prefix(upTo:)`, `prefix(through:)`, and `suffix(from:)` at first 
appear to belong to the same family as the other `prefix` and `suffix`
methods, but deeper examination reveals otherwise. They are the only 
operations which take indices, and they don't cleanly extend to the 
other operations which belong to these families. (For instance, it 
would not make sense to add a `dropPrefix(upTo:)` method; it would be 
equivalent to `suffix(from:)`.)

Also, on `Int`-indexed collections like `Array`, `prefix(_:)` and 
`prefix(upTo:)` are identical, but there is little relationship between 
`suffix(_:)` and `suffix(from:)`, which is confusing.

`suffix(from:)` is a particularly severe source of confusion. The other 
`suffix` APIs all have parameters relative to the *end* of the 
collection, but `suffix(from:)`'s index is still relative to the 
*beginning* of the array. This is obvious if you think deeply about the 
meaning of an index, but we don't really want to force our users to 
stare at a strange API until they have an epiphany.

We believe these operations have much more in common with slicing a 
collection using a range, and that reimagining them as slicing APIs 
will be more fruitful. Thus, the bottom of the table above should 
probably be split into a separate one and combined with a table of 
subrange APIs:

|                                                  | Type                         | Get                       | Remove                              | Replace                                                     |
| ------------------------------------------------ | ---------------------------- | ------------------------- | ----------------------------------- | ----------------------------------------------------------- |
| **Based&nbsp;on&nbsp;Index,&nbsp;Arbitrary**     |
| (i: Index) ..< (j: Index)                        | Range\<Index>                | C[i ..< j]                | C.removeSubrange(i&nbsp;..<&nbsp;j) | C.replaceSubrange(i&nbsp;..<&nbsp;j,&nbsp;with:&nbsp;[x,y]) |
| &nbsp;&nbsp;...Countable                         | CountableRange\<Index>       | C[i ..< j]                | C.removeSubrange(i&nbsp;..<&nbsp;j) | C.replaceSubrange(i&nbsp;..<&nbsp;j,&nbsp;with:&nbsp;[x,y]) |
| (i: Index) ... (j: Index)                        | ClosedRange\<Index>          | C[i ... j]                | C.removeSubrange(i ... j)           | C.replaceSubrange(i ... j, with: [x,y])                     |
| &nbsp;&nbsp;...Countable                         | CountableClosedRange\<Index> | C[i ... j]                | C.removeSubrange(i ... j)           | C.replaceSubrange(i ... j, with: [x,y])                     |
| **Based&nbsp;on&nbsp;Index,&nbsp;From&nbsp;End** |
| startIndex ..< (i: Index)                        | upTo: Index                  | C.prefix(upTo:&nbsp;i)    | -                                   | -                                                           |
| (i: Index) ..< endIndex                          | from: Index                  | C.suffix(from:&nbsp;i)    | -                                   | -                                                           |
| startIndex ... (i: Index)                        | through: Index               | C.prefix(through:&nbsp;i) | -                                   | -                                                           |

### Why does it matter?

Many of these APIs are only occasionally necessary, so it's important 
that they be easy to find when needed and easy to understand when 
read. If you know that `prefix(10)` will get the first ten elements but 
don't know what its inverse is, you will probably not guess that it's 
`dropFirst(10)`. The confusing, conflicting names in these APIs are a 
barrier to users adopting them where appropriate.

## Proposed solution

We sever the index-taking APIs from the others, forming two separate 
families, which we will call the "Sequence-end operations" and the 
"index-based operations". We then consider and redesign them along 
separate lines.

### Sequence-end operations

Each of these APIs should be renamed to use a directional word based on 
its row in the table:

| Operand                          | Directional word   |
| -------------------------------- | ------------------ |
| **Fixed Size**                   |
| First 1                          | first              |
| Last 1                           | last               |
| First (n: Int)                   | prefix             |
| &nbsp;&nbsp;...with closure      | prefix             |
| Last (n: Int)                    | suffix             |
| &nbsp;&nbsp;...with closure      | suffix             |
| **Searching From End**           |
| First&nbsp;matching&nbsp;element | first              |
| &nbsp;&nbsp;...with closure      | first              |
| Last matching element            | last               |
| &nbsp;&nbsp;...with closure      | last               |

To accomplish this, `starts(with:)` should be renamed to 
`hasPrefix(_:)`, and other APIs should have directional words replaced 
or added as appropriate.

Additionally, the word `drop` in the "Exclude" APIs should be replaced 
with `removing`. These operations omit the same elements which the 
`remove` operations delete, so even though the types are not always the 
same (`removing` returns `SubSequence`, not `Self`), we think they are 
similar enough to deserve to be treated as nonmutating forms.

These changes yield (altered names **bold**):

|                                  | Get                           | Index                                 | Exclude                                   | Remove (1)            | Pop (1)      | Equate (2)                                 |
| -------------------------------- | ----------------------------- | ------------------------------------- | ----------------------------------------- | --------------------- | ------------ | ------------------------------------------ |
| **Fixed Size**                   |
| First 1                          | C.first                       | -                                     | **S.removingFirst()**                     | C.removeFirst()       | C.popFirst() | -                                          |
| Last 1                           | C.last                        | -                                     | **S.removingLast()**                      | C.removeLast()        | C.popLast()  | -                                          |
| First (n: Int)                   | S.prefix(3)                   | -                                     | **S.removingPrefix(3)**                   | **C.removePrefix(3)** | -            | **S.hasPrefix([x,y,z])**                   |
| &nbsp;&nbsp;...with closure      | S.prefix(while:&nbsp;isPrime) | -                                     | **S.removingPrefix(while:&nbsp;isPrime)** | -                     | -            | **S.hasPrefix([x,y,z],&nbsp;by:&nbsp;==)** |
| Last (n: Int)                    | S.suffix(3)                   | -                                     | **S.removingSuffix(3)**                   | **C.removeSuffix(3)** | -            | -                                          |
| &nbsp;&nbsp;...with closure      | -                             | -                                     | -                                         | -                     | -            | -                                          |
| **Searching From End**           |
| First&nbsp;matching&nbsp;element | -                             | **C.firstIndex(of:&nbsp;x)**          | -                                         | -                     | -            | -                                          |
| &nbsp;&nbsp;...with closure      | S.first(where:&nbsp;isPrime)  | **C.firstIndex(where:&nbsp;isPrime)** | -                                         | -                     | -            | -                                          |
| Last matching element            | -                             | -                                     | -                                         | -                     | -            | -                                          |
| &nbsp;&nbsp;...with closure      | -                             | -                                     | -                                         | -                     | -            | -                                          |

### Index-based operations

Because these APIs look up elements based on their indices, we believe 
these operations should be exposed as subscripts, and ideally should 
look like other slicing operations:

```swift
let head = people[..<i]
let tail = people[i..<]
let rearrangedPeople = tail + head
```

<!-- Comment to make my editor happy -->

We will accomplish this by introducing two new types, `IncompleteRange` 
and `IncompleteClosedRange`. These are similar to `Range` and 
`ClosedRange`, except that the bounds are optional.

To construct them, we will introduce both prefix and suffix operators 
taking a non-optional bound, and infix operators taking optional bounds. 
(We offer both because `c[..<i]` is more convenient than `c[nil ..< i]`, 
but doesn't allow you to dynamically choose between supplying and 
omitting a bound.) These will follow the existing convention: `..<` will 
construct the half-open `IncompleteRange`, while `...` will construct 
`IncompleteClosedRange`.

Rather than continuing to proliferate overloads of slicing subscripts, 
we will also introduce a new `RangeExpression` protocol which allows 
any range-like type to convert itself into a plain `Range<Index>` 
appropriate to the collection in question. Thus, there should only be 
two range subscripts: one taking `Range<Index>`, and one taking 
everything else.

We will also modify the existing `removeSubrange(_:)` and 
`replaceSubrange(_:with:)` calls to take `RangeExpression` instances, 
thereby merging many existing variants into one while simultaneously 
extending them to support `IncompleteRange` and `IncompleteClosedRange`.
Though technically additive, we believe this is an easy win.

Thus, the table above becomes:

|                                                  | Type                              | Get                  | Remove                              | Replace                                                     |
| ------------------------------------------------ | --------------------------------- | -------------------- | ----------------------------------- | ----------------------------------------------------------- |
| **Based&nbsp;on&nbsp;Index,&nbsp;Arbitrary**     |
| (i: Index) ..< (j: Index)                        | Range\<Index>                     | C[i&nbsp;..<&nbsp;j] | C.removeSubrange(i&nbsp;..<&nbsp;j) | C.replaceSubrange(i&nbsp;..<&nbsp;j,&nbsp;with:&nbsp;[x,y]) |
| &nbsp;&nbsp;...Countable                         | CountableRange\<Index>            | C[i ..< j]           | C.removeSubrange(i&nbsp;..<&nbsp;j) | C.replaceSubrange(i&nbsp;..<&nbsp;j,&nbsp;with:&nbsp;[x,y]) |
| (i: Index) ... (j: Index)                        | ClosedRange\<Index>               | C[i ... j]           | C.removeSubrange(i ... j)           | C.replaceSubrange(i ... j, with: [x,y])                     |
| &nbsp;&nbsp;...Countable                         | CountableClosedRange\<Index>      | C[i ... j]           | C.removeSubrange(i ... j)           | C.replaceSubrange(i ... j, with: [x,y])                     |
| **Based&nbsp;on&nbsp;Index,&nbsp;From&nbsp;End** |
| startIndex ..< (i: Index)                        | **IncompleteRange\<Index>**       | **C[..\<i]**         | **C.removeSubrange(..\<i)**         | **C.replaceSubrange(..\<i,&nbsp;with:&nbsp;[x,y])**         |
| (i: Index) ..< endIndex                          | **IncompleteRange\<Index>**       | **C[i..\<]**         | **C.removeSubrange(i..\<)**         | **C.replaceSubrange(i..\<, with: [x,y])**                   |
| startIndex ... (i: Index)                        | **IncompleteClosedRange\<Index>** | **C[...i]**          | **C.removeSubrange(...i)**          | **C.replaceSubrange(...i, with: [x,y])**                    |

However, it should be implemented with merely:

|                                               | Type                                                | Get                  | Remove                              | Replace                                                     |
| --------------------------------------------- | --------------------------------------------------- | -------------------- | ----------------------------------- | ----------------------------------------------------------- |
| (i:&nbsp;Index)&nbsp;..<&nbsp;(j:&nbsp;Index) | Range\<Index>                                       | C[i&nbsp;..<&nbsp;j] | C.removeSubrange(i&nbsp;..<&nbsp;j) | C.replaceSubrange(i&nbsp;..<&nbsp;j,&nbsp;with:&nbsp;[x,y]) |
| Everything else                               | RangeExpression where&nbsp;Bound&nbsp;==&nbsp;Index | C[i&nbsp;...&nbsp;j] | C.removeSubrange(i&nbsp;...&nbsp;j) | C.replaceSubrange(i&nbsp;...&nbsp;j,&nbsp;with:&nbsp;[x,y]) |

## Detailed design

### Sequence-end operations

The following methods should be renamed as follows wherever they appear 
in the standard library. These are simple textual substitutions; we 
propose no changes whatsoever to types, parameter interpretations, or 
other semantics.

| Old method                                        | New method                                              |
| ------------------------------------------------- | ------------------------------------------------------- |
| `dropFirst() -> SubSequence`                      | `removingFirst() -> SubSequence`                        |
| `dropLast() -> SubSequence`                       | `removingLast() -> SubSequence`                         |
| `dropFirst(_ n: Int) -> SubSequence`              | `removingPrefix(_ n: Int) -> SubSequence`               |
| `drop(@noescape while predicate: (Iterator.Element) throws -> Bool) rethrows -> SubSequence` | `removingPrefix(@noescape while predicate: (Iterator.Element) throws -> Bool) rethrows -> SubSequence` |
| `dropLast(_ n: Int) -> SubSequence`               | `removingSuffix(_ n: Int) -> SubSequence`               |
| `removeFirst(_ n: Int)`                           | `removePrefix(_ n: Int)`                                |
| `removeLast(_ n: Int)`                            | `removeSuffix(_ n: Int)`                                |
| `starts<PossiblePrefix: Sequence>(with possiblePrefix: PossiblePrefix) -> Bool where ...` | `hasPrefix<PossiblePrefix: Sequence>(_ possiblePrefix: PossiblePrefix) -> Bool where ...` |
| `starts<PossiblePrefix : Sequence>(with possiblePrefix: PossiblePrefix, by areEquivalent: @noescape (Iterator.Element, Iterator.Element) throws -> Bool) rethrows -> Bool where ...` | `hasPrefix<PossiblePrefix : Sequence>(_ possiblePrefix: PossiblePrefix, by areEquivalent: @noescape (Iterator.Element, Iterator.Element) throws -> Bool) rethrows -> Bool where ...` |
| `index(of element: Iterator.Element) -> Index?`   | `firstIndex(of element: Iterator.Element) -> Index?` |
| `index(where predicate: @noescape (Iterator.Element) throws -> Bool) rethrows -> Index?` | `firstIndex(where predicate: @noescape (Iterator.Element) throws -> Bool) rethrows -> Index?` |

### Index-based operations

We will first present an idealized version of the design which cannot be 
fully implemented in Swift 3 due to generics bugs and limitations. Then 
we will present the minor changes necessary to implement it. The 
two should be source-compatible unless users conform their own types to 
`RangeExpression`.

#### Idealized design

The `RangeExpression` protocol is defined like so:

```swift
/// A type which can be used to slice a collection. A `RangeExpression` can 
/// convert itself to a `Range<Bound>` of indices within a given collection; 
/// the collection can then slice itself with that `Range`.
public protocol RangeExpression {
  /// Returns `self` expressed as a range of indices within `collection`.
  /// 
  /// -Parameter collection: The collection `self` should be 
  ///                        relative to.
  /// 
  /// -Returns: A `Range<Bound>` suitable for slicing `collection`. 
  ///           The return value is *not* guaranteed to be inside 
  ///           its bounds. Callers should apply the same preconditions 
  ///           to the return value as they would to a range provided 
  ///           directly by the user.
  public func relative<C: Collection>(to collection: C) -> Range<Bound> where C.Index == Bound {
}
```

The following existing types will be conformed to `RangeExpressible`:

* `Range`
* `CountableRange`
* `ClosedRange`
* `CountableClosedRange`

The `Range` conformance is not strictly necessary, but allows APIs 
which do not need to be overridden to implement only a `RangeExpression`-based
variant. Type inference favors concrete members over generic ones, so 
it should prefer to use parameters explicitly typed as `Range<Index>` 
over parameters of a generic type constrained to 
`RangeExpression where Bound == Index`.

The `Indexable` and `MutableIndexable` subscripts which take range types 
other than `Range` itself will be removed. So will the 
`RangeReplaceableCollection` subscripts which take range types other than 
`Range`. Instead, they will be replaced with single generic versions 
taking a `RangeExpression where Bound == Index`, using `relative(to:)` 
to convert them to `Range`s and then calling through to the plain `Range` 
variants:

```swift
extension Collection {
  /// Accesses a contiguous subrange of the collection's elements.
  ///
  /// The accessed slice uses the same indices for the same elements as the
  /// original collection. Always use the slice's `startIndex` property
  /// instead of assuming that its indices start at a particular value.
  ///
  /// This example demonstrates getting a slice of an array of strings, finding
  /// the index of one of the strings in the slice, and then using that index
  /// in the original array.
  ///
  ///     let streets = ["Adams", "Bryant", "Channing", "Douglas", "Evarts"]
  ///     let streetsSlice = streets[2 ..< streets.endIndex]
  ///     print(streetsSlice)
  ///     // Prints "["Channing", "Douglas", "Evarts"]"
  ///
  ///     let index = streetsSlice.index(of: "Evarts")    // 4
  ///     print(streets[index!])
  ///     // Prints "Evarts"
  ///
  /// - Parameter bounds: A range of the collection's indices. The bounds of
  ///   the range must be valid indices of the 
  public subscript<R>(bounds: R) -> SubSequence where R: RangeExpression, R.Bound == Index {
    get {
      return self[bounds.relative(to: self)]
    }
  }
}
extension MutableCollection {
  /// Accesses a contiguous subrange of the collection's elements.
  ///
  /// The accessed slice uses the same indices for the same elements as the
  /// original collection. Always use the slice's `startIndex` property
  /// instead of assuming that its indices start at a particular value.
  ///
  /// This example demonstrates getting a slice of an array of strings, finding
  /// the index of one of the strings in the slice, and then using that index
  /// in the original array.
  ///
  ///     let streets = ["Adams", "Bryant", "Channing", "Douglas", "Evarts"]
  ///     let streetsSlice = streets[2 ..< streets.endIndex]
  ///     print(streetsSlice)
  ///     // Prints "["Channing", "Douglas", "Evarts"]"
  ///
  ///     let index = streetsSlice.index(of: "Evarts")    // 4
  ///     streets[index!] = "Eustace"
  ///     print(streets[index!])
  ///     // Prints "Eustace"
  ///
  /// - Parameter bounds: A range of the collection's indices. The bounds of
  ///   the range must be valid indices of the collection.
  public subscript<R>(bounds: R) -> SubSequence where R: RangeExpression, R.Bound == Index {
    get {
      return self[bounds.relative(to: self)]
    }
    set {
      self[bounds.relative(to: self)] = newValue
    }
  }
}
extension RangeReplaceableCollection {
  /// Replaces the specified subrange of elements with the given collection.
  ///
  /// This method has the effect of removing the specified range of elements
  /// from the collection and inserting the new elements at the same location.
  /// The number of new elements need not match the number of elements being
  /// removed.
  ///
  /// In this example, three elements in the middle of an array of integers are
  /// replaced by the five elements of a `Repeated<Int>` instance.
  ///
  ///      var nums = [10, 20, 30, 40, 50]
  ///      nums.replaceSubrange(1...3, with: repeatElement(1, count: 5))
  ///      print(nums)
  ///      // Prints "[10, 1, 1, 1, 1, 1, 50]"
  ///
  /// If you pass a zero-length range as the `subrange` parameter, this method
  /// inserts the elements of `newElements` at `subrange.startIndex`. Calling
  /// the `insert(contentsOf:at:)` method instead is preferred.
  ///
  /// Likewise, if you pass a zero-length collection as the `newElements`
  /// parameter, this method removes the elements in the given subrange
  /// without replacement. Calling the `removeSubrange(_:)` method instead is
  /// preferred.
  ///
  /// Calling this method may invalidate any existing indices for use with this
  /// collection.
  ///
  /// - Parameters:
  ///   - subrange: The subrange of the collection to replace. The bounds of
  ///     the range must be valid indices of the collection.
  ///   - newElements: The new elements to add to the collection.
  ///
  /// - Complexity: O(*m*), where *m* is the combined length of the collection
  ///   and `newElements`. If the call to `replaceSubrange` simply appends the
  ///   contents of `newElements` to the collection, the complexity is O(*n*),
  ///   where *n* is the length of `newElements`.
  public mutating func replaceSubrange<R, C>(
    _ subrange: R,
    with newElements: C
  ) where R : RangeExpression, R.Bound == Index, C : Collection, C.Iterator.Element == Iterator.Element {
    replaceSubrange(subrange.relative(to: self), with: newElements)
  }

  /// Removes the elements in the specified subrange from the collection.
  ///
  /// All the elements following the specified position are moved to close the
  /// gap. This example removes two elements from the middle of an array of
  /// measurements.
  ///
  ///     var measurements = [1.2, 1.5, 2.9, 1.2, 1.5]
  ///     measurements.removeSubrange(1..<3)
  ///     print(measurements)
  ///     // Prints "[1.2, 1.5]"
  ///
  /// Calling this method may invalidate any existing indices for use with this
  /// collection.
  ///
  /// - Parameter bounds: The range of the collection to be removed. The
  ///   bounds of the range must be valid indices of the collection.
  ///
  /// - Complexity: O(*n*), where *n* is the length of the collection.
  public mutating func removeSubrange<R>(_ bounds: R) where R : RangeExpression, R.Bound == Index {
    removeSubrange(bounds.relative(to: self))
  }
}
```

The `Collection`- and `RangeReplaceableCollection`-mimicking APIs on 
`String` will be similarly replaced with `RangeExpression`-based 
versions, except that here the `Range` versions will be removed too, 
so all calls will be through `RangeExpression`.

The `IncompleteRange` and `IncompleteClosedRange` types will be very 
similar. They are designed to be useful generally, not merely with 
`RangeExpressible`. Below is the interface for `IncompleteRange`; 
`IncompleteClosedRange` would be analogous.

```swift
prefix operator ..< {}
postfix operator ..< {}

/// A `Range` which may not have all of its bounds specified.
/// The `IncompleteRange` can be completed by providing a 
/// `Range` or `CountableRange` from which it can retrieve 
/// default upper and lower bounds.
public struct IncompleteRange<Bound : Comparable> {
  /// The lowest value within the range. 
  /// If `nil`, completing the range will adopt the default value's 
  /// `lowerBound`.
  public let lowerBound: Bound?
  /// The value just above the highest value within the range.
  /// If `nil`, completing the range will adopt the default value's 
  /// `upperBound`.
  public let upperBound: Bound?
}

extension IncompleteRange {
  /// Returns a `Range` with the same `upperBound` 
  /// and `lowerBound` as the current instance. `nil` bounds are 
  /// filled in from `defaultBounds`.
  /// 
  /// This method does not check whether `lowerBound` and `upperBound` 
  /// lie within `defaultBounds`. 
  public func completed(by defaultBounds: Range<Bound>) -> Range<Bound>
  
  /// Returns a `Range` with the same `upperBound` 
  /// and `lowerBound` as the current instance. `nil` bounds are 
  /// filled in from `defaultBounds`.
  /// 
  /// This method does not check whether `lowerBound` and `upperBound` 
  /// lie within `defaultBounds`. 
  /// Nor does it check whether the resulting `lowerBound` is below 
  /// its `upperBound`.
  public func completed(byUnchecked defaultBounds: Range<Bound>) -> Range<Bound>
}

extension IncompleteRange where
    Bound : Strideable,
    Bound.Stride : SignedInteger {
  /// Returns a `CountableRange` with the same `upperBound` 
  /// and `lowerBound` as the current instance. `nil` bounds are 
  /// filled in from `defaultBounds`.
  /// 
  /// This method does not check whether `lowerBound` and `upperBound` 
  /// lie within `defaultBounds`. 
  public func completed(by defaultBounds: CountableRange<Bound>) -> CountableRange<Bound>
  
  /// Returns a `CountableRange` with the same `upperBound` 
  /// and `lowerBound` as the current instance. `nil` bounds are 
  /// filled in from `defaultBounds`.
  /// 
  /// This method does not check whether `lowerBound` and `upperBound` 
  /// lie within `defaultBounds`. 
  /// Nor does it check whether the resulting `lowerBound` is below 
  /// its `upperBound`.
  public func completed(byUnchecked defaultBounds: CountableRange<Bound>) -> CountableRange<Bound>
}

extension IncompleteRange: RangeExpression { ... }

/// Constructs an `IncompleteRange` with the provided upper 
/// bound and an unknown lower bound.
public prefix func ..< <Bound: Comparable>(upperBound: Bound) -> IncompleteRange<Bound>

/// Constructs an `IncompleteRange` with the provided lower 
/// bound and an unknown upper bound.
public postfix func ..< <Bound: Comparable>(lowerBound: Bound) -> IncompleteRange<Bound>

/// Constructs an `IncompleteRange` with the provided upper 
/// and lower bounds. Either or both may be `nil`, in which case the 
/// bound will be provided when the `IncompleteRange` is 
/// completed.
public func ..< <Bound: Comparable>(lowerBound: Bound?, upperBound: Bound?) -> IncompleteRange<Bound>
```

Finally, since they are now redundant, `prefix(upTo:)`, 
`prefix(through:)`, and `suffix(from:)` will be removed.

#### Actual design

The actual design varies from the ideal one in four ways:

1. The `CountableRange` variants of `completed(by:)` require a slightly 
   different set of constraints to match a workaround on those types.

2. Swift 3 does not support generic subscripts, so we must instead 
   generate subscripts for each known `RangeExpression` type.

3. Because of the complex way `Collection`'s protocols are layered, it 
   is necessary to attach subscripts to `Indexable` and `MutableIndexable`, 
   and constrain `relative(to:)`'s parameter to `Indexable`, instead of 
   `Collection` and `MutableCollection`.

4. Swift currently has trouble with the constraints on 
   `RangeExpression.relative(to:)` if it is a protocol requirement.
   Thus, it is instead provided as an extension method. A method with 
   simpler generic constraints is instead used as the requirement.

These will not affect source compatibility except when a user conforms 
their own types to `RangeExpression`, so we suggest we place warnings in 
the documentation, effectively pre-deprecating its required method.

The actual design of `RangeExpression` is thus as follows:

```swift
/// A type which can be used to slice a collection. A `RangeExpression` can 
/// convert itself to a `Range<Bound>` of indices within a given collection; 
/// the collection can then slice itself with that `Range`.
/// 
/// -Warning: The requirements of `RangeExpression` are likely to change 
///           in a future version of Swift. If you conform your own 
///           types to `RangeExpression`, be prepared to migrate them.
public protocol RangeExpression {
  /// The type of the bounds of the `Range` produced by this 
  /// type when used as a `RangeExpression`.
  associatedtype Bound : Comparable
  
  /// Returns `self` expressed as a `Range<Bound>` suitable for 
  /// slicing a collection with the indicated properties.
  /// 
  /// -Parameter bounds: The range of indices in the collection.
  ///                    Equivalent to `startIndex ..< endIndex`
  ///                    in `Collection`.
  /// 
  /// -Parameter offset: A function which can be used to add to or 
  ///                    subtract from a bound. Equivalent to 
  ///                    `index(_:offsetBy:)` in `Collection`.
  /// 
  /// -Returns: A `Range<Bound>` suitable for slicing a collection. 
  ///           The return value is *not* guaranteed to be inside 
  ///           `bounds`. Callers should apply the same preconditions 
  ///           to the return value as they would to a range provided 
  ///           directly by the user.
  /// 
  /// -Warning: This method is likely to be replaced in a future version of Swift.
  ///           If you are calling this method, we recommend using the 
  ///           `relative(to:)` extension method instead. If you are implementing 
  ///           it, be prepared to migrate your code.
  /// 
  /// -Recommended: `relative(to:)`
  // 
  // WORKAROUND unfiled - We want to have this requirement, but it triggers a generics bug
  // func relative<C: Indexable>(to collection: C) -> Range<Bound> where C.Index == Bound
  func relative<BoundDistance: SignedInteger>(to bounds: Range<Bound>, offsettingBy offset: (Bound, BoundDistance) -> Bound) -> Range<Bound>
}

extension RangeExpression {
  /// Returns `self` expressed as a range of indices within `collection`.
  /// 
  /// -Parameter collection: The collection `self` should be 
  ///                        relative to.
  /// 
  /// -Returns: A `Range<Bound>` suitable for slicing `collection`. 
  ///           The return value is *not* guaranteed to be inside 
  ///           its bounds. Callers should apply the same preconditions 
  ///           to the return value as they would to a range provided 
  ///           directly by the user.
  /// 
  /// -RecommendedOver: `relative(to:offsettingBy:)`
  public func relative<C: Indexable>(to collection: C) -> Range<Bound> where C.Index == Bound {
    let bounds = Range(uncheckedBounds: (lower: collection.startIndex, upper: collection.endIndex))
    return relative(to: bounds, offsettingBy: collection.index(_:offsetBy:))
  }
}
```

A full working prototype of `RangeExpression` and the `IncompleteRange` 
types is available [in this pull request](https://github.com/apple/swift/pull/3737).

## Impact on existing code

Obviously, any code using these APIs under their old names or designs 
would have to be transitioned to the new names and designs.

The sequence-end operations would be by far the simplest to handle; 
these are simple renamings and could be handed by 
`@available(renamed:)` and migration support. The only complication 
is that some overloads have transitioned to a new base name, while 
others have stayed with the old one, but we suspect the migrator is up 
to this task.

The index-based operations are more difficult to migrate. The patterns 
would be roughly:

```swift
collection.prefix(upTo: i)    => collection[..<i]
collection.prefix(through: i) => collection[...i]
collection.suffix(from: i)    => collection[i..<]
```

A custom fix-it would be ideal, but is probably not necessary; an 
`@available(message:)` would do. Presumably this would have to be a 
special case in the migrator as well.

The other changes to the handling of subranges are source-compatible.

## Alternatives considered

#### `skipping` instead of `removing`

If the type differences are seen as disqualifying `removing` as a 
replacement for `drop`, we suggest using `skipping` instead.

There are, of course, *many* possible alternatives to `skipping`; this 
is almost a perfect subject for bikeshedding. We've chosen `skipping` 
because:

1. It is not an uncommon word, unlike (say) `omitting`. This means 
   non-native English speakers and schoolchildren are more likely to 
   recognize it.

2. It is an -ing verb, unlike (say) `without`. This makes it fit common 
   Swift naming patterns more closely.

3. It does not imply danger, unlike (say) `dropping`, nor some sort of 
   ongoing process, unlike (say) `ignoring`. This makes its behavior 
   more obvious.

If you want to suggest an alternative on swift-evolution, please do not 
merely mention a synonym; rather, explain *why* it is an improvement on 
either these axes or other ones. (We would be particularly interested in 
names other than `removing` which draw an analogy to something else in 
Swift.)

#### `collection[to/through/from:]` instead of `IncompleteRange`

Rather than add new types and operators to replace 
`prefix(upTo/through:)` and `suffix(from:)`, we could merely transform 
these methods into subscripts with parameter labels. This would be a 
simpler design, but these terms have proven imperfect in the `stride` 
calls, and labeled subscripts are rare (actually, we believe they're 
unprecedented in the standard library).

#### `longestPrefix(where:)` instead of `prefix(while:)`

The name `prefix(while:)` isn't perfect; it seems to imply more state 
than is really involved. (There *is* a stateful loop, but it's 
contained within the method.) A name like `longestPrefix(where:)` might 
read better and avoid this implication, but we think it's important that 
`prefix(_:)` and `prefix(while:)` be given parallel names, and 
`longestPrefix(3)` doesn't make much sense.

#### No `RangeExpression` protocol

The `RangeExpression` protocol could be severed from this proposal, but 
this seems like a good opportunity to refactor our subrange handling.

### Other alternatives

* Rather than using `first` and `last` for the "First matching" and 
  "Last matching" categories, we could use a distinct term. These 
  methods have different performance characteristics than the others, 
  and modeling that might be helpful. However, it's difficult to find 
  a good term—an earlier version of this proposal used `earliest` and 
  `latest`, which don't read perfectly—and the level of confusion is 
  pretty low.

* We considered using `first` and `last` as the basis for both 
  single-element and multiple-element operations (such that `prefix(3)` 
  would become `first(3)`, etc.), but:
  
  1. These seemed like distinct functionalities, particularly since 
     their types are different.
  
  2. We're not comfortable with heavily overloading a property with a 
     bunch of methods, and didn't want to make `first` and `last` into 
	 methods.
  
  3. Most APIs work fine, but `hasFirst(_:)` is atrocious, and we see 
     no better alternative which includes the word `first`.

* We considered moving `first` and `last` to `Sequence` and possibly 
  making them methods, but our understanding is that the core team has 
  considered and rejected this approach in the past.

* We considered moving `removingFirst` and `removingLast` to Collection 
  and making them properties, to match `first` and `last`, but this 
  seemed like the sort of foolish consistency that Ralph Waldo Emerson 
  warned of.

## Future directions

**Note**: The rest of this proposal is *highly* speculative and there's 
probably no need to read further.

### Other Sequence API cleanups

#### Seriously source-breaking

* There is an ongoing discussion about which, if any, of `map`, 
  `flatMap`, `filter`, and `reduce` ought to be renamed to more closely 
  match Swift naming conventions. There is also discussion about 
  relabeling many closure parameters.
  
  The "Future directions" section below suggests `every(where:)` as an 
  alternative to `filter` which could be extended in ways compatible 
  with this proposal.

#### Significantly source-breaking

* The `removeSubrange(_:)` and `replaceSubrange(_:with:)` APIs are 
  rather oddly named. They might be better renamed to, for instance, 
  `remove(in:)` and `replace(in:with:)`.

* It is not clear how important `removingFirst()` and `removingLast()` 
  actually are, given that they're exactly equivalent to 
  `removingPrefix(1)` and `removingSuffix(1)`, and their corresponding 
  "get" members are on `Collection` instead of `Sequence`. They could 
  be removed.

#### Slightly source-breaking

* `removeFirst/Last()` and `popFirst/Last()` are very nearly redundant; 
  their only difference is that the `remove` methods have a 
  non-`Optional` return type and require the collection not be empty, 
  while the `pop` methods have an `Optional` return type and return 
  `nil` if it's empty.
  
  These operations could be merged, with the `remove` operations taking 
  on the preconditions of the current `pop` operations; additionally, 
  `removePrefix(_:)` and `removeSuffix(_:)` could drop their equivalent 
  preconditions requiring that the elements being removed exist. These 
  changes would simplify the standard library and make these methods 
  more closely parallel the equivalent `removing` methods, which do 
  not have similar preconditions.
  
  Performance-critical code which wants to avoid the checks necessary 
  to remove these preconditions could switch to `remove(at:)` and 
  `removeSubrange(_:)`, which would continue to reject invalid indices.

### Adding sequence and collection operations

This exercise in renaming suggests all sorts of other APIs we might 
add, and a few we might rename.

In general, we have not attempted to carefully scrutinize the usefulness 
of each of these APIs; instead, we have merely listed the ones which we 
can imagine some kind of use for. The main exception is the "Pop" 
operation; we can imagine several different, and rather incompatible, 
ways to extend it, and we're not going to take the time to sort out our 
thoughts merely to write a "Future directions" section.

#### Filling in the sequence-end API table

The gaps in the table suggest a number of APIs we could offer in the 
future. Here, we have filled in all options which are at least coherent:

|                                  | Get                  | Index                     | Exclude                        | Remove (1)                   | Pop (1)      | Equate (2)              |
| -------------------------------- | -------------------- | ------------------------- | ------------------------------ | ---------------------------- | ------------ | ----------------------- |
| **Fixed Size**                   |                                                                                                                                                          
| First 1                          | C.first              | **C.firstIndex**          | S.removingFirst()              | C.removeFirst()              | C.popFirst() | -                       |
| Last 1                           | C.last               | **C.lastIndex**           | S.removingLast()               | C.removeLast()               | C.popLast()  | -                       |
| First (n: Int)                   | S.prefix(\_:)        | **C.prefixIndex(\_:)**    | S.removingPrefix(\_:)          | C.removePrefix(\_:)          | -            | S.hasPrefix(\_:)        |
| &nbsp;&nbsp;...with closure      | S.prefix(while:)     | **C.prefixIndex(while:)** | S.removingPrefix(while:)       | **C.removePrefix(while:)**   | -            | S.hasPrefix(\_:by:)     |
| Last (n: Int)                    | S.suffix(\_:)        | **C.suffixIndex(\_:)**    | S.removingSuffix(\_:)          | C.removeSuffix(\_:)          | -            | **S.hasSuffix(\_:)**    |
| &nbsp;&nbsp;...with closure      | **S.suffix(while:)** | **C.suffixIndex(while:)** | **S.removingSuffix(while:)**   | **C.removeSuffix(while:)**   | -            | **S.hasSuffix(\_:by:)** |
| **Searching From End**           |
| First&nbsp;matching&nbsp;element | **S.first(\_:)**     | C.firstIndex(of:)         | **S.removingFirst(\_:)**       | **C.removeFirst(\_:)**       | -            | -                       |
| &nbsp;&nbsp;...with closure      | S.first(where:)      | C.firstIndex(where:)      | **S.removingFirst(where:)**    | **C.removeFirst(where:)**    | -            | -                       |
| Last matching element            | **S.last(\_:)**      | **C.lastIndex(of:)**      | **S.removingLast(\_:)**        | **C.removeLast(\_:)**        | -            | -                       |
| &nbsp;&nbsp;...with closure      | **S.last(where:)**   | **C.lastIndex(where:)**   | **S.removingLast(where:)**     | **C.removeLast(where:)**     | -            | -                       |

To explain a few entries which might not be immediately obvious: 
`firstIndex` and `lastIndex` would be `nil` if the collection is empty, 
and `lastIndex` would be the index before `endIndex`. `prefixIndex` 
would return the last index of the prefix, and `suffixIndex` would 
return the first index of the suffix; alternatively, these could be 
named with `Indices` and return ranges. `first(_:)` and `last(_:)` 
would return the first and last element equal to the provided value; on 
a `Set`, they would be roughly equivalent to `NSSet.member(_:)`.

The changes we consider most worthy include:

* Adding corresponding `last` and `suffix` methods for all 
  `first` and `prefix` methods.
  
* Adding corresponding `while:` versions of all appropriate 
  prefix/suffix APIs.

Ones that could be useful, but can usually be emulated with more work:

* Adding `remove`/`removing`-by-content APIs.

* Adding `prefix/suffixIndex(while:)`.

Ones that are mere conveniences or may not have strong use cases:

* `first/lastIndex` and `prefix/suffixIndex(_:)`.

* `first/last(_:)`.

#### "All" and "Every" as operands

One could imagine adding rows to this table for "all" and "every 
matching". In addition to creating some useful new API, this would 
also suggest some interesting renaming for existing APIs:

* `allIndices` would be a name for `indices`.

* `removeAll()` is actually an existing name which happens to fit this 
  pattern.

* `every(where:)` would be a name for `filter`. Though some of us 
  believe `filter` is a strong term of art, we do note that 
  `every(where:)` does not cause confusion about the sense of its test, 
  a major complaint about `filter`.

In the table below, **bold** indicates new functionality; *italics* 
indicates existing functionality renamed to fit this pattern.

|                                  | Get                | Index                    | Exclude                     | Remove (1)                | Pop (1)      | Equate (2)          |
| -------------------------------- | ------------------ | ------------------------ | --------------------------- | ------------------------- | ------------ | ------------------- |
| **Fixed Size**                   |
| First 1                          | C.first            | C.firstIndex             | S.removingFirst()           | C.removeFirst()           | C.popFirst() | -                   |
| Last 1                           | C.last             | C.lastIndex              | S.removingLast()            | C.removeLast()            | C.popLast()  | -                   |
| First (n: Int)                   | S.prefix(\_:)      | C.prefixIndex(\_:)       | S.removingPrefix(\_:)       | C.removePrefix(\_:)       | -            | S.hasPrefix(\_:)    |
| &nbsp;&nbsp;...with closure      | S.prefix(while:)   | C.prefixIndex(while:)    | S.removingPrefix(while:)    | C.removePrefix(while:)    | -            | S.hasPrefix(\_:by:) |
| Last (n: Int)                    | S.suffix(\_:)      | C.suffixIndex(\_:)       | S.removingSuffix(\_:)       | C.removeSuffix(\_:)       | -            | S.hasSuffix(\_:)    |
| &nbsp;&nbsp;...with closure      | S.suffix(while:)   | C.suffixIndex(while:)    | S.removingSuffix(while:)    | C.removeSuffix(while:)    | -            | S.hasSuffix(\_:by:) |
| All                              | -                  | *allIndices*             | -                           | C.removeAll()             | -            | -                   |
| **Searching From End**           |
| First&nbsp;matching&nbsp;element | S.first(\_:)       | C.firstIndex(of:)        | S.removingFirst(\_:)        | C.removeFirst(\_:)        | -            | -                   |
| &nbsp;&nbsp;...with closure      | S.first(where:)    | C.firstIndex(where:)     | S.removingFirst(where:)     | C.removeFirst(where:)     | -            | -                   |
| Last matching element            | S.last(\_:)        | C.lastIndex(of:)         | S.removingLast(\_:)         | C.removeLast(\_:)         | -            | -                   |
| &nbsp;&nbsp;...with closure      | S.last(where:)     | C.lastIndex(where:)      | S.removingLast(where:)      | C.removeLast(where:)      | -            | -                   |
| Every&nbsp;matching&nbsp;element | **S.every(\_:)**   | **C.everyIndex(of:)**    | **S.removingEvery(\_:)**    | **C.removeEvery(\_:)**    | -            | -                   |
| &nbsp;&nbsp;...with closure      | *S.every(where:)*  | **C.everyIndex(where:)** | **S.removingEvery(where:)** | **C.removeEvery(where:)** | -            | -                   |

An alternative to the `every` methods is to give them names based on 
`all` or `any`, but these tend to require breaks from the naming 
patterns of the matching `first` and `last` methods to remain 
grammatical.
