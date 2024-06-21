# Regex builder DSL

* Proposal: [SE-0351](0351-regex-builder.md)
* Authors: [Richard Wei](https://github.com/rxwei), [Michael Ilseman](https://github.com/milseman), [Nate Cook](https://github.com/natecook1000), [Alejandro Alonso](https://github.com/azoy)
* Review Manager: [Ben Cohen](https://github.com/airspeedswift)
* Implementation: [apple/swift-experimental-string-processing](https://github.com/apple/swift-experimental-string-processing/tree/main/Sources/RegexBuilder)
  * Available in nightly toolchain snapshots with `import _StringProcessing`
* Status: **Implemented (Swift 5.7)**
* Review: ([pitch](https://forums.swift.org/t/pitch-regex-builder-dsl/56007))
   ([first review](https://forums.swift.org/t/se-0351-regex-builder-dsl/56531))
       ([revision](https://forums.swift.org/t/returned-for-revision-se-0351-regex-builder-dsl/57224))
  ([second review](https://forums.swift.org/t/se-0351-second-review-regex-builder-dsl/58721))
     ([acceptance](https://forums.swift.org/t/accepted-se-0351-regex-builder-dsl/58972))

**Table of Contents**
- [Regex builder DSL](#regex-builder-dsl)
  - [Introduction](#introduction)
  - [Motivation](#motivation)
  - [Proposed solution](#proposed-solution)
  - [Detailed design](#detailed-design)
    - [`RegexComponent` protocol](#regexcomponent-protocol)
    - [Concatenation](#concatenation)
    - [Capture](#capture)
    - [Mapping Output](#mapping-output)
    - [Reference](#reference)
    - [Alternation](#alternation)
    - [Repetition](#repetition)
      - [Repetition behavior](#repetition-behavior)
    - [Anchors and Lookaheads](#anchors-and-lookaheads)
    - [Subpattern](#subpattern)
    - [Scoping](#scoping)
    - [Composability](#composability)
  - [Source compatibility](#source-compatibility)
  - [Effect on ABI stability](#effect-on-abi-stability)
  - [Effect on API resilience](#effect-on-api-resilience)
  - [Future directions](#future-directions)
    - [Conversion to textual regex](#conversion-to-textual-regex)
    - [Recursive subpatterns](#recursive-subpatterns)
  - [Alternatives considered](#alternatives-considered)
    - [Operators for quantification and alternation](#operators-for-quantification-and-alternation)
    - [Postfix `capture` and `tryCapture` methods](#postfix-capture-and-trycapture-methods)
    - [Unify quantifiers under `Repeat`](#unify-quantifiers-under-repeat)
    - [Free functions instead of types](#free-functions-instead-of-types)
    - [Support `buildOptional` and `buildEither`](#support-buildoptional-and-buildeither)
    - [Flatten optionals](#flatten-optionals)
    - [Structured rather than flat captures](#structured-rather-than-flat-captures)
    - [Unify `Capture` with `TryCapture`](#unify-capture-with-trycapture)

## Introduction

[Declarative string processing] aims to offer powerful pattern matching capabilities with expressivity, clarity, type safety, and ease of use. To achieve this, we propose to introduce a result-builder-based DSL, **regex builder**, for creating and composing regular expressions (**regex**es).

Regex builder is part of the Swift Standard Library but resides in a standalone module named `RegexBuilder`. By importing `RegexBuilder`, you get all necessary API for building a regex.

```swift
import RegexBuilder

let emailPattern = Regex {
  let word = OneOrMore(.word)
  Capture {
    ZeroOrMore {
      word
      "."
    }
    word
  }
  "@"
  Capture {
    word
    OneOrMore {
      "."
      word
    }
  }
} // => Regex<(Substring, Substring, Substring)>

let email = "My email is my.name@mail.swift.org."
if let match = try emailPattern.firstMatch(in: email) {
  let (wholeMatch, name, domain) = match.output
  // wholeMatch: "my.name@mail.swift.org"
  //       name: "my.name"
  //     domain: "mail.swift.org"
}
```

This proposal introduces all core API for creating and composing regexes that echos the textual [regex syntax] and [strongly typed regex captures], but does not formally specify the matching semantics or define character classes.

## Motivation

Regex is a fundemental and powerful tool for textual pattern matching. It is a domain-specific language often expressed as text. For example, given the following bank statement:

```
CREDIT    04062020    PayPal transfer    $4.99
CREDIT    04032020    Payroll            $69.73
DEBIT     04022020    ACH transfer       $38.25
DEBIT     03242020    IRS tax payment    $52249.98
```

One can write the follow textual regex to match each line:

```
(CREDIT|DEBIT)\s+(\d{2}\d{2}\d{4})\s+([\w\s]+\w)\s+(\$\d+\.\d{2})
```

While a regex like this is very compact and expressive, it is very difficult read, write and use:

1. Syntactic special characters, e.g. `\`, `(`, `[`, `{`, are too dense to be readable.
2. It contains a hierarchy of subpatterns fit into a single line of text.
3. No code completion when typing syntactic components.
4. Capturing groups produce raw data (i.e. a range or a substring) and can only be converted to other data structures after matching.
5. While comments `(?#...)` can be added inline, it only complicates readability.

## Proposed solution

We introduce regex builder, a result-builder-based API for creating and composing regexes. This API resides in a new module named `RegexBuilder` that is to be shipped as part of the Swift toolchain.

With regex builder, the regex for matching a bank statement can be written as the following:

```swift
import RegexBuilder

enum TransactionKind: String {
   case credit = "CREDIT"
   case debit = "DEBIT"
}

struct Date {
  var month, day, year: Int
  init?(mmddyyyy: String) { ... }
}

struct Amount {
  var valueTimes100: Int
  init?(twoDecimalPlaces text: Substring) { ... }
}

let statementPattern = Regex {
  // Parse the transaction kind.
  TryCapture {
    ChoiceOf {
      "CREDIT"
      "DEBIT"
    }
  } transform: {
    TransactionKind(rawValue: String($0))
  }
  OneOrMore(.whitespace)
  // Parse the date, e.g. "01012021".
  TryCapture {
    Repeat(.digit, count: 2)
    Repeat(.digit, count: 2)
    Repeat(.digit, count: 4)
  } transform: { Date(mmddyyyy: $0) }
  OneOrMore(.whitespace)
  // Parse the transaction description, e.g. "ACH transfer".
  Capture {
    OneOrMore(CharacterClass(.word, .whitespace))
    CharacterClass.word
  } transform: { String($0) }
  OneOrMore(.whitespace)
  "$"
  // Parse the amount, e.g. `$100.00`.
  TryCapture {
    OneOrMore(.digit)
    "."
    Repeat(.digit, count: 2)
  } transform: { Amount(twoDecimalPlaces: $0) }
} // => Regex<(Substring, TransactionKind, Date, String, Amount)>


let statement = """
  CREDIT    04062020    PayPal transfer    $4.99
  CREDIT    04032020    Payroll            $69.73
  DEBIT     04022020    ACH transfer       $38.25
  DEBIT     03242020    IRS tax payment    $52249.98
  """
for match in statement.matches(of: statementPattern) {
  let (line, kind, date, description, amount) = match.output
  ...
}
```

Regex builder addresses all of textual regexes' shortcomings presented in the [Motivation](#motivation) section:
1. Capture groups and quantifiers are expressed as API calls that are easy to read.
2. Scoping and indentations clearly distinguish subpatterns in the hierarchy.
3. Code completion is available when the developer types an API call.
4. Capturing groups can be transformed into structured data at the regex declaration site.
5. Normal code comments can be written within a regex declaration to further improve readability.

## Detailed design

### `RegexComponent` protocol

One of the goals of the regex builder DSL is allowing the developers to easily compose regexes from common currency types and literals, or even define custom patterns to use for matching. We introduce `RegexComponent` in the implicitly-imported `Swift` module, a protocol that unifies all types that can represent a component of a regex. Since regexes are composable, the `Regex` type itself conforms to `RegexComponent`.

```swift
public protocol RegexComponent<RegexOutput> {
  associatedtype RegexOutput
  var regex: Regex<RegexOutput> { get }
}

extension Regex: RegexComponent {
  public typealias RegexOutput = Output
  public var regex: Regex<Output> { self }
}
```

Note:
- `RegexComponent` and `Regex`'s conformance to `RegexComponent` are available without importing `RegexBuilder`. All other types and conformances introduced in this proposal are in the `RegexBuilder` module.
- The associated type `RegexOutput` intentionally has a `Regex` prefix. `Output` would cause confusion in standard library conforming types such as `String`, i.e. `String.Output`.

By conforming standard library types to `RegexComponent`, we allow them to be used inside the regex builder DSL as a match target. These conformances are available in the `RegexBuilder` module.

```swift
// A string represents a regex that matches the string.
extension String: RegexComponent {
  public var regex: Regex<Substring> { get }
}

// A substring represents a regex that matches the substring.
extension Substring: RegexComponent {
  public var regex: Regex<Substring> { get }
}

// A character represents a regex that matches the character.
extension Character: RegexComponent {
  public var regex: Regex<Substring> { get }
}

// A unicode scalar represents a regex that matches the scalar.
extension UnicodeScalar: RegexComponent {
  public var regex: Regex<Substring> { get }
}

// To be introduced in a future pitch.
extension CharacterClass: RegexComponent {
  public var regex: Regex<Substring> { get }
}
```

All of the regex builder DSL in the rest of this pitch will accept generic components that conform to `RegexComponent`.

### Concatenation

A regex can be viewed as a concatenation of smaller regexes. In the regex builder DSL, `RegexComponentBuilder` is the basic facility to allow developers to compose regexes by concatenation.

```swift
@resultBuilder
public enum RegexComponentBuilder { ... }
```

A closure marked with `@RegexComponentBuilder` will be transformed to produce a `Regex` by concatenating all of its components, where the result type's `Output` type will be a `Substring` followed by concatenated captures (tuple when plural).

> #### Recap: Regex capturing basics
> 
> `Regex` is a generic type with generic parameter `Output`.
>
> ```swift
> struct Regex<Output> { ... }
> ```
> 
> When a regex does not contain any capturing groups, its `Output` type is `Substring`, which represents the whole matched portion of the input.
>
> ```swift
> let noCaptures = #/a/# // => Regex<Substring>
> ```
>
> When a regex contains capturing groups, i.e. `(...)`, the `Output` type is extended as a tuple to also contain *capture types*. Capture types are tuple elements after the first element.
> 
> ```swift
> //                           ________________________________
> //                        .0 |                           .0 |
> //                  ____________________                _________
> let yesCaptures = #/a(?:(b+)c(d+))+e(f)?/# // => Regex<(Substring, Substring, Substring, Substring?)>
> //                      ---- ----   ---                            ---------  ---------  ----------
> //                    .1 | .2 |   .3 |                              .1 |       .2 |       .3 |
> //                       |    |      |                                 |          |          |
> //                       |    |      |_______________________________  |  ______  |  ________|
> //                       |    |                                        |          |
> //                       |    |______________________________________  |  ______  |
> //                       |                                             |
> //                       |_____________________________________________|
> //                                                                 ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
> //                                                                          Capture types
> ```

We introduce a new initializer `Regex.init(_:)` which accepts a `@RegexComponentBuilder` closure. This initializer is the entry point for creating a regex using the regex builder DSL.

```swift
extension Regex {
  public init<R: RegexComponent>(
    @RegexComponentBuilder _ content: () -> R
  ) where R.RegexOutput == Output
}
```

Example:

```swift
Regex {
  regex0 // Regex<Substring>
  regex1 // Regex<(Substring, Int)>
  regex2 // Regex<(Substring, Float)>
  regex3 // Regex<(Substring, Substring)>
} // Regex<(Substring, Int, Float, Substring)>
```

This above regex will be transformed to:

```swift
Regex {
  let e0 = RegexComponentBuilder.buildExpression(regex0) // Regex<Substring>
  let e1 = RegexComponentBuilder.buildExpression(regex1) // Regex<(Substring, Int)>
  let e2 = RegexComponentBuilder.buildExpression(regex2) // Regex<(Substring, Float)>
  let e3 = RegexComponentBuilder.buildExpression(regex3) // Regex<(Substring, Substring)>
  let r0 = RegexComponentBuilder.buildPartialBlock(first: e0)
  let r1 = RegexComponentBuilder.buildPartialBlock(accumulated: r0, next: e1)
  let r2 = RegexComponentBuilder.buildPartialBlock(accumulated: r1, next: e2)
  let r3 = RegexComponentBuilder.buildPartialBlock(accumulated: r2, next: e3)
  return r3
} // Regex<(Substring, Int, Float, Substring)>
```

The following example creates a regex by concatenating subpatterns.

```swift
let regex = Regex {
  "regex builder "
  "is "
  "so easy"
}
let match = try regex.prefixMatch(in: "regex builder is so easy!")
match?.0 // => "regex builder is so easy"
```

<details>
<summary>API definition</summary>

Basic methods in `RegexComponentBuilder`, e.g. `buildBlock()`, provides support for creating the most fundamental blocks. The `buildExpression` method wraps a user-provided component in a `RegexComponentBuilder.Component` structure, before passing the component to other builder methods. This is used for saving the source location of the component so that runtime errors can be reported with an accurate location.

```swift
@resultBuilder
public enum RegexComponentBuilder {
  /// Returns an empty regex.
  public static func buildBlock() -> Regex<Substring>

  /// A builder component that stores a regex component and its source location
  /// for debugging purposes.
  public struct Component<Value: RegexComponent> {
    public var value: Value
    public var file: String
    public var function: String
    public var line: Int
    public var column: Int
  }

  /// Returns a component by wrapping the component regex in `Component` and
  /// recording its source location.
  public static func buildExpression<R: RegexComponent>(
    _ regex: R, 
    file: String = #file, 
    function: String = #function, 
    line: Int = #line,
    column: Int = #column
  ) -> Component<R>
}
```

`RegexComponentBuilder` utilizes `buildPartialBlock` to be able to concatenate all components' capture types to a single result tuple. `buildPartialBlock(first:)` provides support for creating a regex from a single component, and `buildPartialBlock(accumulated:next:)` support for creating a regex from multiple results.

Before Swift supports variadic generics, `buildPartialBlock(accumulated:next:)` must be overloaded to support concatenating regexes of supported capture quantities (arities). It is overloaded up to `arity^2` times to account for all possible pairs of regexes that make up 10 captures.

In the initial version of the DSL, we plan to support regexes with up to 10 captures, as 10 captures are sufficient for most use cases. These overloads can be superceded by variadic versions of `buildPartialBlock(first:)` and `buildPartialBlock(accumulated:next:)` in a future release.

```swift
extension RegexComponentBuilder {
  @_disfavoredOverload
  public static func buildPartialBlock<R: RegexComponent>(
    first r: Component<R>
  ) -> Regex<R.RegexOutput>

  // The following builder methods implement what would be possible with
  // variadic generics (using imaginary syntax) as a single method:
  //
  //   public static func buildPartialBlock<
  //     AccumulatedWholeMatch, NextWholeMatch,
  //     AccumulatedCapture..., NextCapture...,
  //     Accumulated: RegexComponent, Next: RegexComponent
  //   >(
  //     accumulated: Accumulated, next: Component<Next>
  //   ) -> Regex<(Substring, AccumulatedCapture..., NextCapture...)>
  //   where Accumulated.RegexOutput == (AccumulatedWholeMatch, AccumulatedCapture...),
  //         Next.RegexOutput == (NextWholeMatch, NextCapture...)
  
  public static func buildPartialBlock<W0, W1, C0, R0: RegexComponent, R1: RegexComponent>(
    accumulated: R0, next: Component<R1>
  ) -> Regex<(Substring, C0)> where R0.RegexOutput == W0, R1.RegexOutput == (W1, C0)
  
  public static func buildPartialBlock<W0, W1, C0, C1, R0: RegexComponent, R1: RegexComponent>(
    accumulated: R0, next: Component<R1>
  ) -> Regex<(Substring, C0, C1)> where R0.RegexOutput == W0, R1.RegexOutput == (W1, C0, C1)
  
  public static func buildPartialBlock<W0, W1, C0, C1, C2, R0: RegexComponent, R1: RegexComponent>(
    accumulated: R0, next: Component<R1>
  ) -> Regex<(Substring, C0, C1, C2)> where R0.RegexOutput == W0, R1.RegexOutput == (W1, C0, C1, C2)

  // ... `O(arity^2)` overloads of `buildPartialBlock(accumulated:next:)`
}
```

To support `if #available(...)` statements, `buildLimitedAvailability(_:)` is defined with overloads to support up to 10 captures. The overload for non-capturing regexes, due to the lack of generic constraints, must be annotated with `@_disfavoredOverload` in order not shadow other overloads. We expect that a variadic-generic version of this method will eventually superseded all of these overloads.

```swift
extension RegexComponentBuilder {
  // The following builder methods implement what would be possible with
  // variadic generics (using imaginary syntax) as a single method:
  //
  //   public static func buildLimitedAvailability<
  //     Component, WholeMatch, Capture...
  //   >(
  //     _ component: Component
  //   ) where Component.RegexOutput == (WholeMatch, Capture...)

  @_disfavoredOverload
  public static func buildLimitedAvailability<R: RegexComponent>(
    _ component: Component<R>
  ) -> Regex<Substring>
  
  public static func buildLimitedAvailability<W, C0, R: RegexComponent>(
    _ component: Component<R>
  ) -> Regex<(Substring, C0?)>
  
  public static func buildLimitedAvailability<W, C0, C1, R: RegexComponent>(
    _ component: Component<R>
  ) -> Regex<(Substring, C0?, C1?)>
  
  // ... `O(arity)` overloads of `buildLimitedAvailability(_:)`
}
```

`buildOptional` and `buildEither` are intentionally not supported due to ergonomic issues and fundamental semantic differences between regex conditionals and result builder conditionals. Please refer to the [alternatives considered](#support-buildoptional-and-buildeither) section for detailed rationale.

</details>

### Capture

Capture is a common regex feature that saves a portion of the input upon match. In regex builder, `Capture` and `TryCapture` are regex components that produce a new regex by inserting the captured pattern's whole match (`.0`) to the `.1` position of `RegexOutput`. When a transform closure is provided, the whole match (`.0`) of the captured content will be transformed to using the closure.

```swift
public struct Capture<Output>: RegexComponent { ... }
public struct TryCapture<Output>: RegexComponent { ... }
```

To do a simple capture, you provide `Capture` with a regex component or a regex component builder closure.

```swift
// Equivalent: '(CREDIT|DEBIT)'
Capture {
  ChoiceOf {
    "CREDIT"
    "DEBIT"
  }
} // `.RegexOutput == (Substring, Substring)`
```

A capture will be represented in the type signature as a slice of the input, i.e. `Substring`. To transform the captured substring into another value during matching, specify a `transform:` closure.

```swift
// This example is similar to the one above, however in this example we
// transform the result of the capture into:
// "Transaction Kind: CREDIT" or "Transaction Kind: DEBIT"
Capture {
  ChoiceOf {
    "CREDIT"
    "DEBIT"
  }
} transform: {
  "Transaction Kind: \($0)"
} // `.RegexOutput == (Substring, String)`
```

The transform closure can throw. When a transform closure throws during matching, the matching will abort and the error will be propagated directly to the top-level matching API that's being called, e.g. `Regex.wholeMatch(in:)` and `Regex.prefixMatch(in:)`. Aborting is useful for cases where you know that matching can never succeed or when you detect that an important invariant has been violated and the matching procedure needs to be aborted.

An alternative version of capture is called `TryCapture`, which works in cases where you want to transform the capture, but the transformation may return nil. When a nil is returned, the regex engine backtracks and tries an alternative. For example, `TryCapture` makes it easy to directly transform a capture by calling a failable initializer during matching.

```swift
enum TransactionKind: String {
  case credit = "CREDIT"
  case debit = "DEBIT"
}

TryCapture {
  ChoiceOf {
    "CREDIT"
    "DEBIT"
  }
} transform: {
  // This initializer may return nil which is why we used TryCapture.
  TransactionKind(rawValue: String($0))
}
```

<details>
<summary>API definition</summary>

```swift
public struct Capture<Output>: RegexComponent {
  public var regex: Regex<Output> { get }
}

public struct TryCapture<Output>: RegexComponent {
  public var regex: Regex<Output> { get }
}
```

Below are `Capture` and `TryCapture` initializer variants on capture arity 0. Higher capture arities are omitted for simplicity.

```swift
extension Capture {
  public init<R: RegexComponent, W>(
    _ component: R
  ) where Output == (Substring, W), R.RegexOutput == W
  
  public init<R: RegexComponent, W>(
    _ component: R, as reference: Reference<W>
  ) where Output == (Substring, W), R.RegexOutput == W
  
  public init<R: RegexComponent, W, NewCapture>(
    _ component: R,
    transform: @Sendable @escaping (W) throws -> NewCapture
  ) where Output == (Substring, NewCapture), R.RegexOutput == W
  
  public init<R: RegexComponent, W>(
    @RegexComponentBuilder _ component: () -> R
  ) where Output == (Substring, W), R.RegexOutput == W

  // ... `O(arity)` overloads
}
  
extension TryCapture {
  public init<R: RegexComponent, W, NewCapture>(
    _ component: R,
    transform: @Sendable @escaping (W) throws -> NewCapture?
  ) where Output == (Substring, NewCapture), R.RegexOutput == W
  
  public init<R: RegexComponent, W, NewCapture>(
    @RegexComponentBuilder _ component: () -> R,
    transform: @Sendable @escaping (W) throws -> NewCapture?
  ) where Output == (Substring, NewCapture), R.RegexOutput == W

  // ... `O(arity)` overloads
}
```

</details>

### Mapping Output

In addition to transforming individual captures within a regex, you can also map the output of an entire regex to a different output type. You can use the `mapOutput(_:)` methods to reorder captures, flatten nested optionals, or create instances of a custom type.

This example shows how you can transform the output of a regex with three capture groups into an instance of a custom `SemanticVersion` type, matching strings such as `"1.0.0"` or `"1.0"`:

```swift
struct SemanticVersion: Hashable {
  var major, minor, patch: Int
}

let semverRegex = Regex {
  TryCapture(OneOrMore(.digit)) { Int($0) }
  "."
  TryCapture(OneOrMore(.digit)) { Int($0) }
  Optionally {
    "."
    TryCapture(OneOrMore(.digit)) { Int($0) }
  }
}.mapOutput { _, c1, c2, c3 in
  SemanticVersion(major: c1, minor: c2, patch: c3 ?? 0)
}

let semver1 = "1.11.4".firstMatch(of: semverRegex)?.output
// semver1 == SemanticVersion(major: 1, minor: 11, patch: 4)
let semver2 = "0.6".firstMatch(of: semverRegex)?.output
// semver2 == SemanticVersion(major: 0, minor: 6, patch: 0)
```

<details>
<summary>API definition</summary>

Note: This extension is defined in the standard library, not the `RegexBuilder` module.

```swift
extension Regex {
  /// Returns a regex that transforms its matches using the given closure.
  ///
  /// When you call `mapOutput(_:)` on a regex, you change the type of
  /// output available on each match result. The `body` closure is called 
  /// when each match is found to transform the result of the match.
  ///
  /// - Parameter body: A closure for transforming the output of this
  ///   regex. 
  /// - Returns: A regex that has `NewOutput` as its output type.
  func mapOutput<NewOutput>(_ body: @escaping (Output) -> NewOutput) -> Regex<NewOutput>
}
```
</details>

### Reference

Reference is a feature that can be used to achieve named captures and named backreferences from textual regexes. Simply state what type the reference will hold on to and you can use it later once you've matched a string to get back a specific capture. Note the type you pass to reference will be whatever the result of a capture's transform is. A capture with no transform always has a reference type of `Substring`.

```swift
let kind = Reference(Substring.self)

let regex = Capture(as: kind) {
  ChoiceOf {
    "CREDIT"
    "DEBIT"
  }
}

let input = "CREDIT"
if let result = try regex.firstMatch(in: input) {
  print(result[kind]) // Optional("CREDIT")
}
```

Capturing stores the most recently captured content, and references can be used as a name to look up the result of matching. The reference itself can also be used within a regex (commonly called a "backreference") to match the most recently captured content during matching.

```swift
let a = Reference(Substring.self)
let b = Reference(Substring.self)
let c = Reference(Substring.self)
let regex = Regex {
  Capture("abc", as: a)
  Capture("def", as: b)
  ZeroOrMore {
    Capture("hij", as: c)
  }
  a
  Capture(b)
}

if let result = try regex.firstMatch(in: "abcdefabcdef") {
  print(result[a]) // => Optional("abc")
  print(result[b]) // => Optional("def")
  print(result[c]) // => nil
}
```

A regex is considered invalid when it contains a use of reference without it ever being used as the `as:` argument to an initializer of `Capture` or `TryCapture` in the regex. When this occurs in the regex builder DSL, a runtime error will be reported.

Similarly, the argument to a `Regex.Match.subscript(_:)` must have been used as the `as:` argument to an initializer of `Capture` or `TryCapture` in the regex that produced the match.

<details>
<summary>API definition</summary>
  
```swift
/// A reference to a regex capture.
public struct Reference<Capture>: RegexComponent {
  public init(_ captureType: Capture.Type = Capture.self)
  public var regex: Regex<Capture>
}

extension Capture {
  public init<R: RegexComponent, W, NewCapture>(
    _ component: R,
    as reference: Reference<NewCapture>,
    transform: @escaping (Substring) throws -> NewCapture
  ) where Output == (Substring, NewCapture), R.RegexOutput == W
  
  public init<R: RegexComponent, W>(
    as reference: Reference<W>,
    @RegexComponentBuilder _ component: () -> R
  ) where Output == (Substring, W), R.RegexOutput == W

  // ... `O(arity)` overloads
}
  
extension TryCapture {
  public init<R: RegexComponent, W, NewCapture>(
    _ component: R,
    as reference: Reference<NewCapture>,
    transform: @escaping (Substring) throws -> NewCapture?
  ) where Output == (Substring, NewCapture), R.RegexOutput == W
  
  public init<R: RegexComponent, W, NewCapture>(
    as reference: Reference<NewCapture>,
    @RegexComponentBuilder _ component: () -> R,
    transform: @escaping (Substring) throws -> NewCapture?
  ) where Output == (Substring, NewCapture), R.RegexOutput == W

  // ... `O(arity)` overloads
}

extension Regex.Match {
  /// Returns the capture referenced by the given reference.
  ///
  /// - Precondition: The reference must have been captured in the regex that produced this match.
  public subscript<Capture>(_ reference: Reference<Capture>) -> Capture? { get }
}
```

</details>

### Alternation

An alternation is used to match one of multiple patterns. When one pattern in an alternation does not match successfully, the regex engine tries the next pattern until there's a successful match. An alternation wraps its underlying patterns' capture types in an `Optional` and concatenates them together, first to last.

```swift
let choice = ChoiceOf {
  regex0 // Regex<Substring>
  regex1 // Regex<(Substring, Int)>
  regex2 // Regex<(Substring, Float)>
  regex3 // Regex<(Substring, Substring)>
} // => Regex<(Substring, Int?, Float?, Substring?)>
```

`AlternationBuilder` is a result builder type for creating alternations from components of a block.

```swift
@resultBuilder
public struct AlternationBuilder { ... }
```

To the developer, the top-level API is a type named `ChoiceOf`. This type has an initializer that accepts an `@AlternationBuilder` closure.

```swift
public struct ChoiceOf<Output>: RegexComponent {
  ...
  public init<R: RegexComponent>(
    @AlternationBuilder builder: () -> R
  ) where R.RegexOutput == Output
}
```

For example, the following code creates an alternation of two subpatterns.

```swift
let regex = Regex {
  ChoiceOf {
    "CREDIT"
    "DEBIT"
  }
}
let match = try regex.prefixMatch(in: "DEBIT    04032020    Payroll $69.73")
match?.0 // => "DEBIT"
```

<details>
<summary>API definition</summary>

`AlternationBuilder` is mostly similar to `RegexComponent` with the following distinctions:
- Empty blocks are not supported.
- Capture types are wrapped in a layer of `Optional` before being concatenated in the resulting `Output` type.
- `buildEither(first:)` and `buildEither(second:)` are overloaded for each supported capture arity because they need to wrap capture types in `Optional`.

```swift
public struct ChoiceOf<Output>: RegexComponent {
  public var regex: Regex<Output> { get }
  public init<R: RegexComponent>(
    @AlternationBuilder builder: () -> R
  ) where R.RegexOutput == Output
}

@resultBuilder
public enum AlternationBuilder {
  public typealias Component<Value> = RegexComponentBuilder.Component<Value>

  /// Returns a component by wrapping the component regex in `Component` and
  /// recording its source location.
  public static func buildExpression<R: RegexComponent>(
    _ regex: R, 
    file: String = #file, 
    function: String = #function, 
    line: Int = #line,
    column: Int = #column
  ) -> Component<R>

  // The following builder methods implement what would be possible with
  // variadic generics (using imaginary syntax) as a single method:
  //
  //   public static func buildPartialBlock<
  //     R, WholeMatch, Capture...
  //   >(
  //     first component: Component<R>
  //   ) -> Regex<(Substring, Capture?...)>
  //   where Component.RegexOutput == (WholeMatch, Capture...),

  @_disfavoredOverload
  public static func buildPartialBlock<R: RegexComponent>(
    first r: Component<R>
  ) -> Regex<Substring>

  public static func buildPartialBlock<W, C0, R: RegexComponent>(
    first r: Component<R>
  ) -> Regex<(Substring, C0?)> where R.RegexOutput == (W, C0)

  public static func buildPartialBlock<W, C0, C1, R: RegexComponent>(
    first r: Component<R>
  ) -> Regex<(Substring, C0?, C1?)> where R.RegexOutput == (W, C0, C1)

  // The following builder methods implement what would be possible with
  // variadic generics (using imaginary syntax) as a single method:
  //
  //   public static func buildPartialBlock<
  //     AccumulatedWholeMatch, NextWholeMatch,
  //     AccumulatedCapture..., NextCapture...,
  //     Accumulated: RegexComponent, Next: RegexComponent
  //   >(
  //     accumulated: Accumulated, next: Component<Next>
  //   ) -> Regex<(Substring, AccumulatedCapture..., NextCapture...)>
  //   where Accumulated.RegexOutput == (AccumulatedWholeMatch, AccumulatedCapture...),
  //         Next.RegexOutput == (NextWholeMatch, NextCapture...)
  
  public static func buildPartialBlock<W0, W1, C0, R0: RegexComponent, R1: RegexComponent>(
    accumulated: R0, next: Component<R1>
  ) -> Regex<(Substring, C0?)>  where R0.RegexOutput == W0, R1.RegexOutput == (W1, C0)
  
  public static func buildPartialBlock<W0, W1, C0, C1, R0: RegexComponent, R1: RegexComponent>(
    accumulated: R0, next: Component<R1>
  ) -> Regex<(Substring, C0?, C1?)>  where R0.RegexOutput == W0, R1.RegexOutput == (W1, C0, C1)
  
  public static func buildPartialBlock<W0, W1, C0, C1, C2, R0: RegexComponent, R1: RegexComponent>(
    accumulated: R0, next: Component<R1>
  ) -> Regex<(Substring, C0?, C1?, C2?)> where R0.RegexOutput == W0, R1.RegexOutput == (W1, C0, C1, C2)

  // ... `O(arity^2)` overloads of `buildPartialBlock(accumulated:next:)`
}

extension AlternationBuilder {
  // The following builder methods implement what would be possible with
  // variadic generics (using imaginary syntax) as a single method:
  //
  //   public static func buildLimitedAvailability<
  //     Component, WholeMatch, Capture...
  //   >(
  //     _ component: Component
  //   ) -> Regex<(Substring, Capture?...)>
  //   where Component.RegexOutput == (WholeMatch, Capture...)

  @_disfavoredOverload
  public static func buildLimitedAvailability<R: RegexComponent>(
    _ component: Component<R>
  ) -> Regex<Substring>
  
  public static func buildLimitedAvailability<W, C0, R: RegexComponent>(
    _ component: Component<R>
  ) -> Regex<(Substring, C0?)>
  
  public static func buildLimitedAvailability<W, C0, C1, R: RegexComponent>(
    _ component: Component<R>
  ) -> Regex<(Substring, C0?, C1?)>
  
  // ... `O(arity)` overloads of `buildLimitedAvailability(_:)`
  
  public static func buildLimitedAvailability<W, C0, C1, C2, C3, C4, C5, C6, C7, C8, C9, R: RegexComponent>(
    _ component: Component<R>
  ) -> Regex<(Substring, C0?, C1?, C2?, C3?, C4?, C5?, C6?, C7?, C8, C9?)> where R.RegexOutput == (W, C0, C1, C2, C3, C4, C5, C6, C7, C8, C9)
}
```

</details>

### Repetition

One of the most useful features of regex is repetition, aka. quantification, as it allows you to match a specific range of number of occurrences of a subpattern. Regex builder provides 5 repetition components: `One`, `OneOrMore`, `ZeroOrMore`, `Optionally`, and `Repeat`.

```swift
public struct One<Output>: RegexComponent { ... }
public struct OneOrMore<Output>: RegexComponent { ... }
public struct ZeroOrMore<Output>: RegexComponent { ... }
public struct Optionally<Output>: RegexComponent { ... }
public struct Repeat<Output>: RegexComponent { ... }
```

| Repetition in regex builder | Textual regex equivalent |
|-----------------------------|--------------------------|
| `One(...)`                  | `...`                    |
| `OneOrMore(...)`            | `...+`                   |
| `ZeroOrMore(...)`           | `...*`                   |
| `Optionally(...)`           | `...?`                   |
| `Repeat(..., count: n)`     | `...{n}`                 |
| `Repeat(..., n...)`         | `...{n,}`                |
| `Repeat(..., n...m)`        | `...{n,m}`               |

`One`, `OneOrMore` and count-based `Repeat` are quantifiers that produce a new regex with the original capture types. Their `Output` type is `Substring` followed by the component's capture types. `ZeroOrMore`, `Optionally`, and range-based `Repeat` are quantifiers that produce a new regex with optional capture types. Their `Output` type is `Substring` followed by the component's capture types wrapped in `Optional`.

| Quantifier                                           | Component `Output`         | Result `Output`            |
|------------------------------------------------------|----------------------------|----------------------------|
| `One`<br>`OneOrMore`<br>`Repeat(..., count: ...)`    | `(WholeMatch, Capture...)` | `(Substring, Capture...)`  |
| `One`<br>`OneOrMore`<br>`Repeat(..., count: ...)`    | `WholeMatch` (non-tuple)   | `Substring`                |
| `ZeroOrMore`<br>`Optionally`<br>`Repeat(..., n...m)` | `(WholeMatch, Capture...)` | `(Substring, Capture?...)` |
| `ZeroOrMore`<br>`Optionally`<br>`Repeat(..., n...m)` | `WholeMatch` (non-tuple)   | `Substring`                |

<details>
<summary>API definition</summary>

```swift
public struct One<Output>: RegexComponent {
  public var regex: Regex<Output> { get }
}
 
public struct OneOrMore<Output>: RegexComponent {
  public var regex: Regex<Output> { get }
}

public struct ZeroOrMore<Output>: RegexComponent {
  public var regex: Regex<Output> { get }
}

public struct Optionally<Output>: RegexComponent {
  public var regex: Regex<Output> { get }
}

public struct Repeat<Output>: RegexComponent {
  public var regex: Regex<Output> { get }
}
```

Due to the lack of variadic generics, initializers must be overloaded for every supported capture arity.

```swift
extension One {
  // The following builder methods implement what would be possible with
  // variadic generics (using imaginary syntax) as a single set of methods:
  //
  //   public init<
  //     Component: RegexComponent, WholeMatch, Capture...
  //   >(
  //     _ component: Component,
  //     _ behavior: RegexRepetitionBehavior = .eager
  //   )
  //   where Output == (Substring, Capture...)>,
  //         Component.RegexOutput == (WholeMatch, Capture...)
  //
  //   public init<
  //     Component: RegexComponent, WholeMatch, Capture...
  //   >(
  //     _ behavior: RegexRepetitionBehavior = .eager,
  //     @RegexComponentBuilder _ component: () -> Component
  //   )
  //   where Output == (Substring, Capture...),
  //         Component.RegexOutput == (WholeMatch, Capture...)

  @_disfavoredOverload
  public init<Component: RegexComponent>(
    _ component: Component,
    _ behavior: RegexRepetitionBehavior? = nil
  ) where Output == Substring
  
  @_disfavoredOverload
  public init<Component: RegexComponent>(
    _ behavior: RegexRepetitionBehavior? = nil,
    @RegexComponentBuilder _ component: () -> Component
  ) where Output == Substring
  
  public init<W, C0, Component: RegexComponent>(
    _ component: Component,
    _ behavior: RegexRepetitionBehavior? = nil
  ) where Output == (Substring, C0), Component.RegexOutput == (W, C0)
  
  public init<W, C0, Component: RegexComponent>(
    _ behavior: RegexRepetitionBehavior? = nil,
    @RegexComponentBuilder _ component: () -> Component
  ) where Output == (Substring, C0), Component.RegexOutput == (W, C0)
  
  // ... `O(arity)` overloads
}
 
extension OneOrMore {
  // The following builder methods implement what would be possible with
  // variadic generics (using imaginary syntax) as a single set of methods:
  //
  //   public init<
  //     Component: RegexComponent, WholeMatch, Capture...
  //   >(
  //     _ component: Component,
  //     _ behavior: RegexRepetitionBehavior = .eager
  //   )
  //   where Output == (Substring, Capture...)>,
  //         Component.RegexOutput == (WholeMatch, Capture...)
  //
  //   public init<
  //     Component: RegexComponent, WholeMatch, Capture...
  //   >(
  //     _ behavior: RegexRepetitionBehavior = .eager,
  //     @RegexComponentBuilder _ component: () -> Component
  //   )
  //   where Output == (Substring, Capture...),
  //         Component.RegexOutput == (WholeMatch, Capture...)

  @_disfavoredOverload
  public init<Component: RegexComponent>(
    _ component: Component,
    _ behavior: RegexRepetitionBehavior? = nil
  ) where Output == Substring
  
  @_disfavoredOverload
  public init<Component: RegexComponent>(
    _ behavior: RegexRepetitionBehavior? = nil,
    @RegexComponentBuilder _ component: () -> Component
  ) where Output == Substring
  
  public init<W, C0, Component: RegexComponent>(
    _ component: Component,
    _ behavior: RegexRepetitionBehavior? = nil
  ) where Output == (Substring, C0), Component.RegexOutput == (W, C0)
  
  public init<W, C0, Component: RegexComponent>(
    _ behavior: RegexRepetitionBehavior? = nil,
    @RegexComponentBuilder _ component: () -> Component
  ) where Output == (Substring, C0), Component.RegexOutput == (W, C0)
  
  // ... `O(arity)` overloads
}

extension ZeroOrMore {
  // The following builder methods implement what would be possible with
  // variadic generics (using imaginary syntax) as a single set of methods:
  //
  //   public init<
  //     Component: RegexComponent, WholeMatch, Capture...
  //   >(
  //     _ component: Component,
  //     _ behavior: RegexRepetitionBehavior = nil
  //   )
  //   where Output == (Substring, Capture?...)>,
  //         Component.RegexOutput == (WholeMatch, Capture...)
  //
  //   public init<
  //     Component: RegexComponent, WholeMatch, Capture...
  //   >(
  //     _ behavior: RegexRepetitionBehavior? = nil,
  //     @RegexComponentBuilder _ component: () -> Component
  //   )
  //   where Output == (Substring, Capture?...),
  //         Component.RegexOutput == (WholeMatch, Capture...)

  @_disfavoredOverload
  public init<Component: RegexComponent>(
    _ component: Component,
    _ behavior: RegexRepetitionBehavior? = nil
  ) where Output == Substring
  
  @_disfavoredOverload
  public init<Component: RegexComponent>(
    _ behavior: RegexRepetitionBehavior? = nil,
    @RegexComponentBuilder _ component: () -> Component
  ) where Output == Substring
  
  public init<W, C0, Component: RegexComponent>(
    _ component: Component,
    _ behavior: RegexRepetitionBehavior? = nil
  ) where Output == (Substring, C0?), Component.RegexOutput == (W, C0)
  
  public init<W, C0, Component: RegexComponent>(
    _ behavior: RegexRepetitionBehavior? = nil,
    @RegexComponentBuilder _ component: () -> Component
  ) where Output == (Substring, C0?), Component.RegexOutput == (W, C0)
  
  // ... `O(arity)` overloads
}

extension Optionally {
  // The following builder methods implement what would be possible with
  // variadic generics (using imaginary syntax) as a single set of methods:
  //
  //   public init<
  //     Component: RegexComponent, WholeMatch, Capture...
  //   >(
  //     _ component: Component,
  //     _ behavior: RegexRepetitionBehavior? = nil
  //   )
  //   where Output == (Substring, Capture?...),
  //         Component.RegexOutput == (WholeMatch, Capture...)
  //
  //   public init<
  //     Component: RegexComponent, WholeMatch, Capture...
  //   >(
  //     _ behavior: RegexRepetitionBehavior? = nil,
  //     @RegexComponentBuilder _ component: () -> Component
  //   )
  //   where Output == (Substring, Capture?...)>,
  //         Component.RegexOutput == (WholeMatch, Capture...)
  
  @_disfavoredOverload
  public init<Component: RegexComponent>(
    _ component: Component,
    _ behavior: RegexRepetitionBehavior? = nil
  ) where Output == Substring
  
  @_disfavoredOverload
  public init<Component: RegexComponent>(
    _ behavior: RegexRepetitionBehavior? = nil,
    @RegexComponentBuilder _ component: () -> Component
  ) where Output == Substring
  
  public init<W, C0, Component: RegexComponent>(
    _ component: Component,
    _ behavior: RegexRepetitionBehavior? = nil
  ) where Output == (Substring, C0?), Component.RegexOutput == (W, C0)
  
  public init<W, C0, Component: RegexComponent>(
    _ behavior: RegexRepetitionBehavior? = nil,
    @RegexComponentBuilder _ component: () -> Component
  ) where Output == (Substring, C0?), Component.RegexOutput == (W, C0)
  
  // ... `O(arity)` overloads
}

extension Repeat {
  // The following builder methods implement what would be possible with
  // variadic generics (using imaginary syntax) as a single set of methods:
  //
  //   public init<
  //     Component: RegexComponent, WholeMatch, Capture...
  //   >(
  //     _ component: Component,
  //     count: Int,
  //     _ behavior: RegexRepetitionBehavior? = nil
  //   )
  //   where Output == (Substring, Capture...),
  //         Component.RegexOutput == (WholeMatch, Capture...)
  //
  //   public init<
  //     Component: RegexComponent, WholeMatch, Capture...
  //   >(
  //     count: Int,
  //     _ behavior: RegexRepetitionBehavior? = nil,
  //     @RegexComponentBuilder _ component: () -> Component
  //   )
  //   where Output == (Substring, Capture...),
  //         Component.RegexOutput == (WholeMatch, Capture...)
  //
  //   public init<
  //     Component: RegexComponent, WholeMatch, Capture..., RE: RangeExpression
  //   >(
  //     _ component: Component,
  //     _ expression: RE,
  //     _ behavior: RegexRepetitionBehavior? = nil
  //   )
  //   where Output == (Substring, Capture?...),
  //         Component.RegexOutput == (WholeMatch, Capture...)
  //
  //   public init<
  //     Component: RegexComponent, WholeMatch, Capture..., RE: RangeExpression
  //   >(
  //     _ expression: RE,
  //     _ behavior: RegexRepetitionBehavior? = nil,
  //     @RegexComponentBuilder _ component: () -> Component
  //   )
  //   where Output == (Substring, Capture?...),
  //         Component.RegexOutput == (WholeMatch, Capture...)
  
  // Nullary

  @_disfavoredOverload
  public init<Component: RegexComponent>(
    _ component: Component,
    count: Int,
    _ behavior: RegexRepetitionBehavior? = nil
  ) where Output == Substring, R.Bound == Int
  
  @_disfavoredOverload
  public init<Component: RegexComponent>(
    count: Int,
    _ behavior: RegexRepetitionBehavior? = nil,
    @RegexComponentBuilder _ component: () -> Component
  ) where Output == Substring, R.Bound == Int
  
  @_disfavoredOverload
  public init<Component: RegexComponent, RE: RangeExpression>(
    _ component: Component,
    _ expression: RE,
    _ behavior: RegexRepetitionBehavior? = nil
  ) where Output == Substring, R.Bound == Int
  
  @_disfavoredOverload
  public init<Component: RegexComponent, RE: RangeExpression>(
    _ expression: RE,
    _ behavior: RegexRepetitionBehavior? = nil,
    @RegexComponentBuilder _ component: () -> Component
  ) where Output == Substring, R.Bound == Int
  
  
  // Unary

  public init<W, C0, Component: RegexComponent>(
    _ component: Component,
    count: Int,
    _ behavior: RegexRepetitionBehavior? = nil
  )
  where Output == (Substring, C0),
        Component.RegexOutput == (Substring, C0),
        R.Bound == Int
  
  public init<W, C0, Component: RegexComponent>(
    count: Int,
    _ behavior: RegexRepetitionBehavior? = nil,
    @RegexComponentBuilder _ component: () -> Component
  )
  where Output == (Substring, C0),
        Component.RegexOutput == (Substring, C0),
        R.Bound == Int
  
  public init<W, C0, Component: RegexComponent, RE: RangeExpression>(
    _ component: Component,
    _ expression: RE,
    _ behavior: RegexRepetitionBehavior? = nil
  )
  where Output == (Substring, C0?),
        Component.RegexOutput == (W, C0),
        R.Bound == Int
  
  public init<W, C0, Component: RegexComponent, RE: RangeExpression>(
    _ expression: RE,
    _ behavior: RegexRepetitionBehavior? = nil,
    @RegexComponentBuilder _ component: () -> Component
  )
  where Output == (Substring, C0?),
        Component.RegexOutput == (W, C0),
        R.Bound == Int
  
  // ... `O(arity)` overloads
}
```

</details>

#### Repetition behavior

Repetition behavior defines how eagerly a repetition component should match the input. Behavior can be unspecified, in which case it will default to `.eager` unless an option is provided to change the default (see [Unicode for String Processing](https://github.com/apple/swift-experimental-string-processing/blob/main/Documentation/Evolution/ProposalOverview.md#unicode-for-string-processing)).

```swift
/// Specifies how much to attempt to match when using a quantifier.
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

| Repetition behavior in regex builder | Textual regex equivalent |
|--------------------------------------|--------------------------|
| `.eager`                             | no suffix                |
| `.reluctant`                         | suffix `?`               |
| `.possessive`                        | suffix `+`               |

To demonstrate how each repetition behavior works, let's look at the following
example. Suppose we want to make a regex that wants to capture an html tag, e.g.
`<code>`. We might start with something like the following:


```swift
let tag = Reference(Substring.self)

let htmlRegex = Regex {
  "<"
  Capture(as: tag) {
    // Remember, the default behavior is .eager here!
    OneOrMore(.any)
  }
  ">"
}

let input = #"<code>print("hello world!")</code>"#

if let result = htmlRegex.firstMatch(in: input) {
  print(result[tag])
}
```

The code above prints `code>print("hello world!")</code`, which is unexpected. This is because `OneOrMore(.any)` has eager behavior by default, and it matched as many characters as possible.

If we change `OneOrMore(.any)` to `OneOrMore(.any, .possessive)`, matching fails. What happened in this case was that the regex found our starting "<", but the repetition regex component `OneOrMore(.any, .possessive)` ran all the way to the end of the string (because we're asking for any character). After reaching the end, we couldn't find a match for the end `">"` because our string was out of characters. This is intended for `.possessive` because it doesn't backtrack the string to find a match for the ending `">"`.

The desired behavior in this case is `.reluctant`, where the repetition will match as little of the input string as possible. If we use `OneOrMore(.any, .reluctant)`, the code prints expected output `<code>`.

### Anchors and Lookaheads

Anchors are a way to constrain a regex, or part of a regex, to matching particular locations within an input string. Regex builder provides anchors that correspond to regex syntax anchors. Regex builder also provides two types that represent look-ahead assertions — essentially a non-consuming sub-regex that has to match (or not match) before the regex can proceed. 

```swift
/// A regex component that matches a specific condition at a particular position
/// in an input string.
///
/// You can use anchors to guarantee that a match only occurs at certain points
/// in an input string, such as at the beginning of the string or at the end of
/// a line.
public struct Anchor: RegexComponent {
  /// An anchor that matches at the start of a line, including the start of
  /// the input string.
  ///
  /// This anchor is equivalent to `^` in regex syntax when the `m` option
  /// has been enabled or `anchorsMatchLineEndings(true)` has been called.
  public static var startOfLine: Anchor { get }

  /// An anchor that matches at the end of a line, including at the end of
  /// the input string.
  ///
  /// This anchor is equivalent to `$` in regex syntax when the `m` option
  /// has been enabled or `anchorsMatchLineEndings(true)` has been called.
  public static var endOfLine: Anchor { get }

  /// An anchor that matches at a word boundary.
  ///
  /// Word boundaries are identified using the Unicode default word boundary
  /// algorithm by default. To specify a different word boundary algorithm,
  /// see the `RegexComponent.wordBoundaryKind(_:)` method.
  ///
  /// This anchor is equivalent to `\b` in regex syntax.
  public static var wordBoundary: Anchor { get }

  /// An anchor that matches at the start of the input string.
  ///
  /// This anchor is equivalent to `\A` in regex syntax.
  public static var startOfSubject: Anchor { get }

  /// An anchor that matches at the end of the input string.
  ///
  /// This anchor is equivalent to `\z` in regex syntax.
  public static var endOfSubject: Anchor { get }

  /// An anchor that matches at the end of the input string or at the end of
  /// the line immediately before the the end of the string.
  ///
  /// This anchor is equivalent to `\Z` in regex syntax.
  public static var endOfSubjectBeforeNewline: Anchor { get }

  /// An anchor that matches at a grapheme cluster boundary.
  ///
  /// This anchor is equivalent to `\y` in regex syntax.
  public static var textSegmentBoundary: Anchor { get }

  /// An anchor that matches at the first position of a match in the input
  /// string.
  ///
  /// This anchor is equivalent to `\y` in regex syntax.
  public static var firstMatchingPositionInSubject: Anchor { get }

  /// The inverse of this anchor, which matches at every position that this 
  /// anchor does not.
  ///
  /// For the `wordBoundary` and `textSegmentBoundary` anchors, the inverted
  /// version corresponds to `\B` and `\Y`, respectively.
  public var inverted: Anchor { get }
}

/// A regex component that allows a match to continue only if its contents
/// match at the given location.
///
/// A lookahead is a zero-length assertion that its included regex matches at
/// a particular position. Lookaheads do not advance the overall matching
/// position in the input string — once a lookahead succeeds, matching continues
/// in the regex from the same position.
public struct Lookahead: RegexComponent {
  /// Creates a lookahead from the given regex component.
  public init(_ component: some RegexComponent)

  /// Creates a lookahead from the regex generated by the given builder closure.
  public init(@RegexComponentBuilder _ component: () -> some RegexComponent)
}

/// A regex component that allows a match to continue only if its contents
/// do not match at the given location.
///
/// A negative lookahead is a zero-length assertion that its included regex 
/// does not match at a particular position. Lookaheads do not advance the
/// overall matching position in the input string — once a lookahead succeeds,
/// matching continues in the regex from the same position.
public struct NegativeLookahead: RegexComponent {
  /// Creates a negative lookahead from the given regex component.
  public init(_ component: some RegexComponent)

  /// Creates a negative lookahead from the regex generated by the given builder
  /// closure.
  public init(@RegexComponentBuilder _ component: () -> some RegexComponent)
}
```

### Subpattern

In textual regex, one can refer to a subpattern to avoid duplicating the subpattern, for example:

```
(you|I) say (goodbye|hello); (?1) say (?2)
```

The above regex is equivalent to

```
(you|I) say (goodbye|hello); (you|I) say (goodbye|hello)
```

With regex builder, there is no special API required to reuse existing subpatterns, as a subpattern can be defined modularly using a `let` binding inside or outside a regex builder closure.

```swift
Regex {
  let subject = ChoiceOf {
    "I"
    "you"
  }
  let object = ChoiceOf {
    "goodbye"
    "hello"
  }
  subject
  "say"
  object
  ";"
  subject
  "say"
  object
}
```

### Scoping

Because the regex engine backtracks by default when trying to match on a string, sometimes this backtracking can be wasted performance because we don't want to try various possibilities to eventually (maybe) find a match.

In textual regexes, atomic groups (`(?>...)`) solve this problem by informing the regex engine to actually discard the backtrack location of a group, that is, defining a scope for backtracking. In regex builder, the `Local` type serves this purpose.

```swift
public struct Local<Output>: RegexComponent { ... }
```

For example, the following regex matches string `abcc` but not `abc`.

```swift
Regex {
  "a"
  Local {
    ChoiceOf {
      "bc"
      "b"
    }
  }
  "c"
}
```

If our input is `abcc`, we'll successfully find a match, however if we try to match against `abc` we won't get a match. The reason behind this is that in the `ChoiceOf` we actually matched the "bc" case first, but due to the local group we immediately disregard the backtracking location and continue to try and the rest of the regex. Since we matched the "bc", we don't have anymore string left to match the "c" and our local group will not try and attempt to match the other option, "b".

<details>
<summary>API definition</summary>

```swift
public struct Local<Output>: RegexComponent {
  public var regex: Regex<Output>

  // The following builder methods implement what would be possible with
  // variadic generics (using imaginary syntax) as a single set of methods:
  //
  //   public init<WholeMatch, Capture..., Component: RegexComponent>(
  //     @RegexComponentBuilder _ component: () -> Component
  //   ) where Output == (Substring, Capture...), Component.RegexOutput == (WholeMatch, Capture...)

  @_disfavoredOverload
  public init<Component: RegexComponent>(
    @RegexComponentBuilder _ component: () -> Component
  ) where Output == Substring

  public init<W, C0, Component: RegexComponent>(
    @RegexComponentBuilder _ component: () -> Component
  ) where Output == (Substring, C0), Component.RegexOutput == (W, C0)
  
  public init<W, C0, C1, Component: RegexComponent>(
    @RegexComponentBuilder _ component: () -> Component
  ) where Output == (Substring, C0, C1), Component.RegexOutput == (W, C0, C1)
  
  // ... `O(arity)` overloads
}
```

</details>

### Composability

Let's put everything together now and parse this example bank statement.

```
CREDIT    04062020    PayPal transfer    $4.99
CREDIT    04032020    Payroll            $69.73
DEBIT     04022020    ACH transfer       $38.25
DEBIT     03242020    IRS tax payment    $52249.98
```

Here we have 2 types of transaction kinds, CREDIT and DEBIT, we have a date
denoted by mmddyyyy, a description, and the amount paid.

```swift
enum TransactionKind: String {
  case credit = "CREDIT"
  case debit = "DEBIT"
}

struct Date {
  var month: Int
  var day: Int
  var year: Int

  init?(mmddyyyy: String) {
    ...
  }
}

let statementRegex = Regex {
  // First, let's capture the transaction kind by wrapping our `ChoiceOf` in a
  // `TryCapture` because our initializer can return nil on failure.
  TryCapture {
    ChoiceOf {
      "CREDIT"
      "DEBIT"
    }
  } transform: {
    TransactionKind(rawValue: String($0))
  }

  OneOrMore(.whitespace)

  // Next, lets represent our date as 3 separate repeat quantifiers. The first
  // two will require 2 digit characters, and the last will require 4. Then
  // we'll take the entire substring and try to parse a date out.
  TryCapture {
    Repeat(.digit, count: 2)
    Repeat(.digit, count: 2)
    Repeat(.digit, count: 4)
  } transform: {
    Date(mmddyyyy: String($0))
  }

  OneOrMore(.whitespace)

  // Next, grab the description which can be any combination of word characters,
  // digits, etc.
  Capture {
    OneOrMore(.any, .reluctant)
  }

  OneOrMore(.whitespace)

  "$"

  // Finally, we'll grab one or more digits which will represent the whole
  // dollars, match the decimal point, and finally get 2 digits which will be
  // our cents.
  TryCapture {
    OneOrMore(.digit)
    "."
    Repeat(.digit, count: 2)
  } transform: {
    Double($0)
  }
}

for match in statement.matches(of: statementRegex) {
  let (line, kind, date, description, amount) = match.output
  ...
}
```

## Source compatibility

Regex builder will be shipped in a new module named `RegexBuilder`, and thus will not affect the source compatibility of the existing code.

## Effect on ABI stability

The proposed feature does not change the ABI of existing features.

## Effect on API resilience

The proposed feature relies heavily upon overloads of `buildBlock` and `buildPartialBlock(accumulated:next:)` to work for different capture arities. In the fullness of time, we are hoping for variadic generics to supercede existing overloads. Such a change should not involve ABI-breaking modifications as it is merely a change of overload resolution.

## Future directions

### Conversion to textual regex

Sometimes it may be useful to convert a regex created using regex builder to textual regex. This may be achieved in the future by extending `RegexComponent` with a computed property.

```swift
extension RegexComponent {
  public func makeTextualRegex() -> String?
}
```

It is worth noting that the internal representation of a `Regex` is _not_ textual regex, but an efficient pattern matching bytecode compiled from an abstract syntax tree. Moreover, not every `Regex` can be converted to textual regex. Regex builder supports arbitrary types that conform to the `RegexComponent` protocol, including `CustomMatchingRegexComponent` (pitched in [String Processing Algorithms]) which can be implemented with arbitrary code. If a `Regex` contains a `CustomMatchingRegexComponent`, it cannot be converted to textual regex.

### Recursive subpatterns

Sometimes, a textual regex may also use `(?R)` or `(?0)` to recusively evaluate the entire regex. For example, the following textual regex matches "I say you say I say you say hello".

```
(you|I) say (goodbye|hello|(?R))
```

For this, `Regex` offers a special initializer that allows its pattern to recursively reference itself. This is somewhat akin to a fixed-point combinator.

```swift
extension Regex {
  public init<R: RegexComponent>(
    @RegexComponentBuilder _ content: (Regex<Substring>) -> R
  ) where R.RegexOutput == Match
}
```

With this initializer, the above regex can be expressed as the following using regex builder.

```swift
Regex { wholeSentence in
  ChoiceOf {
   "I"
   "you"
  }
  "say"
  ChoiceOf {
    "goodbye"
    "hello"
    wholeSentence
  }
}
```

There are some concerns with this design which we need to consider:
- Due to the lack of labeling, the argument to the builder closure can be arbitrarily named and cause confusion.
- When there is an initializer that accepts a result builder closure, overloading that initializer with the same argument labels could lead to bad error messages upon interor type errors.

## Alternatives considered
 
### Semicolons or parentheses instead of `One`
 
In the DSL syntax as described in the first version of this proposal, there was a problem with the use of leading-dot syntax for character classes and other "atoms" and the builder syntax:
```swift
Regex {
  .digit
  OneOrMore(.whitespace)
}
```
worked as expected, but:
```swift
Regex {
  OneOrMore(.whitespace)
  .digit
}
```
did not, because `.digit` parses as a property on `OneOrMore` rather than a regex component. This could have been resolved by making people use either semicolons:
```swift
Regex {
  OneOrMore(.whitespace);
  .digit
}
```
or parentheses:
```swift
Regex {
  OneOrMore(.whitespace)
  (.digit)
}
```

Instead we decided to introduce the quantifier `One` to resolve the ambiguity:
```swift
Regex {
  OneOrMore(.whitespace)
  One(.digit)
}
```

This increase the API surface, which is mildly undesirable, but feels much more stylistically consistent with the rest of the DSL and with Swift as whole. We also considered a "two protocol" approach that would force the use of `One` in these cases by making it impossible to use the dot-prefixed "atoms" within builder blocks, but this seems like too much heavy machinery to resolve the problem.
 
### Operators for quantification and alternation

While `ChoiceOf` and quantifier types provide a general way of creating alternations and quantifications, we recognize that some synctactic sugar can be useful for creating one-liners like in textual regexes, e.g. infix operator `|`, postfix operator `*`, etc.

```swift
// The following functions implement what would be possible with variadic
// generics (using imaginary syntax) as a single function:
//
//   public func | <
//     R0: RegexComponent, R1: RegexComponent,
//     WholeMatch0, WholeMatch1, 
//     Capture0..., Capture1...
//   >(
//     _ r0: RegexComponent,
//     _ r1: RegexComponent
//   ) -> Regex<(Substring, Capture0?..., Capture1?...)>
//     where R0.RegexOutput == (WholeMatch0, Capture0...),
//           R1.RegexOutput == (WholeMatch1, Capture1...)

@_disfavoredOverload
public func | <R0, R1>(lhs: R0, rhs: R1) -> Regex<Substring> where R0: RegexComponent, R1: RegexComponent {

public func | <R0, R1, W1, C0>(lhs: R0, rhs: R1) -> Regex<(Substring, C0?)> where R0: RegexComponent, R1: RegexComponent, R1.RegexOutput == (W1, C0)

public func | <R0, R1, W1, C0, C1>(lhs: R0, rhs: R1) -> Regex<(Substring, C0?, C1?)> where R0: RegexComponent, R1: RegexComponent, R1.RegexOutput == (W1, C0, C1)

// ... `O(arity^2)` overloads.
```

However, like `RegexComponentBuilder.buildPartialBlock(accumulated:next:)`, operators such as `|`, `+`, `*`, `.?` require a large number of overloads to work with regexes of every capture arity, compounded by the fact that operator type checking is prone to performance issues in Swift. Here is a list of

| Opreator      | Meaning                   | Required number of overloads |
|---------------|---------------------------|------------------------------|
| Infix `\|`    | Choice of two             | `O(arity^2)`                 |
| Postfix `*`   | Zero or more eagerly      | `O(arity)`                   |
| Postfix `*?`  | Zero or more reluctantly  | `O(arity)`                   |
| Postfix `*+`  | Zero or more possessively | `O(arity)`                   |
| Postfix `+`   | One or more eagerly       | `O(arity)`                   |
| Postfix `+?`  | One or more reluctantly   | `O(arity)`                   |
| Postfix `++`  | One or more possessively  | `O(arity)`                   |
| Postfix `.?`  | Optionally eagerly        | `O(arity)`                   |
| Postfix `.??` | Optionally reluctantly    | `O(arity)`                   |
| Postfix `.?+` | Optionally possessively   | `O(arity)`                   |

 When variadic generics are supported in the future, we may be able to define one function per operator and reduce type checking burdens.
 
### Postfix `capture` and `tryCapture` methods

An earlier iteration of regex builder declared `capture` and `tryCapture` as methods on `RegexComponent`, meaning that you can append `.capture(...)` to any subpattern within a regex to capture it. For example:

```swift
Regex {
  OneOrMore {
    r0.capture()
    r1
  }.capture()
} // => Regex<(Substring, Substring, Substring)>
```

However, there are two shortcomings of this design:

1. When a subpattern to be captured contains multiple components, the developer has to explicitly group them using a `Regex { ... }` block.

    ```swift
    let emailPattern = Regex {
      let word = OneOrMore(.word)
      Regex { // <= Had to explicitly group multiple components
        ZeroOrMore {
          word
          "."
        }
        word
      }.capture()
      "@"
      Regex {
        word
        OneOrMore {
          "."
          word
        }
      }.capture()
    } // => Regex<(Substring, Substring, Substring)>
    ```

2. When there are nested captures, it is harder to number the captures visually because the order `capture()` appears is flipped in the postfix (method) notation.

    ```swift
    let emailSuffixPattern = Regex {
      "@"
      Regex {
        word
        OneOrMore {
          "."
          word.capture() // top-level domain (.0)
        }
      }.capture() // full domain (.1)
    } // => Regex<(Substring, Substring, Substring)>
      //
      //          full domain ^~~~~~~~~
      //                top-level domain ^~~~~~~~~
    ```
    
    In comparison, prefix notation (`Capture` and `TryCapture` as a types) makes it easier to visually capture captures as you can number captures in the order they appear from top to bottom. This is consistent with textual regexes where capturing groups are numbered by the left parenthesis of the group from left to right.

    ```swift
    let emailSuffixPattern = Regex {
      Capture { // full domain (.0)
        word
        OneOrMore {
          "."
          Capture(word) // top-level domain (.1)
        }
      }
    } // => Regex<(Substring, Substring, Substring)>
      //
      //          full domain ^~~~~~~~~
      //                top-level domain ^~~~~~~~~
    ```
  
### Unify quantifiers under `Repeat`

Since `Repeat` is the most general version of quantifiers, one could argue for all quantifiers to be unified under the type `Repeat`, for example:

```swift
Repeat(oneOrMore: r)
Repeat(zeroOrMore: r)
Repeat(optionally: r)
```

However, given that one-or-more (`+`), zero-or-more (`*`) and optional (`?`) are the most common quantifiers in textual regexes, we believe that these quantifiers deserve their own type and should be written as a single word instead of two. This can also reduce visual clutter when the quantification is used in multiple places of a regex.

### Free functions instead of types

One could argue that type such as `OneOrMore<Output>` could be defined as a top-level function that returns `Regex`. While it is entirely possible to do so, it would lose the name scoping benefits of a type and pollute the top-level namespace with `O(arity^2)` overloads of quantifiers, `capture`, `tryCapture`, etc. This could be detrimental to the usefulness of code completion.

Another reason to use types instead of free functions is consistency with existing result-builder-based DSLs such as SwiftUI.

### Support `buildOptional` and `buildEither`

To support `if` statements, an earlier iteration of this proposal defined `buildEither(first:)`, `buildEither(second:)` and `buildOptional(_:)` as the following:

```swift
extension RegexComponentBuilder {
  public static func buildEither<
    Component, WholeMatch, Capture...
  >(
    first component: Component
  ) -> Regex<(Substring, Capture...)>
  where Component.RegexOutput == (WholeMatch, Capture...)

  public static func buildEither<
    Component, WholeMatch, Capture...
  >(
    second component: Component
  ) -> Regex<(Substring, Capture...)>
  where Component.RegexOutput == (WholeMatch, Capture...)

  public static func buildOptional<
    Component, WholeMatch, Capture...
  >(
    _ component: Component?
  ) where Component.RegexOutput == (WholeMatch, Capture...)
}
```

However, multiple-branch control flow statements (e.g. `if`-`else` and `switch`) would need to be required to produce either the same regex type, which is limiting, or an "either-like" type, which can be difficult to work with when nested. Unlike `ChoiceOf`, producing a tuple of optionals is not an option, because the branch taken would be decided when the builder closure is executed, and it would cause capture numbering to be inconsistent with conventional regex.

Moreover, result builder conditionals does not work the same way as regex conditionals.  In regex conditionals, the conditions are themselves regexes and are evaluated by the regex engine during matching, whereas result builder conditionals are evaluated as part of the builder closure.  We hope that a future result builder feature will support "lifting" control flow conditions into the DSL domain, e.g. supporting `Regex<Bool>` as a condition.

### Flatten optionals

With the proposed design, `ChoiceOf` with `AlternationBuilder` wraps every component's capture type with an `Optional`. This means that any `ChoiceOf` with optional-capturing components would lead to a doubly-nested optional captures. This could make the result of matching harder to use.

```swift
ChoiceOf {
  OneOrMore(Capture(.digit)) // RegexOutput == (Substring, Substring)
  Optionally {
    ZeroOrMore(Capture(.word)) // RegexOutput == (Substring, Substring?)
    "a"
  } // RegexOutput == (Substring, Substring??)
} // RegexOutput == (Substring, Substring?, Substring???)
```

One way to improve this could be overloading quantifier initializers (e.g. `ZeroOrMore.init(_:)`) and `AlternationBuilder.buildPartialBlock` to flatten any optionals upon composition. However, this would be non-trivial. Quantifier initializers would need to be overloaded `O(2^arity)` times to account for all possible positions of `Optional` that may appear in the `Output` tuple. Even worse, `AlternationBuilder.buildPartialBlock` would need to be overloaded `O(arity!)` times to account for all possible combinations of two `Output` tuples with all possible positions of `Optional` that may appear in one of the `Output` tuples.

### Structured rather than flat captures

We propose inferring capture types in such a way as to align with the traditional numbering of backreferences. This is because much of the motivation behind providing regex in Swift is their familiarity.

If we decided to deprioritize this motivation, there are opportunities to infer safer, more ergonomic, and arguably more intuitive types for captures. For example, to be consistent with traditional regex backreferences quantifications of multiple or nested captures had to produce parallel arrays rather than an array of tuples.

```swift
OneOrMore {
  Capture {
    OneOrMore(.hexDigit)
  }
  ".."
  Capture {
    OneOrMore(.hexDigit)
  }
}

// Flat capture types:
// => `RegexOutput == (Substring, Substring, Substring)>`

// Structured capture types:
// => `RegexOutput == (Substring, (Substring, Substring))`
```

Similarly, an alternation of multiple or nested captures could produce a structured alternation type (or an anonymous sum type) rather than flat optionals.

This is cool, but it adds extra complexity to regex builder and it isn't as clear because the generic type no longer aligns with the traditional regex backreference numbering. We think the consistency of the flat capture types trumps the added safety and ergonomics of the structured capture types.

### Unify `Capture` with `TryCapture`

The primary difference between `Capture` and `TryCapture` at the API level is that `TryCapture`'s transform closure returns an `Optional` of the target type, whereas `Capture`'s transform closure returns the target type. `TryCapture` would cause the regex engine to backtrack when the transform closure returns nil, whereas `Capture` does not backtrack.

It has been argued in the review thread that the distinction between `Capture` and `TryCapture` need not be reflected at the type name level, but could be differentiated by argument label, e.g. `transform:`/`tryTransform:` or `map:`/`compactMap:`. However, doing so may cause ambiguity in cases where the transform closure is not the second, but the first, trailing closure in the initializer.

```swift
extension Capture {
  public init<R: RegexComponent, W, NewCapture>(
    _ component: R,
    map: @escaping (Substring) throws -> NewCapture
  ) where Output == (Substring, NewCapture), R.RegexOutput == W

  public init<R: RegexComponent, W, NewCapture>(
    _ component: R,
    compactMap: @escaping (Substring) throws -> NewCapture?
  ) where Output == (Substring, NewCapture), R.RegexOutput == W
}
```

In this case, since the argument label will not be specfied for the first trailing closure, using `Capture` where the component is a non-builder-closure may cause type-checking ambiguity.

```swift
Regex {
  Capture(OneOrMore(.digit)) {
    Int($0)
  } // Which output type, `(Substring, Substring)` or `(Substring, Substring?)`?
}
```

Spelling out `TryCapture` also has the benefit of clarity, as it makes clear that a capture's transform closure can cause the regex engine to backtrack. Since backtracking can be expensive, one could choose to throw errors instead and use a normal `Capture`.

```swift
Regex {
  Capture(OneOrMore(.digit)) {
    guard let number = Int($0) else {
      throw MyCustomParsingError.invalidNumber($0)
    }
    return number
  }
}
```

[Declarative String Processing]: https://github.com/apple/swift-experimental-string-processing/blob/main/Documentation/DeclarativeStringProcessing.md
[Strongly Typed Regex Captures]: https://github.com/apple/swift-experimental-string-processing/blob/main/Documentation/Evolution/StronglyTypedCaptures.md
[Regex Syntax]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0355-regex-syntax-run-time-construction.md
[String Processing Algorithms]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0357-regex-string-processing-algorithms.md
