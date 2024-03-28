# Multi-statement expressions using `then`

* Proposal: [SE-NNNN](NNNN-then-keyword.md)
* Authors: [Ben Cohen](https://github.com/airspeedswift), [Hamish Knight](https://github.com/hamishknight)
* Review Manager: TBD
* Status: **Awaiting Implementation**
* Implementation: available on `main` via `-enable-experimental-feature ThenStatements` and `-enable-experimental-feature DoExpressions`

## Introduction

This proposal introduces a `then` keyword, for the purpose of determining the value of an `if` or `switch` expression that contains multiple statements in a single branch. It also introduces `do` expressions.

## Motivation

[SE-0380](https://github.com/apple/swift-evolution/blob/main/proposals/0380-if-switch-expressions.md) introduced the ability to use `if` and `switch` statements as expressions. As that proposal lays out, this allows for much improved syntax for example when initializing variables:

```
let width = switch scalar.value {
    case 0..<0x80: 1
    case 0x80..<0x0800: 2
    case 0x0800..<0x1_0000: 3
    default: 4
}
```

where otherwise techniques such as an immediately-executed closure, or explicitly-typed definitive initialization would be needed.

However, the proposal left as a future direction the ability to have a branch of the `switch` contain multiple statements:

```swift
let width = switch scalar.value {
    case 0..<0x80: 1
    case 0x80..<0x0800: 2
    case 0x0800..<0x1_0000: 3
    default: 
      log("this is unexpected, investigate this")
      4  // error: Non-expression branch of 'switch' expression may only end with a 'throw'
}
```

When such branches are necessary, currently users must fall back to the old techniques.

This proposal introduces a new contextual keyword, `then`, which allows a `switch` to remain an expression:

```
let width = switch scalar.value {
    case 0..<0x80: 1
    case 0x80..<0x0800: 2
    case 0x0800..<0x1_0000: 3
    default: 
      log("this is unexpected, investigate this")
      then 4
}
```

`then` can similarly be used to allow multi-statement branches in `if` expressions.

The introduction of this keyword also makes stand-alone `do` expressions more viable. These have two use cases: 

1. To produce a value from both the success and failure paths of a `do`/`catch` block:
    ```swift
    let foo: String = do {
        try bar()
    } catch {
        "Error \(error)"
    }
    ```
2. The ability to initialize a variable when this cannot easily be done with a single expression:

    ```swift
    let icon: IconImage = do {
        let image = NSImage(
                        systemSymbolName: "something", 
                        accessibilityDescription: nil)!
        let preferredColor = NSColor(named: "AccentColor")!
        then IconImage(
                image, 
                isSymbol: true, 
                isBackgroundSupressed: true, 
                preferredColor: preferredColor.cgColor)
    }
    ```

While the above can be composed as a single expression, declaring separate variables and then using them is much clearer.

In other cases, this cannot be done because an API is structured to require you first create a value, then mutate part of it:

```swift
let motionManager: CMMotionManager = {
    let manager = CMMotionManager()
    manager.deviceMotionUpdateInterval = 0.05
    return manager
}()
```

This immediately-executed closure pattern is commonly seen in Swift code. So much so that in some cases, users assume that even single expressions must be surrounded in a closure. `do` expressions would provide a clearer idiom for grouping these.

## Detailed Design

A new contextual keyword `then` will be introduced. `if` and `switch` expressions will no longer be limited to a single expression per branch. Instead, they can execute multiple statements, and then end with a `then` expression, which becomes the value of that branch of the expression.

Additionally `do` statements will become expressions, with rules matching those of `if` and `switch` expressions from SE-0380: 

- They can be used to return vales from functions, to assign values to variables, and to declare variables.
- They will not be usable more generally as sub-expressions, arguments to functions etc
- Both the `do` branch, and each `catch` branch if present, must either be a single expression, or yield a value using `then`.
- Further `if`, `switch`, and `do` expressions may be nested inside the `do` or `catch` branches, and `do` expressions can be nested inside `if` and `switch` expressions.
- The `do` and any `catch` branches must all produce the same type, when type checked independently (see SE-0380 for justification of this).
- If a block either explicitly throws, or terminates the program (e.g. with `fatalError`), it does not need to produce a value and can have multiple statements before terminating.

### Nested use of `then`

If needed, a `then` must be the last expression in a branch. Allowing it in other positions, and all paths to be checked as producing a value using Swift's definite initialization feature, would lead to similar complexities to those that caused control flow like `break`,`continue`, and `return`, to be ruled out during SE-380.

A `then` keyword only applies to the innermost `if`, `switch`, or `do` - it cannot apply to an outer expression even if e.g. the inner `if` is not an expression. For example, the following code will not compile:

```swift
let x = if .random() {
  print("hello")
  if .random() {
    then 1 // this `then` is intended to apply to the outer `if`
  } else {
    then 2
  }
} else {
  3
}
```

and should be rewritten as:

```swift
let x = if .random() {
  print("hello")
  then if .random() {
    1
  } else {
    2
  }
} else {
  3
}
```

If the inner branches above also needed a `then`, this could still be used:

```swift
let x = if .random() {
  print("hello")
  then if .random() {
    print("world")
    then 1 // this then applies to the inner if exression
  } else {
    2  // then not needed here, though it would be allowed
  }
} else {
  3
}
```

A `then` cannot be nested inside the `else` of a `guard` even though this might be considered the "last statement":

```
let x = if .random() {
  guard .random() else {
    then 0
  }
  then 1
} else {
  0
}
```

as this implies that `guard` is also an expression (a future direction of SE-380 that could still be explored further) and that you could replace the above `guard` with an `if`, which would not be valid.

### Parsing Ambiguities with `then`

`then` will be introduced as a contextual keyword, with some heuristics to preserve source compatibility in all but rare cases. Similar rules were applied to `await` when it became a new contextual keyword. 

To ensure existing use of `then` as a variable name continues to work, a heuristic will be added to avoid parsing it as a keyword when followed by an infix or suffix operator:

```swift
// without heuristic, this would produce
// error: 'then' may only appear as the last statement in an 'if' or 'switch' expression
then = DispatchTime.now()
```

Prefix operators would be permitted, allowing `then -1` to parse correctly. `then - 1` would parse as an expression with `then` as a variable. This follows similar existing rules around whitespace and disambiguation of operators.

Similarly:
- `then(` is a function call, `then (` is a `then` statement.
- `then[` is a subscript, `then [` is a `then` statement
- `then{` & `then {` are always trailing closures. If you want a `then` statement you have to do `then ({...})`

This does mean that `then /^ x/` would parse `/^` to be an infix operator. This is not a problem with the similar case of `return /^ x/` because `return` is not a contextual keyword (you can't do e.g `func return` or `let return`). `then #/^ x/#` would parse as a regular expression.

`then.foo` is a member access, `then .foo` is a `then` statement, as is:

```swift
then
  .member
```

If member access was still desired, back ticks could be used:

```swift
`then`
  .member
```

This is a potential (albeit unlikely) source break, but the back tick fix can be applied to the 5.9 compiler today to ensure existing code can compile with both the old and new compiler.

With these rules in place, the full source compatibility suite passes with this feature enabled.


## Alternatives Considered

Many of the alternatives considered and future directions in [SE-0380](https://github.com/apple/swift-evolution/blob/main/proposals/0380-if-switch-expressions.md) remain applicable to this proposal.

The choice of the keyword `then` invites bikeshedding. Java uses `yield` – however this is already used for a different purpose in Swift.

Many languages (such Ruby) use a convention that the last expression in a block is the value of the outer expression, without any keyword. For example:

```swift
let width = switch scalar.value {
    case 0..<0x80: 1
    case 0x80..<0x0800: 2
    case 0x0800..<0x1_0000: 3
    default: 
      log("this is unexpected, investigate this")
      4  // would now be allowed, with no `then` keyword.
}
```

 This has the benefit of not requiring the a whole new contextual keyword. It can be argued that the last expression without any indicator to mark the expression value explicitly in multi-statement expressions is subtle and can make code harder to read, as a user must examine branches closely to understand the exact location type of the expression value. On the other hand, this is lessened by the requirement that the `if` expression be used to either assign or return a value, and not found in arbitrary positions.

Note that if bare last expression became the rule for `if` and `do`, it raises the question of whether this also be applied to closure returns also, and perhaps even function returns, which would be a major and pervasive change to Swift (though opinions would likely be split on whether this was an improvement or a regression).

A variant of the bare last expression rule can be found in Rust, where semicolons are required, _except_ for the last expression in an `if` or similar expression. This rule could also be applied to Swift:

```swift
let width = switch scalar.value {
    case 0..<0x80: 1
    case 0x80..<0x0800: 2
    case 0x0800..<0x1_0000: 3
    default: 
      log("this is unexpected, investigate this"); // load-bearing semicolon
      4  // allowed as the preceding statement ends with a semicolon
}
```

This option likely works better in Rust, where semicolons are otherwise required. In Swift, they are only optional for uses such as placing multiple statements on one line, making this solution less appealing.

## Source compatibility

As discussed in detailed design, there are rare edge cases where this new rule may break source, but none have been found in the compatibility test suite. Where they do occur, backticks can be applied, and this fix will back deploy to earlier compiler versions.

## Effect on ABI stability

This proposal has no impact on ABI stability.

