# Improved `zip` API

- Proposal: [SE-NNNN](nnnn-improved-zip-api.md)
- Author: [Dennis Vennink](https://github.com/dennisvennink)
- Review Manager: To Be Determined
- Status: **Awaiting Review**
- Implementation: [apple/swift#18769](https://github.com/apple/swift/pull/18769)
- Bug: [SR-3760](https://bugs.swift.org/browse/SR-3760)

## Introduction

This proposal addresses the limitations of the Standard Library's current `zip` API by adding improvements in the form of enhancements and additions.

It's comprised of:

- the *types* `Zip`[`2`–`6`]`Index`, `Zip`[`2`–`6`]`Sequence` and `ZipLongest`[`2`–`6`]`Sequence`;
- the *global `func`tions* `zip(_:_:`[`_:`[`_:`[`_:`[`_:`]]]]`)`, `zipLongest(_:_:`[`_:`[`_:`[`_:`[`_:`]]]]`)`;
- and the *`extension` method* `unzip()` *on `Sequence` and `LazyCollectionProtocol`*.

### Discussions

It has been discussed in the following Evolution forum threads:

- [Improved `zip` API](https://forums.swift.org/t/improved-zip-api/18949);
- [Add Various `zip`-Related Types and Operations](https://forums.swift.org/t/add-various-zip-related-types-and-operations/13806);
- and [Add a Padded Version of `zip` to Complement the Current `zip`](https://forums.swift.org/t/add-a-padded-version-of-zip-to-complement-the-current-zip/11760).

Its development has been discussed in the following Development forum thread:

- [Implementation of Collection Versions of `zip`](https://forums.swift.org/t/implementation-of-collection-versions-of-zip/15384).

## Motivation

The Standard Library's current `zip` API consists solely of the global `func`tion `zip(_:_:)`, which creates a [`Sequence`](https://developer.apple.com/documentation/swift/zip2sequence) of [pairs](https://developer.apple.com/documentation/swift/zip2sequence/element) from a pair of `Sequence`s by zipping their respective elements until the shortest `Sequence` is exhausted, truncating the latter elements of the longest `Sequence`.

It's primarily used to iterate over two `Sequence`s simultaneously:

```swift
func dotProduct <T: Numeric> (_ vector1: [T], _ vector2: [T]) -> T {
  return zip(vector1, vector2).map(*).reduce(0, +)
}

print(dotProduct([-1.5, 2], [4, 3]))
// Prints "0.0"
```

And since it's lazily implemented we can also `zip` over (pseudo-)infinite `Sequence`s; which is a common functional pattern:

```swift
extension Sequence {
  func enumerated () -> Zip2Sequence<PartialRangeFrom<Int>, Self> {
    return zip(0..., self)
  }
}

print(Array("abc".enumerated()))
// Prints "[(0, "a"), (1, "b"), (2, "c")]"
```

However, it's missing certain `protocol` conformances, and features found in many other programming languages, like non-truncating and inverse `zip` operations and support for higher arities.

### Missing `protocol` Conformances

#### `Collection`, `BidirectionalCollection` and `RandomAccessCollection`

Currently, `Zip2Sequence`'s only conformance is to `Sequence`. Since [SE-0234](0234-remove-sequence-subsequence.md) removed `SubSequence` as an `associatedtype` from `Sequence`, we can extend `Zip2Sequence` to conditionally conform to `protocol`s higher up the hierarchy without introducing a new type.

Adding these additional conformances has a number of benefits:

1. It makes their properties and methods available for use.
2. It makes existing type inferred code that uses operations with improved default implementations higher up the hierarchy more capable or performant.
3. It guarantees that the instance can be non-destructively consumed by iteration when the underlying `Sequence`s conform to `Collection`.

However, the granularity of `Sequence`'s `protocol` hierarchy makes a correct implementation of these conformances non-trivial.

Each `Collection` places additional conditional constraints on the underlying `Sequence`s to comply with its semantics. These constraints hinder members lower in the `protocol` hierarchy from using a `Collection`'s more capable or performant default implementations in a generic context. This means that these members will have to be reimplemented on an unconstrained `extension`. But which members specifically can only be found out by manually tracking code paths in the Standard Library's source code as the compiler doesn't provide fixits.

#### `LazySequenceProtocol`

Because `Zip2Sequence` doesn't conform to `LazySequenceProtocol` when its underlying `Sequence`s conform to `LazySequenceProtocol`, some subsequent operations for which lazy variants exist are not used, even though `Zip2Sequence` itself is implemented lazily:

```swift
func id <T> (_ x: T) -> T {
  return x
}

print(type(of: zip([0, 1, 2].lazy, "abc".lazy).map(id)))
// Prints "Array<(Int, Character)>"
```

Here, we workaround this limitation by calling `lazy` on `zip(_:_:)`'s result to make consequent operations lazy again:

```swift
print(type(of: zip([0, 1, 2].lazy, "abc".lazy).lazy.map(id)))
// Prints "LazyMapSequence<Zip2Sequence<LazySequence<Array<Int>>, LazySequence<String>>, (Int, Character)>"
```

While this doesn't have a negative impact on performance—as `LazyMapSequence` will unwrap `LazySequence`—it's not exactly elegant.

### Non-Truncating `zip` Operation

Some successive operations require all elements of the underlying `Sequence`s, including the latter elements of the *longest* `Sequence`. Such an operation is useful when the lengths of the underlying `Sequence`s are indeterminable at compile time and helps to prevent potential correctness traps. It's also versatile in that it composes nicely with other operations.

Both eager and lazy concrete type-`return`ing implementations of this operation can be created from existing operations and language constructs.

Here, we use state and a `while` loop to create an eager imperative implementation:

```swift
func zipLongest <Sequence1: Sequence, Sequence2: Sequence> (_ sequence1: Sequence1, _ sequence2: Sequence2) -> [(Sequence1.Element?, Sequence2.Element?)] {
  var (iterator1, iterator2) = (sequence1.makeIterator(), sequence2.makeIterator())
  var result = [(Sequence1.Element?, Sequence2.Element?)]()

  result.reserveCapacity(Swift.max(sequence1.underestimatedCount, sequence2.underestimatedCount))

  while true {
    let (element1, element2) = (iterator1.next(), iterator2.next())

    if element1 == nil && element2 == nil {
      break
    } else {
      result.append((element1, element2))
    }
  }

  return result
}

print(zipLongest([0, 1, 2], [0]))
// Prints "[(Optional(0), Optional(0)), (Optional(1), nil), (Optional(2), nil)]"
```

And here, we use `sequence(state:next:)` and `lazy` to create a lazy functional implementation:

```swift
func zipLongest <Sequence1: Sequence, Sequence2: Sequence> (_ sequence1: Sequence1, _ sequence2: Sequence2) -> LazySequence<UnfoldSequence<(Sequence1.Element?, Sequence2.Element?), (Sequence1.Iterator, Sequence2.Iterator)>> {
  return sequence(state: (sequence1.makeIterator(), sequence2.makeIterator())) {
    let (element1, element2) = ($0.0.next(), $0.1.next())

    return element1 == nil && element2 == nil ? nil : (element1, element2)
  }.lazy
}

print(Array(zipLongest([0, 1, 2], [0])))
// Prints "[(Optional(0), Optional(0)), (Optional(1), nil), (Optional(2), nil)]"
```

However, these implementations won't scale along with the capabilities of the underlying `Sequence`s. A type that does, similar to `Zip2Sequence`, is tricky to implement for the [aforementioned](#collection-bidirectionalcollection-and-randomaccesscollection) reasons.

### Inverse `zip` Operation

Most programming languages that include `zip` also include some variant of its inverse operation, usually named `unzip`, as an idiom. Swift should follow suit.

Fortunately, this operation doesn't require any new types. But implementing it is tricky as some common performance and correctness traps exist.

Here, `reduce(into:_:)` can't reserve the capacity up-front:

```swift
func unzip <Base: Sequence, Element1, Element2> (_ sequence: Base) -> ([Element1], [Element2]) where Base.Element == (Element1, Element2) {
  return sequence.reduce(into: ([Element1](), [Element2]())) { (result, element) in
    result.0.append(element.0)
    result.1.append(element.1)
  }
}
```

And a lazy variant can't be implemented on a type that only conforms to `LazySequenceProtocol` since the operation requires multiple passes.

### Support for Higher Arities

`zip` is currently limited to a pair of `Sequence`s.

Here, we workaround it by nesting multiple calls to `zip(_:_:)` and calling `map(_:)` on its result to realign its elements:

```swift
import UIKit

let components = (
  red: sequence(state: 0.0...1.0) { CGFloat.random(in: $0) },
  green: sequence(state: 0.0...1.0) { CGFloat.random(in: $0) },
  blue: sequence(state: 0.0...1.0) { CGFloat.random(in: $0) },
  alpha: repeatElement(CGFloat(1.0), count: 3)
)

zip(components.red, zip(components.green, zip(components.blue, components.alpha)))
  .map { UIColor(red: $0, green: $1.0, blue: $1.1.0, alpha: $1.1.1) }
  .forEach { print($0) }
// Prints "UIExtendedSRGBColorSpace 0.357433 0.784562 0.903349 1"
// Prints "UIExtendedSRGBColorSpace 0.411219 0.622004 0.842976 1"
// Prints "UIExtendedSRGBColorSpace 0.615279 0.888622 0.651203 1"
```

However, this becomes less readable and increasingly more complicated for higher arities and is less performant.

For non-truncating `zip` operations this becomes a readability issue as both the values and tuples will be wrapped in `Optional`s.

## Proposed Solution

### Enhancements

#### `Zip2Sequence`

<table name="figure-1">
  <thead>
    <tr>
      <th rowspan="1" colspan="1" align="left">Conforms to</th>
      <th rowspan="1" colspan="1" align="left">Conditions</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td rowspan="1" colspan="1" align="left"><code>Sequence</code></td>
      <td rowspan="1" colspan="1" align="left"><code>Sequence</code></td>
    </tr>
    <tr>
      <td rowspan="1" colspan="1" align="left"><code>Collection</code></td>
      <td rowspan="1" colspan="1" align="left"><code>Collection</code></td>
    </tr>
    <tr>
      <td rowspan="1" colspan="1" align="left"><code>BidirectionalCollection</code></td>
      <td rowspan="1" colspan="1" align="left"><code>RandomAccessCollection</code></td>
    </tr>
    <tr>
      <td rowspan="1" colspan="1" align="left"><code>RandomAccessCollection</code></td>
      <td rowspan="1" colspan="1" align="left"><code>RandomAccessCollection</code></td>
    </tr>
    <tr>
      <td rowspan="1" colspan="1" align="left"><code>LazySequenceProtocol</code></td>
      <td rowspan="1" colspan="1" align="left"><code>LazySequenceProtocol</code></td>
    </tr>
  </tbody>
</table>

<sub>**Figure 1 — Conditional Conformances of `Zip2Sequence` and `ZipLongest2Sequence`**</sub>

I propose extending `Zip2Sequence` to conform to the `protocol`s listed in the left-hand column of [figure 1](#figure-1) on the condition that the underlying `Sequence`s conform to the `protocol`s on their right.

##### Examples

Revisiting the benefits that have been laid out in the motivation by example, it makes previously unavailable properties and methods available for use:

```swift
let xs = zip(0..., [0, 1, 2])

print(xs[xs.startIndex], xs.last, xs.count, separator: ", ")
// Prints "(0, 0), Optional((2, 2)), 3"
```

It makes existing type inferred code that uses operations with improved default implementations higher up the hierarchy more capable or performant, e.g., `Sequence.reversed()` versus `BidirectionalCollection.reversed()`, which is O(*n*) versus O(*1*), respectively.

It guarantees that the instance can be non-destructively consumed by iteration when the underlying `Sequence`s conform to `Collection`, e.g., `Collection` itself and `RandomAccessCollection`.

And finally, laziness works as expected:

```swift
func id <T> (_ x: T) -> T {
  return x
}

print(type(of: zip([0, 1, 2].lazy, "abc".lazy).map(id)))
// Prints "LazyMapSequence<Zip2Sequence<LazySequence<Array<Int>>, LazySequence<String>>, (Int, Character)>"
```

### Additions

#### `zipLongest(_:_:)` and `ZipLongest2Sequence`

<table name="figure-2">
  <thead>
    <tr>
      <th rowspan="3" colspan="1" align="left">Languages</th>
      <th rowspan="1" colspan="4" align="left">Operations</th>
    </tr>
    <tr>
      <th rowspan="1" colspan="3" align="left"><code>zip</code></th>
      <th rowspan="2" colspan="1" align="left">Inverse <code>zip</code></th>
    </tr>
    <tr>
      <th rowspan="1" colspan="1" align="left">Shortest</th>
      <th rowspan="1" colspan="1" align="left">Equal</th>
      <th rowspan="1" colspan="1" align="left">Longest</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="1" colspan="1" align="left">Clojure</th>
      <td rowspan="1" colspan="2" align="center"><code>(map </code>(<code>vector</code>|<code>list</code>)<code> xs…)</code></td>
      <td rowspan="1" colspan="1" align="center">–</td>
      <td rowspan="1" colspan="1" align="center"><code>(apply map </code>(<code>vector</code>|<code>list</code>)<code> zs)</code></td>
    </tr>
    <tr>
      <th rowspan="1" colspan="1" align="left">D</th>
      <td rowspan="1" colspan="3" align="center"><code>zip(</code>[<code>StoppingPolicy=shortest, </code>]<code>xs…)</code></td>
      <td rowspan="1" colspan="1" align="center"><code>zs.front.length.iota.map!(i =&gt; transversal(zs, i))</code></td>
    </tr>
    <tr>
      <th rowspan="2" colspan="1" align="left">Elixir</th>
      <td rowspan="1" colspan="2" align="center"><code>Enum.zip(xs, ys)</code></td>
      <td rowspan="2" colspan="1" align="center">–</td>
      <td rowspan="2" colspan="1" align="center"><code>zs |> Enum.unzip</code></td>
    </tr>
    <tr>
      <td rowspan="1" colspan="2" align="center"><code>zs |> Enum.zip</code></td>
    </tr>
    <tr>
      <th rowspan="1" colspan="1" align="left">Elm</th>
      <td rowspan="1" colspan="2" align="center"><code>List.map2 Tuple.pair xs ys</code></td>
      <td rowspan="1" colspan="1" align="center">–</td>
      <td rowspan="1" colspan="1" align="center"><code>List.unzip zs</code></td>
    </tr>
    <tr>
      <th rowspan="2" colspan="1" align="left">Erlang</th>
      <td rowspan="2" colspan="1" align="center">–</td>
      <td rowspan="1" colspan="1" align="center"><code>lists:zip</code>[<code>3</code>]<code>(Xs…).</code></td>
      <td rowspan="2" colspan="1" align="center">–</td>
      <td rowspan="2" colspan="1" align="center"><code>lists:unzip</code>[<code>3</code>]<code>(Zs).</code></td>
    </tr>
    <tr>
      <td rowspan="1" colspan="1" align="center"><code>lists:zipwith</code>[<code>3</code>]<code>(F, Xs…).</code></td>
    </tr>
    <tr>
      <th rowspan="1" colspan="1" align="left">F#</th>
      <td rowspan="1" colspan="1" align="center">–</td>
      <td rowspan="1" colspan="1" align="center"><code>List.zip</code>[<code>3</code>]<code> xs…</code></td>
      <td rowspan="1" colspan="1" align="center">–</td>
      <td rowspan="1" colspan="1" align="center"><code>List.unzip</code>[<code>3</code>]<code> zs</code></td>
    </tr>
    <tr>
      <th rowspan="2" colspan="1" align="left">Haskell</th>
      <td rowspan="1" colspan="2" align="center"><code>zip</code>[<code>3–7</code>]<code> xs…</code></td>
      <td rowspan="2" colspan="1" align="center">–</td>
      <td rowspan="2" colspan="1" align="center"><code>unzip</code>[<code>3–7</code>]<code> zs</code></td>
    </tr>
    <tr>
      <td rowspan="1" colspan="2" align="center"><code>zipWith</code>[<code>3</code>]<code>f xs…</code></td>
    </tr>
    <tr>
      <th rowspan="1" colspan="1" align="left">Kotlin</th>
      <td rowspan="1" colspan="2" align="center"><code>xs.zip(ys</code>[<code>, f</code>]<code>)</code></td>
      <td rowspan="1" colspan="1" align="center">–</td>
      <td rowspan="1" colspan="1" align="center"><code>zs.unzip()</code></td>
    </tr>
    <tr>
      <th rowspan="2" colspan="1" align="left">Python</th>
      <td rowspan="2" colspan="2" align="center"><code>zip(</code>[<code>xs…</code>]<code>)</code></td>
      <td rowspan="1" colspan="1" align="center"><code>zip_longest(xs…</code>[<code>, fillvalue=None</code>]<code>)</code></td>
      <td rowspan="2" colspan="1" align="center"><code>zip(&#42;zs)</code></td>
    </tr>
    <tr>
      <td rowspan="1" colspan="1" align="center"><code>map(None, xs…)</code> (2.x)</td>
    </tr>
    <tr>
      <th rowspan="1" colspan="1" align="left">Ruby</th>
      <td rowspan="1" colspan="3" align="center"><code>xs.zip(ys…)</code></td>
      <td rowspan="1" colspan="1" align="center"><code>zs.transpose</code></td>
    </tr>
    <tr>
      <th rowspan="1" colspan="1" align="left">Rust</th>
      <td rowspan="1" colspan="2" align="center"><code>xs.iter().zip(ys)</code></td>
      <td rowspan="1" colspan="1" align="center">–</td>
      <td rowspan="1" colspan="1" align="center"><code>zs.iter().cloned().unzip()</code></td>
    </tr>
    <tr>
      <th rowspan="2" colspan="1" align="left">Scala</th>
      <td rowspan="1" colspan="2" align="center"><code>(xs…).zipped</code></td>
      <td rowspan="2" colspan="1" align="center"><code>xs.zipAll(ys, x, y)</code></td>
      <td rowspan="2" colspan="1" align="center"><code>zs.unzip</code></td>
    </tr>
    <tr>
      <td rowspan="1" colspan="2" align="center"><code>xs.zip(ys…)</code></td>
    </tr>
    <tr>
      <th rowspan="1" colspan="1" align="left"><em>Swift</em></th>
      <td rowspan="1" colspan="2" align="center"><em><code>zip(xs, ys)</code></em></td>
      <td rowspan="1" colspan="1" align="center"><em>–</em></td>
      <td rowspan="1" colspan="1" align="center"><em>–</em></td>
    </tr>
  </tbody>
</table>

<sub>**Figure 2 — Overview of `zip` APIs in Other Languages** &nbsp; More specialised unary `zip` operations have been omitted, e.g., Elm's `List.indexedMap xs`, Python's `enumerate(xs)`, Kotlin's `xs.withIndex()` and `xs.zipWithNext()` and Scala's `xs.zipWithIndex`.</sub>

Most of the languages listed in [figure 2](#figure-2) provide a truncating `zip` operation. Those that don't, like Elm and Clojure, provide the more generalised n-ary [`map`](https://en.wikipedia.org/wiki/Map_%28higher-order_function%29) operation. Erlang and Haskell provide both, where the latter is named `zipwith` and `zipWith`, respectively. However, the number of languages that do provide a non-truncating `zip` operation lies lower and every language is opinionated about its name and design.

Naming wise: Ruby and D reuse `zip`, Python uses `zip_longest`, and Scala uses `zipAll`. Similarly, from a design perspective: Scala requires default elements that match the types of the underlying lists; D fills the missing elements of the shorter lists with a default value; and Python and Ruby fill the missing elements of the shorter lists with a null value. Ruby chooses its behaviour based on the length of the first list and Python allows us to set a default value for the missing elements of all lists, but not per list.

Swift's API Design Guidelines state that we should [stick to established terms of art](https://swift.org/documentation/api-design-guidelines#stick-to-established-meaning) and [embrace precedents](https://swift.org/documentation/api-design-guidelines#embrace-precedent), if applicable. But contrary to `zip` this operation doesn't have a highly precedented name. Reusing `zip(_:_:)` is [not possible](https://swift.org/documentation/api-design-guidelines#similar-methods-can-share-a-base-name) as both operations `return` different types. This means that we have to create a newly named member. A commonality that all precedents share is the `zip` prefix along with a reference to the length of the underlying lists in its name, arguments or documentation: Ruby mentions length in its documentation; D and Python use `longest` in their argument and name, respectively; and Scala uses `all` in its name.

Following this trend and given Swift's first-class support for `Optional`s, I propose to introduce a new global `func`tion named `zipLongest(_:_:)` along with its associated `return` type `ZipLongest2Sequence`. It creates a `Sequence` of 2-tuples from a 2-tuple of `Sequence`s by zipping their respective elements until the longest `Sequence` is exhausted, padding the latter elements of the longest `Sequence` with `nil`. Its API mirrors our existing one, its name has precedents in D and Python and its behaviour has precedents in Ruby and Python.

This addition requires cross-referencing documentation between `zip(_:_:)` and `zipLongest(_:_:)` that attends the user to their differences.

##### Examples

Here, we use it to compare two `Sequence`s for equality, effectively rewriting `==(_:_:)` and `Sequence.elementsEqual(_:)`:

```swift
print(zipLongest([0, 1, 2], [0, 1]).lazy.allSatisfy(==))
// Prints "false"
```

If we would have used `zip(_:_:)` here, we would have gotten a false positive:

```swift
print(zip([0, 1, 2], [0, 1]).lazy.allSatisfy(==))
// Prints "true"
```

And here, we use it to provide default values for the latter elements of the shortest `Sequence`:

```swift
let xs = (0...).prefix(Int.random(in: 0...5))
let ys = (0...).prefix(Int.random(in: 0...5))

print(Array(zipLongest(xs, ys).map { ($0.0 ?? 42, $0.1 ?? 42) }))
// Prints "[(0, 0), (1, 42), (2, 42), (3, 42)]"
```

#### `Zip2Index`

The conditional conformance of both `Zip2Sequence` and `ZipLongest2Sequence` to `Collection`, `BidirectionalCollection` and `RandomAccessCollection` requires the introduction of a new `Index` type. I propose putting it in the global scope—as the implementations of both `Zip2Sequence.Index` and `ZipLongest2Sequence.Index` are identical—and naming it `Zip2Index`.

#### `unzip()`

Most of the inverse `zip` operations listed in [figure 2](#figure-2) are named `unzip`, are bound to the global or list namespace, and `return` a tuple of lists. Notable exceptions are: Elm, that mixes both the n-ary `map` and `unzip` idioms; Kotlin, Ruby, Rust and Scala, which are bound to instances of lists; and Clojure, D and Ruby, which `return` a list of lists.

Next to embracing precedents, the Swift API Guidelines also [prefers methods and properties to global `func`tions](https://swift.org/documentation/api-design-guidelines#prefer-method-and-properties-to-functions) as a general convention. Unlike `zip(_:_:)` and `zipLongest(_:_:)`, it's possible to add this operation as an `extension` on a type. The primary argument *against* doing so would be that it splits the `zip` API up in two. However, the overall consensus in the pitch phase was that it reads better as a method; whereas `zip` typically comes first in a chain of operations, its inverse typically comes last.

I therefore propose to introduce `unzip()` as an `extension` on both `Sequence` and `LazyCollectionProtocol`. It creates a 2-tuple of `Sequence`s by, either eagerly or lazily, unzipping a `Sequence` of 2-tuples until it is exhausted.

##### Examples

Here, we use it to create a pair of `Sequence`s from a single `Sequence` which can then be operated on further:

```swift
import Foundation

let query = "MONKEY"
let (characters, names) =
  (0x1F601...0x1F64F)
    .compactMap(UnicodeScalar.init)
    .filter { $0.properties.name!.contains(query) }
    .map { (Character($0), $0.properties.name!) }
    .unzip()

print(characters, names)
// Prints "["🙈", "🙉", "🙊"] ["SEE-NO-EVIL MONKEY", "HEAR-NO-EVIL MONKEY", "SPEAK-NO-EVIL MONKEY"]"
```

#### Higher Arities

<img src="https://gist.githubusercontent.com/dennisvennink/27a31ac2d8fc6ef665ce6522bb73db39/raw/a12387421ecceeecfdd6b6e3a1d7078342d71496/arities.svg?sanitize=true" name="figure-3"/>

<sub>**Figure 3 — Overview of Arities of `zip` Operations in Other Languages** &nbsp; Sorted by type system, number of arities and alphabetical order. Lower is better.</sub>

[Figure 3](#figure-3) shows the correlation between a language's type system and its amount of supported arities. Most static languages have to resort to additional overloads or types. Notable outlier here is D, due to its variadic templating system. Most dynamic languages can provide any arity. Notable outlier here is Erlang, due to it uniquely identifying its functions by `name/arity`, e.g., `zip/2` or `unzip3/3`.

In the absence of [variadic generics](https://github.com/apple/swift/blob/master/docs/GenericsManifesto.md#variadic-generics) we currently have to create overloads and introduce new types for higher arities. As [SE-0015](0015-tuple-comparison-operators.md) introduced `==`, `!=`, `<`, `<=`, `>` and `>=` on tuples up to arity 6, I propose extending this to all existing and newly introduced `zip` operations.

## Detailed Design

Only arity 2 is detailed here. The algorithms are written in such a way that they scale to higher arities. This is done so that these higher arities can be generated by GYB. For the sake of brevity, the documentation and compiler directives have been stripped.

### Types

#### `Zip2Index`

`Zip2Index` is a `struct` that contains the indices of the underlying `Collection`s. Like [`Dictionary.Index`](https://developer.apple.com/documentation/swift/dictionary/index), its `init`ialiser is `internal` to protect the user from creating invalid indices. In `<(_:_:)`, we check to see if the indices hold their [asymmetric property](https://developer.apple.com/documentation/swift/comparable#overview) when compared.

##### Implementation

<details>
  <summary>Show / Hide</summary>

```swift
public struct Zip2Index <Collection1: Collection, Collection2: Collection> {
  public let index1: Collection1.Index
  public let index2: Collection2.Index

  internal init (_ index1: Collection1.Index, _ index2: Collection2.Index) {
    self.index1 = index1
    self.index2 = index2
  }
}

extension Zip2Index: Comparable {
  public static func < (lhs: Zip2Index, rhs: Zip2Index) -> Bool {
    precondition(
      lhs.index1 <= rhs.index1 && lhs.index2 <= rhs.index2 || lhs.index1 >= rhs.index1 && lhs.index2 >= rhs.index2,
      "Indices failed to hold their asymmetric property when compared"
    )

    return (lhs.index1, lhs.index2) < (rhs.index1, rhs.index2)
  }
}
```
</details>

#### `Zip2Sequence`

`Zip2Sequence` places additional conditional constraints on the conformances to both `BidirectionalCollection` and `RandomAccessCollection`. Therefore, the default implementations of these two protocols that refine **and** are used by default implementations of members lower in the `protocol` hierarchy need to be reimplemented in an unconstrained `extension`.

In the case of `BidirectionalCollection`, these are:

1. `distance(from:to:)`,
2. `index(_:offsetBy:)` and
3. `index(_:offsetBy:limitedBy:)`.

In the case of `RandomAccessCollection`, this is `index(_:offsetBy:limitedBy:)`.

These implementations should trap when the underlying `Collection`s don't support backwards traversal and should run in O(*1*) when the underlying `Collection`s conform to `RandomAccessCollection`.

`Zip2Sequence.Index` follows the indices of the underlying `Sequence`s until the shortest `Sequence` is exhausted. Its `endIndex` consists of the `endIndex` of both `Sequence`s, skipping the latter indices of the longest `Sequence`.

##### Implementation

<details>
  <summary>Show / Hide</summary>

```swift
public struct Zip2Sequence <Sequence1: Sequence, Sequence2: Sequence> {
  internal let _sequence1: Sequence1
  internal let _sequence2: Sequence2

  internal init (_ sequence1: Sequence1, _ sequence2: Sequence2) {
    _sequence1 = sequence1
    _sequence2 = sequence2
  }
}
```

```swift
extension Zip2Sequence {
  public struct Iterator {
    internal var _baseStream1: Sequence1.Iterator
    internal var _baseStream2: Sequence2.Iterator
    internal var _reachedEnd: Bool = false

    internal init (_ iterator1: Sequence1.Iterator, _ iterator2: Sequence2.Iterator) {
      _baseStream1 = iterator1
      _baseStream2 = iterator2
    }
  }
}
```

```swift
extension Zip2Sequence.Iterator: IteratorProtocol {
  public typealias Element = (Sequence1.Element, Sequence2.Element)

  public mutating func next () -> Element? {
    guard !_reachedEnd else {
      return nil
    }

    guard let element1 = _baseStream1.next(), let element2 = _baseStream2.next() else {
      _reachedEnd.toggle()

      return nil
    }

    return (element1, element2)
  }
}
```

```swift
extension Zip2Sequence: Sequence {
  public typealias Element = (Sequence1.Element, Sequence2.Element)

  public __consuming func makeIterator () -> Iterator {
    return Iterator(_sequence1.makeIterator(), _sequence2.makeIterator())
  }

  public var underestimatedCount: Int {
    return Swift.min(_sequence1.underestimatedCount, _sequence2.underestimatedCount)
  }
}
```

```swift
extension Zip2Sequence: Collection where Sequence1: Collection, Sequence2: Collection {
  public typealias Index = Zip2Index<Sequence1, Sequence2>

  public var startIndex: Index {
    return _sequence1.isEmpty || _sequence2.isEmpty ?
      endIndex :
      Index(_sequence1.startIndex, _sequence2.startIndex)
  }

  public var endIndex: Index {
    return Index(_sequence1.endIndex, _sequence2.endIndex)
  }

  public subscript (position: Index) -> Element {
    precondition(position != endIndex, "Index out of range")

    return (_sequence1[position.index1], _sequence2[position.index2])
  }

  public func index (after i: Index) -> Index {
    precondition(i != endIndex, "Cannot increment beyond endIndex")

    let index1 = _sequence1.index(after: i.index1)
    let index2 = _sequence2.index(after: i.index2)

    return index1 == _sequence1.endIndex || index2 == _sequence2.endIndex ?
      endIndex :
      Index(index1, index2)
  }
}
```

```swift
extension Zip2Sequence: BidirectionalCollection where Sequence1: RandomAccessCollection, Sequence2: RandomAccessCollection {
  public func index (before i: Index) -> Index {
    precondition(i != startIndex, "Cannot decrement beyond startIndex")

    return index(i, offsetBy: -1)
  }
}
```

```swift
extension Zip2Sequence: RandomAccessCollection where Sequence1: RandomAccessCollection, Sequence2: RandomAccessCollection {}
```

```swift
extension Zip2Sequence: LazySequenceProtocol where Sequence1: LazySequenceProtocol, Sequence2: LazySequenceProtocol {}
```

```swift
extension Zip2Sequence where Sequence1: Collection, Sequence2: Collection {
  public func distance (from start: Index, to end: Index) -> Int {
    guard start != end else {
      return 0
    }

    let distance1 = _sequence1.distance(from: start.index1, to: end.index1)
    let distance2 = _sequence2.distance(from: start.index2, to: end.index2)

    return start < end ?
      Swift.min(distance1, distance2) :
      Swift.max(distance1, distance2)
  }

  public func index (_ i: Index, offsetBy distance: Int) -> Index {
    let limit = distance < 0 ? startIndex : endIndex
    let offsetIndex = index(i, offsetBy: distance, limitedBy: limit)

    precondition(offsetIndex != nil, "Index out of range")

    return offsetIndex!
  }

  public func index (_ i: Index, offsetBy distance: Int, limitedBy limit: Index) -> Index? {
    precondition(startIndex <= i && i <= endIndex, "Index out of range")

    guard distance != 0 else {
      return i
    }

    let distanceFromStartIndexToIndex1 = _sequence1.distance(from: _sequence1.startIndex, to: i.index1)
    let distanceFromStartIndexToIndex2 = _sequence2.distance(from: _sequence2.startIndex, to: i.index2)
    let distanceFromStartIndexToOffsetIndex = Swift.min(distanceFromStartIndexToIndex1, distanceFromStartIndexToIndex2) + distance
    let count1 = _sequence1.count
    let count2 = _sequence2.count
    let count = Swift.min(count1, count2)

    precondition(
      0 <= distanceFromStartIndexToOffsetIndex && distanceFromStartIndexToOffsetIndex <= count,
      "Index out of range"
    )

    guard let offsetIndex1 = _sequence1.index(
            i.index1,
            offsetBy: (distanceFromStartIndexToOffsetIndex == count ? count1 : distanceFromStartIndexToOffsetIndex) - distanceFromStartIndexToIndex1,
            limitedBy: limit.index1
          ),
          let offsetIndex2 = _sequence2.index(
            i.index2,
            offsetBy: (distanceFromStartIndexToOffsetIndex == count ? count2 : distanceFromStartIndexToOffsetIndex) - distanceFromStartIndexToIndex2,
            limitedBy: limit.index2
          ) else {
      return nil
    }

    return Index(offsetIndex1, offsetIndex2)
  }
}
```
</details>

#### `ZipLongest2Sequence`

The same conditional constraints that apply to `Zip2Sequence` also apply to `ZipLongest2Sequence`.

`ZipLongest2Sequence.Index` works slightly different from `Zip2Sequence.Index`; it follows the indices of the underlying `Sequence`s until the longest `Sequence` is exhausted. If a `Sequence` is shorter and the index is at or past the end of the shortest `Sequence`, the index of the shortest `Sequence` will be identical to its `endIndex`.

##### Implementation

<details>
  <summary>Show / Hide</summary>

```swift
public struct ZipLongest2Sequence <Sequence1: Sequence, Sequence2: Sequence> {
  internal let _sequence1: Sequence1
  internal let _sequence2: Sequence2

  internal init (_ sequence1: Sequence1, _ sequence2: Sequence2) {
    _sequence1 = sequence1
    _sequence2 = sequence2
  }
}
```

```swift
extension ZipLongest2Sequence {
  public struct Iterator {
    internal var _baseStream1: Sequence1.Iterator
    internal var _baseStream2: Sequence2.Iterator

    internal init (_ iterator1: Sequence1.Iterator, _ iterator2: Sequence2.Iterator) {
      _baseStream1 = iterator1
      _baseStream2 = iterator2
    }
  }
}
```

```swift
extension ZipLongest2Sequence.Iterator: IteratorProtocol {
  public typealias Element = (Sequence1.Element?, Sequence2.Element?)

  public mutating func next () -> Element? {
    let element1 = _baseStream1.next()
    let element2 = _baseStream2.next()

    return element1 == nil && element2 == nil ?
      nil :
      (element1, element2)
  }
}
```

```swift
extension ZipLongest2Sequence: Sequence {
  public typealias Element = (Sequence1.Element?, Sequence2.Element?)

  public __consuming func makeIterator () -> Iterator {
    return Iterator(_sequence1.makeIterator(), _sequence2.makeIterator())
  }

  public var underestimatedCount: Int {
    return Swift.max(_sequence1.underestimatedCount, _sequence2.underestimatedCount)
  }
}
```

```swift
extension ZipLongest2Sequence: Collection where Sequence1: Collection, Sequence2: Collection {
  public typealias Index = Zip2Index<Sequence1, Sequence2>

  public var startIndex: Index {
    return Index(_sequence1.startIndex, _sequence2.startIndex)
  }

  public var endIndex: Index {
    return Index(_sequence1.endIndex, _sequence2.endIndex)
  }

  public subscript (position: Index) -> Element {
    precondition(position != endIndex, "Index out of range")

    return (
      position.index1 == _sequence1.endIndex ?
        nil :
        _sequence1[position.index1],
      position.index2 == _sequence2.endIndex ?
        nil :
        _sequence2[position.index2]
    )
  }

  public func index (after i: Index) -> Index {
    precondition(i != endIndex, "Cannot increment beyond endIndex")

    return Index(
      i.index1 == _sequence1.endIndex ?
        _sequence1.endIndex :
        _sequence1.index(after: i.index1),
      i.index2 == _sequence2.endIndex ?
        _sequence2.endIndex :
        _sequence2.index(after: i.index2)
    )
  }
}
```

```swift
extension ZipLongest2Sequence: BidirectionalCollection where Sequence1: RandomAccessCollection, Sequence2: RandomAccessCollection {
  public func index (before i: Index) -> Index {
    precondition(i != startIndex, "Cannot decrement beyond startIndex")

    return index(i, offsetBy: -1)
  }
}
```

```swift
extension ZipLongest2Sequence: RandomAccessCollection where Sequence1: RandomAccessCollection, Sequence2: RandomAccessCollection {}
```

```swift
extension ZipLongest2Sequence: LazySequenceProtocol where Sequence1: LazySequenceProtocol, Sequence2: LazySequenceProtocol {}
```

```swift
extension ZipLongest2Sequence where Sequence1: Collection, Sequence2: Collection {
  public func distance (from start: Index, to end: Index) -> Int {
    guard start != end else {
      return 0
    }

    let distance1 = _sequence1.distance(from: start.index1, to: end.index1)
    let distance2 = _sequence2.distance(from: start.index2, to: end.index2)

    return start < end ?
      Swift.max(distance1, distance2) :
      Swift.min(distance1, distance2)
  }

  public func index (_ i: Index, offsetBy distance: Int) -> Index {
    let limit = distance < 0 ? startIndex : endIndex
    let offsetIndex = index(i, offsetBy: distance, limitedBy: limit)

    precondition(offsetIndex != nil, "Index out of range")

    return offsetIndex!
  }

  public func index (_ i: Index, offsetBy distance: Int, limitedBy limit: Index) -> Index? {
    precondition(startIndex <= i && i <= endIndex, "i is not a valid index")

    guard distance != 0 else {
      return i
    }

    let distanceFromStartIndexToIndex1 = _sequence1.distance(from: _sequence1.startIndex, to: i.index1)
    let distanceFromStartIndexToIndex2 = _sequence2.distance(from: _sequence2.startIndex, to: i.index2)
    let distanceFromStartIndexToOffsetIndex = Swift.max(distanceFromStartIndexToIndex1, distanceFromStartIndexToIndex2) + distance
    let count1 = _sequence1.count
    let count2 = _sequence2.count
    let count = Swift.max(count1, count2)

    precondition(
      0 <= distanceFromStartIndexToOffsetIndex && distanceFromStartIndexToOffsetIndex <= count,
      "Index out of range"
    )

    let offsetIndex = Index(
      _sequence1.index(
        i.index1,
        offsetBy: (distanceFromStartIndexToOffsetIndex >= count1 ? count1 : distanceFromStartIndexToOffsetIndex) - distanceFromStartIndexToIndex1
      ),
      _sequence2.index(
        i.index2,
        offsetBy: (distanceFromStartIndexToOffsetIndex >= count2 ? count2 : distanceFromStartIndexToOffsetIndex) - distanceFromStartIndexToIndex2
      )
    )

    guard distance < 0 ?
            !(offsetIndex < limit && limit <= i) :
            !(i <= limit && limit < offsetIndex) else {
      return nil
    }

    return offsetIndex
  }
}
```
</details>

### Global Functions

#### `zip(_:_:)`

##### Implementation

<details>
  <summary>Show / Hide</summary>

```swift
public func zip <Sequence1: Sequence, Sequence2: Sequence> (_ sequence1: Sequence1, _ sequence2: Sequence2) -> Zip2Sequence<Sequence1, Sequence2> {
  return Zip2Sequence(sequence1, sequence2)
}
```
</details>

#### `zipLongest(_:_:)`

##### Implementation

<details>
  <summary>Show / Hide</summary>

```swift
public func zipLongest <Sequence1: Sequence, Sequence2: Sequence> (_ sequence1: Sequence1, _ sequence2: Sequence2) -> ZipLongest2Sequence<Sequence1, Sequence2> {
  return ZipLongest2Sequence(sequence1, sequence2)
}
```
</details>

### `extension`s on `Sequence` and `LazyCollectionProtocol`

#### `unzip()`

As the lazy variant of `unzip()` takes multiple passes through the underlying `Sequence`, `Base` needs to conform to both `Collection` and `LazySequenceProtocol`.

##### Implementation

<details>
  <summary>Show / Hide</summary>

```swift
extension Sequence {
  public func unzip <Element1, Element2> () -> ([Element1], [Element2]) where Element == (Element1, Element2) {
    var result1 = [Element1]()
    var result2 = [Element2]()

    result1.reserveCapacity(underestimatedCount)
    result2.reserveCapacity(underestimatedCount)

    for element in self {
      result1.append(element.0)
      result2.append(element.1)
    }

    return (result1, result2)
  }
}
```

```swift
extension LazyCollectionProtocol {
  public func unzip <Element1, Element2> () -> (LazyMapCollection<Elements, Element1>, LazyMapCollection<Elements, Element2>) where Elements.Element == (Element1, Element2) {
    return (map({ $0.0 }), map({ $0.1 }))
  }
}
```
</details>

## Source Compatibility

This proposal doesn't introduce source compatibility changes.

## Effect on ABI Stability

This proposal doesn't change the ABI of existing language features.

## Effect on API Resilience

New members and additional `protocol` conformances can be added to `Zip`[`2`–`6`]`Sequence` and `ZipLongest`[`2`–`6`]`Sequence` without breaking their ABI.

For `Zip`[`2`–`6`]`Sequence`, conformance to `MutableCollection` and `RangeReplaceableCollection` is possible when the underlying `Sequence`s conform to `MutableCollection` and `RangeReplaceableCollection`, respectively. Conformance to `ExpressibleByArrayLiteral` and `ExpressibleByDictionaryLiteral` is possible when the underlying `Sequence`s conform to `RangeReplaceableCollection`. Similar conformances for `ZipLongest`[`2`–`6`]`Sequence` are not possible.

In the case of `ZipLongest`[`2`–`6`]`Sequence`, interest has been shown in members that provide views on the common prefix and the non-truncated suffix, as well as operations that calculate the divergence or convergence of the elements of the underlying `Sequence`s according to some predicate. To put things in perspective, Rust and Scala are the only other languages from [figure 2](#figure-2) that introduce new types for their `zip` operations. However, these types don't introduce any new `zip`-specific operations.

## Alternatives Considered

### `zip(_:default:_:default:)`

`zip(_:default:_:default:)` is easily composable from `zipLongest(_:_:)`, `map(_:)` and `??(_:_:)`:

```swift
zipLongest(xs, ys).map { ($0.0 ?? 42, $0.1 ?? 42) }
```
