# Introduce Sequence.compactMap(_:)

* Proposal: [SE-0187](0187-introduce-filtermap.md)
* Author: [Max Moiseev](https://github.com/moiseev)
* Review Manager: [John McCall](https://github.com/rjmccall)
* Status: **Implemented (Swift 4.1)**
* Implementation: [apple/swift#12819](https://github.com/apple/swift/pull/12819)
* Decision Notes:
    [Review #0](https://forums.swift.org/t/draft-introduce-sequence-filteredmap/6872),
    [Review #1](https://forums.swift.org/t/review-se-0187-introduce-sequence-filtermap/6977),
    [Review #2](https://forums.swift.org/t/accepted-and-focused-re-review-se-0187-introduce-sequence-filtermap/7076),
    [Rationale](https://forums.swift.org/t/accepted-with-revisions-se-0187-introduce-sequence-filtermap/7290)
* Previous Revision: [1](https://github.com/swiftlang/swift-evolution/blob/2d24b0ce9f138858b8341467170d6d8ba973827f/proposals/0187-introduce-filtermap.md)

## Introduction

We propose to deprecate the controversial version of a `Sequence.flatMap` method
and provide the same functionality under a different, and potentially more
descriptive, name.

## Motivation

The Swift standard library currently defines 3 distinct overloads for `flatMap`:

~~~swift
Sequence.flatMap<S>(_: (Element) -> S) -> [S.Element]
    where S : Sequence
Optional.flatMap<U>(_: (Wrapped) -> U?) -> U?
Sequence.flatMap<U>(_: (Element) -> U?) -> [U]
~~~

The last one, despite being useful in certain situations, can be (and often is)
misused. Consider the following snippet:

~~~swift
struct Person {
  var age: Int
  var name: String
}

func getAges(people: [Person]) -> [Int] {
  return people.flatMap { $0.age }
}
~~~

What happens inside `getAges` is: thanks to the implicit promotion to
`Optional`, the result of the closure gets wrapped into a `.some`, then
immediately unwrapped by the implementation of `flatMap`, and appended to the
result array. All this unnecessary wrapping and unwrapping can be easily avoided
by just using `map` instead.

~~~swift
func getAges(people: [Person]) -> [Int] {
  return people.map { $0.age }
}
~~~

It gets even worse when we consider future code modifications, like the one
where Swift 4 introduced a `String` conformance to the `Collection` protocol.
The following code used to compile (due to the `flatMap` overload in question).

~~~swift
func getNames(people: [Person]) -> [String] {
  return people.flatMap { $0.name }
}
~~~

But it no longer does, because now there is a better overload that does not
involve implicit promotion. In this particular case, the compiler error would be
obvious, as it would point at the same line where `flatMap` is used. Imagine
however if it was just a `let names = people.flatMap { $0.name }` statement, and
the `names` variable were used elsewhere. The compiler error would be
misleading.

## Proposed solution

We propose to deprecate the controversial overload of `flatMap` and re-introduce
the same functionality under a new name. The name being `compactMap(_:)` as we
believe it best describes the intent of this function.

For reference, here are the alternative names from other languages:
- Haskell, Idris
  ` mapMaybe :: (a -> Maybe b) -> [a] -> [b]`
- Ocaml (Core and Batteries) 
  `filter_map : 'a t -> f:('a -> 'b option) -> 'b t`
- F# 
  `List.choose : ('T -> 'U option) -> 'T list -> 'U list`
- Rust 
  `fn filter_map<B, F>(self, f: F) -> FilterMap<Self, F>   where F: FnMut(Self::Item) -> Option<B>`
- Scala
  ` def collect[B](pf: PartialFunction[A, B]): List[B]`

Filtering `nil` elements from the `Sequence` is very common, therefore we also
propose adding a `Sequence.compact()` function. This function should only be
available for sequences of optional elements, which is not expressible in
current Swift syntax. Until we have the missing features, using
`xs.compactMap { $0 }` is an option.

## Source compatibility

Since the old function will still be available (although deprecated) all
the existing code will compile, producing a deprecation warning and a fix-it.

## Effect on ABI stability

This is an additive API change, and does not affect ABI stability.

## Effect on API resilience

Ideally, the deprecated `flatMap` overload would not exist at the time when ABI
stability is declared, but in the worst case, it will be available in a
deprecated form from a library post-ABI stability.

## Alternatives considered

It was attempted in the past to warn about this kind of misuse and do the right
thing instead by means of a deprecated overload with a non-optional-returning
closure. The attempt failed due to another implicit promotion (this time to
`Any`).

The following alternative names for this function were considered:
- `mapNonNil(_:) `
  Does not communicate what happens to nil’s
- `mapSome(_:) `
  Reads more like «map some elements of the sequence, but not the others»
  rather than «process only the ones that produce an Optional.some»
- `filterMap(_:)`
  Considered confusing, due to similarity with `filter`, but without any control
  over what gets filtered out. Besides, even though it can be implemented as a
  series of calls to `filter` and `map`, the order of these calls is different
  from what the `filterMap` name suggests.
