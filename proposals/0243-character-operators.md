# Character Literal Operators

* Proposal: [SE-0243](0243-character-operators.md)
* Authors: [Dianna ma (“Taylor Swift”)](https://github.com/tayloraswift), [John Holdsworth](https://github.com/johnno1962)
* Review manager: [Ben Cohen](https://github.com/airspeedswift)
* Status: **Second review** 
* Implementation: [apple/swift#71749](https://github.com/apple/swift/pull/71749)
* Previous Revision: [1](https://github.com/apple/swift-evolution/blob/9713526f3423270c27082c620c75b2e5bc92050e/proposals/0243-codepoint-and-character-literals.md)
* Threads: [1](https://forums.swift.org/t/prepitch-character-integer-literals/10442) [2](https://forums.swift.org/t/se-0243-codepoint-and-character-literals/21188) [3](https://forums.swift.org/t/single-quoted-character-literals-why-yes-again/61898)

## Introduction

This proposal improves Swift's character-literal ergonomics. This support is fundamental not only to parsing tasks within the Swift language but also to tasks that require developers to extract and manipulate data. Areas that would benefit include handling domain-specific languages (DSLs) and parsing commonly-used data formats such as JSON. Any workflow based on lexical analysis or tokenization requirements will gain from this proposal.

The Swift community previously considered single-quote syntax for character literals. While working on Swift's Lexer code, another solution came to light.  Adding well-chosen operators to the Standard Library tidied up the Lexer implementation with minimal impact on the language. These operators didn't burn the single-quote for future reserved use, they served all the most pressing use-cases effectively and demonstrated a small but measurable performance improvement.

This improvement was validated through our work on [PR 2439](https://github.com/apple/swift-syntax/pull/2439#issuecomment-1922292277). The patch showcased how to streamline character-binary integer interchange for low level code. This proposal offers the same readable solution that seamlessly integrates with the established character and style of Swift. Additionally, it provides a slight performance boost, making it a valuable enhancement for performant code.

To see how the proposal simplifies code, consider how the PR above resulted in the following changes from:

```Swift
    switch self.previous {
     case UInt8(ascii: " "), UInt8(ascii: "\r"), UInt8(ascii: "\n"), UInt8(ascii: "\t"),  // whitespace
       UInt8(ascii: "("), UInt8(ascii: "["), UInt8(ascii: "{"),  // opening delimiters
       UInt8(ascii: ","), UInt8(ascii: ";"), UInt8(ascii: ":"),  // expression separators
```
to:

```
    switch self.previous {
     case " ", "\r", "\n", "\t",  // whitespace
       "(", "[", "{",  // opening delimiters
       ",", ";", ":",  // expression separators
```

And in other places from
```
if self.isAtStartOfFile, self.peek(at: 1) == UInt8(ascii: "!") {
```
to:
```
if self.isAtStartOfFile, self.peek(at: 1) == "!" {
```

## Motivation

Swift's existing character-literal constructs are hard to read and an effort to construct. Contorted expressions like `UInt8(ascii: "c")` and `UnicodeScalar("c").value` provide Swift's current entry points to the binary integer equivalent of unicode scalars.

Since `Data` is not always `UInt8` or `UInt32`, these frequently must be combined with casts. User ergonomics are crying out for improvement. Consider the previous version of our lexer code. To `switch` over a range of values, our implementation in [PR 2439](https://github.com/apple/swift-syntax/pull/2439#issuecomment-1922292277) was cluttered with ergonomically unsound expressions like the previously mentioned `UInt8(ascii: "x")`.

Swift deserves better.

Our proposed change has precedent. Swift allows you to define custom operators for both equivalence and the pattern matching used in `switch` statements and elsewhere. Adding binary operators allows direct comparisons between `Integer` types and Unicode scalars. This approach effectively compiles a more readable solution to the same results.

## Proposed solution

We propose to introduce the following code to "stdlib/public/core/UnicodeScalar.swift" in the Swift standard library:

```Swift
/// Extends `UInt8` to allow direct comparisons with double quoted literals.
extension UInt8 {
  /// Returns a Boolean indicating whether the `UInt8` is equal to the provided Unicode scalar.
  ///
  /// - Parameters:
  ///   - i: The `UInt8` value to compare.
  ///   - s: The Unicode scalar to compare against.
  /// - Returns: `true` when the `UInt8` is equal to the provided Unicode scalar; otherwise, `false`.
  @_transparent @_alwaysEmitIntoClient
  public static func == (i: Self, s: Unicode.Scalar) -> Bool {
    return i == UInt8(ascii: s)
  }

  /// Returns a Boolean indicating whether the `UInt8` is not equal to the provided Unicode scalar.
  ///
  /// - Parameters:
  ///   - i: The `UInt8` value to compare.
  ///   - s: The Unicode scalar to compare against.
  /// - Returns: `true` if the `UInt8` is not equal to the provided Unicode scalar; otherwise, `false`.
  @_transparent @_alwaysEmitIntoClient
  public static func != (i: Self, s: Unicode.Scalar) -> Bool {
    return i != UInt8(ascii: s)
  }

  /// Enables pattern matching of Unicode scalars in switch statements.
  ///
  /// - Parameters:
  ///   - s: The Unicode scalar to match.
  ///   - i: The `UInt8` value to match against.
  /// - Returns: `true` if the Unicode scalar matches the `UInt8` value; otherwise, `false`.
  @_transparent @_alwaysEmitIntoClient
  public static func ~= (s: Unicode.Scalar, i: Self) -> Bool {
    return i == UInt8(ascii: s)
  }
}

/// Extends `Optional<UInt8>` to allow direct comparisons with double quoted literals.
extension UInt8? {
  /// Returns a Boolean value indicating whether the optional `UInt8` is equal to the provided Unicode scalar.
  ///
  /// - Parameters:
  ///   - i: The optional `UInt8` value to compare.
  ///   - s: The Unicode scalar to compare against.
  /// - Returns: `true` if the optional `UInt8` is equal to the provided Unicode scalar; otherwise, `false`.
  @_transparent @_alwaysEmitIntoClient
  public static func == (i: Self, s: Unicode.Scalar) -> Bool {
    return i == UInt8(ascii: s)
  }

  /// Returns a Boolean value indicating whether the optional `UInt8` is not equal to the provided Unicode scalar.
  ///
  /// - Parameters:
  ///   - i: The optional `UInt8` value to compare.
  ///   - s: The Unicode scalar to compare against.
  /// - Returns: `true` if the optional `UInt8` is not equal to the provided Unicode scalar; otherwise, `false`.
  @_transparent @_alwaysEmitIntoClient
  public static func != (i: Self, s: Unicode.Scalar) -> Bool {
    return i != UInt8(ascii: s)
  }

  /// Allows pattern matching of Unicode scalars in switch statements.
  ///
  /// - Parameters:
  ///   - s: The Unicode scalar to match.
  ///   - i: The optional `UInt8` value to match against.
  /// - Returns: `true` if the Unicode scalar matches the optional `UInt8` value; otherwise, `false`.
  @_transparent @_alwaysEmitIntoClient
  public static func ~= (s: Unicode.Scalar, i: Self) -> Bool {
    return i == UInt8(ascii: s)
  }
}

/// Extends `Array` where Element is a FixedWidthInteger, providing initialization from a string of Unicode scalars.
extension Array where Element: FixedWidthInteger {
  /// Initializes an array of Integers with Unicode scalars represented by the provided string.
  ///
  /// - Parameter scalars: A string containing Unicode scalars.
  @inlinable @_alwaysEmitIntoClient @_unavailableInEmbedded
  public init(scalars: String) {
    self.init(scalars.unicodeScalars.map { Element(unicode: $0) })
  }
}

/// Extends `FixedWidthInteger` providing initialization from a Unicode scalar.
extension FixedWidthInteger {
  /// Initializes a FixedWidthInteger with the value of the provided Unicode scalar.
  ///
  /// - Parameter unicode: The Unicode scalar to initialize from.
  /// - Note: Construct with value `v.value`.
  @inlinable @_alwaysEmitIntoClient
  public init(unicode v: Unicode.Scalar) {
    _precondition(v.value <= Self.max,
                  "Code point value does not fit into type")
    self = Self(v.value)
  }
}
```

This last initializer is optional. It provides an alternate to the existing `IntX(UncodeScalar("c").value)` incantation currently needed for non-ASCII code points.

## Source compatibility

Our proposed operator suite is additive. After running the existing test suite, it does change diagnostics on a limited part of invalid pattern matching code. We believe this diagnostic information was already flawed, and the change inconsequential.

## Effect on ABI stability

Each new operator has been annotated with `@_alwaysEmitIntoClient`. Any code that adopts these operators will back-port to versions of the Swift runtime before these operators were added.

## Effect on API resilience

The operators are simple and focused. We don't anticipate the need to evolve their ABI.

## Alternatives considered

This proposal emerges from a history of consideration. This scaled-back proposal presents less collateral impact on the language than previously reviewed proposals. At the same time, it satisfies the most important use cases.

Our proposed approach embraces Swift's existing language features rather than changing the language to reach its solution. We believe this enhancement is sufficiently useful to merit inclusion in the Standard Library. It will support and improve Swift's tooling and provide better ergonomics for Swift's user base without forcing Swift adopters to write their own solutions. Its inclusion will promote discovery, providing a feature that people might have expected to "simply work".