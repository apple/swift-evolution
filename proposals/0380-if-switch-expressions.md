# `if` and `switch` expressions

* Proposal: [SE-0380](0380-if-switch-expressions.md)
* Authors: [Ben Cohen](https://github.com/airspeedswift), [Hamish Knight](https://github.com/hamishknight)
* Review Manager: [Holly Borla](https://github.com/hborla)
* Status: **Implemented (Swift 5.9)**
* Implementation: [apple/swift#612178](https://github.com/apple/swift/pull/62178), including a downloadable toolchain.
* Review: ([pitch](https://forums.swift.org/t/pitch-if-and-switch-expressions/61149)), ([review](https://forums.swift.org/t/se-0380-if-and-switch-expressions/61899)), ([acceptance](https://forums.swift.org/t/accepted-with-modifications-se-0380-if-and-switch-expressions/62695))

## Introduction

This proposal introduces the ability to use `if` and `switch` statements as expressions, for the purpose of:
- Returning values from functions, properties, and closures;
- Assigning values to variables; and
- Declaring variables.

## Motivation

Swift has always had a terse but readable syntax for closures, which allows the `return` to be omitted when the body is a single expression. [SE-0255: Implicit Returns from Single-Expression Functions](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0255-omit-return.md) extended this to functions and properties with a single expression as their body.

This omission of the `return` keyword is in keeping with Swift's low-ceremony approach, and is in common with many of Swift's peer "modern" languages. However, Swift differs from its peers in the lack of support for `if` and `switch` expressions.

In some cases, this causes ceremony to make a return (ahem), for example:

```swift
public static func width(_ x: Unicode.Scalar) -> Int {
  switch x.value {
    case 0..<0x80: return 1
    case 0x80..<0x0800: return 2
    case 0x0800..<0x1_0000: return 3
    default: return 4
  }
}
```

In other cases, a user might be tempted to lean heavily on the harder-to-read ternary syntax:

```swift
let bullet = isRoot && (count == 0 || !willExpand) ? ""
    : count == 0    ? "- "
    : maxDepth <= 0 ? "▹ " : "▿ "
```

Opinions vary on this kind of code, from enthusiasm to horror, but it's accepted that it's reasonable to find this syntax _too_ terse.

Another option is to use Swift's definite initialization feature:

```swift
let bullet: String
if isRoot && (count == 0 || !willExpand) { bullet = "" }
else if count == 0 { bullet = "- " }
else if maxDepth <= 0 { bullet = "▹ " }
else { bullet = "▿ " }
```

Not only does this add the ceremony of explicit typing and assignment on each branch (perhaps tempting overly terse variable names), it is only practical when the type is easily known. It cannot be used with opaque types, and is very inconvenient and ugly if the type is a complex nested generic.

Programmers less familiar with Swift might not know this technique, so they may be tempted to take the approach of `var bullet = ""`.  This is more bug prone where the default value may not be desired in _any_ circumstances, but definitive initialization won't ensure that it's overridden.

Finally, a closure can be used to simulate an `if` expression:

```swift
let bullet = {
    if isRoot && (count == 0 || !willExpand) { return "" }
    else if count == 0 { return "- " }
    else if maxDepth <= 0 { return "▹ " }
    else { return "▿ " }
}()
```

This also requires `return`s, plus some closure ceremony. But here the `return`s are more than ceremony – they require extra cognitive load to understand they are returning from a closure, not the outer function.

This proposal introduces a new syntax that avoids all of these problems:

```swift
let bullet =
    if isRoot && (count == 0 || !willExpand) { "" }
    else if count == 0 { "- " }
    else if maxDepth <= 0 { "▹ " }
    else { "▿ " }
```

Similarly, the `return` ceremony could be dropped from the earlier example:

```swift
public static func width(_ x: Unicode.Scalar) -> Int {
  switch x.value {
    case 0..<0x80: 1
    case 0x80..<0x0800: 2
    case 0x0800..<0x1_0000: 3
    default: 4
  }
}
```

Both these examples come from posts by [Nate Cook](https://forums.swift.org/t/if-else-expressions/22366/48) and [Michael Ilseman](https://forums.swift.org/t/omitting-returns-in-string-case-study-of-se-0255/24283), documenting many more examples where the standard library code would be much improved by this feature.


## Detailed Design

`if` and `switch` statements will be usable as expressions, for the purpose of:

- Returning values from functions, properties, and closures (either with implicit or explicit `return`);
- Assigning values to variables; and
- Declaring variables.

There are of course many other places where an expression can appear, including as a sub-expression, or as an argument to a function. This is not being proposed at this time, and is discussed in the future directions section.

For an `if` or `switch` to be used as an expression, it would need to meet these criteria:

**Each branch of the `if`, or each `case` of the `switch`, must be a single expression.**

Each of these expressions becomes the value of the overall expression if the branch is chosen.

This does have the downside of requiring fallback to the existing techniques when, for example, a single expression has a log line above it. This is in keeping with the current behavior of `return` omission.

An exception to this rule is if a branch either explicitly throws, or terminates the program (e.g. with `fatalError`), in which case no value for the overall expression needs to be produced. In these cases, multiple expressions could be executed on that branch prior to that point.

In the case where a branch throws, either because a call in the expression throws (which requires a `try`) or with an explicit `throw`, there is no requirement to mark the overall expression with an additional `try` (e.g. before the `if`).

Within a branch, further `if` or `switch` expressions may be nested.

**Each of those expressions, when type checked independently, must produce the same type.**

This has two benefits: it dramatically simplifies the compiler's work in type checking the expression, and it makes it easier to reason about both individual branches and the overall expression.

It has the effect of requiring more type context in ambiguous cases. The following code would _not_ compile:

```swift
let x = if p { 0 } else { 1.0 }
```

since when type checked individually, `0` is of type `Int`, and `1.0` is of type `Double`. The fix would be to disambiguate each branch. In this case, either by rewriting `0` as `0.0`, or by providing type context e.g. `0 as Double`.

This can be resolved by providing type context to each of the branches:

```swift
let y: Float = switch x.value {
    case 0..<0x80: 1
    case 0x80..<0x0800: 2.0
    case 0x0800..<0x1_0000: 3.0
    default: 4.5
}
```

This decision is in keeping with other recent proposals such as [SE-0244: Opaque Result Types](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0244-opaque-result-types.md):

```swift
// Error: Function declares an opaque return type 'some Numeric', but the
// return statements in its body do not have matching underlying types
func f() -> some Numeric {
    if Bool.random() {
        return 0
    } else  {
        return 1.0
    }
}
```

This rule will require explicit type context for declarations in order to determine the type of `nil` literals:

```swift
// invalid:
let x = if p { nil } else { 2.0 }
// valid with required type context:
let x: Double? = if p { nil } else { 2.0 }
```

Of course, when returning from a function or assigning to an existing variable, this type context is always provided.

It is also in keeping with [SE-0326: Enable multi-statement closure parameter/result type inference]( https://github.com/swiftlang/swift-evolution/blob/main/proposals/0326-extending-multi-statement-closure-inference.md):

```swift
func test<T>(_: (Int?) -> T) {}

// invalid
test { x in
  guard let x { return nil }
  return x
}

// valid with required type context:
test { x -> Int? in
  guard let x { return nil }
  return x
}
```

It differs from the behavior of the ternary operator (`let x = p ? 0 : 1.0` compiles, with `x: Double`).

However, the impact of bidirectional inference on the performance of the type checker would likely prohibit this feature from being implemented today, even if it were considered preferable. This is especially true in cases where there are many branches. This decision could be revisited in future: switching to full bidirectional type inference may be source breaking in theory, but probably not in practice (the proposal authors can't think of any examples where it would be).

Bidirectional inference also makes it very difficult to reason about each of the branches individually, leading to sometimes unexpected results:

```swift
let x = if p {
  [1]
} else {
  [1].lazy.map(expensiveOperation)
}
```

With full bidirectional inference, the `Array` in the `if` branch would force the `.lazy.map` in the `else` branch to be unexpectedly eager.

The one exception to this rule is that some branches could produce a `Never` type. This would be allowed, so long as all non-`Never` branches are of the same type:

```swift
// x is of type Int, discounting the type of the second branch
let x = if .random() {
  1
} else {
  fatalError()
}
```

**In the case of `if` statements, the branches must include an `else`**

This rule is consistent with the current rules for definitive initialization and return statements with `if` e.g.

```swift
func f() -> String {
    let b = Bool.random()
    if b == true {
        return "true"
    } else if b == false { // } else { here would compile
        return "false"
    }
} // Missing return in global function expected to return 'String'
```

This could be revisited in the future across the board (to DI, return values, and `if` expressions) if logic similar to that of exhaustive switches were applied, but this would be a separate proposal.

**The expression is not part of a result builder expression**

`if` and `switch` statements are already expressions when used in the context of a result builder, via the `buildEither` function. This proposal does not change this feature.

The variable declaration form of an `if` will be allowed in result builders.

**Pattern matching bindings may occur within an `if` or `case`**

For example, returns could be dropped from

```swift
    private func balance() -> Tree {
        switch self {
        case let .node(.B, .node(.R, .node(.R, a, x, b), y, c), z, d):
            .node(.R, .node(.B,a,x,b),y,.node(.B,c,z,d))
        case let .node(.B, .node(.R, a, x, .node(.R, b, y, c)), z, d):
            .node(.R, .node(.B,a,x,b),y,.node(.B,c,z,d))
        case let .node(.B, a, x, .node(.R, .node(.R, b, y, c), z, d)):
            .node(.R, .node(.B,a,x,b),y,.node(.B,c,z,d))
        case let .node(.B, a, x, .node(.R, b, y, .node(.R, c, z, d))):
            .node(.R, .node(.B,a,x,b),y,.node(.B,c,z,d))
        default:
            self
        }
    }
```

and optional unwrapping could be used with `if let`:

```swift
// equivalent to let x = foo.map(process) ?? someDefaultValue
let x = if let foo { process(foo) } else { someDefaultValue }
```

## Future Directions

This proposal chooses a narrow path of only enabling expressions in the 3 cases laid out at the start. This is intended to cover the vast majority of use cases, but could be followed up by expanded functionality covering many other use cases. Further cases could be added in later proposals once the community has had a chance to use this feature in practice – including source breaking versions introduced under a language variant.

### Full Expressions

A feel for the kind of expressions this could produce can be found in [this commit](https://github.com/apple/swift/compare/main...hamishknight:express-yourself#diff-7db38bc4b6f7872e5a631989c2925f5fac21199e221aa9112afbbc9aae66a2de) which adds this functionality to the parser.

Full expressions would include various fairly conventional examples not proposed here:

```swift
let x = 1 + if .random() { 3 } else { 4 }
```

but also some pretty strange ones such as

```swift
for b in [true] where switch b { case true: true case false: false } {}
```

The strange examples can mostly be considered "weird but harmless" but there are some source breaking edge cases, in particular in result builders:

```swift
var body: some View {
    VStack {
        if showButton {
            Button("Click me!", action: clicked)
        } else {
            Text("No button")
        }
        .someStaticProperty
    }
}
```

In this case, if `if` expressions were allowed to have postfix member expressions (which they aren't today, even in result builders), it would be ambiguous whether this should be parsed as a modifier on the `if` expression, or as a new expression. This could only be an issue for result builders, but the parser does not have the ability to specialize behavior for result builders. Note, this issue can happen today (and is why `One` exists for Regex Builders) but could introduce a new ambiguity for code that works this way today.

### `do` Expressions

`do` blocks could similarly be transformed into expressions, for example:

```swift
let foo: String = do {
    try bar()
} catch {
    "Error \(error)"
}
```

### Guard

Often enthusiasm for `guard` leads to requests for `guard` to have parity with `if`. Returning a value from a `guard`'s else is very common, and could potentially be sugared as

```swift
guard hasNativeStorage else { nil }
```

This is appealing, but is really a different proposal, of allowing omission `return` in `guard` statements.

### Multi-statement branches

The requirement that every branch be just a single expression leads to an unfortunate usability cliff:

```swift
let decoded =
  if isFastUTF8 {
    Log("Taking the fast path")
    withFastUTF8 { _decodeScalar($0, startingAt: i) }
  } else
    Log("Running error-correcting slow-path")
    foreignErrorCorrectedScalar(
      startingAt: String.Index(_encodedOffset: i))
  }
```

This is consistent with other cases, like multi-statement closures. But unlike in that case, where all that is needed is a `return` from the closure, this requires the user refactor the code back to the old mechanisms.

The trouble is, there is no great solution here. The approach taken by some other languages such as rust is to allow a bare expression at the end of the scope to be the expression value for that scope. There are stylistic preferences for and against this. More importantly, this would be a fairly radical new direction for Swift, and if proposed should probably be considered for all such cases (like function and closure return values too).

Alternatively, a new keyword could be introduced to make explicit that an expression value is being used as the value for this branch (Java uses `yield` for this in `switch` expressions).

### Either

As mentioned above, in result builders an `if` can be used to construct an `Either` type, which means the expressions in the branches could be of different types.

This could be done with `if` expressions outside result builders too, and would be a powerful new feature for Swift. However, it is large in scope (including the introduction of a language-integrated `Either` type) and should be considered in a separate proposal, probably after the community has adjusted to the more vanilla version proposed here.

## Alternatives Considered

### Sticking with the Status Quo

The list of [commonly rejected proposals](https://github.com/swiftlang/swift-evolution/blob/main/commonly_proposed.md) includes the subject of this proposal:

> **if/else and switch as expressions**: These are conceptually interesting things to support, but many of the problems solved by making these into expressions are already solved in Swift in other ways. Making them expressions introduces significant tradeoffs, and on balance, we haven't found a design that is clearly better than what we have so far.

The motivation section above outlines why the alternatives that exist today fall short. One of the reasons this proposal is narrow in scope is to bring the majority of value while avoiding resolving some of these more difficult trade-offs.

The lack of this feature puts Swift's [claim](https://www.swift.org/about/) to be a modern programming language under some strain. It is one of the few modern languages (Go being the other notable exception) not to support something along these lines.

### Alternative syntax

Instead of extending the current implicit return mechanism, where a single expression is treated as the returned value, this proposal could introduce a new syntax for expression versions of `if`/`switch`. For example, in Java:

```java
var response = switch (utterance) {
    case "thank you" -> "you’re welcome";
    case "atchoo" -> "gesundheit";
    case "fire!" -> {
        log.warn("fire detected");
        yield "everybody out!";  // yield = value of multi-statement branch
    };
    default -> {
        throw new IllegalStateException(utterance);
    };
};
```

A similar suggestion was made during [SE-0255: Implicit Returns from Single-Expression Functions](https://forums.swift.org/t/se-0255-implicit-returns-from-single-expression-functions/), where an alternate syntax for single-expression functions was discussed e.g. `func sum() -> Element = reduce(0, +)`. In that case, the core team did not consider introduction of a separate syntax for functions to be sufficiently motivated.

The main benefit to the alternate `->` syntax is to make it more explicit, but comes at the cost of needing to know about two different kinds of switch syntax. Note that this is orthogonal to, and does not solve, the separate goal of providing a way of explicitly "yielding" an expression value in the case of multi-statement branches (also shown here in this java example) versus taking the "last expression" approach.

Java did not introduce this syntax for `if` expressions. Since this is a goal for Swift, this implies:

```swift
let x = 
  if .random() -> 1
  else -> fatalError()
```

However, this then poses an issue when evolving to multi-statement branches. Unlike with `switch`, these would require introducing braces, leaving a combination of both braces _and_ a "this is an expression" sigil:

```swift
let x = 
  if .random() -> {
    let y = someComputation()
    y * 2
  } else -> fatalError()
```

Unlike Java and C, this "braces for 2+ arguments" style of `if` is out of keeping in Swift.

It is also not clear if the `->` would work well if expression status is brought to more kinds of statement e.g.

```swift
let foo: String = 
  do ->
    try bar()
  catch ns as NSError ->
    "Error \(error)"
```

or mixed branches with expressions and a return:

```swift
let x = 
  if .random() -> 1
  else -> return 2
```

If a future direction of full expressions is considered, the `->` form may not work so well, especially when single-line expressions are desired e.g.

```swift
// is this (p ? 1 : 2) + 3
// or p ? 1 : (2 + 3)
let x = if p -> 1 else -> 2 + 3
``` 

### Support for control flow

An earlier version of this proposal allowed use of `return` in a branch. Similar to `return`, statements that `break` or `continue` to a label, were considered a future direction.

Allowing new control flow out of expressions could be unexpected and error-prone. Swift currently only allows control flow out of expressions through thrown errors, which must be explicitly marked with `try` (or, in the case of `if` or `switch` branches, with `throw`) as an indication of the control flow to the programmer. Allowing other control flow out of expressions would undermine this principle. The control flow impact of nested return statements would become more difficult to reason about if we extend SE-0380 to support multiple-statement branches in the future. The use-cases for this functionality presented in the review thread were also fairly niche. Given the weak motivation and the potential problems introduced, the Language Workgroup accepts SE-0380 without this functionality.

## Source compatibility

As proposed, this addition has one source incompatibility, related to unreachable code. The following currently compiles, albeit with a warning that the `if` statement is unreachable (and the values in the branches unused):

```swift
func foo() {
  return
  if .random() { 0 } else { 0 }
}
```

but under this proposal, it would fail to compile with an error of "Unexpected non-void return value in void function" because it now parses as returning the `Int` expression from the `if`. This could be fixed in various ways (with `return;` or `return ()` or by `#if`-ing out the dead code explicitly).

Another similar case can occur if constant evaluation leads the compiler to ignore dead code:

```swift
func foo() -> Int {
  switch true {
  case true:
    return 0
  case false:
    print("unreachable")
  }
}
```

This currently _doesn't_ warn that the `false` case is unreachable (though probably should), but without special handling would after this proposal result in a type error that `()` does not match expected type `Int`.

Given these examples all require dead code, it seems reasonable to accept this source break rather than gate this change under a language version or add special handling to avoid the break.

## Effect on ABI stability

This proposal has no impact on ABI stability.

## Acknowledgments

Much of this implementation layers on top of ground work done by [Pavel Yaskevich](https://github.com/xedin), particularly the work done to allow [multi-statement closure type inference](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0326-extending-multi-statement-closure-inference.md).

Both [Nate Cook](https://forums.swift.org/t/if-else-expressions/22366/48) and [Michael Ilseman](https://forums.swift.org/t/omitting-returns-in-string-case-study-of-se-0255/24283) provided analysis of use cases in the standard library and elsewhere. Many community members have made a strong case for this change, most recently [Dave Abrahams](https://forums.swift.org/t/if-else-expressions/22366).
