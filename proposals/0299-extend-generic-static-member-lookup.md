# Extending Static Member Lookup in Generic Contexts

* Proposal: [SE-0299](0299-extend-generic-static-member-lookup.md)
* Authors: [Pavel Yaskevich](https://github.com/xedin), [Sam Lazarus](https://github.com/sl), [Matt Ricketson](https://github.com/ricketson)
* Review Manager: [Joe Groff](https://github.com/jckarter)
* Status: **Implemented (Swift 5.5)**
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-with-modification-se-0299-extending-static-member-lookup-in-generic-contexts/45238)
* Implementation: [apple/swift#34523](https://github.com/apple/swift/pull/34523)
* Decision Notes: [First return for revision](https://forums.swift.org/t/returned-for-revision-se-0299-extending-static-member-lookup-in-generic-contexts/44466), [First review thread](https://forums.swift.org/t/se-0299-extending-static-member-lookup-in-generic-contexts/43958)
* Previous Revision: [1](https://github.com/swiftlang/swift-evolution/blob/4dd3a9c85195185ab7ad99c468732c5b568d51ac/proposals/0299-extend-generic-static-member-lookup.md)

## Introduction

Using static member declarations to provide semantic names for commonly used values which can then be accessed via leading dot syntax is an important tool in API design, reducing type repetition and improving call-site legibility. Currently, when a parameter is generic, there is no effective way to take advantage of this syntax. This proposal aims to relax restrictions on accessing static members on protocols to afford the same call-site legibility to generic APIs.

Swift-evolution thread: [Extending Static Member Lookup in Generic Contexts](https://forums.swift.org/t/proposal-static-member-lookup-on-protocol-metatypes/41946)

## Motivation

### Background

Today, Swift supports static member lookup on concrete types. For example, SwiftUI extends types like `Font` and `Color` with pre-defined, commonly-used values as static properties:

```swift
extension Font {
  public static let headline: Font
  public static let subheadline: Font
  public static let body: Font
  ...
}

extension Color {
  public static let red: Color
  public static let green: Color
  public static let blue: Color
  ...
}
```

SwiftUI offers view modifiers that accept instances of `Font` and `Color`, including the static values offered above:

```swift
VStack {
  Text(item.title)
    .font(Font.headline)
    .foregroundColor(Color.primary)
  Text(item.subtitle)
    .font(Font.subheadline)
    .foregroundColor(Color.secondary)
}
```

However, this example shows how “fully-qualified” accessors, include the `Font` and `Color` type names when accessing their static properties, are often redundant in context: we know from the `font()` and `foregroundColor()` modifier names that we’re expecting fonts and colors, respectively, so the type names just add unnecessary repetition.

Fortunately, Swift’s static member lookup on concrete types is clever enough that it can infer the base type from context, allowing for the use of enum-like “leading dot syntax” (including a good autocomplete experience). This improves legibility, but without loss of clarity:

```swift
VStack {
  Text(item.title)
    .font(.headline)
    .foregroundColor(.primary)
  Text(item.subtitle)
    .font(.subheadline)
    .foregroundColor(.secondary)
}
```

### The Problem

**Swift static member lookup is not currently supported for members of protocols in generic functions, so there is no way to use leading dot syntax at a generic call site.** For example, SwiftUI defines a `toggleStyle` view modifier like so:

```swift
extension View {
  public func toggleStyle<S: ToggleStyle>(_ style: S) -> some View
}
```

which accepts instances of the `ToggleStyle` protocol, e.g.

```swift
public protocol ToggleStyle {
  associatedtype Body: View
  func makeBody(configuration: Configuration) -> Body
}

public struct DefaultToggleStyle: ToggleStyle { ... }
public struct SwitchToggleStyle: ToggleStyle { ... }
public struct CheckboxToggleStyle: ToggleStyle { ... }
```

Today, SwiftUI apps must write the full name of the concrete conformers to `ToggleStyle` when using the `toggleStyle` modifier:

```swift
Toggle("Wi-Fi", isOn: $isWiFiEnabled)
  .toggleStyle(SwitchToggleStyle())
```

However, this approach has a few downsides:

* **Repetitive:** Only the “Switch” component of the style name is important, since we already know that the modifier expects a type of `ToggleStyle`.
* **Poor discoverability:** There is no autocomplete support to expose the available `ToggleStyle` types to choose from, so you have to know them in advance.

These downsides are impossible to avoid for generic parameters like above, which discourages generalizing functions. API designers should not have to choose between good design and easy-to-read code.

Instead, we could ideally support leading dot syntax for generic types with known protocol conformances, allowing syntax like this:

```swift
Toggle("Wi-Fi", isOn: $isWiFiEnabled)
  .toggleStyle(.switch)
```

### Existing Workarounds

There are ways of achieving the desired syntax today without changing the language, however they are often too complex and too confusing for API clients.

When SwiftUI was still in beta, it included one such workaround in the form of the `StaticMember` type:

```swift
// Rejected SwiftUI APIs:

public protocol ToggleStyle {
  // ...
  typealias Member = StaticMember<Self>
}

extension View {
  public func toggleStyle<S: ToggleStyle>(_ style: S.Member) -> some View
}

public struct StaticMember<Base> {
  public var base: Base
  public init(_ base: Base)
}

extension StaticMember where Base: ToggleStyle {
  public static var `default`: StaticMember<DefaultToggleStyle> { get }
  public static var `switch`: StaticMember<SwitchToggleStyle> { get }
  public static var checkbox: StaticMember<CheckboxToggleStyle> { get }
}

// Leading dot syntax (using rejected workaround):

Toggle("Wi-Fi", isOn: $isWiFiEnabled)
  .toggleStyle(.switch)
```

However, `StaticMember` *serves no purpose* outside of achieving a more ideal syntax elsewhere. Its inclusion is hard to comprehend for anyone looking at the public facing API, as the type itself is decoupled from its actual purpose. SwiftUI removed `StaticMember` before exiting beta for exactly that reason: developers were commonly confused by its existence, declaration complexity, and usage within the framework.

In a [prior pitch](https://forums.swift.org/t/protocol-metatype-extensions-to-better-support-swiftui/25469), [Matthew Johnson](https://forums.swift.org/u/anandabits) rightly called out how framework-specific solutions like `StaticMember` are not ideal: this is a general-purpose problem, which demands a general-purpose solution, not framework-specific solutions like `StaticMember`.

## Proposed solution

We propose *partially* lifting the current limitation placed on referencing of static members from protocol metatypes in order to improve call site ergonomics of the language and make leading dot syntax behave consistently for all possible base types.

More specifically, we propose allowing static members declared in extensions of protocols to be referenced by leading dot syntax if the declaring extension or member itself constrains `Self` to be a concrete type.

The scope of this proposal is limited by design: partially lifting this restriction is an incremental step forward that doesn’t require making significant changes to the implementation of protocols, but also does not foreclose making further improvements in the future such as generally supporting protocol metatype extensions (more on this in *Alternatives Considered*, below).

## Detailed design

The type-checker is able to infer any protocol conformance requirements placed on a particular argument from the call site of a generic function. In our previous example, the `toggleStyle` function requires its argument conform to `ToggleStyle`. Based on that information, the type-checker should be able to resolve a base type for a leading dot syntax argument as a type which conforms to the `ToggleStyle` protocol. It can’t simply use the type `ToggleStyle` because only types conforming to a protocol can provide a witness method to reference. To discover such a type and produce a well-formed reference there are two options:

* Do a global lookup for any type which conforms to the given protocol and use it as a base;
* Require that protocol extension declaring static member(s) or member itself (i.e. generic function/subscript) has 'Self' bound to a concrete type via a same-type generic requirement that would be used to provide a witness for the reference.

The second option is a much better choice that avoids having to do a global lookup and conformance checking and is consistent with the semantics of leading dot syntax, namely, the requirement that result and base types of the chain have to be convertible. This leads to a new rule: if member either binds 'Self' directly (via same-type generic requirement), or is declared in a protocol extension that has `Self` bound to a concrete type, it should be possible to reference such a member on a protocol metatype, using leading dot syntax, by implicitly replacing the protocol with a conforming type referred by `Self`.

This approach works well for references without an explicit base, let’s consider an example:

```swift
// Existing SwiftUI APIs:

public protocol ToggleStyle { ... }

public struct DefaultToggleStyle: ToggleStyle { ... }
public struct SwitchToggleStyle: ToggleStyle { ... }
public struct CheckboxToggleStyle: ToggleStyle { ... }

extension View {
  public func toggleStyle<S: ToggleStyle>(_ style: S) -> some View
}

// Possible SwiftUI APIs:

extension ToggleStyle where Self == DefaultToggleStyle {
  public static var `default`: Self { .init() }
}

extension ToggleStyle where Self == SwitchToggleStyle {
  public static var `switch`: Self { .init() }
}

extension ToggleStyle where Self == CheckboxToggleStyle {
  public static var checkbox: Self { .init() }
}

// Leading dot syntax (using proposed solution):

Toggle("Wi-Fi", isOn: $isWiFiEnabled)
  .toggleStyle(.switch)
```

In the case of `.toggleStyle(.switch)`, the reference to the member `.switch` is re-written to be `SwitchToggleStyle.switch` in the type-checked AST.

Note that declaring members this way pollutes the namespace of each concrete type by creating members like `DefaultToggleStyle.default`, but we believe this is an acceptable trade-off to improve call site ergonomics.

It's also possible to bind `Self` to a type with generic parameters:

```swift
public struct CustomToggleStyle<T>: ToggleStyle {
  ...
}

extension ToggleStyle {
  public static func custom<T>(_: T) -> Self where Self == CustomToggleStyle<T> {
     ...
  }
}

Toggle("Wi-Fi", isOn: $isWiFiEnabled)
   .toggleStyle(.custom(42)) // base type is inferred to be `CustomToggleStyle<Int>` based on the argument type.
```

To make this work the type-checker would attempt to infer protocol conformance requirements from context, e.g. the call site of a generic function (in this case there is only one such requirement - the protocol `ToggleStyle`), and propagate them to the type variable representing the implicit base type of the chain. If there is no other contextual information available, e.g. the result type couldn’t be inferred to some concrete type, the type-checker would attempt to bind base to the type of the inferred protocol requirement. 

Member lookup filtering is adjusted to find static members declared in extension of a protocol metatype. Type-checker would then attempt to find innermost generic signature (either signature of context or itself, if it's some kind of a generic function) and make sure 'Self' parameter of a protocol is bound to a concrete type before accepting the member. When a reference to such a member is considered in expression context, type-checker would replace implicit base type with the concrete type referred by `Self` to form a valid reference to a static member.

## Source compatibility

This is a purely additive change and does not have any effect on source compatibility.



## Effect on ABI stability

This change is frontend only and would not impact ABI.



## Effect on API resilience

This is not an API-level change and would not impact resilience.


## Alternatives considered

### Allow declaring static members directly on protocol metatypes

There have been multiple discussions on this topic on the Swift forums. The most recent one being [a post from Matthew Johnson](https://forums.swift.org/t/protocol-metatype-extensions-to-better-support-swiftui/25469), which suggests adding a special syntax to the language to enable static member references on protocol metatypes. After investigating this direction we determined that supporting this would require significant changes to the implementation of protocols.

Due to its narrow scope, the proposed design is simpler and does not require any syntax changes, while still satisfying all the intended use cases. We stress that this is an incremental improvement, which should not impede our ability to support protocol metatype extensions in the future.

One concrete concern is whether the kind of static member lookup proposed here would be ambiguous with static member lookup on a hypothetical future protocol metatype property. We do not believe it would be, since lookup could be prioritized on the metatype over conforming types. Further, these kinds of namespace and lookup conflicts would likely need to be addressed in a future metatype extension proposal regardless of whether the lookup extension proposed here is accepted or not.

### Allow leading dot syntax for any extensions on protocol metatypes where the return type can be used as the base type

While technically feasible (as the compiler can use the concrete return type as the base type of the expression), this approach leads to the pollution of the protocols namespace. Consider the SwiftUI use case with this approach. The following use case would be valid as the types of the `default`, `switch` and `checkbox` static members all conform to `ToggleStyle`:

```swift
extension ToggleStyle {
  public static var `default`: DefaultToggleStyle { .init() }
  public static var `switch`: SwitchToggleStyle { .init() }
  public static var checkbox: CheckboxToggleStyle { .init() }
}
```

This unfortunately leads to all of the following being valid:

```swift
DefaultToggleStyle.checkbox
SwitchToggleStyle.default
CheckboxToggleStyle.switch
// and so on and so forth
```

# Revision History

The [initial revision of this proposal](https://github.com/swiftlang/swift-evolution/blob/4dd3a9c85195185ab7ad99c468732c5b568d51ac/proposals/0299-extend-generic-static-member-lookup.md)
allowed contextual member lookup to find protocol extension members without
the `Self` requirement, but as noted above, the Core Team rejected this ability
because of the potential for namespace pollution.
