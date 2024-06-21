# A Vision for Macros in Swift

As Swift evolves, it gains new language features and capabilities. There are different categories of features: some fill in gaps, taking existing syntax that is not permitted and giving it a semantics that fit well with the existing language, for example conditional conformance or allowing existential values for protocols with `Self` or associated type requirements. Others introduce new capabilities or paradigms to the language, such as the addition of concurrency, ownership types, or comprehensive reflection. 

There is another large category of language features that provide syntactic sugar to eliminate common boilerplate, taking something that can be written out in long-form and making it more concise. Such features don't technically add any expressive power to the language, because you can always write the long-form version, but their effect can be transformational if it enables use cases that would otherwise have been unwieldy. The synthesis of `Codable` conformances, for example, is sheer boilerplate reduction, but it makes `Codable` support ubiquitous throughout the Swift ecosystem. Property wrappers allow one to factor out logic for property access, and have enabled a breadth of powerful libraries. New language features in this category are hard to evaluate, because there is a fundamental question of whether the feature is "worth it": does the set of use cases made better by this feature outweigh the cost of making the language larger and more complicated?



## Democratizing syntactic sugar with macros

Macros are a feature present in a number of languages that allow one to perform some kind of transformation on the program's input source code to produce a different program. The mechanism of transformation varies greatly, from lexical expansion in C macros, to custom rules that rewrite one syntax into other syntax, to programs that arbitrarily manipulate the abstract syntax tree (AST) of the program. Macro systems exist in C, LISP, Scheme, Scala, Racket, Rust, and a number of other languages, and each design has its own tradeoffs.

In all of these languages, macros have the effect of democratizing syntactic sugar. Many tasks that would have required a new language feature or an external source-generating tool could, instead, be implemented as a macro. Doing so has trade-offs: many more people can implement a macro than can take a feature through the language's evolution process, but the macro implementation will likely have some compromises---non-ideal syntax, worse diagnostics, worse compile-time performance. Overall, the hope is that a macro system can keep the language smaller and more focused, yet remain expressive because it is extensible enough to support libraries for many different domains. As a project, a macro system should reduce the desire for new syntactic-sugar features, leaving more time for more transformative feature work. Even in the cases where a new language feature is warranted, a macro system can allow more experimentation with the feature to best understand how it should work, and then be "promoted" to a full language feature once we've gained experience from the macro version.

### Use cases for macros

There are many use cases for macros, but before we look forward to the new use cases that become possible with macros, let's take a look backward at existing Swift language features that might have been macros had this feature existed before:

