# Tuple conformances

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Slava Pestov](https://github.com/slavapestov)
* Review Manager: TBD
* Status: **Implementation in progress**
* Upcoming Feature Flag: `TupleConformances`
* Previous Proposal: [SE-0283 Tuples Conform to Equatable, Comparable, and Hashable](0283-tuples-are-equatable-comparable-hashable.md)
* Review: ([pitch](https://forums.swift.org/...))

## Introduction

Tuples cannot conform to protocols today, and this surfaces in the form of obvious limitations, such as not being able to use a tuple of `Hashable` values as a `Dictionary` key.

## Motivation

The desire for tuples to conform to certain standard library protocols was previously addressed by the **Motivation** of [SE-0283](0283-tuples-are-equatable-comparable-hashable.md), which proposed built-in language support for `Equatable`, `Comparable` and `Hashable` tuples. Independently, the Swift concurrency effort added a language extension where a tuple of `Sendable` values is itself `Sendable`. We propose unifying all of these special-case behaviors with _user-defined tuple conformances_, which can now be expressed using parameter packs ([SE-0393](0393-parameter-packs.md)). Both SE-0283 and SE-0393 list tuple conformances as a **Future direction**.

## Proposed solution

We propose introducing the _parameterized extension_ syntax, described in the [Generics Manifesto](https://github.com/apple/swift/blob/main/docs/GenericsManifesto.md). This syntax will be allowed in one specific case to declare a tuple conformance in its most general form:

```swift
extension <each T> (repeat each T): P where repeat each T: P { ... }
```

We will also allow a generic type alias describing a tuple to be extended with a conditional conformance; we propose that the below `Tuple` type alias is added to the standard library to facilitate this:

```swift
protocol Shape {
  func draw()
}

typealias Tuple<each Element> = (repeat each Element)

extension Tuple: Shape where repeat each Element: Shape {
  func draw() {
    repeat (each self).draw()
  }
}
```

Recall that a _requirement_ in a protocol is implemented by a _witness_ in the concrete conforming type. In the above, we declare a tuple extension, and so the witness for `draw()` implements the protocol requirement `draw()` on tuples. The actual implementation calls `draw()` on each element, which is known to itself conform to `Shape`. Notice the use of `each self` within the `repeat` pattern in the body of `draw()`.

## Detailed design

Any unlabeled tuple can be obtained via type substitution from the "most general" unlabeled tuple type. If `each T` is some type parameter pack, this most general type is `(repeat each T)`; that is, the tuple type formed from a pack expansion over the elements of `each T`.

Today, the extended type of an extension must be a nominal type, be it a struct, enum, class, or protocol. We propose allowing the most general tuple type to be extended; this is called a _tuple extension_. As extensions can declare protocol conformances, a tuple extension can then implement the protocol requirements for the most general tuple type. This is called a _tuple conformance_.

This means the type of `self` within a tuple extension is `(repeat each T)`, where `each T` is a generic parameter of the extension declaring the conformance. As a consequence of [SE-0399](0399-tuple-of-value-pack-expansion.md), a reference to `each self` within a pack expansion expression will expand over the elements of the tuple.

As with extensions of structs, enums, and classes, `Self` within a tuple extension refers to the type of `self`, that is, `(repeat each T)`.

Once a tuple conformance to some protocol `P` has been declared, an arbitrary tuple type will satisfy a conformance requirement to `P` as long as the elements of the tuple satisfy the conditional requirements of the tuple conformance. We will see below that the conditional requirements must consist of exactly one requirement `repeat each T: P`. Therefore, the empty tuple `()` will conform to every protocol that has a tuple conformance.

An invocation of a protocol requirement with a value of tuple type forms a pack from elements of the tuple value. The types of these elements become the type pack for the generic argument to `each T`.

### Orphan rule

For the most part, tuple conformances behave as if they were user-defined retroactive conformances on a standard library type. In particular, it would not be valid for two modules to define two distinct tuple conformances to the same protocol. For this reason, we prohibit tuple conformances to protocols outside of the defining module.

### One-element tuple unwrapping

Under the rules laid out in the parameter packs proposal, one-element tuple types are unwrapped after substitution. This means that tuple conformances must be _coherent_ with respect to this unwrapping.

This imposes some restrictions on the form that tuple conformances can take. We can understand all of the below restrictions in the form of a commutative diagram. The top row shows the most general tuple type, the corresponding tuple conformance, and the witness for some associated type `A`. Now, we apply a substitution to each object, replacing the type parameter pack `each T` with a pack containing a single concrete type, say `X`. We require that all paths through the diagram that begin and end at the same object produce the same result:

```
(repeat each T) ---> [(repeat each T): P] ---> (repeat each T).A
      |                        |                        |
      |                        |                        |
      v                        v                        v
      X -------------------> [X: P] -----------------> X.A
```

Concretely, these restrictions are as follows:

- A tuple extension must declare conformance to exactly one protocol.
- The conditional requirements of this conformance must be exactly `repeat each T: P` where `each T` is the type parameter pack of the extension, and `P` is the conformed protocol.

  That is, a tuple extension `extension Tuple: P where repeat each T: Q` would not make sense, because in the one-element case this decays to `X: P where X: Q`; a statement that is false in the general case when `P` and `Q` may be unrelated protocols.
- An associated type requirement `A` of `P` must be witnessed by a type alias whose underlying type is exactly `(repeat (each T).A)`; that is, a tuple type projecting `A` from each element.

  That is, if `X.A` is `Int`, and `Y.A` is `String`, then we have no choice but to require that `(X, Y).A` is equal to `(Int, String)`.

Note that as a consequence of all of these rules, the empty tuple `()` will conform to every protocol that has a tuple conformance.

### Dynamic behavior

The above rules allow us to guarantee that a tuple conformance witness is never invoked with a one-element pack, with the call forwarding directly to the element conformance in that case. Thus, the runtime type of `Self` in a tuple conformance must always be a bona-fide tuple type, and not an unwrapped element.

If some function itself uses parameter packs to form a tuple value from a pack, calling protocol requirements on this value will either invoke the tuple conformance witness or witness for a single element, depending on the size of the pack.

### Labeled tuples and variance

Tuple labels are not something that parameter packs can abstract over. However, the expression type system defines a subtype relationship between a labeled tuple and the corresponding unlabeled tuple.

By analogy with classes, if a conformance is declared on a non-final class `C`, and there is a subclass relationship where `D` inherits from `C`, then the conformance is also inherited by `D`.

For the substitution of `D` for `C` to be valid in the case of class inheritance, we require that `Self` is only used in covariant or contravariant position, not invariant. We must therefore impose the same restrictions on tuples as we currently have on non-final classes.

This allows the following:

- Conforming to a protocol like `Equatable`, where `Self` appears in parameter position.
- Conforming to a hypothetical `Clonable` protocol, with a `func clone() -> Self` requirement that returns `Self`.

On the other hand, this is prohibited:

- Conforming to a protocol with a requirement having `Self` in invariant position, such as `func f() -> G<Self>`.

  In this case, it would not be fully sound to take a labeled tuple, and apply `G<>` to the corresponding unlabeled tuple type.

### Scope of usage

Due to the subtle static and dynamic behaviors outlined above, we expect tuple conformances to remain an advanced feature. For many purposes, it is better to declare a special-purpose variadic generic struct via [SE-0398](0398-variadic-types.md), and conform that to a protocol, because this offers complete flexibility without any complications around coherence:

```
struct EggFactory<each Bird> {}

extension EggFactory: OmletMaker where repeat each Bird: Chicken {}
```

This pattern also allows the variadic type to define custom constructors and accessors to enforce invariants, and so on.

Tuples should only conform to protocols that have obvious "algebraic" implementation that generalizes to all combinations of element types in an inductive manner, such as the three standard library protocols discussed above.

For example, it would probably not be a good idea to conform tuples to `IteratorProtocol`, because there are at least two obvious implementations; either a `zip`, or a concatenation (in which case we would also need a requirement that all sequences have the same element type, something a tuple conformance cannot even express).

## Source compatibility

As part of this proposal, we will implement tuple conformances for `Equatable`, `Comparable` and `Hashable` in the standard library. These will also replace the static overloads of the comparison operators and `==` on tuples. These changes should remain source compatible with existing code.

## ABI compatibility

**This is tentative and subject to change. None of this is a promise or commitment.**

We hope that it will be possible to use both user-defined tuple conformances, and the tuple conformances to `Equatable`, `Comparable` and `Hashable` on older runtimes.

A tuple conformance to a protocol with associated types will probably require at least a Swift 5.9 runtime.

Dynamic casts, for example `((1, 2) as Any) as? any Equatable` will possibly requires a new Swift runtime with support for tuple conformances.

## Acknowledgments

I would like to thank Alejandro Alonso for the previous design and implementation of [SE-0283 Tuples Conform to Equatable, Comparable, and Hashable](0283-tuples-are-equatable-comparable-hashable.md).