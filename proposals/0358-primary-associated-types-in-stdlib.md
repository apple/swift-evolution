# Primary Associated Types in the Standard Library

* Proposal: [SE-0358](0358-primary-associated-types-in-stdlib.md)
* Authors: [Karoy Lorentey](https://github.com/lorentey)
* Review Manager: [John McCall](https://github.com/rjmccall)
* Status: **Implemented (Swift 5.7)**
* Implementation: [apple/swift#41843](https://github.com/apple/swift/pull/41843)
* Review: ([pitch](https://forums.swift.org/t/pitch-primary-associated-types-in-the-standard-library/56426/)) ([review](https://forums.swift.org/t/se-0358-primary-associated-types-in-the-standard-library/57432)) ([partial acceptance](https://forums.swift.org/t/se-0358-primary-associated-types-in-the-standard-library/57432/14)) ([revision and extension](https://forums.swift.org/t/se-0358-primary-associated-types-in-the-standard-library/57432/32)) ([acceptance](https://forums.swift.org/t/accepted-se-0358-primary-associated-types-in-the-standard-library/58547))
* Related Proposals:
   - [SE-0023] API Design Guidelines
   - [SE-0346] Lightweight same-type requirements for primary associated types
   
[SE-0023]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0023-api-guidelines.md
[SE-0346]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0346-light-weight-same-type-syntax.md

## Introduction

[SE-0346] introduced the concept of primary associated types to the language. This document proposes to adopt this feature in the Swift Standard Library, adding primary associated types to select existing protocols. Additionally, we provide some general API design recommendations that protocol authors may find helpful when adding support for this language feature.

## Motivation

In order for the lightweight constraint syntax introduced in [SE-0346] to be actually usable, protocol definitions inside and outside the Standard Library need to be extended with primary associated type declarations.

See [SE-0346] for several motivating examples for these changes.

## API Design Guidelines

Primary associated types add a new facet to the design of protocols. For every public protocol with associated type requirements, we need to carefully consider which of them (if any) we want to mark as primary. On the one hand, we want to allow people to use the shorthand syntax whenever possible; on the other hand, we only get one chance to decide this: once a protocol gains a primary associated type annotation, most subsequent changes would be source-breaking.

We've found the following guidelines helpful when considering the adoption of primary associated types within the Standard Library. We haven't had enough real-life experience with this new feature to propose these guidelines for general use -- however, the recommendations below can still serve as a useful starting point.

(Aside: If you decide to follow these guidelines when annotating your own protocols, and they lead you to a choice that you later regret, please post a note on the Swift forums! Negative examples are going to be extremely helpful while revising the guidelines for general use. We're also looking for (positive or negative) examples for multiple primary associated types on a single protocol.)

1. **Let usage inform your design.**

   If you are considering adding a primary associated type declaration to a preexisting protocol, then look at its existing clients to discover which associated types get typically constrained. Is there one particular type that is used overwhelmingly more than any other? If so, then it will probably be a good choice for the primary.

   For example, in the case of `Sequence`, use sites overwhelmingly tend to constrain `Element` -- `Iterator` is almost never mentioned in `where` clauses. This makes it fairly clear that `Element` is the right choice for the primary type.

   If you're designing a new protocol, think about which type people will most likely want to constrain. Sometimes it may not even be one you planned to have as an associated type!

   For example, protocol `Clock` in [SE-0329](0329-clock-instant-duration.md) initially only had `Instant` as an associated type. As it turns out, in actual use cases, people are far more likely to want to constrain `Instant.Duration` rather than `Instant` itself. Clocks tend to be far too closely coupled to their instants for it to serve as a useful constraint target -- `some Clock<ContinuousClock.Instant>` is effectively just a circuitous way of spelling `ContinuousClock`. On the other hand, `some Clock<Swift.Duration>` captures all clocks that measure elapsed time in physical seconds -- a far more useful abstraction. Therefore, we decided to add `Clock.Duration` for the express purpose to serve as the primary associated type.

2. **Consider clarity at the point of use.** To prevent persistent confusion, _people familiar with the protocol_ ought to be able to correctly intuit the meaning of a same-type constraint such as `some Sequence<Int>`.

   Lightweight constraint specifications share the same angle-bracketed syntax as generic type arguments, including the same limitations. In particular, the language does not support argument labels in such lists, which prevents us from clarifying the role of the type names provided. A type name such as `Foo<Int, String>` on its own provides no hints about the role of its generic arguments `Int` and `String`; likewise, it isn't possible to decipher the role of `Character` in a same-type requirement such as `some Bar<Character>`, unless the reader is already somewhat familiar with the protocol `Bar`.

   The best candidates for primary associated types tend to be those that have a simple, obvious relationship to the protocol itself. A good heuristic is that if the relationship can be described using a simple preposition, then the associated type will probably make a viable primary:

   - `Collection` *of* `Int`
   - `Identifiable` *by* `String`
   - `SIMD` *of* `Float`
   - `RawRepresentable` *by* `Int32`

   Associated types that don't support this tend to have a more complex / idiosyncratic role in their protocol, and often make poor choices for a primary associated type.

   For example, `Numeric` has an associated type called `Magnitude` that does sometimes appear in associated type constraints. However, its role seems too subtle and non-obvious to consider marking it as primary. The meaning of `Int` in `some Numeric<Int>` is unlikely to be clear to readers, even if they are deeply familiar with Swift's numeric protocol hierarchy.

3. **Not every protocol needs primary associated types.** Don't feel obligated to add a primary associated type just because it is possible to do so. If you don't expect people will want to constrain an associated type in practice, there is little reason to mark it as a primary. Similarly, if there are multiple possible choices that seem equally useful, it might be best not to select one. (See point 2 above.)

   For example, `ExpressibleByIntegerLiteral` is not expected to be mentioned in generic function declarations, so there is no reason to mark its sole associated type (`IntegerLiteral`) as the primary.

4. **Limit yourself to just one primary associated type.** In most cases, it's best not to declare more than one primary associated type on any protocol.

   While the language does allow this, [SE-0346] requires clients using the lightweight syntax to always explicitly constrain all primary associated types, which may become an obstacle. Clients don't have an easy way to indicate that they want to leave one of the types unconstrained -- to do that, they need to revert to classic generic syntax, partially or entirely giving up on the lightweight variant:

   ```swift
   protocol MyDictionaryProtocol<Key, Value> {
     associatedtype Key: Equatable
     associatedtype Value
     ...
   }

   // This function is happy to work on any dictionary-like thing
   // as long as it has string keys.
   func twiddle(_ items: some MyDictionaryProtocol<String, ???>) -> Int { ... }

   // Possible approaches:
   func twiddle<Value>(_ items: some MyDictionaryProtocol<String, Value>) -> Int { ... }
   func twiddle<T: MyDictionaryProtocol>(_ items: T) -> Int where T.Key == String { ... }
   ```
    
   Of course, if the majority of clients actually do want to constrain both `Key` and `Value`, then having them both marked primary can be an appropriate choice.


## Proposed solution

The table below lists all public protocols in the Standard Library with associated type requirements, along with their proposed primary associated type, as well as a list of other associated types.

[note]: #alternatives-considered

| Protocol                                             | Primary        | Others                                                                                                                                                                                                               |
|------------------------------------------------------|----------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `Sequence`                                           | `Element`      | `Iterator`                                                                                                                                                                                                           |
| `IteratorProtocol`                                   | `Element`      | --                                                                                                                                                                                                                   |
| `Collection`                                         | `Element`      | `Index`, `Iterator`, `SubSequence`, `Indices`                                                                                                                                                                        |
| `MutableCollection`                                  | `Element`      | `Index`, `Iterator`, `SubSequence`, `Indices`                                                                                                                                                                        |
| `BidirectionalCollection`                            | `Element`      | `Index`, `Iterator`, `SubSequence`, `Indices`                                                                                                                                                                        |
| `RandomAccessCollection`                             | `Element`      | `Index`, `Iterator`, `SubSequence`, `Indices`                                                                                                                                                                        |
| `RangeReplaceableCollection`                         | `Element`      | `Index`, `Iterator`, `SubSequence`, `Indices`                                                                                                                                                                        |
| `LazySequenceProtocol`                               | -- [(1)][note] | `Element`, `Iterator`, `Elements`                                                                                                                                                                                    |
| `LazyCollectionProtocol`                             | -- [(1)][note] | `Element`, `Index`, `Iterator`, `SubSequence`, `Indices`, `Elements`                                                                                                                                                 |
| `Identifiable`                                       | `ID`           | --                                                                                                                                                                                                                   |
| `RawRepresentable`                                   | `RawValue`     | --                                                                                                                                                                                                                   |
| `RangeExpression`                                    | `Bound`        | --                                                                                                                                                                                                                   |
| `Strideable`                                         | `Stride`       | --                                                                                                                                                                                                                   |
| `SetAlgebra`                                         | `Element`      | `ArrayLiteralElement`                                                                                                                                                                                                |
| `OptionSet`                                          | -- [(2)][note] | `Element`, `ArrayLiteralElement`, `RawValue`                                                                                                                                                                         |
| `Numeric`                                            | --             | `IntegerLiteralType`, `Magnitude`                                                                                                                                                                                    |
| `SignedNumeric`                                      | --             | `IntegerLiteralType`, `Magnitude`                                                                                                                                                                                    |
| `BinaryInteger`                                      | --             | `IntegerLiteralType`, `Magnitude`, `Stride`, `Words`                                                                                                                                                                 |
| `UnsignedInteger`                                    | --             | `IntegerLiteralType`, `Magnitude`, `Stride`, `Words`                                                                                                                                                                 |
| `SignedInteger`                                      | --             | `IntegerLiteralType`, `Magnitude`, `Stride`, `Words`                                                                                                                                                                 |
| `FixedWidthInteger`                                  | --             | `IntegerLiteralType`, `Magnitude`, `Stride`, `Words`                                                                                                                                                                 |
| `FloatingPoint`                                      | --             | `IntegerLiteralType`, `Magnitude`, `Stride`, `Exponent`                                                                                                                                                              |
| `BinaryFloatingPoint`                                | --             | `IntegerLiteralType`, `FloatLiteralType`, `Magnitude`, `Stride`, `Exponent`, `RawSignificand`, `RawExponent`                                                                                                         |
| `SIMD`                                               | `Scalar`       | `ArrayLiteralElement`, `MaskStorage`                                                                                                                                                                                 |
| `SIMDStorage`                                        | --             | `Scalar`                                                                                                                                                                                                             |
| `SIMDScalar`                                         | --             | `SIMDMaskScalar`, `SIMD2Storage`, `SIMD4Storage`, ..., `SIMD64Storage`                                                                                                                                               |
| `KeyedEncodingContainerProtocol`                     | --             | `Key`                                                                                                                                                                                                                |
| `KeyedDecodingContainerProtocol`                     | --             | `Key`                                                                                                                                                                                                                |
| `ExpressibleByIntegerLiteral`                        | --             | `IntegerLiteralType`                                                                                                                                                                                                 |
| `ExpressibleByFloatLiteral`                          | --             | `FloatLiteralType`                                                                                                                                                                                                   |
| `ExpressibleByBooleanLiteral`                        | --             | `BooleanLiteralType`                                                                                                                                                                                                 |
| `ExpressibleByUnicodeScalarLiteral`                  | --             | `UnicodeScalarLiteralType`                                                                                                                                                                                           |
| `ExpressibleByExtended-`<br>`GraphemeClusterLiteral` | --             | `UnicodeScalarLiteralType`, `ExtendedGraphemeClusterLiteralType`                                                                                                                                                     |
| `ExpressibleByStringLiteral`                         | --             | `UnicodeScalarLiteralType`, `ExtendedGraphemeClusterLiteralType`, `StringLiteralType`                                                                                                                                |
| `ExpressibleByStringInterpolation`                   | --             | `UnicodeScalarLiteralType`, `ExtendedGraphemeClusterLiteralType`, `StringLiteralType`, `StringInterPolation`                                                                                                         |
| `ExpressibleByArrayLiteral`                          | --             | `ArrayLiteralElement`                                                                                                                                                                                                |
| `ExpressibleByDictionaryLiteral`                     | --             | `Key`, `Value`                                                                                                                                                                                                       |
| `StringInterpolationProtocol`                        | --             | `StringLiteralType`                                                                                                                                                                                                  |
| `Unicode.Encoding`                                   | --             | `CodeUnit`, `EncodedScalar`, `ForwardParser`, `ReverseParser`                                                                                                                                                        |
| `UnicodeCodec`                                       | --             | `CodeUnit`, `EncodedScalar`, `ForwardParser`, `ReverseParser`                                                                                                                                                        |
| `Unicode.Parser`                                     | --             | `Encoding`                                                                                                                                                                                                           |
| `StringProtocol`                                     | --             | `Element`, `Index`, `Iterator`, `SubSequence`, `Indices`, `UnicodeScalarLiteralType`, `ExtendedGraphemeClusterLiteralType`, `StringLiteralType`, `StringInterPolation`, `UTF8View`, `UTF16View`, `UnicodeScalarView` |
| `CaseIterable`                                       | --             | `AllCases`                                                                                                                                                                                                           |
| `Clock`                                              | `Duration`     | `Instant`                                                                                                                                                                                                            |
| `InstantProtocol`                                    | `Duration`     | --                                                                                                                                                                                                                   |
| `AsyncIteratorProtocol`                              | -- [(3)][note] | `Element`                                                                                                                                                                                                            |
| `AsyncSequence`                                      | -- [(3)][note] | `AsyncIterator`, `Element`                                                                                                                                                                                           |
| `GlobalActor`                                        | --             | `ActorType`                                                                                                                                                                                                          |
| `DistributedActor`                                   | -- [(4)][note] | `ID`, `ActorSystem`, `SerializationRequirement`                                                                                                                                                                      |
| `DistributedActorSystem`                             | -- [(4)][note] | `ActorID`, `SerializationRequirement`, `InvocationEncoder`, `InvocationDecoder`, `ResultHandler`                                                                                                                     |
| `DistributedTargetInvocationEncoder`                 | -- [(4)][note] | `SerializationRequirement`                                                                                                                                                                                           |
| `DistributedTargetInvocationDecoder`                 | -- [(4)][note] | `SerializationRequirement`                                                                                                                                                                                           |
| `DistributedTargetInvocationResultHandler`           | -- [(4)][note] | `SerializationRequirement`                                                                                                                                                                                           |

As of Swift 5.6, the following public protocols don't have associated type requirements, so they are outside of the scope of this proposal.

```swift
Equatable, Hashable, Comparable, Error, AdditiveArithmetic,
DurationProtocol, Encodable, Decodable, Encoder, Decoder,
UnkeyedEncodingContainer, UnkeyedDecodingContainer,
SingleValueEncodingContainer, SingleValueDecodingContainer,
ExpressibleByNilLiteral, CodingKeyRepresentable,
CustomStringConvertible, LosslessStringConvertible, TextOutputStream,
TextOutputStreamable, CustomPlaygroundDisplayConvertible,
CustomReflectable, CustomLeafReflectable, MirrorPath,
RandomNumberGenerator, CVarArg, Sendable, UnsafeSendable, Actor,
AnyActor, Executor, SerialExecutor, DistributedActorSystemError
```

## Detailed design

```swift
public protocol Sequence<Element>
public protocol IteratorProtocol<Element>
public protocol Collection<Element>: Sequence
public protocol MutableCollection<Element>: Collection
public protocol BidirectionalCollection<Element>: Collection
public protocol RandomAccessCollection<Element>: BidirectionalCollection
public protocol RangeReplaceableCollection<Element>: Collection

public protocol Identifiable<ID>
public protocol RawRepresentable<RawValue>
public protocol RangeExpression<Bound>
public protocol Strideable<Stride>: Comparable

public prococol SetAlgebra<Element>: Equatable, ExpressibleByArrayLiteral

public protocol SIMD<Scalar>: ...

public protocol Clock<Duration>: Sendable
public protocol InstantProtocol<Duration>: Comparable, Hashable, Sendable
```

## Source compatibility

None. The new annotations enable new ways to use these protocols, but they are tied to new syntax, and they do not affect existing code.

## Effect on ABI stability

None. The annotations aren't ABI impacting, and the new capabilities deploy back to any previous Swift Standard Library release.

## Effect on API resilience

Once introduced, primary associated types cannot be removed from a protocol or reordered without breaking source compatibility.

[SE-0346] requires usage sites to always list every primary associated type defined by a protocol. Until/unless this restriction is lifted, adding a new primary associated type to a protocol that already has some will also be a source breaking change.

Therefore, we will not be able to make any changes to the list of primary associated types of any of the protocols that are affected by this proposal once this ships in a Standard Library release.

## Alternatives considered

(1) It is tempting to declare `Element` as the primary associated type for `LazySequenceProtocol` and `LazyCollectionProtocol`, for consistency with other protocols in the collection hierarchy. However, in actual use, `Elements` seems just as useful (if not more) to be easily constrained. We left the matter of selecting one of these as primary unresolved for now; as we get more experience with the lightweight constraint syntax, we may revisit these protocols.

(2) In the `OptionSet` protocol, the `Element` type is designed to always be `Self`, so `RawValue` would be the most practical choice for the primary associated type. However, to avoid potential confusion, we left `OptionSet` without a primary associated type annotation.

(3) `AsyncSequence` and `AsyncIteratorProtocol` logically ought to have `Element` as their primary associated type. However, we have [ongoing evolution discussions][rethrows] about adding a precise error type to these. If those discussions bear fruit, then it's possible we may want to _also_ mark the potential new `Error` associated type as primary. To prevent source compatibility complications, adding primary associated types to these two protocols is deferred to a future proposal.

[rethrows]: https://forums.swift.org/t/se-0346-lightweight-same-type-requirements-for-primary-associated-types/55869/70

(4) Declaring primary associated types on the distributed actor protocols would be desirable, but it was [deferred to a future proposal](https://forums.swift.org/t/pitch-primary-associated-types-in-the-standard-library/56426/47), to prevent interfering with potential future language improvements that would make them more useful in this use case.

## Revisions

- [2022-05-28](https://github.com/swiftlang/swift-evolution/blob/716db41ccefde348ac38bd2fd1eb5bd7842be7b6/proposals/0358-primary-associated-types-in-stdlib.md): Initial proposal version.
- 2022-06-22: Removed the primary associated type declaration from the `OptionSet` protocol. The API guidelines section has revised wording; it no longer proposes the new guidelines for inclusion in the official Swift API Guidelines document. Adjusted wording to prefer the term "lightweight constraint syntax" to "lightweight same-type requirements", as the new syntax can be used for more than just to express same-type constraints.
