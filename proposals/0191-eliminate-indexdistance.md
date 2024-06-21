# Eliminate `IndexDistance` from `Collection`

* Proposal: [SE-0191](0191-eliminate-indexdistance.md)
* Author: [Ben Cohen](https://github.com/airspeedswift)
* Review Manager: [Doug Gregor](https://github.com/DougGregor)
* Status: **Implemented (Swift 4.1)**
* Implementation: [apple/swift#12641](https://github.com/apple/swift/pull/12641)
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-se-0191-eliminate-indexdistance-from-collection/7191)

## Introduction

Eliminate the associated type `IndexDistance` from `Collection`, and modify all uses to the concrete type `Int` instead.

## Motivation

`Collection` allows for the distance between two indices to be any `SignedInteger` type via the `IndexDistance` associated type. While in practice the distance between indices is almost always
an `Int`, generic algorithms on `Collection` need to either constrain `IndexDistance == Int` or write their algorithm to be generic over any `SignedInteger`.

Swift 4.0 introduced the ability to constrain associated types with `where` clauses
([SE-142](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0142-associated-types-constraints.md)) and will soon allow protocol constraints
to be recursive ([SE-157](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0157-recursive-protocol-constraints.md)). With these features,
writing generic algorithms against `Collection` is finally a realistic tool for intermediate Swift programmers. You no longer need to know to
constrain `SubSequence.Element == Element` or `SubSequence: Collection`, missing constraints that previously led to inexplicable error messages.

At this point, the presence of `IndexDistance` is of of the biggest hurdles that new users trying to write generic algorithms face. If you want to
write code that will compile against any distance type, you need to constantly juggle with explicit type annotations (i.e. you need to write `let i:
IndexDistance = 0` instead of just `let i = 0`), and perform `numericCast` to convert from one distance type to another.

But these `numericCasts` are hard to use correctly. Given two collections with different index distances, it's very hard to reason about whether your
`numericCast` is casting from the smaller to larger type correctly. This turns any problem of writing a generic collection algorithm into both a collection _and_
numeric problem. And chances are you are going to need to interoperate with a method that takes or provides a concrete `Int` anyway (like `Array.reserveCapacity` inside
`Collection.map`). Much of the generic code in the standard library would trap if ever presented with a collection with a distance greater than `Int.max`.
Additionally, this generalization makes specialization less likely and increases compile-time work.

For these reasons, it's common to see algorithms constrained to `IndexDistance == Int`. In fact, the inconvenience of having to deal with generic index
distances probably encourages more algorithms to be constrained to `Index == Int`, such as [this
code](https://github.com/airspeedswift/swift-package-manager/blob/472c647dcad3adf4344a06ef7ba91d2d4abddc94/Sources/Basic/OutputByteStream.swift#L119) in
the Swift Package Manager. Converting this function to work with any index type would be straightforward. Converting it to work with any index distance
as well would be much trickier.

The general advice from [The Swift Programming
Language](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/TheBasics.html#//apple_ref/doc/uid/TP40014097-CH5-ID309) when writing Swift code is to encourage users to stick to using `Int` unless they have a special reason not to:

> Unless you need to work with a specific size of integer, always use `Int` for integer values in your code. [...] `Int` is preferred, even when the values to be stored are known to be nonnegative. A consistent use of Int for integer values aids
code interoperability, avoids the need to convert between different number types, and matches integer type inference[.]

There are two main use cases for keeping `IndexDistance` as an associated type rather than concretizing it to be `Int`: tiny collections that might
benefit from tiny distances, and huge collections that need to address greater than `Int.max` elements. For example, it may seem wasteful to force a
type that presents the bits in a `UInt` as a collection to need to use a whole `Int` for its distance type. Or you may want to create a gigantic
collection, such as one backed by a memory mapped file, with a size great than `Int.max`. The most likely scenario for this is on 32-bit processors where a collection would be constrained to 2 billion elements.

These use cases are very niche, and do not seem to justify the considerable impedance to generic programming that `IndexDistance` causes. Therefore,
this proposal recommends removing the associated type and replacing all references to it with `Int`.

## Proposed solution

Scrap the `IndexDistance` associated type. Switch all references to it in the standard library to the concrete `Int` type:

```swift
protocol Collection {
	var count: Int { get }
	func index(_ i: Index, offsetBy n: Int) -> Index
	func index(_ i: Index, offsetBy n: Int, limitedBy limit: Index) -> Index?
	func distance(from start: Index, to end: Index) -> Int
}
// and in numerous extensions in the standard library
```

The one instance where a concrete type uses an `IndexDistance` other than `Int` in the standard library is `AnyCollection`, which uses `Int64`. This would be changed to `Int`.

## Source compatibility

This can be split into 2 parts:

Algorithms that currently constrain `IndexDistance` to `Int` in their `where` clause, and algorithms that use `IndexDistance` within the body of a
method, can be catered for by a deprecated typealias for `IndexDistance` inside an extension on `Collection`. This is the common case.

Collections that truly take advantage of the ability to define non-`Int` distances would be source-broken, with no practical way of making this
compatible in a 4.0 mode. It's worth noting that there are no such types in the Swift source compatibility suite.

## Effect on ABI stability

This removes an associated type and changes function signatures, so must be done before declaring ABI stability

## Alternatives considered

None other than status quo.
