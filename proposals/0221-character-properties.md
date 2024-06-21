# Character Properties

* Proposal: [SE-0221](0221-character-properties.md)
* Authors: [Michael Ilseman](https://github.com/milseman), [Tony Allevato](https://github.com/allevato)
* Review Manager: [Ben Cohen](https://github.com/airspeedswift)
* Status: **Implemented (Swift 5.0)**
* Implementation: [apple/swift#20520](https://github.com/apple/swift/pull/20520)
* Review: [Discussion thread](https://forums.swift.org/t/se-0221-character-properties/14686), [Announcement thread](https://forums.swift.org/t/accepted-with-modification-se-0221-character-properties/14944/2)
* Previous Revision: [1](https://github.com/swiftlang/swift-evolution/blob/fdb725c240033c5273860b0a66d2189d62a97608/proposals/0221-character-properties.md)

## Introduction

@allevato (a co-author here) proposed [Add Unicode Properties to Unicode.Scalar](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0211-unicode-scalar-properties.md), which exposes Unicode properties from the [Unicode Character Database](http://unicode.org/reports/tr44/). These are Unicode expert/enthusiast oriented properties that give a finer granularity of control and answer highly-technical and specific Unicody enquiries.

However, they are not ergonomic and Swift makes no attempt to clarify their interpretation or usage: meaning and proper interpretation is directly tied to the Unicode Standard and the version of Unicode available at run time. There’s some low-hanging ergo-fruit ripe for picking by exposing properties directly on `Character`.

Pitch thread: [Character and String properties](https://forums.swift.org/t/pitch-character-and-string-properties/11620)

## Motivation

`String` is a collection whose element is `Character`, which represents an [extended grapheme cluster](https://unicode.org/reports/tr29/#Grapheme_Cluster_Boundaries) (commonly just called “grapheme”).  This makes `Character` one of the first types encountered both by newcomers to Swift as well as by experienced Swift developers playing around in new domains (e.g. scripting). Yet `Character` exposes little functionality other than the ability to order it with respect to other characters, and a way to access the raw [Unicode scalar values](https://unicode.org/glossary/#unicode_scalar_value) that comprise it.

This proposal adds several queries to increase the usefulness of `Character` and approachability of programming in Swift. It tries to walk the fuzzy line between what Swift can give reasonably good answers to, and what would require the user to adopt more elaborate linguistic analysis frameworks or techniques.

## Proposed Solution
(Note that Unicode does not define properties on graphemes in general. Swift is defining its own semantics in terms of Unicode semantics derived from scalar properties, semantics on strings, or both)

### Character Properties

```swift
extension Character {
  /// Whether this Character is ASCII.
  @inlinable
  public var isASCII: Bool { ... }

  /// Returns the ASCII encoding value of this Character, if ASCII.
  ///
  /// Note: "\r\n" (CR-LF) is normalized to "\n" (LF), which will return 0x0A
  @inlinable
  public var asciiValue: UInt8? { ... }

  /// Whether this Character represents whitespace, including newlines.
  ///
  /// Examples:
  ///   * "\t" (U+0009 CHARACTER TABULATION)
  ///   * " " (U+0020 SPACE)
  ///   * U+2029 PARAGRAPH SEPARATOR
  ///   * U+3000 IDEOGRAPHIC SPACE
  ///
  public var isWhitespace: Bool { ... }

  /// Whether this Character represents a newline.
  ///
  /// Examples:
  ///   * "\n" (U+000A): LINE FEED (LF)
  ///   * "\r" (U+000D): CARRIAGE RETURN (CR)
  ///   * "\r\n" (U+000A U+000D): CR-LF
  ///   * U+0085: NEXT LINE (NEL)
  ///   * U+2028: LINE SEPARATOR
  ///   * U+2029: PARAGRAPH SEPARATOR
  ///
  public var isNewline: Bool { ... }

  /// Whether this Character represents a number.
  ///
  /// Examples:
  ///   * "7" (U+0037 DIGIT SEVEN)
  ///   * "⅚" (U+215A VULGAR FRACTION FIVE SIXTHS)
  ///   * "㊈" (U+3288 CIRCLED IDEOGRAPH NINE)
  ///   * "𝟠" (U+1D7E0 MATHEMATICAL DOUBLE-STRUCK DIGIT EIGHT)
  ///   * "๒" (U+0E52 THAI DIGIT TWO)
  ///
  public var isNumber: Bool { ... }

  /// Whether this Character represents a whole number. See
  /// `Character.wholeNumberValue`
  @inlinable
  public var isWholeNumber: Bool { ... }

  /// If this Character is a whole number, return the value it represents, else
  /// nil.
  ///
  /// Examples:
  ///   * "1" (U+0031 DIGIT ONE) => 1
  ///   * "५" (U+096B DEVANAGARI DIGIT FIVE) => 5
  ///   * "๙" (U+0E59 THAI DIGIT NINE) => 9
  ///   * "万" (U+4E07 CJK UNIFIED IDEOGRAPH-4E07) => 10_000
  ///
  public var wholeNumberValue: Int? { ... }

  /// Whether this Character represents a hexadecimal digit.
  ///
  /// Hexadecimal digits include 0-9, Latin letters a-f and A-F, and their
  /// fullwidth compatibility forms. To get their value, see
  /// `Character.hexadecimalDigitValue`
  @inlinable
  public var isHexadecimalDigit: Bool { ... }

  /// If this Character is a hexadecimal digit, returns the value it represents,
  /// else nil.
  ///
  /// Hexadecimal digits include 0-9, Latin letters a-f and A-F, and their
  /// fullwidth compatibility forms.
  public var hexadecimalDigitValue: Int? { ... }

  /// Whether this Character is a letter.
  ///
  /// Examples:
  ///   * "A" (U+0041 LATIN CAPITAL LETTER A)
  ///   * "é" (U+0065 LATIN SMALL LETTER E, U+0301 COMBINING ACUTE ACCENT)
  ///   * "ϴ" (U+03F4 GREEK CAPITAL THETA SYMBOL)
  ///   * "ڈ" (U+0688 ARABIC LETTER DDAL)
  ///   * "日" (U+65E5 CJK UNIFIED IDEOGRAPH-65E5)
  ///   * "ᚨ" (U+16A8 RUNIC LETTER ANSUZ A)
  ///
  public var isLetter: Bool { ... }

  /// Perform case conversion to uppercase
  ///
  /// Examples:
  ///   * "é" (U+0065 LATIN SMALL LETTER E, U+0301 COMBINING ACUTE ACCENT)
  ///     => "É" (U+0045 LATIN CAPITAL LETTER E, U+0301 COMBINING ACUTE ACCENT)
  ///   * "и" (U+0438 CYRILLIC SMALL LETTER I)
  ///     => "И" (U+0418 CYRILLIC CAPITAL LETTER I)
  ///   * "π" (U+03C0 GREEK SMALL LETTER PI)
  ///     => "Π" (U+03A0 GREEK CAPITAL LETTER PI)
  ///   * "ß" (U+00DF LATIN SMALL LETTER SHARP S)
  ///     => "SS" (U+0053 LATIN CAPITAL LETTER S, U+0053 LATIN CAPITAL LETTER S)
  ///
  /// Note: Returns a String as case conversion can result in multiple
  /// Characters.
  public func uppercased() -> String { ... }

  /// Perform case conversion to lowercase
  ///
  /// Examples:
  ///   * "É" (U+0045 LATIN CAPITAL LETTER E, U+0301 COMBINING ACUTE ACCENT)
  ///     => "é" (U+0065 LATIN SMALL LETTER E, U+0301 COMBINING ACUTE ACCENT)
  ///   * "И" (U+0418 CYRILLIC CAPITAL LETTER I)
  ///     => "и" (U+0438 CYRILLIC SMALL LETTER I)
  ///   * "Π" (U+03A0 GREEK CAPITAL LETTER PI)
  ///     => "π" (U+03C0 GREEK SMALL LETTER PI)
  ///
  /// Note: Returns a String as case conversion can result in multiple
  /// Characters.
  public func lowercased() -> String { ... }

  /// Whether this Character is considered uppercase.
  ///
  /// Uppercase Characters vary under case-conversion to lowercase, but not when
  /// converted to uppercase.
  ///
  /// Examples:
  ///   * "É" (U+0045 LATIN CAPITAL LETTER E, U+0301 COMBINING ACUTE ACCENT)
  ///   * "И" (U+0418 CYRILLIC CAPITAL LETTER I)
  ///   * "Π" (U+03A0 GREEK CAPITAL LETTER PI)
  ///
  @inlinable
  public var isUppercase: Bool { ... }

  /// Whether this Character is considered lowercase.
  ///
  /// Lowercase Characters vary under case-conversion to uppercase, but not when
  /// converted to lowercase.
  ///
  /// Examples:
  ///   * "é" (U+0065 LATIN SMALL LETTER E, U+0301 COMBINING ACUTE ACCENT)
  ///   * "и" (U+0438 CYRILLIC SMALL LETTER I)
  ///   * "π" (U+03C0 GREEK SMALL LETTER PI)
  ///
  @inlinable
  public var isLowercase: Bool { ... }

  /// Whether this Character changes under any form of case conversion.
  @inlinable
  public var isCased: Bool { ... }

  /// Whether this Character represents a symbol
  ///
  /// Examples:
  ///   * "®" (U+00AE REGISTERED SIGN)
  ///   * "⌹" (U+2339 APL FUNCTIONAL SYMBOL QUAD DIVIDE)
  ///   * "⡆" (U+2846 BRAILLE PATTERN DOTS-237)
  ///
  public var isSymbol: Bool { ... }

  /// Whether this Character represents a symbol used in mathematical formulas
  ///
  /// Examples:
  ///   * "+" (U+002B PLUS SIGN)
  ///   * "∫" (U+222B INTEGRAL)
  ///   * "ϰ" (U+03F0 GREEK KAPPA SYMBOL)
  ///
  /// Note: This is not a strict subset of isSymbol. This includes characters
  /// used both as letters and commonly in mathematical formulas. For example,
  /// "ϰ" (U+03F0 GREEK KAPPA SYMBOL) is considered a both mathematical symbol
  /// and a letter.
  ///
  public var isMathSymbol: Bool { ... }

  /// Whether this Character represents a currency symbol
  ///
  /// Examples:
  ///   * "$" (U+0024 DOLLAR SIGN)
  ///   * "¥" (U+00A5 YEN SIGN)
  ///   * "€" (U+20AC EURO SIGN)
  public var isCurrencySymbol: Bool { ... }

  /// Whether this Character represents punctuation
  ///
  /// Examples:
  ///   * "!" (U+0021 EXCLAMATION MARK)
  //   * "؟" (U+061F ARABIC QUESTION MARK)
  ///   * "…" (U+2026 HORIZONTAL ELLIPSIS)
  ///   * "—" (U+2014 EM DASH)
  ///   * "“" (U+201C LEFT DOUBLE QUOTATION MARK)
  ///
  public var isPunctuation: Bool { ... }
}
```

## Detailed Semantics and Rationale

Some fuzziness is inherent in modeling human writing systems and the rules of grapheme breaking allow for semantically meaningless, yet technically valid, graphemes. In light of all this, we make a best effort and try to discover some principle to follow. Principles are useful for evaluating tradeoffs, but are not hard rules that always lead to a single clear answer.

The closest applicable principle might be something similar to W3C’s [Principle of Tolerance](https://www.w3.org/DesignIssues/Principles.html), paraphrased as “Be liberal in what you accept, conservative in what you produce”. Character properties can be roughly grouped into those that “produce” specific values or behaviors, and those that “accept” graphemes under a fuzzy classification.

### Restrictive Properties

Properties that provide a clear interpretation or which the stdlib produces a specific value for should be restrictive. One example is `wholeNumberValue`. `wholeNumberValue` *produces* an `Int` from a `Character`, which means it needs to be *restrictive*, permitting only the graphemes with unambiguous whole number values. It only returns a value for single-scalar graphemes whose sole scalar has an integral numeric value. Thus, `wholeNumberValue` returns nil for “7̅” (7 followed by U+0305 COMBINING OVERLINE) as there is no clear interpretation of the value. Any attempt to produce a specific integer from “7̅” would be suspect.

Restrictive properties typically accept/reject based on an analysis of the entire grapheme.

* Values:  `isASCII` / `asciiValue`, `isWholeNumber` / `wholeNumberValue`, `isHexDigit` / `hexDigitValue`
* Casing:  `isUppercase` / `uppercased()`, `isLowercase` / `lowercased()`, `isCased`

### Permissive Properties

Where there is no clear interpretation or specific value to produce, we try to be as permissive as reasonable. For example, `isLetter` just queries the first scalar to see if it is “letter-like”, and thus handles unforeseeable combinations of a base letter-like scalar with subsequent combining, modifying, or extending scalars. `isLetter` merely answers a general (fuzzy) question, but doesn’t prescribe further interpretation.

Permissive APIs should in general be non-inlinable and their documentation may be less precise regarding details and corner cases. This allows for a greater degree of library evolution. Permissive properties typically accept/reject based on an analysis of part of the grapheme.

* Fuzzy queries: `isNumber`, `isLetter`, `isSymbol` / `isMathSymbol` / `isCurrencySymbol`, `isPunctuation`

#### Newlines and Whitespace

Newlines encompass more than hard line-breaks in traditional written language; they are common terminators for programmer strings. Whether a `Character` such as `"\n\u{301}"` (a newline with a combining accent over it) is a newline is debatable. Either interpretation can lead to inconsistencies. If true, then a program might skip the first scalar in a new entry (whatever such a combining scalar at the start could mean). If false, then a `String` with newline terminators inside of it would return false for `myStr.contains { $0.isNewline }`, which is counter-intuitive. The same is true of whitespace.

We recommend that the precise semantics of `isWhitespace` and `isNewline` be unspecified regarding graphemes consisting of leading whitespace/newlines followed by combining scalars.

## Source Compatibility
The properties on `Character` are strictly additive.

## Effect on ABI Stability
These changes are ABI-additive: they introduce new ABI surface area to keep stable.

The proposed solution includes recommended `@inlinable` annotations on properties which derive their value from other properties (thus benefitting from optimizations), or which are well-defined and stable under future Unicode versions (e.g. ASCII-related properties).

## Additions and Alternatives Considered

### Titlecase

Titlecase can be useful for some legacy scalars (ligatures) as well as for Strings when combined with word-breaking logic. However, it seems pretty obscure to surface on Character directly.

### String.Lines, String.Words

These have been deferred from this pitch to keep focus and await a more generalized lazy split collection.

### Rename Permissive `isFoo` to `hasFoo`

This was mentioned above in discussion of `isNewline` semantics and could also apply to `isWhitespace`. However, it would be awkward for `isNumber` or `isLetter`. What the behavior should be for exotic whitespace and newlines is heavily debatable. We’re sticking to `isNewline/isWhitespace` for now, but are open to argument.

### Design as `Character.has(OptionSet<…>, exclusively: …)`

There could be something valuable to glean from this, but we reject this approach as somewhat un-Swifty with a poor discovery experience, especially for new or casual users. It does, however, make the semantic distinctions above very explicit at the call site.

### Add Failable FixedWidthInteger/FloatingPoint Initializers Taking Character

In addition to (or perhaps instead of) properties like `wholeNumberValue`, add `Character`-based `FixedWidthInteger.init?(_:Character)`. Similarly `FloatingPoint.init?(_:Character)` which includes vulgar fractions and the like (if single-scalar, perhaps). However, these do not have direct counterparts in this pitch as named, at least without an explicit argument label clarifying their semantics.

We could consider adding something like `FixedWidthInteger.init?(hexDigit: Character)` and `FixedWithInteger.init?(wholeNumber: Character)` which correspond to `hexDigitValue` and `wholeNumberValue`, and similarly a counterpart for `String`. But, we don’t feel this carries its weight as surfaced directly at the top level of e.g. `Int`. We prefer to keep this avenue open for future directions involving more general number parsing and grapheme evaluation logic.

### Drop `isASCII/HexDigit/WholeNumber`: Check for `nil` Instead

This alternative is to drop `isASCII`, `isHexDigit`, and `isWholeNumber` and instead use `if let` or compare explicitly to `nil`.

We decided to provide these convenience properties both for discoverability as well as use in more complex expressions: `c.isHexDigit && c.isLetter`, `c.isASCII && c.isWhitespace`, etc. We don’t think they add significant weight or undue API surface area.

### Add `numericValue: Double?` in addition to, or instead of, `wholeNumberValue: Int?`. Alternatively, add a `rationalValue: (numerator: Int, denominator: Int)?`.

Unicode defines numeric values for whole numbers, hex digits, and rational numbers (vulgar fractions). As an implementation artifact (ICU only vends a double), Unicode.Scalar.Properties’s `numericValue` is a double rather than an enum of a rational or whole number. We could follow suit and add such a value to `Character`, restricted to single-scalar graphemes. We could also remove `wholeNumberValue`, letting users test if the double is integral. Alternatively or additionally, we could provide a `rationalValue` capable of handling non-whole-numbers.

As far as adding a `rationalValue` is concerned, we do not feel that support for vulgar fractions and other obscure Unicode scalars (e.g. baseball score-keeping) warrants an addition to `Character` directly. `wholeNumberValue` producing an Int is a more fluid solution than a Double which happens to be integral, so we’re hesitant to replace `wholeNumberValue` entirely with a `numericValue`. Since `numericValue` would only add utility for these obscure characters, we’re not sure if it’s worth adding.

Suggestions for alternative names for `wholeNumberValue`would be appreciated.
