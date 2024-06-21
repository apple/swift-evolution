# Member Macro Conformances

* Proposal: [SE-0407](0407-member-macro-conformances.md)
* Authors: [Doug Gregor](https://github.com/DougGregor)
* Review Manager: [John McCall](https://github.com/rjmccall)
* Status: **Implemented (Swift 5.9.2)**
* Vision: [Macros](https://github.com/swiftlang/swift-evolution/blob/main/visions/macros.md)
* Implementation: [apple/swift#67758](https://github.com/apple/swift/pull/67758)
* Review: ([pitch](https://forums.swift.org/t/pitch-member-macros-that-know-what-conformances-are-missing/66590)) ([review](https://forums.swift.org/t/se-0407-member-macro-conformances/66951)) ([acceptance](https://forums.swift.org/t/accepted-se-0407-member-macro-conformances/67345))

## Introduction

The move from conformance macros to extension macros in [SE-0402](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0402-extension-macros.md) included the ability for extension macros to learn about which protocols the type already conformed to (e.g., because a superclass conformed or an explicit conformance was stated somewhere), so that the macro could avoid adding declarations and conformances that aren't needed. It also meant that any new declarations added are part of an extension---not the original type definition---which is generally beneficial, because it means that (e.g.) a new initializer doesn't suppress the memberwise initializer. It's also usually considered good form to split protocol conformances out into their own extensions.

However, there are some times when the member used for the conformance really needs to be part of the original type definition. For example:

- An initializer in a non-final class needs to be a `required init` to satisfy a protocol requirement.
- An overridable member of a non-final class.
- A stored property or case can only be in the primary type definition.

For these cases, a member macro can produce the declarations. However, member macros aren't provided with any information about which protocol conformances they should provide members for, so a macro might erroneously try to add conforming members to a type that already conforms to the protocol (e.g., through a superclass). This can make certain macros---such as macros that implement the `Encodable` or `Decodable` protocols---unimplemented.

## Proposed solution

To make it possible for a member macro to provide the right members for a conformance, we propose to extend member macros with the same ability that extension macros have to reason about conformances. Specifically:

* The `attached` attribute specifying a `member` role gains the ability to specify the set of protocol conformances it is interested in, the same way an `extension` macro specifies the conformances it can provide.
* The `expansion` operation for a `MemberMacro` -conforming implementation receives the set of protocols that were stated (as above) and which the type does not already conform to.

This information allows a macro to reason about which members it should produce to satisfy conformances. Member macros that are interested in conformances are often going to also be extension macros, which work along with the member macro to provide complete conformance information.

As an example, consider a `Codable` macro that provides the `init(from:)` and `encode(to:)` operations required by the `Decodable` and `Encodable` protocols, respectively. Such a macro could be defined as follows:

```swift
@attached(member, conformances: Decodable, Encodable, names: named(init(from:), encode(to:)))
@attached(extension, conformances: Decodable, Encodable, names: named(init(from:), encode(to:)))
macro Codable() = #externalMacro(module: "MyMacros", type: "CodableMacro")
```

This macro has several important decisions to make about where and how to generate `init(from:)` and `encode(to:)`:

* For a struct, enum, actor, or final class, `init(from:)` and `encode(to:)` should be emitted into an extension (via the member role) along with the conformance. This is both good style and, for structs, ensures that the initializer doesn't inhibit the memberwise initializer.
* For a non-final class, `init(from:)` and `encode(to:)` should be emitted into the main class definition (via the member role) so that they can be overridden by subclasses.
* For a class that inherits `Encodable` or `Decodable` conformances from a superclass, the implementations of `init(from:)` and `encode(to:)` need to call the superclass's initializer and method, respectively, to decode/encode the entire class hierarchy.

Given existing syntactic information about the type (including the presence or absence of `final`), and providing both the member and extension roles  with information about which conformances the type needs (as proposed here), all of the above decisions can be made in the macro implementation, allowing a flexible implementation of a `Codable` macro that accounts for all manner of types.

## Detailed design

The specification of the `conformances` argument for the `@attached(member, ...)` attribute matches that of the corresponding argument for extension macros documented in [SE-0402](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0402-extension-macros.md).  

For macro implementations, the `expansion` requirement in the  `MemberMacro` protocol is augmented with a `conformingTo:` argument that receives the same set of protocols as for extension macros. The `MemberMacro` protocol is now defined as follows:

```swift
protocol MemberMacro: AttachedMacro {
  /// Expand an attached declaration macro to produce a set of members.
  ///
  /// - Parameters:
  ///   - node: The custom attribute describing the attached macro.
  ///   - declaration: The declaration the macro attribute is attached to.
  ///   - missingConformancesTo: The set of protocols that were declared
  ///     in the set of conformances for the macro and to which the declaration
  ///     does not explicitly conform. The member macro itself cannot declare
  ///     conformances to these protocols (only an extension macro can do that),
  ///     but can provide supporting declarations, such as a required
  ///     initializer or stored property, that cannot be written in an
  ///     extension.
  ///   - context: The context in which to perform the macro expansion.
  ///
  /// - Returns: the set of member declarations introduced by this macro, which
  /// are nested inside the `attachedTo` declaration.
  static func expansion(
    of node: AttributeSyntax,
    providingMembersOf declaration: some DeclGroupSyntax,
    conformingTo protocols: [TypeSyntax],
    in context: some MacroExpansionContext
  ) throws -> [DeclSyntax]
}
```

Note that member macro definitions don't provide the conformances themselves; that is still part of the extension macro role.

## Source compatibility

This proposal uses the existing syntactic space for the `@attached` attribute and is a pure extension; it does not have any source compatibility impact.

## ABI compatibility

As a macro feature, this proposal does not affect ABI in any way.

## Implications on adoption

This feature can be freely adopted in source code with no deployment constraints or affecting source or ABI compatibility. Uses of any macro that employs this feature can also be removed from the source code by expanding the macro in place.

## Alternatives considered

### Extensions that affect the primary type definition

A completely different approach to the stated problem would be to introduce a form of extension that adds members to the type as-if they were written directly in the type definition. For example:

```swift
class MyClass { ... }

@implementation extension MyClass: Codable {
  required init(from decoder: Decoder) throws { ... }
  func encode(to coder: Coder) throws { ... }
}
```

The members of `@implementation` extensions would follow the same rules as members in the main type definition. For example, stored properties could be defined in the `@implementation` extension, as could `required` initializers, and any overridable methods, properties, or subscripts. The deinitializer and enum cases could also be defined in `@implementation` extensions if that were deemed useful.

There would be some limitations on `@implementation` extensions: they could only be defined in the same source file as the original type, and these extensions might not be permitted to have any additional generic constraints. Protocols don't have implementations per se, and therefore might not support implementation extensions.

Given the presence of `@implementation` extensions, the extension to member macros in this proposal would no longer be needed, because one could achieve the desired effect using an extension macro that produces an `@implementation` extension for cases where it needs to extend the implementation itself.

The primary drawback to this notion of implementation extensions is that it would no longer be possible to look at the primary definition of a type to find its full "shape": its stored properties, designated/required initializers, overridable methods, and so on.  Instead, that information could be scattered amongst the original type definition and any implementation extensions, requiring readers to stitch together a view of the whole type. This would have a particularly negative effect on macros that want to reason about the shape of the type, because macros only see a single entity (such as a class or struct definition) and not extensions to that entity. The `Codable` macro discussed in this proposal, for example, would not be able to encode or decode stored properties that are written in an implementation extension, and the [`Observable`](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0395-observability.md) macro would silently fail to observe any properties written in an implementation extension. By trying to use implementation extensions to address the shortcoming of macros described in this proposal, we would end up creating a larger problem for those same macros.
