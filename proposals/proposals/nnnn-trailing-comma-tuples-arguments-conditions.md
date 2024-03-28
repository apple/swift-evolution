# Allow trailing comma in tuples, arguments and if/guard/while conditions

- Proposal: SE-NNNN
- Author: Mateus Rodrigues
- Review Manager: TBD
- Status: Awaiting Review
- Implementation: [swift](https://github.com/mateusrodriguesxyz/swift/commit/114b3b4ba1440d6f681b7b5f1086633b504eb2fb), [swift-syntax](https://github.com/mateusrodriguesxyz/swift-syntax/commit/17d3123f1868f450c878cb017403c4c686e8fc86)
- Review: [pitch](https://forums.swift.org/t/pitch-allow-trailing-comma-in-tuples-arguments-and-if-guard-while-conditions/70170/48)

## Introduction

This proposal aims to allow the use of trailing commas, currently restricted to array and dictionary literals, in tuples, arguments and if/guard/while conditions.

## Motivation

### Development Quality of Life Improvement

A trailing comma is a optional comma after the last item in a series of elements:

```swift
let rank = [
  "Player 1",
  "Player 3",
  "Player 2",
]
```

Using trailing commas makes it easy to add, remove, reorder and comment in/out elements, without the need to add or delete the comma while doing any of these manipulations.

Consider the following SwiftUI modifier:

```swift
func frame(
    width: CGFloat? = nil,
    height: CGFloat? = nil,
    alignment: Alignment = .center
) -> some View
```

`frame(width:)`, `frame(height:)`, `frame(width:alignment:)`, `frame(height:alignment:)`, `frame(width:height:)`, `frame(width:height:alignment:)` are all valid calls but you can't easily swipe between `frame(width:)` and `frame(width:alignment:)` by commenting in/out `alignment` without add/remove trailing comma.

```swift
.frame(
    width: 500,
//    alignment: .leading
) ❌ Unexpected ',' separator
```

The introduction of [parameter packs](https://github.com/apple/swift-evolution/blob/main/proposals/0393-parameter-packs.md) allows more APIs that are comma-separated lists at call site and would benefit from trailing comma.

```swift
extension [S] {
    func sorted<each T: Comparable>(_ keyPath: repeat KeyPath<S, each T>) { }
}

arrayOfS.sorted(
  \.a, 
  \.b,
//  \.c
) ❌ Unexpected ',' separator
```

Since [#21381](https://github.com/apple/swift/pull/21381) has been merged back in 2019 **enum associated values** supports default values and are a good fit for trailing comma as well.


**Tuples** use are very close to arguments list and, although may not be so frequently used, it seems natural that they adopt trailing comma too.

**Multiple conditions** in `if`, `guard` and `while` are also comma-separated list and add, remove, reorder and comment in/out are not uncommon practice during development.

```swift
if 
   condition1,
   condition2,
//   condition3
{ ❌ Cannot convert value of type '() -> ()' to expected condition type 'Bool'
                    
} ❌ Expected '{' after 'if' condition
```
### Code Generation

**Plugins** and **Macros** have made it possible to generate code using swift and trailing comma would allow generate list of arguments and conditions without worrying about a special condition for the last element.

### Code Diff

A tangential motivation is that trailing comma makes version-control diffs cleaner.

Without trailing comma:
```diff
foo(
-  a: Int
+  a: Int,
+  b: Int
)
```
With trailing comma:
```diff
foo(
  a: Int,
+  b: Int,
)
```

> [!NOTE]  
> A similar proposal was [rejected](https://forums.swift.org/t/rejected-se-0084-allow-trailing-commas-in-parameter-lists-and-tuples/2777) back in 2016 for Swift 3. It's been 8 years since that, the swift language has evolved a lot, some changes highlighted above as motivation, and the code style that "puts the terminating right parenthesis on a line following the arguments to that call" has been widely adopted by community, swift standard library codebase, swift-format, docc documentation and Xcode. Therefore, not encourage or endorse this code style doesn't hold true anymore nor is a reason for rejection.

## Proposed solution

This proposal adds support for trailing comma to:

### Arguments

Including declaration and call of initializers, functions and enum case associated values.

```swift

func foo(
    a: Int = 0, 
    b: Int = 0, 
) {
}

foo(
    a: 1,
    b: 2,
)

```

### Tuples

```swift
(1, 2, 3,)
```

### Conditions

Including `if`, `guard` and `while`.

```swift
if 
   condition1,
   condition2,
   condition3,
{
                    
}
```

## Source compatibility

This change is purely additive and has no impact on existing code.

## Future directions

### Allow trailing comma anywhere there's a comma-separated list

Although this proposal focuses on the most requested use cases for trailing comma, there's other places with comma-separated list and the restriction could be consistently lifted for all of these.

#### Subclass and Protocol Conformance

```swift
class C2: C1, P1, P2, { } 
```

#### Generics

```swift
struct S<T1, T2,> where T1: P2, T2: P2, { }
```
#### Switch Case

```swift
switch number {
    case 1, 2, 3,:
        ...
    default:
        ...
}
```

## Alternatives considered

### Eliding commas

A different approach to address the exact same motivation is to allow the comma between two expressions to be elided when they are separated by a newline.

```swift
print(
    "red"
    "green"
    "blue"
)
```
This was even [proposed](https://forums.swift.org/t/se-0257-eliding-commas-from-multiline-expression-lists/22889/188) and returned to revision back in 2019.

Even though both approach are not mutually exclusive, this proposal is about consistently extend an existing behavior in the language while eliding comma is a more serious change to the language.
