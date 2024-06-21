# `buildPartialBlock` for result builders

* Proposal: [SE-0348](0348-buildpartialblock.md)
* Author: [Richard Wei](https://github.com/rxwei)
* Implementation: [apple/swift#41576](https://github.com/apple/swift/pull/41576)
* Review Manager: [Ben Cohen](https://github.com/airspeedswift)
* Status: **Implemented (Swift 5.7)**

## Overview

We introduce a new result builder customization point that allows components of a block to be combined pairwise.

```swift
@resultBuilder
enum Builder {
    /// Builds a partial result component from the first component.
    static func buildPartialBlock(first: Component) -> Component
    
    /// Builds a partial result component by combining an accumulated component
    /// and a new component.  
    /// - Parameter accumulated: A component representing the accumulated result
    ///   thus far. 
    /// - Parameter next: A component representing the next component after the
    ///   accumulated ones in the block.
    static func buildPartialBlock(accumulated: Component, next: Component) -> Component
}
```

When `buildPartialBlock(first:)` and `buildPartialBlock(accumulated:next:)` are both provided, the [result builder transform](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0289-result-builders.md#the-result-builder-transform) will transform components in a block into a series of calls to `buildPartialBlock`, combining one subsequent line into the result at a time.

```swift
// Original
{
    expr1
    expr2
    expr3
}

// Transformed
// Note: `buildFinalResult` and `buildExpression` are called only when they are defined, just like how they behave today.
{
    let e1 = Builder.buildExpression(expr1)
    let e2 = Builder.buildExpression(expr2)
    let e3 = Builder.buildExpression(expr3)
    let v1 = Builder.buildPartialBlock(first: e1)
    let v2 = Builder.buildPartialBlock(accumulated: v1, next: e2)
    let v3 = Builder.buildPartialBlock(accumulated: v2, next: e3)
    return Builder.buildFinalResult(v3)
}
```

The primary goal of this feature is to reduce the code bloat caused by overloading `buildBlock` for multiple arities, allowing libraries to define builder-based generic DSLs with joy and ease.

## Motivation

Among DSLs powered by result builders, it is a common pattern to combine values with generic types in a block to produce a new type that contains the generic parameters of the components.  For example, [`ViewBuilder`](https://developer.apple.com/documentation/swiftui/viewbuilder) and [`SceneBuilder`](https://developer.apple.com/documentation/swiftui/scenebuilder) in SwiftUI use `buildBlock` to combine views and scenes without losing strong types. 

```swift
extension SceneBuilder {
  static func buildBlock<Content>(Content) -> Content
  static func buildBlock<C0, C1>(_ c0: C0, _ c1: C1) -> some Scene where C0: Scene, C1: Scene
  ...
  static func buildBlock<C0, C1, C2, C3, C4, C5, C6, C7, C8, C9>(_ c0: C0, _ c1: C1, _ c2: C2, _ c3: C3, _ c4: C4, _ c5: C5, _ c6: C6, _ c7: C7, _ c8: C8, _ c9: C9) -> some Scene where C0: Scene, C1: Scene, C2: Scene, C3: Scene, C4: Scene, C5: Scene, C6: Scene, C7: Scene, C8: Scene, C9: Scene
}
``` 

Due to the lack of variadic generics, `buildBlock` needs to be overloaded for any supported block arity. This unfortunately increases code size, causes significant code bloat in the implementation and documentation, and it is often painful to write and maintain the boiletplate.

While this approach works for types like `ViewBuilder` and `SceneBuilder`, some builders need to define type combination rules that are far too complex to implement with overloads. One such example is [`RegexComponentBuilder`](https://github.com/apple/swift-experimental-string-processing/blob/85c7d906dd871364357156126278d9d427936ca4/Sources/_StringProcessing/RegexDSL/Builder.swift#L13) in [Declarative String Processing](https://github.com/apple/swift-experimental-string-processing/blob/main/Documentation/DeclarativeStringProcessing.md). 

The regex builder DSL is designed to allow developers to easily compose regex patterns. [Strongly typed captures](https://github.com/apple/swift-experimental-string-processing/blob/main/Documentation/Evolution/StronglyTypedCaptures.md#strongly-typed-regex-captures) are represented as part of the `Match` generic parameter in the `Regex` type, which has a builder-based initializer.

```swift
struct Regex<Match> {
   init(@RegexComponentBuilder _ builder: () -> Self)
}
```

> #### Recap: Regular expression capturing basics
> 
> When a regular expression does not contain any capturing groups, its `Match` type is `Substring`, which represents the whole matched portion of the input.
>
> ```swift
> let noCaptures = #/a/# // => Regex<Substring>
> ```
>
> When a regular expression contains capturing groups, i.e. `(...)`, the `Match` type is extended as a tuple to also contain *capture types*. Capture types are tuple elements after the first element.
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

Using the result builder syntax, the regular expression above becomes:

```swift
let regex = Regex {
    "a"                             // => Regex<Substring>
    OneOrMore {                     // {
        Capture { OneOrMore("b") }  //     => Regex<(Substring, Substring)>
        "c"                         //     => Regex<Substring>
        Capture { OneOrMore("d") }  //     => Regex<(Substring, Substring)>
    }                               // } => Regex<(Substring, Substring, Substring)>
    "e"                             // => Regex<Substring>
    Optionally { Capture("f") }     // => Regex<(Substring, Substring?)>
}                                   // => Regex<(Substring, Substring, Substring, Substring?)>
                                    //                      ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
                                    //                               Capture types

let result = "abbcddbbcddef".firstMatch(of: regex)
// => MatchResult<(Substring, Substring, Substring, Substring?)>
```

`RegexComponentBuilder` concatenates the capture types of all components as a flat tuple, forming a new `Regex` whose `Match` type is `(Substring, CaptureType...)`. We can define the following `RegexComponentBuilder`:

```swift
@resultBuilder
enum RegexComponentBuilder {
    static func buildBlock() -> Regex<Substring>
    static func buildBlock<Match>(_: Regex<Match>) -> Regex<Match>
    // Overloads for non-tuples:
    static func buildBlock<WholeMatch0, WholeMatch1>(_: Regex<WholeMatch0>, _: Regex<WholeMatch1>) -> Regex<Substring>
    static func buildBlock<WholeMatch0, WholeMatch1, WholeMatch2>(_: Regex<WholeMatch0>, _: Regex<WholeMatch1>, _: Regex<WholeMatch2>) -> Regex<Substring>
    ...
    static func buildBlock<WholeMatch0, WholeMatch1, WholeMatch2, ..., WholeMatch10>(_: Regex<WholeMatch0>, _: Regex<WholeMatch1>, _: Regex<WholeMatch2>, ..., _: Regex<WholeMatch10>) -> Regex<Substring>
    // Overloads for tuples:
    static func buildBlock<W0, W1, C0>(_: Regex<(W0, C0)>, _: Regex<W1>) -> Regex<(Substring, C0)>
    static func buildBlock<W0, W1, C0>(_: Regex<W0>, _: Regex<(W1, C0)>) -> Regex<(Substring, C0)>
    static func buildBlock<W0, W1, C0, C1>(_: Regex<(W0, C0, C1)>, _: Regex<W1>) -> Regex<(Substring, C0, C1)>
    static func buildBlock<W0, W1, C0, C1>(_: Regex<(W0, C0)>, _: Regex<(W1, C1)>) -> Regex<(Substring, C0, C1)>
    static func buildBlock<W0, W1, C0, C1>(_: Regex<W0>, _: Regex<(W1, C0, C1)>) -> Regex<(Substring, C0, C1)>
    ...
    static func buildBlock<W0, W1, W2, W3, W4, C0, C1, C2, C3, C4, C5, C6>(
       _: Regex<(W0, C0, C1)>, _: Regex<(W1, C2)>, _: Regex<(W3, C3, C4, C5)>, _: Regex<(W4, C6)>
    ) -> Regex<(Substring, C0, C1, C2, C3, C4, C5, C6)>
    ...
}
```

Here we just need to overload for all tuple combinations for each arity...  Oh my!  That is an `O(arity!)` combinatorial explosion of `buildBlock` overloads; compiling these methods alone could take hours.

## Proposed solution

This proposal introduces a new block-building approach similar to building heterogeneous lists. Instead of calling a single method to build an entire block wholesale, this approach recursively builds a partial block by taking one new component at a time, and thus significantly reduces the number of overloads.

We introduce a new customization point to result builders via two user-defined static methods:

```swift
@resultBuilder
enum Builder {
    static func buildPartialBlock(first: Component) -> Component
    static func buildPartialBlock(accumulated: Component, next: Component) -> Component
}
```

When `buildPartialBlock(first:)` and `buildPartialBlock(accumulated:next:)` are both defined, the result builder transform will turn components in a block into a series of calls to `buildPartialBlock`, combining components from top to bottom.

With this approach, many result builder types with overloaded `buildBlock` can be simplified. For example, the `buildBlock` overloads in SwiftUI's `SceneBuilder` could be simplified as the following:

```swift
extension SceneBuilder {
    static func buildPartialBlock(first: some Scene) -> some Scene 
    static func buildPartialBlock(accumulated: some Scene, next: some Scene) -> some Scene 
}
```

Similarly, the overloads of `buildBlock` in `RegexComponentBuilder` can be vastly reduced from `O(arity!)`, down to `O(arity^2)` overloads of `buildPartialBlock(accumulated:next:)`. For an arity of 10, 100 overloads are trivial compared to over 3 million ones.

```swift
extension RegexComponentBuilder {
    static func buildPartialBlock<M>(first regex: Regex<M>) -> Regex<M>
    static func buildPartialBlock<W0, W1>(accumulated: Regex<W0>, next: Regex<W1>) -> Regex<Substring>
    static func buildPartialBlock<W0, W1, C0>(accumulated: Regex<(W0, C0)>, next: Regex<W1>) -> Regex<(Substring, C0)>
    static func buildPartialBlock<W0, W1, C0>(accumulated: Regex<W0>, next: Regex<(W1, C0)>) -> Regex<(Substring, C0)>
    static func buildPartialBlock<W0, W1, C0, C1>(accumulated: Regex<W0>, next: Regex<(W1, C0, C1)>) -> Regex<(Substring, C0, C1)>
    static func buildPartialBlock<W0, W1, C0, C1>(accumulated: Regex<(W0, C0, C1)>, next: Regex<W1>) -> Regex<(Substring, C0, C1)>
    static func buildPartialBlock<W0, W1, C0, C1>(accumulated: Regex<(W0, C0)>, next: Regex<(W1, C1)>) -> Regex<(Substring, C0, C1)>
    ...
}
```

### Early adoption feedback

- In the [regex builder DSL](https://forums.swift.org/t/pitch-regex-builder-dsl/56007), `buildPartialBlock` reduced the number of required overloads from millions (`O(arity!)`) down to hundreds (`O(arity^2)`).
- In the pitch thread, [pointfreeco/swift-parsing reported](https://forums.swift.org/t/pitch-buildpartialblock-for-result-builders/55561/10) that `buildPartialBlock` enabled the deletion of 21K lines of generated code, increased arity support, and reduced compile times from 20 seconds to <2 seconds in debug mode.

## Detailed design

When a type is marked with `@resultBuilder`, the type was previously required to have at least one static `buildBlock` method. With this proposal, such a type is now required to have either at least one static `buildBlock` method, or both `buildPartialBlock(first:)` and `buildPartialBlock(accumulated:next:)`.

In the result builder transform, the compiler will look for static members `buildPartialBlock(first:)` and `buildPartialBlock(accumulated:next:)` in the builder type. If the following conditions are met:

* Both methods `buildPartialBlock(first:)` and `buildPartialBlock(accumulated:next:)` exist.
* The availability of the enclosing declaration is greater than or equal to the availability of `buildPartialBlock(first:)` and `buildPartialBlock(accumulated:next:)`.

Then, a non-empty block will be transformed to the following:

```swift
// Original
{
    expr1
    expr2
    expr3
}

// Transformed
// Note: `buildFinalResult` and `buildExpression` are called only when they are defined, just like how they behave today.
{
    let e1 = Builder.buildExpression(expr1)
    let e2 = Builder.buildExpression(expr2)
    let e3 = Builder.buildExpression(expr3)
    let v1 = Builder.buildPartialBlock(first: e1)
    let v2 = Builder.buildPartialBlock(accumulated: v1, next: e2)
    let v3 = Builder.buildPartialBlock(accumulated: v2, next: e3)
    return Builder.buildFinalResult(v3)
}
```

Otherwise, the result builder transform will transform the block to call `buildBlock` instead as proposed in [SE-0289](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0289-result-builders.md).

## Source compatibility

This proposal does not intend to introduce source-breaking changes. Although, if an existing result builder type happens to have static methods named `buildPartialBlock(first:)` and `buildPartialBlock(accumulated:next:)`, the result builder transform will be creating calls to those methods instead and may cause errors depending on `buildPartialBlock`'s type signature and implementation. Nevertheless, such cases should be extremely rare.

## Effect on ABI stability

This proposal does not contain ABI changes.

## Effect on API resilience

This proposal does not contain API changes.

## Alternatives considered

### Prefer viable `buildBlock` overloads to `buildPartialBlock`

As proposed, the result builder transform will always prefer `buildPartialBlock` to `buildBlock` when they are both defined. One could argue that making a single call to a viable overload of `buildBlock` would be more efficient and more customizable. However, because the result builder transform currently operates before type inference has completed, it would increase the type checking complexity to decide to call `buildBlock` or `buildPairwiseBlock` based on argument types. None of the informal requirements (`buildBlock`, `buildOptional`, etc) of result builders depend on argument types when being transformed to by the result builder transform.

### Use nullary `buildPartialBlock()` to form the initial value

As proposed, the result builder transform calls unary `buildPartialBlock(first:)` on the first component in a block before calling `buildPartialBlock(accumulated:next:)` on the rest.  While it is possible to require the user to define a nullary `buildPartialBlock()` method to form the initial result, this behavior may be suboptimal for result builders that do not intend to support empty blocks, e.g. SwiftUI's `SceneBuilder`. Plus, the proposed approach does allow the user to define an nullary `buildBlock()` to support building an empty block.

### Rely on variadic generics

It can be argued that variadic generics would resolve the motivations presented. However, to achieve the concatenating behavior needed for `RegexComponentBuilder`, we would need to be able to express nested type sequences, perform collection-like transformations on generic parameter packs such as dropping elements and splatting. 

```swift
extension RegexComponentBuilder {
  static func buildBlock<(W, (C...))..., R...>(_ components: Regex<(W, C...)>) -> Regex<(Substring, (R.Match.dropFirst()...).splat())>
}
```

Such features would greatly complicate the type system.

### Alternative names

#### Overload `buildBlock` method name

Because the proposed feature overlaps `buildBlock`, one could argue for reusing `buildBlock` as the method base name instead of `buildPartialBlock` and using arguemnt labels to distinguish whether it is the pairwise version, e.g. `buildBlock(partiallyAccumulated:next:)` or `buildBlock(combining:into:)`.

```swift
extension Builder {
  static func buildBlock(_: Component) -> Component
  static func buildBlock(partiallyAccumulated: Component, next: Component) -> Component
}
```

However, the phrase "build block" does not have a clear indication that the method is in fact building a partial block, and argument labels do not have the prominence to carry such indication. 

#### Use `buildBlock(_:)` instead of `buildPartialBlock(first:)`

The unary base case method `buildPartialBlock(first:)` and `buildBlock(_:)` can be viewed as being functionally equivalent, so one could argue for reusing `buildBlock`. However, as mentioned in [Overload `buildBlock` method name](#overload-buildblock-method-name), the phrase "build block" lacks clarity.

A more important reason is that `buildPartialBlock(first:)`, especially with its argument label `first:`, leaves space for a customization point where the developer can specify the direction of combination. As a future direction, we could allow `buildPartialBlock(last:)` to be defined instead of `buildPartialBlock(first:)`, and in this scenario the result builder transform will first call `buildPartialBlock(last:)` on the _last_ component and then call `buildPartialBlock(accumulated:next:)` on each preceeding component.

#### Different argument labels

The proposed argument labels `accumulated:` and `next:` took inspirations from some precedents in the standard library:
- "accumulated" is used as an argument name of [`Array.reduce(into:_:)`](https://developer.apple.com/documentation/swift/array/3126956-reduce).
- "next" is used as an argument name of [`Array.reduce(_:_:)`](https://developer.apple.com/documentation/swift/array/2298686-reduce)   

Meanwhile, there are a number of alternative argument labels considered in the place of `accumulated:` and `next:`.  

Possible replacements for `accumulated:`:
- `partialResult:`
- `existing:`
- `upper:`
- `_:`

Possible replacements for `next:`:
- `new:`
- `_:`

We believe that "accumulated" and "next" have the best overall clarity.
