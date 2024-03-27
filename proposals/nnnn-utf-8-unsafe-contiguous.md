# UTF-8 Processing Over Unsafe Contiguous Bytes

* Proposal: [SE-NNNN](nnnn-utf-8-unsafe-contiguous.md)
* Authors: [Michael Ilseman](https://github.com/milseman)
* Review Manager: TBD
* Status: **Awaiting implementation**
* Implementation: (pending)
* Upcoming Feature Flag: (pending)
* Review: ([pitch](https://forums.swift.org/t/pitch-utf-8-processing-over-unsafe-contiguous-bytes/69715))

## Introduction and Motivation

Native `String`s are stored as validly-encoded UTF-8 bytes in a contiguous memory buffer. The standard library implements `String` functionality on top of this buffer, taking advantage of the validly-encoded invariant and specialized Unicode knowledge. We propose exposing this functionality as API for more advanced libraries and developers.

This pitch focuses on a portion of the broader API and functionality discussed in [Pitch: Unicode Processing APIs](https://forums.swift.org/t/pitch-unicode-processing-apis/69294). That broader pitch can be divided into 3 kinds of API additions:

1. Unicode processing API for working with contiguously-stored valid UTF-8 bytes
2. `Element`-based stream processing functionality. E.g., a stream of `UInt8` can be turned into a stream of `Unicode.Scalar` or `Character`s.
3. Stream-of-buffers processing functionality, which provides a lower-level / more efficient implementation for the second area.

This pitch focuses on the first.

## Proposed Solution

We propose `UnsafeValidUTF8BufferPointer` which exposes a similar API surface as `String` for validly-encoded UTF-8 code units in contiguous memory.


## Detailed Design

`UnsafeValidUTF8BufferPointer` consists of a (non-optional) raw pointer and a length, with some flags bit-packed in.

```swift
/// An unsafe buffer pointer to validly-encoded UTF-8 code units stored in
/// contiguous memory.
///
/// UTF-8 validity is checked upon creation.
///
/// `UnsafeValidUTF8BufferPointer` does not manage the memory or guarantee
/// memory safety. Any overlapping writes into the memory can lead to undefined 
/// behavior.
///
@frozen
public struct UnsafeValidUTF8BufferPointer {
  @usableFromInline
  internal var _baseAddress: UnsafeRawPointer

  // A bit-packed count and flags (such as isASCII)
  @usableFromInline
  internal var _countAndFlags: UInt64
}
```

It differs from `UnsafeRawBufferPointer` in that its contents, upon construction, are guaranteed to be validly-encoded UTF-8. This guarantee speeds up processing significantly relative to performing validation on every read. It is unsafe because it is an API surface on top of `UnsafeRawPointer`, inheriting all the unsafety therein and developers must manually guarantee invariants such as lifetimes and exclusivity. It is further based on `UnsafeRawPointer` instead of `UnsafePointer<UInt8>` so as not to [bind memory to a type](https://developer.apple.com/documentation/swift/unsaferawpointer#Typed-Memory).


### Validation and creation

`UnsafeValidUTF8BufferPointer` is validated at initialization time, and encoding errors are thrown.

```swift
extension Unicode.UTF8 {
  @frozen
  public enum EncodingErrorKind: Error {
    case unexpectedContinuationByte
    case expectedContinuationByte
    case overlongEncoding
    case invalidCodePoint

    case invalidStarterByte

    case unexpectedEndOfInput
  }
}
```

```swift
// All the initializers below are `throw`ing, as they validate the contents
// upon construction.
extension UnsafeValidUTF8BufferPointer {
  @frozen
  public struct DecodingError: Error, Sendable, Hashable, Codable {
    public var kind: UTF8.EncodingErrorKind
    public var offsets: Range<Int>
  }

  // ABI traffics in `Result`
  @usableFromInline
  internal static func _validate(
    baseAddress: UnsafeRawPointer, length: Int
  ) -> Result<UnsafeValidUTF8BufferPointer, DecodingError>

  @_alwaysEmitIntoClient
  public init(baseAddress: UnsafeRawPointer, length: Int) throws(DecodingError)

  @_alwaysEmitIntoClient
  public init(nulTerminatedCString: UnsafeRawPointer) throws(DecodingError)

  @_alwaysEmitIntoClient
  public init(nulTerminatedCString: UnsafePointer<CChar>) throws(DecodingError)

  @_alwaysEmitIntoClient
  public init(_: UnsafeRawBufferPointer) throws(DecodingError)

  @_alwaysEmitIntoClient
  public init(_: UnsafeBufferPointer<UInt8>) throws(DecodingError)
}
```

#### Unsafety and encoding validity

Every way to construct a `UnsafeValidUTF8BufferPointer` ensures that its contents are validly-encoded UTF-8. Thus, it has no new source of unsafety beyond the unsafety inherent in unsafe pointer's requirement that lifetime and exclusive access be manually enforced by the programmer. A write into this memory which violates encoding validity would also violate exclusivity.

If we did not guarantee UTF-8 encoding validity, we'd be open to new security and safety concerns beyond unsafe pointers.

With invalidly-encoded contents, memory safety would become more nuanced. An ill-formed leading byte can dictate a scalar length that is longer than the memory buffer. The buffer may have bounds associated with it, which differs from the bounds dictated by its contents.

Additionally, a particular scalar value in valid UTF-8 has only one encoding, but invalid UTF-8 could have the same value encoded as an [overlong encoding](https://en.wikipedia.org/wiki/UTF-8#Overlong_encodings), which would compromise code that checks for the presence of a scalar value by looking at the encoded bytes (or that does a byte-wise comparison).

`UnsafeValidUTF8BufferPointer` is unsafe in the all ways that unsafe pointers are unsafe, but not in more ways.


### Accessing contents

Flags and raw contents can be accessed:

```swift
extension UnsafeValidUTF8BufferPointer {
  /// Returns whether the validated contents were all-ASCII. This is checked at
  /// initialization time and remembered.
  @inlinable
  public var isASCII: Bool

  /// Access the underlying raw bytes
  @inlinable
  public var rawBytes: UnsafeRawBufferPointer
}
```

Like `String`, `UnsafeValidUTF8BufferPointer` provides views for accessing `Unicode.Scalar`s, `UTF16.CodeUnit`s, and `Character`s.

```swift
extension UnsafeValidUTF8BufferPointer {
  /// A view of the buffer's contents as a bidirectional collection of `Unicode.Scalar`s.
  @frozen
  public struct UnicodeScalarView {
    public var buffer: UnsafeValidUTF8BufferPointer

    @inlinable
    public init(_ buffer: UnsafeValidUTF8BufferPointer)
  }

  @inlinable
  public var unicodeScalars: UnicodeScalarView

  /// A view of the buffer's contents as a bidirectional collection of `Character`s.
  @frozen
  public struct CharacterView {
    public var buffer: UnsafeValidUTF8BufferPointer

    @inlinable
    public init(_ buffer: UnsafeValidUTF8BufferPointer)
  }

  @inlinable
  public var characters: CharacterView

  /// A view off the buffer's contents as a bidirectional collection of transcoded
  /// `UTF16.CodeUnit`s.
  @frozen
  public struct UTF16View {
    public var buffer: UnsafeValidUTF8BufferPointer

    @inlinable
    public init(_ buffer: UnsafeValidUTF8BufferPointer)
  }

  @inlinable
  public var utf16: UTF16View
}
```

These are bidirectional collections, as in `String`. Their indices, however, are distinct from each other because they mean different things. For example, a scalar-view index is scalar aligned but not necessarily `Character` aligned, and a transcoded index which points mid-scalar doesn't have a corresponding position in the raw bytes.

```swift
extension UnsafeValidUTF8BufferPointer.UnicodeScalarView: BidirectionalCollection {
  public typealias Element = Unicode.Scalar

  @frozen
  public struct Index: Comparable, Hashable {
    @usableFromInline
    internal var _byteOffset: Int

    @inlinable
    public var byteOffset: Int { get }

    @inlinable
    public static func < (lhs: Self, rhs: Self) -> Bool

    @inlinable
    internal init(_uncheckedByteOffset offset: Int)
  }

  @inlinable
  public subscript(position: Index) -> Element { _read }

  @inlinable
  public func index(after i: Index) -> Index

  @inlinable
  public func index(before i: Index) -> Index

  @inlinable
  public var startIndex: Index

  @inlinable
  public var endIndex: Index
}


extension UnsafeValidUTF8BufferPointer.CharacterView: BidirectionalCollection {
  public typealias Element = Character

  @frozen
  public struct Index: Comparable, Hashable {
    @usableFromInline
    internal var _byteOffset: Int

    @inlinable
    public var byteOffset: Int { get }

    @inlinable
    public static func < (lhs: Self, rhs: Self) -> Bool

    @inlinable
    internal init(_uncheckedByteOffset offset: Int)
  }

  // Custom-defined for performance to avoid double-measuring
  // grapheme cluster length
  @frozen
  public struct Iterator: IteratorProtocol {
    @usableFromInline
    internal var _buffer: UnsafeValidUTF8BufferPointer

    @usableFromInline
    internal var _position: Index

    @inlinable
    public var buffer: UnsafeValidUTF8BufferPointer { get }

    @inlinable
    public var position: Index { get }

    public typealias Element = Character

    public mutating func next() -> Character?

    @inlinable
    internal init(
      _buffer: UnsafeValidUTF8BufferPointer, _position: Index
    )
  }

  @inlinable
  public func makeIterator() -> Iterator

  @inlinable
  public subscript(position: Index) -> Element { _read }

  @inlinable
  public func index(after i: Index) -> Index

  @inlinable
  public func index(before i: Index) -> Index

  @inlinable
  public var startIndex: Index

  @inlinable
  public var endIndex: Index
}

extension UnsafeValidUTF8BufferPointer.UTF16View: BidirectionalCollection {
  public typealias Element = Unicode.Scalar

  @frozen
  public struct Index: Comparable, Hashable {
    // Bitpacked byte offset and transcoded offset
    @usableFromInline
    internal var _byteOffsetAndTranscodedOffset: UInt64

    /// Offset of the first byte of the currently-indexed scalar
    @inlinable
    public var byteOffset: Int { get }

    /// Offset of the transcoded code unit within the currently-indexed scalar
    @inlinable
    public var transcodedOffset: Int { get }

    @inlinable
    public static func < (lhs: Self, rhs: Self) -> Bool

    @inlinable
    internal init(
      _uncheckedByteOffset offset: Int, _transcodedOffset: Int
    )
  }

  @inlinable
  public subscript(position: Index) -> Element { _read }

  @inlinable
  public func index(after i: Index) -> Index

  @inlinable
  public func index(before i: Index) -> Index

  @inlinable
  public var startIndex: Index

  @inlinable
  public var endIndex: Index
}
```

### Canonical equivalence

```swift
// Canonical equivalence
extension UnsafeValidUTF8BufferPointer {
  /// Whether `self` is equivalent to `other` under Unicode Canonical Equivalance
  public func isCanonicallyEquivalent(
    to other: UnsafeValidUTF8BufferPointer
  ) -> Bool

  /// Whether `self` orders less than `other` (under Unicode Canonical Equivalance
  /// using normalized code-unit order)
  public func isCanonicallyLessThan(
    _ other: UnsafeValidUTF8BufferPointer
  ) -> Bool
}
```



## Alternatives Considered

### Other names

We're not particularly attached to the name `UnsafeValidUTF8BufferPointer`. Other names could include:

- `UnsafeValidUTF8CodeUnitBufferPointer`
- `UTF8.UnsafeValidBufferPointer`
- `UTF8.UnsafeValidCodeUnitBufferPointer`
- `UTF8.ValidlyEncodedCodeUnitUnsafeBufferPointer`
- `UnsafeContiguouslyStoredValidUTF8CodeUnitsBuffer`

etc.

For `isCanonicallyLessThan`, another name could be `canonicallyPrecedes`, `lexicographicallyPrecedesUnderNFC`, etc.

### Static methods instead of initializers

`UnsafeValidUTF8BufferPointer`s could instead be created by static methods on `UTF8`:

```swift
extension Unicode.UTF8 {
  static func validate(
    ...
  ) throws -> UnsafeValidUTF8BufferPointer
}
```

### Hashable and other conformances

`UnsafeValidUTF8BufferPointer` follows `UnsafeRawBufferPointer` and `UnsafeBufferPointer` in not conforming to `Sendable`, `Hashable`, `Equatable`, `Comparable`, `Codable`, etc.

### `UTF8.EncodingErrorKind` as a `struct`

We may want to use the [raw-representable struct pattern](https://github.com/apple/swift-system/blob/9a812b5fef1e7f27f8594fee5463bd88c5b691ec/Sources/System/Errno.swift#L14) for `UTF8.EncodingErrorKind` instead of an exhaustive enum. That is, we may want to define it as:

```swift
extension Unicode.UTF8 {
  @frozen
  public struct EncodingErrorKind: Error, Sendable, Hashable, Codable {
    public var rawValue: UInt8

    @inlinable
    public init(rawValue: UInt8) {
      self.rawValue = rawValue
    }

    @inlinable
    public static var unexpectedContinuationByte: Self {
      .init(rawValue: 0x01)
    }

    @inlinable
    public static var overlongEncoding: Self {
      .init(rawValue: 0x02)
    }

    // ...
  }
}
```

This would allow us to grow the kinds or errors or else add some error-nuance to the future, at the loss of exhaustive switches inside `catch`es.

For example, an unexpected-end-of-input error, which happens when a scalar is in the process of being decoded but not enough bytes have been read, could be reported in different ways. It could be reported as a distinct kind of error (particularly useful for stream processing which may want to resume with more content) or it could be a `expectedContinuationByte` covering the end-of-input position. As a value, it could have a distinct value or be an alias to the same value.




## Future Directions

### A non-escapable `ValidUTF8BufferView`

Future improvements to Swift enable a non-escapable type (["BufferView"](https://github.com/atrick/swift-evolution/blob/fd63292839808423a5062499f588f557000c5d15/visions/language-support-for-BufferView.md)) to provide safely-unmanaged buffers via dependent lifetimes for use within a limited scope. We should add a corresponding type for validly-encoded UTF-8 contents, following the same API shape.


### Shared-ownership buffer

We could propose a managed or shared-ownership validly-encoded UTF-8 buffer. E.g.:

```swift
struct ValidlyEncodedUTF8SharedBuffer {
  var contents: UnsafeValidlyEncodedUTF8BufferPointer
  var owner: AnyObject?
}
```

where "shared" denotes that ownership is shared with the `owner` field, as opposed to an allocation exclusively managed by this type (the way `Array` or `String` would). Thus, it could be backed by a native `String`, an instance of `Data` or `Array<UInt8>` (if ensured to be validly encoded), etc., which participate fully in their COW semantics by retaining their storage.

This would enable us to create shared strings, e.g.

```swift
extension String {
  /// Does not copy the given storage, rather shares it
  init(sharing: ValidlyEncodedUTF8SharedBuffer)
}
```

Also, this could allow us to present API which repairs invalid contents, since a repair operation would need to create and manage its own allocation.


#### Alternative: More general formulation (💥🐮)

We could add the more general ["deconstructed COW"](https://forums.swift.org/t/idea-bytes-literal/44124/50)

```swift
/// A buffer of `T`s in contiguous memory
struct SharedContiguousStorage<T> {
  var rawContents: UnsafeRawBufferPointer
  var owner: AnyObject?
}
```

where the choice of `Raw` pointers is necessary to avoid type-binding the memory, but other designs are possible too. 

However, this type alone loses static knowledge of the UTF-8 validity, so we'd still need a separate type for validly encoded UTF-8.

Instead, we could parameterize over a unsafe-buffer-pointer-like protocol:

```swift
struct SharedContiguousStorage<UnsafeBuffer: UnsafeBufferPointerProtocol> {
  var contents: UnsafeBuffer
  var owner: AnyObject?    
}

extension String {
  /// Does not copy the given storage, rather shares it
  init(sharing: SharedContiguousStorage<UnsafeValidUTF8BufferPointer>)
}
```

Accessing the stored pointer would still need to be done carefully, as it would have lifetime dependent on `owner`. In current Swift, that would likely need to be done via a closure-taking API.


### `protocol ContiguouslyStoredValidUTF8`

We could define a protocol for validly-encoded UTF-8 bytes in contiguous memory, somewhat analogous to a low-level `StringProtocol`. Both an unsafe and a shared-ownership type could conform to provide the same API.

However, we'd want to be careful to future-proof such a protocol so that a  `ValidUTF8BufferView` could conform as well. In the mean-time, even if we go with adding a shared-ownership type, Unicode processing operations can be performed by accessing the unsafe buffer pointer.

### Extend to `Element`-based or buffer-based streams

We could define a segment of validly encoded UTF-8, which is not necessarily aligned across any particular boundary. This would be a significantly different API shape than `String`'s views. Accessing the start of content would require passing in initial state and reaching the end would produce a state to be fed into the next segment. 

It would make an awkward fit directly on top of `Collection`, so this would be a new API shape. For example, it could be akin to a `StatefulCollection` that in addition to having `startIndex/endIndex` would have `startState/endState`. Concerns such as bidirectionality, where exactly `endIndex` points to (the start or end of the partial value at the tail), etc, requires further thought.

### Regex or regex-like support

Future API additions would be to support `Regex`es on such buffers. 

Another future direction could be to add many routines corresponding to the underlying operations performed by the regex engine, such as:

```swift
extension UnsafeValidUTF8BufferPointer.CharacterView {
  func matchCharacterClass(
    _: CharacterClass,
    startingAt: Index,
    limitedBy: Index    
  ) throws -> Index?

  func matchQuantifiedCharacterClass(
    _: CharacterClass,
    _: QuantificationDescription,
    startingAt: Index,
    limitedBy: Index    
  ) throws -> Index?
}
```

which would be useful for parser-combinator libraries who wish to expose `String`'s model of Unicode by using the stdlib's accelerated implementation.

### Transcoded views, normalized views, case-folded views, etc

We could provide lazily transcoded, normalized, case-folded, etc., views. If we do any of these for `UnsafeValidUTF8BufferPointer`, we should consider adding equivalents on `String`, `Substring`, etc. If we were to make any new protocols or changes to protocols, we'd want to also future-proof for a `ValidUTF8BufferView`.

For example, transcoded views can be generalized:

```swift
extension UnsafeValidUTF8BufferPointer {
  /// A view off the buffer's contents as a bidirectional collection of transcoded
  /// `Encoding.CodeUnit`s.
  @frozen
  public struct TranscodedView<Encoding: _UnicodeEncoding> {
    public var buffer: UnsafeValidUTF8BufferPointer

    @inlinable
    public init(_ buffer: UnsafeValidUTF8BufferPointer)
  }
}
```

Note that since UTF-16 has such historical significance that even with a fully-generic transcoded view, we'd likely want a dedicated, specialized type for UTF-16.

We could similarly provide lazily-normalized views of code units or scalars under NFC or NFD (which the stdlib already distributes data tables for), possibly generic via a protocol for 3rd party normal forms.

Finally, case-folded functionality can be accessed in today's Swift via [scalar properties](https://developer.apple.com/documentation/swift/unicode/scalar/properties-swift.struct), but we could provide convenience collections ourselves as well.


### UTF-8 to/from UTF-16 breadcrumbs API

String's implementation caches distances between UTF-8 and UTF-16 views, as some imported Cocoa APIs use random access to the UTF-16 view. We could formalize and expose API for this.


### `NUL`-termination concerns and C bridging

`UnsafeValidUTF8BufferPointer` is capable of housing interior `NUL` characters, just like `String`. We could add additional flags and initialization options to detect a trailing `NUL` byte beyond the count and treat it as a terminator. In those cases, we could provide a `withCStringIfAvailable` style API.

### Index rounding operations

Unlike String, `UnsafeValidUTF8BufferPointer`'s view's `Index` types are distinct, which avoids a [mess of problems](https://forums.swift.org/t/string-index-unification-vs-bidirectionalcollection-requirements/55946). Interesting additions to both `UnsafeValidUTF8BufferPointer` and `String` would be explicit index-rounding for a desired behavior.


### Canonical Spaceships

Should a `ComparisonResult` (or [spaceship](https://forums.swift.org/t/pitch-comparison-reform/5662)) be added to Swift, we could support that operation under canonical equivalence in a single pass rather than subsequent calls to `isCanonicallyEquivalent(to:)` and `isCanonicallyLessThan(_:)`.


### Other Unicode functionality

For the purposes of this pitch, we're not looking to expand the scope of functionality beyond what the stdlib already does in support of `String`'s API. Other functionality can be considered future work.
