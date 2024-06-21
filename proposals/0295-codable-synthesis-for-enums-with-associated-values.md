# Codable synthesis for enums with associated values

* Proposal: [SE-0295](0295-codable-synthesis-for-enums-with-associated-values.md)
* Authors: [Dario Rexin](https://github.com/drexin)
* Review Manager: [Saleem Abdulrasool](https://github.com/compnerd)
* Status: **Implemented (Swift 5.5)**
* Implementation: [apple/swift#34855](https://github.com/apple/swift/pull/34855)
* Pitch: [Forum Discussion](https://forums.swift.org/t/codable-synthesis-for-enums-with-associated-values/41493)
* Previous Review: [Forum Discussion](https://forums.swift.org/t/se-0295-codable-synthesis-for-enums-with-associated-values/42408)
* Previous Review 2: [Forum Discussion](https://forums.swift.org/t/se-0295-codable-synthesis-for-enums-with-associated-values-second-review/44036)

## Introduction

Codable was introduced in [SE-0166](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0166-swift-archival-serialization.md)
with support for synthesizing `Encodable` and `Decodable` conformance for
`class` and `struct` types, that only contain values that also conform
to the respective protocols.

This proposal will extend the support for auto-synthesis of these conformances
to enums with associated values.

## Motivation

Currently auto-synthesis only works for enums conforming to `RawRepresentable`.
There have been discussions about adding general support for enums in the past,
but the concrete structure of the encoded values was never agreed upon.
We believe that having a solution for this is an important quality of life
improvement.

## Proposed solution

There are two models of evolution, the one designated by the language and one that is useful in the regular use of enumerations. This proposal subsets the supported cases of automatic synthesis of Codable where the two models align. We believe this to be important to retain flexibility for the user to change the shape of the enumeration.

### Structure of encoded enums

The following enum with associated values

```swift
enum Command: Codable {
  case load(key: String)
  case store(key: String, value: Int)
}
```

would be encoded to

```json
{
  "load": {
    "key": "MyKey"
  }
}
```

and 

```json
{
  "store": {
    "key": "MyKey",
    "value": 42
  }
}
```

The top-level container contains a single key that matches the name of the enum case,
which points to another container that contains the values as they would be encoded
for structs and classes.

Associated values can also be unlabeled, in which case an identifier will be generated in the form of `_$N`, where `$N` is the 0-based position of the parameter. Using generated identifiers allows more flexibility in evolution of models than using an `UnkeyedContainer` would. If a user defined parameter has an identifier that conflicts with a generated identifier, the compiler will produce a diagnostic message.

```swift
enum Command: Codable {
  case load(String)
  case store(key: String, Int)
}
```

would encoded to

```json
{
  "load": {
    "_0": "MyKey"
  }
}
```

and 

```json
{
  "store": {
    "key": "MyKey",
    "_1": 42
  }
}
```

An enum case without associated values would be encoded as an empty `KeyedContainer`,
i.e.

```swift
enum Command: Codable {
  case dumpToDisk
}
```

would encode to:

```json
{
  "dumpToDisk": {}
}
```

This allows these cases to evolve in the same manner as cases with associated values, without breaking compatibility.

### Synthesized code

Given that enums are encoded into a nested structure, there are multiple `CodingKeys` declarations. One
that contains the keys for each of the enum cases, which as before is called `CodingKeys`, and one for each case that contain the keys for the
associated values, that are prefixed with the capilized case name, e.g. `LoadCodingKeys` for `case load`.

```swift
enum Command: Codable {
  case load(key: String)
  case store(key: String, value: Int)
}
```

Would have the compiler generate the following `CodingKeys` declarations:

```swift

// contains keys for all cases of the enum
enum CodingKeys: CodingKey {
  case load
  case store
}

// contains keys for all associated values of `case load`
enum LoadCodingKeys: CodingKey {
  case key
}

// contains keys for all associated values of `case store`
enum StoreCodingKeys: CodingKey {
  case key
  case value
}
```

The `encode(to:)` implementation would look as follows:

```swift
public func encode(to encoder: Encoder) throws {
  var container = encoder.container(keyedBy: CodingKeys.self)
  switch self {
  case let .load(key):
    var nestedContainer = container.nestedContainer(keyedBy: LoadCodingKeys.self, forKey: .load)
    try nestedContainer.encode(key, forKey: .key)
  case let .store(key, value):
    var nestedContainer = container.nestedContainer(keyedBy: StoreCodingKeys.self, forKey: .store)
    try nestedContainer.encode(key, forKey: .key)
    try nestedContainer.encode(value, forKey: .value)
  }
}
```

and the `init(from:)` implementation would look as follows:

```swift
public init(from decoder: Decoder) throws {
  let container = try decoder.container(keyedBy: CodingKeys.self)
  if container.allKeys.count != 1 {
    let context = DecodingError.Context(
      codingPath: container.codingPath,
      debugDescription: "Invalid number of keys found, expected one.")
    throw DecodingError.typeMismatch(Command.self, context)
  }

  switch container.allKeys.first.unsafelyUnwrapped {
  case .load:
    let nestedContainer = try container.nestedContainer(keyedBy: LoadCodingKeys.self, forKey: .load)
    self = .load(
      key: try nestedContainer.decode(String.self, forKey: .key))
  case .store:
    let nestedContainer = try container.nestedContainer(keyedBy: StoreCodingKeys.self, forKey: .store)
    self = .store(
      key: try nestedContainer.decode(String.self, forKey: .key),
      value: try nestedContainer.decode(Int.self, forKey: .value))
  }
}
```

### User customization

For the existing cases, users can customize which properties are included in the encoded respresentation
and map the property name to a custom name for the encoded representation by providing a custom `CodingKeys`
declaration, instead of having the compiler generate one. The same should apply to the enum case.

Users can define custom `CodingKeys` declarations for all, or a subset
of the cases. If some of the cases in an enum should not be codable,
they can be excluded from the `CodingKeys` declaration.

**Example**

```swift
enum Command: Codable {
  case load(key: String)
  case store(key: String, value: Int)
  case dumpToDisk

  enum CodingKeys: CodingKey {
    case load
    case store
    // don't include `dumpToDisk`
  }
}
```

The compiler will now only synthesize the code for the `load` and `store`
cases. An attempt to en- or decode a `dumpToDisk` value will cause an error
to be thrown.

Customizing which values will be included follows the same rules as the
existing functionality. Values that are excluded must have a default value
defined, if a `Decodable` conformance should be synthesized. If only `Encodable`
is synthesized, this restriction does not apply.

**Example**

```swift
enum Command: Codable {
  case load(key: String, someLocalInfo: Int)

  // invalid, because `someLocalInfo` has no default value
  enum LoadCodingKeys: CodingKey {
    case key
  }
}
```

```swift
enum Command: Codable {
  case load(key: String, someLocalInfo: Int = 0)

  // valid, because `someLocalInfo` has a default value
  enum LoadCodingKeys: CodingKey {
    case key
  }
}
```

```swift
enum Command: Codable {
  case load(key: String)

  // invalid, because `someUnknownKey` does not map to a parameter in `load`
  enum LoadCodingKeys: CodingKey {
    case key
    case someUnknownKey
  }
}
```

Keys can be mapped to other names by conforming to `RawRepresentable`:

**Example**

```swift
enum Command: Codable {
  case load(key: String)
  case store(key: String, Int)

  enum CodingKeys: String, CodingKey {
    case load = "lade"
  }

  enum LoadCodingKeys: String, CodingKey {
    case key = "schluessel"
  }
}
```

would encode to:

```json
{
  "lade": {
    "schluessel": "MyKey"
  }
}
```

### Evolution and compatibility

Enum cases can evolve in the same way as structs and classes. Adding new fields, or removing existing ones is compatible, as long as the values are optional and the identifiers for the other cases don't change. This is in opposition to the evolution model of the language, where adding or removing associated values is a source and binary breaking change. We believe that a lot of use cases benefit from the ability to evolve the model, where source and binary compatibility are not an issue, e.g. in applications, services, or for internal types. If binary compatibility is important, evolution can be supported by having a single struct or class with all the parameters as the associated value.

### Unsupported cases

This proposal specifically does not support auto-synthesis for enums with overloaded case identifiers. This decision has been made because there is no clear way to support the feature, while also allowing the model to evolve, without severe restrictions. The separate cases in an enum typically have different semantics associated with them, so it is crucial to be able to properly identify the different cases and reject unknown cases.

#### Evolution with overloaded case identifiers

In this proposal, we are using keys as descriminators. For overloaded case names that would not be sufficient to identify the different overloads. An alternative would be to use the full name, including labels, e.g. `"store(key:,value:)"` for `case store(key: String, value: Int)`. This leads to several problems.

1. Not a valid enum case identifier, so no user customization possible
2. Not forward/backward compatible because it changes when parameters are added

An alternative solution would be to match the keys against the parameter names when decoding an object. This approach also has issues. Overloads can share parameter names, so a message that contains additional keys, that the current code version does not know about, would cause ambiguity.

**Example**

```swift
enum Test: Codable {
  case x(y: Int)
}
```

```json
{
  "x": {
    "y": 42,
    "z": "test"
  }
}
```

This could either mean that a parameter has been added to the existing case in a new code version, which should be ok under backwards compatibility rules, but could also mean that an overload has been added that partially matches the original case. If it is a different case, we should reject the message, but there is no way for the old code to decide this.

#### Ambiguities without evolution

Even when ignoring evolution, ambiguities exist with overloaded case identifiers. If two overloads share the same parameter names and one of them has additional optional parameters, the encoder can decide to drop `nil` values. This would cause ambiguities when only the shared keys are present.

**Example**

```swift
enum Test: Codable {
  case x(y: Int)
  case x(y: Int, z: String?)
}
```

Both cases would match the following input:

```json
{
  "x": {
    "y": 42
  }
}
```

Another ambiguity can be created by using the same parameter names for two overloads, only in a different order. Some formats do not guarantee the ordering of the keys, so the following definition leads to ambiguity when decoding:

```swift
enum Test: Codable {
  case x(y: Int, z: String)
  case x(z: String, y: Int)
}
```

Both cases would match the following input:

```json
{
  "x" : {
    "y": 42,
    "z": "test"
  }
}
```

## Source compatibility

Existing source is not affected by these changes.

## Effect on ABI stability

None

## Effect on API resilience

None

## Alternatives considered

Previous discussions in the forums have been considered, specifically separating
the discriminator and value into individual key/value pairs discussed in [this forum thread](https://forums.swift.org/t/automatic-codable-conformance-for-enums-with-associated-values-that-themselves-conform-to-codable/11499).
While we do believe that there is value in doing this, we think that the default
behavior should more closely follow the structure of the types that are encoded.
A future proposal could add customization options to change the structure to meet
individual requirements.

### Parameterless cases

Alternative ways to encode these cases have been discussed and the following problems have been found:

1. Encode as plain values, as we do today with `RawRepresentable`

The problem with this is that `Decoder` does not have APIs to check which type of container is present and adding such APIs would be a breaking change. The existing APIs to read containers are `throws`, but it is not specified in which cases an error should occur. 

2. Encode as `nil`

This representation is problematic, because `Encoder` implementations can decide to drop `nil` values, which would also mean losing the key and with it the ability to identify the case.


## Acknowledgements

While iterating on this proposal, a lot of inspiration was drawn from the Rust library [serde](https://serde.rs/container-attrs.html).
