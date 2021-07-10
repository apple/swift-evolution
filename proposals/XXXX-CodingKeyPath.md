# `CodingKeyPath`

* Proposal: [SE-NNNN](NNNN-codingkeypath.md)
* Author: [Cal Stephens](https://twitter.com/calstephens98)
* Review Manager: TBD
* Status: **Waiting for Review**
* Implementation: [`apple/swift#30570`](https://github.com/apple/swift/pull/30570)

## Introduction

Today, encoding and decoding `Codable` objects using the compiler's synthesized implementation requires that your object graph has a one-to-one mapping to the object graph of the target payload. This decreases the control that authors have over their `Codable` models.

I propose that we add a new `CodingKeyPath` type that allows consumers to key into nested objects using [dot notation](https://developer.apple.com/documentation/objectivec/nsobject/1416468-value).

_Swift-evolution thread: [CodingKeyPath: Add support for Encoding and Decoding nested objects with dot notation](https://forums.swift.org/t/codingkeypath/34710)_

## Motivation

Application authors often have little to no control over the structure of the encoded payloads they receive. It is often desirable to rename or reorganize fields of the payload at the time of decoding.

Here is a theoretical JSON payload representing a Swift Evolution proposal ([SE-0274](https://github.com/apple/swift-evolution/blob/master/proposals/0274-magic-file.md)):

```json
{
  "id": "SE-0274",
  "title": "Concise magic file names",
  "metadata": {
    "review_start_date": "2020-01-08T00:00:00Z",
    "review_end_date": "2020-01-16T00:00:00Z"
  }
}
```

The consumer of this payload may prefer to hoist fields from the `metadata` object to the root level:

```swift
struct EvolutionProposal: Codable {
  var id: String
  var title: String
  var reviewStartDate: Date
  var reviewEndDate: Date
}
```

Today, this would require writing a fair amount of boilerplate. The consumer would need to either [write custom encoding and decoding implementation](https://github.com/calda/CodingKeyPath/blob/master/Playgrounds/NestedCodingKeys_WithCustomCodingImplementation.playground/Contents.swift) or [proxy to Codable subtypes](https://github.com/calda/CodingKeyPath/blob/master/Playgrounds/NestedCodingKeys_ProxyingToCodableSubtypes.playground/Contents.swift).

## Proposed solution

I propose that we add a new `CodingKeyPath` type that allows consumers to key into nested objects using [dot notation](https://developer.apple.com/documentation/objectivec/nsobject/1416468-value).

```swift
struct EvolutionProposal: Codable {
  var id: String
  var title: String
  var reviewStartDate: Date
  var reviewEndDate: Date
    
  enum CodingKeyPaths: String, CodingKeyPath {
    case id
    case title
    case reviewStartDate = "metadata.review_start_date"
    case reviewEndDate = "metadata.review_end_date"
  }
}
```

### Prior art

[`NSDictionary.value(forKeyPath:)`](https://developer.apple.com/documentation/objectivec/nsobject/1416468-value) supports retrieving nested values using dot notation.

Many existing model parsing frameworks support dot notation for accessing nested keys. Some examples include:
 - **[Mantle](https://github.com/Mantle/Mantle#mtlmodel)**, _"Model framework for Cocoa and Cocoa Touch"_
 - **[Unbox](https://github.com/JohnSundell/Unbox#key-path-support)**, _"The easy to use Swift JSON decoder"_
 - **[ObjectMapper](https://github.com/tristanhimmelman/ObjectMapper#easy-mapping-of-nested-objects)**, _"Simple JSON Object mapping written in Swift"_

## Detailed design

### New Standard Library types

```swift
/// A type that can be used as a key path for encoding and decoding.
public protocol CodingKeyPath {
  /// The components of this path. Derived automatically for a `CodingKeyPaths` enum:
  ///
  ///   enum CodingKeyPaths: String, CodingKeyPath {
  ///     /// components = ["rootValue"]
  ///     case rootValue       
  ///
  ///     /// components = ["nestedObject", "value"]
  ///     case nestedValue = "nestedObject.value" 
  ///  }
  ///
  var components: [CodingKey] { get }
}

/// A container for encoding with a `CodingKeyPath` type.
///  - Internally wraps a `KeyedEncodingContainer`. 
///  - Recursively follows a `CodingKeyPath` by encoding a `nestedContainer` for each component.
public struct KeyPathEncodingContainer<K: CodingKeyPath> {
  public mutating func encode<T>(_ value: T, forKeyPath keyPath: K) throws where T: Encodable
  public mutating func encodeIfPresent<T>(_ value: T?, forKeyPath keyPath: K) throws where T: Encodable
}

/// A container for decoding with a `CodingKeyPath` type.
///  - Internally wraps a `KeyedDecodingContainer`. 
///  - Recursively follows a `CodingKeyPath` by decoding a`nestedContainer` for each component.
public struct KeyPathDecodingContainer<K> where K: CodingKeyPath {  
  public func decode<T>(_ type: T.Type, forKeyPath keyPath: K) throws -> T where T: Decodable
  public func decodeIfPresent<T>(_ type: T.Type, forKeyPath keyPath: K) throws -> T? where T: Decodable
}
```

### New Standard Library methods

This proposal doesn't add any new requirements on the `Encoder` and `Decoder` protocols, so all existing implementations (`JSONEncoder`, `PlistDecoder`, etc.) will receive this behavior automatically.
 - `KeyPathEncodingContainer` and `KeyPathDecodingContainer` simply wrap the existing `KeyedEncodingContainer` and `KeyedEncodingContainer` types, so they don't require any additional support.
 
```swift
public extension Encoder {
  func keyPathContainer<KeyPath>(keyedBy type: KeyPath.Type) -> KeyPathEncodingContainer<KeyPath> where KeyPath: CodingKeyPath
}

public extension KeyedEncodingContainer {
  mutating func nestedKeyPathContainer<KeyPath>(keyedBy type: KeyPath.Type, forKey key: Key) throws -> KeyPathEncodingContainer<KeyPath> where KeyPath: CodingKeyPath
}

public extension UnkeyedEncodingContainer {
  mutating func nestedKeyPathContainer<KeyPath>(keyedBy type: KeyPath.Type) throws -> KeyPathEncodingContainer<KeyPath> where KeyPath: CodingKeyPath
}
```

```swift
public extension Decoder {
  func keyPathContainer<KeyPath>(keyedBy type: KeyPath.Type) throws -> KeyPathDecodingContainer<KeyPath> where KeyPath: CodingKeyPath
}

public extension KeyedDecodingContainer {
  mutating func nestedKeyPathContainer<KeyPath>(keyedBy type: KeyPath.Type, forKey key: Key) throws -> KeyPathDecodingContainer<KeyPath> where KeyPath: CodingKeyPath
}

public extension UnkeyedDecodingContainer {
  mutating func nestedKeyPathContainer<KeyPath>(keyedBy type: KeyPath.Type) throws -> KeyPathDecodingContainer<KeyPath> where KeyPath: CodingKeyPath
}
```

### Codable Conformance Synthesis

The compiler with synthesize `init(from decoder: Decoder)` and `encode(to encoder: Encoder)` implementations for types that provide a `CodingKeyPaths` enum.

 - It is invalid for a type to provide both a `CodingKeys` enum and a `CodingKeyPaths` enum.
 - If a `Codable` type doesn't provide either a `CodingKeys` enum or a `CodingKeyPaths` enum, the compiler will synthesize a `CodingKeys` enum.
 - The compiler will never automatically synthesize a `CodingKeyPaths` enum.

```swift
struct EvolutionProposal: Codable {
  var id: String
  var title: String
  var reviewStartDate: Date
  var reviewEndDate: Date

  enum CodingKeyPaths: String, CodingKeyPath {
    case id
    case title
    case reviewStartDate = "metadata.reviewStartDate"
    case reviewEndDate = "metadata.reviewEndDate"
  }
    
  /// Synthesized by the compiler:
  init(from decoder: Decoder) throws {
    let container = try decoder.keyPathContainer(keyedBy: CodingKeyPaths.self)
    id = try container.decode(String.self, forKeyPath: .id)
    title = try container.decode(String.self, forKeyPath: .title)
    reviewStartDate = try container.decode(Date.self, forKeyPath: .reviewStartDate)
    reviewEndDate = try container.decode(Date.self, forKeyPath: .reviewEndDate)
  }
    
  /// Synthesized by the compiler:
  func encode(to encoder: Encoder) throws {
    var container = encoder.keyPathContainer(keyedBy: CodingKeyPaths.self)
    try container.encode(id, forKeyPath: .id)
    try container.encode(title, forKeyPath: .title)
    try container.encode(reviewStartDate, forKeyPath: .reviewStartDate)
    try container.encode(reviewEndDate, forKeyPath: .reviewEndDate)
  }
    
}
```

## Source compatibility

This proposal is purely additive, so it has no appreciable effect on source compatibility.

 - Code synthesis behavior and/or source validity may change for Codable models that currently have a subtype named `CodingKeyPaths`.
 - A quick [GitHub search](https://github.com/search?q=enum+CodingKeyPaths&type=Code) for `enum CodingKeyPaths` doesn't yield any relevant results, so this seems like a non-issue.

## Effect on ABI stability

This proposal is purely additive, so it has no effect on ABI stability.

## Effect on API resilience

This proposal is purely additive to the public API of the Standard Library. If this proposal was adopted and implemented, it would not be able to be removed resiliently.

## Alternatives considered

### Support indexing into arrays or other advanced operations

This design could potentially support advanced operations like indexing into arrays (`metadata.authors[0].email`, etc). Objective-C Key-Value Coding paths, for example, has a [very complex and sophisticated](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/KeyValueCoding/CollectionOperators.html#//apple_ref/doc/uid/20002176-BAJEAIEE) DSL. 
 - The author believes that there isn't enough need or existing precident to warrant a more complex design.
 - Indexing into arrays seems useful on the surface, but would be quite limited in practice.
    - For example, you would be able to index into the first element of an array (`[0]`) but not the last element of the array.
    - Additionally, `UnkeyedEncodingContainer` and `UnkeyedDecodingContainer` only support sequential access (no performant support for random access).

### Name the type `CodingPath` instead of `CodingKeyPath`

In the pitch thread for this proposal, it [was brought up](https://forums.swift.org/t/codingkeypath-add-support-for-encoding-and-decoding-nested-objects-with-dot-notation/34710/14?u=cal) that the name `CodingKeyPath` could potentially cause confusion with the existing `KeyPath` type. We could potentially choose a different name for this type, like `CodingPath`. 

We would also need to rename the other types and methods added in this proposal:
- `encoder.keyPathContainer(keyedBy: CodingKeyPaths.self)` would become `encoder.pathContainer(keyedBy: CodingPaths.self)`
- `KeyPathEncodingContainer` would become `PathEncodingContainer`

### Make this the default behavior for `CodingKeys`

Valid JSON keys may contain dots:

```json
{
    "id": "SE-0274",
    "title": "Concise magic file names",
    "metadata.review_start_date": "2020-01-08T00:00:00Z",
    "metadata.review_end_date": "2020-01-16T00:00:00Z"
}
```

It's practically guaranteed that there are existing `Codable` models that rely on this behavior. We can't add dot-notation keypath semantics to the existing `CodingKeys` type without breaking backwards compatibility for these models.

 - We could make this the default decoding behavior without breaking backwards compatibility by preferring the flat key when an exact match is present.

 - We cannot make this the default encoding behavior without breaking backwards compatibility. Encoding must be a one-to-one mapping (unlike decoding, which can potentially be a many-to-one mapping).

### Enable this behavior by setting a flag on the encoder or decoder

A [previous version](https://forums.swift.org/t/add-support-for-encoding-and-decoding-nested-json-keys/34039) of this proposal added `NestedKeyEncodingStrategy` and `NestedKeyDecodingStrategy` configuration flags to `Foundation.JSONEncoder` and `Foundation.JSONDecoder`:

```swift
let decoder = JSONDecoder()
decoder.nestedKeyDecodingStrategy = .useDotNotation
try decoder.decode(EvolutionProposal.self, from: Data(originalJsonPayload.utf8))
```

Tony Parker (on the Foundation team at Apple) [noted two main drawbacks](https://forums.swift.org/t/add-support-for-encoding-and-decoding-nested-json-keys/34039/11?u=cal) to that approach: 

> 1. It applies "globally" across the entire archive. That moves part of the behavior of how encode/decode works from the type itself (where the most knowledge about structure lies) into the encoder/decoder.
>
> 2. It does not apply across different kinds of encoders and decoders. If EvolutionProposal specified the keys with the . syntax then it would effectively require JSONEncoder to encode and decode itself, because part of the data structure is now part of the key name instead.

This `CodingKeyPaths` approach described in this proposal is:
 1. Configured on a per-type basis
 2. Compatible "for free" with all existing `Encoder` and `Decoder` implementations.

### Enable this behavior by setting a static flag on the `CodingKeys` type

We could potentially allow authors to opt-in to this behavior by configuring a static flag on their `CodingKeys` type:

```swift
// In the Standard Library:

public protocol CodingKey {
  // A new protocol requirement:
  static var options: CodingKeyOptions { get }
}

public struct CodingKeyOptions {
  var dotNotationRepresentsNestedPath: Bool
}

// Default configuration to preserve source compatability and existing behavior:
public extension CodingKey {
  static var options: CodingKeyOptions {
    CodingKeyOptions(dotNotationRepresentsNestedPath: false)
  }
}
```

```swift
// EvolutionProposal.swift
struct EvolutionProposal: Codable { 
  enum CodingKeys: String, CodingKey {
    case id
    case title
    case reviewStartDate = "metadata.review_start_date"
    case reviewEndDate = "metadata.review_end_date"

    static var options: CodingKeyOptions { 
      CodingKeyOptions(dotNotationRepresentsNestedPath: true) 
    }
  }
}
```

This approach seems appealing on the surface:
 - We would only need to introduce one new type to the Standard Library (`CodingKeyOptions`)
 - `CodingKeyOptions` could be extended in the future to provide other customization points. 
     - For example, we could add a key-transformation option similar to `Foundation.JSONEncoder.KeyEncodingStrategy.convertToSnakeCase`.
     
The unfortunate downside is that it's not possible to introduce new behavior on the existing `CodingKeys` type without breaking backward compatability with existing `Encoder` and `Decoder` implementations. 

 - We could update Foundation's encoders and decoders (`JSONEncoder`, `PlistEncoder`, etc.) to respect these new options, but existing third-party implementations would also need to be updated. 
 - We shouldn't introduce options that aren't guaranteed to be respected in the concrete `Encoder` or `Decoder` implementation being used.
    
The only way to add new behavior to all existing `Encoder` and `Decoder` implementations is to introduce a new enhanced version of `CodingKey`, along with corresponding enchanced `KeyedEncodingContainer` and `KeyedDecodingContainer` wrappers:

```swift
/// Like a `CodingKey`, but with additional configuration options. ("CodingKey 2.0")
public protocol ConfigurableCodingKey {
  var stringValue: String { get }
  var intValue: Int? { get }
  static var options: CodingKeyOptions { get }
}

public struct CodingKeyOptions {
  var dotNotationRepresentsNestedPath: Bool
}

public extension Encoder {
  func container<ConfigurableKey: ConfigurableCodingKey>(keyedBy: ConfigurableKey) -> ConfiguredKeyedEncodingContainer<ConfigurableKey>
}

/// This `ConfigurableKeyedEncodingContainer` would wrap existing `KeyedEncodingContainer` implementations,
/// which would allows the Standard Library to apply additional transformations.
/// All existing `Encoder` implementations would get this support "for free".
public struct ConfigurableKeyedEncodingContainer<ConfigurableKey: ConfigurableCodingKey> {

  private let underlyingKeyedEncodingContainer: KeyedEncodingContainer<_>

  public func encode<T: Encodable>(_ value: T, atKey key: ConfigurableKey) {
    // Apply transformations to the key as specified by the `CodingKeyOptions`
    // The Standard Library can add arbitrary complex key transformations here
    // and it would apply to all existing `Encoder` implementations.
  }
   
}

// along with a corresponding `ConfigurableKeyedDecodingContainer` implementation.
```

 - The [`CodingKeyPath` implementation](https://github.com/calda/CodingKeyPath/blob/e6ae6005d9731bad8f96ad6d6bfd98c9cbc9f18f/CodingKeyPath/CodingKeyPath.swift#L32) in this proposal uses this exact approach to add additional behavior on top of the existing `KeyedEncodingContainer` and `KeyedDecodingContainer` APIs.
 
 - This would be an improvement over the existing `CodingKeys` type, but it has worse ergonomics than `CodingKeys` and the proposed `CodingKeyPaths`.
 
    - The author belives there aren't enough additional use cases for a `static` `CodingKeyOptions` customization point for it to pull its syntactic weight.
 
    - Static type-level configuration is less useful than per-property configuration, which cannot be done ergonomically using the existing `CodingKeys` design.
 
 - A "key" and a "path" have fundamentally different encoding and decoding semantics. It seems more appropriate to treat a `CodingKeyPath` as a distinct type rather than a flag or option on some `CodingKey` type.
 
### Introduce an annotation-based alternative to CodingKeys

Instead of building upon the design of `CodingKeys`, we could design an entirely new system using property-wrapper-like annotations.

```swift
struct EvolutionProposal: Codable {

  // @Key("id")  (compiler-synthesized)
  var id: String
  
  // @Key("title")  (compiler-synthesized)
  var title: String
  
  @Path("metadata.review_start_date")
  var reviewStartDate: Date
  
  @Path("metadata.review_end_date")
  var reviewEndDate: Date
  
}
```

The author believes it's more appropriate to extend and built upon the existing `CodingKeys`-based system:

 - `CodingKeys` cannot be removed or replaced, since that would be massively source-breaking.
 - The language should not include two separate / competing `Codable` systems.
