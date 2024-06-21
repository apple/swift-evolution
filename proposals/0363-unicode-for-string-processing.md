# Unicode for String Processing

* Proposal: [SE-0363](0363-unicode-for-string-processing.md)
* Authors: [Nate Cook](https://github.com/natecook1000), [Alejandro Alonso](https://github.com/Azoy)
* Review Manager: [Ben Cohen](https://github.com/airspeedswift)
* Status: **Implemented (Swift 5.7)**
* Implementation: [apple/swift-experimental-string-processing][repo]
* Review: ([pitch](https://forums.swift.org/t/pitch-unicode-for-string-processing/56907)), ([review](https://forums.swift.org/t/se-0363-unicode-for-string-processing/58520)), ([acceptance](https://forums.swift.org/t/accepted-se-0363-unicode-for-string-processing/59998))

### Version History

- Version 1: Initial version
- Version 2: 
  - Improved option method API names
  - Added Unicode property APIs to match regex syntax
  - Added `CharacterClass.noneOf(_:)` and sequence-based `init`
  - Clarified default state of options
  - Added detail around switching semantic modes
  - Added detail about Unicode property matching in character mode
  - Revised details of custom character class matching
  - Removed `\O`/`.anyUnicodeScalar`

### Table of Contents

  - [Introduction](#introduction)
  - [Motivation](#motivation)
  - [Proposed solution](#proposed-solution)
  - [Detailed design](#detailed-design)
    - [Options](#options)
      - [Case insensitivity](#case-insensitivity)
      - [Single line mode (`.` matches newlines)](#single-line-mode--matches-newlines)
      - [Multiline mode](#multiline-mode)
      - [ASCII-only character classes](#ascii-only-character-classes)
      - [Unicode word boundaries](#unicode-word-boundaries)
      - [Matching semantic level](#matching-semantic-level)
      - [Default repetition behavior](#default-repetition-behavior)
    - [Character Classes](#character-classes)
      - [“Any”](#any)
      - [Digits](#digits)
      - ["Word" characters](#word-characters)
      - [Whitespace and newlines](#whitespace-and-newlines)
      - [Unicode properties](#unicode-properties)
      - [POSIX character classes: `[:NAME:]`](#posix-character-classes-name)
      - [Custom classes](#custom-classes)
  - [Source compatibility](#source-compatibility)
  - [Effect on ABI stability](#effect-on-abi-stability)
  - [Effect on API resilience](#effect-on-api-resilience)
  - [Future directions](#future-directions)
  - [Alternatives considered](#alternatives-considered)

## Introduction

This proposal describes `Regex`'s rich Unicode support during regex matching, along with the character classes and options that define and modify that behavior.

This proposal is one component of a larger [regex-powered string processing initiative](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0350-regex-type-overview.md). For the status of each proposal, [see this document](https://github.com/apple/swift-experimental-string-processing/blob/main/Documentation/Evolution/ProposalOverview.md) — discussion of other facets of the overall regex design is out of scope of this proposal and better discussed in the most relevant review.

## Motivation

Swift's `String` type provides, by default, a view of `Character`s or [extended grapheme clusters][graphemes] whose comparison honors [Unicode canonical equivalence][canoneq]. Each character in a string can be composed of one or more Unicode scalar values, while still being treated as a single unit, equivalent to other ways of formulating the equivalent character:

```swift
let str = "Cafe\u{301}" // "Café"
str == "Café"           // true
str.dropLast()          // "Caf"
str.last == "é"         // true (precomposed e with acute accent)
str.last == "e\u{301}"  // true (e followed by composing acute accent)
```

This default view is fairly novel. Most languages that support Unicode strings generally operate at the Unicode scalar level, and don't provide the same affordance for operating on a string as a collection of grapheme clusters. In Python, for example, Unicode strings report their length as the number of scalar values, and don't use canonical equivalence in comparisons:

```python
cafe = u"Cafe\u0301"
len(cafe)                     # 5
cafe == u"Café"               # False
```

Existing regex engines follow this same model of operating at the Unicode scalar level. To match canonically equivalent characters, or have equivalent behavior between equivalent strings, you must normalize your string and regex to the same canonical format.

```python
# Matches a four-element string
re.match(u"^.{4}$", cafe)     # None
# Matches a string ending with 'é'
re.match(u".+é$", cafe)       # None

cafeComp = unicodedata.normalize("NFC", cafe)
re.match(u"^.{4}$", cafeComp) # <re.Match object...>
re.match(u".+é$", cafeComp)   # <re.Match object...>
```

With Swift's string model, this behavior would surprising and undesirable — Swift's default regex semantics must match the semantics of a `String`.

<details><summary>Other engines</summary>

Other regex engines match character classes (such as `\w` or `.`) at the Unicode scalar value level, or even the code unit level, instead of recognizing grapheme clusters as characters. When matching the `.` character class, other languages will only match the first part of an `"e\u{301}"` grapheme cluster. Some languages, like Perl, Ruby, and Java, support an additional `\X` metacharacter, which explicitly represents a single grapheme cluster.

| Matching  `"Cafe\u{301}"` | Pattern: `^Caf.` | Remaining | Pattern:  `^Caf\X` | Remaining |
|---|---|---|---|---|
| C#, Rust, Go, Python | `"Cafe"` | `"´"` | n/a | n/a |
| NSString, Java, Ruby, Perl | `"Cafe"` | `"´"` | `"Café"` | `""` |

Other than Java's `CANON_EQ` option, the vast majority of other languages and engines are not capable of comparing with canonical equivalence.

</details>

## Proposed solution

In a regex's simplest form, without metacharacters or special features, matching behaves like a test for equality. A string always matches a regex that simply contains the same characters.

```swift
let str = "Cafe\u{301}"     // "Café"
str.contains(/Café/)        // true
```

From that point, small changes continue to comport with the element counting and comparison expectations set by `String`:

```swift
str.contains(/Caf./)        // true
str.contains(/.+é/)         // true
str.contains(/.+e\u{301}/)  // true
str.contains(/\w+é/)        // true
```

For compatibility with other regex engines and the flexibility to match at both `Character` and Unicode scalar level, you can switch between matching levels for an entire regex or within select portions. This powerful capability provides the expected default behavior when working with strings, while allowing you to drop down for Unicode scalar-specific matching.

By default, literal characters and Unicode scalar values (e.g. `\u{301}`) are coalesced into characters the same way as a normal string, as shown above. Metacharacters, like `.` and `\w`, and custom character classes each match a single element at the current matching level.

For example, these matches fail, because by the time the engine attempts to match the "`\u{301}`" Unicode scalar literal in the regex, the full `"é"` character in `str` has been matched, even though that character is made up of two Unicode scalar values:

```swift
str.contains(/Caf.\u{301}/)    // false - `.` matches "é" character
str.contains(/Caf\w\u{301}/)   // false - `\w` matches "é" character
str.contains(/.+\u{301}/)      // false - `.+` matches each character
```

Alternatively, we can drop down to use Unicode scalar semantics if we want to match specific Unicode sequences. For example, these regexes match an `"e"` followed by any modifier with the specified parameters:

```swift
str.contains(/e[\u{300}-\u{314}]/.matchingSemantics(.unicodeScalar))
// true - matches an "e" followed by a Unicode scalar in the range U+0300 - U+0314
str.contains(/e\p{Nonspacing Mark}/.matchingSemantics(.unicodeScalar))
// true - matches an "e" followed by a Unicode scalar with general category "Nonspacing Mark"
```

Matching in Unicode scalar mode is analogous to comparing against a string's `UnicodeScalarView` — individual Unicode scalars are matched without combining them into characters or testing for canonical equivalence.

```swift
str.contains(/Café/.matchingSemantics(.unicodeScalar))
// false - "e\u{301}" doesn't match with /é/
str.contains(/Cafe\u{301}/.matchingSemantics(.unicodeScalar))
// true - "e\u{301}" matches with /e\u{301}/
```

Swift's `Regex` follows the level 2 guidelines for Unicode support in regular expressions described in [Unicode Technical Standard #18][uts18], with support for Unicode character classes, canonical equivalence, grapheme cluster matching semantics, and level 2 word boundaries enabled by default. In addition to selecting the matching semantics, `Regex` provides options for selecting different matching behaviors, such as ASCII character classes or Unicode scalar semantics, which corresponds more closely with other regex engines.

## Detailed design

First, we'll discuss the options that let you control a regex's behavior, and then explore the character classes that define the your pattern.

As detailed below, there are a few differences in defaults between Swift's `Regex` and the typical regex engine. In particular:

- `Regex` matches at the Swift `Character` level, instead of matching Unicode scalars, UTF-16 code units, or bytes. A regex that deliberately matches multi-scalar characters may need to switch to Unicode scalar semantics.
- `Regex` uses "default" word boundaries, instead of "simple" word boundaries. A regex that expects `\b` to always match the boundary between a word character (`\w`) and a non-word character (`\W`) may need to switch to simple word boundaries.
- For multi-line regex literals, extended syntax is automatically enabled, which ignores whitespace both in patterns and within custom character classes. To use semantic whitespace, you can temporarily disable extended mode (`(?-x:...)`), quote a section of your pattern (`\Q...\E`), or escape a space explicitly (`a\ b`).

### Options

Options can be enabled and disabled in two different ways: as part of [regex internal syntax][internals], or applied as methods when declaring a `Regex`. For example, both of these `Regex`es are declared with case insensitivity:

```swift
let regex1 = /(?i)banana/
let regex2 = Regex {
    "banana"
}.ignoresCase()
```

Because the `ignoresCase()` option method is defined on `Regex`, you can always use the more readable option-setting interface in conjunction with regex literals or run-time compiled `Regex`es:

```swift
let regex3 = /banana/.ignoresCase()
```

Calling an option-setting method like `ignoresCase()` acts like wrapping the callee in an option-setting group `(?:...)`. That is, while it sets the behavior for the callee, it doesn’t override options that are applied to more specific regions. In this example, the middle `"na"` in `"banana"` matches case-sensitively, despite the outer call to `ignoresCase()`:

```swift
let regex4 = Regex {
    "ba"
    Regex { "na" }.ignoresCase(false)
    "na"
}.ignoresCase()

"banana".contains(regex4)     // true
"BAnaNA".contains(regex4)     // true
"BANANA".contains(regex4)     // false

// Equivalent to:
let regex5 = /(?i)ba(?-i:na)na/
```

The options that `Regex` supports are shown in the table below, in three groups: Options that affect matching behavior for _both regex syntax and APIs_, options that affect the matching behavior of _regex syntax only_, and options with _structural_ or _syntactic_ effects that are only supported through regex syntax.

| **Matching Behavior**        |                |                                 | Default            |
|------------------------------|----------------|---------------------------------|--------------------|
| Case insensitivity           | `(?i)`         | `ignoresCase()`                 | disabled           |
| ASCII-only character classes | `(?DSWP)`      | `asciiOnlyClasses(_:)`          | `.none`            |
| Unicode word boundaries      | `(?w)`         | `wordBoundaryKind(_:)`          | `.default`         |
| Semantic level               | n/a            | `matchingSemantics(_:)`         | `.graphemeCluster` |
| Default repetition behavior  | n/a            | `defaultRepetitionBehavior(_:)` | `.eager`           |
| **Regex Syntax Only**        |                |                                 |                    |
| Single-line mode             | `(?s)`         | `dotMatchesNewlines()`          | disabled           |
| Multi-line mode              | `(?m)`         | `anchorsMatchNewlines()`        | disabled           |
| Swap eager/reluctant         | `(?U)`         | n/a                             | disabled           |
| **Structural/Syntactic**     |                |                                 |                    |
| Extended syntax              | `(?x)`,`(?xx)` | n/a                             | `xx` enabled in multi-line regex literals; otherwise, off |
| Named captures only          | `(?n)`         | n/a                             | disabled           |

#### Case insensitivity

Regexes perform case sensitive comparisons by default. The `i` option or the `ignoresCase(_:)` method enables case insensitive comparison.

```swift
let str = "Café"
	
str.firstMatch(of: /CAFÉ/)               // nil
str.firstMatch(of: /(?i)CAFÉ/)           // "Café"
str.firstMatch(of: /cAfÉ/.ignoresCase()) // "Café"
```

Case insensitive matching uses case folding to ensure that canonical equivalence continues to operate as expected.

**Regex syntax:** `(?i)...` or `(?i:...)`

**Standard Library API:**

```swift
extension Regex {
    /// Returns a regular expression that ignores casing when matching.
    public func ignoresCase(_ ignoresCase: Bool = true) -> Regex<RegexOutput>
}
```

#### ASCII-only character classes

With one or more of these options enabled, the default character classes match only ASCII values instead of the full Unicode range of characters. Four options are included in this group:

* Regex syntax `(?D)`: Match only ASCII members for `\d`, `[:digit:]`, and `CharacterClass.digit`.
* Regex syntax `(?S)`: Match only ASCII members for `\s`, `[:space:]`, and any of the whitespace-representing `CharacterClass` members.
* Regex syntax `(?W)`: Match only ASCII members for `\w`, `[:word:]`, and `CharacterClass.word`. Also only considers ASCII characters for `\b`, `\B`, and `Anchor.wordBoundary`.
* Regex syntax `(?D)`: Match only ASCII members for all POSIX properties (including `digit`, `space`, and `word`).

This option affects the built-in character classes listed in the "Character Classes" section below. When one or more of these options is enabled, the set of characters matched by those character classes is constrained to the ASCII character set. For example, `CharacterClass.hexDigit` usually matches `0...9`, `a-f`, and `A-F`, in either the ASCII or half-width variants. When the `(?D)` or `.asciiOnlyClasses(.digit)` options are enabled, only the ASCII characters are matched.

```swift
let str = "0x35AB"
str.contains(/0x(\d+)/.asciiOnlyClasses())
```

**Regex syntax:** `(?DSWP)...` or `(?DSWP...)`

**Standard Library API:**

```swift
extension Regex {
    /// Returns a regular expression that only matches ASCII characters as digits.
    public func asciiOnlyClasses(_ kinds: RegexCharacterClassKind = .all) -> Regex<RegexOutput>
}

/// A built-in regex character class kind.
///
/// Pass one or more `RegexCharacterClassKind` classes to `asciiOnlyClasses(_:)`
/// to control whether character classes match any character or only members
/// of the ASCII character set.
public struct RegexCharacterClassKind: OptionSet, Hashable {
    public var rawValue: Int { get }
  
    /// Regex digit-matching character classes, like `\d`, `[:digit:]`, and
    /// `\p{HexDigit}`.
    public static var digit: RegexCharacterClassKind { get }
  
    /// Regex whitespace-matching character classes, like `\s`, `[:space:]`,
    /// and `\p{Whitespace}`.
    public static var whitespace: RegexCharacterClassKind { get }
  
    /// Regex word character-matching character classes, like `\w`.
    public static var wordCharacter: RegexCharacterClassKind { get }
  
    /// All built-in regex character classes.
    public static var all: RegexCharacterClassKind { get }
  
    /// No built-in regex character classes.
    public static var none: RegexCharacterClassKind { get }
}
```

#### Unicode word boundaries

By default, matching word boundaries with the `\b` and `Anchor.wordBoundary` anchors uses Unicode _default word boundaries,_ specified as [Unicode level 2 regular expression support][level2-word-boundaries]. 

Disabling the `w` option switches to _[simple word boundaries][level1-word-boundaries],_ finding word boundaries at points in the input where `\b\B` or `\B\b` match. Depending on the other matching options that are enabled, this may be more compatible with the behavior other regex engines.

As shown in this example, the default matching behavior finds the whole first word of the string, while the match with simple word boundaries stops at the apostrophe:

```swift
let str = "Don't look down!"
	
str.firstMatch(of: /D\S+\b/)
// "Don't"
str.firstMatch(of: /D\S+\b/.wordBoundaryKind(.simple))
// "Don"
```

You can see more differences between level 1 (simple) and level 2 (default) word boundaries in the following table, generated by calling `matches(of: /\b.+\b/)` on the strings in the first column:

| Example             | Level 1                         | Level 2                                   |
|---------------------|---------------------------------|-------------------------------------------|
| I can't do that.    | ["I", "can", "t", "do", "that"] | ["I", "can't", "do", "that", "."]         |
| 🔥😊👍                 | ["🔥😊👍"]                         | ["🔥", "😊", "👍"]                           |
| 👩🏻👶🏿👨🏽🧑🏾👩🏼          | ["👩🏻👶🏿👨🏽🧑🏾👩🏼"]                  | ["👩🏻", "👶🏿", "👨🏽", "🧑🏾", "👩🏼"]            |
| 🇨🇦🇺🇸🇲🇽              | ["🇨🇦🇺🇸🇲🇽"]                      | ["🇨🇦", "🇺🇸", "🇲🇽"]                        |
| 〱㋞ツ              | ["〱", "㋞", "ツ"]              | ["〱㋞ツ"]                                |
| hello〱㋞ツ         | ["hello〱", "㋞", "ツ"]         | ["hello", "〱㋞ツ"]                       |
| 나는 Chicago에 산다 | ["나는", "Chicago에", "산다"]   | ["나", "는", "Chicago", "에", "산", "다"] |
| 眼睛love食物        | ["眼睛love食物"]                | ["眼", "睛", "love", "食", "物"]          |
| 아니ㅋㅋㅋ네        | ["아니ㅋㅋㅋ네"]                | ["아", "니", "ㅋㅋㅋ", "네"]              |
| Re:Zero             | ["Re", "Zero"]                  | ["Re:Zero"]                               |
| \u{d}\u{a}          | ["\u{d}", "\u{a}"]              | ["\u{d}\u{a}"]                            |
| €1 234,56           | ["1", "234", "56"]              | ["€", "1", "234,56"]                      |


**Regex syntax:** `(?-w)...` or `(?-w...)`

**Standard Library API:**

```swift
extension Regex {
  /// Returns a regular expression that uses the specified word boundary algorithm.
  ///
  /// A simple word boundary is a position in the input between two characters
  /// that match `/\w\W/` or `/\W\w/`, or between the start or end of the input
  /// and `\w` character. Word boundaries therefore depend on the option-defined
  /// behavior of `\w`.
  ///
  /// The default word boundaries use a Unicode algorithm that handles some cases
  /// better than simple word boundaries, such as words with internal
  /// punctuation, changes in script, and Emoji.
  public func wordBoundaryKind(_ wordBoundaryKind: RegexWordBoundaryKind) -> Regex<RegexOutput>
}

public struct RegexWordBoundaryKind: Hashable {
  /// A word boundary algorithm that implements the "simple word boundary"
  /// Unicode recommendation.
  ///
  /// A simple word boundary is a position in the input between two characters
  /// that match `/\w\W/` or `/\W\w/`, or between the start or end of the input
  /// and a `\w` character. Word boundaries therefore depend on the option-
  /// defined behavior of `\w`.
  public static var simple: Self { get }

  /// A word boundary algorithm that implements the "default word boundary"
  /// Unicode recommendation.
  ///
  /// Default word boundaries use a Unicode algorithm that handles some cases
  /// better than simple word boundaries, such as words with internal
  /// punctuation, changes in script, and Emoji.
  public static var default: Self { get }
}
```

#### Matching semantic level

To support both matching on `String`'s default character-by-character view and more broadly-compatible Unicode scalar-based matching, you can select a matching level for an entire regex or a portion of a regex constructed with the `RegexBuilder` API.

When matching with *grapheme cluster semantics* (the default), metacharacters like `.` and `\w`, custom character classes, and character class instances like `.any` match a grapheme cluster, corresponding with the default string representation. In addition, matching with grapheme cluster semantics compares characters using their canonical representation, corresponding with the way comparing strings for equality works.

When matching with *Unicode scalar semantics*, metacharacters and character classes match a single Unicode scalar value, even if that scalar comprises part of a grapheme cluster. Canonical representations are _not_ used, corresponding with the way comparison would work when using a string's `UnicodeScalarView`.

These specific levels of matching, and the options to switch between them, are unique to Swift, but not unprecedented in other regular expression engines. Several engines, including Perl, Java, and ICU-based engines like `NSRegularExpression`, support the `\X` metacharacter for matching a grapheme cluster within otherwise Unicode scalar semantic matching. Rust has a related concept in its [`regex::bytes` type][regexbytes], which matches over abitrary bytes by default but allows switching into Unicode mode for segments of the regular expression.

These semantic levels lead to different results when working with strings that have characters made up of multiple Unicode scalar values, such as Emoji or decomposed characters. In the following example, `queRegex` matches any 3-character string that begins with `"q"`.

```swift
let composed = "qué"
let decomposed = "que\u{301}"
	
let queRegex = /^q..$/
	
print(composed.contains(queRegex))
// Prints "true"
print(decomposed.contains(queRegex))
// Prints "true"
```

When using Unicode scalar semantics, however, the regex only matches the composed version of the string, because each `.` matches a single Unicode scalar value.

```swift
let queRegexScalar = queRegex.matchingSemantics(.unicodeScalar)
print(composed.contains(queRegexScalar))
// Prints "true"
print(decomposed.contains(queRegexScalar))
// Prints "false"
```

The index boundaries of the overall match and capture groups are affected by the matching semantic level. With grapheme cluster semantics, the start and end index of the overall match and each capture is `Character`-aligned. Matching with Unicode scalar semantics, on the other hand, can yield string indices that aren't aligned to character boundaries. Take care when using indices that aren't aligned with grapheme cluster boundaries, as they may have to be rounded to a boundary if used in a `String` instance.

```swift
let family = "👨‍👨‍👧‍👦 is a family"

// Grapheme-cluster mode: Yields a character
let firstCharacter = /^./
let characterMatch = family.firstMatch(of: firstCharacter)!.output
print(characterMatch)
// Prints "👨‍👨‍👧‍👦"

// Unicode-scalar mode: Yields only part of a character
let firstUnicodeScalar = /^./.matchingSemantics(.unicodeScalar)
let unicodeScalarMatch = family.firstMatch(of: firstUnicodeScalar)!.output
print(unicodeScalarMatch)
// Prints "👨"

// The end of `unicodeScalarMatch` is not aligned on a character boundary
print(unicodeScalarMatch.endIndex == family.index(after: family.startIndex))
// Prints "false"
```

When there is a boundary between Unicode scalar semantic and grapheme scalar semantic matching in the middle of a regex, an implicit grapheme cluster boundary assertion is added at the start of the grapheme scalar semantic section. That is, the two regexes in the following example are equivalent; each matches a single "word" scalar, followed by a combining mark scalar, followed by one or more grapheme clusters.

```swift
let explicit = Regex {
    Regex {
        CharacterClass.word
        CharacterClass.generalCategory(.combiningMark)
    }.matchingSemantics(.unicodeScalar)
    Anchor.graphemeClusterBoundary       // explicit grapheme cluster boundary
    OneOrMore(.any)
}

let implicit = Regex {
    Regex {
        CharacterClass.word
        CharacterClass.generalCategory(.combiningMark)
    }.matchingSemantics(.unicodeScalar)
    OneOrMore(.any)
}

try implicit.wholeMatch(in: "e\u{301} abc")           // match
try implicit.wholeMatch(in: "e\u{301}\u{327} abc")    // no match
```

The second call to `wholeMatch(in:)` fails because at the point the matching engine exits the inner regex, the matching position is still in the middle of the `"e\u{301}\u{327}` character. This implicit grapheme cluster boundary assertion maintains the guarantee that capture groups over grapheme cluster semantic sections will have valid character-aligned indices.

If a regex starts or ends with a Unicode scalar semantic section, there is no assertion added at the start or end of the pattern. Consider the following regex, which has Unicode scalars for the entire pattern except for a section in the middle that matches a purple heart emoji. When applied to a string with a multi-scalar character before or after the `"💜"`, the resulting match includes a partial character at its beginning and end.

```swift
let regex = Regex {
    CharacterClass.any
    Regex {  // <-- Implicit grapheme cluster boundary assertion, as above
        CharacterClass.binaryProperty(\.isEmoji)
    }.matchingSemantics(.graphemeCluster)
    CharacterClass.any
}.matchingSemantics(.unicodeScalar)

let borahae = "태형💜아미"    // Note: These hangeul characters are decomposed
if let match = borahae.firstMatch(of: regex) {
    print(match.0)
}
// Prints "ᆼ💜ᄋ"
```

Boundaries from a grapheme cluster section into a Unicode scalar also imply a grapheme cluster boundary, but in this case no assertion is needed. This boundary is an emergent property of the fact that under grapheme cluster semantics, matching always happens one character at a time.

**Standard Library API:**

```swift
extension Regex {
  /// Returns a regular expression that matches with the specified semantic
  /// level.
  public func matchingSemantics(_ semanticLevel: RegexSemanticLevel) -> Regex<RegexOutput>
}
	
public struct RegexSemanticLevel: Hashable {
  /// Match at the default semantic level of a string, where each matched
  /// element is a `Character`.
  public static var graphemeCluster: RegexSemanticLevel
  
  /// Match at the semantic level of a string's `UnicodeScalarView`, where each
  /// matched element is a `UnicodeScalar` value.
  public static var unicodeScalar: RegexSemanticLevel
}
```

#### Default repetition behavior

The `defaultRepetitionBehavior(_:)` method lets you set the default behavior for all quantifiers that don't explicitly provide their own behavior. For example, you can make all quantifiers behave possessively, eliminating any quantification-caused backtracking. This option applies both to quanitifiers in regex syntax that don't include an additional `?` or `+` (indicating reluctant or possessive quantification, respectively) and quantifiers in `RegexBuilder` syntax without an explicit behavior parameter.

In the following example, both regexes use possessive quantification:

```swift
let regex1 = /[0-9a-f]+\s*$/.defaultRepetitionBehavior(.possessive)

let regex2 = Regex {
    OneOrMore {
        CharacterClass.anyOf(
            "0"..."9",
            "a"..."f"
        )
    }
    ZeroOrMore(.whitespace)
    Anchor.endOfInput
}.defaultRepetitionBehavior(.possessive)
```

This option is related to, but independent from, the regex syntax option `(?U)`. See below for more about that regex-syntax-only option.

**Standard Library API:**

```swift
extension Regex {
  /// Returns a regular expression where quantifiers use the specified behavior
  /// by default.
  ///
  /// You can call this method to change the default repetition behavior for
  /// quantifier operators in regex syntax and `RegexBuilder` quantifier
  /// methods. For example, in the following example, both regexes use
  /// possessive quantification when matching a quotation surround by `"`
  /// quote marks:
  ///
  ///     let regex1 = /"[^"]*"/.defaultRepetitionBehavior(.possessive)
  ///
  ///     let quoteMark = "\""
  ///     let regex2 = Regex {
  ///         quoteMark
  ///         ZeroOrMore(.noneOf(quoteMark))
  ///         quoteMark
  ///     }.defaultRepetitionBehavior(.possessive)
  ///
  /// This setting only changes the default behavior of quantifiers, and does
  /// not affect regex syntax operators with an explicit behavior indicator,
  /// such as `*?` or `++`. Likewise, calls to quantifier methods such as
  /// `OneOrMore` always use the explicit `behavior`, when given.
  ///
  /// - Parameter behavior: The default behavior to use for quantifiers.
  public func defaultRepetitionBehavior(_ behavior: RegexRepetitionBehavior) -> Regex<RegexOutput>
}

public struct RegexRepetitionBehavior {
  /// Match as much of the input string as possible, backtracking when
  /// necessary.
  public static var eager: RegexRepetitionBehavior { get }

  /// Match as little of the input string as possible, expanding the matched
  /// region as necessary to complete a match.
  public static var reluctant: RegexRepetitionBehavior { get }

  /// Match as much of the input string as possible, performing no backtracking.
  public static var possessive: RegexRepetitionBehavior { get }
}
```

As described in the [Regex Builder proposal][regexbuilder], `RegexBuilder` quantifier APIs include a `nil`-defaulted optional `behavior` parameter. When you pass `nil`, the quantifier uses the default behavior as set by this option. If an explicit behavior is passed, that behavior is used regardless of the default.

```swift
// Example `OneOrMore` initializer
extension OneOrMore {
    public init<W, C0, Component: RegexComponent>(
    _ behavior: RegexRepetitionBehavior? = nil,
    @RegexComponentBuilder _ component: () -> Component
  ) where Output == (Substring, C0), Component.Output == (W, C0)
}
```

#### Single line mode (`.` matches newlines)

The "any" metacharacter (`.`) matches any character in a string *except* newlines by default. With the `s` option enabled, `.` matches any character including newlines.

```swift
let str = """
    <<This string
    uses double-angle-brackets
    to group text.>>
    """
    
str.firstMatch(of: /<<.+>>/)
// nil
str.firstMatch(of: /<<.+>>/.dotMatchesNewLines())
// "This string\nuses double-angle-brackets\nto group text."
```

This option applies only to `.` used in regex syntax and does _not_ affect the behavior of `CharacterClass.any`, which always matches any character or Unicode scalar. To get the default `.` behavior when using `RegexBuilder` syntax, use `CharacterClass.anyNonNewline`.

**Regex syntax:** `(?s)...` or `(?s...)`

**Standard Library API:**

```swift
extension Regex {
  /// Returns a regular expression where the start and end of input
  /// anchors (`^` and `$`) also match against the start and end of a line.
  public func dotMatchesNewlines(_ dotMatchesNewlines: Bool = true) -> Regex<RegexOutput>
}
```

#### Multiline mode

By default, the start and end anchors (`^` and `$`) match only the beginning and end of a string. With the `m` or the option, they also match the beginning and end of each line.

```swift
let str = """
    abc
    def
    ghi
    """
	
str.firstMatch(of: /^abc/)
// "abc"
str.firstMatch(of: /^abc$/)
// nil
str.firstMatch(of: /^abc$/.anchorsMatchLineEndings())
// "abc"
	
str.firstMatch(of: /^def/)
// nil
str.firstMatch(of: /^def$/.anchorsMatchLineEndings())
// "def"
```

This option applies only to anchors used in regex syntax. The anchors defined in `RegexBuilder` are specific about matching at the start/end of the input or the line, and therefore are not affected by this option.

```swift
str.firstMatch(of: Regex { Anchor.startOfInput ; "def" }) // nil
str.firstMatch(of: Regex { Anchor.startOfLine  ; "def" }) // "def"
```

**Regex syntax:** `(?m)...` or `(?m...)`

**Standard Library API:**

```swift
extension Regex {
  /// Returns a regular expression where the start and end of input
  /// anchors (`^` and `$`) also match against the start and end of a line.
  public func anchorsMatchLineEndings(_ matchLineEndings: Bool = true) -> Regex<RegexOutput>
}
```

#### Eager/reluctant toggle

Regex quantifiers (`+`, `*`, and `?`) match eagerly by default when they repeat, such that they match the longest possible substring. Appending `?` to a quantifier makes it reluctant, instead, so that it matches the shortest possible substring.

```swift
let str = "<token>A value.</token>"
	
// By default, the '+' quantifier is eager, and consumes as much as possible.
str.firstMatch(of: /<.+>/)          // "<token>A value.</token>"
	
// Adding '?' makes the '+' quantifier reluctant, so that it consumes as little as possible.
str.firstMatch(of: /<.+?>/)         // "<token>"
```

The `U` option toggles the "eagerness" of quantifiers, so that quantifiers are reluctant by default, and only become eager when `?` is added to the quantifier. This change only applies within regex syntax. See the `defaultRepetitionBehavior(_:)` method, described above, for broader control over repetition behavior, including setting the default for `RegexBuilder` syntax.

```swift
// '(?U)' toggles the eagerness of quantifiers:
str.firstMatch(of: /(?U)<.+>/)      // "<token>"
str.firstMatch(of: /(?U)<.+?>/)     // "<token>A value.</token>"
```

**Regex syntax:** `(?U)...` or `(?U...)`


---

### Character Classes

We propose the following definitions for regex character classes, along with a `CharacterClass` type as part of the `RegexBuilder` module, to encapsulate and simplify character class usage within builder-style regexes.

The two regexes defined in this example will match the same inputs, looking for one or more word characters followed by up to three digits, optionally separated by a space:

```swift
let regex1 = /\w+\s?\d{,3}/
let regex2 = Regex {
    OneOrMore(.word)
    Optionally(.whitespace)
    Repeat(.digit, ...3)
}
```

You can build custom character classes by combining regex-defined classes with individual characters or ranges, or by performing common set operations such as subtracting or negating a character class.


#### “Any”

The simplest character class, representing **any character**, is written as `.` and is sometimes referred to as the "dot" metacharacter. This class always matches a single `Character` or Unicode scalar value, depending on the matching semantic level. This class excludes newlines, unless "single line mode" is enabled (see section above).

When using the `CharacterClass` type in a `RegexBuilder`-defined regex, the `.any` and `.anyNonNewline` provide separate APIs for the two behaviors of `.`, and are therefore unaffected by the current "single line mode" setting.

```swift
"Cafe\u{301}".contains(/C.../)
// true
```

For this example, using Unicode scalar semantics, a dot matches only a single Unicode scalar value, so the combining marks don't get grouped with the commas before them:

```swift
let data = "\u{300},\u{301},\u{302},\u{303},..."
for match in data.matches(of: /(.),/.matchingSemantics(.unicodeScalar)) {
    print(match.1)
}
// Prints:
//  ̀
//  ́
//  ̂
// ...
```

#### Any grapheme cluster

`Regex` also provides a way to match a single grapheme cluster, regardless of the current semantic level. The **any grapheme cluster** character class is written as `\X` or `CharacterClass.anyGraphemeCluster`, and matches from the current location up to the next grapheme cluster boundary. This includes matching newlines, regardless of any option settings. This metacharacter is equivalent to the regex syntax `(?Xs:.)`.


#### Digits

The **decimal digit** character class is matched by `\d` or `CharacterClass.digit`. Both regexes in this example match one or more decimal digits followed by a colon:

```swift
let regex1 = /\d+:/
let regex2 = Regex {
    OneOrMore(.digit)
    ":"
}
```

_Unicode scalar semantics:_ Matches a Unicode scalar that has a `numericType` property equal to `.decimal`. This includes the digits from the ASCII range, from the _Halfwidth and Fullwidth Forms_ Unicode block, as well as digits in some scripts, like `DEVANAGARI DIGIT NINE` (U+096F). This corresponds to the general category `Decimal_Number`.

_Grapheme cluster semantics:_ Matches a character made up of a single Unicode scalar that fits the decimal digit criteria above.


To invert the decimal digit character class, use `\D` or `CharacterClass.digit.inverted`.


The **hexadecimal digit** character class is matched by  `CharacterClass.hexDigit`.

_Unicode scalar semantics:_ Matches a decimal digit, as described above, or an uppercase or small `A` through `F` from the _Halfwidth and Fullwidth Forms_ Unicode block. Note that this is a broader class than described by the `UnicodeScalar.properties.isHexDigit` property, as that property only include ASCII and fullwidth decimal digits.

_Grapheme cluster semantics:_ Matches a character made up of a single Unicode scalar that fits the hex digit criteria above.

To invert the hexadecimal digit character class, use `CharacterClass.hexDigit.inverted`.

*<details><summary>Rationale</summary>*

Unicode's recommended definition for `\d` is its [numeric type][numerictype] of "Decimal" in contrast to "Digit". It is specifically restricted to sets of ascending contiguously-encoded scalars in a decimal radix positional numeral system. Thus, it excludes "digits" such as superscript numerals from its [definition][derivednumeric] and is a proper subset of `Character.isWholeNumber`. 

We interpret Unicode's definition of the set of scalars, especially its requirement that scalars be encoded in ascending chains, to imply that this class is restricted to scalars which meaningfully encode base-10 digits. Thus, we choose to make the grapheme cluster interpretation *restrictive*.

</details>


#### "Word" characters

The **word** character class is matched by `\w` or `CharacterClass.word`. This character class and its name are essentially terms of art within regexes, and represents part of a notional "word". Note that, by default, this is distinct from the algorithm for identifying word boundaries.

_Unicode scalar semantics:_ Matches a Unicode scalar that has one of the Unicode properties `Alphabetic`, `Digit`, or `Join_Control`, or is in the general category `Mark` or `Connector_Punctuation`. 

_Grapheme cluster semantics:_ Matches a character that begins with a Unicode scalar value that fits the criteria above.

To invert the word character class, use `\W` or `CharacterClass.word.inverted`.

*<details><summary>Rationale</summary>*

Word characters include more than letters, and we went with Unicode's recommended scalar semantics. Following the Unicode recommendation that nonspacing marks remain with their base characters, we extend to grapheme clusters similarly to `Character.isLetter`. That is, combining scalars do not change the word-character-ness of the grapheme cluster.

</details>


#### Whitespace and newlines

The **whitespace** character class is matched by `\s` and `CharacterClass.whitespace`.

_Unicode scalar semantics:_ Matches a Unicode scalar that has the Unicode properties `Whitespace`, including a space, a horizontal tab (U+0009), `LINE FEED (LF)` (U+000A), `LINE TABULATION` (U+000B), `FORM FEED (FF)` (U+000C), `CARRIAGE RETURN (CR)` (U+000D), and `NEWLINE (NEL)` (U+0085). Note that under Unicode scalar semantics, `\s` only matches the first scalar in a `CR`+`LF` pair.

_Grapheme cluster semantics:_ Matches a character that begins with a `Whitespace` Unicode scalar value. This includes matching a `CR`+`LF` pair.

The **horizontal whitespace** character class is matched by `\h` and `CharacterClass.horizontalWhitespace`.

_Unicode scalar semantics:_ Matches a Unicode scalar that has the Unicode general category `Zs`/`Space_Separator` as well as a horizontal tab (U+0009).

_Grapheme cluster semantics:_ Matches a character that begins with a Unicode scalar value that fits the criteria above.

The **vertical whitespace** character class is matched by `\v` and `CharacterClass.verticalWhitespace`. Additionally, `\R` and `CharacterClass.newline` provide a way to include the `CR`+`LF` pair, even when matching with Unicode scalar semantics.

_Unicode scalar semantics:_ Matches a Unicode scalar that has the Unicode general category `Zl`/`Line_Separator` or `Zp`/`Paragraph_Separator`, as well as any of the following control characters: `LINE FEED (LF)` (U+000A), `LINE TABULATION` (U+000B), `FORM FEED (FF)` (U+000C), `CARRIAGE RETURN (CR)` (U+000D), and `NEWLINE (NEL)` (U+0085). Only when specified as `\R` or `CharacterClass.newline` does this match the whole `CR`+`LF` pair.

_Grapheme cluster semantics:_ Matches a character that begins with a Unicode scalar value that fits the criteria above.

To invert these character classes, use `\S`, `\H`, and `\V`, respectively, or the `inverted` property on a `CharacterClass` instance.

<details><summary>Rationale</summary>

Note that "whitespace" is a term-of-art and is not correlated with visibility, which is a completely separate concept.

We use Unicode's recommended scalar semantics for horizontal and vertical whitespace, extended to grapheme clusters as in the existing `Character.isWhitespace` property.

</details>


#### Unicode properties

Character classes that match **Unicode properties** are written as `\p{PROPERTY}` or `\p{PROPERTY=VALUE}`, as described in the [Run-time Regex Construction proposal][internals-properties].

While most Unicode properties are only defined at the scalar level, some are defined to match an extended grapheme cluster. For example, `\p{RGI_Emoji_Flag_Sequence}` will match any flag emoji character, which are composed of two Unicode scalar values. Such property classes will match multiple scalars, even when matching with Unicode scalar semantics.

Unicode property matching is extended to `Character`s with a goal of consistency with other regex character classes, and as dictated by prior standard library additions to the `Character` type. For example, for `\p{Decimal}` and `\p{Hex_Digit}`, only single-scalar `Character`s can match, for the reasons described in that section, above. For other Unicode property classes, like `\p{Whitespace}`, the character matches when the first scalar has that Unicode property. Open the following disclosure area to see the full list of properties, along with the rubric for extending them to grapheme clusters.

<details><summary>Unicode properties</summary>

We can choose to extend a Unicode property to a grapheme cluster in one of several ways:

- *single-scalar*: Only a character that comprises a single Unicode scalar value can match
- *first-scalar*: If the first Unicode scalar in a character matches, then the character matches
- *any-scalar*: If any Unicode scalar in a character matches, then the character matches
- *all-scalars*: A character matches if and only if all its Unicode scalar members match

With a few guidelines, we can make headway on classifying Unicode properties:

- Numeric-related properties, like `Numeric_Value`, should only apply to single-scalar characters, for the reasons described in the "Digit" character class section, above.
- Any other properties that directly or approximately correspond to regex or POSIX character classes should use the first-scalar rule. This corresponds with the way `Character.isWhitespace` is implemented and generally matches the perceived categorization of characters.
- Properties that resolve to a unique Unicode scalar, such as `Name`, should only apply to single-scalar characters.
- Properties that govern the way Unicode scalars combine into characters, such as `Canonical_Combining_Class`, or are otherwise only relevant when examining specific Unicode data, such as `Age`, should only apply to single-scalar characters.
- Properties that can naturally apply to a sequence of Unicode scalars, such as `Lowercase_Mapping`, should use an all-scalars approach. This corresponds with the way `Character.isLowercased` and other casing properties are implemented.

In many cases, properties with a *single-scalar* treatment won't match any characters at all, and will only be useful when matching with Unicode scalar semantics. For example, `/\p{Emoji_Modifier}/` matches the five Fitzpatrick skin tone modifier Unicode scalar values that affect the appearance of emoji within a grapheme cluster. When matching with grapheme cluster semantics, no match for the pattern will be found. Using Unicode scalar semantics, however, you can search for all characters that include such a modifier:

```
let regex = /(?u)\y.+?\p{Emoji_Modifier}.+?\y/
for ch in "👩🏾‍🚀🚀 👨🏻‍🎤🎸 🧑🏻‍💻📲".matches(of: regex) {
  print(ch)
}
// Prints:
// 👩🏾‍🚀
// 👨🏻‍🎤
// 🧑🏻‍💻
```

The table below shows our best effort at choosing the right manner of extending.

| Property                            | Extension                     | Notes                             |
|-------------------------------------|-------------------------------|-----------------------------------|
| **General**                         |                               |                                   |
| `Name`                              | single-scalar                 |                                   |
| `Name_Alias`                        | single-scalar                 |                                   |
| `Age`                               | single-scalar                 |                                   |
| `General_Category`                  | first-scalar                  | Numeric categories: single-scalar |
| `Script`                            | first-scalar                  |                                   |
| `White_Space`                       | first-scalar                  | Existing `Character` API          |
| `Alphabetic`                        | first-scalar                  |                                   |
| `Noncharacter_Code_Point`           | single-scalar                 |                                   |
| `Default_Ignorable_Code_Point`      | single-scalar                 |                                   |
| `Deprecated`                        | single-scalar                 |                                   |
| `Logical_Order_Exception`           | single-scalar                 |                                   |
| `Variation_Selector`                | single-scalar                 |                                   |
| <br> **Numeric**                    |                               |                                   |
| `Numeric_Value`                     | single-scalar                 |                                   |
| `Numeric_Type`                      | single-scalar                 |                                   |
| `Hex_Digit`                         | single-scalar                 | Existing `Character` API          |
| `ASCII_Hex_Digit`                   | single-scalar                 |                                   |
| <br> **Identifiers**                |                               |                                   |
| `ID_Start`                          | single-scalar                 |                                   |
| `ID_Continue`                       | single-scalar                 |                                   |
| `XID_Start`                         | single-scalar                 |                                   |
| `XID_Continue`                      | single-scalar                 |                                   |
| `Pattern_Syntax`                    | single-scalar                 |                                   |
| `Pattern_White_Space`               | single-scalar                 |                                   |
| <br> **CJK**                        |                               |                                   |
| `Ideographic`                       | first-scalar                  |                                   |
| `Unified_Ideograph`                 | first-scalar                  |                                   |
| `Radical`                           | first-scalar                  |                                   |
| `IDS_Binary_Operator`               | single-scalar                 |                                   |
| `IDS_Trinary_Operator`              | single-scalar                 |                                   |
| <br> **Case**                       |                               |                                   |
| `Lowercase`                         | first-scalar                  |                                   |
| `Uppercase`                         | first-scalar                  |                                   |
| `Lowercase_Mapping`                 | all-scalars                   |                                   |
| `Titlecase_Mapping`                 | all-scalars                   |                                   |
| `Uppercase_Mapping`                 | all-scalars                   |                                   |
| `Soft_Dotted`                       | first-scalar                  |                                   |
| `Cased`                             | any-scalar                    |                                   |
| `Case_Ignorable`                    | all-scalars                   |                                   |
| `Changes_When_Lowercased`           | all-scalars                   |                                   |
| `Changes_When_Uppercased`           | all-scalars                   |                                   |
| `Changes_When_Titlecased`           | all-scalars                   |                                   |
| `Changes_When_Casefolded`           | all-scalars                   |                                   |
| `Changes_When_Casemapped`           | all-scalars                   |                                   |
| <br> **Normalization**              |                               |                                   |
| `Canonical_Combining_Class`         | single-scalar                 |                                   |
| `Full_Composition_Exclusion`        | single-scalar                 |                                   |
| `Changes_When_NFKC_Casefolded`      | all-scalars                   |                                   |
| <br> **Emoji**                      |                               |                                   |
| `Emoji`                             | first-scalar                  |                                   |
| `Emoji_Presentation`                | any-scalar                    |                                   |
| `Emoji_Modifier`                    | single-scalar                 |                                   |
| `Emoji_Modifier_Base`               | single-scalar                 |                                   |
| <br> **Shaping and Rendering**      |                               |                                   |
| `Join_Control`                      | single-scalar                 |                                   |
| <br> **Bidirectional**              |                               |                                   |
| `Bidi_Control`                      | single-scalar                 |                                   |
| `Bidi_Mirrored`                     | first-scalar                  |                                   |
| <br> **Miscellaneous**              |                               |                                   |
| `Math`                              | first-scalar                  |                                   |
| `Quotation_Mark`                    | first-scalar                  |                                   |
| `Dash`                              | first-scalar                  |                                   |
| `Sentence_Terminal`                 | first-scalar                  |                                   |
| `Terminal_Punctuation`              | first-scalar                  |                                   |
| `Diacritic`                         | single-scalar                 |                                   |
| `Extender`                          | single-scalar                 |                                   |
| `Grapheme_Base`                     | single-scalar                 |                                   |
| `Grapheme_Extend`                   | single-scalar                 |                                   |

</details>

To invert a Unicode property character class, use `\P{...}`.

When using `RegexBuilder` syntax, Unicode property classes are available through the following methods on  `CharacterClass`:

- `static func generalCategory(_: Unicode.GeneralCategory) -> CharacterClass`
- `static func binaryProperty(_: KeyPath<UnicodeScalar.Properties, Bool>, value: Bool = true) -> CharacterClass`
- `static func named(_: String) -> CharacterClass`
- `static func age(_: Unicode.Version) -> CharacterClass`
- `static func numericType(_: Unicode.NumericType) -> CharacterClass`
- `static func numericValue(_: Double) -> CharacterClass`
- `static func lowercaseMapping(_: String) -> CharacterClass`
- `static func uppercaseMapping(_: String) -> CharacterClass`
- `static func titlecaseMapping(_: String) -> CharacterClass`
- `static func canonicalCombiningClass(_: Unicode.CanonicalCombiningClass) -> CharacterClass`

You can see the full `CharacterClass` API with documentation comments in the **Custom Classes** section, below.

#### POSIX character classes: `[:NAME:]` or `\p{NAME}`

**POSIX character classes** represent concepts that we'd like to define at all semantic levels. We propose the following definitions, some of which have been described above. When matching with grapheme cluster semantics, Unicode properties are extended to `Character`s as described in the rationale above, and as shown in the table below. That is, for POSIX class `[:word:]`, any `Character` that starts with a matching scalar is a match, while for `[:digit:]`, a matching `Character` must only comprise a single Unicode scalar value.

| POSIX class  | Unicode property class            | Character behavior   | ASCII mode value              |
|--------------|-----------------------------------|----------------------|-------------------------------|
| `[:lower:]`  | `\p{Lowercase}`                   | starts-with          | `[a-z]`                       |
| `[:upper:]`  | `\p{Uppercase}`                   | starts-with          | `[A-Z]`                       |
| `[:alpha:]`  | `\p{Alphabetic}`                  | starts-with          | `[A-Za-z]`                    |
| `[:alnum:]`  | `[\p{Alphabetic}\p{Decimal}]`     | starts-with          | `[A-Za-z0-9]`                 |
| `[:word:]`   | See \* below                      | starts-with          | `[[:alnum:]_]`                |
| `[:digit:]`  | `\p{DecimalNumber}`               | single-scalar        | `[0-9]`                       |
| `[:xdigit:]` | `\p{Hex_Digit}`                   | single-scalar        | `[0-9A-Fa-f]`                 |
| `[:punct:]`  | `\p{Punctuation}`                 | starts-with          | `[-!"#%&'()*,./:;?@[\\\]{}]`  |
| `[:blank:]`  | `[\p{Space_Separator}\u{09}]`     | starts-with          | `[ \t]`                       |
| `[:space:]`  | `\p{Whitespace}`                  | starts-with          | `[ \t\n\r\f\v]`               |
| `[:cntrl:]`  | `\p{Control}`                     | starts-with          | `[\x00-\x1f\x7f]`             |
| `[:graph:]`  | See \*\* below                    | starts-with          | `[^ [:cntrl:]]`               |
| `[:print:]`  | `[[:graph:][:blank:]--[:cntrl:]]` | starts-with          | `[[:graph:] ]`                |

\* The Unicode scalar property definition for `[:word:]` is `[\p{Alphanumeric}\p{Mark}\p{Join_Control}\p{Connector_Punctuation}]`.  
\*\* The Unicode scalar property definition for `[:cntrl:]` is `[^\p{Space}\p{Control}\p{Surrogate}\p{Unassigned}]`.

#### Custom classes

Custom classes function as the set union of their individual components, whether those parts are individual characters, individual Unicode scalar values, ranges, Unicode property classes or POSIX classes, or other custom classes.

- Individual characters and scalars will be tested using the same behavior as if they were listed in an alternation. That is, a custom character class like `[abc]` is equivalent to `(a|b|c)` under the same options and modes.
- Metacharacters that represent built-in character classes keep their same function inside custom character classes. For example, in `[abc\d]+`, the `\d` matches any digit, so the regex matches the entirety of the string `"0a1b2c3"`, and `[\t\R]` matches a tab or any newline character or newline sequence.
- Metacharacters that represent zero-width assertions have their literal meaning in custom character classes, if one exists. For example, `[\b^]` matches either the BEL control character or a literal carat (`^`), while `\B` is an invalid member of a custom character class.

Ranges in a custom character class require special consideration to avoid unexpected or dangerous results. Using simple lexicographical ordering for comparison is unintuitive when working with multi-scalar characters. For example,
the custom character class `[0-9]` is intended to match only the ten ASCII digits, but because of lexicographical ordering, complex characters like `"3̠̄"` and `"5️⃣"` would fall into that range. Ranges in custom character classes therefore having the following requirements:

- Range endpoints must be single Unicode scalar values. When parsing a regex, endpoints will be converted to their canonical composed form, so that characters that have a multi-Unicode scalar form in source but a single-scalar canonical representation will still be permitted.
- When matching with grapheme cluster semantics, only single-scalar characters will match a range. The same conversion to canonical composed form will be used to support the expectation of matching with canonical equivalence.

```swift
let allDigits = /^[0-9]+$/
"1230".contains(allDigits)              // true
"123̠̄0".contains(allDigits)              // false
"5️⃣".contains(allDigits)                 // false

let cafeExtended = /Caf[à-ÿ]/
"Café".contains(cafeExtended)           // true
"Cafe\u{301}".contains(cafeExtended)    // true
```

Inside regexes, custom classes are enclosed in square brackets `[...]`, and can be nested or combined using set operators like `&&`. For more detail, see the [Run-time Regex Construction proposal][internals-charclass].

With `RegexBuilder`'s `CharacterClass` type, you can use built-in character classes with ranges and groups of characters. For example, to parse a valid octodecimal number, you could define a custom character class that combines `.digit` with a range of characters.

```swift
let octoDecimalRegex: Regex<(Substring, Int?)> = Regex {
    let charClass = CharacterClass(.digit, "a"..."h").ignoresCase()
    Capture {
      OneOrMore(charClass)
    } transform: { Int($0, radix: 18) }
}
```

The full `CharacterClass` API is as follows:

```swift
/// A class of characters that match in a regex.
///
/// A character class can represent individual characters, a group of
/// characters, the set of character that match some set of criteria, or
/// a set algebraic combination of all of the above.
public struct CharacterClass: RegexComponent {
  public var regex: Regex<Substring> { get }

  /// A character class that matches any character that does not match this
  /// character class.
  public var inverted: CharacterClass { get }
}

// MARK: Built-in character classes

extension RegexComponent where Self == CharacterClass {
  /// A character class that matches any element.
  ///
  /// This character class is unaffected by the `dotMatchesNewlines()` method.
  /// To match any character that isn't a newline, see
  /// ``CharacterClass.anyNonNewline``.
  ///
  /// This character class is equivalent to the regex syntax "dot"
  /// metacharacter in single-line mode: `(?s:.)`.
  public static var any: CharacterClass { get }

  /// A character class that matches any element that isn't a newline.
  ///
  /// This character class is unaffected by the `dotMatchesNewlines()` method.
  /// To match any character, including newlines, see ``CharacterClass.any``.
  ///
  /// This character class is equivalent to the regex syntax "dot"
  /// metacharacter with single-line mode disabled: `(?-s:.)`.
  public static var anyNonNewline: CharacterClass { get }

  /// A character class that matches any single `Character`, or extended
  /// grapheme cluster, regardless of the current semantic level.
  ///
  /// This character class is equivalent to `\X` in regex syntax.
  public static var anyGraphemeCluster: CharacterClass { get }

  /// A character class that matches any digit.
  ///
  /// This character class is equivalent to `\d` in regex syntax.
  public static var digit: CharacterClass { get }
  
  /// A character class that matches any hexadecimal digit.
  public static var hexDigit: CharacterClass { get }

  /// A character class that matches any element that is a "word character".
  ///
  /// This character class is equivalent to `\w` in regex syntax.
  public static var word: CharacterClass { get }

  /// A character class that matches any element that is classified as
  /// whitespace.
  ///
  /// This character class is equivalent to `\s` in regex syntax.
  public static var whitespace: CharacterClass { get }
  
  /// A character class that matches any element that is classified as
  /// horizontal whitespace.
  ///
  /// This character class is equivalent to `\h` in regex syntax.
  public static var horizontalWhitespace: CharacterClass { get }

  /// A character class that matches any element that is classified as
  /// vertical whitespace.
  ///
  /// This character class is equivalent to `\v` in regex syntax.
  public static var verticalWhitespace: CharacterClass { get }

  /// A character class that matches any newline sequence.
  ///
  /// This character class is equivalent to `\R` or `\n` in regex syntax.
  public static var newlineSequence: CharacterClass { get }
}

// MARK: anyOf(_:) / noneOf(_:)

extension RegexComponent where Self == CharacterClass {
  /// Returns a character class that matches any character in the given string
  /// or sequence.
  ///
  /// Calling this method with a group of characters is equivalent to listing
  /// those characters in a custom character class in regex syntax. For example,
  /// the two regexes in this example are equivalent:
  ///
  ///     let regex1 = /[abcd]+/
  ///     let regex2 = OneOrMore(.anyOf("abcd"))
  public static func anyOf<S: Sequence>(_ s: S) -> CharacterClass
    where S.Element == Character
    
  /// Returns a character class that matches any Unicode scalar in the given
  /// sequence.
  ///
  /// Calling this method with a group of Unicode scalars is equivalent to
  /// listing them in a custom character class in regex syntax.
  public static func anyOf<S: Sequence>(_ s: S) -> CharacterClass
    where S.Element == UnicodeScalar

  /// Returns a character class that matches none of the characters in the given
  /// string or sequence.
  ///
  /// Calling this method with a group of characters is equivalent to listing
  /// those characters in a negated custom character class in regex syntax. For
  /// example, the two regexes in this example are equivalent:
  ///
  ///     let regex1 = /[^abcd]+/
  ///     let regex2 = OneOrMore(.noneOf("abcd"))
  public static func noneOf<S: Sequence>(_ s: S) -> CharacterClass
    where S.Element == Character
  
  /// Returns a character class that matches none of the Unicode scalars in the
  /// given sequence.
  ///
  /// Calling this method with a group of Unicode scalars is equivalent to
  /// listing them in a negated custom character class in regex syntax.
  public static func noneOf<S: Sequence>(_ s: S) -> CharacterClass
    where S.Element == UnicodeScalar
}

// MARK: Unicode properties

extension CharacterClass {
  /// Returns a character class that matches any element with the given Unicode
  /// general category.
  ///
  /// For example, when passed `.uppercaseLetter`, this method is equivalent to
  /// `/\p{Uppercase_Letter}/` or `/\p{Lu}/`.
  public static func generalCategory(_ category: Unicode.GeneralCategory) -> CharacterClass

    /// Returns a character class that matches any element with the given Unicode
  /// binary property.
  ///
  /// For example, when passed `\.isAlphabetic`, this method is equivalent to
  /// `/\p{Alphabetic}/` or `/\p{Is_Alphabetic=true}/`.
  public static func binaryProperty(
      _ property: KeyPath<UnicodeScalar.Properties, Bool>,
      value: Bool = true
  ) -> CharacterClass
  
  /// Returns a character class that matches any element with the given Unicode
  /// name.
  ///
  /// This method is equivalent to `/\p{Name=name}/`.
  public static func name(_ name: String) -> CharacterClass
  
  /// Returns a character class that matches any element that was included in
  /// the specified Unicode version.
  ///
  /// This method is equivalent to `/\p{Age=version}/`.
  public static func age(_ version: Unicode.Version) -> CharacterClass
  
  /// Returns a character class that matches any element with the given Unicode
  /// numeric type.
  ///
  /// This method is equivalent to `/\p{Numeric_Type=type}/`.
  public static func numericType(_ type: Unicode.NumericType) -> CharacterClass
  
  /// Returns a character class that matches any element with the given numeric
  /// value.
  ///
  /// This method is equivalent to `/\p{Numeric_Value=value}/`.
  public static func numericValue(_ value: Double) -> CharacterClass
  
  /// Returns a character class that matches any element with the given Unicode
  /// canonical combining class.
  ///
  /// This method is equivalent to
  /// `/\p{Canonical_Combining_Class=combiningClass}/`.
  public static func canonicalCombiningClass(
      _ combiningClass: Unicode.CanonicalCombiningClass
  ) -> CharacterClass
  
  /// Returns a character class that matches any element with the given
  /// lowercase mapping.
  ///
  /// This method is equivalent to `/\p{Lowercase_Mapping=value}/`.
  public static func lowercaseMapping(_ value: String) -> CharacterClass
  
  /// Returns a character class that matches any element with the given
  /// uppercase mapping.
  ///
  /// This method is equivalent to `/\p{Uppercase_Mapping=value}/`.
  public static func uppercaseMapping(_ value: String) -> CharacterClass

  /// Returns a character class that matches any element with the given
  /// titlecase mapping.
  ///
  /// This method is equivalent to `/\p{Titlecase_Mapping=value}/`.
  public static func titlecaseMapping(_ value: String) -> CharacterClass
}

// MARK: Set algebra methods

extension CharacterClass {
  /// Returns a character class that combines the characters classes in the
  /// given sequence or collection via union.
  public init(_ characterClasses: some Sequence<CharacterClass>)

  /// Creates a character class that combines the given classes in a union.
  public init(_ first: CharacterClass, _ rest: CharacterClass...)
  
  /// Returns a character class from the union of this class and the given class.
  public func union(_ other: CharacterClass) -> CharacterClass
  
  /// Returns a character class from the intersection of this class and the given class.
  public func intersection(_ other: CharacterClass) -> CharacterClass
  
  /// Returns a character class by subtracting the given class from this class.
  public func subtracting(_ other: CharacterClass) -> CharacterClass
  
  /// Returns a character class matching elements in one or the other, but not both,
  /// of this class and the given class.
  public func symmetricDifference(_ other: CharacterClass) -> CharacterClass
}

// MARK: Range syntax

public func ...(lhs: Character, rhs: Character) -> CharacterClass

@_disfavoredOverload
public func ...(lhs: UnicodeScalar, rhs: UnicodeScalar) -> CharacterClass
```

## Source compatibility

Everything in this proposal is additive, and has no compatibility effect on existing source code.

## Effect on ABI stability

Everything in this proposal is additive, and has no effect on existing stable ABI.

## Effect on API resilience

N/A

## Future directions

### Expanded options and modifiers

The initial version of `Regex` includes only the options described above. Filling out the remainder of options described in the [Run-time Regex Construction proposal][internals] could be completed as future work, as well as additional improvements, such as adding an option that makes a regex match only at the start of a string.

### Extensions to Character and Unicode Scalar APIs

An earlier version of this pitch described adding standard library APIs to `Character` and `UnicodeScalar` for each of the supported character classes, as well as convenient static members for control characters. In addition, regex literals support Unicode property features that don’t currently exist in the standard library, such as a scalar’s script or extended category, or creating a scalar by its Unicode name instead of its scalar value. These kinds of additions have value outside of just their relationship to the `Regex` additions, so they can be pitched and considered in a future proposal.

### Byte semantic mode

A future `Regex` version could support a byte-level semantic mode in addition to grapheme cluster and Unicode scalar semantics. Byte-level semantics would allow matching individual bytes, potentially providing the capability of parsing string and non-string data together.

### More general `CharacterSet` replacement

Foundation's `CharacterSet` type is in some ways similar to the `CharacterClass` type defined in this proposal. `CharacterSet` is primarily a set type that is defined over Unicode scalars, and can therefore sometimes be awkward to use in conjunction with Swift `String`s. The proposed `CharacterClass` type is a `RegexBuilder`-specific type, and as such isn't intended to be a full general purpose replacement. Future work could involve expanding upon the `CharacterClass` API or introducing a different type to fill that role.

## Alternatives considered

### Operate on String.UnicodeScalarView instead of using semantic modes

Instead of providing APIs to select whether `Regex` matching is `Character`-based vs. `UnicodeScalar`-based, we could instead provide methods to match against the different views of a string. This different approach has multiple drawbacks:

* As the scalar level used when matching changes the behavior of individual components of a `Regex`, it’s more appropriate to specify the semantic level at the declaration site than the call site.
* With the proposed options model, you can define a Regex that includes different semantic levels for different portions of the match, which would be impossible with a call site-based approach.

### Binary word boundary option method

A prior version of this proposal used a binary method for setting the word boundary algorithm, called `usingSimpleWordBoundaries()`. A method taking a `RegexWordBoundaryKind` instance is included in the proposal instead, to leave room for implementing other word boundary algorithms in the future.

### More "Swifty" default option settings

Swift's `Regex` includes some default behaviors that don't match other regex engines — in particular, matching characters with `.` and using Unicode's default word boundary algorithm. For other option-based behaviors, `Regex` adheres to the general standard set by other regular expression engines, like having `.` not match newlines and `^` and `$` only match the start and end of the input instead of the beginning and end of each line. This is to ease the process of bringing existing regular expressions and existing knowledge into Swift.

Instead, we could use this opportunity to choose default options that are more ergonomic or intuitive, and provide a `compatibilityOptions()` API that reverts back to the typical settings, including matching based on Unicode scalars instead of characters. This method could additionally be a point of documentation for Swift's choices of default behaviors.

### Include `\O` and `CharacterClass.anyUnicodeScalar`

An earlier draft of this proposal included a metacharacter and `CharacterClass` API for matching an individual Unicode scalar value, regardless of the current matching level, as a counterpart to `\X`/`.anyGraphemeCluster`. The behavior of this character class, particularly when matching with grapheme cluster semantics, is still unclear at this time, however. For example, when matching the expression `\O*`, does the implict grapheme boundary assertion apply between the `\O` and the quantification operator, or should we treat the two as a single unit and apply the assertion after the `*`?

At the present time, we prefer to allow authors to write regexes that explicitly shift into and out of Unicode scalar mode, where those kinds of decisions are handled by the explicit scope of the setting. If common patterns emerge that indicate some version of `\O` would be useful, we can add it in the future.

## Future Work

### Additional protocol to limit option methods

The option-setting methods, like `ignoresCase()`, are implemented as extensions of the `Regex` type instead of on the `RegexComponent` protocol. This makes sure that nonsensical formulations like `"abc".defaultRepetitionBehavior(.possessive)"` are impossible to write, but is somewhat inconvenient when working with `RegexBuilder` syntax, as you need to add an additional `Regex { ... }` block around a quantifier or other grouping scope that you want to have a particular behavior.

One possible future addition would be to add another protocol that refines `RegexComponent`, with a name like `RegexCompoundComponent`, representing types that can hold or more other regex components. Types like `OneOrMore`, `CharacterClass`, and `Regex` itself would all conform, and the option-setting methods would move to an extension on that new protocol, permitting more convenient usage where appropriate.

### API for current options

As we gather information about how regexes are used and extended, we may find it useful to query an existing regex instance for the set of options that are present globally, or at the start of the regex. Likewise, if `RegexBuilder` gains the ability to use a predicate or other call out to other code, that may require providing the current set of options at the time of execution.

Such an options type could have a simple read-write, property accessor interface:

```swift
/// A set of options that affect matching behavior and semantics.
struct RegexOptions {
    /// A Boolean value indicating whether casing is ignored while matching.
    var ignoresCase: Bool
    /// An option set representing any character classes that are matched as ASCII-only.
    var asciiOnlyClasses: RegexCharacterClassKind
    /// The current matching semantics.
    var matchingSemantics: RegexMatchingSemantics
    // etc...
}
```

### Regex syntax for matching level

An earlier draft of this proposal included options within the regex syntax that are equivalent to calling the `matchingSemantics(_:)` method: `(?X)` for switching to grapheme cluster more and `(?u)` for switching to Unicode scalar mode. As these are new additions to regex syntax, and their exclusive behavior has yet to be determined, they are not included in the proposed functionality at this time.

### API for overriding Unicode property mapping

We could add API in the future to change how individual Unicode scalar properties are extended to characters. One such approach could be to provide a modifier method that takes a key path and a strategy:

```swift
// Matches only the character "a"
let regex1 = /\p{name=latin lowercase a}/

// Matches any character with "a" as its first scalar
let regex1 = /\p{name=latin lowercase a}/.extendUnicodeProperty(\.name, by: .firstScalar)`.
```

[repo]: https://github.com/apple/swift-experimental-string-processing/
[option-scoping]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0355-regex-syntax-run-time-construction.md#matching-options
[internals]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0355-regex-syntax-run-time-construction.md
[internals-properties]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0355-regex-syntax-run-time-construction.md#character-properties
[internals-charclass]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0355-regex-syntax-run-time-construction.md#custom-character-classes
[level1-word-boundaries]:https://unicode.org/reports/tr18/#Simple_Word_Boundaries
[level2-word-boundaries]:https://unicode.org/reports/tr18/#RL2.3

[overview]: https://forums.swift.org/t/declarative-string-processing-overview/52459
[charprops]: https://github.com/swiftlang/swift-evolution/blob/master/proposals/0221-character-properties.md
[regexbuilder]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0351-regex-builder.md
[charpropsrationale]: https://github.com/swiftlang/swift-evolution/blob/master/proposals/0221-character-properties.md#detailed-semantics-and-rationale
[canoneq]: https://www.unicode.org/reports/tr15/#Canon_Compat_Equivalence
[graphemes]: https://www.unicode.org/reports/tr29/#Grapheme_Cluster_Boundaries
[meaningless]: https://forums.swift.org/t/declarative-string-processing-overview/52459/121
[scalarprops]: https://github.com/swiftlang/swift-evolution/blob/master/proposals/0211-unicode-scalar-properties.md
[ucd]: https://www.unicode.org/reports/tr44/tr44-28.html
[numerictype]: https://www.unicode.org/reports/tr44/#Numeric_Type
[derivednumeric]: https://www.unicode.org/Public/UCD/latest/ucd/extracted/DerivedNumericType.txt


[uts18]: https://unicode.org/reports/tr18/
[proplist]: https://www.unicode.org/Public/UCD/latest/ucd/PropList.txt
[pcre]: https://www.pcre.org/current/doc/html/pcre2pattern.html
[perl]: https://perldoc.perl.org/perlre
[raku]: https://docs.raku.org/language/regexes
[rust]: https://docs.rs/regex/1.5.4/regex/
[regexbytes]: https://docs.rs/regex/1.5.4/regex/bytes/
[python]: https://docs.python.org/3/library/re.html
[ruby]: https://ruby-doc.org/core-2.4.0/Regexp.html
[csharp]: https://docs.microsoft.com/en-us/dotnet/standard/base-types/regular-expression-language-quick-reference
[icu]: https://unicode-org.github.io/icu/userguide/strings/regexp.html
[posix]: https://pubs.opengroup.org/onlinepubs/9699919799/basedefs/V1_chap09.html
[oniguruma]: https://www.cuminas.jp/sdk/regularExpression.html
[go]: https://pkg.go.dev/regexp/syntax@go1.17.2
[cplusplus]: https://www.cplusplus.com/reference/regex/ECMAScript/
[ecmascript]: https://262.ecma-international.org/12.0/#sec-pattern-semantics
[re2]: https://github.com/google/re2/wiki/Syntax
[java]: https://docs.oracle.com/javase/7/docs/api/java/util/regex/Pattern.html
