# Support recursive constraints on associated types

* Proposal: [SE-0157](0157-recursive-protocol-constraints.md)
* Authors: [Douglas Gregor](https://github.com/DougGregor), [Erica Sadun](https://github.com/erica), [Austin Zheng](https://github.com/austinzheng)
* Review Manager: [John McCall](https://github.com/rjmccall)
* Status: **Implemented (Swift 4.1)**
* Decision Notes: [Rationale](https://forums.swift.org/t/se-0157-support-recursive-constraints-on-associated-types/5494)
* Bug: [SR-1445](https://bugs.swift.org/browse/SR-1445)

## Introduction

This proposal lifts restrictions on associated types in protocols. Their constraints will be allowed to reference any
protocol, including protocols that depend on the enclosing one (recursive constraints).

Further reading: [swift-evolution thread](https://forums.swift.org/t/pitch-plea-recursive-protocol-constraints/4507), _[Completing Generics](https://github.com/apple/swift/blob/master/docs/GenericsManifesto.md#recursive-protocol-constraints-)_

## Motivation

Swift supports defining _associated types_ on protocols using the `associatedtype` keyword.

```swift
protocol Sequence {
    associatedtype Subsequence
}
```

Swift also supports defining _constraints_ on those associated types, for example:

```swift
protocol Foo {
    // For all types X conforming to Foo, X.SomeType must conform to Bar
    associatedtype SomeType: Bar
}
```

However, Swift does not currently support defining _constraints on an associated type that recursively reference the
enclosing protocol_. It would make sense for `SubSequence` to be constrained to be a `Sequence`, as all subsequences
are themselves sequences:

```swift
// Will not currently compile
protocol Sequence {
    associatedtype SubSequence: Sequence
        where Iterator.Element == SubSequence.Iterator.Element, SubSequence.SubSequence == SubSequence

    // Returns a subsequence containing all but the first 'n' items
    // in the original sequence.
    func dropFirst(_ n: Int) -> Self.SubSequence
    // ...
}
```

However, Swift currently doesn't support expressing this constraint at the point where `SubSequence` is declared.
Instead, we must specify it in documentation and/or at each site of use. This results in more verbose code and obscures
intent:

```swift
protocol Sequence {
    // SubSequences themselves must be Sequences.
    // The element type of the subsequence must be identical to the element type of the sequence.
    // The subsequence's subsequence type must be itself.
    associatedtype SubSequence

    func dropFirst(_ n: Int) -> Self.SubSequence
    // ...
}

struct SequenceOfInts : Sequence {
    // This concrete implementation of `Sequence` happens to work correctly.
    // Implicitly:
    // The subsequence conforms to Sequence.
    // The subsequence's element type is the same as the parent sequence's element type.
    // The subsequence's subsequence type is the same as itself.
    func dropFirst(_ n: Int) -> SimpleSubSequence<Int> {
        // ...
    }
}

struct SimpleSubSequence<Element> : Sequence {
    typealias SubSequence = SimpleSubSequence<Element>
    typealias Iterator.Element = Element
    // ...
}
```

## Proposed solution

The first part of the solution we propose is to lift this restriction. From the perspective of the end user, this is a
relatively simple change. It is only a new feature in the sense that certain associated type definitions which were
previously disallowed will now be accepted by the compiler.

Implementation details regarding the compiler changes necessary to implement the first part of the solution can be
found in [this document](https://gist.github.com/DougGregor/e7c4e7bb4465d6f5fa2b59be72dbdba6).

The second part of the solution involves updating the standard library to take advantage of the removal of this
restriction. Such changes are made with [SE-0142](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0142-associated-types-constraints.md)
in mind, and incorporate both recursive constraints and `where` clauses. The changes necessary for this are described
in the _Detailed Design_ section below.

This second change will affect the sort of user code which is accepted by the compiler. User code which uses the
affected protocols and types will require fewer generic parameter constraints to be considered valid. Conversely,
user code which (incorrectly) uses the private protocols removed by this proposal, or which uses the affected public
protocols in an incorrect manner, might cease to be accepted by the compiler after this change is implemented. 

## Detailed design

The following standard library protocols and types will change in order to support recursive protocol constraints.

Note that since the specific collection types conform to `Collection`, and `Collection` refines `Sequence`, not all the
constraints need to be defined on every collection-related associated type.

Default values of all changed associated types remain the same, unless explicitly noted otherwise.

All "Change associated type" entries reflect the complete, final state of the associated type definition, including
removal of underscored protocols and addition of any new constraints.

### `Arithmetic`

* Change associated type: `associatedtype Magnitude : Arithmetic`

### `BidirectionalCollection`

* Remove conformance to `_BidirectionalIndexable`
* Change associated type: `associatedtype SubSequence : BidirectionalCollection`
* Change associated type: `associatedtype Indices : BidirectionalCollection`

### `Collection`

* Remove conformance to `_Indexable`
* Change associated type: `associatedtype SubSequence : Collection where SubSequence.Index == Index`
* Change associated type: `associatedtype Indices : Collection where Indices.Iterator.Element == Index, Indices.Index == Index`

### `Default*Indices` (all variants)

* Declarations changed to `public struct Default*Indices<Elements : *Collection> : *Collection`

### `IndexingIterator`

* Declaration changed to `public struct IndexingIterator<Elements : Collection> : IteratorProtocol, Sequence`

### `LazyFilter*Collection` (all variants)

* Add default associated type conformance: `typealias SubSequence = ${Self}<Base.SubSequence>`

### `LazyMap*Collection` (all variants)

* Add default associated type conformance: `typealias SubSequence = ${Self}<Base.SubSequence>`

### `MutableCollection`

* Change associated type: `associatedtype SubSequence : MutableCollection`

### `RandomAccessCollection`

* Change associated type: `associatedtype SubSequence : RandomAccessCollection`
* Change associated type: `associatedtype Indices : RandomAccessCollection`

### `RangeReplaceableCollection`

* Change associated type: `associatedtype SubSequence : RangeReplaceableCollection`

### `Sequence`

* Change associated type: `associatedtype SubSequence : Sequence where Iterator.Element == SubSequence.Iterator.Element, SubSequence.SubSequence == SubSequence`

### `*Slice` (all variants)

* Add default associated type conformance: `typealias Indices = Base.Indices`

## Source compatibility

From a source compatibility perspective, this is a purely additive change if the user's code is correctly written. It
is possible that users may have written code which defines semantically incorrect associated types, which the compiler
now rejects because of the additional constraints. We do not consider this scenario "source-breaking".

An example of code that currently compiles but is semantically invalid is an implementation of a range-replacable
collection's subsequence that isn't itself range-replaceable. This is a constraint that cannot be enforced by the compiler
without this change. For some time, the `Data` type in Foundation violated this constraint; user-written code that is
similarly problematic will cease to compile using a Swift toolchain that includes these standard library and compiler
changes.

## Impact on ABI stability

Since this proposal involves modifying the standard library, it changes the ABI. In particular, ABI changes enabled by
this proposal are critical to getting the standard library to a state where it more closely resembles the design
envisioned by its engineers.

## Impact on API resilience

This feature cannot be removed without breaking API compatibility, but since it forms a necessary step in crystallizing
the standard library for future releases, it is very unlikely that it will be removed after being accepted.

## Alternatives considered

n/a
