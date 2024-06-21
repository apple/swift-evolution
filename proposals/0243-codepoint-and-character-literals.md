# Integer-convertible character literals

* Proposal: [SE-0243](0243-codepoint-and-character-literals.md)
* Authors: [Dianna Ma (“Taylor Swift”)](https://github.com/tayloraswift), [Chris Lattner](https://github.com/lattner), [John Holdsworth](https://github.com/johnno1962)
* Review manager: [Ben Cohen](https://github.com/airspeedswift)
* Status: **Rejected** ([Rationale](https://forums.swift.org/t/se-0243-codepoint-and-character-literals/21188/341))
* Implementation: [apple/swift#21873](https://github.com/apple/swift/pull/21873)
* Threads: [1](https://forums.swift.org/t/prepitch-character-integer-literals/10442)

## Introduction

Swift’s `String` type is designed for Unicode correctness and abstracts away the underlying binary representation of the string to model it as a `Collection` of grapheme clusters. This is an appropriate string model for human-readable text, as to a human reader, the atomic unit of a string is (usually) the extended grapheme cluster. When treated this way, many logical string operations “just work” the way users expect. 

However, it is also common in programming to need to express values which are intrinsically numeric, but have textual meaning, when taken as an ASCII value. We propose adding a new literal syntax takes single-quotes (`'`), and is transparently convertible to Swift’s integer types. This syntax, but not the behavior, will extend to all “single element” text literals, up to and including `Character`, and will become the preferred literal syntax these types.

## Motivation 

A pain point of using characters in Swift is they lack a first-class literal syntax. Users have to manually coerce string literals to a `Character` or `Unicode.Scalar` type using `as Character` or `as Unicode.Scalar`, respectively. Having the collection share the same syntax as its element also harms code clarity and makes it difficult to tell if a double-quoted literal is being used as a string or character in some cases.

Additional challenges arise when using ASCII scalars in Swift. Swift currently provides no static mechanism to assert that a unicode scalar literal is restricted to the ASCII range, and lacks a readable literal syntax for such values as well. In C, `'a'` is a `uint8_t` literal, equivalent to `97`. Swift has no such equivalent, requiring awkward spellings like `UInt8(ascii: "a")`, or spelling out the values in hex or decimal directly. This harms readability of code, and makes bytestring processing in Swift painful.

```c
static char const hexcodes[16] = {
    '0', '1', '2', '3', '4' ,'5', '6', '7', '8', '9', 
    'a', 'b', 'c', 'd', 'e', 'f'
};
```

```swift
let hexcodes = [
    UInt8(ascii: "0"), UInt8(ascii: "1"), UInt8(ascii: "2"), UInt8(ascii: "3"),
    UInt8(ascii: "4"), UInt8(ascii: "5"), UInt8(ascii: "6"), UInt8(ascii: "7"),
    UInt8(ascii: "8"), UInt8(ascii: "9"), UInt8(ascii: "a"), UInt8(ascii: "b"),
    UInt8(ascii: "c"), UInt8(ascii: "d"), UInt8(ascii: "e"), UInt8(ascii: "f")
]    
```

Higher-level constructs can regain some readability, 

```swift 
let hexcodes = [
    "0", "1", "2", "3",
    "4", "5", "6", "7",
    "8", "9", "a", "b",
    "c", "d", "e", "f"
].map{ UInt8(ascii: $0) }
```

but may not be familiar to all users, and can come at a runtime cost.

In addition, the `init(ascii:)` initializer only exists on `UInt8`. If you're working with other types like `Int8` (common when dealing with C APIs that take `char`), it is much more awkward.  Consider scanning through a `char*` buffer as an `UnsafeBufferPointer<Int8>`:

```swift
for scalar in int8buffer {
    switch scalar {
    case Int8(UInt8(ascii: "a")) ... Int8(UInt8(ascii: "f")):
        // lowercase hex letter
    case Int8(UInt8(ascii: "A")) ... Int8(UInt8(ascii: "F")):
        // uppercase hex letter
    case Int8(UInt8(ascii: "0")) ... Int8(UInt8(ascii: "9")):
        // hex digit
    default:
        // something else
    }
}
```

Transforming `Unicode.Scalar` literals also sacrifices compile-time guarantees. The statement `let char: UInt8 = 1989` is a compile time error, whereas `let char: UInt8 = .init(ascii: "߅")` is a run time error.

ASCII scalars are inherently textual, so it should be possible to express them with a textual literal directly. Just as applying the `String` APIs runs counter to Swift’s stated design goals of safety and efficiency, requiring users to express basic data values in such a verbose way runs counter to our design goal of [expressiveness](https://swift.org/about/#swiftorg-and-open-source).

Integer character literals would provide benefits to `String` users. One of the [future directions](https://gist.github.com/milseman/bb39ef7f170641ae52c13600a512782f#unmanaged-strings) for `String` is to provide performance-sensitive or low-level users with direct access to code units. Having numeric character literals for use with this API is highly motivating. Furthermore, improving Swift’s bytestring ergonomics is an [important part](https://forums.swift.org/t/prepitch-character-integer-literals/10442/140?u=taylorswift) of our long term goal of expanding into embedded platforms.

## Proposed solution 

The most straightforward solution is to conform Swift’s integer types to `ExpressibleByUnicodeScalarLiteral`. Due to ABI constraints, it is not currently possible to add this conformance, so we will add the conformance *implementations* to the standard library, and allow users to “enable” to the feature by declaring this conformance in user code, for example:

```swift 
extension Int8: ExpressibleByUnicodeScalarLiteral { }
```
Once the Swift ABI supports retroactive conformances, this conformance can be declared in the standard library, making it available by default.

These integer conversions will only be valid for the ASCII range `U+0 ..< U+128`; unicode scalar literals outside of that range will be invalid and will generate compile-time errors similar to the way we currently diagnose overflowing integer literals. This is a conservative approach, as allowing transparent unicode conversion to integer types carries encoding pitfalls users may not anticipate or easily understand.

Because it is currently possible to call literal initializers at run-time, a run-time precondition failure will occur if a non-ASCII value is passed to the integer initializer. (We expect the compiler to elide this precondition check for “normal” invocations.)

```swift 
let u: Unicode.Scalar = '\u{FF}'
let i1: Int = '\u{FF}'                          // compile-time error
let i2: Int = .init(unicodeScalarLiteral: u)    // run-time error
```

| `ExpressibleBy`… | `UnicodeScalarLiteral` | `ExtendedGraphemeClusterLiteral` | `StringLiteral` | 
| --- | --- | --- | --- |
| `UInt8:`, … , `Int:` | yes* (initially opt-in) | no | no |
| `Unicode.Scalar:` | yes | no | no |
| `Character:` | yes (inherited) | yes | no |
| `String:` | yes | yes | yes |
| `StaticString:` | yes | yes | yes |

> Cells marked with an asterisk `*` indicate behavior that is different from the current language behavior.

The ASCII range restriction will only apply to single-quote literals coerced to a `Unicode.Scalar` and (either statically or dynamically) converted to an integer type. Any valid `Unicode.Scalar` can be written as a single-quoted unicode scalar literal, and any valid `Character` can be written as a single-quoted character literal. 

|                    | `'a'` | `'é'` | `'β'` | `'𓀎'` | `'👩‍✈️'` | `"ab"` |
| --- | --- | --- | --- | --- | --- | --- |
| `:String`          | `"a"`    | `"é"`  | `"β"`    | `"𓀎"` | `"👩‍✈️"` | "ab"
| `:Character`       | `'a'`    | `'é'`  | `'β'`    | `'𓀎'` | `'👩‍✈️'`
| `:Unicode.Scalar`  | U+0061   | U+00E9 | U+03B2   | U+1300E
| `:UInt32`          | 97       | 
| `:UInt16`          | 97       | 
| `:UInt8`           | 97       | 
| `:Int8`            | 97       |  

With these changes, the hex code example can be written much more naturally:

```swift
let hexcodes: [UInt8] = [
    '0', '1', '2', '3', '4', '5', '6', '7', '8', '9', 
    'a', 'b', 'c', 'd', 'e', 'f'
]

for scalar in int8buffer {
    switch scalar {
    case 'a' ... 'f':
        // lowercase hex letter
    case 'A' ... 'F':
        // uppercase hex letter
    case '0' ... '9':
        // hex digit
    default:
        // something else
    }
}
```

### Choice of single quotes

We propose to adopt the `'x'` syntax for all textual literal types up to and including `ExtendedGraphemeClusterLiteral`, but not including `StringLiteral`. These literals will be used to express integer types, `Character`, `Unicode.Scalar`, and types like `UTF16.CodeUnit` in the standard library. 

The default inferred literal type for `let x = 'a'` will be `Character`, following the principle of least surprise. This also allows for a natural user-side syntax for differentiating methods overloaded on both `Character` and `String`.

Use of single quotes for character/scalar literals is precedented in other languages, including C, Objective-C, C++, Java, and Rust, although different languages have slightly differing ideas about what a “character” is.  We choose to use the single quote syntax specifically because it reinforces the notion that strings and character values are different: the former is a sequence, the later is an element. Character types also don't support string literal interpolation, which is another reason to move away from double quotes.

### Single quotes in Swift, a historical perspective

In Swift 1.0, single quotes were reserved for some yet-to-be determined syntactical purpose. Since then, pretty much all of the things that might have used single quotes have already found homes in other parts of the Swift syntactical space:

- syntax for [multi-line string literals](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0168-multi-line-string-literals.md) uses triple quotes (`"""`)

- string interpolation syntax uses standard double quote syntax. 

- raw-mode string literals settled into the `#""#` syntax. 

- In current discussions around [regex literals](https://forums.swift.org/t/string-update/7398/6), most people seem to prefer slashes (`/`).

Given that, and the desire for lightweight syntax for single chararcter syntax, and the precedent in other languages for characters, it is natural to use single quotes for this purpose.

### Existing double quote initializers for characters

We propose deprecating the double quote literal form for `Character` and `Unicode.Scalar` types and slowly migrating them out of Swift.

```swift
let c2 = 'f'               // preferred
let c1: Character = "f"   // deprecated
```

## Detailed Design 

The only standard library changes will be to extend `FixedWidthInteger` to have the `init(unicodeScalarLiteral:)` initializer required by `ExpressibleByUnicodeScalarLiteral`. Static ASCII range checking will be done in the type checker, dynamic ASCII range checking will be done by a runtime precondition.

```swift
extension FixedWidthInteger {
    init(unicodeScalarLiteral: Unicode.Scalar)
}
```

The default inferred type for all single-quoted literals will be `Character`. This addresses a language pain point where declaring a `Character` requires type context.

## Source compatibility 

This proposal could be done in a way that is strictly additive, but we feel it is best to deprecate the existing double quote initializers for characters, and the `UInt8.init(ascii:)` initializer.  

Here is a specific sketch of a deprecation policy: 
  
  * Continue accepting these in Swift 5 mode with no change.  
  
  * Introduce the new syntax support into Swift 5.1.
  
  * Swift 5.1 mode would start producing deprecation warnings (with a fixit to change double quotes to single quotes.)
  
  * The Swift 5 to 5.1 migrator would change the syntax (by virtue of applying the deprecation fixits.)
  
  * Swift 6 would not accept the old syntax.

During the transition period, `"a"` will remain a valid unicode scalar literal, but attempting to initialize integer types with double-quoted ASCII literals will produce an error. 

```
let ascii:Int8 = "a" // error
```

However, as this will only be possible in new code, and will produce a deprecation warning from the outset, this should not be a problem.

## Effect on ABI stability 

All changes except deprecating the `UInt8.init(ascii:)` initializer are either additive, or limited to the type checker, parser, or lexer.

Removing `UInt8.init(ascii:)` would break ABI, but this is not necessary to implement the proposal, it’s merely housekeeping.

## Effect on API resilience 

None. 

## Alternatives considered 

### Integer initializers 

Some have proposed extending the `UInt8(ascii:)` initializer to other integer types (`Int8`, `UInt16`, … , `Int`). However, this forgoes compile-time validity checking, and entails a substantial increase in API surface area for questionable gain. 

### Lifting the ASCII range restriction 

Some have proposed allowing any unicode scalar literal whose codepoint index does not overflow the target integer type to be convertible to that integer type. Consensus was that this is an easy source of unicode encoding bugs, and provides little utility to the user. If people change their minds in the future, this restriction can always be lifted in a source and ABI compatible way.

### Single-quoted ASCII strings

Some have proposed allowing integer *array* types to be expressible by *multi-character* ASCII strings such as `'abcd'`. We consider this to be out of scope of this proposal, as well as unsupported by precedent in C and related languages.
