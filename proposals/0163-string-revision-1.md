# String Revision: Collection Conformance, C Interop, Transcoding

* Proposal: [SE-0163](0163-string-revision-1.md)
* Authors: [Ben Cohen](https://github.com/airspeedswift), [Dave Abrahams](https://github.com/dabrahams/)
* Review Manager: [John McCall](https://github.com/rjmccall)
* Status: **Implemented (Swift 4.0)**
* Revision: 2
* Previous Revision: [1](https://github.com/swiftlang/swift-evolution/blob/7513547ddac66b06770a1fd620aad915d75987ff/proposals/0163-string-revision-1.md)
* Decision Notes: [Rationale #1](https://forums.swift.org/t/accepted-se-0163-string-revision-collection-conformance-c-interop-transcoding/5716/2), [Rationale #2](https://forums.swift.org/t/accepted-se-0163-string-revision-collection-conformance-c-interop-transcoding/5952)

## Introduction

This proposal is to implement a subset of the changes from the [Swift 4
String
Manifesto](https://github.com/apple/swift/blob/master/docs/StringManifesto.md).

Specifically:

 * Make `String` conform to `BidirectionalCollection`
 * Make `String` conform to `RangeReplaceableCollection` 
 * Create a `Substring` type for `String.SubSequence`
 * Create a `StringProtocol` protocol to allow for generic operations over both types.
 * Consolidate on a concise set of C interop methods.
 * Revise the transcoding infrastructure.
 * Sink Unicode-specific functionality into a `Unicode` namespace.

Other existing aspects of `String` remain unchanged for the purposes of this 
proposal.

## Motivation

This proposal follows up on a number of recommendations found in the manifesto:

`Collection` conformance was dropped from `String` in Swift 2. After
reevaluation, the feeling is that the minor discrepancies with
required `RangeReplaceableCollection` semantics (the fact that some
characters may merge when Strings are concatenated) are outweighed by
the significant benefits of restoring these conformances. For more
detail on the reasoning,
see
[here](https://github.com/apple/swift/blob/master/docs/StringManifesto.md#string-should-be-a-collection-of-characters-again)

While it is not a collection, the Swift 3 string does have slicing operations.
`String` is currently serving as its own subsequence, allowing  substrings
to share storage with their "owner". This can lead to memory leaks when small substrings of larger
strings are stored long-term (see [here](https://github.com/apple/swift/blob/master/docs/StringManifesto.md#substrings)
for more detail on this problem). Introducing a separate type of `Substring` to
serve as `String.Subsequence` is recommended to resolve this issue, in a similar
fashion to `ArraySlice`.

As noted in the manifesto, support for interoperation with nul-terminated C
strings in Swift 3 is scattered and incoherent, with 6 ways to transform a C
string into a `String` and four ways to do the inverse. These APIs should be
replaced with a simpler set of methods on `String`.

## Proposed solution

A new type, `Substring`, will be introduced. Similar to `ArraySlice` it will
be documented as only for short- to medium-term storage:

> **Important**
>
> Long-term storage of `Substring` instances is discouraged. A substring holds a
> reference to the entire storage of a larger string, not just to the portion it
> presents, even after the original string’s lifetime ends. Long-term storage of
> a substring may therefore prolong the lifetime of elements that are no longer
> otherwise accessible, which can appear to be memory leakage.

Aside from minor differences, such as having a `SubSequence` of `Self`
and a larger size to describe the range of the subsequence,
`Substring` will be near-identical from a user perspective.

In order to be able to write extensions across both `String` and
`Substring`, a new `StringProtocol` protocol to which the two types
will conform will be introduced. For the purposes of this proposal,
`StringProtocol` will be defined as a protocol to be used whenever you
would previously extend `String`. It should be possible to substitute
`extension StringProtocol { ... }` in Swift 4 wherever 
`extension String { ... }` was written in Swift 3, with one exception: any
passing of `self` into an API that takes a concrete `String` will need to be
rewritten as `String(self)`. If `Self` is a `String` then this should
effectively optimize to a no-op, whereas if `Self` is a `Substring` then this
will force a copy, helping to avoid the "memory leak" problems described above.

The exact nature of the protocol – such as which methods should be
protocol requirements vs which can be implemented as protocol
extensions, are considered implementation details and so not covered
in this proposal.

`StringProtocol` will conform to `BidirectionalCollection`.
`RangeReplaceableCollection` conformance will be added directly onto
the `String` and `Substring` types, as it is possible future
`StringProtocol`-conforming types might not be range-replaceable
(e.g. an immutable type that wraps a `const char *`).

The C string interop methods will be updated to a variant of those
described
[here](https://github.com/apple/swift/blob/master/docs/StringManifesto.md#c-string-interop):
two `withCString` operations and two `init(cString:)` constructors,
one each for UTF8 and for arbitrary encodings. The primary change is
to remove "non-repairing" variants of construction from nul-terminated
C strings. In both of the construction APIs, any invalid encoding
sequence detected will have its longest valid prefix replaced by
`U+FFFD`, the Unicode replacement character, per the Unicode
specification. This covers the common case. The replacement can be
done physically in the underlying storage and the validity of the
result can be recorded in the String's encoding such that future
accesses need not be slowed down by possible error repair
separately. Construction that is aborted when encoding errors are
detected can be accomplished using APIs on the encoding.

Additionally, an `init` that takes a collection of code units and an encoding 
will allow for construction of a `String` from arbitrary collections – for example,
an `UnsafeBufferPointer` containing a non-nul-terminated C string.

The current transcoding support will be updated to improve usability and
performance. The primary changes will be:

 - to allow transcoding directly from one encoding to another without having
   to triangulate through an intermediate scalar value
 - to add the ability to transcode an input collection in reverse, allowing the
   different views on `String` to be made bi-directional
 - to ensure that the APIs can be 
   used to create performant bidirectional decoded and transcoded
   views of underlying code units.
 - to replace the `UnicodeCodec` with a stateless `Unicode.Encoding`
   protocol having associated `ForwardParser` and `ReverseParser`
   types for decoding.

The standard library currently lacks a `Latin1` codec, so a 
`enum Latin1: Unicode.Encoding` type will be added.

## Detailed design

### The `Unicode` Namespace

A `Unicode` “namespace” will be added for components related to
low-level Unicode operations such as transcoding and grapheme
breaking. Absent more direct language support, `Unicode` will, for the
time being, be implemented as a caseless `enum`.  [The caseless `enum`
technique is precedented by `CommandLine`, which vends the equivalent
of `argc` and `argv` for command-line applications.]

```swift
enum Unicode {
  enum ASCII : Unicode.Encoding { ... }
  enum UTF8 : Unicode.Encoding { ... }
  enum UTF16 : Unicode.Encoding { ... }
  enum UTF32 : Unicode.Encoding { ... }
  ...
  enum ParseResult<T> { ... }
  struct Scalar { ... }
}
```

The names `UTF8`, `UTF16`, `UTF32`, and `Scalar` correspond
to entities that exist in Swift 3.  For backward compatibility they will
be exposed to Swift 3 programs with their legacy spellings:

```swift
@available(swift, obsoleted: 4.0, renamed: "Unicode.UTF8")
public typealias UTF8 = Unicode.UTF8
@available(swift, obsoleted: 4.0, renamed: "Unicode.UTF16")
public typealias UTF16 = Unicode.UTF16
@available(swift, obsoleted: 4.0, renamed: "Unicode.UTF32")
public typealias UTF32 = Unicode.UTF32
@available(swift, obsoleted: 4.0, renamed: "Unicode.Scalar")
public typealias UnicodeScalar = Unicode.Scalar
```

Unicode-specific protocols will be presented as members of this
namespace.  Pending the addition of more direct language support, 
typealiases will be used to bring them in from underscored names in 
the `Swift` namespace.  The intention is that diagnostics and
documentation will display the nested, non-underscored names.

```swift
protocol _UnicodeEncoding { ... }
protocol _UnicodeParser { ... }
extension Unicode {
  typealias Encoding = _UnicodeEncoding
  typealias Parser = _UnicodeParser
}
```

`UnicodeCodec` will be updated to refine `Unicode.Encoding`, and
deprecated for Swift 4.  Existing models of `UnicodeCodec` such as
`UTF8` will inherit `Unicode.Encoding` conformance for Swift 3.

As noted [below](#higher-level-unicode-processing) we anticipate
adding many more Unicode-specific components to the `Unicode`
namespace in the near future.

### `String`, `Substring`, and `StringProtocol`

The following additions will be made to the standard library:

```swift
protocol StringProtocol : BidirectionalCollection {
  // Implementation detail as described above
}

extension String : StringProtocol, RangeReplaceableCollection {
  typealias SubSequence = Substring
  subscript(bounds: Range<String.Index>) -> Substring { 
    ...
  }
}

struct Substring : StringProtocol, RangeReplaceableCollection {
  typealias SubSequence = Substring
  // near-identical API surface area to String
}
```

The slicing operations on `String` will be amended to return
`Substring`:

```swift
struct String {
  subscript(bounds: Range<Index>) -> Substring { ... }
}
```

Note that properties or methods that due to their nature create new
`String` storage (such as `lowercased()`) will _not_ change.

C string interopability will be consolidated on the following methods:

```swift
extension String {
  /// Constructs a `String` having the same contents as `codeUnits`.
  ///
  /// - Parameter codeUnits: a collection of code units in
  ///   the given `encoding`.
  /// - Parameter encoding: describes the encoding in which the code units
  ///   should be interpreted.
  init<C: Collection, Encoding: Unicode.Encoding>(
    decoding codeUnits: C, as encoding: Encoding.Type
  )
    where C.Iterator.Element == Encoding.CodeUnit

  /// Constructs a `String` having the same contents as `nulTerminatedUTF8`.
  ///
  /// - Parameter nulTerminatedUTF8: a sequence of contiguous UTF-8 encoded 
  ///   bytes ending just before the first zero byte (NUL character).
  init(cString nulTerminatedUTF8: UnsafePointer<CChar>)
  
  /// Constructs a `String` having the same contents as `nulTerminatedCodeUnits`.
  ///
  /// - Parameter nulTerminatedCodeUnits: a sequence of contiguous code units in
  ///   the given `encoding`, ending just before the first zero code unit.
  /// - Parameter encoding: describes the encoding in which the code units
  ///   should be interpreted.
  init<Encoding: Unicode.Encoding>(
    decodingCString nulTerminatedCodeUnits: UnsafePointer<Encoding.CodeUnit>,
    as: Encoding.Type)
    
  /// Invokes the given closure on the contents of the string, represented as a
  /// pointer to a null-terminated sequence of UTF-8 code units.
  func withCString<Result>(
    _ body: (UnsafePointer<CChar>) throws -> Result) rethrows -> Result

  /// Invokes the given closure on the contents of the string, represented as a
  /// pointer to a null-terminated sequence of code units in the given encoding.
  func withCString<Result, Encoding: Unicode.Encoding>(
    encodedAs: Encoding.Type,
    _ body: (UnsafePointer<Encoding.CodeUnit>) throws -> Result
  ) rethrows -> Result
}
```

Additionally, the current ability to pass a Swift `String` directly
into methods that take a C string (`UnsafePointer<CChar>`) will remain
as-is.

### Low-level Unicode Processing

A new protocol, `Unicode.Encoding`, will be added to replace the
current `UnicodeCodec` protocol.

```swift
extension Unicode { typealias Encoding = _UnicodeEncoding }

public protocol _UnicodeEncoding {
  /// The basic unit of encoding
  associatedtype CodeUnit : UnsignedInteger, FixedWidthInteger
  
  /// A valid scalar value as represented in this encoding
  associatedtype EncodedScalar : BidirectionalCollection
    where EncodedScalar.Iterator.Element == CodeUnit

  /// A unicode scalar value to be used when repairing
  /// encoding/decoding errors, as represented in this encoding.
  ///
  /// If the Unicode replacement character U+FFFD is representable in this
  /// encoding, `encodedReplacementCharacter` encodes that scalar value.
  static var encodedReplacementCharacter : EncodedScalar { get }

  /// Converts from encoded to encoding-independent representation
  static func decode(_ content: EncodedScalar) -> Unicode.Scalar

  /// Converts from encoding-independent to encoded representation, returning
  /// `nil` if the scalar can't be represented in this encoding.
  static func encode(_ content: Unicode.Scalar) -> EncodedScalar?

  /// Converts a scalar from another encoding's representation, returning
  /// `nil` if the scalar can't be represented in this encoding.
  ///
  /// A default implementation of this method will be provided 
  /// automatically for any conforming type that does not implement one.
  static func transcode<FromEncoding : UnicodeEncoding>(
    _ content: FromEncoding.EncodedScalar, from _: FromEncoding.Type
  ) -> EncodedScalar?

  /// A type that can be used to parse `CodeUnits` into
  /// `EncodedScalar`s.
  associatedtype ForwardParser : Unicode.Parser
    where ForwardParser.Encoding == Self
    
  /// A type that can be used to parse a reversed sequence of
  /// `CodeUnits` into `EncodedScalar`s.
  associatedtype ReverseParser : Unicode.Parser
    where ReverseParser.Encoding == Self
}
```

Parsing `CodeUnits` into `EncodedScalar`s, in either direction, is
done with models of `Unicode.Parser`:

```swift
extension Unicode {  typealias Parser = _UnicodeParser }

/// Types that separate streams of code units into encoded Unicode
/// scalar values.
public protocol _UnicodeParser {
  /// The encoding with which this parser is associated
  associatedtype Encoding : Unicode.Encoding

  /// Constructs an instance that can be used to begin parsing `CodeUnit`s at
  /// any Unicode scalar boundary.
  init()

  /// Parses a single Unicode scalar value from `input`.
  mutating func parseScalar<I : IteratorProtocol>(
    from input: inout I
  ) -> Unicode.ParseResult<Encoding.EncodedScalar>
  where I.Element == Encoding.CodeUnit
}

extension Unicode { 
  /// The result of attempting to parse a `T` from some input.
  public enum ParseResult<T> {
  /// A `T` was parsed successfully
  case valid(T)
  
  /// The input was entirely consumed.
  case emptyInput
  
  /// An encoding error was detected.
  ///
  /// `length` is the number of underlying code units consumed by this
  /// error (when decoding, the length of the longest prefix that
  /// could be recognized of a valid encoding sequence).
  case error(length: Int)
  }
}
```

### Higher-Level Unicode Processing

The Unicode processing APIs proposed here are intentionally extremely
low-level.  We have proven that they are sufficient to implement
higher-level constructs, but those designs are still baking and not
yet ready for review.  We expect to propose generic `Iterator`,
`Sequence`, and `Collection` views that expose transcoded or segmented
views of arbitrary underlying storage, as separate components in the
`Unicode` namespace.

## Source compatibility

Adding collection conformance to `String` should not materially impact source
stability as it is purely additive: Swift 3's `String` interface currently
fulfills all of the requirements for a bidirectional range replaceable
collection.

Altering `String`'s slicing operations to return a different type is source
breaking. The following mitigating steps are proposed:

 - Add a deprecated subscript operator that will run in Swift 3 compatibility
   mode and which will return a `String` not a `Substring`.
 
 - Add deprecated versions of all current slicing methods to similarly return a
   `String`.
   
i.e.:

```swift
extension String {
  @available(swift, obsoleted: 4)
  subscript(bounds: Range<Index>) -> String {
    return String(characters[bounds])
  }

  @available(swift, obsoleted: 4)
  subscript(bounds: ClosedRange<Index>) -> String {
    return String(characters[bounds])
  }
}
```       

In a review of 77 popular Swift projects found on GitHub, these changes
resolved any build issues in the 12 projects that assumed an explicit `String`
type returned from slicing operations.

Due to the change in internal implementation, this means that these operations
will be _O(n)_ rather than _O(1)_. This is not expected to be a major concern,
based on experiences from a similar change made to Java, but projects will be
able to work around performance issues without upgrading to Swift 4 by
explicitly typing slices as `Substring`, which will call the Swift 4 variant,
and which will be available but not invoked by default in Swift 3 mode.

The C string interoperability methods outside the ones described in the
detailed design will remain in Swift 3 mode, be deprecated in Swift 4 mode, and
be removed in a subsequent release. `UnicodeCodec` will be similarly deprecated.

## Effect on ABI stability

As a fundamental currency type for Swift, it is essential that the
`String` type (and its associated subsequence) is in a good long-term
state before being locked down when Swift declares ABI stability.
Shrinking the size of `String` to be 64 bits is an important part of
the story.  As full ABI stablity is not planned for Swift 4, it is
currently unclear when the transition to a 64-bit memory layout will
occur.

## Effect on API resilience

Decisions about the API resilience of the `String` type are still to be
determined, but are not adversely affected by this proposal.

## Alternatives considered

For a more in-depth discussion of some of the trade-offs in string design, see
the manifesto and associated [evolution thread](https://forums.swift.org/t/strings-in-swift-4/4939).

This proposal does not yet introduce an implicit conversion from `Substring` to
`String`. The decision on whether to add this will be deferred pending feedback
on the initial implementation. The intention is to make a preview toolchain
available for feedback, including on whether this implicit conversion is
necessary, prior to the release of Swift 4.


