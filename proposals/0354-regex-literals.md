# Regex Literals

* Proposal: [SE-0354](0354-regex-literals.md)
* Authors: [Hamish Knight](https://github.com/hamishknight), [Michael Ilseman](https://github.com/milseman), [David Ewing](https://github.com/DaveEwing)
* Review Manager: [Ben Cohen](https://github.com/airspeedswift)
* Status: **Implemented (Swift 5.7)**
* Upcoming Feature Flag: `BareSlashRegexLiterals` (implemented in Swift 5.8)
* Implementation: [apple/swift#42119](https://github.com/apple/swift/pull/42119), [apple/swift#58835](https://github.com/apple/swift/pull/58835)
  * Bare slash syntax `/.../` available with `-enable-bare-slash-regex`
* Review: ([first pitch](https://forums.swift.org/t/pitch-regular-expression-literals/52820))
         ([second pitch](https://forums.swift.org/t/pitch-2-regex-literals/56736))
         ([first review](https://forums.swift.org/t/se-0354-regex-literals/57037))
             ([revision](https://forums.swift.org/t/returned-for-revision-se-0354-regex-literals/57366))
        ([second review](https://forums.swift.org/t/se-0354-second-review-regex-literals/57367))
           ([acceptance](https://forums.swift.org/t/accepted-with-modifications-se-0354-regex-literals/58537))

## Introduction

We propose the introduction of regex literals to Swift source code, providing compile-time checks and typed-capture inference. Regex literals help complete the story told in *[Regex Type and Overview][regex-type]*.

## Motivation

In *[Regex Type and Overview][regex-type]* we introduced the `Regex` type, which is able to dynamically compile a regex pattern:

```swift
let pattern = #"(\w+)\s\s+(\S+)\s\s+((?:(?!\s\s).)*)\s\s+(.*)"#
let regex = try! Regex(pattern)
// regex: Regex<AnyRegexOutput>
```

The ability to compile regex patterns at run time is useful for cases where it is e.g provided as user input, however it is suboptimal when the pattern is statically known for a number of reasons:

- Regex syntax errors aren't detected until run time, and explicit error handling (e.g `try!`) is required to deal with these errors.
- No special source tooling support, such as syntactic highlighting, code completion, and refactoring support, is available.
- Capture types aren't known until run time, and as such a dynamic `AnyRegexOutput` capture type must be used.
- The syntax is overly verbose, especially for e.g an argument to a matching function.

## Proposed solution

A regex literal may be written using `/.../` delimiters:

```swift
// Matches "<identifier> = <hexadecimal value>", extracting the identifier and hex number
let regex = /(?<identifier>[[:alpha:]]\w*) = (?<hex>[0-9A-F]+)/
// regex: Regex<(Substring, identifier: Substring, hex: Substring)>
```

Forward slashes are a regex term of art. The association between forward slashes and regexes dates back to 1969's ed, the first Unix editor, and it was inherited by subsequent interactive text tools like less and vim. The syntax was also adopted by the sed language; from there it passed to Perl, and then to Ruby and Javascript. Forward slash is instantly recognizable as a regex; the only common alternative is an ordinary string literal passed to a library API, which usually has extra overhead, requires more escaping, and defers regex syntax errors to runtime. The proposed Swift regex literals do not have these limitations, so forward slash provides the right behavioral cues to developers. There are over fifty years of precedents for forward slash and very little for anything else. 

Perl and Ruby additionally allow for [user-selected delimiters](https://perldoc.perl.org/perlop#Quote-and-Quote-like-Operators) to avoid having to escape any slashes inside a regex. For that purpose, we propose the extended literal `#/.../#`.

An extended literal, `#/.../#`, avoids the need to escape forward slashes within the regex. It allows an arbitrary number of balanced `#` characters around the literal and escape. When the opening delimiter is followed by a new line, it supports a multi-line literal where whitespace is non-semantic and line-ending comments are ignored.

The compiler will parse the contents of a regex literal using regex syntax outlined in *[Regex Construction][internal-syntax]*, diagnosing any errors at compile time. The capture types and labels are automatically inferred based on the capture groups present in the regex. Regex literals allows editors and source tools to support features such as syntax coloring inside the literal, highlighting sub-structure of the regex, and conversion of the literal to an equivalent result builder DSL (see *[Regex builder DSL][regex-dsl]*).

A regex literal also allows for seamless composition with the Regex DSL, enabling lightweight intermixing of a regex syntax with other elements of the builder:

```swift
// A regex for extracting a currency (dollars or pounds) and amount from input 
// with precisely the form /[$£]\d+\.\d{2}/
let regex = Regex {
  Capture { /[$£]/ }
  TryCapture {
    /\d+/
    "."
    /\d{2}/
  } transform: {
    Amount(twoDecimalPlaces: $0)
  }
}
```

This flexibility allows for terse matching syntax to be used when it's suitable, and more explicit syntax where clarity and strong types are required.

Due to the existing use of `/` in comment syntax and operators, there are some syntactic ambiguities to consider. While there are quite a few cases to consider, we do not feel that the impact of any individual case is sufficient to disqualify the syntax. Some of these ambiguities require a couple of source breaking language changes, and as such the `/.../` syntax requires upgrading to a new language mode in order to use.

## Detailed design

### Typed captures

Regex literals have their capture types statically determined by the capture groups present. This follows a similar inference behavior to [the DSL][regex-dsl], and is explored in more detail in *[Strongly Typed Captures][strongly-typed-captures]*. We are proposing the following inference behavior for regex literals:

- A `Substring` is always present for the entire match.
- If any captures are present, a tuple is formed with the `Substring`, with subsequent elements representing the capture types. Captures are ordered according to [their numbering][capture-numbering].

The type of a capture is `Substring` by default, however it gets wrapped in an optional if it is not guaranteed to have a value on a successful match. This occurs when it is nested within a quantification that may be zero, which includes `?`, `*`, and any range quantifier with a `0` lower bound, e.g `{0,n}`. It also occurs when it appears in a branch of an alternation. For example:

```swift
let regex1 = /([ab])?/
// regex1: Regex<(Substring, Substring?)>

let regex2 = /([ab])|\d+/
// regex2: Regex<(Substring, Substring?)>
```

A zero quantifier or alternation nested within a capture do not produce an optional capture, unless the capture itself is inside a zero quantifier or alternation:

```swift
let regex = /([ab]*)cd/
// regex: Regex<(Substring, Substring)>
```

In this case, if the `*` quantifier is matched zero times, the resulting capture will be an empty string.

The optional wrapping does not become nested, at most one layer of optionality is applied. For example:

```swift
let regex = /(.)*|\d/
// regex: Regex<(Substring, Substring?)>
```

This behavior differs from that of the DSL, which does apply multiple layers of optionality in such cases due to a current limitation of result builders.

### Named captures

One additional feature of typed captures that is currently unique to the literal is the ability to infer labeled tuple elements for named capture groups. For example:

```swift
func matchHexAssignment(_ input: String) -> (String, Int)? {
  let regex = /(?<identifier>[[:alpha:]]\w*) = (?<hex>[0-9A-F]+)/
  // regex: Regex<(Substring, identifier: Substring, hex: Substring)>
  
  guard let match = input.wholeMatch(of: regex), 
        let hex = Int(match.hex, radix: 16) 
  else { return nil }
  
  return (String(match.identifier), hex)
}
```

This allows the captures to be referenced as `match.identifier` and `match.hex`, in addition to numerically (like unnamed capture groups) as `match.1` and `match.2`. This label inference behavior is not available in the DSL, however users are able to [bind captures to named variables instead][dsl-captures].

### Extended delimiters `#/.../#`, `##/.../##`

Backslashes may be used to write forward slashes within the regex literal, e.g `/foo\/bar/`. However, this can be quite syntactically noisy and confusing. To avoid this, a regex literal may be surrounded by an arbitrary number of balanced number signs. This changes the delimiter of the literal, and therefore allows the use of forward slashes without escaping. For example:

```swift
let regex = #/usr/lib/modules/([^/]+)/vmlinuz/#
// regex: Regex<(Substring, Substring)>
```

The number of `#` characters may be further increased to allow the use of e.g `/#` within the literal. This is similar in style to the raw string literal syntax introduced by [SE-0200], however it has a couple of key differences. Backslashes do not become literal characters. Additionally, a multi-line literal, where whitespace and line-ending comments are ignored, is supported when the opening delimiter is followed by a newline.

```swift
let regex = #/
  usr/lib/modules/ # Prefix
  (?<subpath> [^/]+)
  /vmlinuz          # The kernel
/#
// regex: Regex<(Substring, subpath: Substring)>
```

#### Escaping of backslashes

This syntax differs from raw string literals `#"..."#` in that it does not treat backslashes as literal within the regex. A string literal `#"\n"#` represents the literal characters `\n`. However a regex literal `#/\n/#` remains a newline escape sequence.

One of the primary motivations behind this escaping behavior in raw string literals is that it allows the contents to be easily transportable to/from e.g external files where escaping is unnecessary. For string literals, this suggests that backslashes be treated as literal by default. For regex literals however, it instead suggests that backslashes should retain their semantic meaning. This enables interoperability with regexes taken from outside your code without having to adjust escape sequences to match the delimiters used.

With string literals, escaping can be tricky without the use of raw syntax, as backslashes may have semantic meaning to the consumer, rather than the compiler. For example:

```swift
// Matches '\' <word char> <whitespace>* '=' <whitespace>* <digit>+
let regex = try NSRegularExpression(pattern: "\\\\\\w\\s*=\\s*\\d+", options: [])
```

In this case, the intent is not for the compiler to recognize any of these sequences as string literal escapes, it is instead for `NSRegularExpression` to interpret them as regex escape sequences. As such, a raw string may be used to treat the backslashes literally, allowing `NSRegularExpression` to directly process the escapes, e.g `#"\\\w\s*=\s*\d+"#`.

However this is not an issue for regex literals, as the regex parser is the only possible consumer of such escape sequences. Such a regex can be directly spelled as:

```swift
let regex = /\\\w\s*=\s*\d+/
// regex: Regex<Substring>
```

Backslashes still require escaping to be treated as literal, however we don't expect this to be as common of an occurrence as needing to write a regex escape sequence such as `\s`, `\w`, or `\p{...}`, within a regex literal with extended delimiters `#/.../#`.

#### Multi-line literals

Extended regex delimiters additionally support a multi-line literal when the opening delimiter is followed by a new line. For example:

```swift
let regex = #/
  # Match a line of the format e.g "DEBIT  03/03/2022  Totally Legit Shell Corp  $2,000,000.00"
  (?<kind>    \w+)                \s\s+
  (?<date>    \S+)                \s\s+
  (?<account> (?: (?!\s\s) . )+)  \s\s+ # Note that account names may contain spaces.
  (?<amount>  .*)
/#
```

In such a literal, [extended regex syntax][extended-regex-syntax] `(?x)` is enabled. This means that whitespace in the regex becomes non-semantic (including within character classes), and end-of-line comments are supported with `# comment` syntax.

This mode is supported with any (non-zero) number of `#` characters in the delimiter. Similar to multi-line strings introduced by [SE-0168], the closing delimiter must appear on a new line. To avoid parsing confusion, such a literal will not be parsed if a closing delimiter is not present. This avoids inadvertently treating the rest of the file as regex if you only type the opening.

Extended syntax in such a literal may not be disabled with `(?-x)`, however it may be disabled for the contents of a group `(?-x:...)` or quoted sequence `\Q...\E`, as long as they do not span multiple lines. Supporting semantic whitespace over multiple lines would require stripping leading and trailing whitespace while maintaining the verbatim newlines. This could feasibly be supported, however we feel that its behavior could potentially be confusing.

If desired, newlines may be written using `\n`, or by using a backslash to escape the literal newline character:

```swift
let regex = #/
  a\
  b\
  c
/#
// regex = /a\nb\nc/
```

### Ambiguities of `/.../` with comment syntax

Line comment syntax `//` and block comment syntax `/*` will continue to be parsed as comments. An empty regex literal is not a particularly useful thing to express, but can be written as `#//#` if desired. `*` would be an invalid starting character of a regex, and therefore does not pose an issue.

A parsing conflict does however arise when a block comment surrounds a regex literal ending with `*`, for example:

  ```swift
  /*
  let regex = /[0-9]*/
  */
  ```

In this case, the block comment prematurely ends on the second line, rather than extending all the way to the third line as the user would expect. This is already an issue today with `*/` in a string literal, though it is more likely to occur in a regex given the prevalence of the `*` quantifier. This issue can be avoided in many cases by using line comment syntax `//` instead, which it should be noted is the syntax that Xcode uses when commenting out multiple lines.


### Ambiguity of `/.../` with infix operators

There is a minor ambiguity when infix operators are used with regex literals. When used without whitespace, e.g `x+/y/`, the expression will be treated as using an infix operator `+/`. Whitespace is therefore required for regex literal interpretation, e.g `x + /y/`. Alternatively, extended literals may be used, e.g `x+#/y/#`.

### Regex syntax limitations in `/.../`

In order to help avoid a parsing ambiguity, a `/.../` regex literal will not be parsed if it starts or ends with a space or tab character. This restriction may be avoided by using the extended `#/.../#` literal.

#### Rationale

The restriction on the ending character helps avoid breaking source compatibility with prefix and infix `/` operators in certain cases. Such cases are explored in the next section. The restriction on the starting character is due to a parsing ambiguity that arises when a `/.../` regex literal starts a new line. This is particularly problematic for result builders, where we expect it to be frequently used, in particular within a `Regex` builder:

```swift
let digit = Regex {
  TryCapture(OneOrMore(.digit)) { Int($0) }
}
// Matches against <digit>+ (' + ' | ' - ') <digit>+
let regex = Regex {
   digit
   / [+-] /
   digit
}
```

Instead of being parsed as 3 result builder elements, the second of which being a regex literal, this is instead parsed as a single operator chain with the operands `digit`, `[+-]`, and `digit`. This will therefore be diagnosed as semantically invalid.

To avoid this issue, a regex literal may not start with a space or tab character. If a space or tab is needed as the first character, it must be either escaped, e.g:

```swift
let regex = Regex {
   digit
   /\ [+-] /
   digit
}
```

or an extended literal must be used, e.g:

```swift
let regex = Regex {
   digit
   #/ [+-] /#
   digit
}
```

This restriction takes advantage of the fact that infix operators require consistent spacing on either side. This includes both space characters as well as newlines. For example:

```swift
let a = 0 + 1 // Valid
let b = 0+1   // Also valid
let c = 0
+ 1 // Valid operator chain because the newline before '+' is whitespace.

let d = 0 +1 // Not valid, '+' is treated as prefix, which cannot then appear next to '0'.
let e = 0+ 1 // Same but postfix
let f = 0
+1 // Not a valid operator chain, same as 'd', except '+1' is no longer sequenced with '0'.
```

In much the same way as `f`, by requiring the first character of a regex literal not to be space or tab, we ensure it cannot be treated as an operator chain:

```swift
let g = 0
/1 + 2/ // Must be a regex
```

### How `/.../` is parsed

A `/.../` regex literal will be parsed when an opening `/` is encountered in expression position, and there is a closing `/` present. As such, the following will continue to parse as normal:

```swift
// Infix '/' is never in an expression position in valid code (unless unapplied).
let a = 1 / 2 / 3

// None of these '/^/' cases are in expression position.
infix operator /^/
func /^/ (lhs: Int, rhs: Int) -> Int { 0 }
let b = 0 /^/ 1

// Also fine.
prefix operator /
prefix func / (_ x: Int) -> Int { x }
let c = /0 // No closing '/', so not a regex literal. The '//' of this comment doesn't count either.
```

But `let r = /^/` will be parsed as a regex.

A regex literal may be used with a prefix operator, e.g `let r = ^^/x/` is parsed as `let r = ^^(/x/)`. In this case, when encountering operator characters containing `/` in an expression position, the characters up to the first `/` are split into a prefix operator, and regex literal parsing continues as normal.

As already discussed, a regex literal may not start or end with a space or tab. This means that the following will continue to be parsed as normal:

```swift
// Unapplied '/' in a call to 'reduce':
let x = array.reduce(1, /) / 5
let y = array.reduce(1, /) + otherArray.reduce(1, /)

// Prefix '/' with another '/' on the same line:
foo(/a, /b)
bar(/x) / 2

// Unapplied operators:
baz(!/, 1) / 2
qux(/, /)
qux(/^, /)
qux(!/, /)

let d = hasSubscript[/] / 2 // Unapplied infix '/' and infix '/'

let e = !/y / .foo() // Prefix '!/' with infix '/' and operand '.foo()'
```

However this is not sufficient to disambiguate cases such as:

```swift
// Prefix '/' used multiple times on the same line without trailing whitespace:
(/x).foo(/y)
bar(/x) + bar(/y)

// Cases where the closing '/' is not used with whitespace:
bar(/x)/2
baz(!/, 1)/2

// Prefix '/^' with postfix '/':
let f = (/^x)/
```

In all of these cases, the opening `/` appears in expression position, and there is a potential closing `/` that is used without whitespace. To avoid source breakage for such cases, one further heuristic is employed. A regex literal will not be parsed if it contains an unbalanced `)`. This takes both escapes and custom character classes into consideration, and therefore only applies to syntax that would already be invalid for a regex. As such, all of the above cases will continue to be parsed as normal.

This additional heuristic also allows for straightforward disambiguation in source breaking cases where the regex is valid. For example, the following cases will become regex literals:

```swift
foo(/a, b/) // Will become regex literal '/a, b/'
qux(/, !/)  // Will become regex literal '/, !/'
qux(/,/)    // Will become regex literal '/,/'

let g = hasSubscript[/]/2 // Will become regex literal '/]/'

let h = /0; let f = 1/ // Will become the regex literal '/0; let y = 1/'
let i = /^x/           // Will become the regex literal '/^x/'
```

However they can be readily disambiguated by inserting parentheses:

```swift
// Now a prefix and postfix '/':
foo((/a), b/)

// Now unapplied operators:
qux((/), !/)
qux((/),/)
let g = hasSubscript[(/)]/2

let h = (/0); let f = 1/ // Now prefix '/' and postfix '/'
let i = (/^x)/           // Now prefix '/^' and postfix '/'
```

or, in some cases, by inserting whitespace:

```swift
qux(/, /)
let g = hasSubscript[/] / 2
```

We however expect these cases will be fairly uncommon. A similar case is the use of an unapplied infix operator with two `/` characters, for example:

```swift
baz(/^/) // Will become the regex literal '/^/' rather than an unapplied operator
```

This cannot be disambiguated with parentheses or whitespace, however it can be disambiguated using a closure. For example:

```swift
baz({ $0 /^/ $1 }) // Is now infix '/^/'
```

This takes advantage of the fact that a regex literal will not be parsed in an infix operator position.

## Source Compatibility

As explored above, the parsing of `/.../` does have potential to break source in cases where all of the following hold:

- `/` appears in an expression position.
- There is a closing `/` on the same line.
- The first and last character of the literal is not space or tab.
- There are no unbalanced `)` characters within the literal.

However we expect these cases will be uncommon, and can be disambiguated with parentheses or closures if needed.

To accommodate the cases where source may be broken, `/.../` regex literals will be introduced in Swift 6 mode. However, projects may adopt the syntax earlier by passing the compiler flag `-enable-bare-slash-regex` or the [upcoming feature flag](0362-piecemeal-future-features.md) `BareSlashRegexLiterals`. Note this does not affect the extended delimiter syntax `#/.../#`, which will be usable immediately.

## Future Directions

### Modern literal syntax

We could support a more modern Swift-like syntax in regex literals. For example, comments could be done with `//` and `/* ... */`, and quoted sequences could be done with `"..."`. This would however be incompatible with the syntactic superset of regex syntax we intend to parse, and as such may need to be introduced using a new literal kind, with no obvious choice of delimiter.

However, such a syntax would lose out on the familiarity benefits of standard regex, and as such may lead to an "uncanny valley" effect. It's also possible that the ability to use regex literals in the DSL lessens the benefit that this syntax would bring.

### Typed captures for duplicate named group

PCRE allows duplicate capture group names when `(?J)` is set. However this would be incompatible with labeled tuple elements for the captures, as tuples may not have duplicate names. Given we do not currently support `(?J)` in regex literals, the handling of typed captures here is left as future work. 

### Typed captures for branch reset alternations

PCRE and Perl support a branch reset construct `(?|(a)|(b))` where a child alternation resets the capture numbering for each branch, allowing `(a)` and `(b)` to share the same capture number. This would require unifying their types for the purposes of typed captures. Given we do not currently support this construct, the handling of typed captures here is left as future work.

### Library-extensible protocol support

A regex literal describes a string processing algorithm which can be ran over some model of String. The precise semantics of running over extended grapheme clusters vs Unicode scalar values is part of [Unicode for String Processing][regex-unicode]. Libraries may wish to extend this behavior, but the approach presented by various `ExpressibleBy*` protocols is underpowered as libraries would need access to the structure of the algorithm itself.

A better (and future) approach is to open up the regex parser's AST, API, and AST actions to libraries. Here's some examples of why a library might want to customize regex:

A library may wish to provide support for a different or higher level model of string. For example, using localized comparison or tailored grapheme-cluster breaks. Such a use case would need access to the structure of the string processing algorithm literal.

A library may wish to provide support for running over another engine, such as ICU, PCRE, or Javascript. Such a use case would want to pretty-print Swift's regex syntax into one of these syntax variants.

A library may wish to provide their own higher-level structure around which regex literals can be embedded for the purpose of multi-tier processing. For example, processing URLs where regex literal-character portions would be converted into percent-encoded equivalents (with some kind of character class customization/mapping as well). Additionally, a library may have the desire to explicitly delineate patterns that evaluate within a component vs patterns spanning multiple components. Such an approach would benefit from access to the real AST and rich semantic API.

## Alternatives Considered

### Alternative delimiter to `/.../`

Given the fact that `/.../` is an existing term of art for regular expressions, we feel it should be the preferred delimiter syntax. It should be noted that the syntax has become less popular in some communities such as Perl, however we still feel that it is a compelling choice, especially with extended delimiters `#/.../#`. Additionally, while there are some syntactic ambiguities, we do not feel they are sufficient to disqualify the syntax. To evaluate this trade-off, below is a list of alternative delimiters that would not have the same ambiguities, and would not therefore require source breaking changes.

#### Extended literal delimiters only `#/.../#`

We could choose to avoid adding the bare forward slash syntax, and instead require at least one `#` character to be present in the delimiter. This would retain some of the familiarity of `/.../` while avoiding the parsing ambiguities and source breaking changes.

However we feel that `/.../` is the better choice of default syntax, especially for simple regex where the additional noise of the `#` characters would be undesirable. While there are some parsing ambiguities to contend with, we do not feel they outweigh the benefits of having a lightweight and instantly recognizable syntax for regex.

#### Prefixed quote `re'...'`

We could choose to use `re'...'` delimiters, for example:

```swift
// Matches "<identifier> = <hexadecimal value>", extracting the identifier and hex number
let regex = re'([[:alpha:]]\w*) = ([0-9A-F]+)'
```

The use of two letter prefix could potentially be used as a namespace for future literal types. It would also have obvious extensions to extended and multi-line literals using `re#'...'#` and `re'''...'''` respectively. However, it is unusual for a Swift literal to be prefixed in this way. We also feel that its similarity to a string literal might have users confuse it with a raw string literal. 

Also, there are a few items of regex grammar that use the single quote character as a metacharacter. These include named group definitions and references such as `(?'name')`, `(?('name'))`, `\g'name'`, `\k'name'`, as well as callout syntax `(?C'arg')`. The use of a single quote conflicts with the `re'...'` delimiter as it will be considered the end of the literal. However, alternative syntax exists for all of these constructs, e.g `(?<name>)`, `\k<name>`, and `(?C"arg")`. Those could be required instead. An extended regex literal syntax e.g `re#'...'#` would also avoid this issue.

#### Prefixed double quote `re"...."`

This would be a double quoted version of `re'...'`, more similar to string literal syntax. This has the advantage that single quote regex syntax e.g `(?'name')` would continue to work without requiring the use of the alternative syntax or extended literal syntax. However it could be argued that regex literals are distinct from string literals in that they introduce their own specific language to parse. As such, regex literals are more like "program literals" than "data literals", and the use of single quote instead of double quote may be useful in expressing this difference.

#### Single letter prefixed quote `r'...'`

This would be a slightly shorter version of `re'...'`. While it's more concise, it could potentially be confused to mean "raw", especially as Python uses this syntax for raw strings.

#### Single quotes `'...'`

This would be an even more concise version of `re'...'` that drops the prefix entirely. However, given how close it is to string literal syntax, it may not be entirely clear to users that `'...'` denotes a regex as opposed to some different form of string literal (e.g some form of character literal, or a string literal with different escaping rules).

We could help distinguish it from a string literal by requiring e.g `'/.../'`, though it may not be clear that the `/` characters are part of the delimiters rather than part of the literal. Additionally, this would potentially rule out the use of `'...'` as a future literal kind. 

#### Magic literal `#regex(...)`

We could opt for for a more explicitly spelled out literal syntax such as `#regex(...)`. This is a more heavyweight option, similar to `#selector(...)`. As such, it may be considered syntactically noisy as e.g a function argument `str.match(#regex([abc]+))` vs `str.match(/[abc]+/)`.

Such a syntax would require the containing regex to correctly balance parentheses for groups, otherwise the rest of the line might be incorrectly considered a regex. This could place additional cognitive burden on the user, and may lead to an awkward typing experience. For example, if the user is editing a previously written regex, the syntax highlighting for the rest of the line may change, and unhelpful spurious errors may be reported. With a different delimiter, the compiler would be able to detect and better diagnose unbalanced parentheses in the regex.

We could avoid the parenthesis balancing issue by requiring an additional internal delimiter such as `#regex(/.../)`. However this is even more heavyweight, and it may be unclear that `/` is part of the delimiter rather than part of an argument. Alternatively, we could replace the internal delimiter with another character such as ```#regex`...` ```, `#regex{...}`, or `#regex/.../`. However those would be inconsistent with the existing `#literal(...)` syntax and the first two would overload the existing meanings for the ``` `` ``` and `{}` delimiters.

It should also be noted that `#regex(...)` would introduce a syntactic inconsistency where the argument of a `#literal(...)` is no longer necessarily valid Swift syntax, despite being written in the form of an argument.

##### On future extensibility to other foreign language snippets

One of the benefits of `#regex(...)` or `re'...'` is the extensibility to other kinds of foreign langauge snippets, such as SQL. Nothing in this proposal precludes a scalable approach to foreign language snippets using `#lang(...)` or `lang'...'`. If or when that happens, regex could participate as well, but the proposed syntax would still be valuable as regex literals *are* unique in their prevalence as fragments passed directly to API, as well as components of a result builder DSL.


#### Shortened magic literal `#(...)`

We could reduce the visual weight of `#regex(...)` by only requiring `#(...)`. However it would still retain the same issues, such as still looking potentially visually noisy as an argument, and having suboptimal behavior for parenthesis balancing. It is also not clear why regex literals would deserve such privileged syntax.

#### Double slash `// ... //`

Rather than using single forward slash delimiters `/.../`, we could use double slash delimiters. This would have previously been comment syntax, and would therefore be potentially source breaking. In particular, file header comments frequently use this style. Even if they successfully parse as a regex, they would receive different syntax highlighting, and emit a spurious error about being unused.

This would also significantly impact a variety of commonly occurring comments, some examples from the Swift repository include:

```swift
// rdar://41219750

// Please submit a bug report (https://swift.org/contributing/#reporting-bugs)

//   let pt = CGPoint(x: 1.0, y: 2.0) // Here we query for CGFloat.
```

This syntax also means the editor would not be able to automatically complete the closing delimiter, as it would initially appear to be a regular comment. This further means that typing the literal would receive comment syntax highlighting until the closing delimiter is written.

#### Reusing string literal syntax

Instead of supporting a first-class literal kind for regex, we could instead allow users to write a regex in a string literal, and parse, diagnose, and generate the appropriate code when it's coerced to the `Regex` type.

```swift
let regex: Regex = #"([[:alpha:]]\w*) = ([0-9A-F]+)"#
```

However we decided against this because:

- We would not be able to easily apply custom syntax highlighting and other editor features for the regex syntax.
- It would require a `Regex` contextual type to be treated as a regex, otherwise it would be defaulted to `String`, which may be undesired.
- In an overloaded context it may be ambiguous or unclear whether a string literal is meant to be interpreted as a literal string or regex.
- Regex-specific escape sequences such as `\w` would likely require the use of raw string syntax `#"..."#`, as they are otherwise invalid in a string literal.
- It wouldn't be compatible with other string literal features such as interpolations.

### No custom literal

Instead of adding a custom regex literal, we could require users to explicitly write `try! Regex("[abc]+")`. This would be similar to `NSRegularExpression`, and loses all the benefits of parsing the literal at compile time. This would mean:

- No source tooling support (e.g syntax highlighting, refactoring actions) would be available.
- Parse errors would be diagnosed at run time rather than at compile time.
- We would lose the type safety of typed captures.
- More verbose syntax is required.

We therefore feel this would be a much less compelling feature without first class literal support.

### Non-semantic whitespace by default for single-line literals

We could choose to enable non-semantic whitespace by default for single-line literals, matching the behavior of multi-line literals. While this is quite compelling for better readability, we feel that it would lose out on the familiarity and compatibility of the single-line literal.

Non-semantic whitespace can always be enabled explicitly with `(?x)`:

```swift
let r = /(?x) abc | def/
```

or by writing a multi-line literal:

```swift
let r = #/
  abc | def
/#
```

### Multi-line literal with semantic whitespace by default

We could choose semantic whitespace by default within a multi-line regex literal. Such a literal would require a whitespace stripping rule, while keeping newlines of the contents verbatim. To enable non-semantic whitespace in such a literal, you would either have to explicitly write `(?x)` at the very start of the literal:

```swift
let regex = #/
(?x) abc | def
/#
```

Or we could support an explicit specifier as part of the delimiter syntax. For example:

```swift
let regex = #/x
  abc | def
/#
```

However, we don't find either of these options particularly compelling. The former is somewhat verbose considering we expect it to be a common mode for multi-line literals, and it would change meaning if indented at all. The latter wouldn't extend to other matching options, and wouldn't be usable within a single-line literal.

We ultimately feel that non-semantic whitespace is a much more useful default for a multi-line regex literal, and unlike the single-line case, does not lose out on compatibility or familiarity. We could still enforce the specification of `(?x)` or `x`, however they would retain the same drawbacks. We are therefore not convinced they would be beneficial, and feel that the literal being split over multiple lines provides enough signal to indicate different semantics.

#### Supporting the full matching option syntax as part of the delimiter

Rather than supporting a specifier such as `x` on the delimiter, we could support the full range of matching option syntax on the delimiter. For example:

```swift
let regex = #/(?xi)
  abc | def
/#
```

However this would be more verbose, and would add additional complexity to the lexing logic which needs to be able to distinguish between an unterminated single-line literal, and a multi-line literal. It would also be limited to the isolated syntax, and e.g wouldn't support `(?xi:...)`. As we expect non-semantic whitespace to be the frequently desired mode in such a literal, we are not convinced the extra complexity or verbosity is beneficial. 

### Allow matching option flags on the literal `/.../x`

We could choose to support Perl-style specification of matching options on the literal. This could feasibly be supported without introducing source compatibility issues, as identifiers cannot normally be sequenced with a regex literal. However it is unusual for a Swift literal to be suffixed like that. For matching options that affect runtime matching, e.g `i`, we intend on exposing API such as `/.../.ignoresCase()`. The only remaining options that affect parsing instead of matching are `x`, `xx`, `n`, and `J`. These cannot be exposed as API, however the multi-line literal already provides a way to enter extended syntax mode, and we feel writing `(?n)` or `(?J)` at the start of the literal is a suitable alternative to `/.../n` and `/.../J`.

For the multi-line literal, we could require the specification of the `x` flag to enable extended syntax mode. However this would still require the `#/` delimiter. As such, it would lose out on the familiarity of the `/.../x` syntax, and wouldn't provide much visual signal for non-trivial literals. For example:

```swift
let regex = #/
  # Match a line of the format e.g "DEBIT  03/03/2022  Totally Legit Shell Corp  $2,000,000.00"
  (?<kind>    \w+)                \s\s+
  (?<date>    \S+)                \s\s+
  (?<account> (?: (?!\s\s) . )+)  \s\s+ # Note that account names may contain spaces.
  (?<amount>  .*)
/x#
```

### Using `///` or `#///` for multi-line

Instead of re-using the extended delimiter syntax `#/.../#` for multi-line regex literals, we could choose a delimiter that more closely parallels the multi-line string delimiter `"""`. `///` would be the obvious choice, but unfortunately already signifies a documentation comment. As such, it would likely not be viable without further heuristics and regex syntax limitations. A possible alternative is to require at least one `#` character, e.g `#///`. This would be more syntactically viable at the cost of being more verbose. However it may seem odd that a `///` delimiter does not exist for such a literal.

In either case, we are not convinced that drawing a parallel to multi-line string literals is particularly desirable, as multi-line regex literals have considerably different semantics. For example, whitespace is non-semantic and backslashes treat newlines as literal, rather than eliding them:

```swift
let str = """
  a\
  b\
  c
"""
// str = "  a  b  c"

let re = #/
  a\
  b\
  c
/#
// re = /a\nb\nc/
```

For multi-line string literals, the two main reasons for choosing `"""` over `"` were:

1. Editing: It was felt that typing `"` and temporarily messing up the source highlighting of the rest of the file was a bad experience.
2. Visual weight: It was felt that a single `"` written after potentially paragraphs of text would be difficult to notice.

However we do not feel that these are serious issues for regex literals. The `#/` delimiter has plenty of visual weight, and we require a closing `/#` before the literal is treated as multi-line. While it may be possible for the closing `/#` of an existing multi-line regex literal to be treated as a closing delimiter when typing `#/` above, we feel such cases will be quite a bit less common than the string literal `"` case.

### No multi-line literal

We could choose to only support single-line regex literals, with more complex multi-line cases requiring the DSL. However we feel that the ability to write non-semantic whitespace multi-line regex literals is quite a compelling feature that is not covered by the DSL. We feel that confining the literal's ability to work with non-semantic whitespace to the single-line case would lose a lot of the benefits of the extended syntax.

### Restrict feature set to that of the builder DSL

The regex builder DSL is unable to provide some of the features presented such as named captures as tuble labels. An alternative could be to cut those features from the literal out of concern they may lead to an over-use of the literals. However, to do so would remove the clearest demonstration of the need for better type-level operations including working with labeled tuples.

Similarly, there is no literal equivalent for some of the regex builder features, but that isn't an argument against them. The regex builder DSL has references which serves this role (though not as concisely) and they are useful beyond just naming captures.

Regex literals should not be outright avoided, they should be used well. Artifically hampering their usage doesn't provide any benefit and we wouldn't want to lock these limitations into Swift's ABI.



[SE-0168]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0168-multi-line-string-literals.md
[SE-0200]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0200-raw-string-escaping.md

[pitch-status]: https://github.com/apple/swift-experimental-string-processing/blob/main/Documentation/Evolution/ProposalOverview.md
[regex-type]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0350-regex-type-overview.md
[strongly-typed-captures]: https://github.com/apple/swift-experimental-string-processing/blob/main/Documentation/Evolution/StronglyTypedCaptures.md
[regex-unicode]: https://github.com/apple/swift-experimental-string-processing/blob/main/Documentation/Evolution/ProposalOverview.md#unicode-for-string-processing

[internal-syntax]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0355-regex-syntax-run-time-construction.md
[extended-regex-syntax]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0355-regex-syntax-run-time-construction.md#extended-syntax-modes

[capture-numbering]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0355-regex-syntax-run-time-construction.md#group-numbering

[regex-dsl]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0351-regex-builder.md
[dsl-captures]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0351-regex-builder.md#capture-and-reference
