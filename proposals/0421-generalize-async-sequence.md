# Generalize effect polymorphism for `AsyncSequence` and `AsyncIteratorProtocol`

* Proposal: [SE-0421](0421-generalize-async-sequence.md)
* Authors: [Doug Gregor](https://github.com/douggregor), [Holly Borla](https://github.com/hborla)
* Review Manager: [Freddy Kellison-Linn](https://github.com/Jumhyn)
* Status: **Implemented (Swift 6.0)**
* Review: ([pitch](https://forums.swift.org/t/pitch-generalize-asyncsequence-and-asynciteratorprotocol/69283))([review](https://forums.swift.org/t/se-0421-generalize-effect-polymorphism-for-asyncsequence-and-asynciteratorprotocol/69662)) ([acceptance](https://forums.swift.org/t/accepted-se-0421-generalize-effect-polymorphism-for-asyncsequence-and-asynciteratorprotocol/69973))

## Introduction

This proposal generalizes `AsyncSequence` in two ways:
1. Proper `throws` polymorphism is accomplished with adoption of typed throws.
2. A new overload of the `next` requirement on `AsyncIteratorProtocol` includes an isolated parameter to abstract over actor isolation.

## Table of Contents

* [Introduction](#introduction)
* [Motivation](#motivation)
* [Proposed solution](#proposed-solution)
* [Detailed design](#detailed-design)
    + [Adopting typed throws](#adopting-typed-throws)
        - [Error type inference from `for try await` loops](#error-type-inference-from-for-try-await-loops)
    + [Adopting primary associated types](#adopting-primary-associated-types)
    + [Adopting isolated parameters](#adopting-isolated-parameters)
    + [Default implementations of `next()` and `next(isolation:)`](#default-implementations-of-next-and-nextisolation)
    + [Associated type inference for `AsyncIteratorProtocol` conformances](#associated-type-inference-for-asynciteratorprotocol-conformances)
* [Source compatibility](#source-compatibility)
* [ABI compatibility](#abi-compatibility)
* [Implications on adoption](#implications-on-adoption)
* [Future directions](#future-directions)
    + [Add a default argument to `next(isolation:)`](#add-a-default-argument-to-nextisolation)
* [Alternatives considered](#alternatives-considered)
    + [Avoiding an existential parameter in `next(isolation:)`](#avoiding-an-existential-parameter-in-nextisolation)
* [Acknowledgments](#acknowledgments)

## Motivation

`AsyncSequence` and `AsyncIteratorProtocol` were intended to be polymorphic over the `throws` effect and actor isolation. However, the current API design has serious limitations that impact expressivity in generic code, `Sendable` checking, and runtime performance.

Some `AsyncSequence`s can throw during iteration, and others never throw. To enable callers to only require `try` when the given sequence can throw, `AsyncSequence` and `AsyncIteratorProtocol` used an experimental feature to try to capture the throwing behavior of a protocol. However, this approach was insufficiently general, which has also [prevented `AsyncSequence` from adopting primary associated types](https://forums.swift.org/t/se-0346-lightweight-same-type-requirements-for-primary-associated-types/55869/70). Primary associated types on `AsyncSequence` would enable hiding concrete implementation details behind constrained opaque or existential types, such as in transformation APIs on `AsyncSequence`:

```swift
extension AsyncSequence {
  // 'AsyncThrowingMapSequence' is an implementation detail hidden from callers.
  public func map<Transformed>(
    _ transform: @Sendable @escaping (Element) async throws -> Transformed
  ) -> some AsyncSequence<Transformed, any Error> { ... }
}
```

Additionally, `AsyncSequence` types are designed to work with `Sendable` and non-`Sendable` element types, but it's currently impossible to use an `AsyncSequence` with non-`Sendable` elements in an actor-isolated context:

```swift
class NotSendable { ... }

@MainActor
func iterate(over stream: AsyncStream<NotSendable>) {
  for await element in stream { // warning: non-sendable type 'NotSendable?' returned by implicitly asynchronous call to nonisolated function cannot cross actor boundary

  }
}
```

Because `AsyncIteratorProtocol.next()` is `nonisolated async`, it always runs on the generic executor, so calling it from an actor-isolated context crosses an isolation boundary. If the result is non-`Sendable`, the call is invalid under strict concurrency checking.

More fundamentally, calls to `AsyncIteratorProtocol.next()` from an actor-isolated context are nearly always invalid in practice today. Most concrete `AsyncIteratorProtocol` types are not `Sendable`; concurrent iteration using `AsyncIteratorProtocol` is a programmer error, and the iterator is intended to be used/mutated from the isolation domain that formed it. However, when an iterator is formed in an actor-isolated context and `next()` is called, the non-`Sendable` iterator is passed across isolation boundaries, resulting in a diagnostic under strict concurrency checking.

Finally, `next()` always running on the generic executor is the source of unnecessary hops between an actor and the generic executor.

## Proposed solution

This proposal introduces a new associated type `Failure` to  `AsyncSequence` and `AsyncIteratorProtocol`, adopts both `Element` and `Failure` as primary associated types, adds a new protocol requirement to `AsyncIteratorProtocol` that generalizes the existing `next()` requirement by throwing the `Failure` type, and adds an `isolated` parameter to the new requirement to abstract over actor isolation:

```swift
@available(SwiftStdlib 5.1, *)
protocol AsyncIteratorProtocol<Element, Failure> {
  associatedtype Element

  mutating func next() async throws -> Element?

  @available(SwiftStdlib 6.0, *)
  associatedtype Failure: Error = any Error

  @available(SwiftStdlib 6.0, *)
  mutating func next(isolation actor: isolated (any Actor)?) async throws(Failure) -> Element?
}

@available(SwiftStdlib 5.1, *)
public protocol AsyncSequence<Element, Failure> {
  associatedtype AsyncIterator: AsyncIteratorProtocol
  associatedtype Element where AsyncIterator.Element == Element

  @available(SwiftStdlib 6.0, *)
  associatedtype Failure = AsyncIterator.Failure where AsyncIterator.Failure == Failure

  func makeAsyncIterator() -> AsyncIterator
}
```

The new `next(isolation:)` has a default implementation so that conformances will continue to behave as they do today. Code generation for `for-in` loops will switch over to calling `next(isolation:)` instead of `next()` when the context has appropriate availability.

## Detailed design

### Adopting typed throws

Concrete `AsyncSequence` and `AsyncIteratorProtocol` types determine whether calling `next()` can `throw`. This can be described in each protocol with a `Failure` associated type that is thrown by the `AsyncIteratorProtcol.next(isolation:)` requirement. Describing the thrown error with an associated type allows conformances to fulfill the requirement with a type parameter, which means that libraries do not need to expose separate throwing and non-throwing concrete types that otherwise have the same async iteration functionality.

#### Error type inference from `for try await` loops

The `Failure` associated type is only accessible at runtime in the Swift 6.0 standard library; code running against older standard library versions does not include the `Failure` requirement in the witness tables for `AsyncSequence` and `AsyncIteratorProtocol` conformances. This impacts error type inference from `for try await` loops.

When the thrown error type of an `AsyncIteratorProtocol` is available, either through the associated type witness (because the context has appropriate availability) or because the iterator type is concrete, iteration over an async sequence throws its `Failure` type:

```swift
struct MyAsyncIterator: AsyncIteratorProtocol {
  typealias Failure = MyError
  ...
}

func iterate<S: AsyncSequence>(over s: S) where S.AsyncIterator == MyAsyncIterator {
  let closure = {
    for try await element in s {
      print(element)
    }
  }
}
```

In the above code, the type of `closure` is `() async throws(MyError) -> Void`.

When the thrown error type of an `AsyncIteratorProtocol` is not available, iteration over an async sequence throws `any Error`:

```swift
@available(SwiftStdlib 5.1, *)
func iterate(over s: some AsyncSequence) {
  let closure = {
    for try await element in s {
      print(element)
    }
  }
}
```

In the above code, the type of `closure` is `() async throws(any Error) -> Void`.

When the `Failure` type of the given async sequence is constrained to `Never`, `try` is not required in the `for-in` loop:

```swift
struct MyAsyncIterator: AsyncIteratorProtocol {
  typealias Failure = Never
  ...
}

func iterate<S: AsyncSequence>(over s: S) where S.AsyncIterator == MyAsyncIterator {
  let closure = {
    for await element in s {
      print(element)
    }
  }
}
```

In the above code, the type of `closure` is `() async -> Void`.

### Adopting primary associated types

The `Element` and `Failure` associated types are promoted to primary associated types. This enables using constrained existential and opaque `AsyncSequence` and `AsyncIteratorProtocol` types, e.g. `some AsyncSequence<Element, Never>` or `any AsyncSequence<Element, any Error>`.

### Adopting isolated parameters

The `next(isolation:)` requirement abstracts over actor isolation using [isolated parameters](/proposals/0313-actor-isolation-control.md). For callers to `next(isolation:)` that pass an iterator value that cannot be transferred across isolation boundaries under [SE-0414: Region based isolation](/proposals/0414-region-based-isolation.md), the call is only valid if it does not cross an isolation boundary. Explicit callers can pass in a value of `#isolation` to use the isolation of the caller, or `nil` to evaluate `next(isolation:)` on the generic executor.

Desugared async `for-in` loops will call `AsyncIteratorProtocol.next(isolation:)` instead of `next()` when the context has appropriate availability, and pass in an isolated argument value of `#isolation` of type `(any Actor)?`. The `#isolation` macro always expands to the isolation of the caller so that the call does not cross an isolation boundary.

### Default implementations of `next()` and `next(isolation:)`

Because existing `AsyncIteratorProtocol`-conforming types only implement `next()`, the standard library provides a default implementation of `next(isolation:)`:

```swift
extension AsyncIteratorProtocol {
  /// Default implementation of `next(isolation:)` in terms of `next()`, which is
  /// required to maintain backward compatibility with existing async iterators.
  @available(SwiftStdlib 6.0, *)
  @available(*, deprecated, message: "Provide an implementation of 'next(isolation:)'")
  public mutating func next(isolation actor: isolated (any Actor)?) async throws(Failure) -> Element? {
    nonisolated(unsafe) var unsafeIterator = self
    do {
      let element = try await unsafeIterator.next()
      self = unsafeIterator
      return element
    } catch {
      throw error as! Failure
    }
  }
}
```

Note that the default implementation of `next(isolation:)` necessarily violates `Sendable` checking in order to pass `self` from a possibly-isolated context to a `nonisolated` one. Though this is generally unsafe, this is how calls to `next()` behave today, so existing conformances will maintain the behavior they already have. Implementing `next(isolation:)` directly will eliminate the unsafety.

To enable conformances of `AsyncIteratorProtocol` to only implement `next(isolation:)`, a default implementation is also provided for `next()`:

```swift
extension AsyncIteratorProtocol {
  @available(SwiftStdlib 6.0, *)
  public mutating func next() async throws -> Element? {
    // Callers to `next()` will always run `next(isolation:)` on the generic executor.
    try await next(isolation: nil)
  }
}
```

Both function requirements of `AsyncIteratorProtocol` have default implementations that are written in terms of each other, meaning that it is a programmer error to implement neither of them. Types that are available prior to the Swift 6.0 standard library must provide an implementation of `next()`, because the default implementation is only available with the Swift 6.0 standard library.

To avoid silently allowing conformances that implement neither requirement, and to facilitate the transition of conformances from `next()` to `next(isolation:)`, we add a new availability rule where the witness checker diagnoses a protocol conformance that uses an deprecated, obsoleted, or unavailable default witness implementation. Deprecated implementations will produce a warning, while obsoleted and unavailable implementations will produce an error.

Because the default implementation of `next(isolation:)` is deprecated, conformances that do not provide a direct implementation will produce a warning. This is desirable because the default implementation of `next(isolation:)` violates `Sendable` checking, so while it's necessary for source compatibilty, it's important to aggressively suggest that conforming types implement the new method.

### Associated type inference for `AsyncIteratorProtocol` conformances

When an `AsyncIteratorProtocol`-conforming type provides a `next(isolation:)` function, the `Failure` type is inferred based on whether (and what) `next(isolation:)` throws using the rules described in [SE-0413](/proposals/0413-typed-throws.md).

If the `AsyncIteratorProtocol`-conforming type uses the default implementation of `next(isolation:)`, then the `Failure` associated type is inferred from the `next` function instead. Whatever type is thrown from the `next` function (including `Never` if it is non-throwing) is inferred as the `Failure` type.

## Source compatibility

The new requirements to `AsyncSequence` and `AsyncIteratorProtocol` are additive, with default implementations and `Failure` associated type inference heuristics that ensure that existing types that conform to these protocols will continue to work.

The experimental "rethrowing conformances" feature used by `AsyncSequence` and `AsyncIteratorProtocol` presents some challenges for source compatibility. Namely, one can declare a `rethrows` function that considers conformance to these rethrowing protocols as sources of errors for rethrowing. For example, the following `rethrows` function is currently valid:

```swift
extension AsyncSequence {
  func contains(_ value: Element) rethrows -> Bool where Element: Hashable { ... }
}
```

With the removal of the experimental "rethrowing conformances" feature, this function becomes ill-formed because there is no closure argument that can throw. To preserve source compatibility for such functions, this proposal introduces a specific rule that allows requirements on `AsyncSequence` and `AsyncIteratorProtocol` to be involved in `rethrows` checking: a `rethrows` function is considered to be able to throw `T.Failure` for every `T: AsyncSequence` or `T: AsyncIteratorProtocol` conformance requirement. In the case of this `contains` operation, that means it can throw `Self.Failure`. The rule permitting the definition of these `rethrows` functions will only be permitted prior to Swift 6.

## ABI compatibility

This proposal is purely an extension of the ABI of the standard library and does not change any existing features. Note that the addition of a new `next(isolation:)` requirement, rather than modifying the existing `next()` requirement, is necessary to maintain ABI compatibility, because changing `next()` to abstract over actor isolation requires passing the actor as a parameter in order to hop back to that actor after any `async` calls in the implementation. The typed throws ABI is also different from the rethrows ABI, so the adoption of typed throws alone necessitates a new requirement.

## Implications on adoption

The associated `Failure` types of `AsyncSequence` and `AsyncIteratorProtocol` are only available at runtime with the Swift 6.0 standard library, because code that runs against prior standard library versions does not have a witness table entry for `Failure`. Code that needs to access the `Failure` type through the associated type, e.g. to dynamic cast to it or constrain it in a generic signature, must be availability constrained. For this reason, the default implementations of `next()` and `next(isolation:)` have the same availability as the Swift 6.0 standard library.

This means that concrete `AsyncIteratorProtocol` conformances cannot switch over to implementing `next(isolation:)` only (without providing an implementation of `next()`) if they are available earlier than the Swift 6.0 standard library.

Similarly, primary associated types of `AsyncSequence` and `AsyncIteratorProtocol` must be gated behind Swift 6.0 availability.

Once the concrete `AsyncIteratorProtocol` types in the standard library, such as `Async{Throwing}Stream.Iterator`, implement `next(isolation:)` directly, code that iterates over those concrete `AsyncSequence` types in an actor-isolated context may exhibit fewer hops to the generic executor at runtime.

## Future directions

### Add a default argument to `next(isolation:)`

Most calls to `next(isolation:)` will pass the isolation of the enclosing context. We could consider lifting the restriction that protocol requirements cannot have default arguments, and adding a default argument value of `#isolated` as described in the [pitch for actor isolation inheritance](https://forums.swift.org/t/pitch-inheriting-the-callers-actor-isolation/68391).

## Alternatives considered

### Avoiding an existential parameter in `next(isolation:)`

The isolated parameter to `next(isolation:)` has existential type `(any Actor)?` because a `nil` value is used to represent `nonisolated`. There is no concrete `Actor` type that describes a `nonisolated` context, which necessitates using `(any Actor)?` instead of `some Actor` or `(some Actor)?`. Potential alternatives to this are:

1. Represent `nonisolated` with some other value than `nil`, or a specific declaration in the standard library that has a concrete optional actor type to enable `(some Actor)?`. Any solution in this category requires the compiler to have special knowledge of the value that represents `nonisolated` for actor isolation checking of the call.
2. Introduce a separate entrypoint for `next(isolation:)` that is always `nonisolated`. This defeats the purpose of having a single implementation of `next(isolation:)` that abstracts over actor isolation.

Note that the use of an existential type `(any Actor)?` means that [embedded Swift](/visions/embedded-swift.md) would need to support class existentials in order to use `next(isolation:)`.

## Acknowledgments

Thank you to Franz Busch and Konrad Malawski for starting the discussions about typed throws and primary associated type adoption for `AsyncSequence` and `AsyncIteratorProtocol` in the [Typed throws in the Concurrency module](https://forums.swift.org/t/pitch-typed-throws-in-the-concurrency-module/68210/1) pitch. Thank you to John McCall for specifying the rules for generalized isolated parameters in the [pitch for inheriting the caller's actor isolation](https://forums.swift.org/t/pitch-inheriting-the-callers-actor-isolation/68391).
