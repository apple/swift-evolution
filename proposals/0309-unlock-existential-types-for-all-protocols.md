# Unlock existentials for all protocols

* Proposal: [SE-0309](0309-unlock-existential-types-for-all-protocols.md)
* Authors: [Anthony Latsis](https://github.com/AnthonyLatsis), [Filip Sakel](https://github.com/filip-sakel), [Suyash Srijan](https://github.com/theblixguy)
* Review Manager: [Joe Groff](https://github.com/jckarter)
* Status: **Implemented (Swift 5.7)**
* Implementation: [apple/swift#33767](https://github.com/apple/swift/pull/33767), [apple/swift#39492](https://github.com/apple/swift/pull/39492), [apple/swift#41198](https://github.com/apple/swift/pull/41198)
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-se-0309-unlock-existentials-for-all-protocols/47902), [Additional Commentary](https://forums.swift.org/t/se-0309-unlock-existential-types-for-all-protocols/47515/123)

## Introduction

Swift allows one to use a protocol as a type when its *requirements* meet a rather unintuitive list of criteria, among which is the absence of associated type requirements, and emits the following error otherwise: `Protocol can only be used as a generic constraint because it has 'Self' or associated type requirements`. Our objective is to *alleviate* this limitation so as to impact only the ability to access certain members (instead of preemptively sealing off the entire protocol interface), and adjust the specified criteria to further reduce the scope of the restriction.

This proposal is a preamble to a series of changes aimed at generalizing value-level abstraction (existentials) and improving its interaction with type-level abstraction (generics). For an in-depth exploration of the relationships among different built-in abstractions models, we recommend reading the [design document for improving the UI of the generics model](https://forums.swift.org/t/improving-the-ui-of-generics/22814).

Swift-Evolution Pitch Threads: [Thread #1](https://forums.swift.org/t/lifting-the-self-or-associated-type-constraint-on-existentials/18025), [Thread #2](https://forums.swift.org/t/unlock-existential-types-for-all-protocols/40665)

## Motivation

When a protocol is used as a type, that type is also known as an *existential type*. Unlike values of `some` types, which represent a value of some *specific* type that conforms to the given constraints, and cannot be reassigned to a value of a different conforming type, an existential value is akin to a box that can hold any value of any conforming type dynamically at any point in time. Existential types allow values of varying concrete types to be used interchangeably as values of the same existential type, abstracting the difference between the underlying conforming types at the *value level*. For convenience, we will be using the term "existential" to refer to such values and their protocol or protocol composition types throughout the proposal. We also wish to draw a distinction between an associated type *requirement* (the declaration), and an associated type (aka a dependent member type). For example, `Self.Element` and `Self.SubSequence.Element` are distinct associated types that point to the same associated type requirement.

### Inconsistent Language Semantics

The compiler permits the use of a protocol as a type *unless*
1) the protocol has an associated type requirement, or
2) the type of a method/property/subscript/initializer requirement contains a reference to `Self` in [non-covariant](https://en.wikipedia.org/wiki/Covariance_and_contravariance_(computer_science)) position:

```swift
// 'Identifiable' has an associated type requirement.
public protocol Identifiable {
  associatedtype ID: Hashable

  var id: ID { get }
}

// 'Equatable' has a operator method requirement containing a `Self` reference in contravariant parameter position.
public protocol Equatable {
  static func == (lhs: Self, rhs: Self) -> Bool
}
```
The first condition is a relic of an incomplete implementation of protocol witness tables that didn't allow associated type metadata to be recovered dynamically. Despite having the same restrictive impact when violated, the second condition is actually meant to govern the type safety of individual member accesses. Consider the following protocol interface:

```swift
protocol P {
  func foo() -> Self
  
  func bar(_: Self)
}
```

Accessing a member on an existential value requires the ability to spell the type of that member outside its protocol context. Today, the one means to representing the dynamic `Self` type of an existential value is type erasure — the substitution of `Self` with a representable supertype, like `P`. On the other hand, type erasure is safe to perform only in [covariant](https://en.wikipedia.org/wiki/Covariance_and_contravariance_(computer_science)) position. For example, calling `foo` on a value of type `P` with its covariant `Self` result type-erased to `P` is safe, whereas allowing one to pass a type-erased value to `bar` would expose the opportunity to pass in an argument of non-matching type.

In contrast to requirements, protocol extension members cannot afford to retroactively jeopardize the existential availability of the entire protocol. This is when the second condition shows its true colors, with the restriction forced to take on a more reasonable, on-demand manifestation in the form of a member access failure:

```swift 
protocol P {}
extension P {
  func method(_: Self) {}
}

func callMethod(p: P) {
  p.method // error: member 'method' cannot be used on value of protocol type 'P'; use a generic constraint instead
}
```

As we hope it became clear, a requirement such as `func bar(_: Self)` or an associated type requirement alone cannot speak for the rest of the interface. Some protocols still have a useful subset of functionality that does not rely on `Self` and associated types whatsoever, or does so in a way that is compatible with existential values, like `func foo() -> Self`. In a refined protocol, some requirements may also happen to exclusively rely on an associated type with a fully concrete, known implementation, and become safe to invoke using a value of the refined protocol type. This brings us to a well-known implementation hole with counterintuitive behavior; in the snippet below, `Animal` is still assumed as having an associated type requirement despite the same-type constraint that effectively predefines a fully concrete implementation for it.

```swift
protocol Animal: Identifiable where ID == String {}
extension Animal {
  var name: String { id }
}
```

The current semantic inconsistency also discourages authors from refining their existing protocols with other, useful ones in fear of losing existential qualification.

### Library Evolution

Removing the type-level restriction would mean that adding defaulted requirements to a protocol is always both a binary- and source-compatible change, since it could no longer interfere with existing uses of the protocol.

### Type-Erasing Wrappers

Beyond making incremental progress toward the goal of [generalized existentials](https://github.com/apple/swift/blob/main/docs/GenericsManifesto.md#generalized-existentials), removing this restriction is a necessary — albeit not sufficient — condition for eliminating the need for manual type-erasing wrappers like [`AnySequence`](https://developer.apple.com/documentation/swift/anysequence). These containers are not always straightforward to implement, and can become a pain to mantain in resilient environments, since the wrapper must evolve in parallel to the protocol. In the meantime, wrapping the unconstrained existential type instead of resorting to `Any` or boxing the value in a subclass or closure will enable type-erasing containers to be written in a way that's easier for the compiler to optimize, and ABI-compatible with future generalized existentials. For requirements that cannot be accessed on the existential directly, it will be possible to forward the call through the convolution of writing protocol extension methods to open the value inside and have full access to the protocol interface inside the protocol extension:

```swift
protocol Foo {
  associatedtype Bar

  func foo(_: Bar) -> Bar
}

private extension Foo {
  // Forward to the foo method in an existential-accessible way, asserting that
  // the '_Bar' generic argument matches the actual 'Bar' associated type of the
  // dynamic value.
  func _fooThunk<_Bar>(_ bar: _Bar) -> _Bar {
    assert(_Bar.self == Bar.self)
    let result = foo(unsafeBitCast(bar, to: Bar.self))
    return unsafeBitCast(result, to: _Bar.self)
  }
}

struct AnyFoo<Bar>: Foo {
  private var _value: Foo

  init<F: Foo>(_ value: F) where F.Bar == Bar {
    self._value = value
  }
  
  func foo(_ bar: Bar) -> Bar {
    return self._value._fooThunk(bar)
  }
}
```

## Proposed Solution

We suggest allowing any protocol to be used as a type and exercise the restriction on individual member accesses uniformly across extension members and requirements. Additionally, the adjusted access criteria for protocol members shall account for associated types with known implementations:

```swift
protocol IntCollection: RangeReplaceableCollection where Self.Element == Int {}
extension Array : IntCollection where Element == Int {}

var array: any IntCollection = [3, 1, 4, 1, 5]

array.append(9) // OK, 'Self.Element' is known to be 'Int'.
```

Having lowered the limitation, the mere presence of an associated type requirement will no longer preclude member accesses, but references to `Self`-rooted associated types *will* for the same reasons some `Self` references do today. As alluded to back in [Inconsistent Language Semantics](#inconsistent-language-semantics), references to covariant `Self` are already getting automatically replaced with the base object type, permitting usage of `Self`-returning methods on existential values:

```swift
protocol Copyable {
  func copy() -> Self
}

func test(_ c: Copyable) {
  let x = c.copy() // OK, x is of type 'Copyable'
}
```

Because they tend to outnumber direct uses of `Self` in protocol contexts, and for the sake of consistency, we believe that extending covariant type erasure to associated types is a reasonable undertaking in light of the primary focus:

```swift
func test(_ collection: RandomAccessCollection) {
  // func dropLast(_ k: Int = 1) -> SubSequence
  let x = collection.dropLast() // OK, x is of type 'RandomAccessCollection'
}
```
___

This way, a protocol or protocol extension member (method/property/subscript/initializer) may be used on an existential value *unless*:
* The type of the invoked member (accessor — for storage declarations), as viewed in context of the *base type*, contains references to `Self` or `Self`-rooted associated types in [non-covariant](https://en.wikipedia.org/wiki/Covariance_and_contravariance_(computer_science)) position.

> The following types will be considered covariant:
> * Function types in their result type.
> * Tuple types in either of their element types.
> * [Swift.Optional](https://developer.apple.com/documentation/swift/optional) in its `Wrapped` type.
> * [Swift.Array](https://developer.apple.com/documentation/swift/array) in its `Element` type.
> * [Swift.Dictionary](https://developer.apple.com/documentation/swift/dictionary) in its `Value` type.

## Detailed Design

Once more, we note that not all requirements will be accessible on existential values. For instance, the `==` operation still cannot be used on two values of the [`Equatable`](https://developer.apple.com/documentation/swift/equatable) type, because it cannot be proved that their dynamic types match without additional dynamic checks:

```swift
let lhs: Equatable = "Paul"
let rhs: Equatable = "Alex"

lhs == rhs ❌

if let ownerName = lhs as? String, let petName = rhs as? String {
  print(ownerName == petName) ✅ // false
}
```

### Diagnostics

Invoking an incompatible member on an existential value will trigger an error comprising a terse description of the issue and a suggestion to use the generic approach (if applicable) in order to gain full access to the protocol interface. For the common case when the existential base is a reference to a function or subscript parameter, the diagnostic will include a fix-it that turns it into a generic parameter (again, if applicable, since generic functions are not allowed in some local contexts).

```swift
extension Sequence {
  public func enumerated() -> EnumeratedSequence<Self> {
    return EnumeratedSequence(_base: self)
  }
}

func printEnumerated(s: Sequence) {
  // error: member 'enumerated' cannot be used on value of type protocol type 'Sequence'
  // because it references 'Self' in invariant position; use a conformance constraint
  // instead. [fix-it: printEnumerated(s: Sequence) -> printEnumerated<S: Sequence>(s: S)]
  for (index, element) in s.enumerated() {
    print("\(index) : \(element)")
  }
}

let collection: RangeReplaceableCollection = [1, 2, 3]
// error: member 'append' cannot be used on value of protocol type 'RangeReplaceableCollection'
// because it references associated type 'Element' in contravariant position; use a conformance
// constraint instead.
collection.append(4)
```

In an ideal world, one could imagine the compiler to accompany the error with a note pointing to the specific type reference that is preventing the member from being used. We are inclined toward leaving this out of scope for several reasons:
* Retrieval of the *relevant* source location information both logically and mechanically poses an earnest challenge with the current workings of various compiler components and the potential involvement of generic constraints.
* There is no certainty in whether a concept of these high-precision notes can outplay an educational note in the general case, or is worth indefinitely dragging out a resolution to this particular proposal.

To showcase just one embodiment of the difficulties involved, consider this relatively simple code:
```swift
struct G<T> {}

protocol P {
  associatedtype A
  associatedtype B
  
  func method() -> B
}

protocol Q: P where B == G<A> {}
```
Notice how the associated type that would preclude a call to `method` on a value of type `Q` is actually `A`, not `B` as the result type may suggest, due to the same-type constraint on the protocol.

#### Non-conformable Existentials

A peculiar side effect of lowering the limitation is the expansion of the domain of existential types that cannot be conformed to. Some are such for fundamental reasons, and others could be made conformable with the adoption of appropriate features. One example of the latter is a composition between two unrelated protocols, each constraining the same associated type to different concrete types:
```swift
protocol P1 {
  associatedtype A
} 
protocol P2: P1 where A == Int {}

protocol Q1 {
  associatedtype A
}
protocol Q2: Q1 where A == Bool {}

func foo(_: P2 & Q2) {

}
```
Any code relying on a non-conformable type is effectively dead and type-safe to keep around. Likewise, Swift may provide a way give formally distinct requirements, like `P1.A` and `Q1.A`, distinct implementations in the future. To spare us from having to deal with a source compatibility dilemma, we propose to at best warn about the ambiguities that arise in these types with messages we already use for generic parameters in similar circumstances.

### Associated Types with Known Implementations

By a known implementation, we mean that an associated type is bound to a concrete type under the generic signature of a given existential type. A known implementation has two embodiments: an explicit same-type constraint, i.e. `A == Int`, or an actual implementation for the associated type requirement, found via some other constraint, like a superclass requirement:

```swift
class Class: P {
  typealias A = Int
}

protocol P {
  associatedtype A
}
protocol Q: P {
  func takesA(arg: A)
}

func testComposition(arg: Q & Class) {
  arg.takesA(arg: 0) // OK, 'A' is known to be 'Int'.
}
```
A reference to a `Self`-rooted associated type with a known implementation will not prevent one from accessing a member.

### Covariant Erasure for Associated Types

When invoking a member, `Self`-rooted associated types that
* do **not** have a known implementation and
* appear in covariant position within the type of the member

will be type-erased to their upper bounds as per the generic signature of the existential that is used to access the member. The upper bounds can be either a class, protocol, protocol composition, or `Any`, depending on the *presence* and *kind* of generic constraints on the associated type. As such, references to these associated types are also acceptable for accessing members on existential values. The essence of this behavior was presented a tad [earlier](#proposed-solution) alongside the proposed solution.  

## Source Compatibility & Effect on ABI Stability

The proposed changes are ABI-additive and source-compatible.

## Effect on API Resilience

Adding defaulted requirements to a protocol will become an always-source-compatible change.

## Alternatives Considered

The concerns that were raised in regards to our stance can be attributed to one of the following observations:
* The fraction of available API is implicitly determined and requires careful inspection to be reasoned about.
* The current syntax for existentials is dangerously lightweight and strongly implies an API surface that may not match reality.

Both statements are referring to *existing* problems — due to protocol extensions and the way we spell existential types — that have not been addressed and would become more widespread unless something else is done prior or in addition to implementing the proposed solution. 

### Let the User Decide on Availability

Since the portion of available API is implicit, it is not apparent anywhere in code what API are being vended or claimed when using an existential, which could lead to developers proliferating the use of type-erasing wrapper dummies to avoid coding themselves into member unavailability time bombs (although, this technique would help only against unsolicited requirements, and not extension members). A solution suggests introducing
* a protocol-specific opt-in declaration modifier as a means of promising that the all *requirements* will always be available for use on the existential, and
* a second opt-in declaration modifier to explicitly mark protocol members that are intended as available on the existential.

The first modifier is to statically prevent the addition of requirements that are incompatible with the existential, and the second is to forestall accidental unavailability and enhance discoverability.

In our opinion, the pitfall of unexpected unavailability has to do mostly with inappropriate application of value-level abstraction, and is best addressed by reviewing the language guide and following the somewhat *established* roadmap for [generalized existentials](https://github.com/apple/swift/blob/main/docs/GenericsManifesto.md#generalized-existentials) (which includes syntax renovation and explicit opening of existential values), rather than taking a less principled detour. In swift-evolution discussion, the community pointed out several notable flaws:
* Using these modifiers feels like completely losing sight of generic programming, where no such usability limitations exist.
* The ability to access a member does not so much depend on its declared type as on the one of a multiple of existential types that is used to access it, and the invoked accessor (for storage declarations).
* This approach seems likely to lead to trade-offs between optimal design and compliance with the modifier.
* Being a source-compatible addition, modifiers can merely offer the *option* to be explicit.

### Syntactic Matters First

So far, existentials are the only built-in abstraction model (on par with generics and opaque types) that doesn't have its own idiosyncratic syntax; they are spelled as the bare protocol name or a composition thereof. Although the syntax strongly suggets that the protocol as a type and the protocol as a constraint are one thing, in practice, they serve different purposes, and this manifests most confusingly in the *"Protocol (the type) does not conform to Protocol (the constraint)"* paradox. This could be qualified as a missing feature in the language; the bottom line is that the syntax is tempting developers when it should be contributing to weighted decisions. Because using existential types is syntactically lightweight in comparison to using other abstractions, and similar to using a base class, the more possibilities they offer, the more *users are* vulnerable to unintended or inappropriate type erasure by following the path of initial least resistance.

With regard to a source-compatible adoption of new adornments, we could
* force all newly supported existential types to use the new syntax, i.e. `Any<P>` or [`any P` as a dual to `some P`](https://forums.swift.org/t/improving-the-ui-of-generics/22814#heading--clarifying-existentials), and
* allow the respective protocols of all newly supported existential types to be annotated with an attribute to opt into the old syntax (to allow for source-compatible addition of requirements).

Though, if we *were* going to introduce a new syntax for existentials, we think it'd be much less confusing if we took the potentially source-breaking path and did so uniformly, deprecating the existing syntax after a late-enough language version, than to have yet another attribute and two syntaxes where one only works some of the time. We also believe that drawing a tangible line between protocols that "do" and "do not" have limited access to their API is ill-advised due to the relative nature of this phenomenon.

## Future Directions

* Simplify the implementation of Standard Library type-erasing wrappers, such as [`AnyHashable`](https://github.com/apple/swift/blob/main/stdlib/public/core/AnyHashable.swift) and [`AnyCollection`](https://github.com/apple/swift/blob/main/stdlib/public/core/ExistentialCollection.swift), using the practical advice from [earlier](#type-erasing-wrappers).
* Deemphasize existential types.

  It is often that people reach for existential types when they should be employing generic contraints — "should" not merely for performance reasons, but because they truly do not need or intend for any type erasure. Even though the compiler is sometimes able to back us up performance-wise by turning existential code into generic code (as in  `func foo(s: Sequence)` vs `func foo<S: Sequence>(s: S)`), there is an important difference between the two abstractions. Existential types provide value-level abstraction, that is, they eliminate the type-level distinction between different values of the type, and cannot maintain type relationships between independent existential values. Under most cirumstances, value-level abstraction only really makes sense in mutable state, in the elements of heterogeneous containers, or, unless our support of [`some` types](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0244-opaque-result-types.md) can turn the tide, in the storage of larger type-erasing constructs. A fitting starting point for giving value-level abstraction more careful consideration early on is the [Language Guide](https://docs.swift.org/swift-book/LanguageGuide/TheBasics.html).
* Make existential types "self-conforming" by automatically opening them when passed as generic arguments to functions. Generic instantiations could have them opened as opaque types.
* Add an `init(_ box: Hashable)` initializer to `AnyHashable` to alleviate confusion and aid in usability. This initializer would be treated as a workaround, and deprecated should automatic opening of existential types become available.
* Introduce [`any P` as a dual to `some P`](https://forums.swift.org/t/improving-the-ui-of-generics/22814#heading--clarifying-existentials) for explicitly spelling existential types.
* Allow constraining existential types, i.e. `let collection: any Collection<Self.Element == Int> = [1, 2, 3]`.
