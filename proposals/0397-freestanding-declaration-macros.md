# Freestanding Declaration Macros

* Proposal: [SE-0397](0397-freestanding-declaration-macros.md)
* Authors: [Doug Gregor](https://github.com/DougGregor), [Richard Wei](https://github.com/rxwei), [Holly Borla](https://github.com/hborla)
* Review Manager: [John McCall](https://github.com/rjmccall)
* Status: **Implemented (Swift 5.9)**
* Vision: [Macros](https://github.com/swiftlang/swift-evolution/blob/main/visions/macros.md)
* Implementation: On `main` behind the experimental flag `FreestandingMacros`
* Review: ([review](https://forums.swift.org/t/se-0397-freestanding-declaration-macros/64655)) ([partial acceptance and second review](https://forums.swift.org/t/se-0397-second-review-freestanding-declaration-macros/64997)) ([acceptance](https://forums.swift.org/t/accepted-se-0397-freestanding-declaration-macros/65167))
* Previous revisions: ([1](https://github.com/swiftlang/swift-evolution/blob/c0f1e6729b6ca1a4fc2367efe68612fde175afe4/proposals/0397-freestanding-declaration-macros.md))

## Introduction

[SE-0382 "Expression macros"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0382-expression-macros.md) introduced macros into Swift. The approach involves an explicit syntax for uses of macros (prefixed by `#`), type checking for macro arguments prior to macro expansion, and macro expansion implemented via separate programs that operate on the syntax tree of the arguments.

This proposal generalizes the `#`-prefixed macro expansion syntax introduced for expression macros to also allow macros to generate declarations, enabling a number of other use cases, including:

* Generating data structures from a template or other data format (e.g., JSON).
* Subsuming the `#warning` and `#error` directives introduced in [SE-0196](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0196-diagnostic-directives.md) as macros.

## Proposed solution

The proposal extends the notion of "freestanding" macros introduced in [SE-0382 "Expression macros"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0382-expression-macros.md) to also allow macros to introduce new declarations. Like expression macros, freestanding declaration macros are expanded using the `#` syntax, and have type-checked macro arguments. However, freestanding declaration macros can be used any place that a declaration is provided, and never produce a value. 

As with other macros, freestanding declaration macros are declared with the `macro` introducer. They will use the `@freestanding` attribute with the new `declaration` role and, optionally, a set of *introduced names* as described in [SE-0389 "Attached macros"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0389-attached-macros.md#specifying-newly-introduced-names). For example, a freestanding declaration macro would have an attribute like this:

```swift
@freestanding(declaration)
```

whereas a freestanding declaration macro that introduced an enum named `CodingKeys` would have an attribute like this:

```swift
@freestanding(declaration, names: named(CodingKeys))
```

Implementations of freestanding declaration macros are types that conform to the `DeclarationMacro` protocol, which is defined as follows:

```swift
public protocol DeclarationMacro: FreestandingMacro {
  static func expansion(
    of node: some FreestandingMacroExpansionSyntax,
    in context: some MacroExpansionContext
  ) throws -> [DeclSyntax]
}
```

Declaration macros can be used anywhere that a declaration is permitted, e.g., in a function or closure body, at the top level, or within a type definition or extension thereof. Declaration macros produce zero or more declarations. The `warning` directive introduced by [SE-0196](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0196-diagnostic-directives.md) can be described as a freestanding declaration macro as follows:

```swift
/// Emits the given message as a warning, as in SE-0196.
@freestanding(declaration) 
macro warning(_ message: String) = #externalMacro(module: "MyMacros", type: "WarningMacro")
```

Given this macro declaration, the syntax

```swift
#warning("unsupported configuration")
```

can be used anywhere a declaration can occur. 

The implementation of a `warning` declaration macro extracts the string literal argument (producing an error if there wasn't one) and emits a warning. It returns an empty list of declarations:

```swift
public struct WarningMacro: DeclarationMacro {
  public static func expansion(
    of node: some FreestandingMacroExpansionSyntax, 
    in context: some MacroExpansionContext
  ) throws -> [DeclSyntax] {
    guard let messageExpr = node.argumentList.first?.expression.as(StringLiteralExprSyntax.self),
          messageExpr.segments.count == 1,
          let firstSegment = messageExpr.segments.first,
          case let .stringSegment(message) = firstSegment else {
      throw SimpleError(node, "warning macro requires a non-interpolated string literal")
    }

    context.diagnose(Diagnostic(node: Syntax(node), message: SimpleDiagnosticMessage(
      message: message.description,
      diagnosticID: .init(domain: "test", id: "error"),
      severity: .warning)))
    return []
  }
}
```

## Detailed design

### Syntax

The syntactic representation of a freestanding macro expansion site is a macro expansion declaration. A macro expansion declaration is described by the following grammar. It is based on the production rule as a [macro expansion expression](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0382-expression-macros.md#macro-expansion), but with the addition of attributes and modifiers:

```
declaration -> macro-expansion-declaration
macro-expansion-declaration -> attributes? declaration-modifiers? '#' identifier generic-argument-clause[opt] function-call-argument-clause[opt] trailing-closures[opt]
```

At top level and function scope where both expressions and declarations are allowed, a freestanding macro expansion site is first parsed as a macro expansion expression. It will be replaced by a macro expansion declaration later during type checking, if the macro resolves to a declaration macro. It is ill-formed if a macro expansion expression resolves to a declaration macro but isn't the outermost expression. This parsing rule is required in case an expression starts with a macro expansion expression, such as in the following infix expression:

```swift
#line + 1
#line as Int?
```

#### Attributes and modifiers

Any attributes and modifiers written on a freestanding macro declaration are implicitly applied to each declaration produced by the macro expansion. For example:

```swift
@available(toasterOS 2.0, *)
public #gyb(
  """
  struct Int${0} { ... }
  struct UInt${0} { ... }
  """,
  [8, 16, 32, 64]
)
```

would expand to:

```swift
@available(toasterOS 2.0, *)
public struct Int8 { ... }

@available(toasterOS 2.0, *)
public struct UInt8 { ... }

@available(toasterOS 2.0, *)
public struct Int16 { ... }

@available(toasterOS 2.0, *)
public struct UInt16 { ... }

@available(toasterOS 2.0, *)
public struct Int32 { ... }

@available(toasterOS 2.0, *)
public struct UInt32 { ... }

@available(toasterOS 2.0, *)
public struct Int64 { ... }

@available(toasterOS 2.0, *)
public struct UInt64 { ... }
```

### Restrictions

Like attached peer macros, a freestanding declaration macro can expand to any declaration that is syntactically and semantically well-formed within the context where the macro is expanded. It shares the same requirements and restrictions:

- [**Specifying newly-introduced names**](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0389-attached-macros.md#specifying-newly-introduced-names)
  - Note that only `named(...)` and `arbitrary` are allowed as macro-introduced names for a declaration macro. `overloaded`, `prefixed`, and `suffixed` do not make sense when there is no declaration from which to derive names.
- [**Visibility of names used and introduced by macros**](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0389-attached-macros.md#visibility-of-names-used-and-introduced-by-macros)
- [**Restrictions on `arbitrary` names**](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0389-attached-macros.md#restrictions-on-arbitrary-names)
- [**Permitted declaration kinds**](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0389-attached-macros.md#permitted-declaration-kinds)

One additional restriction is that a macro declaration can have at most one freestanding macro role. This is because top level and function scopes allow a combination of expressions, statements, and declarations, which would be ambiguous to a freestanding macro expansion with multiple roles.

```swift
@freestanding(expression)
@freestanding(declaration) // error: a macro cannot have multiple freestanding macro roles 
macro foo()
```

### Examples

#### SE-0196 `warning` and `error`

The `#warning` and `#error` directives introduced in [SE-0196](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0196-diagnostic-directives.md): can be implemented directly as declaration macros:

```swift
/// Emit a warning containing the given message.
@freestanding(declaration) macro warning(_ message: String)

/// Emit an error containing the given message
@freestanding(declaration) macro error(_ message: String)
```

### Template code generation

The Swift Standard Library makes extensive use of the [gyb](https://github.com/apple/swift/blob/main/utils/gyb.py) tool to generate boilerplate-y Swift code such as [tgmath.swift.gyb](https://github.com/apple/swift/blob/main/stdlib/public/Platform/tgmath.swift.gyb). The template code is written in `.gyb` files, which are processed by the gyb tool separately before Swift compilation. With freestanding declaration macros, one could write a macro to accept a string as a template and a list of replacement values, allowing templates to be defined inline and eliminating the need to set up a separate build phase.

```swift
@freestanding(declaration, names: arbitrary)
macro gyb(String, [Any]) = #externalMacro(module: "MyMacros", type: "GYBMacro")

#gyb(
  """
  public struct Int${0} { ... }
  public struct UInt${0} { ... }
  """,
  [8, 16, 32, 64]
)
```

This expands to:

```swift
  public struct Int8 { ... }
  public struct UInt8 { ... }

  public struct Int16 { ... }
  public struct UInt16 { ... }

  public struct Int32 { ... }
  public struct UInt32 { ... }

  public struct Int64 { ... }
  public struct UInt64 { ... }
```

### Data model generation

Declaring a data model for an existing textual serialization may need some amount of eyeballing and is prone to errors.  A freestanding declaration macro can be used to analyze a template textual serialization, e.g. JSON, and declare a model data structure against the template.

```swift
@freestanding(declaration, names: arbitrary)
macro jsonModel(String) = #externalMacro(module: "MyMacros", type: "JSONModelMacro")

struct JSONValue: Codable {
  #jsonModel("""
  "name": "Produce",
  "shelves": [
    {
      "name": "Discount Produce",
      "product": {
        "name": "Banana",
        "points": 200,
        "description": "A banana that's perfectly ripe."
      }
    }
  ]
  """)
}
```

This expands to:

```swift
struct JSONValue: Codable {
  var name: String
  var shelves: [Shelves]

  struct Shelves: Codable {
    var name: String
    var product: Product

    struct Product: Codable {
      var description: String
      var name: String
      var points: Double
    }
  }
}
```

## Source compatibility

Freestanding macros use the same syntax introduced for [expression macros](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0382-expression-macros.md), which were themselves a pure extension without an impact on source compatibility. Because a given macro can only have a single freestanding role, and we retain the parsing rules for macro expansion expressions, this proposal introduces no new ambiguities with SE-0382.

## Effect on ABI stability

Macros are a source-to-source transformation tool that have no ABI impact.

## Effect on API resilience

Macros are a source-to-source transformation tool that have no effect on API resilience.

## Alternatives considered

### Multiple freestanding macro roles on a single macro

The proposed feature bans declaring a macro as having multiple freestanding macro roles such as being both `@freestanding(expression)` and `@freestanding(declaration)`.  But such a scenario could be allowed with proper rules.

One possible solution would be to expand such a macro based on its expansion context. If it's being expanded where a declaration is allowed, it will always be expanded as a declaration. Otherwise, it's expanded as an expression.

```swift
@freestanding(expression)
@freestanding(declaration)
macro dualRoleMacro()

// File scope
#dualRoleMacro // expanded as a declaration

func foo() {
  #dualRoleMacro // expanded as a declaration
    
  _ = #dualRoleMacro // expanded as an expression
    
  bar(#dualRoleMacro) // expanded as an expression
    
  takesClosure {
    #dualRoleMacro // expanded as a declaration
  }
}
```

If a future use case deems this feature necessary, this restriction can be lifted following its own proposal.

## Revision History

- Scoped code item macros out as a future direction.  

## Future directions

### Code item macros

A code item macro is another kind of freestanding macro that can produce any mix of declarations, statements, and expressions, which are collectively called "code items" in the grammar. Code item macros can be used for top-level code and within the bodies of functions and closures. They are declaration with `@freestanding(codeItem)`. For example, we could declare a macro that logs when we are entering and exiting a function:

```swift
@freestanding(codeItem) macro logEntryExit(arguments: Any...)
```

Code item macros are implemented as types conforming to the `CodeItemMacro` protocol:

```swift
public protocol CodeItemMacro: FreestandingMacro {
  /// Expand a macro described by the given freestanding macro expansion declaration
  /// within the given context to produce a set of code items, which can be any mix of
  /// expressions, statements, and declarations.
  static func expansion(
    of node: some FreestandingMacroExpansionSyntax,
    in context: some MacroExpansionContext
  ) async throws -> [CodeBlockItemSyntax]
}
```

The `logEntryExit` macro could introduce code such as:

```swift
print("- Entering \(#function)(\(arguments))")
defer {
  print("- Exiting \(#function)(\(arguments))")  
}
```

Code item macros can only introduce new declarations that have unique names, created with `makeUniqueName(_:)`. They cannot introduce named declarations, because doing so affects the ability to type-check without repeatedly expanding the macro with potentially complete information. See the section on the visibility of names used and introduced by macros.

Code item macros are currently under both `FreestandingMacros` and `CodeItemMacros` experimental feature flags.
