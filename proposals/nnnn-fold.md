# Reducing a sequence onto its own elements

* Proposal: [SE-nnnnnnnn](nnnnnnnn-fold.md)
* Authors: [Erica Sadun](https://github.com/erica), [Nate Cook](https://github.com/nnnnnnnn)
* Review Manager: TBD
* Status: **Awaiting implementation**

<!--
*During the review process, add the following fields as needed:*

* Implementation: [apple/swift#NNNNN](https://github.com/apple/swift/pull/NNNNN)
* Decision Notes: [Rationale](https://forums.swift.org/), [Additional Commentary](https://forums.swift.org/)
* Bugs: [SR-NNNN](https://bugs.swift.org/browse/SR-NNNN), [SR-MMMM](https://bugs.swift.org/browse/SR-MMMM)
* Previous Revision: [1](https://github.com/apple/swift-evolution/blob/...commit-ID.../proposals/NNNN-filename.md)
* Previous Proposal: [SE-XXXX](XXXX-filename.md)

-->

## Introduction

This proposal introduces sequence folding as an overload to Swift's `reduce` method. This design simplifies call sites and can eliminate common errors associated with `reduce`.

Swift-evolution thread: [Pitch: Reducing a sequence onto its own elements to simplify code and reduce errors](https://forums.swift.org/t/pitch-reducing-a-sequence-onto-its-own-elements-to-simplify-code-and-reduce-errors/19243/52)

## Motivation

While Swift's `reduce` method plays an important role in functional coding, many calls to the method can be simplified by tweaking the API. This proposal introduces two variations of folding operator that seeds from the elements passed to `reduce`, removing the need for an explicit value seed. A basic version allows simple calls. The `inout` operation improves performance.

The Standard Library `reduce` method combines elements of a sequence by applying a closure:

```swift
public func reduce<Result>(
  _ initialResult: Result, 
  _ nextPartialResult: (Result, Self.Element) throws -> Result) rethrows 
  -> Result { ... })
```

`reduce(:_,:_)` is used to calculate sums and products, append strings together, search for maxima and minima, and so forth. The call site supplies a seed value (the `initialResult`) of an arbitrary generic `Return` type. Each subsequent partial result is built by applying the `nextPartialResult` closure to the previous result and the next element of the sequence. The `reduce` method returns after exhausting all elements in the sequence. 

The proposed `reduce(:_)` overload in both variations omits the initial seed. It is seeded by the first element of the sequence if one exists. Otherwise it returns `nil` for empty sequences. Like `reduce(:_,:_)`, it combines sequence elements with a closure:

```swift
public func reduce(
    _ nextPartialResult:
    (_ partialResult: Element, _ current: Element) throws -> Element) rethrows 
    -> Element?
    
public func reduce(
    _ nextPartialResult:
    (_ partialResult: inout Element, _ current: Element) throws -> Void) rethrows
    -> Element?
```

The re-architected calls are simpler to read, although they necessarily bring sequence reduction into the `Optional` space. They eliminate the first argument and form a result from the sequence itself:

```swift
let sum: Int = intSequence.reduce(0, +)
let sum: Int? = intSequence.reduce(+)
let sum: Int? = intSequence.reduce(+=)

let product: Int = intSequence.reduce(1, *)
let product: Int? = intSequence.reduce(*)
let product: Int? = intSequence.reduce(*=)
```

`reduce(_:, _:)`'s initial result seed uses a "magic value". It is supplied by the call site, and not the language or the Standard Library. Because of this, it can be technically brittle. As the following sections explore, that initial result can be the source of both semantic and seeding errors.

### Semantic Errors

Most functional programmers treat `reduce(_:_:)` as a monoid-based folding operation. A monoid is an algebraic operation that combines two values to produce another value within the same domain. Each monoid declares an identity that leaves other values unchanged when called with its function.

For example, summing numbers has an identity of 0. Adding 0 to any number returns that number. Similarly, the product identity for numbers is `1`, and the `and` identity for Boolean values is `true`. When constructing `reduce`, many Swift developers use the identity to seed the first argument:

```swift
let boolean: Bool = booleanSequence.reduce(true, { $0 && $1 })
```

While Swift's `reduce(_:_:)` method does not require monoids, it's common to use a monoid to populate `reduce`'s two arguments: the algebraic identity as the initial result and its binary `(Element, Element) -> Element` function to calculate the next partial result. 

The monoid approach ensures that any non-empty sequence can be reduced correctly. The initial value will not affect the operation. Essentially, this best practices approach to `reduce(_:,_:)` says: "use a first argument that has no effect on the final result." In doing so, it's worth considering eliminating this identity.

When following the monoid pattern, any empty sequence returns its identity. The product of no numbers is `1`, the combination of no matrices is the identity matrix, and the greatest common divisor of no 0-based natural numbers is 0, and so forth. Removing that identity means approaching empty sequences in a different manner.

The design of `reduce(:_)` returns `nil` in the absense of sequence members rather than an identity. It does this for two reasons:

* Some identities are not universal across Swift types, creating a bar to building generic implementations.
* Some identities simply cannot be represented in Swift.

#### Generics and Identities

Swift previously decided that the minimum value of an empty integer array should not be `Int.max`, even though `Int.max` is the identity of the `min` function across `Int`. We know this because Swift has already adopted an optional approach in the Standard Library for no-element sequences of any `Comparable` elements, which may not support a `min` or `max` property:

```swift
/// - Returns: The sequence's minimum element. If the sequence has no
///   elements, returns `nil`.
public func min() -> Self.Element? // ...
```

Since extreme values are not guaranteed to be present for every `Comparable` type, the design of the `min()` and `max()` methods cannot produce an identity for every sequence fed to them. In the absense of identities, they return `nil`. Adding a requirement to support an extreme-reporting protocol (for example, `Comparable & ExtremeReporting`) would narrow the number of types serviced by these features to the detriment of the language.

`reduce(:_)` follows the no-identity pattern to provide a consistent alternative to `reduce(:_:_)` when processing empty sequences of any type. It always returns `nil`, indicating a missing value, instead of the default seed.

```swift
let minimumValue: Int = [99, 2, -55, 6]
  .reduce(.max) { $0 > $1 ? $1 : $0 } // -55

let minimumValue: Int? = [99, 2, -55, 6]
  .reduce({ $0 > $1 ? $1 : $0 }) // -55
  
let minimumValue: Int = []
  .reduce(.max) { $0 > $1 ? $1 : $0 } // Specific to `Int`

let minimumValue: T? = emptySequenceOfT
  .reduce({ $0 > $1 ? $1 : $0 }) // nil, regardless of `T`
```

#### Unrepresentable Identities

In some cases, it's simply not reasonable to represent an identity in Swift. For example, the intersection identity for sets is the complete set. Consider the following code. It returns a set of strings common to all the sets passed to it. No identity can be used here because a canonical set of all possible strings cannot be constructed within the Swift language. In this case, a solution can be modeled with `reduce(_:)` but not `reduce(_:_:)`:

```swift
// This returns a set of common strings
// e.g., let stringSetSequence: [Set<String>] = 
//   [["now", "is", "the", "time"], ["today", "is", "the", "day"]]
// returns {"is", "the"}

// Not constructable with `reduce(_:_:)`
let result = stringSetSequence
    .reduce(WHAT_GOES_HERE?, { $0.intersection($1) })

// Constructable with `reduce(_:)`
let result = stringSetSequence
    .reduce({ $0.intersection($1) })
```

Absent an identity that can be specified within the language, there is no other option than to return `nil`, representing a missing value, which is what the `reduce(_:)` design does. 

### Seeding Errors

The first argument of `result(_:_:)` is prone to error. These initial result errors may be simple call-site typos. The coder may know the correct identity but misstate it in code. For example, they may replace one well-known identity with another, as is common when forming a product, or they may simply type the right identity incorrectly. 

These scenarios are easily remedied with adequate tests. For example, `[value].reduce(identity, f)` should always equal `value` for values across the domain. Still, these errors are better avoided than fixed.

In other cases, a coder may populate the first argument with the wrong value, not knowing the right one. This is less easily resolved as the person writing the code may write incorrect or insufficient tests as a result of confirmation bias.

Using `reduce(:_)` eliminates both types of errors because there is no need to supply and validate an identity element in code.

#### Call-Site Typos

Magic values rely on proper recall and text entry and are a point of coding fragility. This fragility extends to well-known algebraic identities like 0 for addition and 1 for multiplication. In using `reduce(:_,:_)`, a simple brain freeze may introduce errors when seeding the first argument, as shown in the example below. Converting from `reduce(:_,:_)` to `reduce(:_)` eliminates this class of errors from the call site:

```swift
// common typo
let product: Int = intSequence.reduce(0, *)

// always correct, no magic value
let product: Int? = intSequence.reduce(*)

//  another common typo
let minimumValue: Int = intSequence
  .reduce(.min) { $0 > $1 ? $1 : $0 }
  
// again correct, no constant substitution
let minimumValue: Int? = intSequence
  .reduce({ $0 > $1 ? $1 : $0 })
```

#### Incorrect Identities

Eliminating identities is an important benefit when using less common seeds. For example, you can build minimum bounding frames for rectangles using both `reduce(:_,:_)` and `reduce(:_)` but specifying the wrong `reduce(:_,:_)` seed introduces the following value error:

```swift
let frames = [
  CGRect(x: 10, y: 10, width: 50, height: 50),
  CGRect(x: 40, y: 80, width: 20, height: 30),
  CGRect(x: -5, y: 10, width: 10, height: 10),
]

frames.reduce(CGRect.zero, { $0.union($1) })
// (x: -5.0, y: 0.0, width: 65.0, height: 110.0), incorrect

frames.reduce(CGRect.null, { $0.union($1) })
// (x: -5.0, y: 10.0, width: 65.0, height: 100.0), correct

frames.reduce({ $0.union($1) })
// (x: -5.0, y: 10.0, width: 65.0, height: 100.0), correct
```

When the coder selects the more common `.zero` constant over the less well known `.null`, the `.zero` seed pulls the `y` bounds to `0.0`, leaving ten extra points of space preceding the minimum frame. While `CGRect.null` returns the right results, `reduce(:_)` uses a simpler approach that cannot be affected by incorrectly chosen identities.

If the developer uses insufficient test cases that wrap the origin, they may not discover the error. Using `reduce(:_)` gets rid of these errors and removes any responsibility for representing the identity from code.

### Working with Optionals

At first glance, using optionals may appear to impose a burden on the coder. It seems that yet another Swift construct places a barrier between a value and its use. To the contrary, you can often consider an empty list reduction as a separate outcome and treat it as such in code. 

For example, think about a processing a list followed by a count. "Cart subtotal: $49.92" versus "Your shopping cart is empty". A conditional binding offers a natural way of expressing these two outcomes:

```
if let subtotal = items.map({ $0.cost }).reduce(+) {
    return "\(localized: "Cart subtotal"): \(localizedCurrency: subtotal)"
} else {
    return "\(localized: "Your shopping cart is empty")"
}
```

The alternative for this specific example is to create a special case for a sum of zero, saving little space and shifting the logic to the derived sum rather than the inherent emptiness of the list. In such circumstances, the optional approach is preferred. It uses fundamental logic closely tied to the input rather than a derived outcome. The complexity of the code is otherwise roughly equal.

## Detailed design

This design introduces two unseeded variations of `reduce(:_,:_)`. These overloads use the first value of each sequence to form the initial partial result. If the sequence is empty, it returns `nil`.

While `reduce` uses a partial result generator with a potentially distinct return type `f(Result, Element) -> Result`, `reduce(:_)` constrains its results to the same type as the source element `f(Element, Element) -> Element`. This change is required as the seed value will always be the sequence element type should a first value exist and `nil` otherwise.

Although applications of `reduce` may return a type distinct from the sequence elements, this can often be broken down into `(Element) -> Result` and `(Result, Result) -> Result` steps. For example, `let stringSum = stringSequence.reduce(0, { $0 + $1.count })` is essentially the same as `stringSequence.map({ $0.count }).reduce(0, +)`.

The design of `reduce(:_)` uses an iterator to distinguish empty sequences from those with values. 

## `inout` Variation

Following the lead of [SE-0171](https://github.com/apple/swift-evolution/blob/master/proposals/0171-reduce-with-inout.md), ([forum discussion](https://forums.swift.org/t/reduce-with-inout/4897), see also: [this thread](https://forums.swift.org/t/why-doesnt-reduce-into-include-a-variant-that-takes-an-inout-input/19395) about performance), this proposal includes two variations:

* an `(Element, Element) -> Element` version, and
* an `(inout Element, Element) -> Void` version to enhance performance for operations that lend themselves to `inout` closures.

On normal sequence operations like sums and products, the inout variation offers minor efficiency gains:

```
Task: Multiply array of 10_000_000 1's together

// Reduce with *
Starting Test: Element, Element -> Element
Ending Test: Element, Element -> Element
Elapsed time: 2.0035429719719104

// Reduce with *=
Starting Test: inout Element, Element -> Void
Ending Test: inout Element, Element -> Void
Elapsed time: 1.8242954809684306
```

When using indirect property access, the inout version outperforms by 500 to 1, allowing the result to accumulate changes without repeated allocations and copying:

```
public struct Event { var count: Int, logs: [String] }

// Task 1, no inout 
// Note: a direct constructor performs equally
let result = events.reduce({ (allEvents, event) -> Event in
  var allEvents = allEvents
  allEvents.count += event.count
  allEvents.logs += event.logs
  return allEvents
})

// Task 2, inout
let result = events.reduce({ (allEvents: inout Event, event: Event) in
  allEvents.count += event.count
  allEvents.logs += event.logs
})

// (E, E) -> E
Starting Test: Element -> Element subfields with var allEvents
Ending Test: Element -> Element subfields with var allEvents
Elapsed time: 1.6208692869986407

// (inout E, E) -> E
Starting Test: inout Element -> Element subfields
Ending Test: inout Element -> Element subfields
Elapsed time: 0.0029899069922976196
```

### Precedent 

This design follows the Standard Library precedent for sequence extremes (`min`, `max`) by returning `nil` when a sequence has no elements. This allows seedless calls that apply across many types without having to declare identities for each type and each operation.

### Naming

This design overloads `reduce` to accept a single closure argument. Here is a quick overview of alternate naming options.

Generally speaking, a `fold` or `reduce` higher order function processes a data structure to build a return value. An `unfold` seeds a start value to generate a data structure. 

Many languages include up to four styles of reduction: left to right with an initial value, right to left with an initial value, left to right without an initial value, and right to left without an initial value. Swift currently implements just one of these, the left-to-right `reduce(:_,:_)`, which takes an initial value.

The name `fold` is a term of art, commonly interchanged with `reduce` in various languages.  Other terms include accumulate, aggregate, compress, and inject. The names are used somewhat interchangeably among languages, with a slight tendency towards `fold` for using an initial value and `reduce` without. 

If the `reduce` feature were being designed today, this proposal would recommend `fold` over `reduce`, and prefer overloading `fold` for both applications. As `reduce` already exists, it overloads the existing method.

|Language|Fold with Initial Value|Fold without Initial Value|
|--------|-----------------------|--------------------------|
|C# 3.0|Aggregate|Aggregate|
|C++|accumulate||
|CFML, Clojure, CLisp, D, Java 8+, Perl, Python|reduce|reduce|
|Elm, Erlang, Standard ML|foldl, foldr||
|F#|fold, foldBack|reduce, reduceBack|
|Gosu|fold, reduce||
|Groovy|inject|inject|
|Haxe, Rust|fold||
|JavaScript|reduce, reduceRight|reduce, reduceRight|
|Kotlin|fold, foldRight|reduce, reduceRight|
|Logtalk, OCaml|fold\_left, fold\_right||
|Oz|FoldL, FoldR||
|PHP|array\_reduce|array\_reduce|
|R|Reduce|Reduce|
|Ruby|inject, reduce|inject, reduce|
|Scala|foldLeft, foldRight|reduceLeft, reduceRight|
|Scheme|fold-left, fold-right|reduce-left, reduce-right|
|Haskell|foldl, fold|foldl1, foldr1|
|Xtend|fold|reduce|

### Preliminary Implementation

```swift
import Foundation

extension Sequence {
  /// Combines the elements of the sequence using a closure, returning the
  /// result (or nil, for a no-element sequence).
  ///
  /// This method uses an (Element, Element) -> Element closure.
  /// Prefer the (inout Element, Element) -> Element version of reduce(_:)
  /// instead for efficiency when the result is a copy-on-write type,
  /// for example an Array or a Dictionary.
  ///
  /// Use the `reduce(_:)` method to produce a combined value from a
  /// sequence. For example, you can return the sum or product of a
  /// sequence's elements or its minimum or maximum value.
  /// Each `nextPartialResult` closure is called sequentially, accumulating
  /// the value initialized to the first element of the sequence.
  /// This example shows how to find the sum of an array of numbers.
  ///
  ///     let numbers = [1, 2, 3, 4]
  ///     let numberSum = numbers.reduce({ x, y in
  ///         x + y
  ///     })
  ///     // numberSum == 10
  ///
  /// Alternatively:
  ///
  ///     let numberSum = numbers.reduce(+) // 10
  ///     let numberProduct = numbers.reduce(*) // 24
  ///
  /// When `numbers.reduce(_:)` is called, the following steps occur:
  ///
  /// 1. The partial result is initialized from the first sequence member,
  ///    returning nil for an empty sequence. The first number is 1.
  /// 2. The closure is called repeatedly with the current partial result and
  ///    each successive member of the sequence
  /// 3. When the sequence is exhausted, the method returns the last value
  ///    returned from the closure.
  ///
  /// If the sequence has no elements, `reduce` returns `nil`.
  ///
  /// If the sequence has one element, `reduce` returns that element.
  ///
  /// For example, `reduce` can combine elements to calculate the minimum
  /// bounds fitting a set of rectangles defined by an array of `CGRect`
  ///
  ///     let frames = [
  ///       CGRect(x: 10, y: 10, width: 50, height: 50),
  ///       CGRect(x: 40, y: 80, width: 20, height: 30),
  ///       CGRect(x: -5, y: 10, width: 10, height: 10),
  ///     ]
  ///
  ///     frames.reduce({ $0.union($1) })
  ///     // (x: -5.0, y: 10.0, width: 65.0, height: 100.0)
  ///
  /// - Parameters:
  ///   - nextPartialResult: A closure that combines an accumulating value and
  ///     an element of the sequence into a new accumulating value, to be used
  ///     in the next call of the `nextPartialResult` closure or returned to
  ///     the caller.
  ///   - partialResult: The accumulated value of the sequence, initialized as the
  ///     first sequence member
  ///   - current: The next element of the sequence to combine into the partial result
  /// - Returns: The final accumulated value or if the sequence has
  ///   no elements, returns `nil`.
  ///
  /// - Complexity: O(*n*), where *n* is the length of the sequence.
  ///
  /// - Note: Prefer the `inout` variation of `reduce(:_)` when processing
  ///   complex fields. The `inout` version avoids repeated copies and construction
  ///   for better performance.
  public func reduce(
    _ nextPartialResult:
    (_ partialResult: Element, _ current: Element) throws -> Element) rethrows
    -> Element? {
    var iterator = makeIterator()
    guard var accumulator = iterator.next() else {
      return nil
    }
    while let element = iterator.next() {
      accumulator = try nextPartialResult(accumulator, element)
    }
    return accumulator
  }
  
  /// Combines the elements of the sequence using a closure with
  /// an initial `inout` argument, returning the result (or nil,
  /// for a no-element sequence).
  ///
  /// Use the `reduce(_:)` method to produce a combined value from a
  /// sequence. For example, you can return the sum or product of a
  /// sequence's elements or its minimum or maximum value.
  /// Each `nextPartialResult` closure is called sequentially, accumulating
  /// the value initialized to the first element of the sequence.
  /// This example shows how to find the sum of an array of numbers.
  ///
  ///     let numbers = [1, 2, 3, 4]
  ///     let numberSum = numbers.reduce({ x, y in
  ///         x += y
  ///     })
  ///     // numberSum == 10
  ///
  /// Alternatively:
  ///
  ///     let numberSum = numbers.reduce(+=) // 10
  ///     let numberProduct = numbers.reduce(*=) // 24
  ///
  /// When `numbers.reduce(_:)` is called, the following steps occur:
  ///
  /// 1. The partial result is initialized from the first sequence member,
  ///    returning nil for an empty sequence. The first number is 1.
  /// 2. The closure is called repeatedly with the current partial result
  ///    and each successive member of the sequence
  /// 3. When the sequence is exhausted, the method returns the last value
  ///    established by the closure.
  ///
  /// If the sequence has no elements, `reduce` returns `nil`.
  ///
  /// If the sequence has one element, `reduce` returns that element.
  ///
  /// This method uses an (inout Element, Element) -> Element closure.
  /// Prefer it over the (Element, Element) -> Element version
  /// of reduce(_:) for efficiency when otherwise repeatedly
  /// constructing a value. For example, prefer this form of reduce
  /// to combine the data from a sequence of the following `Event` type
  /// into a single instance. Copying or constructing a new instance
  /// to perform the field updates underperforms in the non `inout`
  /// version:
  ///
  /// ```
  /// public struct Event { var count: Int, logs: [String] }
  /// ```
  ///
  /// - Parameters:
  ///   - nextPartialResult: A closure that combines an accumulating value and
  ///     an element of the sequence into a new accumulating value, to be used
  ///     in the next call of the `nextPartialResult` closure or returned to
  ///     the caller.
  ///   - partialResult: The accumulated value of the sequence, initialized as the
  ///     first sequence member
  ///   - current: The next element of the sequence to combine into the partial result
  /// - Returns: The final accumulated value or if the sequence has
  ///   no elements, returns `nil`.
  ///
  /// - Complexity: O(*n*), where *n* is the length of the sequence.
  public func reduce(
    _ nextPartialResult:
    (_ partialResult: inout Element, _ current: Element) throws -> Void) rethrows
    -> Element? {
    var iterator = makeIterator()
    guard var accumulator = iterator.next() else {
      return nil
    }
    while let element = iterator.next() {
      try nextPartialResult(&accumulator, element)
    }
    return accumulator
  }
}

```

## Source compatibility

This change is purely additive.

## Effect on ABI stability

N/A

## Effect on API resilience

N/A

## Alternatives considered

* The forum thread discussed introducing monoids either as a protocol or type, for example as [shown here](https://forums.swift.org/t/pitch-reducing-a-sequence-onto-its-own-elements-to-simplify-code-and-reduce-errors/19243/4?u=erica_sadun) and [shown here](https://gist.github.com/erica/6368ed5924d803c7948189ce61b5b57e), allowing calls to `sequenceOfInt.fold(Add.self)` or `["a", "bc"].reduce(String.join)` or `frames.reduce(CGRect.union)`. This may be an avenue worth exploring in the future but its scope lies outside this proposal.

* Stephen Celis had a [really fun approach](https://forums.swift.org/t/pitch-reducing-a-sequence-onto-its-own-elements-to-simplify-code-and-reduce-errors/19243/39?u=erica_sadun) for combining keypaths with `reduce`.

## Acknowledgements

Thanks Soroush Khanlou, Tim Vermeulen, Lily Vulcano, Davide De Franceschi, Stephen Cellis, Matthew Johnson, Nevin, Brandon Williams, Tellow Krinkle, David Hart, Peter Tomaselli, Ben Cohen, Lantua, Stephen Cellis, and everyone else who offered their feedback, functional programming experience, and design insights.
