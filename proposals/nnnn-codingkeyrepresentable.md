# Allow coding of non- `String` / `Int` keyed `Dictionary` into a `KeyedContainer`

* Proposal: [SE-NNNN](NNNN-codingkeyrepresentable.md)
* Authors: [Morten Bek Ditlevsen](https://github.com/mortenbekditlevsen)
* Review Manager: TBD
* Status: **Awaiting review**

*During the review process, add the following fields as needed:*

* Implementation: [apple/swift#34458](https://github.com/apple/swift/pull/34458)
* Decision Notes: [Rationale](https://forums.swift.org/t/pitch-allow-coding-of-non-string-int-keyed-dictionary-into-a-keyedcontainer/44593)

## Introduction

The current conformance of Swift's `Dictionary` to the `Codable` protocols has a somewhat-surprising limitation in that dictionaries whose key type is not `String` or `Int` (values directly representable in `CodingKey` types) encode not as `KeyedContainer`s but as `UnkeyedContainer`s. This behavior has caused much confusion for users and I would like to offer a way to improve the situation.

Swift-evolution thread: [[Pitch] Allow coding of non-`String`/`Int` keyed `Dictionary` into a `KeyedContainer`](https://forums.swift.org/t/pitch-allow-coding-of-non-string-int-keyed-dictionary-into-a-keyedcontainer/44593)

## Motivation

The primary motivation for this pitch lays in the much-discussed confusion of this default behavior:

* [Dictionarys encoding strategy](https://forums.swift.org/t/dictionarys-encoding-strategy/11973)
* [JSON Encoding / Decoding weird encoding of dictionary with enum values](https://forums.swift.org/t/json-encoding-decoding-weird-encoding-of-dictionary-with-enum-values/12995)
* [Bug or PEBKAC](https://forums.swift.org/t/bug-or-pebkac/33796)
* [Using RawRepresentable String and Int keys for Codable Dictionaries](https://forums.swift.org/t/using-rawrepresentable-string-and-int-keys-for-codable-dictionaries/26899)

The common situations where people have found the behavior confusing include:

* Using `enum`s as keys (especially when `RawRepresentable`, and backed by `String` or `Int` types)
* Using `String` wrappers (like the generic [Tagged](https://github.com/pointfreeco/swift-tagged) library or custom wrappers) as keys
* Using `Int8` or other `Int*` flavours as keys

In the various discussions, there are clear and concise explanations for this behavior, but it is also mentioned that supporting encoding of `RawRepresentable` `String` and `Int` keys into keyed containers may indeed be considered to be a bug, and is an oversight in the implementation ([JSON Encoding / Decoding weird encoding of dictionary with enum values, reply by Itai Ferber](https://forums.swift.org/t/json-encoding-decoding-weird-encoding-of-dictionary-with-enum-values/12995/7)).

There's a bug at [bugs.swift.org](http://bugs.swift.org) tracking the issue: [SR-7788](https://bugs.swift.org/browse/SR-7788)

Unfortunately, it is too late to change the behavior now:

1. It is a breaking change with respect to existing behavior, with backwards-compatibility ramifications (new code couldn't decode old data and vice versa), and
2. The behavior is tied to the Swift stdlib, so the behavior would differ between consumers of the code and what OS versions they are on

Instead, I propose the addition of a new protocol to the standard library. Opting in to this protocol for the key type of a `Dictionary` will allow the `Dictionary` to encode/decode to/from a `KeyedContainer`.

## Proposed Solution

I propose adding a new protocol to the standard library: `CodingKeyRepresentable`

Types conforming to `CodingKeyRepresentable` indicate that they can be represented by a `CodingKey` instance (which they can offer), allowing them to opt in to having dictionaries use their `CodingKey` representations in order to encode into `KeyedContainer`s.

The opt-in can only happen for a version of Swift where the protocol is available, so the user will be in full control of the situation. For instance I am currently using my own workaround, but once I only support iOS versions running a specific future Swift version with this feature, I could skip my own workaround and rely on this behavior instead.

I have a draft PR for the proposed solution: [#34458](https://github.com/apple/swift/pull/34458)

## Examples

```swift
// Same as stdlib's _DictionaryCodingKey
struct _AnyCodingKey: CodingKey {
    let stringValue: String
    let intValue: Int?
    
    init?(stringValue: String) {
        self.stringValue = stringValue
        self.intValue = Int(stringValue)
    }
    
    init?(intValue: Int) {
        self.stringValue = "\(intValue)"
        self.intValue = intValue
    }
}

struct ID: Hashable, CodingKeyConvertible {
    static let knownID1 = ID(stringValue: "<some-identifier-1>")
    static let knownID2 = ID(stringValue: "<some-identifier-2>")
    
    let stringValue: String
    
    var codingKey: CodingKey {
        return _AnyCodingKey(stringValue: stringValue)
    }
    
    init?(codingKey: CodingKey) {
        stringValue = codingKey.stringValue
    }
    
    init(stringValue: String) {
        self.stringValue = stringValue
    }
}

let data: [ID: String] = [
    .knownID1: "...",
    .knownID2: "...",
]

let encoder = JSONEncoder()
try String(data: encoder.encode(data), encoding: .utf8)

/*
{
    "<some-identifier-1>": "...",
    "<some-identifier-2>": "...",
}
*/
```

## Detailed Design

The proposed solution adds a new protocol, `CodingKeyRepresentable`:

```swift
/// Indicates that the conforming type can provide a `CodingKey` to be used when
/// encoding into a keyed container.
public protocol CodingKeyRepresentable {
    var codingKey: CodingKey { get }
    init?(codingKey: CodingKey)
}
```

In the conditional `Encodable` conformance on `Dictionary`, the following extra case can handle such conforming types:

```swift
    } else if let _ = Key.self as? CodingKeyRepresentable.Type {
      // Since the keys are CodingKeyRepresentable, we can use the `codingKey`
      // to create `_DictionaryCodingKey` instances.
      var container = encoder.container(keyedBy: _DictionaryCodingKey.self)
      for (key, value) in self.dict {
        let codingKey = (key as! CodingKeyRepresentable).codingKey
        let dictionaryCodingKey = _DictionaryCodingKey(codingKey: codingKey)
        try container.encode(value, forKey: dictionaryCodingKey)
      }
    } else {
      // Keys are Encodable but not Strings or Ints, so we cannot arbitrarily

```

In the conditional `Decodable` conformance on `Dictionary`, we can similarly handle conforming types:

```swift
    } else if let codingKeyRepresentableType = Key.self as? CodingKeyRepresentable.Type {
      // The keys are CodingKeyRepresentable, so we should be able to expect a keyed container.
      let container = try decoder.container(keyedBy: _DictionaryCodingKey.self)
      for key in container.allKeys {
        let value = try container.decode(Value.self, forKey: key)
        let k = codingKeyRepresentableType.init(codingKey: key)
        self.dict[k as! Key] = value
      }
    } else {
      // We should have encoded as an array of alternating key-value pairs.
```

## Impact on Existing Code

No direct impact, since adoption of this protocol is additive.

However, special care must be taken in *adopting* the protocol, since adoption on any type `T` which has previously been encoded as a dictionary key can introduce backwards incompatibility with archives. It is always safe to adopt `CodingKeyConvertible` on new types, or types newly-conforming to `Codable`.

## Other Considerations

### Conforming stdlib types to `CodingKeyRepresentable`

Along the above lines, we do not propose conforming any existing stdlib or Foundation type to `CodingKeyRepresentable` due to backwards-compatibility concerns. Should end-user code require this conversion on existing types, we recommend writing wrapper types which conform on those types' behalf (for example, a `MyUUIDWrapper` which contains a `UUID` and conforms to `CodingKeyRepresentable` to allow using `UUID`s as dictionary keys directly).

### Adding an `AnyCodingKey` type to the standard library

Since types that conform to `CodingKeyRepresentable` will need to supply a `CodingKey`, most likely generated dynamically from type contents, this may be a good time to introduce a general key type which can take on any `String` or `Int` value it is initialized from.

`Dictionary` already uses exactly such a key type internally (`_DictionaryCodingKey`), as do `JSONEncoder` / `JSONDecoder` with `_JSONKey` (and `PropertyListEncoder` / `PropertyListDecoder` with `_PlistKey`), so generalization could be useful. The implementation of this type could match the implementation of `_AnyCodingKey` provided above.

## Alternatives Considered

### Why not just make the type conform to `CodingKey` directly?

For two reasons:

1. In the rare case in which a type already conforms to `CodingKey`, this runs the risk of behavior-breaking changes
2. `CodingKey` requires exposure of a `stringValue` and `intValue` property, which are only relevant when encoding and decoding; forcing types to expose these properties arbitrarily seems unreasonable

### Why not refine `RawRepresentable`, or use a `RawRepresentable where RawValue == CodingKey` constraint?

`RawRepresentable` conformance for types indicates a lossless conversion between the source type and its underlying `RawValue` type; this conversion is often the "canonical" conversion between a source type and its underlying representation, most commonly between `enum`s backed by raw values, and option sets similarly backed by raw values.

In contrast, we expect conversion to and from `CodingKey` to be *incidental* , and representative only of the encoding and decoding process. We wouldn't suggest (or expect) a type's canonical underlying representation to be a `CodingKey`, which is what a `protocol CodingKeyRepresentable: RawRepresentable where RawValue == CodingKey` would require. Similarly, types which are already `RawRepresentable` with non- `CodingKey` raw values couldn't adopt conformance this way, and a big impetus for this feature is allowing `Int`- and `String`-backed `enum`s to participate as dictionary coding keys.

### Why not use an Associated Type for `CodingKey`
It was suggested during the pitch phase to use an associated type for the `CodingKey` in the `CodingKeyRepresentable` protocol.

The presented use case was perfectly valid - and demonstrated using the following example:

```swift
enum MyKey: Int, CodingKey {
    case a = 1
    case b = 3
    case c = 5

    var intValue: Int? { rawValue }

    var stringValue: String {
        switch self {
        case .a: return "a"
        case .b: return "b"
        case .c: return "c"
        }
    }

    init?(intValue: Int) { self.init(rawValue: intValue) }

    init?(stringValue: String) {
        guard let rawValue = RawValue(stringValue) else { return nil }
        self.init(rawValue: rawValue)
    }
}

struct MyCustomType: CodingKeyRepresentable {
    typealias CodingKey = MyKey

    var useB = false

    var codingKey: CodingKey {
        useB ? .b : .a
    }

    init?(codingKey: CodingKey) {
        switch codingKey {
        case .a: useB = false
        case .b: useB = true
        case .c: return nil // .c is unsupported
        }
    }
}
```

An analysis of this suggestion hints that the non-zero cost of doing type erasure for pulling out the key values at the consuming site might not carry it's weight ([https://forums.swift.org/t/pitch-allow-coding-of-non-string-int-keyed-dictionary-into-a-keyedcontainer/44593/9](https://forums.swift.org/t/pitch-allow-coding-of-non-string-int-keyed-dictionary-into-a-keyedcontainer/44593/9)):

Because `associatedtype` s have non-zero cost on the consuming side (e.g. checking for `CodingKeyRepresentable` conformance, using the key type), I think that the associated type definition would need to carry its weight. Despite the name, I think that the key difference between `CodingKeyRepresentable` and `RawRepresentable` is that the *identity* of the `RawValue` type is crucial to `RawRepresentable` , but not so in the `CodingKeyRepresentable` case.

On the *consuming* side of `CodingKeyRepresentable.codingKey` (e.g. in `Dictionary` ), I don't believe key type identity is necessarily useful enough:

* The main use for the `.codingKey` value is immediate retrieval of the underlying `String` / `Int` values. `Dictionary` would either pull those values out for immediate use and throw away the original key
* In a non-generic context (or even one not predicated on `CodingKeyRepresentable` conformance), you can't meaningfully get at the key type. The type-erasure song and dance you have to do to get the key values won't be able to hand you a typed key (and the pain of doing that dance is that because it doesn't make sense to expose a public protocol for doing the erasure, every consumer that wants to do this needs to reinvent the wheel and add another protocol for doing it; we had to do it a few times for `Optional` s and it's a bit of a shame)
* Even if it were necessary to get a meaningful key type, the majority use-case for this feature, I believe, will be to provide dynamic-value keys for non-enumerable types (e.g. `struct` s like `UUID` [though yes, we can't make it conform]); for these types, you can't necessarily define a `CodingKey` s *`enum`* and instead, you'd likely want to use a more generic key type like `AnyCodingKey` (which by definition doesn't have identity)

On the *producing* side (e.g. in `MyCustomType` ), I'm also not sure the utility is necessarily enough: in general, the majority of `CodingKeyRepresentable` types (I believe) will only really care about the `String` / `Int` values of the keys, since they will be initialized dynamically (again, I think of `UUID` initialization from a `CodingKey.stringValue` — you can do this from *any* `CodingKey` ).

I believe that the constrained `MyKey` example above will be the minority use-case, but expressed without the `associatedtype` constraint too:

```swift
enum MyKey: Int, CodingKey {
    case a = 1, b = 3, c = 5

    // There are several ways to express this, just an example:
    init?(codingKey: CodingKey) {
        if let key = codingKey.intValue.flatMap(Self.init(intValue:)) {
            self = key
        } else if let key = Self(stringValue: codingKey.stringValue) {
            self = key
        } else {
            return nil
        }
    }
}

struct MyCustomType: CodingKeyRepresentable {
    var useB = false

    var codingKey: CodingKey {
        useB ? MyKey.b : MyKey.a
    }

    init?(codingKey: CodingKey) {
        switch MyKey(codingKey: codingKey) {
        case .a: useB = false
        case .b: useB = true
        default: return nil
        }    
    }
}
```

I personally find this equally as expressive, and I think that not requiring the associated type gives more flexibility without a significant loss, especially with non- `enum` types in mind.


### Add workarounds to each `Encoder` / `Decoder`

Following a suggestion from @itaiferber, I have previously tried to provide a solution to this issue — not in general, but instead solving it by providing a `DictionaryKeyEncodingStrategy` for `JSONEncoder` : [#26257](https://github.com/apple/swift/pull/26257)

The idea there was to be able to express an opt-in to the new behavior directly in the `JSONEncoder` and `JSONDecoder` types by venting a new encoding/decoding 'strategy' configuration. I have since changed my personal opinion about this and I believe that the problem should not just be fixed for specific `Encoder` / `Decoder` pairs, but rather for all.

The implementation of this was not very pretty, involving casts and iterations over the dictionaries to be encoded/decoded.

### Await design of `newtype`

I have heard mentions of a `newtype` design, that basically tries to solve the issue that the [Tagged](https://github.com/pointfreeco/swift-tagged) library solves: namely creating type safe wrappers around other primitive types.

I am in no way an expert in this, and I don't know how this would be implemented, but *if* it were possible to tell that `SomeType` is a `newtype` of `String`, then this could be used to provide a new implementation in the `Dictionary` `Codable` conformance, and since this feature does not exist in older versions of Swift (providing that this is a feature that requires changes to the Swift run-time), then adding this to the `Dictionary` `Codable` conformance would not be behavior breaking.

But those are an awful lot of ifs and buts, and it only solves one of the issues that people appear to run in to (the wrapping issue) — and not for instance `String` based enums or `Int8`-based keys.

### Do nothing

It is of course possible to handle this situation manually during encoding.

A rather unintrusive way of handling the situation is by using a property wrapper as suggested here: [CodableKey](https://forums.swift.org/t/bug-or-pebkac/33796/12).

This solution needs to be applied for each `Dictionary` and is a quite elegant workaround. But it is still a workaround for something that could be fixed in the stdlib.

A few drawbacks to the property wrapper solution were given during the pitch phase:

* Using `Int8` (or any other numeric stdlib type for that matter) as key requires it to conform to `CodingKey` . This conformance would have to come from the stdlib to prevent conformance collisions across e.g. Swift packages. And IMHO those types shouldn't provide a `CodingKey` conformance per se...
* It's not straightforward to simply encode/decode e.g. a `Dictionary<Int8, String>` that is not a property of another `Codable` type (also mentioned in the example in the linked post).
* It's impossible to *add* a `Codable` conformance to an object that is already defined. So if I define a struct ( `MyType` ) having a `Dictionary<Int8, String>` in one file, I can't simply put an `extension MyType: Codable { /* ... */ }` into another file.

## Acknowledgements
Many thanks to [Itai Ferber](https://github.com/itaiferber) for providing input and feedback, for revising the pitch and for helping me shape the overall direction.
