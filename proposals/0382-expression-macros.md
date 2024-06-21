# Expression Macros

* Proposal: [SE-0382](0382-expression-macros.md)
* Authors: [Doug Gregor](https://github.com/DougGregor)
* Review Manager: [Xiaodi Wu](https://github.com/xwu)
* Status: **Implemented (Swift 5.9)**
* Implementation: Partial implementation is available in `main` under the experimental feature flag `Macros`. An [example macro repository](https://github.com/DougGregor/swift-macro-examples) provides a way to experiment with this feature.
* Review: ([pitch](https://forums.swift.org/t/pitch-expression-macros/61499)) ([pitch #2](https://forums.swift.org/t/pitch-2-expression-macros/61861)) ([review #1](https://forums.swift.org/t/se-0382-expression-macros/62090)) ([returned for revision](https://forums.swift.org/t/returned-for-revision-se-0382-expression-macros/62898)) ([pitch #3](https://forums.swift.org/t/se-0382-expression-macros-mini-pitch-for-updates/62810)) ([review #2](https://forums.swift.org/t/se-0382-second-review-expression-macros/63064)) ([acceptance](https://forums.swift.org/t/accepted-se-0382-expression-macros/63495))

## Introduction

Expression macros provide a way to extend Swift with new kinds of expressions, which can perform arbitrary syntactic transformations on their arguments to produce new code. Expression macros make it possible to extend Swift in ways that were only previously possible by introducing new language features, helping developers build more expressive libraries and eliminate extraneous boilerplate.

## Motivation

Expression macros are one part of the [vision for macros in Swift](https://github.com/swiftlang/swift-evolution/pull/1927), which lays out general motivation for introducing macros into the language. Expressions in particular are an area where the language already provides decent abstractions for factoring out runtime behavior, because one can create a function that you call as an expression from anywhere. However, with a few hard-coded exceptions like `#file` and `#line`, an expression cannot reason about or modify the source code of the program being compiled. Such use cases will require external source-generating tools, which don't often integrate cleanly with other tooling.

## Proposed solution

This proposal introduces the notion of expression macros, which are used as expressions in the source code (marked with `#`) and are expanded into expressions. Expression macros can have parameters and a result type, much like a function, which describes the effect of the macro expansion on the expression without requiring the macro to actually be expanded first.

The actual macro expansion is implemented with source-to-source translation on the syntax tree: the expression macro is provided with the syntax tree for the macro expansion itself (e.g., starting with the `#` and ending with the last argument), which it can rewrite into the expanded syntax tree. That expanded syntax tree will be type-checked against the result type of the macro.

As a simple example, let's consider a `stringify` macro that takes a single argument as input and produces a tuple containing both the original argument and also a string literal containing the source code for that argument. This macro could be used in source code as, for example:

```swift
#stringify(x + y)
```

and would be expanded into

```swift
(x + y, "x + y")
```

The type signature of a macro is part of its declaration, which looks a lot like a function:

```swift
@freestanding(expression) macro stringify<T>(_: T) -> (T, String)
```

### Type-checked macro arguments and results

Macro arguments are type-checked against the parameter types of the macro prior to instantiating the macro. For example, the macro argument `x + y` will be type-checked; if it is ill-formed (for example, if `x` is an `Int` and `y` is a `String`), the macro will never be expanded. If it is well-formed, the generic parameter `T` will be inferred to the result of `x + y`, and that type is carried through to the result type of the macro. There are several benefits to this type-checked model:

* Macro implementations are guaranteed to have well-typed arguments as inputs, so they don't need to be concerned about incorrect code being passed into the macro.
* Tools can treat macros much like functions, providing the same affordances for code completion, syntax highlighting, and so on, because the macro arguments follow the same rules as other Swift code. 
* A macro expansion expression can be partially type-checked without having to expand the macro. This allows tools to still have reasonable results without performing macro expansion, as well as improving compile-time performance because the same macro will not be expanded repeatedly during type inference.

When the macro is expanded, the expanded syntax tree is type-checked against the result type of the macro. In the `#stringify(x + y)` case, this means that if `x + y` had type `Int`, the expanded syntax tree (`(x + y, "x + y")`) is type-checked with a contextual type of `(Int, String)`.

The type checking of macro expressions is similar to type-checking a call, allowing type inference information to flow from the macro arguments to the result type and vice-versa. For example, given:

```swift
let (a, b): (Double, String) = #stringify(1 + 2)
```

the integer literals `1` and `2` would be assigned the type `Double`.

### Syntactic translation

Macro expansion is a syntactic operation, which takes as input a well-formed syntax tree consisting of the full macro expansion expression (e.g., `#stringify(x + y)`) and produces a syntax tree as output. The resulting syntax tree is then type-checked based on the macro result type.

Syntactic translation has a number of benefits over more structured approaches such as direct manipulation of a compiler's Abstract Syntax Tree (AST) or internal representation (IR):

* A macro expansion can use the full Swift language to express its effect. If it can be written as Swift source code at that position in the grammar, a macro can expand to it.
* Swift programmers understand Swift source code, so they can reason about the output of a macro when applied to their source code. This helps both when authoring and using macros.
* Source code that uses macros can be "expanded" to eliminate the use of the macro, for example to make it easier to reason about or debug, or make it work with an older Swift compiler that doesn't support macros.
* The compiler's AST and internal representation need not be exposed to clients, which would limit the ability of the compiler to evolve and improve due to backward-compatibility concerns.

On the other hand, purely syntactic translations have a number of downsides, too:

* Syntactic macro expansions are prone to compile-time failures, because we're effectively working with source code as strings, and it's easy to introduce (e.g.) syntax errors or type errors in the macro implementation.
* Syntactic macro expansions are re-parsed and re-type-checked, which incurs more compile-time overhead than an approach that (say) manipulated the AST or IR directly.
* Syntactic macros are not *hygienic*, meaning that the way in which a macro expansion is processed depends on the environment in which it is expanded, and can affect that environment. 

The proposed macro design attempts to mitigate these problems, but they are somewhat fundamental to the use of syntactic macros. On balance, the ease-of-use and easy-of-interpretation of syntactic macros outweighs these problems.

### Macros defined as separate programs

Macro definitions operate on syntax trees. Broadly speaking, there are two different ways in which a macro's expansion operation can be defined:

* *A declarative set of transformations*: this involves extending the language with a special syntax that allows macros to define how the macro is expanded given the macro inputs, and the compiler applies those rules for each macro expansion. The C preprocessor employs a simplistic form of this, but Racket's [pattern-based macros](https://docs.racket-lang.org/guide/pattern-macros.html) and Rust's [declarative macros](https://doc.rust-lang.org/book/ch19-06-macros.html#declarative-macros-with-macro_rules-for-general-metaprogramming)  offer more advanced rules that match the macro arguments to a pattern and then perform a rewrite to new syntax as described in the macro. For Swift to adopt this approach, we would likely need to invent a pattern language for matching and rewriting syntax trees.
* *An executable program that transforms the source*: this involves running a program that manipulates the program syntax directly. How the program is executed depends a lot on the environment: [Scala 3 macros](https://docs.scala-lang.org/scala3/guides/macros/macros.html) benefit from the use of the JVM so they can intertwine target code (the program being generated) and host code (where the compiler is running), whereas Rust's [procedural macros](https://doc.rust-lang.org/reference/procedural-macros.html) are built into a separate crate that the compiler interacts with. For Swift to adopt this approach, we would either need to build a complete interpreter for Swift code in the compiler, or take an approach similar to Rust's and build macro definitions as separate programs that the compiler can interact with.

We propose the latter approach, where a macro definition is a separate program that operates on Swift syntax trees using the [swift-syntax](https://github.com/apple/swift-syntax/) package. Expression macros are defined as types that conform to the `ExpressionMacro` protocol:

```swift
public protocol ExpressionMacro: FreestandingMacro {
  /// Expand a macro described by the given freestanding macro expansion
  /// within the given context to produce a replacement expression.
  static func expansion(
    of node: some FreestandingMacroExpansionSyntax,
    in context: some MacroExpansionContext
  ) async throws -> ExprSyntax
}
```

The `expansion(of:in:)` method takes as arguments the syntax node for the macro expansion expression (e.g., `#stringify(x + y)`) and a "context" that provides more information about the compilation context in which the macro is being expanded. It produces a macro result that includes the rewritten syntax tree.

The specifics of  `Macro`, `ExpressionMacro`, and `MacroExpansionContext` will follow in the Detailed Design section.

### The `stringify` macro implementation

Let's continue with the implementation of the `stringify` macro. It's a new type `StringifyMacro` that conforms to `ExpressionMacro`: 

```swift
import SwiftSyntax
import SwiftSyntaxBuilder
import SwiftSyntaxMacros

public struct StringifyMacro: ExpressionMacro {
  public static func expansion(
    of node: some FreestandingMacroExpansionSyntax,
    in context: some MacroExpansionContext
  ) -> ExprSyntax {
    guard let argument = node.argumentList.first?.expression else {
      fatalError("compiler bug: the macro does not have any arguments")
    }

    return "(\(argument), \(literal: argument.description))"
  }
}
```

The `expansion(of:in:)` function is fairly small, because the `stringify` macro is relatively simple. It extracts the macro argument from the syntax tree (the `x + y` in `#stringify(x + y)`) and then forms the resulting tuple expression by interpolating in the original argument both as a value and then as source code in [a string literal](https://github.com/apple/swift-syntax/blob/main/Sources/SwiftSyntaxBuilder/ConvenienceInitializers.swift#L259-L265). That string is then parsed as an expression (producing an `ExprSyntax` node) and returned as the result of the macro expansion. This is a simple form of quasi-quoting provided by the `SwiftSyntaxBuilder` module, implemented by making the major syntax nodes (`ExprSyntax` in this case, for expressions) conform to [`ExpressibleByStringInterpolation`](https://developer.apple.com/documentation/swift/expressiblebystringinterpolation), where existing syntax nodes can be interpolated into string literals containing the expanded Swift code.

The `StringifyMacro` struct is the implementation for the `stringify` macro declared earlier. We will need to tie these together in the source code via some mechanism. We propose to provide a builtin macro that names the module and the `ExpressionMacro` type name within the macro declaration following an `=`, e.g.,

```swift
@freestanding(expression)
macro stringify<T>(_: T) -> (T, String) =
  #externalMacro(module: "ExampleMacros", type: "StringifyMacro")
```

## Detailed design

There are two major pieces to the macro design: how macros are declared and expanded within a program, and how they are implemented as separate programs. The following sections provide additional details.

### Macro declarations

A macro declaration is described by the following grammar:

```
declaration -> macro-declaration

macro-declaration -> macro-head identifier generic-parameter-clause[opt] macro-signature macro-definition[opt] generic-where-clause[opt]

macro-head -> attributes[opt] declaration-modifiers[opt] 'macro'

macro-signature -> parameter-clause macro-function-signature-result[opt]

macro-function-signature-result -> '->' type

macro-definition -> '=' expression
```

The `@freestanding(expression)` attribute applies only to macros. It indicates that the macro is an expression macro. The "freestanding" terminology comes from the [macros vision document](https://github.com/swiftlang/swift-evolution/pull/1927), and is used to describe macros that are expanded with the leading `#` syntax.

Macro signatures are function-like, with a parameter clause (that may be empty) and an optional result type.

Macros can only be declared at file scope. They can be overloaded in the same way as functions, so long as the argument labels, parameter types, or result type differ.

The `macro-definition` provides the implementation used to expand the macro. It is parsed as a general expression, but must always be a `macro-expansion-expression`, so all non-builtin macros are defined in terms of other macros, terminating in a builtin macro whose definition is provided by the compiler. The arguments provided within the `macro-expansion-expression` of the macro definition must either be direct references to the parameters of the enclosing macro or must be literals. The `macro-expansion-expression` is type-checked (to ensure that the argument and result types make sense), but no expansion is performed at the time of definition. Rather, expansion of the macro referenced by the `macro-definition` occurs when the macro being declared is expanded. See the following section on macro expansion for more information.

Macro parameters may have default arguments, but those default arguments can only consist of literal expressions and other macro expansions.

Macros can have opaque result types. The rules for uniqueness of opaque result types for macros are somewhat different from opaque result types of functions, because each macro expansion can easily produce a different type. Therefore, each macro expansion producing an opaque result type will be considered to have a distinct type, e.g., the following is ill-formed:

```swift
@freestanding(expression) macro someMacroWithOpaqueResult() -> some Collection<UInt8>

var a = #someMacroWithOpaqueResult
a = #someMacroWithOpaqueResult // cannot assign value with type of macro expansion here to opaque type from macro expansion above
```

### Macro expansion

A macro expansion expression is described by the following grammar:

```
primary-expression -> macro-expansion-expression
macro-expansion-expression -> '#' identifier generic-argument-clause[opt] function-call-argument-clause[opt] trailing-closures[opt]
```

The `#` syntax for macro expansion expressions was specifically chosen because Swift already contains a number of a `#`-prefixed expressions that are macro-like in nature, some of which could be implemented directly as expression macros. The macro referenced by the `identifier` must be an expression macro, as indicated by `@freestanding(expression)` on the corresponding macro declaration.

Both `function-call-argument-clause` and `trailing-closures` are optional. When both are omitted, the macro is expanded as-if the empty argument list `()` was provided. Macros are not first-class entities in the way functions are, so they cannot be passed around as values and do not need an "unapplied macro" syntax. This allows `#line` et al to be macros without requiring them to be written as `#line()`. There is some precedent for this with property wrappers, which will also be used for attached macros.

When a macro expansion is encountered in the source code, its expansion occurs in two phases. The first phase is the type-check phase, where the arguments to the macro are type-checked against the parameters of the named macro, and the result type of the named macro is checked against the context in which the macro expansion occurs. This type-checking is equivalent to that performed for a function call, and does not involve the macro definition.

The second phase is the macro expansion phase, during which the syntax of the macro arguments is provided to the macro definition. For builtin-macro definitions, the behavior at this point depends on the semantics of the macro, e.g., the `externalMacro` macro invokes the external program and provides it with the source code of the macro expansion. For other macros, the arguments are substituted into the `macro-expansion-expression` of the definition. For example:

```swift
@freestanding(expression) macro prohibitBinaryOperators<T>(_ value: T, operators: [String]) -> T =
    #externalMacro(module: "ExampleMacros", type: "ProhibitBinaryOperators")
@freestanding(expression) macro addBlocker<T>(_ value: T) -> T = #prohibitBinaryOperators(value, operators: ["+"])

#addBlocker(x + y * z)
```

Here, the macro expansion of `#addBlocker(x + y * z)` will first expand to `#prohibitBinaryOperators(x + y * z, operators: ["+"])`. Then that expansion will be processed by the `ExampleMacros.ProhibitBinaryOperators`, which would be defined as a struct conforming to `ExpressionMacro`. 

Macro expansion produces new source code (in a syntax tree), which is then type-checked using the original macro result type as its contextual type. For example, the `stringify` example macro returned a `(T, String)`, so when given an argument of type `Int`, the result of expanding the macro would be type-checked as if it were on the right-hand side of

```swift
let _: (Int, String) = <macro expansion result>
```

Macro expansion expressions can occur within the arguments to a macro. For example, consider:

```swift
#addBlocker(#stringify(1 + 2))
```

The first phase of the macro type-check does not perform any macro expansion: the macro expansion expression `#stringify(1 + 2)` will infer that its `T` is `Int`, and will produce a value of type `(Int, String)`. The `addBlocker` macro expansion expression will infer that its `T` is `(Int, String)`, and the result is the same.

The second phase of macro expansions occurs outside-in. First, the `addBlocker` macro is expanded, to `#prohibitBinaryOperators(#stringify(1 + 2), operators: ["+"])`. Then, the `prohibitBinaryOperators` macro is expanded given those (textual) arguments. The expansion result it produces will be type-checked, which will end up type-checking `#stringify(1 + 2)` again and, finally, expanding `#stringify(1 + 2)`.

From an implementation perspective, the compiler reserves the right to avoid performing repeated type checking of the same macro arguments. For example, we type-checked `#stringify(1 + 2)` in the first phase of the expansion of `prohibitBinaryOperators`, and then again on the expanded result. When the compiler recognizes that the same syntax node is being re-used unmodified, it can re-use the types computed in the first phase. This is an important performance optimization for the type checker.

Macro expansion cannot be recursive: if the expansion of a given macro produces source code that expands that same macro, the program is ill-formed. This prevents unbounded macro expansion.

With the exception of the built-in macro declarations for source locations (e.g., `#fileID`, `#line`), a macro cannot be used as the default argument of a parameter. The existing features for source locations have special behavior when they appear as a default argument, wherein they are expanded by the caller using the source-location information at the call site rather than in the function declaration where they appear. This is useful, existing behavior that we cannot change, but it might not make sense for all macros, and could be surprising. Therefore, we prohibit such default argument that are (non-built-in) macros to avoid confusion, and are open to revisiting this restriction in the future.

### Macro implementation library

Macro definitions will make use of the [swift-syntax](https://github.com/apple/swift-syntax) package, which provides the Swift syntax tree manipulation and parsing capabilities for Swift tools. The `SwiftSyntaxMacros` module will provide the functionality required to define macros.

#### `Macro` protocols

The `Macro` protocol is the root protocol for all kinds of macro definitions. At present, it does not have any requirements:

```swift
public protocol Macro { }
```

All "freestanding" macros conform to the `FreestandingMacro` protocol:

```swift
public protocol FreestandingMacro: Macro { }
```

The `ExpressionMacro` protocol is used to describe expression macros, and is a form of freestanding macro:

```swift
public protocol ExpressionMacro: FreestandingMacro {
  /// Expand a macro described by the given freestanding macro expansion syntax node
  /// within the given context to produce a replacement expression.
  static func expansion(
    of node: some FreestandingMacroExpansionSyntax,
    in context: some MacroExpansionContext
  ) async throws -> ExprSyntax
}
```

The `FreestandingMacroExpansionSyntax` protocol is the `swift-syntax` node describing the `macro-expansion-expression` grammar term from above, so it carries the complete syntax tree (including all whitespace and comments) of the macro expansion as it appears in the source code. 

Macro definitions should conform to the `ExpressionMacro` protocol and implement their syntactic transformation via `expansion(of:in:)`, returning the new expression as a syntax node.

If the macro expansion cannot proceed for some reason, the `expansion(of:in:)` operation can throw an error rather than try to produce a new syntax node. The compiler will then report the error to the user. More detailed diagnostics can be provided via the macro expansion context.

The macro expansion operation is asynchronous, to account for potentially-asynchronous operations that will eventually be added to `MacroExpansionContext`. For example, operations that require additional communication with the compiler to get types of subexpressions, access files in the program, and so on.

#### `MacroExpansionContext`

The macro expansion context provides additional information about the environment in which the macro is being expanded. This context can be queried as part of the macro expansion:

```swift
/// Protocol whose conforming types provide information about the context in
/// which a given macro is being expanded.
public protocol MacroExpansionContext: AnyObject {
   /// Generate a unique name for use in the macro.
  public func makeUniqueName(_ name: String) -> TokenSyntax

  /// Emit a diagnostic (i.e., warning or error) that indicates a problem with the macro
  /// expansion.
  public func diagnose(_ diagnostic: Diagnostic)

  /// Retrieve a source location for the given syntax node.
  ///
  /// - Parameters:
  ///   - node: The syntax node whose source location to produce.
  ///   - position: The position within the syntax node for the resulting
  ///     location.
  ///   - filePathMode: How the file name contained in the source location is
  ///     formed.
  ///
  /// - Returns: the source location within the given node, or `nil` if the
  ///   given syntax node is not rooted in a source file that the macro
  ///   expansion context knows about.
  func location(
    of node: some SyntaxProtocol,
    at position: PositionInSyntaxNode,
    filePathMode: SourceLocationFilePathMode
  ) -> AbstractSourceLocation?
}
```

The `makeUniqueName()` function allows one to create new, unique names so that the macro expansion can produce new declarations that won't conflict with any other declarations in the same scope. It produces an identifier token containing the unique name, which will also incorporate the `name` identifier for better debuggability. This allows macros to be more hygienic, by not introducing new names that could affect the way that the code provided via macro expansion arguments is type-checked.

It is intended that `MacroExpansionContext` will grow over time to include more information about the build environment in which the macro is being expanded. For example, information about the target platform (such as OS, architecture, and deployment version) and any compile-time definitions passed via `-D`, should be included as part of the context.

The `diagnose` method allows a macro implementation to provide diagnostics as part of macro expansion. The [`Diagnostic`](https://github.com/apple/swift-syntax/blob/main/Sources/SwiftDiagnostics/Diagnostic.swift) type used in the parameter is part of the swift-syntax library, and its form is likely to change over time, but it is able to express the different kinds of diagnostics a compiler or other tool might produce, such as warnings and errors, along with range highlights, Fix-Its, and attached notes to provide more clarity. A macro definition can introduce diagnostics if, for example, the macro argument successfully type-checked but used some Swift syntax that the macro implementation does not understand. The diagnostics will be presented by whatever tool is expanding the macro, such as the compiler. A macro that emits diagnostics is still expected to produce an expansion result unless it also throws an error, in which case both emitted diagnostics and the error will be reported.

The `location` operation allows one to determine source location information for a syntax node. The resulting source location contains the file, line, and column for the corresponding syntax node. The `position` and `filePathMode` can be used to customize the resulting output, e.g., which part of the syntax node to point at and how to render the file name.

```swift
/// Describe the position within a syntax node that can be used to compute
/// source locations.
public enum PositionInSyntaxNode {
  /// Refers to the start of the syntax node's leading trivia, which is
  /// the first source location covered by the syntax node.
  case beforeLeadingTrivia

  /// Refers to the start of the syntax node's first token, which
  /// immediately follows the leading trivia.
  case afterLeadingTrivia

  /// Refers to the end of the syntax node's last token, right before the
  /// trailing trivia.
  case beforeTrailingTrivia

  /// Refers just past the end of the source text that is covered by the
  /// syntax node, after all trailing trivia.
  case afterTrailingTrivia
}

/// Describes how a source location file path will be formed.
public enum SourceLocationFilePathMode {
  /// A file ID consisting of the module name and file name (without full path),
  /// as would be generated by the macro expansion `#fileID`.
  case fileID

  /// A full path name as would be generated by the macro expansion `#filePath`,
  /// e.g., `/home/taylor/alison.swift`.
  case filePath
}
```

Source locations are described in an abstract form that can be interpolated into source code (they are expressions) in places that expect a string literal (for the file name) or integer literal (for line and column). As with `makeUniqueName` returning a `TokenSyntax` rather than a `String`, this abstraction allows the compiler to introduce a different kind of syntax node (that might not even be expressible in normal Swift) to represent these values.

```swift
/// Abstractly represents a source location in the macro.
public struct AbstractSourceLocation {
  /// A primary expression that represents the file and is `ExpressibleByStringLiteral`.
  public let file: ExprSyntax

  /// A primary expression that represents the line and is `ExpressibleByIntegerLiteral`.
  public let line: ExprSyntax

  /// A primary expression that represents the column and is `ExpressibleByIntegerLiteral`.
  public let column: ExprSyntax
}
```

### Macros in the Standard Library

#### `externalMacro` definition

The builtin `externalMacro` macro is declared as follows:

```swift
macro externalMacro<T>(module: String, type: String) -> T
```

The arguments identify the module name and type name of the type that provides an external macro definition. Note that the `externalMacro` macro is special in that it can only be expanded to define another macro. It is an error to use it anywhere else, which is why it does not include an `@freestanding(expression)` attribute.

#### Builtin macro declarations

As previously noted, expression macros use the same leading `#` syntax as a number of built-in expressions like `#line`. With the introduction of expression macros, we propose to subsume those built-in expressions into macros that come as part of the Swift standard library. The actual macro implementations are provided by the compiler, and may even involve things that aren't necessarily implementable with the pure syntactic macro. However, by providing macro declarations we remove special cases from the language and benefit from all of the tooling affordances provided for macros.

We propose to introduce a number of macro declarations into the Swift standard library. There are several different kinds of such macros.

##### Source-location macros

```swift
// File and path-related information
@freestanding(expression) macro fileID<T: ExpressibleByStringLiteral>() -> T
@freestanding(expression) macro file<T: ExpressibleByStringLiteral>() -> T
@freestanding(expression) macro filePath<T: ExpressibleByStringLiteral>() -> T

// Current function
@freestanding(expression) macro function<T: ExpressibleByStringLiteral>() -> T

// Source-location information
@freestanding(expression) macro line<T: ExpressibleByIntegerLiteral>() -> T
@freestanding(expression) macro column<T: ExpressibleByIntegerLiteral>() -> T

// Current shared object handle.
@freestanding(expression) macro dsohandle() -> UnsafeRawPointer
```

The operations that provide information about the current location in source code are mostly implementable as `ExpressionMacro`-conforming types, using the `location` operation on the `MacroExpansionContext`. The exceptions are `#file`, which would need an extension to `MacroExpansionContext` to determine whether we are in a compilation mode where `#file` behaves like `#fileID` vs. behaving like [`#filePath`](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0285-ease-pound-file-transition.md); `dsohandle`, which requires specific compiler support; and `#function`, which would require contextual information that is not available in the `MacroExpansionContext`.

The type signatures of these macros capture most of the type system behavior of the existing `#file`, `#line`, etc., because they are treated like literals and therefore can pick up any contextual type that implements the proper `ExpressibleBy*` protocol. However, the implementations above would fail to type-check code like this:

```swift
let x = #file
```

with an error such as 

```
error: generic parameter 'T' could not be inferred
```

To match the existing behavior of the built-in `#file`, `#line`, etc. would require a defaulting rule that matches what we get for literal types. At present, this requires special handling in the compiler, but a future extension to the language to enable default generic arguments would likely allow us to express this notion directly in the type system.

##### Objective-C helper macros

The Swift `#selector` and `#keyPath` expressions can have their syntax and type-checking behavior expressed in terms of macro declarations:

```swift
@freestanding(expression) macro selector<T>(_ method: T) -> Selector
@freestanding(expression) macro selector<T>(getter property: T) -> Selector
@freestanding(expression) macro selector<T>(setter property: T) -> Selector
@freestanding(expression) macro keyPath<T>(_ property: T) -> String
```

These macros cannot be implemented in terms of `ExpressionMacro` based on the facilities in this proposal, because one would need to determine which declarations are referenced within the argument of a macro expansion such as `#selector(getter: Person.name)`. However, providing them with macro declarations that have built-in implementations makes them less special, removing some special cases from more of the language.

##### Object literals

```swift
@freestanding(expression) macro colorLiteral<T: ExpressibleByColorLiteral>(red: Float, green: Float, blue: Float, alpha: Float) -> T
@freestanding(expression) macro imageLiteral<T: ExpressibleByImageLiteral>(resourceName: String) -> T
@freestanding(expression) macro fileLiteral<T: ExpressibleByFileReferenceLiteral>(resourceName: String) -> T
```

The object literals allow one to reference a resource in a program of various kinds. The three kinds of object literals (color, image, and file) can be described as expression macros. The type signatures provided above are not exactly how type checking currently works for object literals, because they aren't necessarily generic. Rather, when they are used, the compiler currently looks for a specially-named type (e.g., `_ColorLiteralType`) in the current module and uses that as the type of the corresponding color literal. To maintain that behavior, we propose to type-check macro expansions for object literals by performing the same lookup that is done today (e.g., for `_ColorLiteralType`) and then using that type as the generic argument for the corresponding macro. That way, the type checking behavior is unchanged when moving from special object literal expressions in the language to macro declarations with built-in implementations.

### Sandboxing macro implementations

The details of how macro implementation modules are built and provided to the compiler will be left to a separate proposal. However, it's important to call out here that macro implementations will be executed in a sandbox [like SwiftPM plugins](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0303-swiftpm-extensible-build-tools.md#security), preventing file system and network access. This is both a security precaution and a practical way of encouraging macros to not depend on any state other than the specific macro expansion node they are given to expand and its child nodes (but not its parent nodes), and the information specifically provided by the macro expansion context. If in the future macros need access to other information, this will be accomplished by extending the macro expansion context, which also provides a mechanism for the compiler to track what information the macro actually queried.

## Tools for using and developing macros

One of the primary concerns with macros is their ease of use and development: how do we know what a macro does to a program? How does one develop and debug a new macro?

With the right tool support, the syntactic model of macro expansion makes it easy to answer the first question. The tools will need to be able to show the developer what the expansion of any use of a macro is. At a minimum, this should include flags that can be passed to the compiler to expand macros (the prototype provides `-Xfrontend -dump-macro-expansions` for this),  and possibly include a mode to write out a "macro-expanded" source file akin to how C compilers can emit a preprocessed source file. Other tools such as IDEs should be able to show the expansion of a given use of a macro so that developers can inspect what a macro is doing. Because the result is always Swift source code, one can reason about it more easily than (say) inspecting the implementation of a macro that manipulates an AST or IR.

The fact that macro implementations are separate programs actually makes it easier to develop macros. One can write unit tests for a macro implementation that provides the input source code for the macro (say, `#stringify(x + y)`), expands that macro using facilities from swift-syntax, and verifies that the resulting code is free of syntax errors and matches the expected result. Most of the "builtin" macro examples were developed this way in the [syntax macro test file](https://github.com/apple/swift-syntax/blob/main/Tests/SwiftSyntaxMacrosTest/MacroSystemTests.swift).

## Example expression macros

There are many uses for expression macros beyond what has been presented here. This section will collect several examples of macro implementations based on existing built-in `#` expressions as well as ones that come from the Swift forums and other sources of inspiration. Prototype implementations of a number of these macros are [available in the swift-syntax repository](https://github.com/apple/swift-syntax/blob/main/Tests/SwiftSyntaxMacrosTest/MacroSystemTests.swift).

* The `#colorLiteral` macro provides syntax for declaring a color with a given red, green, blue, and alpha values. It can be declared and implemented as follows

  ```swift
  // Declaration of #colorLiteral
  @freestanding(expression) macro colorLiteral(red: Float, green: Float, blue: Float, alpha: Float) -> _ColorLiteralType
    = SwiftBuiltinMacros.ColorLiteralMacro
  
  // Implementation of #colorLiteral
  struct ColorLiteralMacro: ExpressionMacro {
    /// Replace the label of the first element in the tuple with the given
    /// new label.
    func replaceFirstLabel(
      of tuple: TupleExprElementListSyntax, with newLabel: String
    ) -> TupleExprElementListSyntax{
      guard let firstElement = tuple.first else {
        return tuple
      }
  
      return tuple.replacing(
        childAt: 0, with: firstElement.withLabel(.identifier(newLabel)))
    }
   
    static func expansion(
      of node: some FreestandingMacroExpansionSyntax,
      in context: some MacroExpansionContext
    ) -> ExprSyntax {
      let argList = replaceFirstLabel(
        of: node.argumentList, with: "_colorLiteralRed"
      )
      let initSyntax: ExprSyntax = ".init(\(argList))"
      if let leadingTrivia = node.leadingTrivia {
        return MacroResult(initSyntax.withLeadingTrivia(leadingTrivia))
      }
      return initSyntax
    }  
  }
  ```

  The same approach can be used for file and image literals.

* [Power assertions](https://forums.swift.org/t/a-possible-vision-for-macros-in-swift/60900/87) by Kishikawa Katsumi: this assertion macro captures intermediate values within the assertion expression so that when the assertion fails, those values are displayed. The results are exciting!

  ```swift
  #powerAssert(mike.isTeenager && john.age < mike.age)
               |    |          |  |    |   | |    |
               |    true       |  |    42  | |    13
               |               |  |        | Person(name: "Mike", age: 13)
               |               |  |        false
               |               |  Person(name: "John", age: 42)
               |               false
               Person(name: "Mike", age: 13)
  ```

## Source compatibility

Macros are a pure extension to the language, utilizing new syntax, so they don't have an impact on source compatibility.

## Effect on ABI stability

Macros are a source-to-source transformation tool that have no ABI impact.

## Effect on API resilience

Macros are a source-to-source transformation tool that have no effect on API resilience.

## Future Directions

There are a number of potential directions one could take macros, both by providing additional information to the macro implementations themselves and expanding the scope of macros.

### Macro argument type information

The arguments to a macro are fully type-checked before the macro implementation is invoked. However, information produced while performing that type-check is not provided to the macro, which only gets the original source code. In some cases, it would be useful to also have information determined during type checking, such as the types of the arguments and their subexpressions, the full names of the declarations referenced within those expressions, and any implicit conversions performed as part of type checking. For example,  consider a use of a macro like the [power assertions](https://forums.swift.org/t/a-possible-vision-for-macros-in-swift/60900/87) mentioned earlier:

```swift
#assert(Color(parsing: "red") == .red)
```

The implementation would likely want to separate the two operands to `==` into local variables (with fresh names generated by `createUniqueName`) to capture the values, so they can be printed later. For example, the assertion could be translated into code like the following:

```swift
{
  let _unique1 = Color(parsing: "red")
  let _unique2 = .red
  if !(_unique1 == _unique2) {
    fatalError("assertion failed: \(_unique1) != \(_unique2)")
  }
}()
```

Note, however, that this code will not type check, because initializer for `_unique2` requires context information to determine how to resolve `.red`. If the macro implementation were provided with the types of the two subexpressions, `Color(parsing: "red")` and `.red`, it could have been translated into a something that will type-check properly:

```swift
{
  let _unique1: Color = Color(parsing: "red")
  let _unique2: Color = .red
  if !(_unique1 == _unique2) {
    fatalError("assertion failed: \(_unique1) != \(_unique2)")
  }
}()
```

The macro expansion context could be extended with an operation to produce the type of a given syntax node, e.g.,

```swift
extension MacroExpansionContext {
  func type(of node: ExprSyntax) -> Type?
}
```

When given one of the expression syntax nodes that is part of the macro expansion expression, this operation would produce a representation of the type of that expression. The `Type` would need to be able to represent the breadth of the Swift type system, including structural types like tuple and function types, and nominal types like struct, enum, actor, and protocol names.

Additional information could be provided about the actual resolved declarations. For example, the syntax node for `.red` could be queried to produce a full declaration name `Color.red`, and the syntax node for `==` could resolve to the full name of the declaration of the `==` operator that compares two `Color` values. A macro could then distinguish between different `==` operator implementations.

The main complexity of this future direction is in defining the APIs to be used by macro implementations to describe the Swift type system and related information. It would likely be a simplified form of a type checker's internal representation of types, but would need to remain stable. Therefore, while we feel that the addition of type information is a highly valuable extension for expression macros, the scope of the addition means it would best be introduced as a follow-on proposal.

### Additional kinds of macros

Expressions are just one place in the language where macros could be valuable. Other places could include function or closure bodies (e.g., to add tracing or logging), within type or extension definitions (e.g., to add new members), or on protocol conformances (e.g., to synthesize a protocol conformance). A number of potential ideas are presented in the [vision for macros in Swift](https://forums.swift.org/t/a-possible-vision-for-macros-in-swift/60900). For each of them, we assume that the basic `macro` declaration will stay roughly the same, but the contexts in which the macro can be used would be different, as might the spelling of the expansion (e.g., `@` might be more appropriate if the macro expansion occurs on a declaration), there would be an attribute on the `macro` declaration that indicates what type of macro it is, and there would be a corresponding protocol that inherits from `Macro` in the `SwiftSyntaxMacros` module.

## Revision History

* Revisions based on review feedback:
  * Switch `@expression` to `@freestanding(expression)` to align with the other macros proposals and vision document.
  * Make the `ExpressionMacro.expansion(of:in:)` requirement `async`.
  * Allow macro declarations to have opaque result types, and define the uniqueness rules.
  * Simplify the grammar of macro declarations to be more function-like: they always require a parameter list, and if they have a return value, its type is specified following `->`. To account for macros that take no arguments, omitting both an argument list and trailing closures from a macro expansion expression will implicitly add `()`.
  * Make `MacroExpansionContext` a class-bound protocol, because the state involving diagnostics and unique names needs to be shared, and the implementations could vary significantly between (e.g.) the compiler and a test harness.
  * Introduce a general `location` operation on `MacroExpansionContext` to get the source location of any syntax node from a macro input. Remove the `moduleName` and `fileName`, which were always too limited to be useful.
  * Allow macro parameters to have default arguments, with restrictions on what can occur within a default argument.
  * Clarify that macro expansion cannot be recursive.
  * Rename `createUniqueLocalName` to `makeUniqueName`; the names might not always be local in scope. Also add a parameter to it so developers can provide a partial name that will show up in the unique name.
  * Prohibit the use of non-builtin macros as default arguments of parameters.
* Revisions from the second pitch:
  * Moved SwiftPM manifest changes to a separate proposal that can explore the building of macros in depth. This proposal will focus only on the language aspects.
  * Simplified the type signature of the `#externalMacro` built-in macro.
  * Added `@expression` to the macro to distinguish it from other kinds of macros that could come in the future.
  * Make `expansion(of:in:)` throwing, and have that error be reported back to the user.
  * Expand on how the various builtin standard library macros will work.
* Revisions from the first pitch:
  * Rename `MacroEvaluationContext` to `MacroExpansionContext`. 
  * Remove `MacroResult` and instead allow macros to emit diagnostics via the macro expansion context.
  * Remove `sourceLocationConverter` from the macro expansion context; it provides access to the whole source file, which interferes with incremental builds.
  * Rename `ExpressionMacro.apply` to `expansion(of:in)` to make it clear that it's producing the expansion of a syntax node within a given context.
  * Remove the implementations of `#column`, as well as the implication that things like `#line` can be implemented with macros. Based on the above changes, they cannot.
  * Introduce a new section providing declarations of macros for the various `#` expressions that exist in the language, but will be replaced with (built-in) macros.
  * Replace the `external-macro-name` production for defining macros with the more-general `macro-expansion-expression`, and a builtin macro `externalMacro` that makes it far more explicit that we're dealing with external types that are looked up by name. This also provides additional capabilities for defining macros in terms of other macros.
  * Add much more detail about how macro expansion works in practice.
  * Introduce SwiftPM manifest extensions to define macro plugins.
  * Added some future directions and alternatives considered.


## Acknowledgments

Richard Wei implemented the compiler plugin mechanism on which the prototype implementation depends, as well as helping identify and explore additional use cases for macros. John McCall and Becca Royal-Gordon provided numerous insights into the design and practical implementation of macros. Tony Allevato provided additional feedback on building and sandboxing.
