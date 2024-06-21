# Expression macro as caller-side default argument

* Proposal: [SE-0422](0422-caller-side-default-argument-macro-expression.md)
* Authors: [Apollo Zhu](https://github.com/ApolloZhu)
* Review Manager: [Doug Gregor](https://github.com/DougGregor)
* Status: **Implemented (Swift 6.0)**
* Review: ([pitch](https://forums.swift.org/t/pitch-expression-macro-as-caller-side-default-argument/69019)), ([review](https://forums.swift.org/t/se-0422-expression-macro-as-caller-side-default-argument/69730)), ([acceptance](https://forums.swift.org/t/accepted-se-0422-expression-macro-as-caller-side-default-argument/70050))

## Introduction

This proposal aims to lift the restriction afore set in [SE-0382 "Expression macros"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0382-expression-macros.md) to allow non-built-in expression macros as caller-side default argument expressions.

## Motivation

Built-in magic identifiers like [#line](https://developer.apple.com/documentation/swift/line()) and [#fileID](https://developer.apple.com/documentation/swift/fileID()) are documented as expression macros in the official documentation, but if Swift developers try to implement a similar macro themselves and use it as the default argument for some function, the code will not compile:

```swift
public struct MakeLabeledPrinterMacro: ExpressionMacro {
    public static func expansion(
        of node: some FreestandingMacroExpansionSyntax,
        in context: some MacroExpansionContext
    ) throws -> ExprSyntax {
        return "{ value in print(\"\\(#fileID):\\(#line): \\(value)\") }"
    }
}

public macro LabeledPrinter<T>() -> (T) -> Void
= #externalMacro(module: ..., type: "MakeLabeledPrinterMacro")

public func greet<T>(
    _ thing: T,
    print: (T) -> Void = #LabeledPrinter
//                error: ^ non-built-in macro cannot be used as default argument
) {
    print("Hello, \(thing)")
}
```

This is because built-in expression macros/magic identifiers have a special behavior: when used as default arguments, instead of been expanded at where the expressions are written like all other macros, they are expanded by the caller using the source-location information of the call site:

```swift
// in MyLibrary.swift
public func greet<T>(_ thing: T, file: String = #fileID) {
    print("\(fileID): Hello, \(thing)"
}

// in main.swift
greet("World")
// prints "main.swift: Hello, World" instead of "MyLibrary.swift: ...
```

This a useful existing behavior that should be supported, but could be surprising as it differs from all other macro expansions, and might not be desired for all expression macros.

## Proposed solution

The proposal lifts the restriction and makes non-built-in expression macros behave consistently as built-in magic identifier expression macros:

* if expression macros are used as default arguments, they’ll be expanded with caller side source location information and context;
* if they are used as sub-expressions of default arguments, they’ll be expanded at where they are written

```swift
// in MyLibrary.swift =======
@freestanding(expression)
macro MyFileID<T: ExpressibleByStringLiteral>() -> T = ...

public func callSiteFile(_ file: String = #MyFileID) { file }

public func declarationSiteFile(_ file: String = (#MyFileID)) { file }

public func alsoDeclarationSiteFile(
    file: String = callSiteFile(#MyFileID)
) { file }

// in main.swift ============
print(callSiteFile())            // print main.swift, the current file
print(declarationSiteFile())     // always prints MyLibrary.swift
print(alsoDeclarationSiteFile()) // always prints MyLibrary.swift
```

Macro author can inquire the source location information using `context.location(of:)` just like before and implement `#fileID`, `#line`, and `#column` as shown below:

```swift
public struct MyFileIDMacro: ExpressionMacro {
  public static func expansion(
    of node: some FreestandingMacroExpansionSyntax,
    in context: some MacroExpansionContext
  ) -> ExprSyntax {
    context.location(
        of: node, at: .afterLeadingTrivia, filePathMode: .fileID
    )!.file
  }
}

public struct MyLineMacro: ExpressionMacro {
  public static func expansion(
    of node: some FreestandingMacroExpansionSyntax,
    in context: some MacroExpansionContext
  ) -> ExprSyntax {
    context.location(of: node)!.line
  }
}

public struct MyColumnMacro: ExpressionMacro {
  public static func expansion(
    of node: some FreestandingMacroExpansionSyntax,
    in context: some MacroExpansionContext
  ) -> ExprSyntax {
    context.location(of: node)!.column
  }
}
```

## Detailed design

### Type-checking default argument macro expressions

Since the macro expanded expression might reference declarations that are not available in the scope where the function is declared, macro expressions are not expanded at the primary function declaration. However, macro expression used as a default argument is type checked without expansion to make sure that

1. it is at least as visible as the function using it,
2. its return type matches what that parameter expects, and
3. its arguments, if any, are literals without string interpolation.

### Type-checking macro expanded expressions

For each call to a function that has an expression macro default argument, the macro will be expanded with each call-site’s source location and type-checked in the corresponding caller-side context, as if the macro expression is written at where it is expanded:

```swift
@freestanding(expression)
// expands to `foo + bar`
public macro VariableReferences() -> String = ...

public func preferVariablesFromCallerSide(
    param: String = #VariableReferences
) {
    print(param)
}

// in another file ==========
var foo = "hi "
var bar = "caller"
preferVariablesFromCallerSide()  // prints: hi caller
//                           ^ same as #VariableReferences written here
```

## Source compatibility

As non-built-in macro expressions aren’t allowed as default argument, this change is purely additive and has no impact on existing code.

## ABI compatibility

This feature does not affect the ABI.

## Implications on adoption

This feature can be freely adopted and un-adopted in source code with no deployment constraints and without affecting source or ABI compatibility.

## Future directions

### Allow arguments to default argument macro expressions to be arbitrary expressions

If these arguments can be arbitrary expressions, type-checking the macro expression at function declaration will require any declarations referenced in these expressions to be also in scope:

```swift
@freestanding(expression)
// expands to: "Hello " + string
public macro PrependHello(_ string: String) -> String = ...

// this is needed so it can be referenced in the default argument
public var shadowedVariable: String = "World"

public func preferVariablesFromCallerSide(
    param: String = #PrependHello(shadowedVariable)
) {
    print(param)
}
```

However, as the expanded expression is type-checked in the caller-side context, it’s rather unintuitive that one must add the public variable in the example above, yet it might not be what the macro expanded expressions use. For example, if there's a variable with the same name in scope on the caller side, that variable will be used, and the call to the function might fail to type-check:

```swift
// in another file ==========
var shadowedVariable: Int = 42
preferVariablesFromCallerSide()
// #PrependHello(shadowedVariable) expands to "Hello " + 42
// error: binary operator '+' cannot be applied to operands of type 'String' and 'Int'
```

## Alternatives considered

### Expand non-built-in expression macro default arguments at the primary declaration

While this allows all macro expansions to be expanded at where they are written, it creates an inconsistency for expression macros where they behave differently depending on whether they are built-in or not. Therefore, this alternative won’t be a solution for addressing the surprising behavior of built-in expression macros as caller-side default arguments, while the proposed solution unifies, and clarifies how to make expression macro default arguments expand at caller-side vs. at function declaration.

## Acknowledgments

Thanks to Doug Gregor, Richard Wei, and Holly Borla for early feedback and suggestions on design and implementation.
