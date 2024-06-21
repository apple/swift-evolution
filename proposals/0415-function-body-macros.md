# Function Body Macros

* Proposal: [SE-0415](0415-function-body-macros.md)
* Authors: [Doug Gregor](https://github.com/DougGregor)
* Review Manager: [Tony Allevato](https://github.com/allevato)
* Status: **Implemented (Swift 6.0)**
* Feature Flag: `BodyMacros`
* Review: [pitch](https://forums.swift.org/t/function-body-macros/66471), [review](https://forums.swift.org/t/se-0415-function-body-macros/68847), [returned for revision](https://forums.swift.org/t/returned-for-revision-se-0415-function-body-macros/69114), [second review](https://forums.swift.org/t/se-0415-second-review-function-body-macros/71644), [acceptance](https://forums.swift.org/t/accepted-se-0415-function-body-macros/72013)

## Table of contents

* [Introduction](#introduction)
* [Proposed solution](#proposed-solution)
* [Detailed design](#detailed-design)
   * [Declaring function body macros](#declaring-function-body-macros)
   * [Implementing function body macros](#implementing-function-body-macros)
   * [Composing function body macros](#composing-function-body-macros)
   * [Type checking of functions involving function body macros](#type-checking-of-functions-involving-function-body-macros)
* [Source compatibility](#source-compatibility)
* [Effect on ABI stability](#effect-on-abi-stability)
* [Effect on API resilience](#effect-on-api-resilience)
* [Future directions](#future-directions)
   * [Function body macros on closures](#function-body-macros-on-closures)
* [Alternatives considered](#alternatives-considered)
   * [Eliminating preamble macros](#eliminating-preamble-macros)
   * [Capturing the withSpan pattern in another macro role](#capturing-the-withspan-pattern-in-another-macro-role)
   * [Type-checking bodies as they were written](#type-checking-bodies-as-they-were-written)
* [Revision history](#revision-history)

## Introduction

Macros augment Swift programs with additional code, which can include new declarations, expressions, and statements. One of the key ways in which one might want to augment code---synthesizing or updating the body of a function---is not currently supported by the macro system. One can create new functions that have their own function bodies, but not provide, augment, or replace function bodies for a function declared by the user.

This proposal introduces *function body macros*, which do exactly that: allow the wholesale synthesis of function bodies given a declaration, as well as augmenting an existing function body with more functionality. This opens up a number of new use cases for macros, including:

* Synthesizing function bodies given the function declaration and some metadata, such as automatically synthesizing remote procedure calls that pass along the provided arguments.
* Augmenting function bodies to perform logging/tracing, check preconditions, or establish invariants.
* Replacing function bodies with a new implementation based on the one provided. For example, moving the body into a closure that is executed somewhere else, or treating the body as written as a domain specific language that the macro "lowers" to executable code.

## Proposed solution

This proposal introduces *function body macros*, which are [attached macros](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0389-attached-macros.md) that can augment a function (including initializers, deinitializers, and accessors) with a new body. For example, one could introduce a `Remote` macro that packages up arguments for a remote procedure call:

```swift
 @Remote
 func f(a: Int, b: String) async throws -> String
```

which could expand the function to provide a body, e.g.:

```swift
func f(a: Int, b: String) async throws -> String {
    return try await remoteCall(function: "f", arguments: ["a": a, "b": b])
}
```

One could also use a macro to introduce logging code on entry and exit to a function, expanding the following

```swift
@Logged
func g(a: Int, b: Int) -> Int {
  return a + b
}
```

into

```swift
func g(a: Int, b: Int) -> Int {
  log("Entering g(a: \(a), b: \(b))")
  defer {
    log("Exiting g")
  }
  return a + b
}
```

Or one could provide a macro that makes it easier to assume that a function that cannot be marked as `@MainActor` using [`assumeIsolated`](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0392-custom-actor-executors.md):

```swift
extension MyView: SomeDelegate {
  @AssumeMainActor
  nonisolated func onSomethingHappened(event: Event) {
    myView.title = newTitle(processing: event)
  }
}
```

which could expand to:

```swift
extension MyView: SomeDelegate {
  nonisolated func onSomethingHappened(event: Event) {
    MainActor.assumeIsolated {
      myView.title = newTitle(processing: event)
    }
  }
}
```

Function body macros can be applied to accessors as well, in which case they go on the accessor itself, e.g.,

```swift
var area: Double {
  @Logged get {
    return length * width
  }
}
```

When using the shorthand syntax for get-only properties, a function body macro can be applied to the property itself:

```swift
@Logged var area: Double {
  return length * width
}
```

## Detailed design

### Declaring function body macros

Function body macros are declared with the `body` role, which indicate that they can be attached to any kind of function, and can produce the contents of a function body. For example, here are declarations for the macros used above:

```swift
@attached(body) macro Remote() = #externalMacro(...)

@attached(body) macro Logged() = #externalMacro(...)

@attached(body) macro AssumeMainActor() = #externalMacro(...)
```

Like other attached macros, function body macros have no return type. 

### Implementing function body macros

Body macros are implemented with a type that conforms to the `BodyMacro` protocol:

```swift
/// Describes a macro that can create the body for a function.
public protocol BodyMacro: AttachedMacro {
  /// Expand a macro described by the given custom attribute and
  /// attached to the given declaration and evaluated within a
  /// particular expansion context.
  ///
  /// The macro expansion introduces code block items that will become the body for the
  /// given function. Any existing body will be implicitly ignored.
  static func expansion(
    of node: AttributeSyntax,
    providingBodyFor declaration: some DeclSyntaxProtocol & WithOptionalCodeBlockSyntax,
    in context: some MacroExpansionContext
  ) throws -> [CodeBlockItemSyntax]
}
```

That function may have a function body, which will be replaced by the code items produced from the macro implementation.

### Composing function body macros

At most one `body` macro can be applied to a given function. It receives the function declaration to which it is attached as it was written in the source code and produces a new function body.

### Type checking of functions involving function body macros

When a function body macro is applied, the macro-expanded function body will need to be type checked when it is incorporated into the program. However, the function might already have a body that was written by the developer, which can be inspected by the macro implementation. The function body as written must be syntactically well-formed (i.e., it must conform to the [Swift grammar](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/summaryofthegrammar/)) but will *not* be type-checked, so it need not be semantically well-formed.

This approach follows what other attached macros do: they operate on the syntax of the declaration to which they are attached, and the declaration itself need not have been type-checked before the macro is expanded. However,  this approach does lend itself to potential abuse. For example, one could create a `SQL` macro that expects the function body to be a SQL statement, then rewrites that into code that executes the query. For example, the input could be:

```swift
@SQL
func employees(hiredIn year: Int) -> [String] {
  SELECT 
    name
  FROM
    employees
  WHERE
    YEAR(hire_date) = year;
}
```

However, this would only work for places where the SQL grammar is a subset of the Swift grammar. Collapsing the same function into two lines would produce an error because it is not syntactically well-formed Swift:

```swift
@SQL
func employees(hiredIn year: Int) -> [String] {
  SELECT name FROM employees        // error: consecutive statements on a line must be separated by ';'
    WHERE YEAR(hire_date) = year;
}
```

The requirement for syntactic wellformedness should help rein in the more outlandish uses of function body macros, as well as making sure that existing tools that operate on source code will continue to work well even in the presence of body macros.

## Source compatibility

Function body macros introduce a new macro role into the existing attached macro syntax, and therefore does not have an impact on source compatibility.

## Effect on ABI stability

Macros are a source-to-source transformation tool that have no ABI impact.

## Effect on API resilience

Macros are a source-to-source transformation tool that have no effect on API resilience.

## Future directions

### Function body macros on closures

Function body macros as presented in this proposal are limited to declared functions, initializers, deinitializers, and accessors. In the future, they could be expanded to apply to closures as well, e.g.,

```swift
@Traced(z) { (x, y) in
  x + y
}
```

This extension would involve extending the `BodyMacro` protocol with another `expansion` method that accepts closure syntax. The primary challenge with applying function body macros to closures is the interaction with type inference, because closures generally occur within an expression and some of the macro arguments themselves might be part of the expression. In the example above, the `z` value could come from an outer scope and be the subject of type inference:

```swift
f(0) { z in
  @Traced(z) { (x, y) in
    x + y
  }
}
```

Macros are designed to avoid [multiply instantiating the same macro](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0382-expression-macros.md#macro-expansion), and have existing limitations in place to prevent the type checker from getting into a position where it is not obvious which macro to expand or the same macro needs to be expanded multiple times. To extend function body macros to closures will require a solution to this type-checking issue, and might be paired with lifting other restrictions on (e.g.) freestanding declaration macros.

### Preamble macros

The first reviewed revision of this proposal contained *preamble* macros, which let a macro introduce code at the beginning of a function without changing the rest of the function body. Preamble macros aren't technically necessary, because one could always write a function body macro that injects the preamble code into an existing body. However, preamble macros provide several end-user benefits over function body macros for the cases where they apply:

* Preamble macros can be composed, whereas function body macros cannot.
* Preamble macros don't change the code as written by the user, so they provide a better user experience (e.g., for diagnostics, code completion, and so on).

Preamble macros would be expressed as its own attached macro role (`preamble`), implemented with a type that conforms to the `PreambleMacro` protocol. Details are available in [the prior revision](https://github.com/swiftlang/swift-evolution/blob/f1b9da80315578666352a7d6d40a9f6cc936f69a/proposals/0415-function-body-macros.md). 

Preamble macros have been moved out to Future Directions because they represent a possible future, but not an obviously right one: preamble macros might not add sufficient expressivity to cover the cost of the complexity they introduce, and another kind of macro (like the "wrapper" macro below) might provide a more reasonable tradeoff between expressivity and complexity.

### Wrapper macros

A number of use cases for body macros involve "wrapping" the existing body in additional logic. For example, consider an alternative formulation of the `Traced` macro (let's call it `@TracedWithSpan`) could make use of the [`withSpan` API](https://swiftpackageindex.com/apple/swift-distributed-tracing/1.0.1/documentation/tracing) such that a function such as:

```swift
@TracedWithSpan("Doing complicated math")
func h(a: Int, b: Int) -> Int {
  return a + b
}
```

will expand to:

```swift
func h(a: Int, b: Int) -> Int {
  withSpan("Doing complicated math") {
    return a + b
  }
}
```

This `withSpan` function used here is one instance of a fairly general pattern in Swift, where a function accepts a closure argument and runs it with some extra contextual parameters. As we with the `preamble` macro role mentioned above, we could introduce a special macro role that describes this pattern: the macro would not see the function body that was written by the developer at all, but would instead have a function value representing the body that it could call opaquely. For example, the `TracedWithSpan` example function `h` would expand to:

```swift
func h(a: Int, b: Int) -> Int {
  withSpan("Doing complicated math", body: h-impl)
}
```

With this approach, the original function body for `h` would be type-checked prior to macro expansion, and then would be handed off to the macro as an opaque value `h-impl` to be called by `withSpan`. The macro could introduce its own closure wrapping that body as needed, e.g.,

```swift
@TracedWithSpan("Doing complicated math", { span in
  span.attributes["operation"] = "addition"
})
func myMath(a: Int, b: Int) -> Int {
  return a + b
}
```

could expand to:

```swift
func myMath(a: Int, b: Int) -> Int {
  return withSpan("Doing complicated math") { span in
    span.attributes["operation"] = "addition"
    return myMath-impl()
  }
}
```

The advantage of this approach over allowing a `body` macro to replace a body is that we can type-check the  function body as it was written, and only need to do so once---then it becomes a value of function type that's passed along to the underying macro. Also like preamble macros, this approach can compose, because the result of one macro could produce another value of function type that can be passed along to another macro. [Python decorators](https://www.datacamp.com/tutorial/decorators-python) have been successful in that language for customizing the behavior of functions in a similar manner.

## Alternatives considered

### Type-checking bodies as they were written

As noted previously, not checking the body of functions that was written by the user and then replaced by a `body` macro has some down sides. For one, it allows some abuse, where code that wouldn't make sense in Swift is permitted to be written by the user and then significantly altered by the `body` macro. Moreover, wherever the macro is performing some modification that makes ill-formed code into well-formed code (even by something as simple as introducing a `span` variable like `@Traced` does), tools that cannot reason about the macro expansion might be less useful: code completion won't know to provide `span` as a possible completion, nor will it know what type `span` would have. Therefore, the experience of writing code that makes use of `body` macros could be significantly worse than that for normal Swift code.

On the other hand, type-checking the function bodies before macro expansion has other issues. Type checking is a significant part of compilation time, and having to type-check the body of a function twice---once before macro expansion, once after---could be prohibitively expensive. Type-checking the function body before macro expansion also limits what can be expressed by body macros, including making some use cases (like the `@Traced` macro described earlier) impossible to express without more extensions to the model.

## Revision history

* Revision 3:
  * Narrowed the focus down to `body` macros.
  * Moved preamble macros into Future Directions, added discussion of wrapper macros.
* Revision 2:
  * Clarify that preamble macro-introduced local names can shadow names from outer scopes
  * Clarify the effect of function body macros on single-expression functions and implicit returns
* Revision 1:
  * Allow preamble macros to introduce names.
  * Introduce `@AssumeMainActor` example macro for body macros that perform replacement.
  * Switch `@Traced` example over to be a preamble macro with push/pop operations, so it can nicely introduce `span`.
  * Allow function body macros to be applied to properties that use the shorthand getter syntax.
