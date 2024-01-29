# [Pitch] Result builder scoped unqualified lookup

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Angela Laar](https://github.com/angela-laar), [Matt Ricketson](https://github.com/ricketson)
* Review Manager: TBD
* Status: **Awaiting implementation**

## Introduction

Result builders provide a foundation for defining declarative DSLs - domain-specific languages offering a customized syntax for working in a specific domain, such as, generating diagrams or text processing. Complex DSL APIs that leverage result builders have encountered issues with design scalability and type-checking performance, introducing a critical challenge to be solved. Extending result builders to support scoped, unqualified name lookup within their bodies, i.e. scoped name spacing for builder-specific types, will enable new API patterns that significantly reduce type-checking complexity while also improving call-site aesthetics.

Previous Swift evolution discussion: https://github.com/apple/swift-evolution/blob/main/proposals/0289-result-builders.md#builder-scoped-name-lookup

## Motivation

Swift libraries can use result builders to define domain-specific languages, which are especially useful for declarative APIs such as regex builders in the Swift Standard Library and view builders in SwiftUI.

Result-builder-based DSLs, as the name suggests, operate on a limited, *domain-specific* set of inputs, typically defined as types conforming to a shared protocol. For example, `@RegexComponentBuilder` composes elements conforming to the `RegexComponent` protocol, whereas SwiftUI’s `@ViewBuilder` composes elements conforming to the `View` protocol. SwiftUI in particular makes extensive use of this pattern of pairing result builders with protocols, not just for views, but also (not limited to):

* Scenes (`@SceneBuilder`/`Scene`)
* Commands (`@CommandsBuilder`/`Commands`)
* Toolbars (`@ToolbarContentBuilder`/`ToolbarContent`)
* Table columns (`@TableColumnBuilder`/`TableColumnContent`)
* Table rows (`@TableRowBuilder`/`TableRowContent`)

DSL components do not typically conform to more than one DSL protocol, forming distinct API families with unique component names. In this SwiftUI example, both the `List` and `Text` types conform to the `View` protocol, but not to any of the other DSL protocols listed above, and so are exclusive to the `@ViewBuilder` DSL:

```
public struct List<Content: View>: View { ... }
public struct Text: View { ... }
```

```
@ViewBuilder var example: some View {
    List {
        Text("Apple")
        Text("Orange")
        Text("Banana")
    }
}
```

DSL types are also typically declared as top-level types, like `List` and `Text` above, optimizing for succinctness and clarity at the point of use. However, this causes different DSLs to declare their component types within the same namespace, increasing the risk of type name collisions.

### **Existing Workarounds**

#### Explicit DSL Namespacing

One approach for avoiding DSL type name collisions is to manually namespace types using DSL-specific prefixes or suffixes. SwiftUI’s `Table` API uses the `Table*` prefix for `TableColumn` and `TableRow`, since `Column` and `Row` are too generic on their own and would likely collide with other DSLs or libraries:

```
Table {
    TableColumn("With 15% tip") { ... }
    TableColumn("With 20% tip") { ... }
    TableColumn("With 25% tip") { ... }
} rows: {
    TableRow(Purchase(price:20))
    TableRow(Purchase(price:50))
    TableRow(Purchase(price:75))
}
```

The obvious downside of this approach is the verbosity and boilerplate of writing the `Table*` prefix on every line. It’s always clear at the call site that we’re building a table, so the prefixes are redundant.

The RegexBuilder module in the Standard Library was created in part due to namespacing concerns. The `Regex` type is currently available in the Standard Library without additional import statements. However, the regex builder DSL types are hidden behind the RegexBuilder module to not pollute the top-level namespace. This is because Regex components like `One`, `OneOrMore` and `Capture` had to be defined as top-level types whereas they only make sense within a regex builder DSL block.

#### Sharing DSL Components

Another approach for avoiding name collisions is to define a single component that is shared across multiple DSLs. This works best when the shared component serves a consistent purpose within each DSL.

There are several examples of shared DSL components in SwiftUI, including `ForEach`, `Group`, and `Section`. For example, you can define sections of views in a `List`:

```
List {
    Section("Dogs") {
        Text("Bulldog")
        Text("Beagle")
        Text("Poodle")
    }
    Section("Cats") {
        Text("Bengal")
        Text("Sphynx")
        Text("Siamese")
    }
}
```

You can also use `Section` to organize rows in a `Table`:

```
Table {
    TableColumn("Breed") { ... }
    TableColumn("Species") { ... }
    TableColumn("Description") { ... }
} rows: {
    Section("Dogs") {
        TableRow(Pet.bulldog)
        TableRow(Pet.beagle)
        TableRow(Pet.poodle)
    }
    Section("Cats") {
        TableRow(Pet.bengal)
        TableRow(Pet.sphynx)
        TableRow(Pet.siamese)
    }
}
```

This approach avoids the need for DSL-specific prefixes, e.g. no `TableSection` type, and helps developers transfer their skills from one domain to another, instead of having to relearn how to implement the same concept in each DSL.

Reusing components across DSLs requires a top-level type that conforms to multiple DSL protocols. Because `Section` is a container type, it uses conditional conformances based on its content type, with each extension defining an equivalent initializer that uses the correct result builder associated with the protocol:

```
// Note: This example is not consistent with SwiftUI's public API, using a
// simplified declaration for illustrative purposes.

public struct Section<Content> {}

extension Section: View where Content: View {
    public init(_ title: String? = nil, @ViewBuilder content: () -> Content)
}

extension Section: TableRowContent where Content: TableRowContent {
    public init(_ title: String? = nil, @TableRowBuilder content: () -> Content)
}
```

While there are clear benefits to this pattern mentioned above, there are also several significant weaknesses:

* Sharing types also means sharing ABI and storage. Some DSLs may share a concept, but require different generic type parameters or different stored properties.
* A new DSL defined in a separate module may also not be able to extend an existing shared type. For example, `RegexBuilder` and `SwiftUI` are independent libraries, so it doesn’t make much sense for both to share a `Group` type if both DSLs wanted a `Group` component.
* Since sharing types is not always an option, name collisions are still possible. Sharing types encourages abstraction, favoring lowest-common-denominator names that are even more likely to collide.  For example, if `RegexBuilder` wanted to add its own `Group` type, clients that import both `SwiftUI` and `RegexBuilder` would need to prefix every use of `Group` with either `SwiftUI.` or `RegexBuilder.`.
* The pattern shown above of overloading interfaces across conditional conformances can severely impact type-checking performance.

The negative impact on type-checking performance, in particular, imposes an effective cap on the scalability of this pattern.

#### Type-Checking Performance for Shared DSL Components

Consider the `Section` declaration example from above. At the call site, each extension’s initializer uses the same trailing-closure syntax:

```
Section {
    SomeContent()
}
```

Since there is no other available hint, the only way to resolve the type of this `Section` instance is to first resolve the type of `SomeContent`. This *bottom-up* type inference means that the type checker must collect and keep track of all the possible outcomes at each level in the expression tree of DSL content, until it resolves the types of the leaf content and can then work its way back up. Because the language allows result builders to be arbitrarily composed, all overloads of the `Section` initializer must be attempted, and overload resolution can only fail once it reaches a leaf component that does not meet the requirements of the given result builder.

In other cases, DSL types must be resolved collectively or top-down. For example, the real `TableRowContent` protocol has a `TableRowValue` associated type, and `@TableRowBuilder` is generic on a row value that all of its rows must share. This means that in reality, `Section`’s conditional conformance to `TableRowContent` has several more constraints than shown above:

```
// Note: This example is not consistent with SwiftUI's public API, using a
// simplified declaration for illustrative purposes.

extension Section: TableRowContent where Content: TableRowContent {
    public typealias TableRowValue = Content.TableRowValue

    public init<V>(
        _ title: String? = nil,
        @TableRowBuilder<V> content: () -> Content
    ) where TableRowValue == V
}
```

These collections of generic constraints can form webs of semi-circular dependencies: a table section’s type depends on its content’s type, but the content’s type depends on the builder’s type, and the builder’s type must be consistent with the section’s type!

The type checker is often able to figure it out, at the cost of increased checking time and memory use, but in some cases fails and must emit an “unable to type-check expression in reasonable time” error. Clients can always work around these errors locally by providing more explicit type information, or by breaking their code into smaller expressions, but this is often a frustrating, trial-and-error-driven process.

Further, while `Section` currently conforms to only two protocols, other lower-level components like `ForEach` and `Group` should theoretically be available in most or all current and future SwiftUI DSLs; `Group` currently conforms to eight DSL protocols, following the same conditional conformance pattern shown above.

Some of these performance issues could be addressed by abandoning shared DSL types and adopting the verbose prefix-based approach instead, e.g. `TableSection`, `TableForEach`, `TableGroup`, etc. However, an ideal solution would support DSL namespacing with unqualified lookup, as well as scalable type-checking performance.

## Proposed solution

This proposal introduces new unqualified name lookup rules that allow unqualified names used inside result builder bodies to find declarations inside the result builder type.

This approach also allows DSL authors to combine builder-scoped names and explicit DSL prefixes on global types to enable concise, scoped component names while still sharing an implementation. For example, `TableRowBuilder` and `TableColumnBuilder` can enable the concise component names `Row` and `Column` in their respective DSLs by introducing a scoped typealias to the long-form name:

```
@resultBuilder 
struct TableColumnBuilder {
  typealias Column = TableColumn 
  ... 
}

@resultBuilder 
struct TableRowBuilder {
  typealias Row = TableRow 
  ... 
}

Table {
   Column("With 15% tip") { ... }
   Column("With 20% tip") { ... }
   Column("With 25% tip") { ... }
 } rows: {
    Section("Dogs") {
        Row(Pet.bulldog)
        Row(Pet.beagle)
        Row(Pet.poodle)
    }
    Section("Cats") {
        Row(Pet.bengal)
        Row(Pet.sphynx)
        Row(Pet.siamese)
    }
 }
```

This approach also allows DSLs to declare concise components that do not make sense to introduce at the top-level. In the below example, the `@HTMLBuilder` type can create names for DSL components without polluting the global namespace with these declarations that are unlikely to be used in other contexts. In the result builder body, at the use-site for the unqualified name `div` , the compiler will search the result builder type scope as if this were a qualified lookup, e.g. `HTMLBuilder.div` . By restricting unqualified names to look within the result builder context, type-checking time will be scaled down significantly because scoped DSL components are only discoverable within the DSL body (without explicit qualification) including in editing tools like code completion.~~

```
protocol HTML { ... }

@resultBuilder
struct HTMLBuilder {
  typealias Component = any HTML
  static func buildBlock(_ component: Component...) -> [Component] { ... }
  
  // Standard HTMLBuilder components
  static func body(@HTMLBuilder _ children: () -> Component) -> Component { ... }
  static func div(@HTMLBuilder _ children: () -> Component) -> Component { ... }
  static func p(@HTMLBuilder _ children: () -> Component) -> Component { ... }
  static func h1(_ text: String) -> Component { ... }
}

@HTMLBuilder
var body: [HTML] {
  div {
    h1("Chatper 1. Loomings.")
    p {
      "Call me Ishmael. Some years ago"
    }
    p {
      "There is now your insular city"
    }
  }
}
```



## Detailed design

Declaring API that can be found via unqualified name lookup in result builders is done by writing the declaration in the scope of the result builder type.

```
@resultBuilder
struct Builder {
  static func buildBlock(_ values: Any...) { ... }
  
  // ScopedValue can be found by unqualified lookup inside
  // @Builder bodies.
  struct ScopedValue { ... }
}
```

Declarations in extensions of the result builder type can also be found via unqualified lookup in a result builder context:

```
extension Builder {
  // AnotherValue can be found by unqualified lookup inside
  // @Builder bodies.
  struct AnotherValue { ... }
}
```

Any declaration that can be nested in a type and accessed with qualified lookup on the result builder type, e.g. `Builder.ScopedValue`, can be found using the unqualified name in result builder context, including:

* Types
* Type aliases
* Static functions
* Static properties

For an unqualified name written in a result builder, name lookup will first look inside the result builder type, e.g. as if the programmer had written `Builder.name`. If a declaration of that component name is found, that result is used and other lexical lookup results will not be considered. If a declaration of that component name is not found, lookup will fall back to lexical lookup. This approach is similar to shadowing; even in cases where a type is declared in the global scope and then again in a result builder type, the innermost scope will take precedence over the outer scope. Note, the outer lookup results will not be considered if picking the shadowed declaration fails to type check:

```
// Standard shadowing

struct S: Equatable {}

let globalS = S()

struct Parent {
  struct S {}

  func shadow() -> Bool {
    S() == globalS // error: Binary operator '==' cannot be applied to operands of type 'Parent.S' and 'S'
  }
}

// Result builder shadowing

protocol Component {}

struct Value: Component {}

@resultBuilder
struct Builder {
  static func buildBlock<C: Component>(_ components: C...) -> [C] {
    return components
  }

  struct Value {}
}

@Builder
var body: [some Component] {
  Value() // error: Static method 'buildBlock' requires that 'Builder.Value' conform to 'Component'
}
```

In cases where the inner result fails to type check, the outer result can still be used by writing a qualified name, e.g. a global declaration `Value` can be qualified with the module name `MyModule` as `MyModule.Value`.

## Source compatibility

This is technically a source breaking change. If a declaration already nested inside a result builder type has the same name as another declaration that can be used inside a body with that result builder applied, existing code using that name inside the result builder body will find the nested declaration:

```
@resultBuilder
struct Builder {
  static func buildBlock(_ values: Any...) { ... }
  
  struct Value { ... }
}

struct Value { ... }

@Builder
var body: Any {
  // Lookup of 'Value' in '@Builder' context will change from the 
  // global 'Value' to 'Builder.Value'
  Value() 
}
```

This could be mitigated by including lexical lookup results in an overload set with the inner results found inside the result builder type. This would alleviate source breakage in the case where the builder-scoped declaration fails to type check when used in the result builder, but name lookup results will still silently change if using the scoped declaration is well-typed.

This lookup behavior is consistent with the current behavior of shadowing in the language; considering outer lexical lookup results can be considered generally as a future direction.

In addition, SwiftUI and Regex do not currently declare any public nested types within their builders, so there will not be any immediate source breakage caused by those APIs if this feature is enabled. Of course, future APIs that add nested types within builders will have to consider the potential source impact. But this is already true (and worse) for new DSL APIs today, which must consider the potential source impact of adding new top-level types.

## Effect on ABI stability

This change has no impact on ABI stability.

## Effect on API resilience

This feature does not add any new API resilience rules. Adding new types or functions in result builder types (with appropriate availability) is a resilient change. Moving existing global types used in result builders inside the result builder is ABI breaking.


