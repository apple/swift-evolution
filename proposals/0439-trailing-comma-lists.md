# Allow trailing comma in comma-separated lists

- Proposal: [SE-0439](0439-trailing-comma-lists.md)
- Author: [Mateus Rodrigues](https://github.com/mateusrodriguesxyz)
- Review Manager: [Xiaodi Wu](https://github.com/xwu)
- Status: **Active review (July 1...July 14, 2024)**
- Implementation: https://github.com/swiftlang/swift/pull/74522# gated behind `-enable-experimental-feature TrailingComma`
- Review: ([pitch](https://forums.swift.org/t/pitch-allow-trailing-comma-in-tuples-arguments-and-if-guard-while-conditions/70170)), ([review](https://forums.swift.org/t/se-0439-allow-trailing-comma-in-comma-separated-lists/72876))

## Introduction

This proposal aims to allow the use of trailing commas, currently restricted to array and dictionary literals, in comma-separated lists whenever there are terminators that enable unambiguous parsing.

## Motivation

### Development Quality of Life Improvement

A trailing comma is an optional comma after the last item in a list of elements:

```swift
let rank = [
  "Player 1",
  "Player 3",
  "Player 2",
]
```

Swift's support for trailing commas in array and dictionary literals makes it as easy to append, remove, reorder, or comment out the last element as any other element.

Other comma-separated lists in the language could also benefit from the flexibility enabled by trailing commas. Consider the function [`split(separator:maxSplits:omittingEmptySubsequences:)`](https://swiftpackageindex.com/apple/swift-algorithms/1.2.0/documentation/algorithms/swift/lazysequenceprotocol/split(separator:maxsplits:omittingemptysubsequences:)-4q4x8) from the [Algorithms](https://github.com/apple/swift-algorithms) package, which has a few parameters with default values.


```swift
let numbers = [1, 2, 0, 3, 4, 0, 0, 5]
    
let subsequences = numbers.split(
	separator: 0,
//        maxSplits: 1
) ❌ Unexpected ',' separator
```

### The Language Evolved

Back in 2016, a similar [proposal](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0084-trailing-commas.md) with a narrower scope was reviewed and rejected for Swift 3. Since that time, the language has evolved substantially that challenges the basis for rejection. The code style that "puts the terminating right parenthesis on a line following the arguments to that call" has been widely adopted by community, Swift standard library codebase, swift-format, docc documentation and Xcode. Therefore, not encouraging or endorsing this code style doesn't hold true anymore.

The language has also seen the introduction of [parameter packs](https://github.com/apple/swift-evolution/blob/main/proposals/0393-parameter-packs.md), which enables APIs that are generic over variable numbers of type parameters, and code generation tools like plugins and macros that, with trailing comma support, wouldn't have to worry about a special condition for the last element when generating comma-separated lists.

## Proposed solution

This proposal adds support for trailing commas in comma-separated lists when there's a clear terminator, which are the following:

- Tuples and tuple patterns.

```swift
(1, 2,)
let block: (Int, Int,) -> Void = { (a, b,) in  }
let (a, b,) = (1, 2,)
for (a, b,) in zip(s1, s2) { }
```

- Parameter and argument lists of initializers, functions, enum associated values, expression macros, attributes, and availability specs.

```swift

func foo(a: Int, b: Int,) { }

foo(a: 1, b: 1,)

struct S {
    init(a: Int, b: Int,) { }
}

enum E {
    case foo(a: Int, b: Int,)
}

@Foo(1, 2, 3,) 
struct S { }

f(_: @foo(1, 2,) Int)

#foo(1, 2,)

struct S {
    #foo(1, 2,)
}

if #unavailable(iOS 15, watchOS 9,) { }

```
- Subscripts, including key path subscripts.

```swift
let value = m[x, y,]

let keyPath = \Foo.bar[x,y,]  

f(\.[x,y,])
```

- `if`, `guard` and `while` condition lists.

```swift
if a, b, { }
while a, b, { }
guard a, b, else { }
```

- `switch` case labels.

```swift
switch number {
    case 1, 2,:
        ...
    default:
        ..
}
```

- Closure capture lists.

```swift
{ [a, b,] in }
```

- Inheritance clauses.

```swift
struct S: P1, P2, P3, { }
```

- Generic parameters.

```swift
struct S<T1, T2, T3,> { }
```

- Generic `where` clauses.

```swift
struct S<T1, T2, T3> where T1: P1, T2: P2, { }
```

- String interpolation

```swift
let s = "\(1, 2,)"
```

## Detailed Design

Trailing commas will be supported in comma-separated lists whenever there is a terminator clear enough that the parser can determine the end of the list. The terminator can be the symbols like `)`, `]`, `>`, `{` and `:`, a keyword like `where` or a pattern code like the body of a `if`, `guard` and `while` statement.

Note that the requirement for a terminator means that the following cases will not support trailing comma:

Enum case label lists:

```swift
enum E {
  case a, b, c, // ❌ Expected identifier after comma in enum 'case' declaration
}
```

Inheritance clauses for associated types in a protocol declaration:

```swift
protocol Foo {
  associatedtype T: P1, P2, // ❌ Expected type
}
```

Generic `where` clauses for initializers and functions in a protocol declaration:

```swift
protocol Foo {
  func f<T1, T2>(a: T1, b: T2) where T1: P1, T2: P2, // ❌ Expected type
}
```

Trailing commas will be allowed in single-element lists but not in zero-element lists, since the trailing comma is actually attached to the last element. Supporting a zero-element list would require supporting _leading_ commas, which isn't what this proposal is about.

```swift
(1,) // OK
(,) // ❌ expected value in tuple
```


## Source compatibility

Although this change won't impact existing valid code it will change how some invalid code is parsed. Consider the following:

```swift
if
  condition1,
  condition2,
{ // ❌ Function produces expected type 'Bool'; did you mean to call it with '()'?
  return true
} 

{ print("something") }
```

Currently the parser uses the last comma to determine that whatever follows is the last condition, so `{ return true }` is a condition and `{ print("something") }` is the `if` body.

With trailing comma support, the parser will terminate the condition list before the first block that is a valid `if` body, so `{ return true }` will be parsed as the `if` body and `{ print("something") }` will be parsed as an unused closure expression.

```swift
if
  condition1,
  condition2,
{
  return true
} 

{ print("something") } // ❌ Closure expression is unused
```

## Alternatives considered

### Eliding commas

A different approach to address similar motivations is to allow the comma between two expressions to be elided when they are separated by a newline.

```swift
print(
    "red"
    "green"
    "blue"
)
```
This was even [proposed](https://forums.swift.org/t/se-0257-eliding-commas-from-multiline-expression-lists/22889/188) and returned for revision back in 2019.

The two approaches are not mutually exclusive. There remain unresolved questions about how the language can accommodate elided commas, and adopting this proposal does not prevent that approach from being considered in the future.
