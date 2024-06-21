# Conditional conformances

* Proposal: [SE-0143](0143-conditional-conformances.md)
* Author: [Doug Gregor](https://github.com/DougGregor)
* Review Manager: [Joe Groff](https://github.com/jckarter)
* Status: **Implemented (Swift 4.2)**
* Decision Notes: [Review extended](https://forums.swift.org/t/review-se-0143-conditional-conformances/4130/10), [Rationale](https://forums.swift.org/t/accepted-se-0143-conditional-conformances/4537)
* Previous Revision: [1](https://github.com/swiftlang/swift-evolution/blob/91725ee83fa34c81942a634dcdfa9d2441fbd853/proposals/0143-conditional-conformances.md)

## Introduction

Conditional conformances express the notion that a generic type will
conform to a particular protocol only when its type arguments meet
certain requirements. For example, the `Array` collection can
implement the `Equatable` protocol only when its elements are
themselves `Equatable`, which can be expressed via the following
conditional conformance on `Equatable`:

```swift
extension Array: Equatable where Element: Equatable {
  static func ==(lhs: Array<Element>, rhs: Array<Element>) -> Bool { ... }
}
```

This feature is part of the [generics
manifesto](https://github.com/apple/swift/blob/master/docs/GenericsManifesto.md#conditional-conformances-)
because it's something that fits naturally into the generics model and
is expected to have a high impact on the Swift standard library.

Swift-evolution thread: [here](https://forums.swift.org/t/proposal-draft-conditional-conformances/4110)

## Motivation

Conditional conformances address a hole in the composability of the
generics system. Continuing the `Array` example from above, it's
always been the case that one could use the `==` operator on two
arrays of `Equatable` type, e.g., `[Int] == [Int]` would
succeed. However, it doesn't compose: arrays of arrays of `Equatable`
types cannot be compared (e.g., `[[Int]] == [[Int]]` will fail to
compile) because, even though there is an `==` for arrays of
`Equatable` type, the arrays themselves are never `Equatable`.

Conditional conformances are particularly powerful when building
generic adapter types, which are intended to reflect the capabilities
of their type arguments. For example, consider the "lazy"
functionality of the Swift standard library's collections: using the
`lazy` member of a sequence produces a lazy adapter that conforms to
the `Sequence` protocol, while using the `lazy` member of a collection
produces a lazy adapter that conforms to the `Collection`
protocol. In Swift 3, the only way to model this is with different
types. For example, the Swift standard library has four similar
generic types to handle a lazy collection: `LazySequence`,
`LazyCollection`, `LazyBidirectionalCollection`, and
`LazyRandomAccessCollection`. The Swift standard library uses
overloading of the `lazy` property to decide among these:

```swift
extension Sequence {
  var lazy: LazySequence<Self> { ... }
}

extension Collection {
  var lazy: LazyCollection<Self> { ... }
}

extension BidirectionalCollection {
  var lazy: LazyBidirectionalCollection<Self> { ... }
}

extension RandomAccessCollection {
  var lazy: LazyRandomAccessCollection<Self> { ... }
}
```

This approach causes an enormous amount of repetition, and doesn't
scale well because each more-capable type has to re-implement (or
somehow forward the implementation of) all of the APIs of the
less-capable versions. With conditional conformances, one can provide
a single generic wrapper type whose basic requirements meet the lowest
common denominator (e.g., `Sequence`), but which scale their
capabilities with their type argument (e.g., the `LazySequence`
conforms to `Collection` when the type argument does, and so on).

## Proposed solution
In a nutshell, the proposed solution is to allow a constrained
extension of a `struct`, `enum`, or `class` (but [not a protocol](#alternatives-considered)) to declare protocol
conformances. No additional syntax is necessary for this change,
because it already exists in the grammar; rather, this proposal
removes the limitation that results in the following error:

```
t.swift:1:1: error: extension of type 'Array' with constraints cannot have an inheritance clause
extension Array: Equatable where Element: Equatable { }
^                ~~~~~~~~~
```

Conditional conformances can only be used when the additional
requirements of the constrained extension are satisfied. For example,
given the aforementioned `Array` conformance to `Equatable`:

```swift
func f<T: Equatable>(_: T) { ... }

struct NotEquatable { }

func test(a1: [Int], a2: [NotEquatable]) {
  f(a1)    // okay: [Int] conforms to Equatable because Int conforms to Equatable
  f(a2)    // error: [NotEquatable] does not conform to Equatable because NotEquatable has no conformance to Equatable
}
```

Conditional conformances also have a run-time aspect, because a
dynamic check for a protocol conformance might rely on the evaluation
of the extra requirements needed to successfully use a conditional
conformance. For example:

```swift
protocol P {
  func doSomething()
}

struct S: P {
  func doSomething() { print("S") }
}

// Array conforms to P if it's element type conforms to P
extension Array: P where Element: P {
  func doSomething() {
    for value in self {
      value.doSomething()
    }
  }
}

// Dynamically query and use conformance to P.
func doSomethingIfP(_ value: Any) {
  if let p = value as? P {
    p.doSomething()
  } else {
    print("Not a P")
  }
}

doSomethingIfP([S(), S(), S()]) // prints "S" three times
doSomethingIfP([1, 2, 3])       // prints "Not a P"
```

The `if-let` in `doSomethingIfP(_:)` dynamically queries whether the type stored in `value` conforms to the protocol `P`. In the case of an `Array`, that conformance is conditional, which requires another dynamic lookup to determine whether the element type conforms to `P`: in the first call to `doSomethingIfP(_:)`, the lookup finds the conformance of `S` to `P`. In the second case, there is no conformance of `Int` to `P`, so the conditional conformance cannot be used. The desire for this dynamic behavior motivates some of the design decisions in this proposal.

## Detailed design
Most of the semantics of conditional conformances are
obvious. However, there are a number of issues (mostly involving
multiple conformances) that require more in-depth design.

### Multiple conformances

Swift already bans programs that attempt to make the same type conform
to the same protocol twice, e.g.:

```swift
protocol P { }

struct X : P { }
extension X : P { } // error: X already stated conformance to P
```

This existing ban on multiple conformances is extended to conditional
conformances, including attempts to conform to the same protocol in
two different ways. For example:

```swift
struct SomeWrapper<Wrapped> {
  let wrapped: Wrapped
}

protocol HasIdentity {
  static func ===(lhs: Self, rhs: Self) -> Bool
}

extension SomeWrapper: Equatable where Wrapped: Equatable {
  static func ==(lhs: SomeWrapper<Wrapped>, rhs: SomeWrapper<Wrapped>) -> Bool {
    return lhs.wrapped == rhs.wrapped
  }
}

// error: SomeWrapper already stated conformance to Equatable
extension SomeWrapper: Equatable where Wrapped: HasIdentity {
  static func ==(lhs: SomeWrapper<Wrapped>, rhs: SomeWrapper<Wrapped>) -> Bool {
    return lhs.wrapped === rhs.wrapped
  }
}
```

Furthermore, for consistency, the ban extends even to multiple
conformances that are "clearly" disjoint, e.g.,

```swift
extension SomeWrapper: Equatable where Wrapped == Int {
  static func ==(lhs: SomeWrapper<Int>, rhs: SomeWrapper<Int>) -> Bool {
    return lhs.wrapped == rhs.wrapped
  }
}

// error: SomeWrapper already stated conformance to Equatable
extension SomeWrapper: Equatable where Wrapped == String {
  static func ==(lhs: SomeWrapper<String>, rhs: SomeWrapper<String>) -> Bool {
    return lhs.wrapped == rhs.wrapped
  }
}
```

The section [overlapping
conformances](#overlapping-conformances) describes some of the
complexities introduced by multiple conformances, to justify their
exclusion from this proposal. A follow-on proposal could introduce
support for multiple conformances, but should likely also cover related
features such as [private
conformances](https://github.com/apple/swift/blob/master/docs/GenericsManifesto.md#private-conformances)
that are orthogonal to conditional conformances.


### Implied conditional conformances

Stating a non-conditional conformance to a protocol implicitly states
conformances to any of the protocols that the protocol inherits: one
can declare conformance to the `Collection` protocol, and it implies
conformance to `Sequence` as well. However, with conditional
conformances, the constraints for the conformance to the inherited
protocol may not be clear, and even when there is a clear choice, it
will often be incorrect, so the conformance to the inherited protocol
will need to be stated explicitly. For example, for the first case:

```swift
protocol P { }
protocol Q : P { }
protocol R : P { }

struct X<T> { }

extension X: Q where T: Q { }
extension X: R where T: R { }

// error: X does not conform to protocol P; add
//
//   extension X: P where <#constraints#> { ... }
//
// to state conformance to P.
```

Note that both of the constrained extensions could imply the
conformance to `P`. However, because the two extensions have disjoint
sets of constraints (one requires `T: Q`, the other `T: R`), it
becomes unclear which constraints should apply to the conformance to
`P`: picking one set of constraints (e.g. `T: Q`, from the conformance
of `X` to `Q`) makes the inherited conformance unusable for `X`
instances where `T: R`, which would break type safety (because we
could have `X` instances that conform to `R` but not `P`!). Moreover,
the previously-discussed ban on multiple conformances prohibits
introducing two different conformances of `X` to `P` (one where `T: Q`
and one where `T: R`). Therefore, the program above is ill-formed, and
the correct fix is for the user to introduce an explicit conformance
of `X` to `P` with the appropriate set of constraints, e.g.:

```swift
extension X: P where T: P { }
```

For the second problem mentioned above, when there is an obvious set
of requirements to use in an implied conformance, it is likely to be
wrong, because of how often conditional conformances are used for
wrapper types. For instance:

```swift
protocol R: P { }
protocol S: R { }

struct Y<T> { }

extension Y: R where T: R { }
extension Y: S where T: S { }
```

The conformances of `Y: R` and `Y: S` both imply the conformance
`Y: P`, however the constraints `T: R` are less specialized (more
general) than the constraints `T: S`, because every `S` is also an
`R`. Therefore, it could be that `Y` will conform to `P` when `T: R`, e.g.:

```swift
/// compiler produces the following implied inherited conformance:
extension Y: P where T: R { }
```

However, it is likely that the best conformance is actually the more
relaxed (that is, applicable for more choices of `T`):

``` swift
extension Y: P where T: P { }
```

This is the case for almost all wrappers for the
`Sequence`/`Collection`/`BidirectionalCollection`/... hierarchy (for
instance, as discussed below, `Slice : BidirectionalCollection where
Base : BidirectionalCollection` and similarly for
`RandomAccessCollection`), and for most types conforming to several of
`Equatable`, `Comparable` and `Hashable`.

Implicitly constructing these conformances could be okay if it were
possible to relax the overly-strong requirements when they're noticed
in future. However, it can be backwards incompatible, and so not doing
it implicitly is defaulting to the safer option. The backwards
incompatibility comes from how requirements are inferred in function
signatures: given `struct Z<A: P> {}`, Swift notices that a
declaration `func foo<A>(x: Z<Y<A>>)` requires that `Y<A> : P`, since
it is used in `Z`, and thus, if the implicit inherited conformance
above existed, `A: R`. This conformance is part of the function's
signature (and mangling!) and is available to be used inside `foo`:
that function can use requirements from `R` on values of type `A`. If
the library declaring `Y` was to change to the declaration of the
conformance `Y: P`, the inferred requirement becomes `A: P`, which
changes the `foo`'s mangled name, and what can be done with values of
type `A`. This breaks both API and ABI compatibility.

(Note: the inference above is driven by having a unique conformance,
and thus `Y: P` if *and only if* `A: P`. If overlapping conformances
were allowed, this inference would not be possible. A possible
alternative that's more directly future-proof with overlapping
conformances would be to disable this sort of inference from
conditional conformances, and instead require the user to write `func
foo<A: P>`. This could also allow the conformances to be implied,
since it would no longer be such a backwards-compatibility problem.)

On the other hand, not allowing implicit inherited conformances means
that one cannot insert a superprotocol to an existing protocol: for
instance, if the second example started as `protocol R { }` and was
changed to `protocol R: P { }`. However, we believe this is already
incompatible, for unrelated reasons.

Finally, it is a small change to get implicit behaviour explicitly, by
adding the conformance declaration to the extension that would be
implying the conformance. For instance, if it is correct for the
second example to have `T: R` as the requirement on `Y: P`, the `Y: R`
extension only needs to be changed to include `, P`:

``` swift
extension Y: R, P where T: R { }
```

This is something compilers can, and should, suggest as a fixit.

## Standard library adoption

Adopt conditional conformances to make various standard library types
that already have a suitable `==` conform to `Equatable`. Specifically:

```swift
extension Optional: Equatable where Wrapped: Equatable { /*== already exists */ }
extension Array: Equatable where Element: Equatable { /*== already exists */ }
extension ArraySlice: Equatable where Element: Equatable { /*== already exists */ }
extension ContiguousArray: Equatable where Element: Equatable { /*== already exists */ }
extension Dictionary: Equatable where Value: Equatable { /*== already exists */ }
```

In addition, implement conditional conformances to `Hashable` for the
types above, as well as `Range` and `ClosedRange`:

```swift
extension Optional: Hashable where Wrapped: Hashable { /*...*/ }
extension Array: Hashable where Element: Hashable { /*...*/ }
extension ArraySlice: Hashable where Element: Hashable { /*...*/ }
extension ContiguousArray: Hashable where Element: Hashable { /*...*/ }
extension Dictionary: Hashable where Value: Hashable { /*...*/ }
extension Range: Hashable where Bound: Hashable { /*...*/ }
extension ClosedRange: Hashable where Bound: Hashable { /*...*/ }
```

While the standard library did not previously provide existing
implementations of `hashValue` for these types, conditional `Hashable`
conformance is a natural expectation for them.

Note that `Set` is already (unconditionally) `Equatable` and `Hashable`.

In addition, it is intended that the standard library adopt conditional conformance
to collapse a number of "variants" of base types where other generic parameters
enable conformance to further protocols.

For example, there is a type:

```swift
ReversedCollection<Base: BidirectionalCollection>: BidirectionalCollection
```

that provides a low-cost lazy reversal of any bidirecitonal collection.
There is a variation on that type,

```swift
ReversedRandomAccessCollection<Base: RandomAccessCollection>: RandomAccessCollection
```

 that additionally conforms to `RandomAccessCollection` when its base does.
Users create these types via the `reversed()` extension method on
`BidirectionalCollection` and `RandomAccessCollection` respectively.

With conditional conformance, the `ReversedRandomAccessCollection` variant can
be replaced with a conditional extension:

```swift
extension ReversedCollection: RandomAccessCollection where Base: RandomAccessCollection { }

@available(*, deprecated, renamed: "ReversedCollection")
public typealias ReversedRandomAccessCollection<T: RandomAccessCollection> = ReversedCollection<T>
```

Similar techniques can be used for variants of `Slice`, `LazySequence`,
`DefaultIndices`, `Range` and others. These refactorings are considered an
implementation detail of the existing functionality standard library and should
be applied across the board where applicable.

## Source compatibility

From the language perspective, conditional conformances are purely additive. They introduce no new syntax, but instead provide semantics for existing syntax---an extension that both declares a protocol conformance and has a `where` clause---whose use currently results in a type checker failure. That said, this is a feature that is expected to be widely adopted within the Swift standard library, which may indirectly affect source compatibility.

## Effect on ABI Stability

As noted above, there are a number of places where the standard library is expected to adopt this feature, which fall into two classes:

1. Improve composability: the example in the [introduction](#introduction) made `Array` conform to `Equatable` when its element type does; there are many places in the Swift standard library that could benefit from this form of conditional conformance, particularly so that collections and other types that contain values (e.g., `Optional`) can compose better with generic algorithms. Most of these changes won't be ABI- or source-breaking, because they're additive.
2. Eliminating repetition: the `lazy` wrappers described in the [motivation](#motivation) section could be collapsed into a single wrapper with several conditional conformances. A similar refactoring could also be applied to the range abstractions and slice types in the standard library, making the library itself simpler and smaller. All of these changes are potentially source-breaking and ABI-breaking, because they would remove types that could be used in Swift 3 code. However, there are mitigations: generic typealiases could provide source compatibility to Swift 3 clients, and the ABI-breaking aspect is only relevant if conditional conformances and the standard library changes they imply aren't part of Swift 4.

Aside from the standard library, conditional conformances have an impact on the Swift runtime, which will require specific support to handle dynamic casting. If that runtime support is not available once ABI stability has been declared, then introducing conditional conformances in a later language version either means the feature cannot be deployed backward or that it would provide only more limited, static behavior when used on older runtimes. Hence, there is significant motivation for doing this feature as part of Swift 4. Even if we waited to introduce conditional conformances, we would want to include a hook in the runtime to allow them to be implemented later, to avoid future backward-compatibility issues.

## Effect on Resilience

One of the primary goals of Swift 4 is resilience, which allows libraries to evolve without breaking binary compatibility with the applications that use them. While the specific details of the impact of conditional conformances on resilience will be captured in a more-complete proposal on resilience, possible rules are summarized here:

* A conditional conformance cannot be removed in the new version of a library, because existing clients might depend on it.
* A conditional conformance can be added in a new version of a library, roughly following the rules described in the [library evolution document](https://github.com/apple/swift/blob/master/docs/LibraryEvolution.rst#new-conformances). The conformance itself will need to be annotated with the version in which it was introduced.
* A conditional conformance can be *generalized* in a new version of the library, i.e., it can be effectively replaced by a (possibly conditional) conformance in a new version of the library that is less specialized than the conditional conformance in the older version of the library. For example.

  ```swift
  public struct X<T> { }
  
  // Conformance in version 1.0
  public extension X: Sequence where T: Collection { ... }
  
  // Can be replaced by this less-specialized conformance in version 1.1
  public extension X: Sequence where T: Sequence { ... }
  ```
  
  Such conformances would likely need some kind of annotation.

## Alternatives considered

### Overlapping conformances

As noted in the section on [multiple
conformances](#multiple-conformances), Swift already bans programs
that attempt to make the same type conform to the same protocol
twice. This proposal extends the ban to cases where the conformances
are conditional. Reconsider the example from that section:

```swift
struct SomeWrapper<Wrapped> {
  let wrapped: Wrapped
}

protocol HasIdentity {
  static func ===(lhs: Self, rhs: Self) -> Bool
}

extension SomeWrapper: Equatable where Wrapped: Equatable {
  static func ==(lhs: SomeWrapper<Wrapped>, rhs: SomeWrapper<Wrapped>) -> Bool {
    return lhs.wrapped == rhs.wrapped
  }
}

extension SomeWrapper: Equatable where Wrapped: HasIdentity {
  static func ==(lhs: SomeWrapper<Wrapped>, rhs: SomeWrapper<Wrapped>) -> Bool {
    return lhs.wrapped === rhs.wrapped
  }
}
```

Note that, for an arbitrary type `T`, there are four potential answers to
the question of whether `SomeWrapper<T>` conforms to `Equatable`:

1. No, it does not conform because `T` is neither `Equatable` nor
`HasIdentity`.
2. Yes, it conforms via the first extension of `SomeWrapper` because
`T` conforms to `Equatable`.
3. Yes, it conforms via the second extension of `SomeWrapper` because
`T` conforms to `HasIdentity`.
4. Ambiguity, because `T` conforms to both `Equatable` and
`HasIdentity`.

It is due to the possibility of #4 occurring that we refer to the two conditional conformances in the example as *overlapping*. There are designs that would allow one to address the ambiguity, for example, by writing a third conditional conformance that addresses #4:

```swift
// Possible tie-breaker conformance
extension SomeWrapper: Equatable where Wrapped: Equatable & HasIdentity, {
  static func ==(lhs: SomeWrapper<Wrapped>, rhs: SomeWrapper<Wrapped>) -> Bool {
    return lhs.wrapped == rhs.wrapped
  }
}
```

The design is consistent, because this third conditional conformance is more *specialized* than either of the first two conditional conformances, meaning that its requirements are a strict superset of the requirements of those two conditional conformances. However, there are a few downsides to such a system:

1. To address all possible ambiguities, one has to write a conditional conformance for every plausible combination of overlapping requirements. To *statically* resolve all ambiguities, one must also cover nonsensical combinations where the two requirements are mutually exclusive (or invent a way to state mutual-exclusivity).
2. It is no longer possible to uniquely say what is required to make a generic type conform to a protocol, because there might be several unrelated possibilities. This makes reasoning about the whole system more complex, because it admits divergent interfaces for the same generic type based on their type arguments. At its extreme, this invites the kind of cleverness we've seen in the C++ community with template metaprogramming, which is something Swift has sought to avoid.
3. All of the disambiguation machinery required at compile time (e.g., to determine whether one conditional conformance is more specialized than another to order them) also needs to implements in the run-time, as part of the dynamic casting machinery. One must also address the possibility of ambiguities occurring at run-time. This is both a sharp increase in the complexity of the system and a potential run-time performance hazard.

For these reasons, this proposal *bans overlapping conformances* entirely. While the resulting system is less flexible than one that allowed overlapping conformances, the gain in simplicity in this potentially-confusing area is well worth the cost.

There are several potential solutions to the problem of overlapping conformances (e.g., admitting some form of overlapping conformances that can be resolved at runtime or introducing the notion of conformances that cannot be queried a runtime), but the feature is large enough to warrant a separate proposal that explores the solutions in greater depth.

### Extending protocols to conform to protocols
The most common request related to conditional conformances is to allow a (constrained) protocol extension to declare conformance to a protocol. For example:

```swift
extension Collection: Equatable where Iterator.Element: Equatable {
  static func ==(lhs: Self, rhs: Self) -> Bool {
    // ...
  }
}
```

This protocol extension would make any `Collection` of `Equatable` elements `Equatable`, which is a powerful feature that could be put to good use. Introducing conditional conformances for protocol extensions would exacerbate the problem of overlapping conformances, because it would be unreasonable to say that the existence of the above protocol extension means that no type that conforms to `Collection` could declare its own conformance to `Equatable`, conditional or otherwise.

### Overloading across constrained extensions

Conditional conformances may exacerbate existing problems with
overloading behaving differently with concrete types vs. in a generic
context. For example, consider:

```swift
protocol P {
  func f()
}

protocol Q: P { }
protocol R: Q { }

struct X1<T> { }

extension X1: Q where T: Q {
  func f() {
    // #1: basic implementation of 'f()'
  }
}

extension X1: R where T: R {
  func f() {
    // #2: superfast implementation of f() using some knowledge of 'R'
  }
}

// note: compiler implicitly creates conformance `X1: P` equivalent to
//   extension X1: P where T: Q { }

struct X2: R {
  func f() { }
}

(X1<X2>() as P).f() // calls #1, which was used to satisfy the requirement for 'f'
X1<X2>().f()        // calls #2, which is preferred by overload resolution
```

When satisfying a protocol requirement, Swift chooses the most
specific member that can be used *given the constraints of the
conformance*. In this case, the conformance of `X1` to `P` has the
constraints `T: Q`, so the only `f()` that can be used under those
constraints is the `f()` from the first extension. The `f()` in the
second extension won't necessarily always be available, because `T`
may not conform to `R`. Hence, the call that treats an `X1<X2>` as a
`P` gets the first implementation of `X1.f()`. When using the concrete
type `X1<X2>`, where `X2` conforms to `R`, both `X1.f()`
implementations are visible... and the second is more specialized.

This is not a new problem to Swift. We can write a similar example
using a constrained extension and non-conditional conformances:

```swift
protocol P {
  func f()
}

protocol Q: P { }

struct X3<T> { }

extension X3: Q {
  func f() {
    // #1: basic implementation of 'f()'
  }
}

extension X3 where T: R {
  func f() {
    // #2: superfast implementation of f() using some knowledge of 'R'
  }
}

// note: compiler implicitly creates conformance `X3: P` equivalent to
//   extension X3: P { }

struct X2: R {
  func f() { }
}

(X3<X2>() as P).f() // calls #1, which was used to satisfy the requirement for 'f'
X3<X2>().f()        // calls #2, which is preferred by overload resolution
```

That said, the introduction of conditional conformances might increase
the likelihood of these problems surprising developers.
