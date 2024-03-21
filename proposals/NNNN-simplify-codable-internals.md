# Simplify internals of Codable-related APIs

* Proposal: [SE-NNNN](NNNN-simplify-codable-internals.md)
* Author: [Ben Rimmington](https://github.com/benrimmington)
* Review Manager: TBD
* Status: **Awaiting review**
* Implementation: [apple/swift#63172](https://github.com/apple/swift/pull/63172)
* Review: ([pitch](https://forums.swift.org/t/simplify-internals-of-codable-related-apis/63566))

## Introduction

This proposal aims to simplify the internals of some Codable-related APIs, by
adding two [primary associated types][] and a non-failable initializer to the
standard library.

[primary associated types]: <https://github.com/apple/swift-evolution/blob/main/proposals/0358-primary-associated-types-in-stdlib.md>

## Motivation

* [`Encoder.container(keyedBy:)`][] and [`Decoder.container(keyedBy:)`][]
  methods return type-erased wrappers. These are public structures — each
  storing an instance of an internal subclass — requiring hundreds of lines
  of boilerplate code in the standard library.

* [`CodingUserInfoKey.actorSystemKey`][] and other user-defined keys have a
  [long-standing issue][] where their creation requires force-unwrapping,
  because only a failable initializer is available.

[`Encoder.container(keyedBy:)`]:      <https://developer.apple.com/documentation/swift/encoder/container(keyedby:)>
[`Decoder.container(keyedBy:)`]:      <https://developer.apple.com/documentation/swift/decoder/container(keyedby:)>
[`CodingUserInfoKey.actorSystemKey`]: <https://developer.apple.com/documentation/swift/codinguserinfokey/actorsystemkey>
[long-standing issue]:                <https://github.com/apple/swift/issues/49302>

## Proposed solution

* The public structures cannot be removed, but the internal classes can be
  replaced by [constrained existential types][]:

  ```diff
   public struct KeyedEncodingContainer<K: CodingKey> :
     KeyedEncodingContainerProtocol
   {
     public typealias Key = K

     // Use a constrained existential type.
  -  internal var _box: _KeyedEncodingContainerBase
  +  internal var _box: any KeyedEncodingContainerProtocol<Key>

     // Use a constrained opaque parameter type?
  -  public init<Container: KeyedEncodingContainerProtocol>(
  -    _ container: Container
  -  ) where Container.Key == Key {
  +  public init(_ container: some KeyedEncodingContainerProtocol<Key>) {

       // Remove the internal subclass and its base class.
  -    _box = _KeyedEncodingContainerBox(container)
  +    _box = container
     }
  ```

  (And likewise for the `KeyedDecodingContainer` structure.)

* The existing initializer cannot be made non-failable, but another initializer
  (without an argument label) can be used instead:

  ```diff
   extension CodingUserInfoKey {
  -  public static let actorSystemKey = CodingUserInfoKey(rawValue: "$distributed_actor_system")!
  +  public static let actorSystemKey = CodingUserInfoKey("$distributed_actor_system")
   }
  ```

[constrained existential types]: <https://github.com/apple/swift-evolution/blob/main/proposals/0353-constrained-existential-types.md>

## Detailed design

* Two primary associated types will be added:

  ```diff
  -public protocol KeyedEncodingContainerProtocol {
  +public protocol KeyedEncodingContainerProtocol<Key> {
     associatedtype Key: CodingKey
  ```

  ```diff
  -public protocol KeyedDecodingContainerProtocol {
  +public protocol KeyedDecodingContainerProtocol<Key> {
     associatedtype Key: CodingKey
  ```

* A non-failable initializer will be added:

  ```diff
   public struct CodingUserInfoKey: RawRepresentable, Equatable, Hashable, Sendable {
     public let rawValue: String
     public init?(rawValue: String)
  +  public init(_ rawValue: String)
  ```

## Source compatibility

The primary associated types and non-failable initializer are source-compatible
additions.

## ABI compatibility

The type-erased wrappers are ABI-public resilient structures, and ABI-private
classes. The opaque parameter type is (assumed to be) syntactic sugar for the
existing generic parameter and constraint.

## Implications on adoption

The non-failable initializer can always be emitted into the client.

## Future directions

None.

## Alternatives considered

None.
