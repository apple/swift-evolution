# Introduce existential `any`

* Proposal: [SE-0335](0335-existential-any.md)
* Authors: [Holly Borla](https://github.com/hborla)
* Review Manager: [Doug Gregor](https://github.com/DougGregor)
* Status: **Implemented (Swift 7.0)**
* Upcoming Feature Flag: `ExistentialAny` (implemented in Swift 5.8)
* Implementation: [apple/swift#40282](https://github.com/apple/swift/pull/40282)
* Decision Notes: [Acceptance](https://forums.swift.org/t/accepted-with-modifications-se-0335-introduce-existential-any/54504)

## Contents
  - [Introduction](#introduction)
  - [Motivation](#motivation)
  - [Proposed solution](#proposed-solution)
  - [Detailed design](#detailed-design)
    - [Grammar of explicit existential types](#grammar-of-explicit-existential-types)
    - [Semantics of explicit existential types](#semantics-of-explicit-existential-types)
      - [`Any` and `AnyObject`](#any-and-anyobject)
      - [Metatypes](#metatypes)
      - [Type aliases and associated types](#type-aliases-and-associated-types)
  - [Source compatibility](#source-compatibility)
  - [Effect on ABI stability](#effect-on-abi-stability)
  - [Effect on API resilience](#effect-on-api-resilience)
  - [Alternatives considered](#alternatives-considered)
    - [Rename `Any` and `AnyObject`](#rename-any-and-anyobject)
    - [Use `Any<P>` instead of `any P`](#use-anyp-instead-of-any-p)
  - [Future Directions](#future-directions)
    - [Extending existential types](#extending-existential-types)
    - [Re-purposing the plain protocol name](#re-purposing-the-plain-protocol-name)
  - [Revisions](#revisions)
    - [Changes from the pitch discussion](#changes-from-the-pitch-discussion)
  - [Acknowledgments](#acknowledgments)

## Introduction

Existential types in Swift have an extremely lightweight spelling: a plain protocol name in type context means an existential type. Over the years, this has risen to the level of **active harm** by causing confusion, leading programmers down the wrong path that often requires them to re-write code once they hit a fundamental [limitation of value-level abstraction](https://forums.swift.org/t/improving-the-ui-of-generics/22814#heading--limits-of-existentials). This proposal makes the impact of existential types explicit in the language by annotating such types with `any`.

Swift evolution discussion thread: [[Pitch] Introduce existential `any`](https://forums.swift.org/t/pitch-introduce-existential-any/53520).

## Motivation

Existential types in Swift have significant limitations and performance implications. Some of their limitations are missing language features, but many are fundamental to their type-erasing semantics. For example, given a protocol with associated type requirements, the existential type cannot conform to the protocol itself without a manual conformance implementation, because there is not an obvious concrete associated type that works for any value conforming to the protocol, as shown by the following example:

```swift
protocol P {
  associatedtype A
  func test(a: A)
}

func generic<ConcreteP: P>(p: ConcreteP, value: ConcreteP.A) {
  p.test(a: value)
}

func useExistential(p: P) {
  generic(p: p, value: ???) // what type of value would P.A be??
}
```

Existential types are also significantly more expensive than using concrete types. Because they can store any value whose type conforms to the protocol, and the type of value stored can change dynamically, existential types require dynamic memory unless the value is small enough to fit within an inline 3-word buffer. In addition to heap allocation and reference counting, code using existential types incurs pointer indirection and dynamic method dispatch that cannot be optimized away.

Despite these significant and often undesirable implications, existential types have a minimal spelling. Syntactically, the cost of using one is hidden, and the similar spelling to generic constraints has caused many programmers to confuse existential types with generics. In reality, the need for the dynamism they provided is relatively rare compared to the need for generics, but the language makes existential types too easy to reach for, especially by mistake. The cost of using existential types should not be hidden, and programmers should explicitly opt into these semantics.

## Proposed solution

I propose to make existential types syntactically explicit in the language using the `any` keyword. This proposal introduces the new syntax in the Swift 5 language mode, and this syntax should be required for existential types under the Swift 6 language mode.

In Swift 5, anywhere that an existential type can be used today, the `any` keyword can be used to explicitly denote an existential type:

```swift
// Swift 5 mode

protocol P {}
protocol Q {}
struct S: P, Q {}

let p1: P = S() // 'P' in this context is an existential type
let p2: any P = S() // 'any P' is an explicit existential type

let pq1: P & Q = S() // 'P & Q' in this context is an existential type
let pq2: any P & Q = S() // 'any P & Q' is an explicit existential type
```

In Swift 6, existential types are required be explicitly spelled with `any`:

```swift
// Swift 6 mode

protocol P {}
protocol Q {}
struct S: P, Q {}

let p1: P = S() // error
let p2: any P = S() // okay

let pq1: P & Q = S() // error
let pq2: any P & Q = S() // okay
```

The Swift 6 behavior can be enabled in earlier language modes with the [upcoming feature flag](0362-piecemeal-future-features.md) `ExistentialAny`.

## Detailed design

### Grammar of explicit existential types

This proposal adds the following production rules to the grammar of types:

```
type -> existential-type

existential-type -> 'any' type
```

### Semantics of explicit existential types

The semantics of `any` types are the same as existential types today. Explicit `any` can only be applied to protocols and protocol compositions, or metatypes thereof; `any` cannot be applied to nominal types, structural types, type parameters, and protocol metatypes:

```swift
struct S {}

let s: any S = S() // error: 'any' has no effect on concrete type 'S'

func generic<T>(t: T) {
  let x: any T = t // error: 'any' has no effect on type parameter 'T'
}

let f: any ((Int) -> Void) = generic // error: 'any' has no effect on concrete type '(Int) -> Void'
```

#### `Any` and `AnyObject`

`any` is unnecessary for `Any` and `AnyObject` (unless part of a protocol composition):

```swift
struct S {}
class C {}

let value: any Any = S()
let values: [any Any] = []
let object: any AnyObject = C()

protocol P {}
extension C: P {}

let pObject: any AnyObject & P = C() // okay
```

> **Rationale**: `any Any` and `any AnyObject` are redundant. `Any` and `AnyObject` are already special types in the language, and their existence isn’t nearly as harmful as existential types for regular protocols because the type-erasing semantics is already explicit in the name.

#### Metatypes

The existential metatype, i.e. `P.Type`, becomes `any P.Type`. The protocol metatype, i.e. `P.Protocol`, becomes `(any P).Type`. The protocol metatype value `P.self` becomes `(any P).self`:

```swift
protocol P {}
struct S: P {}

let existentialMetatype: any P.Type = S.self

protocol Q {}
extension S: Q {}

let compositionMetatype: any (P & Q).Type = S.self

let protocolMetatype: (any P).Type = (any P).self
```

> **Rationale**: The existential metatype is spelled `any P.Type` because it's an existential type that is a generalization over metatypes. The protocol metatype is the singleton metatype of the existential type `any P` itself, which is naturally spelled `(any P).Type`.

Under this model, the `any` keyword conceptually acts like an existential quantifier `∃ T`. Formally, `any P.Type` means `∃ T:P . T.Type`, i.e. for some concrete type `T` conforming to `P`, this is the metatype of that concrete type.`(any P).Type` is formally `(∃ T:P . T).Type`, i.e. the metatype of the existential type itself.

The distinction between `any P.Type` and `(any P).Type` is syntactically very subtle. However, `(any P).Type` is rarely useful in practice, and it's helpful to explain why, given a generic context where a type parameter `T` is substituted with an existential type, `T.Type` is the singleton protocol metatype.

##### Metatypes for `Any` and `AnyObject`

Like their base types, `Any.Type` and `AnyObject.Type` remain valid existential metatypes; writing `any` on these metatypes in unnecessary. The protocol metatypes for `Any` and `AnyObject` are spelled `(any Any).Type` and `(any AnyObject).Type`, respectively.

#### Type aliases and associated types

Like plain protocol names, a type alias to a protocol `P` can be used as both a generic constraint and an existential type. Because `any` is explicitly an existential type, a type alias to `any P` can only be used as an existential type, it cannot be used as a generic conformance constraint, and `any` does not need to be written at the use-site:

```swift
protocol P {}
typealias AnotherP = P
typealias AnyP = any P

struct S: P {}

let p2: any AnotherP = S()
let p1: AnyP = S()

func generic<T: AnotherP>(value: T) { ... }
func generic<T: AnyP>(value: T) { ... } // error
```

Once the `any` spelling is required under the Swift 6 language mode, a type alias to a plain protocol name is not a valid type witness for an associated type requirement; existential type witnesses must be explicit in the `typealias` with `any`:

```swift
// Swift 6 code

protocol P {}

protocol Requirements {
  associatedtype A
}

struct S1: Requirements {
  typealias A = P // error: associated type requirement cannot be satisfied with a protocol
}

struct S2: Requirements {
  typealias A = any P // okay
}
```

## Source compatibility

Enforcing that existential types use the `any` keyword will require a source change. To ease the migration, I propose to start allowing existential types to be spelled with `any` with the Swift 5.6 compiler, and require existential types to be spelled with `any` under the Swift 6 language mode. The old existential type syntax will continue to be supported under the Swift 5 language mode, and the transition to the new syntax is mechanical, so it can be performed automatically by a migrator.

[SE-0309 Unlock existentials for all protocols](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0309-unlock-existential-types-for-all-protocols.md) enables more code to be written using existential types. To minimize the amount of new code written that will become invalid in Swift 6, I propose requiring `any` immediately for protocols with `Self` and associated type requirements. This introduces an inconsistency for protocols under the Swift 5 language mode, but this inconsistency already exists today (because you cannot use certain protocols as existential types at all), and the syntax difference serves two purposes:

1. It saves programmers time in the long run by preventing them from writing new code that will become invalid later.
2. It communicates the existence of `any` and encourages programmers to start using it for other existential types before adopting Swift 6.

### Transitioning to `any` in Swift 6

The new `any` syntax will be staged in over several major Swift releases. In the release where `any` is introduced, the compiler will not emit any warnings for the lack of `any` on existential types. After `any` is introduced, warnings will be added to guide programmers toward the new syntax. Finally, these warnings can become errors, or [plain protocol names can be repurposed](#re-purposing-the-plain-protocol-name), in Swift 6.

## Effect on ABI stability

None.

## Effect on API resilience

None.

## Alternatives considered

### Rename `Any` and `AnyObject`

Instead of leaving `Any` and `AnyObject` in their existing spelling, an alternative is to spell these types as `any Value` and `any Object`, respectively. Though this is more consistent with the rest of the proposal, this change would have an even bigger source compatibility impact. Given that `Any` and `AnyObject` aren’t as harmful as other existential types, changing the spelling isn’t worth the churn.

### Use `Any<P>` instead of `any P`

A common suggestion is to spell existential types with angle brackets on `Any`, e.g. `Any<Sequence>`. However, an important aspect of the proposed design is that `any` has symmetry with `some`, where both keywords can be applied to protocol constraints. This symmetry is important for helping programmers understand and remember the syntax, and for future extensions of the `some` and `any` syntax. Opaque types and existential types would both greatly benefit from being able to specify constraints on associated types. This could naturally be done in angle brackets, e.g. `some Sequence<Int>` and `any Sequence<Int>`, or `some Sequence<.Element == Int>` and `any Sequence<.Element == Int>`.

Using the same syntax between opaque types and exsitential types also makes it very easy to replace `any` with `some`, and it is indeed the case that many uses of existential types today could be replaced with opaque types instead.

Finally, the `Any<P>` syntax is misleading because it appears that `Any` is a generic type, which is confusing to the mental model for 2 reasons:

1. A generic type is something programmers can implement themselves. In reality, existential types are a built-in language feature that would be _very_ difficult to replicate with regular Swift code.
2. This syntax creates the misconception that the underlying concrete type is a generic argument to `Any` that is preserved statically in the existential type. The `P` in `Any<P>` looks like an implicit type parameter with a conformance requirement, but it's not; the underlying type conforming to `P` is erased at compile-time.

## Future Directions

### Extending existential types

This proposal provides an obvious syntax for extending existential types in order to manually implement protocol conformances:

```swift
extension any Equatable: Equatable { ... }
```

### Re-purposing the plain protocol name

In other places in the language, a plain protocol name is already sugar for a type parameter conforming to the protocol. Consider a normal protocol extension:

```swift
extension Collection { ... }
```

This extension is a form of universal quantification; it extends all types that conform to `Collection`. This extension introduces a generic context with a type parameter `<Self: Collection>`, which means the above syntax is effectively sugar for a parameterized extension:

```swift
extension <Self> Self where Self: Collection { ... }
```

Changing the syntax of existential types creates an opportunity to expand upon this sugar. If existential types are spelled explicitly with `any`, a plain protocol name could always mean sugar for a type parameter on the enclosing context with a conformance requirement to the protocol. For example, consider the declaration of `append(contentsOf:)` from the standard library:

```swift
extension Array {
  mutating func append<S: Sequence>(contentsOf newElements: S) where S.Element == Element
}
```

Combined with a syntax for constraining associated types in angle brackets, such as in [[Pitch] Light-weight same-type constraint syntax](https://forums.swift.org/t/pitch-light-weight-same-type-constraint-syntax/52889), the above declaration could be simplified to:

```swift
extension Array {
  mutating func append(contentsOf newElements: Sequence<Element>)
}
```

This sugar eliminates a lot of noise in cases where a type parameter is only referred to once in a generic signature, and it enforces a natural model of abstraction, where programmers only need to name an entity when they need to refer to it multiple times.

## Revisions

### Changes from the pitch discussion

* Spell the existential metatype as `any P.Type`, and the protocol metatype as `(any P).Type`.
* Preserve `any` through type aliases.
* Allow `any` on `Any` and `AnyObject`.

## Acknowledgments

Thank you to Joe Groff, who originally suggested this direction and syntax in [Improving the UI of generics](https://forums.swift.org/t/improving-the-ui-of-generics/22814), and to those who advocated for this change in the recent discussion about [easing the learning curve for generics](https://forums.swift.org/t/discussion-easing-the-learning-curve-for-introducing-generic-parameters/52891). Thank you to John McCall and Slava Pestov, who helped me figure out the implementation model.
