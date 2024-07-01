# Safe UTF-8 Processing Over Contiguous Bytes

* Proposal: [SE-NNNN](nnnn-utf8-span.md)
* Authors: [Michael Ilseman](https://github.com/milseman), [Guillaume Lessard](https://github.com/glessard)
* Review Manager: TBD
* Status: **Awaiting implementation**
* Bug: rdar://48132971, rdar://96837923
* Implementation: [Prototype](https://github.com/apple/swift-collections/pull/394)
* Upcoming Feature Flag: (pending)
* Review: ([pitch 1](https://forums.swift.org/t/pitch-utf-8-processing-over-unsafe-contiguous-bytes/69715)) ([pitch 2](https://forums.swift.org/t/pitch-safe-utf-8-processing-over-contiguous-bytes/72742))


## Introduction

We introduce `UTF8Span` for efficient and safe Unicode processing over contiguous storage. `UTF8Span` is a memory safe non-escapable type [similar to `Span`](https://github.com/swiftlang/swift-evolution/pull/2307).

Native `String`s are stored as validly-encoded UTF-8 bytes in an internal contiguous memory buffer. The standard library implements `String`'s API as internal methods which operate on top of this buffer, taking advantage of the validly-encoded invariant and specialized Unicode knowledge. We propose making this UTF-8 buffer and its methods public as API for more advanced libraries and developers.

## Motivation

Currently, if a developer wants to do `String`-like processing over UTF-8 bytes, they have to make an instance of `String`, which allocates a native storage class, copies all the bytes, and is reference counted. The developer would then need to operate within the new `String`'s views and map between `String.Index` and byte offsets in the original buffer.

For example, if these bytes were part of a data structure, the developer would need to decide to either cache such a new `String` instance or recreate it on the fly. Caching more than doubles the size and adds caching complexity. Recreating it on the fly adds a linear time factor and class instance allocation/deallocation and potentially reference counting.

Furthermore, `String` may not be available on all embedded platforms due to the fact that it's conformance to `Comparable` and `Collection` depend on data tables bundled with the stdlib. `UTF8Span` is a more appropriate type for these platforms, and only some explicit API make use of data tables.

### UTF-8 validity and efficiency

UTF-8 validation is particularly common concern and the subject of a fair amount of [research](https://lemire.me/blog/2020/10/20/ridiculously-fast-unicode-utf-8-validation/). Once an input is known to be validly encoded UTF-8, subsequent operations such as decoding, grapheme breaking, comparison, etc., can be implemented much more efficiently under this assumption of validity. Swift's `String` type's native storage is guaranteed-valid-UTF8 for this reason.

Failure to guarantee UTF-8 encoding validity creates security and safety concerns. With invalidly-encoded contents, memory safety would become more nuanced. An ill-formed leading byte can dictate a scalar length that is longer than the memory buffer. The buffer may have bounds associated with it, which differs from the bounds dictated by its contents.

Additionally, a particular scalar value in valid UTF-8 has only one encoding, but invalid UTF-8 could have the same value encoded as an [overlong encoding](https://en.wikipedia.org/wiki/UTF-8#Overlong_encodings), which would compromise code that checks for the presence of a scalar value by looking at the encoded bytes (or that does a byte-wise comparison).


## Proposed solution

We propose a non-escapable `UTF8Span` which exposes a similar API surface as `String` for validly-encoded UTF-8 code units in contiguous memory. We also propose rich API describing the kind and location of encoding errors.

## Detailed design

`UTF8Span` is a borrowed view into contiguous memory containing validly-encoded UTF-8 code units.

```swift
@frozen
public struct UTF8Span: Copyable, ~Escapable {
  public var unsafeBaseAddress: UnsafeRawPointer

  /*
   A bit-packed count and flags (such as isASCII)

   ╔═══════╦═════╦═════╦══════════╦═══════╗
   ║  b63  ║ b62 ║ b61 ║  b60:56  ║ b56:0 ║
   ╠═══════╬═════╬═════╬══════════╬═══════╣
   ║ ASCII ║ NFC ║ SSC ║ reserved ║ count ║
   ╚═══════╩═════╩═════╩══════════╩═══════╝

   ASCII means the contents are all-ASCII (<0x7F).
   NFC means contents are in normal form C for fast comparisons.
   SSC means single-scalar Characters (i.e. grapheme clusters): every
     `Character` holds only a single `Unicode.Scalar`.
   */
  @usableFromInline
  internal var _countAndFlags: UInt64

  @inlinable @inline(__always)
  init<Owner: ~Copyable & ~Escapable>(
    _unsafeAssumingValidUTF8 start: UnsafeRawPointer,
    _countAndFlags: UInt64,
    owner: borrowing Owner
  ) -> dependsOn(owner) Self { }
}

```

### Creation and validation

`UTF8Span` is validated at initialization time, and encoding errors are diagnosed and thrown.

```swift
extension Unicode.UTF8 {
  /**

   The kind and location of a UTF-8 encoding error.

   Valid UTF-8 is represented by this table:

   ╔════════════════════╦════════╦════════╦════════╦════════╗
   ║    Scalar value    ║ Byte 0 ║ Byte 1 ║ Byte 2 ║ Byte 3 ║
   ╠════════════════════╬════════╬════════╬════════╬════════╣
   ║ U+0000..U+007F     ║ 00..7F ║        ║        ║        ║
   ║ U+0080..U+07FF     ║ C2..DF ║ 80..BF ║        ║        ║
   ║ U+0800..U+0FFF     ║ E0     ║ A0..BF ║ 80..BF ║        ║
   ║ U+1000..U+CFFF     ║ E1..EC ║ 80..BF ║ 80..BF ║        ║
   ║ U+D000..U+D7FF     ║ ED     ║ 80..9F ║ 80..BF ║        ║
   ║ U+E000..U+FFFF     ║ EE..EF ║ 80..BF ║ 80..BF ║        ║
   ║ U+10000..U+3FFFF   ║ F0     ║ 90..BF ║ 80..BF ║ 80..BF ║
   ║ U+40000..U+FFFFF   ║ F1..F3 ║ 80..BF ║ 80..BF ║ 80..BF ║
   ║ U+100000..U+10FFFF ║ F4     ║ 80..8F ║ 80..BF ║ 80..BF ║
   ╚════════════════════╩════════╩════════╩════════╩════════╝

   ### Classifying errors

   An *unexpected continuation* is when a continuation byte (`10xxxxxx`) occurs
   in a position that should be the start of a new scalar value. Unexpected
   continuations can often occur when the input contains arbitrary data
   instead of textual content. An unexpected continuation at the start of
   input might mean that the input was not correctly sliced along scalar
   boundaries or that it does not contain UTF-8.

   A *truncated scalar* is a multi-byte sequence that is the start of a valid
   multi-byte scalar but is cut off before ending correctly. A truncated
   scalar at the end of the input might mean that only part of the entire
   input was received.

   A *surrogate code point* (`U+D800..U+DFFF`) is invalid UTF-8. Surrogate
   code points are used by UTF-16 to encode scalars in the supplementary
   planes. Their presence may mean the input was encoded in a different 8-bit
   encoding, such as CESU-8, WTF-8, or Java's Modified UTF-8.

   An *invalid non-surrogate code point* is any code point higher than
   `U+10FFFF`. This can often occur when the input is arbitrary data instead
   of textual content.

   An *overlong encoding* occurs when a scalar value that could have been
   encoded using fewer bytes is encoded in a longer byte sequence. Overlong
   encodings are invalid UTF-8 and can lead to security issues if not
   correctly detected:

   - https://nvd.nist.gov/vuln/detail/CVE-2008-2938
   - https://nvd.nist.gov/vuln/detail/CVE-2000-0884

   An overlong encoding of `NUL`, `0xC0 0x80`, is used in Java's Modified
   UTF-8 but is invalid UTF-8. Overlong encoding errors often catch attempts
   to bypass security measures.

   ### Reporting the range of the error

   The range of the error reported follows the *Maximal subpart of an
   ill-formed subsequence* algorithm in which each error is either one byte
   long or ends before the first byte that is disallowed. See "U+FFFD
   Substitution of Maximal Subparts" in the Unicode Standard. Unicode started
   recommending this algorithm in version 6 and is adopted by the W3C.

   The maximal subpart algorithm will produce a single multi-byte range for a
   truncated scalar (a multi-byte sequence that is the start of a valid
   multi-byte scalar but is cut off before ending correctly). For all other
   errors (including overlong encodings, surrogates, and invalid code
   points), it will produce an error per byte.

   Since overlong encodings, surrogates, and invalid code points are erroneous
   by the second byte (at the latest), the above definition produces the same
   ranges as defining such a sequence as a truncated scalar error followed by
   unexpected continuation byte errors. The more semantically-rich
   classification is reported.

   For example, a surrogate count point sequence `ED A0 80` will be reported
   as three `.surrogateCodePointByte` errors rather than a `.truncatedScalar`
   followed by two `.unexpectedContinuationByte` errors.

   Other commonly reported error ranges can be constructed from this result.
   For example, PEP 383's error-per-byte can be constructed by mapping over
   the reported range. Similarly, constructing a single error for the longest
   invalid byte range can be constructed by joining adjacent error ranges.

   ╔═════════════════╦══════╦═════╦═════╦═════╦═════╦═════╦═════╦══════╗
   ║                 ║  61  ║ F1  ║ 80  ║ 80  ║ E1  ║ 80  ║ C2  ║  62  ║
   ╠═════════════════╬══════╬═════╬═════╬═════╬═════╬═════╬═════╬══════╣
   ║ Longest range   ║ U+61 ║ err ║     ║     ║     ║     ║     ║ U+62 ║
   ║ Maximal subpart ║ U+61 ║ err ║     ║     ║ err ║     ║ err ║ U+62 ║
   ║ Error per byte  ║ U+61 ║ err ║ err ║ err ║ err ║ err ║ err ║ U+62 ║
   ╚═════════════════╩══════╩═════╩═════╩═════╩═════╩═════╩═════╩══════╝

   */
  @frozen
  public struct EncodingError: Error, Sendable, Hashable, Codable {
    /// The kind of encoding error
    public var kind: Unicode.UTF8.EncodingError.Kind

    /// The range of offsets into our input containing the error
    public var range: Range<Int>

    @_alwaysEmitIntoClient
    public init(
      _ kind: Unicode.UTF8.EncodingError.Kind,
      _ range: some RangeExpression<Int>
    )

    @_alwaysEmitIntoClient
    public init(_ kind: Unicode.UTF8.EncodingError.Kind, at: Int)
  }
}

extension UTF8.EncodingError {
  /// The kind of encoding error encountered during validation
  @frozen
  public struct Kind: Error, Sendable, Hashable, Codable, RawRepresentable {
    public var rawValue: UInt8

    @inlinable
    public init(rawValue: UInt8)

    /// A continuation byte (`10xxxxxx`) outside of a multi-byte sequence
    @_alwaysEmitIntoClient
    public static var unexpectedContinuationByte: Self

    /// A byte in a surrogate code point (`U+D800..U+DFFF`) sequence
    @_alwaysEmitIntoClient
    public static var surrogateCodePointByte: Self

    /// A byte in an invalid, non-surrogate code point (`>U+10FFFF`) sequence
    @_alwaysEmitIntoClient
    public static var invalidNonSurrogateCodePointByte: Self

    /// A byte in an overlong encoding sequence
    @_alwaysEmitIntoClient
    public static var overlongEncodingByte: Self

    /// A multi-byte sequence that is the start of a valid multi-byte scalar
    /// but is cut off before ending correctly
    @_alwaysEmitIntoClient
    public static var truncatedScalar: Self
  }
}

@_unavailableInEmbedded
extension UTF8.EncodingError.Kind: CustomStringConvertible {
  public var description: String { get }
}

@_unavailableInEmbedded
extension UTF8.EncodingError: CustomStringConvertible {
  public var description: String { get }
}

extension UTF8Span {
  public init(
    validating codeUnits: Span<UInt8>
  ) throws(EncodingError) -> dependsOn(codeUnits) Self
}
```

### Basic operations

#### Core Scalar API

```swift
extension UTF8Span {
  /// Whether `i` is on a boundary between Unicode scalar values.
  @_alwaysEmitIntoClient
  public func isScalarAligned(_ i: Int) -> Bool

  /// Whether `i` is on a boundary between Unicode scalar values.
  ///
  /// This function does not validate that `i` is within the span's bounds;
  /// this is an unsafe operation.
  @_alwaysEmitIntoClient
  public func isScalarAligned(unchecked i: Int) -> Bool

  /// Whether `range`'s bounds are aligned to `Unicode.Scalar` boundaries.
  @_alwaysEmitIntoClient
  public func isScalarAligned(_ range: Range<Int>) -> Bool

  /// Whether `range`'s bounds are aligned to `Unicode.Scalar` boundaries.
  ///
  /// This function does not validate that `range` is within the span's bounds;
  /// this is an unsafe operation.
  @_alwaysEmitIntoClient
  public func isScalarAligned(unchecked range: Range<Int>) -> Bool

  /// Returns the start of the next `Unicode.Scalar` after the one starting at
  /// `i`, or the end of the span if `i` denotes the final scalar.
  ///
  /// `i` must be scalar-aligned.
  @_alwaysEmitIntoClient
  public func nextScalarStart(_ i: Int) -> Int

  /// Returns the start of the next `Unicode.Scalar` after the one starting at
  /// `i`, or the end of the span if `i` denotes the final scalar.
  ///
  /// `i` must be scalar-aligned.
  ///
  /// This function does not validate that `i` is within the span's bounds;
  /// this is an unsafe operation.
  @_alwaysEmitIntoClient
  public func nextScalarStart(unchecked i: Int) -> Int

  /// Returns the start of the next `Unicode.Scalar` after the one starting at
  /// `i`, or the end of the span if `i` denotes the final scalar.
  ///
  /// `i` must be scalar-aligned.
  ///
  /// This function does not validate that `i` is within the span's bounds;
  /// this is an unsafe operation.
  ///
  /// This function does not validate that `i` is scalar-aligned; this is an
  /// unsafe operation if `i` isn't.
  @_alwaysEmitIntoClient
  public func nextScalarStart(
    uncheckedAssumingAligned i: Int
  ) -> Int

  /// Returns the start of the `Unicode.Scalar` ending at `i`, i.e. the scalar
  /// before the one starting at `i` or the last scalar if `i` is the end of
  /// the span.
  ///
  /// `i` must be scalar-aligned.
  @_alwaysEmitIntoClient
  public func previousScalarStart(_ i: Int) -> Int

  /// Returns the start of the `Unicode.Scalar` ending at `i`, i.e. the scalar
  /// before the one starting at `i` or the last scalar if `i` is the end of
  /// the span.
  ///
  /// `i` must be scalar-aligned.
  ///
  /// This function does not validate that `i` is within the span's bounds;
  /// this is an unsafe operation.
  @_alwaysEmitIntoClient
  public func previousScalarStart(unchecked i: Int) -> Int

  /// Returns the start of the `Unicode.Scalar` ending at `i`, i.e. the scalar
  /// before the one starting at `i` or the last scalar if `i` is the end of
  /// the span.
  ///
  /// `i` must be scalar-aligned.
  ///
  /// This function does not validate that `i` is within the span's bounds;
  /// this is an unsafe operation.
  ///
  ///
  /// This function does not validate that `i` is scalar-aligned; this is an
  /// unsafe operation if `i` isn't.
  @_alwaysEmitIntoClient
  public func previousScalarStart(
    uncheckedAssumingAligned i: Int
  ) -> Int

  /// Decode the `Unicode.Scalar` starting at `i`. Return it and the start of
  /// the next scalar.
  ///
  /// `i` must be scalar-aligned.
  @_alwaysEmitIntoClient
  public func decodeNextScalar(
    _ i: Int
  ) -> (Unicode.Scalar, nextScalarStart: Int)

  /// Decode the `Unicode.Scalar` starting at `i`. Return it and the start of
  /// the next scalar.
  ///
  /// `i` must be scalar-aligned.
  ///
  /// This function does not validate that `i` is within the span's bounds;
  /// this is an unsafe operation.
  @_alwaysEmitIntoClient
  public func decodeNextScalar(
    unchecked i: Int
  ) -> (Unicode.Scalar, nextScalarStart: Int)

  /// Decode the `Unicode.Scalar` starting at `i`. Return it and the start of
  /// the next scalar.
  ///
  /// `i` must be scalar-aligned.
  ///
  /// This function does not validate that `i` is within the span's bounds;
  /// this is an unsafe operation.
  ///
  ///
  /// This function does not validate that `i` is scalar-aligned; this is an
  /// unsafe operation if `i` isn't.
  @_alwaysEmitIntoClient
  public func decodeNextScalar(
    uncheckedAssumingAligned i: Int
  ) -> (Unicode.Scalar, nextScalarStart: Int)

  /// Decode the `Unicode.Scalar` ending at `i`, i.e. the previous scalar.
  /// Return it and the start of that scalar.
  ///
  /// `i` must be scalar-aligned.
  @_alwaysEmitIntoClient
  public func decodePreviousScalar(
    _ i: Int
  ) -> (Unicode.Scalar, previousScalarStart: Int)

  /// Decode the `Unicode.Scalar` ending at `i`, i.e. the previous scalar.
  /// Return it and the start of that scalar.
  ///
  /// `i` must be scalar-aligned.
  ///
  /// This function does not validate that `i` is within the span's bounds;
  /// this is an unsafe operation.
  @_alwaysEmitIntoClient
  public func decodePreviousScalar(
    unchecked i: Int
  ) -> (Unicode.Scalar, previousScalarStart: Int)

  /// Decode the `Unicode.Scalar` ending at `i`, i.e. the previous scalar.
  /// Return it and the start of that scalar.
  ///
  /// `i` must be scalar-aligned.
  ///
  /// This function does not validate that `i` is within the span's bounds;
  /// this is an unsafe operation.
  ///
  ///
  /// This function does not validate that `i` is scalar-aligned; this is an
  /// unsafe operation if `i` isn't.
  @_alwaysEmitIntoClient
  public func decodePreviousScalar(
    uncheckedAssumingAligned i: Int
  ) -> (Unicode.Scalar, previousScalarStart: Int)
}

```

#### Core Character API

```swift
@_unavailableInEmbedded
extension UTF8Span {
  /// Whether `i` is on a boundary between `Character`s (i.e. grapheme
  /// clusters).
  @_alwaysEmitIntoClient
  public func isCharacterAligned(_ i: Int) -> Bool

  /// Whether `i` is on a boundary between `Character`s (i.e. grapheme
  /// clusters).
  ///
  /// This function does not validate that `i` is within the span's bounds;
  /// this is an unsafe operation.
  @_alwaysEmitIntoClient
  public func isCharacterAligned(unchecked i: Int) -> Bool

  /// Returns the start of the next `Character` (i.e. grapheme cluster) after
  /// the one  starting at `i`, or the end of the span if `i` denotes the final
  /// `Character`.
  ///
  /// `i` must be `Character`-aligned.
  @_alwaysEmitIntoClient
  public func nextCharacterStart(_ i: Int) -> Int

  /// Returns the start of the next `Character` (i.e. grapheme cluster) after
  /// the one  starting at `i`, or the end of the span if `i` denotes the final
  /// `Character`.
  ///
  /// `i` must be `Character`-aligned.
  ///
  /// This function does not validate that `i` is within the span's bounds;
  /// this is an unsafe operation.
  @_alwaysEmitIntoClient
  public func nextCharacterStart(unchecked i: Int) -> Int

  /// Returns the start of the next `Character` (i.e. grapheme cluster) after
  /// the one  starting at `i`, or the end of the span if `i` denotes the final
  /// `Character`.
  ///
  /// `i` must be `Character`-aligned.
  ///
  /// This function does not validate that `i` is within the span's bounds;
  /// this is an unsafe operation.
  ///
  /// This function does not validate that `i` is `Character`-aligned; this is
  /// an unsafe operation if `i` isn't.
  @_alwaysEmitIntoClient
  public func nextCharacterStart(
    uncheckedAssumingAligned i: Int
  ) -> Int

  /// Returns the start of the `Character` (i.e. grapheme cluster) ending at
  /// `i`, i.e. the `Character` before the one starting at `i` or the last
  /// `Character` if `i` is the end of the span.
  ///
  /// `i` must be `Character`-aligned.
  @_alwaysEmitIntoClient
  public func previousCharacterStart(_ i: Int) -> Int

  /// Returns the start of the `Character` (i.e. grapheme cluster) ending at
  /// `i`, i.e. the `Character` before the one starting at `i` or the last
  /// `Character` if `i` is the end of the span.
  ///
  /// `i` must be `Character`-aligned.
  ///
  /// This function does not validate that `i` is within the span's bounds;
  /// this is an unsafe operation.
  @_alwaysEmitIntoClient
  public func previousCharacterStart(unchecked i: Int) -> Int

  /// Returns the start of the `Character` (i.e. grapheme cluster) ending at
  /// `i`, i.e. the `Character` before the one starting at `i` or the last
  /// `Character` if `i` is the end of the span.
  ///
  /// `i` must be `Character`-aligned.
  ///
  /// This function does not validate that `i` is within the span's bounds;
  /// this is an unsafe operation.
  ///
  /// This function does not validate that `i` is `Character`-aligned; this is
  /// an unsafe operation if `i` isn't.
  @_alwaysEmitIntoClient
  public func previousCharacterStart(
    uncheckedAssumingAligned i: Int
  ) -> Int

  /// Decode the `Character` starting at `i` Return it and the start of the
  /// next `Character`.
  ///
  /// `i` must be `Character`-aligned.
  @_alwaysEmitIntoClient
  public func decodeNextCharacter(
    _ i: Int
  ) -> (Character, nextCharacterStart: Int)

  /// Decode the `Character` starting at `i` Return it and the start of the
  /// next `Character`.
  ///
  /// `i` must be `Character`-aligned.
  ///
  /// This function does not validate that `i` is within the span's bounds;
  /// this is an unsafe operation.
  @_alwaysEmitIntoClient
  public func decodeNextCharacter(
    unchecked i: Int
  ) -> (Character, nextCharacterStart: Int)

  /// Decode the `Character` starting at `i` Return it and the start of the
  /// next `Character`.
  ///
  /// `i` must be `Character`-aligned.
  ///
  /// This function does not validate that `i` is within the span's bounds;
  /// this is an unsafe operation.
  ///
  /// This function does not validate that `i` is `Character`-aligned; this is
  /// an unsafe operation if `i` isn't.
  @_alwaysEmitIntoClient
  public func decodeNextCharacter(
    uncheckedAssumingAligned i: Int
  ) -> (Character, nextCharacterStart: Int)

  /// Decode the `Character` (i.e. grapheme cluster) ending at `i`, i.e. the
  /// previous `Character`. Return it and the start of that `Character`.
  ///
  /// `i` must be `Character`-aligned.
  @_alwaysEmitIntoClient
  public func decodePreviousCharacter(_ i: Int) -> (Character, Int)

  /// Decode the `Character` (i.e. grapheme cluster) ending at `i`, i.e. the
  /// previous `Character`. Return it and the start of that `Character`.
  ///
  /// `i` must be `Character`-aligned.
  ///
  /// This function does not validate that `i` is within the span's bounds;
  /// this is an unsafe operation.
  @_alwaysEmitIntoClient
  public func decodePreviousCharacter(
    unchecked i: Int
  ) -> (Character, Int)

  /// Decode the `Character` (i.e. grapheme cluster) ending at `i`, i.e. the
  /// previous `Character`. Return it and the start of that `Character`.
  ///
  /// `i` must be `Character`-aligned.
  ///
  /// This function does not validate that `i` is within the span's bounds;
  /// this is an unsafe operation.
  ///
  /// This function does not validate that `i` is `Character`-aligned; this is
  /// an unsafe operation if `i` isn't.
  @_alwaysEmitIntoClient
  public func decodePreviousCharacter(
    uncheckedAssumingAligned i: Int
  ) -> (Character, Int)

}

```

#### Derived Scalar operations

```swift
extension UTF8Span {
  /// Find the nearest scalar-aligned position `<= i`.
  @_alwaysEmitIntoClient
  public func scalarAlignBackwards(_ i: Int) -> Int

  /// Find the nearest scalar-aligned position `<= i`.
  ///
  /// This function does not validate that `i` is within the span's bounds;
  /// this is an unsafe operation.
  @_alwaysEmitIntoClient
  public func scalarAlignBackwards(unchecked i: Int) -> Int

  /// Find the nearest scalar-aligned position `>= i`.
  @_alwaysEmitIntoClient
  public func scalarAlignForwards(_ i: Int) -> Int

  /// Find the nearest scalar-aligned position `>= i`.
  ///
  /// This function does not validate that `i` is within the span's bounds;
  /// this is an unsafe operation.
  @_alwaysEmitIntoClient
  public func scalarAlignForwards(unchecked i: Int) -> Int
}
```

#### Derived Character operations

```swift
@_unavailableInEmbedded
extension UTF8Span {
  /// Find the nearest `Character` (i.e. grapheme cluster)-aligned position
  /// that is `<= i`.
  @_alwaysEmitIntoClient
  public func characterAlignBackwards(_ i: Int) -> Int

  /// Find the nearest `Character` (i.e. grapheme cluster)-aligned position
  /// that is `<= i`.
  ///
  /// This function does not validate that `i` is within the span's bounds;
  /// this is an unsafe operation.
  @_alwaysEmitIntoClient
  public func characterAlignBackwards(unchecked i: Int) -> Int

  /// Find the nearest `Character` (i.e. grapheme cluster)-aligned position
  /// that is `>= i`.
  @_alwaysEmitIntoClient
  public func characterAlignForwards(_ i: Int) -> Int

  /// Find the nearest `Character` (i.e. grapheme cluster)-aligned position
  /// that is `>= i`.
  ///
  /// This function does not validate that `i` is within the span's bounds;
  /// this is an unsafe operation.
  @_alwaysEmitIntoClient
  public func characterAlignForwards(unchecked i: Int) -> Int
}
```

### Collection-like API

#### Comparisons

```swift
extension UTF8Span {
  /// Whether this span has the same bytes as `other`.
  @_alwaysEmitIntoClient
  public func bytesEqual(to other: UTF8Span) -> Bool

  /// Whether this span has the same bytes as `other`.
  @_alwaysEmitIntoClient
  public func bytesEqual(to other: some Sequence<UInt8>) -> Bool

  /// Whether this span has the same `Unicode.Scalar`s as `other`.
  @_alwaysEmitIntoClient
  public func scalarsEqual(
    to other: some Sequence<Unicode.Scalar>
  ) -> Bool

  /// Whether this span has the same `Character`s as `other`.
  @_unavailableInEmbedded
  @_alwaysEmitIntoClient
  public func charactersEqual(
    to other: some Sequence<Character>
  ) -> Bool

}
```

#### Canonical equivalence and ordering

`UTF8Span` can perform Unicode canonical equivalence checks (i.e. the semantics of `String.==` and `Character.==`).

```swift
extension UTF8Span {
  /// Whether `self` is equivalent to `other` under Unicode Canonical
  /// Equivalence.
  @_unavailableInEmbedded
  public func isCanonicallyEquivalent(
    to other: UTF8Span
  ) -> Bool

  /// Whether `self` orders less than `other` under Unicode Canonical
  /// Equivalance using normalized code-unit order (in NFC).
  @_unavailableInEmbedded
  public func isCanonicallyLessThan(
    _ other: UTF8Span
  ) -> Bool
}
```

#### Extracting sub-spans

Similarly to `Span`, we support subscripting and extracting sub-spans. Since a `UTF8Span` is always validly-encoded UTF-8, extracting must happen along Unicode scalar boundaries.

```swift
extension UTF8Span {
  /// Constructs a new `UTF8Span` span over the bytes within the supplied
  /// range of positions within this span.
  ///
  /// `bounds` must be scalar aligned.
  ///
  /// The returned span's first item is always at offset 0; unlike buffer
  /// slices, extracted spans do not generally share their indices with the
  /// span from which they are extracted.
  ///
  /// - Parameter bounds: A valid range of positions. Every position in
  ///     this range must be within the bounds of this `Span`.
  ///
  /// - Returns: A `UTF8Span` over the bytes within `bounds`.
  @_alwaysEmitIntoClient
  public func extracting(_ bounds: some RangeExpression<Int>) -> Self

  /// Constructs a new `UTF8Span` span over the bytes within the supplied
  /// range of positions within this span.
  ///
  /// `bounds` must be scalar aligned.
  ///
  /// This function does not validate that `bounds` is within the span's
  /// bounds; this is an unsafe operation.
  ///
  /// The returned span's first item is always at offset 0; unlike buffer
  /// slices, extracted spans do not generally share their indices with the
  /// span from which they are extracted.
  ///
  /// - Parameter bounds: A valid range of positions. Every position in
  ///     this range must be within the bounds of this `Span`.
  ///
  /// - Returns: A `UTF8Span` over the bytes within `bounds`.
  @_alwaysEmitIntoClient
  public func extracting(
    unchecked bounds: some RangeExpression<Int>
  ) -> Self

  /// Constructs a new `UTF8Span` span over the bytes within the supplied
  /// range of positions within this span.
  ///
  /// This function does not validate that `bounds` is within the span's
  /// bounds; this is an unsafe operation.
  ///
  /// This function does not validate that `bounds` is within the span's
  /// bounds; this is an unsafe operation.
  ///
  /// The returned span's first item is always at offset 0; unlike buffer
  /// slices, extracted spans do not generally share their indices with the
  /// span from which they are extracted.
  ///
  /// - Parameter bounds: A valid range of positions. Every position in
  ///     this range must be within the bounds of this `Span`.
  ///
  /// - Returns: A `UTF8Span` over the bytes within `bounds`.
  @_alwaysEmitIntoClient
  public func extracting(
    uncheckedAssumingAligned bounds: some RangeExpression<Int>
  ) -> Self
}

```

#### Misc.

```swift
extension UTF8Span {
  @_alwaysEmitIntoClient
  public var isEmpty: Bool { get }

  @_alwaysEmitIntoClient
  public var storage: Span<UInt8> { get }

  /// Whether `i` is in bounds
  @_alwaysEmitIntoClient
  public func boundsCheck(_ i: Int) -> Bool

  /// Whether `bounds` is in bounds
  @_alwaysEmitIntoClient
  public func boundsCheck(_ bounds: Range<Int>) -> Bool

  /// Calls a closure with a pointer to the viewed contiguous storage.
  ///
  /// The buffer pointer passed as an argument to `body` is valid only
  /// during the execution of `withUnsafeBufferPointer(_:)`.
  /// Do not store or return the pointer for later use.
  ///
  /// - Parameter body: A closure with an `UnsafeBufferPointer` parameter
  ///   that points to the viewed contiguous storage. If `body` has
  ///   a return value, that value is also used as the return value
  ///   for the `withUnsafeBufferPointer(_:)` method. The closure's
  ///   parameter is valid only for the duration of its execution.
  /// - Returns: The return value of the `body` closure parameter.
  @_alwaysEmitIntoClient
  borrowing public func withUnsafeBufferPointer<
    E: Error, Result: ~Copyable & ~Escapable
  >(
    _ body: (_ buffer: borrowing UnsafeBufferPointer<UInt8>) throws(E) -> Result
  ) throws(E) -> dependsOn(self) Result
}
```

### Queries

`UTF8Span` checks at construction time and remembers whether its contents are all ASCII. Additional checks can be requested and remembered.

```swift
extension UTF8Span {
  /// Returns whether the validated contents were all-ASCII. This is checked at
  /// initialization time and remembered.
  @inlinable @inline(__always)
  public var isASCII: Bool { get }

  /// Returns whether the contents are known to be NFC. This is not
  /// always checked at initialization time and is set by `checkForNFC`.
  @inlinable @inline(__always)
  @_unavailableInEmbedded
  public var isKnownNFC: Bool { get }

  /// Do a scan checking for whether the contents are in Normal Form C.
  /// When the contents are in NFC, canonical equivalence checks are much
  /// faster.
  ///
  /// `quickCheck` will check for a subset of NFC contents using the
  /// NFCQuickCheck algorithm, which is faster than the full normalization
  /// algorithm. However, it cannot detect all NFC contents.
  ///
  /// Updates the `isKnownNFC` bit.
  @_unavailableInEmbedded
  public mutating func checkForNFC(
    quickCheck: Bool
  ) -> Bool

  /// Returns whether every `Character` (i.e. grapheme cluster)
  /// is known to be comprised of a single `Unicode.Scalar`.
  ///
  /// This is not always checked at initialization time. It is set by
  /// `checkForSingleScalarCharacters`.
  @_unavailableInEmbedded
  @inlinable @inline(__always)
  public var isKnownSingleScalarCharacters: Bool { get }

  /// Do a scan, checking whether every `Character` (i.e. grapheme cluster)
  /// is comprised of only a single `Unicode.Scalar`. When a span contains
  /// only single-scalar characters, character operations are much faster.
  ///
  /// `quickCheck` will check for a subset of single-scalar character contents
  /// using a faster algorithm than the full grapheme breaking algorithm.
  /// However, it cannot detect all single-scalar `Character` contents.
  ///
  /// Updates the `isKnownSingleScalarCharacters` bit.
  @_unavailableInEmbedded
  public mutating func checkForSingleScalarCharacters(
    quickCheck: Bool
  ) -> Bool
}
```

### Spans from strings

```swift
@_unavailableInEmbedded
extension String {
  /// ... note that a copy may happen if `String` is not native...
  public var utf8Span: UTF8Span { _read }
}

@_unavailableInEmbedded
extension Substring {
  // ... note that a copy may happen if `Substring` is not native...
  public var utf8Span: UTF8Span { _read }
}
```



## Source compatibility

This proposal is additive and source-compatible with existing code.

## ABI compatibility

This proposal is additive and ABI-compatible with existing code.

## Implications on adoption

The additions described in this proposal require a new version of the standard library and runtime.

## Future directions

### More alignments

Future API could include whether an index is "word aligned" (either [simple](https://www.unicode.org/reports/tr18/#Simple_Word_Boundaries) or [default](https://www.unicode.org/reports/tr18/#Default_Word_Boundaries)), "line aligned", etc.

### Normalization

Future API could include checks for whether the content is in a particular normal form (not just NFC).

### UnicodeScalarView and CharacterView

Like `Span`, we are deferring adding any collection-like types to non-escapable `UTF8Span`. Future work includes adding view types and corresponding iterators.

For an example implementation of those see [the `UTFSpanViews.swift` test file](https://github.com/apple/swift-collections/pull/394).

### More Collectiony algorithms

We propose equality checks (e.g. `scalarsEqual`), as those are incredibly common and useful operations. We have (tentatively) deferred other algorithms until non-escapable collections are figured out.

However, we can add select high-value algorithms if motivated by the community.

### More validation API

Future work includes returning all the encoding errors found in a given input.

```swift
extension UTF8 {
  public static func checkAllErrors(
    _ s: some Sequence<UInt8>
  ) -> some Sequence<UTF8.EncodingError>
```

See [`_checkAllErrors` in `UTF8EncodingError.swift`](https://github.com/apple/swift-collections/pull/394).

### Transcoded views, normalized views, case-folded views, etc

We could provide lazily transcoded, normalized, case-folded, etc., views. If we do any of these for `UTF8Span`, we should consider adding equivalents on `String`, `Substring`, etc.

For example, transcoded views can be generalized:

```swift
extension UTF8Span {
  /// A view of the span's contents as a bidirectional collection of
  /// transcoded `Encoding.CodeUnit`s.
  @frozen
  public struct TranscodedView<Encoding: _UnicodeEncoding> {
    public var span: UTF8Span

    @inlinable
    public init(_ span: UTF8Span)

    ...
  }
}
```

We could similarly provide lazily-normalized views of code units or scalars under NFC or NFD (which the stdlib already distributes data tables for), possibly generic via a protocol for 3rd party normal forms.

Finally, case-folded functionality can be accessed in today's Swift via [scalar properties](https://developer.apple.com/documentation/swift/unicode/scalar/properties-swift.struct), but we could provide convenience collections ourselves as well.


### Regex or regex-like support

Future API additions would be to support `Regex`es on `UTF8Span`. We'd expose grapheme-level semantics, scalar-level semantics, and introduce byte-level semantics.

Another future direction could be to add many routines corresponding to the underlying operations performed by the regex engine, such as:

```swift
extension UTF8Span.CharacterView {
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


### Canonical Spaceships

Should a `ComparisonResult` (or [spaceship](https://forums.swift.org/t/pitch-comparison-reform/5662)) be added to Swift, we could support that operation under canonical equivalence in a single pass rather than subsequent calls to `isCanonicallyEquivalent(to:)` and `isCanonicallyLessThan(_:)`.


### Other Unicode functionality

For the purposes of this pitch, we're not looking to expand the scope of functionality beyond what the stdlib already does in support of `String`'s API. Other functionality can be considered future work.


### Exposing `String`'s storage class

String's internal storage class is null-terminated valid UTF-8 (by substituting replacement characters) and implements range-replaceable operations along scalar boundaries. We could consider exposing the storage class itself, which might be useful for embedded platforms that don't have `String`.

### Yield UTF8Spans in byte parsers

Span's proposal mentions a future direction of byte parsing helpers on a `Cursor` or `Iterator` type on `RawSpan`. We could extend these types (or analogous types on `Span<UInt>`) with UTF-8 parsing code:

```swift
extension RawSpan.Cursor {
  public mutating func parseUTF8(length: Int) throws -> UTF8Span

  public mutating func parseNullTermiantedUTF8() throws -> UTF8Span
}
```

### Track other bits

Future work include tracking whether the contents are NULL-terminated (useful for C bridging), whether the contents contain any newlines or only a single newline at the end (useful for accelerating Regex `.`), etc.

### Putting more API on String

`String` would also benefit from the query API, such as `isKnownNFC` and corresponding scan methods. Because a string may be a lazily-bridged instance of `NSString`, we don't always have the bits available to query or set, but this may become viable pending future improvements in bridging.

### Generalize printing and logging facilities

Many printing and logging protocols and facilities operate in terms of `String`. They could be generalized to work in terms of UTF-8 bytes instead, which is important for embedded.

## Alternatives considered

### Invalid start / end of input UTF-8 encoding errors

Earlier prototypes had `.invalidStartOfInput` and `.invalidEndOfInput` UTF8 validation errors to communicate that the input was perhaps incomplete or not slices along scalar boundaries. In this scenario, `.invalidStartOfInput` is equivalent to `.unexpectedContinuation` with the range's lower bound equal to 0 and `.invalidEndOfInput` is equivalent to `.truncatedScalar` with the range's upper bound equal to `count`.

This was rejected so as to not have two ways to encode the same error. There is no loss of information and `.unexpectedContinuation`/`.truncatedScalar` with ranges are more semantically precise.

### An unsafe UTF8 Buffer Pointer type

An [earlier pitch](https://forums.swift.org/t/pitch-utf-8-processing-over-unsafe-contiguous-bytes/69715) proposed an unsafe version of `UTF8Span`. Now that we have `~Escapable`, a memory-safe `UTF8Span` is better.

### Other names for basic operations

An alternative name for `nextScalarStart(_:)` and `previousScalarStart(_:)` could be something like `scalarEnd(startingAt:)` and `scalarStart(endingAt: i)`. Similarly, `decodeNextScalar(_:)` and `decodePreviousScalar(_:)` could be `decodeScalar(startingAt:)` and `decodeScalar(endingAt:)`. These names are similar to `index(after:)` and `index(before:)`.

However, in practice this buries the direction deeper into the argument label and is more confusing than the `index(before/after:)` analogues. This is especially true when the argument label contains `unchecked` or `uncheckedAssumingAligned`.

That being said, these names are definitely bikesheddable and we'd like suggestions from the community.


### Other bounds or alignment checked formulations

For many operations that take an index that needs to be appropriately aligned, we propose `foo(_:)`, `foo(unchecked:)`, and `foo(uncheckedAssumingAligned:)`.

`foo(_:)` and `foo(unchecked:)` have analogues in `Span` and `foo(uncheckedAssumingAligned:)` is the lowest level interface that a type such as `Iterator` would call (since it maintains index validity and alignment as an invariant).

We could additionally have a `foo(assumingAligned:)` overload that does bounds checking, but it's unclear what the use case would be.

Another alternative is to only have a variant that skips both bounds and alignment checks and call it `foo(unchecked:)`. However, this use of `unchecked:` is far more nuanced than `Span`'s and it's not the case that any `i` in `0..<count` would be valid.

We could also only offer `foo(_:)` and `foo(uncheckedAssumingAligned:)`. Unaligned API such as `isScalarAligned(_:)` and `isScalarAligned(unchecked:)` would keep their names.




## Acknowledgments

Karoy Lorentey, Karl, Geordie_J, and fclout, contributed to this proposal with their clarifying questions and discussions.




