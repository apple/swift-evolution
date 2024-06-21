# Remove `Sequence.SubSequence`

* Proposal: [SE-0234](0234-remove-sequence-subsequence.md)
* Authors: [Ben Cohen](https://github.com/airspeedswift)
* Review Manager: [John McCall](https://github.com/rjmccall)
* Status: **Implemented (Swift 5.0)**
* Implementation: [apple/swift#20221](https://github.com/apple/swift/pull/20221)
* Review: ([review thread](https://forums.swift.org/t/se-0234-removing-sequence-subsequence/17750)) ([acceptance](https://forums.swift.org/t/accepted-se-0234-remove-sequence-subsequence/18002))

## Introduction

This proposal recommends eliminating the associated type from `Sequence`,
moving it up to start at `Collection`. Current customization points on
`Sequence` returning a `SubSequence` will be amended to be extensions returning
concrete types.

Swift-evolution thread: [Discussion thread topic for that proposal](https://forums.swift.org/t/rationalizing-sequence-subsequence/17586)

## Motivation

### Current usage

Today, `Sequence` declares several methods that return a `SubSequence`:

```swift
func dropFirst(_:) -> SubSequence
func dropLast(_:) -> SubSequence
func drop(while:) -> SubSequence
func prefix(_:) -> SubSequence
func prefix(while:) -> SubSequence
func suffix(_:) -> SubSequence
func split(separator:) -> [SubSequence]
```

You don't have to implement them to implement a `Sequence`. They all have default implementations for the default type for `SubSequence`.

But if you think about _how_ you'd implement them generically on a single-pass sequence, you'll quickly realize there is a problem. They all call for completely different return types in their implementation. For example, the ideal way to implement `dropFirst` would be to return a wrapper type that first drops n elements, then starts returning values. `prefix` would ideally be implemented the other way around: return elements until you've returned n, then stop. `suffix` and `dropLast` need to consume the entire sequence to get to the end, buffering as they go, then return what they buffered. `drop(while:)` needs to eagerly drop non-matching elements as soon as it's called (because the closure is not `@escaping`), which means it needs to buffer one element in case that's the first one it needs to return later. `prefix(while:)` also needs to eagerly search and buffer _everything_ it reads.

But the protocol requires all these methods return the same type – `SubSequence`. They can't return specific types for their specific needs. In theory, that could be resolved by having them all return `[Element]`, but that would be wasteful of memory in cases like `prefix(_:)`.

The way the std lib works around this is to make the default `SubSequence` an `AnySequence<Element>`. So internally, there is a `DropFirstSequence` type, that is created when you call `dropFirst` on a `Sequence`. But it is type-erased to be the same type as returned by `prefix`, `suffix` etc., which also return their own custom types, but type erased.

Unfortunately this has two major consequences:
- performance is bad: type-erased wrappers are an optimization barrier; and
- it blocks conditional conformance going from `Sequence` to `Collection`.

Additionally, it makes implementing your own custom `SubSequence` that _isn't_ `AnySequence` extremely hard, because you need to then implement your own version of all these methods, even `split`. So in practice, this is never done.

### Type erasure performance

There is a prototype in [this PR](https://github.com/apple/swift/pull/20175) that replaces the customization points on `Sequence` with regular extensions that each return a specific type.

To see the performance problem, here is how the [benchmarks](https://github.com/apple/swift/pull/20175#issuecomment-434661846) improve if you return a non-type-erased type instead:

#### Performance: -O

TEST                          | OLD  | NEW   | DELTA      | RATIO     
---                           | ---  | ---   | ---        | ---       
**Improvement**               |      |       |            |           
DropWhileSequence             | 2214 | 29    | -98.7%     | **76.34x** 
PrefixSequenceLazy            | 2265 | 52    | -97.7%     | **43.56x** 
PrefixSequence                | 2213 | 52    | -97.7%     | **42.56x** 
DropFirstSequenceLazy         | 2310 | 59    | -97.4%     | **39.15x** 
DropFirstSequence             | 2240 | 59    | -97.4%     | **37.97x** 

#### Performance: -Osize

TEST                          | OLD   | NEW   | DELTA   | RATIO     
---                           | ---   | ---   | ---     | ---       
**Improvement**               |       |       |         |           
DropWhileAnySeqCRangeIter     | 17631 | 163   | -99.1%  | **108.16x** 
DropFirstAnySeqCRangeIterLazy | 21259 | 213   | -99.0%  | **99.81x** 
PrefixAnySeqCRangeIterLazy    | 16679 | 176   | -98.9%  | **94.77x** 
PrefixAnySeqCntRangeLazy      | 15810 | 168   | -98.9%  | **94.11x** 
DropFirstAnySeqCntRangeLazy   | 15717 | 213   | -98.6%  | **73.79x** 
DropWhileSequence             | 2582  | 35    | -98.6%  | **73.77x** 
DropFirstSequenceLazy         | 2671  | 58    | -97.8%  | **46.05x** 
DropFirstSequence             | 2649  | 58    | -97.8%  | **45.67x** 
PrefixSequence                | 2705  | 70    | -97.4%  | **38.64x** 
PrefixSequenceLazy            | 2670  | 70    | -97.4%  | **38.14x** 

These performance improvements are all down to how well the optimizer can eliminate the wrapper abstractions  when there isn’t the barrier of type erasure in the way. In -Onone builds, you don’t see any speedup.

### How does it block conditional conformance?

The problem with `SubSequence` really became clear when conditional conformance
was implemented. With conditional conformance, it becomes really important that
an associated type be able to grow and take on capabilities that line up with
the capabilities you are adding with each new conformance.

For example, the `Slice` type that is the default `SubSequence` for
`Collection` grows in capabilities as it’s base grows. So for example, if the
`Base` is a `RandomAccessCollection`, then so can the `Slice` be. This then
works nicely when you add new conformances to a `Collection` that _uses_
`Slice` as it’s `SubSequence` type. For more detail on this, watch Doug’s
explanation in our WWDC [Swift
Generics](https://developer.apple.com/videos/play/wwdc2018/406/?time=1614) talk
(starts at about minute 26).

But the default type for `Sequence.SubSequence` is `AnySequence`, which is a
conformance dead end. You cannot add additional capabilities to `AnySequence`
because there is nothing to drive them: the type erases all evidence of it’s
wrapped type – that’s it’s point.

This in turn forces two implementations of types that would ideally have a
single unified implementation. For example, suppose you wanted to write
something similar to `EnumeratedSequence` from the standard library, but have
it be a `Collection` as well when it could support it.

First you start with the basic type (note, all this code takes shortcuts for
brevity):

```swift
struct Enumerated<Base: Sequence> {
  let _base: Base
}
extension Sequence {
  func enumerated() -> Enumerated<Self> {
    return Enumerated(_base: self)
  }
}
```

And add `Sequence` conformance:

```swift
extension Enumerated: Sequence {
  typealias Element = (Int,Base.Element)
  struct Iterator: IteratorProtocol {
    var _count: Int
    let _base: Base.Iterator
    mutating func next() -> Element? {
      defer { _count += 1 }
      return _base.next().map { (_count,$0) }
    }
  }
  func makeIterator() -> Enumerated<Base>.Iterator {
    return Iterator(_count: 0, _base: _base.makeIterator())
  }
}
```

Then, you’d want to add `Collection` conformance when the underlying base is
also a collection. Something like this:

```swift
extension Enumerated: Collection where Base: Collection {
  struct Index: Comparable {
    let _count: Int, _base: Base.Index
    static func < (lhs: Index, rhs: Index) -> Bool {
      return lhs._base < rhs._base
    }
  }
  var startIndex: Index { return Index(_count: 0, _base: _base.startIndex) }
  var endIndex: Index { return Index(_count: Int.max, _base: _base.endIndex) }
  subscript(i: Index) -> Element {
    return (i._count, _base[i._base])
  }
}
```

You’d then follow through with conformance to `RandomAccessCollection` too when
the base was. You can see this pattern used throughout the standard library.

But this code won’t compile. The reason is that `Collection` requires that
`SubSequence` also be a collection (and `BidirectionalCollection` requires it
be bi-directional, and so on). This all works perfectly for `Slice`, the
default value for `SubSequence` from `Collection` down, which progressively
acquires these capabilities and grows along with the protocols it supports. But
`AnySequence` can’t, as described above. It blocks all further capabilities.

Because of this, if you want to support collection-like behavior, you’re back
to the bad old days before conditional conformance. You have to declare two
separate types: `EnumeratedSequence` and `EnumeratedCollection`, and define the
`enumerated` function twice. This is bad for code size, and also leaks into
user code, where these two different types appear.

### Why is `Sequence` like this?

The reason why `Sequence` declares this associated type and then forces all
these requirements to return it is to benefit from a specific use case: writing
a generic algorithm on `Sequence`, and then passing a `Collection` to it.
Because these are customization points, when `Collection` is able to provide a
better implementation, that generic algorithm can benefit from it.

For example, suppose you pass an `Array`, which provides random-access, into an
algorithm that then calls `suffix`. Instead of needing to buffer all the
elements, requiring linear time _and_ memory allocation, it can just return a
slice in constant time and no allocation.

You can see this in the regressions from the same PR:

#### Performance: -O

TEST                                | OLD   | NEW   | DELTA      | RATIO     
---                                 | ---   | ---   | ---        | ---       
**Regression**                      |       |       |            |           
DropLastAnySeqCntRangeLazy          | 9     | 20366 | +226163.8% | **0.00x** 
SuffixAnySeqCntRangeLazy            | 14    | 20699 | +147739.4% | **0.00x** 
DropLastAnySeqCntRange              | 9     | 524   | +5721.6%   | **0.02x** 
SuffixAnySeqCntRange                | 14    | 760   | +5328.2%   | **0.02x** 

#### Performance: -Osize

TEST                          | OLD   | NEW   | DELTA   | RATIO     
---                           | ---   | ---   | ---     | ---       
**Regression**                |       |       |         |           
DropLastAnySeqCRangeIterLazy  | 3684  | 20142 | +446.7% | **0.18x** 
SuffixAnySeqCRangeIterLazy    | 3973  | 20223 | +409.0% | **0.20x** 
SuffixAnySeqCntRangeLazy      | 5225  | 20235 | +287.3% | **0.26x** 
DropLastAnySeqCntRangeLazy    | 5256  | 20113 | +282.7% | **0.26x** 
DropFirstAnySeqCntRange       | 15730 | 20645 | +31.2%  | **0.76x** 

What is happening in these is a random-access collection (a `CountableRange`)
is being put inside the type-erasing `AnySequence`, then `suffix` or `dropLast`
is being called on it. The type-erased wrapper is then forwarding on the call
to the wrapped collection. Fetching the suffix of a countable range is
incredibly fast (it basically does nothing, ranges are just two numbers so it’s
just adjusting the lower bound upwards), whereas after removing the
customization points, the `suffix` function that’s called is the one for a
`Sequence`, which needs to iterate the range and buffer the elements.

This is a nice performance tweak, but it doesn’t justify the significant
downsides listed above. It is essentially improving generic performance at the
expense of concrete performance. Normally, these kind of generic improvements
come at just a tiny cost (e.g. slight increase in compile time or binary size)
rather than a significant runtime performance penalty.

## Proposed solution

Remove the `SubSequence` associated type from `Sequence`. It should first
appear from `Collection` onwards.

Remove the methods on `Sequence` that return `SubSequence` from the protocol.
They should remain as extensions only. Each one should return a specific type
best suited to the task:

```swift
extension Sequence {
  public func dropFirst(_ k: Int = 1) -> DropFirstSequence<Self>
  public func dropLast(_ k: Int = 1) -> [Element]
  public func suffix(_ maxLength: Int) -> [Element]
  public func prefix(_ maxLength: Int) -> PrefixSequence<Self>
  public func drop(while predicate: (Element) throws -> Bool) rethrows -> DropWhileSequence<Self>
  public func prefix(while predicate: (Element) throws -> Bool) rethrows -> [Element]
  public func split(
    maxSplits: Int = Int.max,
    omittingEmptySubsequences: Bool = true,
    whereSeparator isSeparator: (Element) throws -> Bool
  ) rethrows -> [ArraySlice<Element>]
}
```

`DropFirstSequence`, `DropWhileSequence` and `PrefixSequence` already exist in
the standard library in underscored form.

This will also have the useful side-effect that these methods can also be
removed as customization points from `Collection` as well, similar to removing
`prefix(upTo:)` in
[SE-0232](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0232-remove-customization-points.md),
because there’s no longer any reasonable customization to be done on a per-collection basis.

Doing this will have considerable benefits to code size as well. For example,
the CoreAudio overlay that declares a handful of collections reduces in size by
25% after applying these changes.

Once done, this change will allow the following simplifying changes and
enhancements to standard library types:

- `LazySequenceProtocol` and `LazyCollectionProtocol` can be collapsed into a
  single protocol.

- The following types can be collapsed, dropping the collection variant (with a
  typealias provided for source compatibility):

    - `FlattenSequence` and `-Collection`
    - `LazySequence` and `-Collection`
    - `LazyMapSequence` and `-Collection`
    - `LazyFilterSequence` and `-Collection`
    - `LazyDropWhileSequence` and `-Collection`
    - `LazyPrefixWhileSequence` and `-Collection`

- The following types can be extended to support `Collection`:

    - `EnumeratedSequence`
    - `JoinedSequence`
    - `Zip2Sequence`

`SubSequence` will continue to be an associated type on `Collection`, and the
equivalent take/drop methods (and split) will continue to return it. Once
the methods are removed from the `Sequence` protocol, they will also no longer
need to be customizable at the `Collection` level so can be extensions only.

## Source compatibility

This is a source-breaking change, in that any code that relies on
`Sequence.SubSequence` will no longer work. For example:

```swift
extension Sequence {
 func dropTwo() -> SubSequence {
   return dropFirst(2)
 }
}
```

There are no examples of this kind of code in the compatibility suite.
Unfortunately there is no way to remove an associated type in a way that only
affects a specific language version, so this would not be something you could
handle as part of upgrading to 5.0.

Additionally, any sequences that define a custom `SubSequence` of their own
will no longer work. This is really a non-problem, because doing so is almost
impossible (it means you even have to implement your own `split`).

## Effect on ABI stability

This is an ABI-breaking change, so must happen before 5.0 is released.

## Alternatives considered

Other than not doing it, a change that split `Sequence.SubSequence` into two
(say `Prefix` and `Suffix`) was considered. This implementation added
significant complexity to the standard library, impacting compile time and code
size, without being a significant enough improvement over the current situation.