* **`Codable`**: What we think of as `Codable` is mostly a library feature, including the `Encodable` and `Decodable` protocols and the various encoding and decoding implementations. The language part of `Codable` is in the synthesis of `init(from:)`, `encode(to:)`, and `CodingKeys` definitions for structs, enums, and classes. A macro that is given information about the stored properties of a type, and the superclass of a class type, could generate the same implementations---and would be easier to implement, improve, and reason about than a bespoke implementation in the compiler. Synthesis for `Equatable`, `Comparable`, and `Hashable` conformances are similar.
* **String interpolation**: String interpolation is implemented as a series of calls into the string interpolation "builder." While the actual parsing of a string interpolation and matching of it to a type that is `ExpressibleByStringInterpolation` is outside the scope of most macro systems, the syntactic transformation into a set of `appendXXX` calls on the builder is something that could be implemented via a macro.
* **Property wrappers**: Property wrappers are integrated into the language syntax via a custom attribute approach (e.g., `@Clamped(0, 100) var percent: Double`), but the actual implementation of the feature is entirely a syntactic transformation that introduces new properties (e.g., the backing storage property `_percent`) and adds accessors to existing properties (e.g., `percent`'s getter becomes `_percent.wrappedValue`). Other built-in language features like `lazy`, `willSet`, and `didSet` use similar syntactic transformations.
* **Result builders**: Result builders are also integrated into the language syntax via a custom attribute, but the actual transformation applied to a closure is entirely syntactic: the compiler introduces calls into the builder's `buildExpression`, `buildBlock`, `buildOptional`, and so on. That syntactic transformation could be expressed via some form of macro.

### When is a language feature better than a macro?

As noted above, a macro system has the potential to replace large parts of existing Swift language features, and enable many new ones. But a macro system is not necessarily a good replacement for a special-built feature:

* A special-built feature might benefit from custom ABI rules.
* A special-built feature might benefit from analyses that would be infeasible to apply in a macro, such as those dependent on data or control flow.
* A special-built feature might be able to offer substantially better diagnostics.
* A special-built feature might be substantially more efficient to apply because it can rely on information and data structures already in the compiler.
* A special-built feature might have capabilities that we need to deny to macros, lest the mere possibility of a macro applying incur massive compile-time costs.

The goal of a macro system should be to be general enough to cover a breadth of potential language features, while still providing decent tooling support and discouraging abuse that makes Swift code hard to reason about.



## Design questions for macros 

At a very high level, a macro takes part of the program's source code at compile time and translates it into other source code that is then compiled into the program. There are three fundamental questions about the use of macros:

* What kind of translation can the macro perform?
* When is a macro expanded?
* How does the compiler expand the macro?

### What kind of translation can the macro perform?

A program's source code goes through several different representations as it is compiled, and a macro system can choose at what point in this translation it operates. We consider three different possibilities:

* **Lexical**: a macro could operate directly on the program text (as a string) or a stream of tokens, and produce a new stream of tokens. The inputs to such a macro would not even have to be valid Swift syntax, which might allow for arbitrary sub-languages to be embedded within a macro. C macros are lexical in nature, and most lexical approaches would inherit the familiar problems of C macros: tooling (such as code completion and syntax highlighting) cannot reason about the inputs to lexical macros, and it's easy for such a macro to produce ill-formed output that results in poor diagnostics.
* **Syntactic**: a macro could operate on a syntax tree and produce a new syntax tree. The inputs to such a macro would be a parsed syntax tree, which is strictly less flexible than a lexical approach because it means the macros can only operate within the bounds of the existing Swift grammar. However, this restriction means that tooling based on the grammar (such as syntax highlighting) would apply to the inputs without having to expand the macro, and macro-using Swift code would follow the basic grammatical structure of Swift. The output of a macro should be a well-formed syntax tree, which will be type-checked by the compiler and integrated into the program.
* **Semantic**: a macro could operate on a type-checked representation of the program, such as an Abstract Syntax Tree (AST) with annotations providing types for expressions, information about which specific declarations are referenced in a function call, any implicit conversions applied to expressions, and so on. Semantic macros have a wealth of additional information that is not provided to lexical or syntactic information, but unlike lexical or syntactic macros, their inputs are restricted to well-typed Swift code. This limits the ability of macros to change the meaning of the code provided to them, which can be viewed both as a negative (less freedom to implement interesting macros) or as a positive (less chance of a macro doing something that confounds the expectations of a Swift programmer). A semantic macro could be required to itself produce a well-typed Abstract Syntax Tree that is incorporated into the program.

Whichever kind of translation we choose, we will need some kind of language or library that is suitable for working with the program at that level. A lexical translation needs to be able to work with program text, whereas a syntactic translation also needs a representation of the program's syntax tree. Semantic translation requires a much larger part of the compiler, including a representation of the type system and the detailed results of fully type-checked code.

### When is a macro expanded?

The expansion of a macro could be initiated in a number of ways, including explicit macro-expansion syntax in the source code or implicit macro expansions that might depend on type checker behavior, e.g., as part of a conversion. The best way for macro expansion to be initiated may depend on the kind of translation that the macro performs: the expansion of a purely lexical or syntactic macro probably needs to be explicitly marked in the source code, because it can change the program structure in surprising ways, whereas a semantic macro might be implicitly expanded as part of type checking because it's working in concert with the type checker.

Swift already has a syntactic pattern that could be used for explicit macro expansion in the form of the `#` prefix, e.g., as a generalization of language features like `#filePath`, `#line`, `#colorLiteral(red: 0.292, green: 0.081, blue: 0.6, alpha: 255)`, and `#warning("unknown platform")`. The general syntax of `#macroName(macro-arguments)` could be used to expand a macro with the given name and arguments. Doing so provides a clear indication of where macros are used, and would support lexical and/or syntactic macros that need to alter the fundamental syntactic structure of their arguments. We refer to macros written with the prefix `#` syntax as *freestanding macros*, because they act as an expression, declaration, or statement on their own, depending on context. For example, one could build a "stringify" macro that produces both its argument value and also a string representation of the argument's source code:

```swift
let (value, code) = #stringify(x + y)  // produces a tuple containing the result of x + y, and the string "x + y"
```

Similarly, Swift's attribute syntax provides an extension point that is already used for features such as property wrappers and result builders. This attribute syntax could be used to expand a macro whose expansion depends on the entity to which the attribute is attached. Therefore, we call these *attached macros*, and they can do things such as create a memberwise initializer for a struct:

```swift
@MemberwiseInit
struct Point {
  var x: Double
  var y: Double
}
```

A `MemberwiseInit` attached macro would need access to the stored properties of the type it is applied to, as well as the ability to create a new declaration `init(x:y:)`. Such a macro would have to tie in to the compiler at an appropriate point where stored properties are known but the set of initializers has not been finalized.

A similar approach could be used to synthesize parts of conformances to a protocol. For example, one could imagine that one could write a declaration whose body is implemented by a macro, e.g.,

```swift
protocol Equatable {
   @SynthesizeEquatable
   func ==(lhs: Self, rhs: Self) -> Bool
}
```

The `@SynthesizeEquatable` attribute could trigger a macro expansion when a particular type that conforms to `Equatable` is missing a suitable implementation of `==`.  It could access the stored properties in the type used for `Self` so it can synthesize an `==` whose body is, e.g., `lhs.x == rhs.x && lhs.y == rhs.y`. 

There are likely many other places where a macro could be expanded, and the key points for any of them are:

* What are the macro arguments and how are they evaluated (tokenized, parsed, type-checked, etc.)?
* What other information is available to macro expansion?
* What can the macro produce (statements, expressions, declarations, attributes, etc.) and how is that incorporated into the resulting program?

### How does the compiler expand the macro?

The prior design sections have focused on what the inputs and outputs of a macro are and where macros can be triggered, but not *how* the macro operates. Again, there are a number of possibilities:

* Macros could use a limited textual expansion mechanism, like the C preprocessor.
* Macros could provide a set of pattern-matching rewrite rules, to identify specific syntax and rewrite it into other syntax, like Rust's [`macro_rules!`](https://doc.rust-lang.org/rust-by-example/macros.html).
* Macros could be arbitrary (Swift) code that manipulates a representation of the program (source code, syntax tree, typed AST, etc.) and produces a new program, like [Scala 3 macros](https://docs.scala-lang.org/scala3/guides/macros/macros.html) , [LISP macros](https://lisp-journey.gitlab.io/blog/common-lisp-macros-by-example-tutorial/), or [Rust procedural macros](https://doc.rust-lang.org/reference/procedural-macros.html). 

These options are ordered roughly in terms of increasing expressive power, where the last is the most flexible because one can write arbitrary Swift code to transform the program. The first two options have the benefit of being able to easily evaluate within the compiler, because they are fundamentally declarative in nature. This means that any tool built on the compiler can show the results of expanding a macro, e.g., within an IDE.

The last option is more complicated, because it involves running arbitrary Swift code. The Swift compiler could conceivably include a complete interpreter for the Swift language, and so long as all of the code that is used in the macro definition is visible to that interpreter (e.g., it does not reference any code for which Swift source is not available), the Swift compiler could interpret the macro definition to produce the expanded result. LISP macros effectively work this way, because LISP is interpreted and can treat the executing program as data.

Alternatively, the macro definition could be compiled separately from the program that uses the macro, for example into a standalone executable or a compiler plugin. The compiler would then invoke that executable or plugin to perform macro expansion each time it is necessary. This approach is taken both by Scala (which uses the JVM's JIT facilities to be able to compile the macro definition and load it into the compiler) and Rust procedural macros (which use a  [`proc-macro`](https://doc.rust-lang.org/reference/linkage.html) crate type for specifically this purpose). A significant benefit of this approach is that the full source code of the macro need not be available as Swift code (so one can use system libraries), macro expansion can be faster (because it's compiled code), and it's easy to test macro definitions outside of the compiler. On the other hand, it means having the Swift compiler run arbitrary code, which opens up questions about security and sandboxing that need to be considered.

### (Un)hygienic macros

A [hygienic macro](https://en.wikipedia.org/wiki/Hygienic_macro#cite_note-hygiene-3) is a macro whose expansion cannot change the binding of names used in its macro arguments. For example, imagine the given use of `myMacro`:

```swift
let x = 3.14159
let y = 2.71828
#myMacro(x + y)
```

The expression `x + y` is type-checked, and `x` and `y` are bound to local variables immediately above. With a hygienic macro, nothing the macro does can change the declarations to which `x` and `y` are bound. A non-hygienic macro could change these bindings. For example, imagine the macro use above expanded to the following:

```swift
{
  let x = 42
  return x + y
}()
```

Here, the macro introduced a new local variable named `x`. With a hygienic macro, the newly-declared `x` is not found by the `x` in `x + y`: it is a different declaration (or it is not permitted to be introduced). With a non-hygienic macro, the `x` in `x + y` will now refer to the local variable introduced by the macro. In this case, the macro expansion for a non-hygienic macro will fail to type-check because one cannot add an `Int` and a `Double`.

Hygienic macros do make some macros harder (or impossible) to write, if the macro intentionally wants to take over some of the names used in its arguments. For example, if one wanted to have a macro intercept access to local variables to (say) record the number of times `x` was dynamically accessed. As such, systems that provide hygienic macros often have a way to intentionally provide names from the environment in which the macro is used, such as Racket's [syntax parameters](https://docs.racket-lang.org/reference/stxparam.html).

A standard approach to dealing with the problem of unintentional name collision in an unhygienic macro is to provide a way to generate unique names within the macro implementation. This approach has been used in LISP macros for decades via [`gensym`](http://clhs.lisp.se/Body/f_gensym.htm), and requires some discipline on the part of the macro implementer to create unique names whenever the macro creates a new declaration.

## An approach to macros in Swift

Based on the menu of design choices above, we propose a macro approach characterized by syntactic translation on already-type-checked Swift code that is implemented via a separate package. The intent here is to allow macros a lot of flexibility to implement interesting transformations, while not giving up the benefits of type-checking the code that the user wrote prior to translation. 

### Macro declarations

A macro declaration indicates how the macro can be used in source code, much like a function declaration indicates the arguments and result type of a function. We declare macros as a new kind of entity, introduced with `macro`,  that indicates that it's a macro definition and provides additional information about the interface to the macro. For example, consider the `stringify` macro described early, which could be defined as follows:

```swift
@freestanding(expression)
macro stringify<T>(_ value: T) -> (T, String) = #externalMacro(module: "MyMacros", type: "StringifyMacro")
```

The `macro` introducer indicates that this is a macro, named `stringify`. The `@freestanding` attribute notes that this is a freestanding macro (used with the `#` syntax) and that it is usable as an expression. The macro is defined (after the `=`) to have an externally-provided macro expansion operation that is the type named `MyMacros.StringifyMacro`.  Because the definition is external, the `stringify` macro function doesn't need a function body. If Swift were to grow a way to implement macros directly here (rather than via a separate package), such macros could have a body but not an `#externalMacro` argument.

A given macro can inhabit several different macro *roles*, each of which can expand in different ways. For example, consider a `Clamping` macro that implements behavior similar to the [property wrapper by the same name](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0258-property-wrappers.md#clamping-a-value-within-bounds):

```swift
@attached(peer, prefixed(_))
@attached(accessor)
macro Clamping<T: Comparable>(min: T, max: T) = #externalMacro(module: "MyMacros", type: "ClampingMacro")
```

The attribute specifies that this is an attached macro, so it can be used as an attribute as, e.g.,

```swift
@Clamping(min: 0, max: 255) var red: Int = 127
```

The `Clamping` macro would be expanded in two different but complementary ways:

* A *peer* declaration `_red` that provides the backing storage:

  ```swift
  private var _red: Int = 127
  ```

* A set of *accessor*s that guard access to this storage, turning the `red` property into a computed property:

  ```swift
  get { _red }
  
  set(__newValue) {
    let __minValue = 0
    let __maxValue = 255
    if __newValue < __minValue {
      _red = __minValue
    } else if __newValue > __maxValue {
      _red = __maxValue
    } else {
      _red = __newValue
    }
  }
  ```


### Macro definitions via a separate program

Macro definitions would be provided in a separate program that performs a syntactic transformation. A macro definition would be implemented using [swift-syntax](https://github.com/apple/swift-syntax), by providing a type that conforms to one of the "macro" protocols in a new library, `SwiftSyntaxMacros`. For example, the `MyMacros` package we're using as an example might look like this:

```swift
import SwiftDiagnostics
import SwiftParser
import SwiftSyntax
import SwiftSyntaxBuilder
import SwiftSyntaxMacros

public struct StringifyMacro: ExpressionMacro {
  static func expansion(
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

Conformance to `ExpressionMacro` indicates a macro definition for an expression macro, and corresponds to `@freestanding(expression)`. There will be several protocols, corresponding to the various roles that macros inhabit. Each protocol has an `expansion` method that will be called with the syntax nodes that are involved in the macro expansion, along with a `context` instance that provides more information about how the macro is being invoked. 

The implementation of these functions makes extensive use of Swift syntax manipulation via the `swift-syntax` package. The inputs and outputs are in terms of syntax nodes: `ExprSyntax` describes the syntax for any kind of expression in Swift, whereas `MacroExpansionExprSyntax` is the syntax for an explicitly-written macro expansion. The `expansion` operation will return a new syntax node that will replace the ones it was given in the program. We use string interpolation as a form of quasi-quoting: the return of `StringifyMacro.expansion` forms a tuple `(\(argument), "\(literal: argument.description)")` where the first argument is the expression itself and the second is the source code translated into a string literal. The resulting string will be parsed into an expression that is returned to the compiler.

Macro implementations are "host" programs that are completely separate from the program in which macros are used. This distinction is most apparent in cross-compilation scenarios, where the host platform (where the compiler is run) differs from the target platform (where the compiled program will run), and these could use different operating systems and processor architectures. Macro implementations are compiled for and executed on the host platform, whereas the results of expanding a macro will be compiled for and executed on the target platform. Therefore, macro implementations are defined as their own kind of target in the Swift package manifest. For example, a package that ties together the macro declaration for `#stringify` and its implementation as `StringifyMacro` follows:

```swift
import PackageDescription
import CompilerPluginSupport

let package = Package(
    name: "MyMacros",
    dependencies: [
      .package(
        url: "https://github.com/apple/swift-syntax.git",
        branch: "main"
      ),
    ],  
    targets: [
        // Macro implementation target contains the StringifyMacro type.
        // Always built for the host platform.
        .macro(name: "StringifyImpl", 
               dependencies: [.product(name: "SwiftSyntaxMacros", package: "swift-syntax")]),

        // Library target provides the macro declaration (public macro stringify) that is
        // used by client code.
        // Built for the target platform.
        .target(name: "StringifyLib", dependencies: ["StringifyImpl"]),

        // Clients of the macro will depend on the library target.
        .executableTarget(name: "StringifyClient", dependencies: ["StringifyLib"]),
    ]
)
```

Conceptually, the separation of `macro` targets into separate programs (for the host platform) from other targets (for the target platform)  means that the individual macros could be built completely separately from each other and from other targets, even if they happen to be for the same platform. In the extreme, this could mean that each `macro` would be allowed to build against a different version of `swift-syntax`, and other targets could choose to also use `swift-syntax` with a different version. Given that `swift-syntax` is modeling the Swift language (which evolves over time), it does not promise a stable API, so having the ability to have different macro implementations depend on different versions of `swift-syntax` is a feature: it would prevent conflicting version requirements in macros from causing problems. 

Note that this separation of dependencies for distinct targets is currently not possible in the Swift Package Manager. In the interim, macro implementations will need to adapt to be built with different versions of the `swift-syntax` package.

#### Diagnostics

A macro implementation can be used to produce diagnostics (e.g., warnings and errors) to indicate problems encountered during macro expansion. The `stringify` macro described above doesn't really have a failure case, but imagine an `#embed("somefile.txt")` macro that takes the contents of a file at build time and turns them into an array of bytes. The macro could have several different failure modes:

* The macro argument isn't a string literal, so it doesn't know what the file name is.
* The file might not be available for reading because it is missing, inaccessible, etc.

These failures would be reported as errors by providing [`Diagnostic`](https://github.com/apple/swift-syntax/blob/main/Sources/SwiftDiagnostics/Diagnostic.swift) instances to the context that specify the underlying problem. The diagnostics would refer to the syntax nodes provided to the macro definition, and the compiler would provide those diagnostics to the user.

In its limit, a macro might perform no translation whatsoever on the syntax tree it is given, but instead be there only to provide diagnostics---for example, as a context-specific, custom lint-like rule that enforces additional constraints on the program.

### Macro roles

The `@freestanding` and `@attached` attributes for macro declarations specify the roles that the macro can inhabit, each of which corresponds to a different place in the source code where the macro can be expanded. Here is a potential set of roles where macro expansion could be warranted. The set of roles could certainly grow over time to enable new capabilities in the language:

* **Expression**: A freestanding macro that can occur anywhere that an expression can occur, and must produce an expression. `#colorLiteral` could fall into this category:

  ```swift
  // In library
  @freestanding(expression)
  macro colorLiteral(red: Double, green: Double, blue: Double, alpha: Double) -> _ColorLiteralType =
    #externalMacro(module: "MyMacros", type: "ColorLiteral")
  
  // In macro definition package
  public struct ColorLiteral: ExpressionMacro {
    public static func expansion(
      expansion: MacroExpansionExprSyntax, 
      in context: MacroExpansionContext
    ) -> ExprSyntax {
      return ".init(\(expansion.argumentList))"
    }
  }
  ```

  With this, an expression like `#colorLiteral(red: 0.5, green: 0.5, blue: 0.25, alpha: 1.0)` would produce a value of the `_ColorLiteralType` (presumably defined by a framework), and would be rewritten by the macro into `.init(red: 0.5, green: 0.5, blue: 0.25, alpha: 1.0)` and type-checked with the `_ColorLiteralType` as context so it would initialize a value of that type.

* **Declaration**: A freestanding macro that can occur anywhere a declaration can occur, such as at the top level, in the definition of a type or extension thereof, or in a function or closure body. The macro can expand to zero or more declarations. These macros could be used to subsume the `#warning` and `#error` directives from [SE-0196](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0196-diagnostic-directives.md):

  ```swift
  /// Emits the given message as a warning, as in SE-0196.
  @freestanding(declaration) macro warning(_ message: String)
  
  /// Emits the given message as an error, as in SE-0196.
  @freestanding(declaration) macro error(_ message: String)
  ```

* **Code item**: A freestanding macro that can occur within a function or closure body and can produce a mix of zero or more statements, expressions, and declarations. 

* **Accessor**: An attached macro that adds accessors to a stored property or subscript, as shown by the `Clamping` macro example earlier. The inputs would be the arguments provided to the macro in the attribute, along with the property or subscript declaration to which the accessors will be attached.  The output would be a set of accessor declarations, i.e., a getter and setter. A `Clamping` macro could be implemented as follows:

  ```swift
  extension ClampingMacro: AccessorMacro {
    static func expansion(
      of node: CustomAttributeSyntax,
      providingAccessorsOf declaration: DeclSyntax,
      in context: MacroExpansionContext
    ) throws -> [AccessorDeclSyntax] {
      let originalName = /* get from declaration */, 
          minValue = /* get from custom attribute node */,
          maxValue = /* get from custom attribute node */
      let storageName = "_\(originalName)",
          newValueName = context.getUniqueName(),
          maxValueName = context.getUniqueName(),
          minValueName = context.getUniqueName()
      return [
        """
        get { \(storageName) }
        """,
        """
        set(\(newValueName)) {
          let \(minValueName) = \(minValue)
          let \(maxValueName) = \(maxValue)
          if \(newValueName) < \(minValueName) {
            \(storageName) = \(minValueName)
          } else if \(newValueName) > \(maxValueName) {
            \(storageName) = \(maxValueName)
          } else {
            \(storageName) = \(newValueName)
          }
        }
        """
      ]
    }  
  }
  ```

* **Witness**: An attached macro that can be expanded to provide a "witness" that satisfies a requirement of a protocol for a given concrete type's conformance to that protocol. Such a macro would take as input the conforming type, the protocol, and a declaration (without a body) that will be created in the conforming type. The output would be that declaration with a body added and (potentially) other modifications. For this to work well, we would almost certainly need to expose a lot of information about the conforming type, such as the set of stored properties. Assuming that exists, let's implement the `synthesizeEquatable` macro referenced earlier in this document:

  ```swift
  // In the standard library
  @attached(witness)
  macro SynthesizeEquatable() = #externalMacro(module: "MyMacros", type: "EquatableSynthesis")
  
  protocol Equatable {
    @SynthesizeEquatable
    static func ==(lhs: Self, rhs: Self) -> Bool
  }
  
  // In the macro definition library
  struct EquatableSynthesis: AttachedMacro {
    /// Expand a macro described by the given custom attribute to
    /// produce a witness definition for the requirement to which
    /// the attribute is attached.
    static func expansion(
      of node: CustomAttributeSyntax,
      witness: DeclSyntax,
      conformingType: TypeSyntax,
      storedProperties: [StoredProperty],
      in context: MacroExpansionContext
    ) throws -> DeclSyntax {
      let comparisons: [ExprSyntax] = storedProperties.map { property in 
        "lhs.\(property.name) == rhs.\(property.name)"
      }
      let comparisonExpr: ExprSyntax = comparisons.map { $0.description }.joined(separator: " && ")
      return witness.withBody(
        """
        {
          return \(comparisonExpr)
        }
        """
      )
    }
  }
  ```

* **Member**: An attached macro that can be applied on a type or extension that expands to one or more declarations that will be inserted as members into that type or extension. As with a conformance macro, a member macro would probably want access to the stored properties of the enclosing type, and potentially other information. As an example, let's create a macro to synthesize a memberwise initializer:

  ```swift
  // In the standard library
  @attached(member)
  macro memberwiseInit(access: Access = .public) = #externalMacro(module: "MyMacros", type: "MemberwiseInit")
  
  // In the macro definition library
  struct MemberwiseInit: MemberMacro {
    static func expansion(
      of node: AttributeSyntax,
      attachedTo declaration: DeclSyntax,
      in context: inout MacroExpansionContext
    ) throws -> [DeclSyntax] {}
      let parameters: [FunctionParameterSyntax] = declaration.storedProperties.map { property in 
        let paramDecl: FunctionParameterSyntax = "\(property.name): \(property.type)"                                                                         
        guard let initializer = property.initializer else {
          return paramDecl
        }
        return paramDecl.withDefaultArgument(
          InitializerClauseSyntax(
            equal: TokenSyntax(.equal, presence: .present),
            value: "\(initializer)"
          )
        )
      }
      
      let assignments: [ExprSyntax] = conformingType.storedProperties.map { property in
        "self.\(property.name) = \(property.name)"
      }
  
      return
        #"""
        public init(\(parameters.map { $0.description }.joined(separator: ", "))) {
          \(assignments.map { $0.description }.joined(separator: "\n"))
        }
        """#
    }
  }
  ```

  Using this macro on a type, e.g.,

  ```swift
  @MemberwiseInit
  class Point {
    var x, y: Int
    var z: Int = 0 
  }
  ```

  would produce code like the following:

  ```swift
  public init(x: Int, y: Int, z: Int = 0) {
    self.x = x
    self.y = y
    self.z = z
  }
  ```

* **Body**: A body macro would allow one to create or replace the body of a function, initializer, or closure through syntactic manipulation. Body macros are attached to one of these entities, e.g.,

  ```swift
  @Traced(logLevel: 2)
  func myFunction(a: Int, b: Int) { ... }
  ```

  where the `Traced` macro is declared as something like:

  ```swift
  @attached(body) macro Traced(logLevel: Int = 0)
  ```

  and can introduce new code into the body to, e.g., perform logging.

* **Conformance**: Conformance macros could introduce protocol conformances to the type or extension to which they are attached. For example, this could be useful when composed with macro roles that create other members, such as a macro that both adds a protocol conformance and also a stored property required by that conformance. 

## Tools for using and developing macros

Macros introduce novel problems for tooling, because the macro expansion process replaces (or augments) code that is explicitly written with other source code that makes it into the final program. The design of a macro system has a large impact on the ability to build good tools, and a poor design can directly impact discoverability, predictability, debuggability, and compile-time efficiency. C macros demonstrate nearly all of these problems:

* C macros are bare identifiers that can be used anywhere in source code, so it is hard to discover where macros are being applied in the source code. C programmers have adopted the `UPPERCASE_MACRO_NAME` convention to try to understand which names are macros and which aren't.
* C macros can expand to an arbitrary sequence of tokens in a manner that destroys program structure, for example, one can close a `struct` or function definition with a C macro, making it hard to predict the scope of effects a macro can have.
* C macros are expanded via logic within the compiler's preprocessor, and therefore offer no debugging capabilities. The only way to see the effect of a macro is to generate preprocessed output for an entire translate unit, then inspect the original code.
* C macros are rarely persisted after a program is built, so debugging a program that has made heavy use of macros requires one to manually map between the original source code (pre-macro) and the generated machine code, with no record of the expansion itself.

The design proposed here for Swift makes it possible to build good tooling despite the challenges macros pose:

* Uses of macros are indicated in the source (with `#` or `@`) to make the use of macros clear.
* Expansions of macros have their effects restricted to the scope in which the macro is used (e.g., augmenting or adding declarations locally), and any effects visible from other parts of the program are declared up front by the macro (e.g., the names it introduces), so one can reason about the effects of a macro expansion.
* Implementations of macros are normal Swift programs, so they can be developed and tested using the normal tools for Swift. Much of the development and testing of a macro can be done outside of the compiler, with unit tests that (for example) test the syntactic transformation on isolated examples that translate Swift code into different Swift code.
* The localized nature of macro effects, and the fact that all macro-expanded code is itself normal Swift code, make it possible to record the results of macro expansion in a way that can reconstitute the effects of macro expansion without rerunning the compiler, allowing useful debugging and diagnostics flows.

Early implementations of Swift macros already provide macro-expansion information in a manner that is amenable to existing tooling. For example, the result of expanding a macro fits into several existing workflows:

* If a warning or error message refers into code generated by macro expansion, the compiler writes the macro-expanded code into a separate Swift file and refers to that file within the diagnostic message. Users can open that file to see the results of that macro expansion to understand the problem. Each such diagnostic provides a stack of notes that refers back to the point where the macro expansion occurred, which may itself be within other macro-expanded code, all the way back to the original source code that triggered the outermost macro expansion. For example:

  ```
  /tmp/swift-generated-sources/@__swiftmacro_9MacroUser14testAddBlocker1a1b1c2oaySi_S2iAA8OnlyAddsVtF03addE0fMf1_.swift:1:4: error: binary operator '-' cannot be applied to two 'OnlyAdds' operands
  oa - oa
  ~~ ^ ~~
  macro_expand.swift:200:7: note: in expansion of macro 'addBlocker' here
    _ = #addBlocker(oa + oa)
        ^~~~~~~~~~~~~~~~~~~~
  ```

* Debug information for macro-expanded code uses a similar scheme to diagnostics, allowing one to see the macro-expanded code while debugging (e.g., in a backtrace), set breakpoints in it, step through it, and so on. Freestanding macros are treated like inline functions, so one can "step into" a macro expansion from the place it occurs in the source code.

* SourceKit, which is used for IDE integration features, provides a new "Expand Macro" refactoring that can expand any single use of a macro right in the source code. This can be used both to see the effects of the macro, as well as to eliminate the use of the macro in favor of (say) a customized version where the macro is used as a starting point.

* Macro implementations can be tested using `swift-syntax` and existing testing tooling. For example, the following test checks the expansion behavior of the `stringify` macro by performing syntactic expansion on the input source code (`sf`) and checking that against the expected result of expansion (in the `XCTAssertEqual` at the end):

  ```swift
  final class MyMacroTests: XCTestCase {
    func testStringify() {
      let sf: SourceFileSyntax =
        #"""
        let a = #stringify(x + y)
        let b = #stringify("Hello, \(name)")
        """#
      let context = BasicMacroExpansionContext.init(
        sourceFiles: [sf: .init(moduleName: "MyModule", fullFilePath: "test.swift")]
      )
      let transformedSF = sf.expand(macros: ["stringify": StringifyMacro.self], in: context)
      XCTAssertEqual(
        transformedSF.description,
        #"""
        let a = (x + y, "x + y")
        let b = ("Hello, \(name)", #""Hello, \(name)""#)
        """#
      )
    }
  }
  ```

All of the above work with existing tooling, creating a baseline development experience that provides discoverability, predictability, and debuggability. Over time, more tooling can be made aware of macros to provide a more polished experience.
