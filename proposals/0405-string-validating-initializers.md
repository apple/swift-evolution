# String Initializers with Encoding Validation

* Proposal: [SE-0405](0405-string-validating-initializers.md)
* Author: [Guillaume Lessard](https://github.com/glessard)
* Review Manager: [Tony Allevato](https://github.com/allevato)
* Status: **Implemented (Swift 6.0)**
* Bugs: rdar://99276048, rdar://99832858
* Implementation: [Swift PR 68419](https://github.com/apple/swift/pull/68419), [Swift PR 68423](https://github.com/apple/swift/pull/68423)
* Review: ([pitch](https://forums.swift.org/t/66206)), ([review](https://forums.swift.org/t/se-0405-string-initializers-with-encoding-validation/66655)), ([acceptance](https://forums.swift.org/t/accepted-with-modifications-se-0405-string-initializers-with-encoding-validation/67134))
* Previous Revisions: [0](https://gist.github.com/glessard/d1ed79b7968b4ad2115462b3d1eba805), [1](https://github.com/swiftlang/swift-evolution/blob/37531427931a57ff2a76225741c99de8fa8b8c59/proposals/0405-string-validating-initializers.md)

## Introduction

We propose adding new `String` failable initializers that validate encoded input, and return `nil` when the input contains any invalid elements.

## Motivation

The `String` type guarantees that it represents well-formed Unicode text. When data representing text is received from a file, the network, or some other source, it may be relevant to store it in a `String`, but that data must be validated first. `String` already provides a way to transform data to valid Unicode by repairing invalid elements, but such a transformation is often not desirable, especially when dealing with untrusted sources. For example a JSON decoder cannot transform its input; it must fail if a span representing text contains any invalid UTF-8.

This functionality has not been available directly from the standard library. It is possible to compose it using existing public API, but only at the cost of extra memory copies and allocations. The standard library is uniquely positioned to implement this functionality in a performant way.

## Proposed Solution

We will add a new `String` initializer that can fail, returning `nil`, when its input is found to be invalid according the encoding represented by a type parameter that conforms to `Unicode.Encoding`.

```swift
extension String {
  public init?<Encoding: Unicode.Encoding>(
    validating codeUnits: some Sequence<Encoding.CodeUnit>,
    as: Encoding.Type
  )
}
```

When processing data obtained from C, it is frequently the case that UTF-8 data is represented by `Int8` (typically as `CChar`) rather than `UInt8`. We will provide a convenience initializer for this use case:

```swift
extension String {
  public init?<Encoding: Unicode.Encoding>(
    validating codeUnits: some Sequence<Int8>,
    as: Encoding.Type
  ) where Encoding.CodeUnit == UInt8
}
```

`String` already features a validating initializer for UTF-8 input, intended for C interoperability.  Its argument label does not convey the expectation that its input is a null-terminated C string, and this has caused errors. We propose to change the labels in order to clarify the preconditions:

```swift
extension String {
  public init?(validatingCString nullTerminatedUTF8: UnsafePointer<CChar>)

  @available(Swift 5.XLIX, deprecated, renamed: "String.init(validatingCString:)")
  public init?(validatingUTF8 cString: UnsafePointer<CChar>)
}
```

Note that unlike `String.init?(validatingCString:)`, the `String.init?(validating:as:)` initializers convert their whole input, including any embedded `\0` code units.

## Detailed Design

We want these new initializers to be performant. As such, their implementation should minimize the number of memory allocations and copies required. We achieve this performance with `@inlinable` implementations that leverage `withContiguousStorageIfAvailable` to provide a concrete (`internal`) code path for the validation cases. The concrete `internal` initializer itself calls a number of functions internal to the standard library.

```swift
extension String {
  /// Creates a new `String` by copying and validating the sequence of
  /// code units passed in, according to the specified encoding.
  ///
  /// This initializer does not try to repair ill-formed code unit sequences.
  /// If any are found, the result of the initializer is `nil`.
  ///
  /// The following example calls this initializer with the contents of two
  /// different arrays---first with a well-formed UTF-8 code unit sequence and
  /// then with an ill-formed UTF-16 code unit sequence.
  ///
  ///     let validUTF8: [UInt8] = [67, 97, 0, 102, 195, 169]
  ///     let valid = String(validating: validUTF8, as: UTF8.self)
  ///     print(valid)
  ///     // Prints "Optional("Café")"
  ///
  ///     let invalidUTF16: [UInt16] = [0x41, 0x42, 0xd801]
  ///     let invalid = String(validating: invalidUTF16, as: UTF16.self)
  ///     print(invalid)
  ///     // Prints "nil"
  ///
  /// - Parameters:
  ///   - codeUnits: A sequence of code units that encode a `String`
  ///   - encoding: A conformer to `Unicode.Encoding` to be used
  ///               to decode `codeUnits`.
  @inlinable
  public init?<Encoding>(
    validating codeUnits: some Sequence<Encoding.CodeUnit>,
    as encoding: Encoding.Type
  ) where Encoding: Unicode.Encoding

  /// Creates a new `String` by copying and validating the sequence of
  /// `Int8` passed in, according to the specified encoding.
  ///
  /// This initializer does not try to repair ill-formed code unit sequences.
  /// If any are found, the result of the initializer is `nil`.
  ///
  /// The following example calls this initializer with the contents of two
  /// different arrays---first with a well-formed UTF-8 code unit sequence and
  /// then with an ill-formed ASCII code unit sequence.
  ///
  ///     let validUTF8: [Int8] = [67, 97, 0, 102, -61, -87]
  ///     let valid = String(validating: validUTF8, as: UTF8.self)
  ///     print(valid)
  ///     // Prints "Optional("Café")"
  ///
  ///     let invalidASCII: [Int8] = [67, 97, -5]
  ///     let invalid = String(validating: invalidASCII, as: Unicode.ASCII.self)
  ///     print(invalid)
  ///     // Prints "nil"
  ///
  /// - Parameters:
  ///   - codeUnits: A sequence of code units that encode a `String`
  ///   - encoding: A conformer to `Unicode.Encoding` that can decode
  ///               `codeUnits` as `UInt8`
  @inlinable
  public init?<Encoding>(
    validating codeUnits: some Sequence<Int8>,
    as encoding: Encoding.Type
  ) where Encoding: Unicode.Encoding, Encoding.CodeUnit == UInt8
}
```

```swift
extension String {
  /// Creates a new string by copying and validating the null-terminated UTF-8
  /// data referenced by the given pointer.
  ///
  /// This initializer does not try to repair ill-formed UTF-8 code unit
  /// sequences. If any are found, the result of the initializer is `nil`.
  ///
  /// The following example calls this initializer with pointers to the
  /// contents of two different `CChar` arrays---first with well-formed
  /// UTF-8 code unit sequences and the second with an ill-formed sequence at
  /// the end.
  ///
  ///     let validUTF8: [CChar] = [67, 97, 102, -61, -87, 0]
  ///     validUTF8.withUnsafeBufferPointer { ptr in
  ///         let s = String(validatingCString: ptr.baseAddress!)
  ///         print(s)
  ///     }
  ///     // Prints "Optional("Café")"
  ///
  ///     let invalidUTF8: [CChar] = [67, 97, 102, -61, 0]
  ///     invalidUTF8.withUnsafeBufferPointer { ptr in
  ///         let s = String(validatingCString: ptr.baseAddress!)
  ///         print(s)
  ///     }
  ///     // Prints "nil"
  ///
  /// - Parameter nullTerminatedUTF8: A pointer to a null-terminated UTF-8 code sequence.
  @_silgen_name("sSS14validatingUTF8SSSgSPys4Int8VG_tcfC")
  public init?(validatingCString nullTerminatedUTF8: UnsafePointer<CChar>)
  
  @available(*, deprecated, renamed: "String.init(validatingCString:)")
  @_silgen_name("_swift_stdlib_legacy_String_validatingUTF8")
  @_alwaysEmitIntoClient
  public init?(validatingUTF8 cString: UnsafePointer<CChar>)
}
```

## Source Compatibility

This proposal consists mostly of additions, which are by definition source compatible.

The proposal includes the renaming of one function from `String.init?(validatingUTF8:)` to `String.init?(validatingCString:)`. The existing function name will be deprecated, producing a warning. A fixit will support an easy transition to the renamed version of the function.

## ABI Compatibility

This proposal adds new functions to the ABI.

The renamed function reuses the existing ABI entry point, making the change ABI-compatible.

## Implications on adoption

This feature requires a new version of the standard library.

## Alternatives considered

#### Initializers specifying the encoding by their argument label

For convenience and discoverability for the most common case, we originally proposed an initializer that specifies the UTF-8 input encoding as part of its argument label:

```swift
extension String {
  public init?(validatingAsUTF8 codeUnits: some Sequence<UTF8.CodeUnit>)
}
```

Reviewers and the Language Steering Group believed that this initializer does not carry its weight, and that the discoverability issues it sought to alleviate would best be solved by improved tooling.

#### Have `String.init?(validating: some Sequence<Int8>)` take a parameter typed as `some Sequence<CChar>`, or as a specific `Collection` of `CChar`

Defining this validating initializer in terms of `some Sequence<CChar>`  would produce a compile-time ambiguity on platforms where `CChar` is typealiased to `UInt8` rather than `Int8`. The reviewed proposal suggested defining it in terms of `UnsafeBufferPointer<CChar>`, since this parameter type would avoid such a compile-time ambiguity. The actual root of the problem is that `CChar` is a typealias instead of a separate type. Given this, discussions during the review period and by the Language Steering Group led to this initializer to be re-defined using `some Sequence<Int8>`. This solves the `CChar`-vs-`UInt8` interoperability issue at source-code level, and preserves as much flexibility as possible without ambiguities.

## Future directions

#### Throw an error containing information about a validation failure

When decoding a byte stream, obtaining the details of a validation failure would be useful in order to diagnose issues. We would like to provide this functionality, but the current input validation functionality is not well-suited for it. This is left as a future improvement.

#### Improve input-repairing initialization

There is only one initializer in the standard library for input-repairing initialization, and it suffers from a discoverability issue. We can add a more discoverable version specifically for the UTF-8 encoding, similarly to one of the additions proposed here.

#### Add normalization options

It is often desirable to normalize strings, but the standard library does not expose public API for doing so. We could add initializers that perform normalization, as well as mutating functions that perform normalization.

#### Other

- Add a (non-failable) initializer to create a `String` from `some Sequence<UnicodeScalar>`.
- Add API devoted to input validation specifically.

## Acknowledgements

Thanks to Michael Ilseman, Tina Liu and Quinn Quinn for discussions about input validation issues.

[SE-0027](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0027-string-from-code-units.md) by [Zachary Waldowski](https://github.com/zwaldowski) was reviewed in February 2016, covering similar ground. It was rejected at the time because the design of `String` had not been finalized. The name `String.init(validatingCString:)` was suggested as part of SE-0027. Lily Ballard later [pitched](https://forums.swift.org/t/21538) a renaming of `String.init(validatingUTF8:)`, citing consistency with other `String` API involving C strings.

