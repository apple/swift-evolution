# Enhancing String Literals Delimiters to Support Raw Text

* Proposal: [SE-0200](0200-raw-string-escaping.md)
* Authors: [John Holdsworth](https://github.com/johnno1962), [Becca Royal-Gordon](https://github.com/beccadax), [Erica Sadun](https://github.com/erica)
* Review Manager: [Doug Gregor](https://github.com/DougGregor)
* Previous Revision: [1](https://github.com/swiftlang/swift-evolution/blob/102b2f2770f0dab29f254a254063847388647a4a/proposals/0200-raw-string-escaping.md)
* Status: **Implemented (Swift 5.0)**
* Implementation: [apple/swift#17668](https://github.com/apple/swift/pull/17668)
* Bugs: [SR-6362](https://bugs.swift.org/browse/SR-6362)
* Review: [Discussion thread](https://forums.swift.org/t/se-0200-enhancing-string-literals-delimiters-to-support-raw-text/15420), [Announcement thread](https://forums.swift.org/t/accepted-se-0200-enhancing-string-literals-delimiters-to-support-raw-text/15822/2)

## Introduction

Like many computer languages, Swift uses an escape character (`\`) to create a special interpretation of subsequent characters within a string literal. Escape character sequences represent a set of predefined, non-printing characters as well as string delimiters (the double quote), the escape character (the backslash itself), and (uniquely in Swift) to allow in-string expression interpolation.

Escape characters provide useful and necessary capabilities but strings containing many escape sequences are difficult to read. Other languages have solved this problem by providing an alternate "raw" string literal syntax which does not process escape sequences. As the name suggests, raw string literals allow you to use "raw" text, incorporating backslashes and double quotes without escaping.

We propose to alter Swift's string literal design to do the same, using a new design which we believe fits Swift's simple and clean syntax. This design supports both single-line and multi-line string literals, and can contain any content whatsoever.

This proposal has been extensively revised based on the Core Team feedback for [SE-0200](https://forums.swift.org/t/returned-for-revision-se-0200-raw-mode-string-literals/11630). It was discussed on the [Swift online forums](https://forums.swift.org/t/pure-bikeshedding-raw-strings-why-yes-again/13866).

### Discussion

Raw strings and their design have been discussed in the following Evolution forum threads:

* [\[Pitch\] Raw mode string literals](https://forums.swift.org/t/pitch-raw-mode-string-literals/7120/5)
* [SE-0200: "Raw" mode string literals (Review thread)](https://forums.swift.org/t/se-0200-raw-mode-string-literals/11048)
* [\[Returned for revision\] SE-0200: "Raw" mode string literals
](https://forums.swift.org/t/returned-for-revision-se-0200-raw-mode-string-literals/11630)
* [\[Pitch v2\]: Raw strings and SE-0200](https://forums.swift.org/t/pitch-v2-raw-strings-and-se-0200/11660)
* [Pure Bikeshedding: Raw Strings (why yes, again!)](https://forums.swift.org/t/pure-bikeshedding-raw-strings-why-yes-again/13866)

## Background

Modern programming languages use two approaches to represent string literals.

* A **conventional string literal** is exactly what you use in Swift today. It allows you to use escape sequences like `\\` and `\"` and `\u{n}` to express backslashes, quotes, and unicode scalars, among other special character sequences.
* A **raw string literal** ignores escape sequences. It allows you to paste raw code. In a raw string literal the sequence `\\\n` represents three backslashes followed by the letter "n", not a backslash followed by a line feed.

This proposal uses the following terms.

* **String literals** represent a sequence of characters in source.
* **String delimiters** establish the boundaries at the start and end of a character sequence. Swift's string delimiter is `"`, the double quote (U+0022).
* **Escape characters** create a special interpretation of one or more subsequent characters within a string literal. Swift's escape character is `\`, the backslash (U+005C).
* **Escape character sequences** (shortened to _escape sequence_) represent special characters. In the current version of Swift, the backslash escape character tells the compiler that a sequence should combine to produce one of these special characters.

## Motivation

Raw strings support non-trivial content which belongs directly in source code -- not in an external file -- but cannot be satisfactorily maintained or read in escaped form.

Hand-escaped strings require time and effort to transform source material to an escaped form. It is difficult to validate the process to ensure the escaped form properly represents the original text. This task is also hard to automate as it may not pick up intended nuances, such as recognizing embedded dialog quotes.

Escaping actively interferes with inspection. Developers should be able to inspect and modify raw strings in-place without removing that text from source code. This is especially important when working with precise content such as code sources and regular expressions.

Backslash escapes are common in other languages and formats, from JSON to LaTeX to Javascript to regular expressions. Embedding these in a string literal currently requires doubling-up escapes, or even quadrupling if the source is pre-escaped. Pre-escaped source should be maintained exactly as presented so it can be used, for example, when contacting web-based services.

Importantly, raw strings are transportable. They allow developers to cut and paste content both from and to the literal string. This allows testing, reconfiguration, and adaption of raw content without the hurdles escaping and unescaping that limit development.

In short, a good raw string feature should let users embed any valid Unicode text snippet in a Swift string literal merely by surrounding it with appropriate delimiters, without altering the content itself.

### Examples

Raw string literals may include characters normally used for escaping (such as the backslash `\` character) and characters normally requiring escaping (such as a double quote `"`). For example, consider the following multiline string. It represents code to be output at some point in the program execution:

```
let separators = """
    public static var newlineSeparators: Set<Character> = [
        // [Zl]: 'Separator, Line'
        "\u{2028}", // LINE SEPARATOR

        // [Zp]: 'Separator, Paragraph'
        "\u{2029}", // PARAGRAPH SEPARATOR
    ]
    """
```

Unescaped backslash literals cause the unicode escape sequences to be evaluated and replaced in-string. This produces the following result:

```
public static var newlineSeparators: Set<Character> = [
    // [Zl]: 'Separator, Line'
    "
", // LINE SEPARATOR

    // [Zp]: 'Separator, Paragraph'
    "
", // PARAGRAPH SEPARATOR
]
```

To preserve the intended text, each backslash must be escaped, for example `\\u{2029}`. This is a relatively minor edit but if the code is being copied in and out of the source to permit testing and modification, then each hand-escaped cycle introduces the potential for error.

Single-line string literals may similarly be peppered with backslashes to preserve their original intent, as in the following examples.

```
// Quoted Text
let quote = "Alice: "How long is forever?" White Rabbit: "Sometimes, just one second.""
let quote = "Alice: \"How long is forever?\" White Rabbit: \"Sometimes, just one second.\""

// and

// Regular Expression
let ucCaseCheck = "enum\s+.+\{.*case\s+[:upper:]"
let ucCaseCheck = "enum\\s+.+\\{.*case\\s+[:upper:]"
```

Escaping hinders readability and interferes with inspection, especially in the latter example, where the content contains secondary escape sequences. Using a raw form ensures the expression can be read and updated as needed in the form that will be passed by the literal string.

### Candidates

A good candidate for using raw strings is non-trivial and is burdened by escaping because it:

* **Is obscured by escaping.** Escaping actively harms code review and validation.
* **Is already escaped.** Escaped material should not be pre-interpreted by the compiler.
* **Requires easy transport between source and code in both directions**, whether for testing or just updating source.

The following example is a poor case for using a raw string:

```
let path = "C:\\AUTOEXEC.BAT"
```

The example is trivial and the escaping is not burdensome. It's unlikely that the string contents will require any further modification or reuse in a raw form.

### Utility

Raw strings are most valuable for the following scenarios.

**Metaprogramming**: Use cases include code-producing-code. This incorporates utility programming and building test cases. Apps may generate color scheme type extensions (in Swift, ObjC, for SpriteKit/SceneKit, literals, etc) or date formatters, perform language-specific escaping, create markup, and more.

Escaping complicates copying and pasting from working code into your source and back. When you're talking about code, and using code, having that code be formatted as an easily updated raw string is especially valuable.

Examples of popular apps that perform these tasks include Kite Compositor and PaintCode. Any utility app that outputs code would benefit in some form.

**Regular expressions**: While we have bigger plans for regular expressions in the future, we think they will be a primary use case for raw strings in the short term, and will continue to have an important place in regex usage in the long term.

Even if we introduce native regular expressions in a future version of Swift, users will still sometimes have to write regular expressions intended for use in other systems. For instance, if you need to send a regex to a server, or embed it in Javascript, or put it in a SQL query, or construct an `NSRegularExpression` and pass it to an existing API which uses that type, you'll still express that regular expression as a string literal, not a native regex. And when you do, raw strings will make that much easier.

A raw string feature would thus help with all regular expressions now and some regular expressions in the future. And if the native regular expression feature involves some form of quoting and escaping, it can follow the by-then-established precedent of this proposal to support "raw regexes".

**Pedagogy**: Not all Swift learning takes place in the playground and not all code described in Swift source files use the Swift programming language.

Code snippets extend beyond playground-only solutions for many applications. Students may be presented with source code, which may be explained in-context within an application or used to populate text edit areas as a starting point for learning.

Removing escaped snippets to external files makes code review harder. Escaping (or re-escaping) code is a tedious process, which is hard to inspect and validate. 

**Data Formats and Domain Specific Languages**: It's useful to incorporate short sections of unescaped or pre-escaped JSON and XML. It may be impractical to use external files and databases for each inclusion. Doing so impacts inspection, maintenance, and updating.

**Windows paths**: Windows uses backslashes to delineate descent through a directory tree: e.g., `C:\Windows\All Users\Application Data`. The more complex the path, the more intrusive the escapes.

## Initial Proposal

"Raw-mode" strings were first discussed during the [SE-0168 Multi-Line String literals](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0168-multi-line-string-literals.md) review and postponed for later consideration. This proposal focuses on raw strings to allow the entry of single and multi-line string literals.

The first iteration of [SE-0200](https://github.com/swiftlang/swift-evolution/blob/102b2f2770f0dab29f254a254063847388647a4a/proposals/0200-raw-string-escaping.md) proposed adopting Python's model, using `r"...raw string..."`. The proposal was returned for revision with the [following feedback](https://forums.swift.org/t/returned-for-revision-se-0200-raw-mode-string-literals/11630):

> During the review discussion, a few issues surfaced with the proposal, including:
>
> The proposed r"..." syntax didn’t fit well with the rest of the language. The most-often-discussed replacement was #raw("..."), but the Core Team felt more discussion (as a pitch) is necessary.
>
> The proposal itself leans heavily on regular expressions as a use case for raw string literals. Several reviewers remarked that the motivation wasn’t strong enough to justify the introduction of new syntax in the language, so a revised proposal will need additional motivating examples in other domains.

To move forward, the new raw string design must provide a suitable Swift-appropriate syntax that works within the language's culture and conventions.

## Prior Art

The following links explore the existing art in other languages. We were inspired by the [Rust raw string RFC discussion](https://github.com/rust-lang/rust/issues/9411) when researching these features.

| Syntax | Language(s) | Possible in Swift? | Swifty? |
| ----- | --------- | ------------- | -------- |
| `'Hello, world!'` | Bourne shell, Perl, PHP, Ruby, Windows PowerShell | Yes | Yes if Rust-style multiplicity allows incorporating `'` into raw strings. May be too narrow a use-case to burn `'`. |
| `q(Hello, world!)` | [Perl](https://en.wikipedia.org/wiki/String_literal) (alternate) | Maybe (depends on delimiter) | No |
| `%q(Hello, world!)` | Ruby (alternate) | No (`%` is a valid prefix operator) | No |
| `@"Hello, world!"` | [C#](https://msdn.microsoft.com/en-us/library/69ze775t.aspx), F# | Yes (but would be awful for Obj-C switchers) | No |
| `R"(Hello, world!)"` | [C++11](https://en.cppreference.com/w/cpp/language/string_literal) | Yes | No |
| `r"Hello, world!"` | [D](https://tour.dlang.org/tour/en/basics/alias-strings), [Python](http://wiki.c2.com/?RawStrings) | Yes | No |
| `r#"Hello, world!"#` | [Rust](https://doc.rust-lang.org/reference/tokens.html#raw-string-literals) | Yes | Would need to drop the opening `r` and maybe change the delimiter from `#`. |
| `"""hello \' world"""` and `raw"Hello, world!"` | [Scala](https://www.scala-lang.org/files/archive/spec/2.13/13-syntax-summary.html) | No/Yes | No |
| ``` `Hello, world!` ``` | [D](https://tour.dlang.org/tour/en/basics/alias-strings), [Go](https://golang.org/ref/spec), \`...\` | No (conflicts with escaped identifiers) | No, needs Rust multiplicity |
| ``` ``...`` ``` | [Java](http://openjdk.java.net/jeps/326), any number of \` | No (conflicts with escaped identifiers) | Yes |


## Design

We determined that Rust's approach to raw string literals is the best starting point, offering the greatest flexibility in the smallest syntactic footprint.

In Rust, raw string literals are written as `r"..."`. To embed double-quotes in a Rust raw string literal, you add one or more pound signs before the opening quote, and put a matching number of pound signs after the closing quote: `r#"..."..."#`, `r##"...#"..."##`, etc. Rust developers assured us that even one pound sign was unusual and more than one almost never needed but it's nice to have the flexibility in the rare cases where you need it.

### Swiftifying Rust's Design

Rust's design distinguishes between conventional and raw string literals. It also includes an asymmetric `r` off its leading edge. We found these distinctions unnecessary and the `r` aesthetically displeasing. Instead, our design powers up a conventional Swift `String` literal and in doing so, allows you to access features normally associated with raw literals.

In this design, there is no separate "raw" syntax; rather, there is a small extension of the conventional string literal syntax. A conventional string literal is either:

* a sequence of characters surrounded by double quotation marks ("), or 
* a string that spans several lines surrounded by three double quotation marks.

These are examples of conventional Swift string literals:

```
"This is a single line Swift string literal"

"""
    This is a multi line
    Swift string literal
    """
```

In this form, the revised string design acts exactly like any other string. You use escape character sequences including string interpolation exactly as you would today. A backslash escape character tells the compiler that a sequence should be interpolated, interpreted as an escaped character, or represent a unicode scalar.

Swift's escape sequences include:

* The special characters `\0` (null character), `\\` (backslash), `\t` (horizontal tab), `\n` (line feed), `\r` (carriage return), `\"` (double quotation mark) and `\'` (single quotation mark) 
* Arbitrary Unicode scalars, written as `\u{n}`, where *n* is a 1–8 digit hexadecimal number with a value equal to a valid Unicode code point
* Interpolated expressions, introduced by `\(` and terminated by `)`

### Expanding Delimiters

Our design adds customizable string delimiters. You may pad a string literal with one or more `#` (pound, Number Sign, U+0023) characters:

```
"This is a Swift string literal"

#"This is also a Swift string literal"#

####"So is this"####
```

The number of pound signs at the start of the string (in these examples, zero, one, and four) must match the number of pound signs at the end of the string. `"This"`, `#"This"#`, and `##"This"##` represent identical string values:

```
static-string-literal -> " quoted-text " |
   """ multiline-quoted-text """ |
   # static-string-literal #
```

Any instance of the delimiter which is not followed by the appropriate number of pound signs is treated as literal string contents, rather than as the end of the string literal. That is, the leading pound signs *change the string's delimiter* from `"` to `"#` (or `"##`', etc.). A plain `"` without pound signs after it is just a double-quote character inside the string.

```
#"She said, "This is dialog!""#
// Equivalent to "She said, \"This is dialog!\""
```

If you do add a backslash, it is interpreted as an extra character. This string literal includes the backslash and both double quote marks inside the string delimiters (`#"` and `"#`):

```
#"A \"quote"."#
```

When you need to include `#"` (pound-quote) or `"#` (quote-pound) in your character sequence, adjust the number of delimiting pound signs. This need should be rare.

### Customized Escape Delimiters

This design uses an *escape delimiter*, that is a sequence of one or more characters to indicate the beginning of an escape character sequence, rather than a single escape character.  Like Swift today, the escape delimiter begins with a backslash (Reverse Solidus, U+005C), but it is now followed by zero or more pound signs
(Number Sign, U+0023). An escape delimiter in a string literal must match the number of pound signs used to delimit either end of the string.

Here is the degenerate case. It is a normal string with no pound signs. The escape delimiter therefore needs no pound signs and a single backslash is sufficient to establish the escape character sequence:

```
"This string has an \(interpolated) item"
```

Strings using custom boundary delimiters mirror their pound sign(s) after the leading backslash, as in these examples which produce identical results to the preceding string literal:

```
#"This string has an \#(interpolated) item"#

####"This string has an \####(interpolated) item"####
```

The escape delimiter customization matches the string. Any backslash that is not followed by the correct number of pound signs is treated as raw text. It is not an escape:

```
#"This is not \(interpolated)"#
```

| String Start Delimiter | Escape Delimiter | String End Delimiter |
| ---------------------- | ---------------- | -------------------- |
| `"` | `\` | `"` |
| `#"` | `\#` | `"#` |
| `##"` | `\##` | `"##` |
| `######"` | `\######` | `"######` |

Inside the string, any backslash that is followed by too few pound signs (like `\#` in a `##""##` string) is not an escape delimiter. It is just that exact string. Any backslash  followed by too many pound signs (like `\##` in a `#""#` string) creates an invalid escape sequence because it is an escape delimiter followed by one or more pound signs.

This escaping rule supports several important features: it provides for raw string support, *and* string interpolation.   We feel this is a huge win, especially for code generation applications. We believe this conceptual leap of elegance simplifies all our previous design workarounds and collapses them into one general solution.

This design retains Rust-inspired custom delimiters, offers all the features of "raw" strings, introduces raw string interpolation, and does this _all_ without adding a new special-purpose string type to Swift.

Yes, this approach requires work:

* You must use pound signs for any raw string.
* You must use a more cumbersome interpolation sequence for raw strings than conventional strings.

Hopefully the tradeoffs are worth it in terms of added expressibility and the resulting design is sufficiently elegant to pass muster.

### Updated Escape Character Sequences Reference

Swift string literals may include the following special character sequences. Swift's escape delimiter begins with a backslash (Reverse Solidus, U+005C), and is followed by zero or more pound signs (Number Sign, U+0023). An escape delimiter in a string literal must match the number of pound signs used to delimit either end of the string.

| Sequence | Escape Characters | Result |
|----------|------------|--------|
| Escape Delimiter + `0` | Digit Zero (U+0030)| Boundary Neutral / Null character U+0000 |
| Escape Delimiter + `\` | Reverse Solidus (U+005C) | Punctuation / Backslash U+005C |
| Escape Delimiter + `t` | Latin Small Letter T (U+0074) | Character Tabulation / Horizontal Tab U+0009 |
| Escape Delimiter + `n` | Latin Small Letter N (U+006E) | Paragraph Separator / Line Feed U+000A |
| Escape Delimiter + `r` | Latin Small Letter R (U+0072) | Paragraph Separator / Carriage Return U+000D |
| Escape Delimiter + `"` | Quotation Mark (U+0022) | Punctuation / Quotation Mark U+0022 |
| Escape Delimiter + `'` | Apostrophe (U+0027) | Punctuation / Apostrophe (Single Quote) U+0027 |
| Escape Delimiter + `u{n}` | Latin Small Letter U (U+0055), Left Curly Bracket (U+007B), Right Curly Bracket (U+007D) | An arbitrary Unicode scalar, where *n* is a 1–8 digit hexadecimal number with a value equal to a valid Unicode code point |
| Escape Delimiter + `(...)` | Left Parenthesis (U+0028), Right Parenthesis (U+0029) | An interpolated expression |


## Examples

Consider the text `\#1`:

```
// What you type today
"\\#1" // escape delimiter + backslash + # + 1

// What you type using the new system
"\\#1" // the same

// What you type with a single pound string delimiter
#"\#\#1"# // escape delimiter + backslash + # + 1

// Or you can adjust the string delimiter for a raw string:
##"\#1"## // backslash + # + 1, no escape sequence
```

Adjusting string delimiters allows you to eliminate escape sequences to present text as intended for use:

```
#"c:\windows\system32"# // vs. "c:\\windows\\system32"
#"\d{3) \d{3} \d{4}"# // vs "\\d{3) \\d{3} \\d{4}"

#"a string with "double quotes" in it"#

##"a string that needs "# in it"##

#"""
    a string with
    """
    in it
    """#
```

The following example terminates with backslash-r-backslash-n, not a carriage return and line feed:

```
#"a raw string containing \r\n"# 
// vs "a raw string containing \\r\\n"
```

The same behavior is extended to multi-line strings:

```
#"""
    a raw string containing \r\n
    """#
```

New line escaping works as per [SE-182](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0182-newline-escape-in-strings.md):

```
#"""
    this backslash and newline will be present in the string \
    this newline will take it's escaped value in this case \#
    and the line continues here with only a space joining. 
    """#
```

Custom-delimited strings allow you to incorporate already-escaped text. For example, you can paste static data without having to worry about re-escaping a JSON message

```
#"""
	[
		{
			"id": "12345",
			"title": "A title that \"contains\" \\\""
		}
	]
	"""#
```

Without custom delimiters, Swift would silently unescape this content, yielding an invalid JSON message. Even if you did remember to escape, this process would be error-prone and difficult to maintain.

However, if you wanted to interpolate the value of the "id" field, you could
still do that without having to double-escape the other backslashes:

```
#"""
	[
		{
			"id": "\#(idNumber)",
			"title": "A title that \"contains\" \\\""
		}
	]
	"""#
```

It is anticipated raw strings will also work in attributes:
```
@available(swift, deprecated: 4.2, message:
  #"Note: "\r\n" (CR-LF) is normalized to "\n" (LF)"#)
```

## Errors

The compiler errors when an escape delimiter is followed by an unrecognized value to complete an escape sequence. For example, using one-`#`-delimited strings:

```
#"printf("%s\n", value_string)"# // no error, no escape sequence
#"printf("%s\#n", value_string)"# // newline escape sequence
#"printf("%s\#x\#n", value_string)"# // error: escape delimiter + x. 
```

The last example can introduce a fixit by adding another `#` to either side of the string so `\#` is no longer the escape delimiter. However, this eliminates the subsequent line feed (a valid escape sequence) that follows unless that, too, is appropriately updated.

There are also wrong ways to add interpolated text. These examples are both errors. The escape delimiter in each case (respectively `\` and `\#`) is followed by `#`, forming an invalid escape sequence:

```
"This is not \#(correct)"

#"This is not \##(correct)"#

```

An escape with too many pounds should be an error with a special message and fix-it. The fixit should suggest that `escape-delimiter + #` instances remove the extra pound or add further `#`-signs to each end of the string.

## Discoverability and Recognition

There are two questions of developer approach: discoverability ("how do I use raws string in Swift") and recognition ("Why do some strings in Swift start with `#`?" and "Why are there `#` signs after backslashes?"). When presented to developers unfamiliar with the new string syntax, we feel that it isn't overly burdensome to search the web for:

* "What do #/pound/number/etc signs mean in Swift strings?"
* "How do I use raw strings in Swift?"
* "How do I add quote marks to strings without escaping in Swift?"
* "How do I interpolate in raw Swift strings?"

## Implementation

Changes are largely confined to the file lib/Parse/Lexer.cpp. They involve a slight modification to the main lexer loop Lexer::lexImpl() to detect strings that have a custom delimiter/are surrounded by 1 or more `#` characters. When the start of a custom-delimited string is detected Lexer::lexStringLiteral() is called with the delimiter’s length. Targeted changes to Lexer::lexCharacter() and Lexer::getEncodedStringSegment() bypass processing of the escape character `\` if the delimiter length is not equal to the number of `#` characters after the `\`, the degenerate case being delimiter length of 0 which is normal string processing.

A new field `StringDelimiterLength` in Token.h carries the string escaping mode from the parsing to code generation phases of compilation.

## Source compatibility

This is a purely additive change. The syntax proposed is not currently valid Swift.

## Effect on ABI stability

None.

## Effect on API resilience

None.

## Alternatives considered

We evaluated many, *many* designs from other languages and worked through a long thread full of bikeshedding. We will list the most notable rejected designs here, but these are just the tip of the iceberg.

### Excluding single quotes and backticks

Single quotes are a common syntax for raw strings in other languages. However, they're also commonly used for character literals (i.e. integer literals containing the value of a Unicode scalar) in other languages. If we use single quotes for raw strings, we cannot use them for character literals or any other future proposal. We see no need to burn single quotes on this feature.

Similarly, while backticks preserve the meaning of "code voice" and "literal", as you are used to in Markdown, they would conflict with escaped identifiers.

### Using "raw" and "rawString"

The original design `r"..."` was rejected in part for not being Swifty, that is, not taking on the look and feel and characteristics of existing parts of the language. Similar approaches like `raw"..."` and `#raw"..."` carry the same issues. Leading text is distracting and competes for attention with the content of the string that follows. And function-like constructs like `#raw("...")` harmonize better with the language syntax, but are even worse for readability.

### Using user-specified delimiter characters

We felt user-specified delimiters overly complicated the design space, were harder to discover and use, and were generally un-Swifty. The pound sign is rarely used and a minor burden on the syntax.

```
@rawString(delimiter: @) @"Hello"@ // no
```

We considered a Perl/Ruby-like approach to arbitrary delimiters, but quickly rejected it. The arbitrary delimiter rules in these languages are complex and have many corner cases; we don't need or want that complexity.

We also rejected a standalone raw string attribute for being wordy and heavy, especially for short literals.
