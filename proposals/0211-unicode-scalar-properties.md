# Add Unicode Properties to `Unicode.Scalar`

* Proposal: [SE-0211](0211-unicode-scalar-properties.md)
* Author: [Tony Allevato](https://github.com/allevato)
* Review Manager: [Ben Cohen](https://github.com/airspeedswift)
* Status: **Implemented (Swift 5.0)**
* Implementation: [apple/swift#15593](https://github.com/apple/swift/pull/15593)
* Decision Notes: [Acceptance](https://forums.swift.org/t/accepted-se-0211-add-unicode-properties-to-unicode-scalar/13857), [Update](https://forums.swift.org/t/update-se-0211-add-unicode-properties-to-unicode-scalar/59727)
* Previous Revision: [1](https://github.com/swiftlang/swift-evolution/blob/9b1c670206052f5c94bcb20df1c30c27a06e9755/proposals/0211-unicode-scalar-properties.md)

## Introduction

We propose adding a number of properties to the `Unicode.Scalar` type to support
both common and advanced text processing use cases, filling in a number of gaps
in Swift's text support compared to other programming languages.

Swift-evolution thread: [Adding Unicode properties to UnicodeScalar/Character](https://forums.swift.org/t/adding-unicode-properties-to-unicodescalar-character/9310)

## Motivation

The Swift `String` type, and related types like `Character` and
`Unicode.Scalar`, provide very rich support for Unicode-correct operations.
String comparisons are normalized, grapheme cluster boundaries are automatically
detected, and string contents can be easily accessed in terms of grapheme
clusters, code points, and UTF-8 and -16 code units.

However, when drilling down to lower levels, like individual code points (i.e.,
`Unicode.Scalar` elements), the current APIs are missing a number of fundamental
features available in other programming languages. `Unicode.Scalar` lacks the
ability to ask whether a scalar is upper/lowercase or what its upper/lowercase
mapping is, if it is a whitespace character, and so forth.

Without pulling in third-party code, users can currently import the
`Darwin/Glibc` module and access C functions like `isspace`, but these only work
with ASCII characters.

### Issues Linking with ICU

The Swift standard library uses the system's ICU libraries to implement its
Unicode support. A third-party developer may expect that they could also link
their application directly to the system ICU to access the functionality that
they need, but this proves problematic on both Apple and Linux platforms.

On Apple operating systems, `libicucore.dylib` is built with function renaming
disabled (function names lack the `_NN` version number suffix). This makes it
fairly straightforward to import the C APIs and call them from Swift without
worrying about which version the operating system is using.

Unfortunately, `libicucore.dylib` is considered to be private API for
submissions to the App Store, so applications doing this will be rejected.
Instead, users must built their own copy of ICU from source and link that into
their applications. This is significant overhead.

On Linux, system ICU libraries are built with function renaming enabled (the
default), so function names have the `_NN` version number suffix. Function
renaming makes it more difficult to use these APIs from Swift; even though the C
header files contain `#define`s that map function names like `u_foo_59` to
`u_foo`, these `#define`s are not imported into Swift—only the suffixed function
names are available. This means that Swift bindings would be fixed to a specific
version of the library without some other intermediary layer. Again, this is
significant overhead.

Therefore, this proposal not only fills in important gaps in the standard
library's capabilities, but removes a significant pain point for users who may
try to access that functionality through other means.

## Proposed solution

We propose adding a nested struct, `Unicode.Scalar.Properties`, which will
encapsulate many of the properties that the Unicode specification defines on
scalars. Supporting types, such as enums representing the values of certain
properties, will also be added to the `Unicode` enum "namespace."

### Scope of This Proposal

This proposal is restricted, by design, to add functionality to `Unicode.Scalar`
only. While we believe that some of the properties described here (and others)
would be valuable on `Character` as well, we have intentionally saved those for
a future proposal in order to keep this one small and focused. Such a future
proposal would likely depend on the design and implementation herein.

## Detailed design

The code snippets below reflect an elided sketch of the proposed public API
only. Full details can be found in the implementation pull request.

In general, the names of the properties inside the `Properties` struct are
derived directly from the names of the properties as they are defined in the
[Unicode Standard](http://unicode.org/reports/tr44/#Property_Index).

```swift
extension Unicode.Scalar {

  // NOT @_fixed_layout
  public struct Properties {
    // Remaining API is defined in the subsections below.
  }

  /// The value that encapsulates the properties exposed.
  public var properties: Properties { get }
}
```

### Boolean Properties

Each of the Boolean properties in the first block below would be implemented by
calling ICU's `u_hasBinaryProperty` function. The official Unicode name of each
property is indicated by the comment next to the computed properties below.

We propose supporting all of the Boolean properties that are currently available
using ICU's `u_hasBinaryProperty` that correspond to properties in the Unicode
Standard (but _not_ ICU-specific properties), with the following exceptions:

* `Grapheme_Link` is omitted because it is deprecated and equivalent to
  canonical combining class 9.
* `Hyphen` is omitted because is deprecated in favor of the `Line_Break`
  property.

```swift
extension Unicode.Scalar.Properties {

  public var isAlphabetic: Bool { get }    // Alphabetic
  public var isASCIIHexDigit: Bool { get }    // ASCII_Hex_Digit
  public var isBidiControl: Bool { get }    // Bidi_Control
  public var isBidiMirrored: Bool { get }    // Bidi_Mirrored
  public var isDash: Bool { get }    // Dash
  public var isDefaultIgnorableCodePoint: Bool { get }    // Default_Ignorable_Code_Point
  public var isDeprecated: Bool { get }    // Deprecated
  public var isDiacritic: Bool { get }    // Diacritic
  public var isExtender: Bool { get }    // Extender
  public var isFullCompositionExclusion: Bool { get }    // Full_Composition_Exclusion
  public var isGraphemeBase: Bool { get }    // Grapheme_Base
  public var isGraphemeExtend: Bool { get }    // Grapheme_Extend
  public var isHexDigit: Bool { get }    // Hex_Digit
  public var isIDContinue: Bool { get }    // ID_Continue
  public var isIDStart: Bool { get }    // ID_Start
  public var isIdeographic: Bool { get }    // Ideographic
  public var isIDSBinaryOperator: Bool { get }    // IDS_Binary_Operator
  public var isIDSTrinaryOperator: Bool { get }    // IDS_Trinary_Operator
  public var isJoinControl: Bool { get }    // Join_Control
  public var isLogicalOrderException: Bool { get }    // Logical_Order_Exception
  public var isLowercase: Bool { get }    // Lowercase
  public var isMath: Bool { get }    // Math
  public var isNoncharacterCodePoint: Bool { get }    // Noncharacter_Code_Point
  public var isQuotationMark: Bool { get }    // Quotation_Mark
  public var isRadical: Bool { get }    // Radical
  public var isSoftDotted: Bool { get }    // Soft_Dotted
  public var isTerminalPunctuation: Bool { get }    // Terminal_Punctuation
  public var isUnifiedIdeograph: Bool { get }    // Unified_Ideograph
  public var isUppercase: Bool { get }    // Uppercase
  public var isWhitespace: Bool { get }    // Whitespace
  public var isXIDContinue: Bool { get }    // XID_Continue
  public var isXIDStart: Bool { get }    // XID_Start
  public var isCaseSensitive: Bool { get }    // Case_Sensitive
  public var isSentenceTerminal: Bool { get }    // Sentence_Terminal (S_Term)
  public var isVariationSelector: Bool { get }    // Variation_Selector
  public var isNFDInert: Bool { get }    // NFD_Inert
  public var isNFKDInert: Bool { get }    // NFKD_Inert
  public var isNFCInert: Bool { get }    // NFC_Inert
  public var isNFKCInert: Bool { get }    // NFKC_Inert
  public var isSegmentStarter: Bool { get }    // Segment_Starter
  public var isPatternSyntax: Bool { get }    // Pattern_Syntax
  public var isPatternWhitespace: Bool { get }    // Pattern_White_Space
  public var isCased: Bool { get }    // Cased
  public var isCaseIgnorable: Bool { get }    // Case_Ignorable
  public var changesWhenLowercased: Bool { get }    // Changes_When_Lowercased
  public var changesWhenUppercased: Bool { get }    // Changes_When_Uppercased
  public var changesWhenTitlecased: Bool { get }    // Changes_When_Titlecased
  public var changesWhenCaseFolded: Bool { get }    // Changes_When_Casefolded
  public var changesWhenCaseMapped: Bool { get }    // Changes_When_Casemapped
  public var changesWhenNFKCCaseFolded: Bool { get }    // Changes_When_NFKC_Casefolded
  public var isEmoji: Bool { get }    // Emoji
  public var isEmojiPresentation: Bool { get }    // Emoji_Presentation
  public var isEmojiModifier: Bool { get }    // Emoji_Modifier
  public var isEmojiModifierBase: Bool { get }    // Emoji_Modifier_Base
}
```

### Case Mappings

The properties below provide full case mappings for scalars. Since a handful of
mappings result in multiple scalars (e.g., "ß" uppercases to "SS"), these
properties are `String`-valued, not `Unicode.Scalar`.

These properties are also common enough that they could be reasonably hoisted
out of `Unicode.Scalar.Properties` and made into instance properties directly on
`Unicode.Scalar`.

```swift
extension Unicode.Scalar.Properties {

  public var lowercaseMapping: String { get }  // u_strToLower
  public var titlecaseMapping: String { get }  // u_strToTitle
  public var uppercaseMapping: String { get }  // u_strToUpper
}
```

### Identification and Classification

We add the following properties for the purposes of identifying and categorizing
scalars.

The `Canonical_Combining_Class` property is somewhat unique in that the Unicode
standard provides names for some, but not all, of the 255 valid values. We
choose to represent this in Swift as a `RawRepresentable` `struct` that wraps
the raw integer value, while still being `Comparable` for the purposes of
implementing manual decomposition logic if necessary, and still providing the
named values through `static let` constants.

```swift
extension Unicode.Scalar.Properties {

  /// Corresponds to the `Age` Unicode property, when a code point was first
  /// defined.
  public var age: Unicode.Version? { get }

  /// Corresponds to the `Name` Unicode property.
  public var name: String? { get }

  /// Corresponds to the `Name_Alias` Unicode property.
  public var nameAlias: String? { get }

  /// Corresponds to the `General_Category` Unicode property.
  public var generalCategory: Unicode.GeneralCategory { get }

  /// Corresponds to the `Canonical_Combining_Class` Unicode property.
  public var canonicalCombiningClass: Unicode.CanonicalCombiningClass { get }
}

extension Unicode {

  /// Represents the version of Unicode in which a scalar was introduced.
  public typealias Version = (major: Int, minor: Int)

  /// General categories returned by
  /// `Unicode.Scalar.Properties.generalCategory`. Listed along with their
  /// two-letter code.
  public enum GeneralCategory {
    case uppercaseLetter  // Lu
    case lowercaseLetter  // Ll
    case titlecaseLetter  // Lt
    case modifierLetter  // Lm
    case otherLetter  // Lo

    case nonspacingMark  // Mn
    case spacingMark  // Mc
    case enclosingMark  // Me

    case decimalNumber  // Nd
    case letterlikeNumber  // Nl
    case otherNumber  // No

    case connectorPunctuation  //Pc
    case dashPunctuation  // Pd
    case openPunctuation  // Ps
    case closePunctuation  // Pe
    case initialPunctuation  // Pi
    case finalPunctuation  // Pf
    case otherPunctuation  // Po

    case mathSymbol  // Sm
    case currencySymbol  // Sc
    case modifierSymbol  // Sk
    case otherSymbol  // So

    case spaceSeparator  // Zs
    case lineSeparator  // Zl
    case paragraphSeparator  // Zp

    case control  // Cc
    case format  // Cf
    case surrogate  // Cs
    case privateUse  // Co
    case unassigned  // Cn
  }

  public struct CanonicalCombiningClass:
    Comparable, Hashable, RawRepresentable
  {
    public static let notReordered = CanonicalCombiningClass(rawValue: 0)
    public static let overlay = CanonicalCombiningClass(rawValue: 1)
    public static let nukta = CanonicalCombiningClass(rawValue: 7)
    public static let kanaVoicing = CanonicalCombiningClass(rawValue: 8)
    public static let virama = CanonicalCombiningClass(rawValue: 9)
    public static let attachedBelowLeft = CanonicalCombiningClass(rawValue: 200)
    public static let attachedBelow = CanonicalCombiningClass(rawValue: 202)
    public static let attachedAbove = CanonicalCombiningClass(rawValue: 214)
    public static let attachedAboveRight = CanonicalCombiningClass(rawValue: 216)
    public static let belowLeft = CanonicalCombiningClass(rawValue: 218)
    public static let below = CanonicalCombiningClass(rawValue: 220)
    public static let belowRight = CanonicalCombiningClass(rawValue: 222)
    public static let left = CanonicalCombiningClass(rawValue: 224)
    public static let right = CanonicalCombiningClass(rawValue: 226)
    public static let aboveLeft = CanonicalCombiningClass(rawValue: 228)
    public static let above = CanonicalCombiningClass(rawValue: 230)
    public static let aboveRight = CanonicalCombiningClass(rawValue: 232)
    public static let doubleBelow = CanonicalCombiningClass(rawValue: 233)
    public static let doubleAbove = CanonicalCombiningClass(rawValue: 234)
    public static let iotaSubscript = CanonicalCombiningClass(rawValue: 240)

    public let rawValue: UInt8

    public init(rawValue: UInt8)
  }
}
```

### Numerics

Many Unicode scalars have associated numeric values. These are not only the
common digits zero through nine, but also vulgar fractions and various other
linguistic characters and ideographs that have an innate numeric value. These
properties are exposed below. They can be useful for determining whether
segments of text contain numbers or non-numeric data, and can also help in the
design of algorithms to determine the values of such numbers.

```swift
extension Unicode.Scalar.Properties {

  /// Corresponds to the `Numeric_Type` Unicode property.
  public var numericType: Unicode.NumericType?

  /// Corresponds to the `Numeric_Value` Unicode property.
  public var numericValue: Double?
}

extension Unicode {

  public enum NumericType {
    case decimal
    case digit
    case numeric
  }
}
```

## Source compatibility

These changes are strictly additive. This proposal does not affect source
compatibility.

## Effect on ABI stability

These changes are strictly additive. This proposal does not affect the ABI of
existing language features.

## Effect on API resilience

The `Unicode.Scalar.Properties` struct is currently defined as a resilient
(non-`@_fixed_layout`) struct whose only stored property in the initial
implementation is the integer value of the scalar whose properties are being
retrieved. All other properties are computed properties, and new properties can
be added without breaking the ABI.

In the future, we may choose to cache certain properties if data show that they
are more frequently accessed than others and that there would be a meaningful
performance improvement by computing them early. It is too soon to make such a
determination now, however.

## Alternatives considered

### API Designs

We considered other representations for the Boolean properties of a scalar:

* A `BooleanProperty` enum with a case for each property, and a
  `Unicode.Scalar.hasProperty` method used to query it. This is very close to
  the underlying ICU C APIs and does not bloat the `Unicode.Scalar` API, but
  makes the kinds of queries users would commonly make less discoverable.
* A `Unicode.Scalar.properties` property whose type conforms to `OptionSet`.
  This would allow us to use the underlying property enum constants as the
  bit-shifts for the option set values, but there are already 64 Boolean
  properties defined by ICU. Since the underlying integral type is part of the
  public API/ABI of the option set, we would not be able to change it in the
  future without breaking compatibility.
* A `Unicode.Scalar.properties` property whose type is a `Set<BooleanProperty>`,
  but we would not be able to form this collection without querying all Boolean
  properties upon any access (the `OptionSet` solution above suffers the same
  problem). This would be needlessly inefficient in almost all usage.

We feel that by putting the properties into a separate
`Unicode.Scalar.Properties` struct, the large number of advanced properties does
not contribute to bloat of the main `Unicode.Scalar` API, and allows us to
cleanly represent not only Boolean properties but other types of properties with
ease.

### Naming

The names of the Boolean properties are all of the form `is<Unicode Property
Name>`, with the exception of a small number of properties whose names already
start with indicative verb forms and read as assertions (e.g.,
`changesWhenUppercased`). This leads to some technical and/or awkward property
names, like `isXIDContinue`.

We considered modifying these names to make them read more naturally like other
Swift APIs; for example, `extendsPrecedingScalar` instead of `isExtender`.
However, since these properties are intended for advanced users who are likely
already somewhat familiar with the Unicode Standard and its definitions, we
decided to keep the names directly derived from the Standard, which makes them
more discoverable to the intended audience.

### `isDefined` Property

The original version of this proposal also included a Boolean `isDefined`
property that was equivalent to the `u_isdefined` function from ICU, which
would evaluate to true for exactly the code points that are assigned (those
with general category other than "Cn"). In Swift, however, this would be
redundant and was thus dropped from the final implementation; its value would
differ from `Unicode.Scalar.Properties.generalCategory != .unassigned` only
for surrogate code points, which cannot be created as `Unicode.Scalar` values.
