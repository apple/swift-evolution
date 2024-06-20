# Conditionals in Collections

* Proposal: [SE-0330](0330-collection-conditionals.md)
* Authors: [John Holdsworth](https://github.com/johnno1962), [Rintaro Ishizaki](https://github.com/rintaro)
* Review Manager: [Ted Kremenek](https://github.com/tkremenek)
* Status: **Awaiting review**
* Previous revision: [1](https://github.com/apple/swift-evolution/blob/973b08fd550e63cafb039c3de7e9a85ad3e7e7d3/proposals/0330-collection-conditionals.md)
* Implementation: [apple/swift#40316](https://github.com/apple/swift/pull/40316)
* Bugs: [SR-8743](https://bugs.swift.org/browse/SR-8743)

## Introduction

This is a proposal to extend the existing Swift language to allow `#if` conditional inclusion of elements in array and dictionary literals. For example:

```swift
let array = [
	1,
	#if os(Linux)
	2,
	#endif
	3
]

let dictionary = [
	#if DEBUG
	"a": 1,
	#if swift(>=5.0)
	"b": 2,
	#endif
	#endif
	"c": 3
]
```

Swift-evolution thread: [Allow conditional inclusion of elements in array/dictionary literals?](https://forums.swift.org/t/allow-conditional-inclusion-of-elements-in-array-dictionary-literals/16171)

## Motivation

The most notable use case for this is conditional inclusion of tests for the Swift version of XCTest though it is certain to have other applications in practice allowing data to be tailored to production/development environments, architecture or build configuration.

## Proposed solution

The solution proposed is to allow `#if` conditionals using their exiting syntax inside collection literals surrounding sublists of elements. These elements would be either included or not included in the resulting array or dictionary instance dependant on the truth of the `#if`, `#elseif` or `#else` i.e. whether they where "active".

## Detailed design

### Grammar changes

This proposal changes `array-literal-items` and `dictionary-literal-items` to accept `conditional-compilation-block` in it.

```diff
- array-literal-items → array-literal-item ','(opt) | array-literal-item ',' array-literal-items
+ array-literal-items → array-literal-item ','(opt) | array-literal-item ',' array-literal-items | conditional-compilation-block array-literal-items

- dictionary-literal-items → dictionary-literal-item ','(opt) | dictionary-literal-item ',' dictionary-literal-items
+ dictionary-literal-items → dictionary-literal-item ','(opt) | dictionary-literal-item ',' dictionary-literal-items | conditional-compilation-block dictionary-literal-items
```

If a `conditional-compilation-block` is inside `array-literal` or `dictionay-literal`, it can only contain `array-literal-items` or `dictionary-literal-items` respectively.

### Array literal or dictionary literal

Given a collection literal with conditional compilation directives in it, whether it is an array literal or dictionary literal is determined solely by the first occurence of the element in the source text. Regardless of whether it is in the active block or not, if there is a colon after the expression of the first element, the literal is treated as a dictionary literal; otherwise, it is treated as an array literal. Therefore, a literal that has a dictionary element in a block and an array element in another block will be treated as an error.

```swift
let literal = [
#if CONDITION_1
  key: value,
#else
  value, // error: expected ':' and a dictionary literal element
#endif
]
```

### Empty dictionary literal

A dictionary literal can be empty depending on the conditional compilation directives. In the following case, if the `CONDITION` evaluates to `false`, this is considered an empty dictionary literal. 

```swift
let literal = [
#if CONDITION
  "foo": "bar",
#endif
]
```

You cannot, and do not need to, write `:` in a block to emulate an empty dictionary literal  `[:]` when another block already determines this.

```swift
let literal = [
#if CONDITION
  "foo": "bar",
#else
  : // error: expected key expression in dictionary literal
#endif
]
```

### Empty block body

Each block in `#if ... #endif` can be empty.

```swift
let literal = [
#if CONDITION_1
  // Empty.
#else
  key1: value1,
#endif
]
```

Even though the top `#if` body is empty, this is parsed as a valid dictionary literal regardless of the value of `CONDITION_1` because the first element that appears (i.e. `key1: value1,`) contains a colon.

### Type inference

Type infrerence for collection literals is done based on the parsed _active_ alements after evaluating the conditional compilation directives.

```swift
class Animal {}
class Dog: Animal {}
class Cat: Animal {}

let array = [
  Dog(),
#if CONDITION_1
  Cat(),
#endif
]
```

In this example, if `CONDITION_1` evaluates to `true`, this array is inferred `Array<Animal>` because `Animal` is the best common type between `Dog()` and `Cat()`, otherwise it's `Array<Dog>` because `Dog()` is the only active element.

### Muitple conditional compilation directives in a collection literal

A collection literal can contain multiple `#if ... #endif` directives.

```swift
let value = [
#if CONDITION
  value1,
#endif
#if CONDITION
  value2,
#endif
  value3
]
```

Also `#if ... #endif` can be nested

```swift
let value = [
  value1,
#if CONDITION_1
#if CONDITION_2
  value2,
#else
  value3,
  value4,
#endif
  value5,
#endif
]
```

### Trailing commas

A trailing comma for the last element in conditional compilation directive blocks is required, regardless of where the directive appears in the literal.

```swift
let array = [
  value1,
#if CONDITION_1
  value2 // error: expected ',' separator
#endif
]
```

> **Rationale**: Making the trailing comma optional depending on the postion of the directive can be confusing. For example, in the following example, it's not clear whether the trailing comma for `value1` is required or not.
>
> ```swift
> let array = [
> #if CONDITION_1
>   value1 // ',' is required? Maybe depending on 'CONDITION_2'??
> #endif
> #if CONDITION_2
>   value2
> #endif
> ]
> ```
>
> Make it always optional is certainly an option. But the following example looks weird (to me).
>
> ```swift
> let array = [
> #if CONDITION_1
>   value1,
>   value2
> #endif
>   value3,
>   value4
> ]
> ```

## Source compatibility

N/A. This is an purely additive proposal for syntax that is not currently valid in Swift.

## Effect on ABI stability

N/A. This is a compile time alteration of a collections's elements. The resulting collection is a conventional container as it would have been without the conditional though exactly which elements are included can affect the collection's type.

## Effect on API resilience

N/A. This is not an API.

## Alternatives considered

### Lexer based preprocessing

Currently, processing of conditional compilation directives in Swift is based on parse trees. This proposal continues that approach.

There has been discussion of achieving the goals of this proposal as well as unlocking many other possible uses of `#if` by switching Swift to a Lexer-based preprocessing model.

A lexer-based solution would have significant impact on the current syntax tooling (e.g. `swift-format`). Most notably, a lexer-based model would require source tooling to have knowledge of the build arguments to be used to perform syntax analysis. Therefore, this proposal does not consider it, and focuses on expanding the current model to collection literals. It also does not close off the possibility of a lexer-based approach from being proposed at a later time, but that should be done as a separate initiative to what is proposed here.
