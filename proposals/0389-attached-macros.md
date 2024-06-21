# Attached Macros

* Proposal: [SE-0389](0389-attached-macros.md)
* Authors: [Doug Gregor](https://github.com/DougGregor), [Holly Borla](https://github.com/hborla), [Richard Wei](https://github.com/rxwei)
* Review Manager: [Tony Allevato](https://github.com/allevato)
* Status: **Implemented (Swift 5.9)**
* Implementation: Implemented on GitHub `main` behind the experimental flag `Macros`. See the [example repository](https://github.com/DougGregor/swift-macro-examples) for more macros.
* Review: ([pitch #1, under the name "declaration macros"](https://forums.swift.org/t/pitch-declaration-macros/62373)) ([pitch #2](https://forums.swift.org/t/pitch-attached-macros/62812)) ([review](https://forums.swift.org/t/se-0389-attached-macros/63165)) ([acceptance](https://forums.swift.org/t/accepted-se-0389-attached-macros/63593))

## Introduction

Attached macros provide a way to extend Swift by creating and extending declarations based on arbitrary syntactic transformations on their arguments. They make it possible to extend Swift in ways that were only previously possible by introducing new language features, helping developers build more expressive libraries and eliminate extraneous boilerplate.

Attached macros are one part of the [vision for macros in Swift](https://github.com/swiftlang/swift-evolution/pull/1927), which lays out general motivation for introducing macros into the language. They build on the ideas and motivation of [SE-0382 "Expression macros"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0382-expression-macros.md) to cover a large new set of use cases; we will refer to that proposal for the basic model of how macros integrate into the language. While expression macros are designed as standalone entities introduced by `#`, attached macros are associated with a specific declaration in the program that they can augment and extend. This supports many new use cases, greatly expanding the expressiveness of the macro system:

* Creating trampoline or wrapper functions, such as automatically creating a completion-handler version of an `async` function or vice-versa.
* Creating members of a type based on its definition, such as forming an [`OptionSet`](https://developer.apple.com/documentation/swift/optionset) from an enum containing flags and conforming it to the `OptionSet` protocol or adding a memberwise initializer.
* Creating accessors for a stored property or subscript, subsuming some of the behavior of [SE-0258 "Property Wrappers"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0258-property-wrappers.md).
* Augmenting members of a type with a new attribute, such as applying a property wrapper to all stored properties of a type.

There is an [example repository](https://github.com/DougGregor/swift-macro-examples) containing a number of macros that have been implemented using the prototype of this feature.

## Proposed solution

The proposal adds *attached macros*, so-called because they are attached to a particular declaration. They are written using the custom attribute syntax (e.g., `@AddCompletionHandler`) that already provides extensibility for declarations through property wrappers, result builders, and global actors. Attached macros can reason about the declaration to which they are attached, and provide additions and changes based on one or more different macro *roles*. Each role has a specific purpose, such as adding members, creating accessors, or adding peers alongside the declaration. A given attached macro can inhabit several different roles, and as such will be expanded multiple times corresponding to the different roles, which allows the various roles to be composed. For example, an attached macro emulating property wrappers might inhabit both the "peer" and "accessor" roles, allowing it to introduce a backing storage  property and also synthesize a getter/setter that go through that backing storage property. Composition of macro roles will be discussed in more depth once the basic macro roles have been established.

As with expression macros, attached declaration macros are declared with `macro`, and have [type-checked macro arguments](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0382-expression-macros.md#type-checked-macro-arguments-and-results) that allow their behavior to be customized. Attached macros are identified with the `@attached` attribute, which also provides the specific role as well as any names they introduce. For example, the aforemented macro to add a completion handler would be declared as follows:

```swift
@attached(peer, names: overloaded)
macro AddCompletionHandler(parameterName: String = "completionHandler")
```

The macro can be used as follows:

```swift
@AddCompletionHandler(parameterName: "onCompletion")
func fetchAvatar(_ username: String) async -> Image? { ... }
```

The use of the macro is attached to `fetchAvatar`, and generates a *peer* declaration alongside `fetchAvatar` whose name is "overloaded" with `fetchAvatar`. The generated declaration is:

```swift
/// Expansion of the macro produces the following.
func fetchAvatar(_ username: String, onCompletion: @escaping (Image?) -> Void) {
  Task.detached {
    onCompletion(await fetchAvatar(username))
  }
}
```

### Implementing attached macros

All attached macros are implemented as types that conform to one of the protocols that inherits from the `AttachedMacro` protocol.  Like the [`Macro` protocol](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0382-expression-macros.md#macro-protocols), the `AttachedMacro` protocol has no requirements, but is used to organize macro implementations. Each attached macro role will have its own protocol that inherits `AttachedMacro`. 

```swift
public protocol AttachedMacro: Macro { }
```

The biggest difference from expression macros is that there is more than one relevant piece of syntax for the macro to consider: each attached macro implementation receives the syntax node for both the attribute (e.g., `@AddCompletionHandler(parameterName: "onCompletion")`) and the declaration to which the macro is attached (`func fetchAvatar`...), and can return new code that's appropriate to the macro role: peer macros return new declarations, accessor macros return getters/getters, and so on. For example, `PeerMacro` is defined as follows:

```swift
public PeerMacro: AttachedMacro {
  /// Expand a macro described by the given attribute to
  /// produce "peer" declarations of the declaration to which it
  /// is attached.
  ///
  /// The macro expansion can introduce "peer" declarations that 
  /// go alongside the given declaration.
  static func expansion(
    of node: AttributeSyntax,
    providingPeersOf declaration: some DeclSyntaxProtocol,
    in context: some MacroExpansionContext
  ) async throws -> [DeclSyntax]
}
```

### Naming macro-produced declarations

Unlike expression macros, attached macros can introduce new declarations. These declarations can have an impact on code elsewhere in the program, for example if a macro provides a declaration of a function named `hello` and that function is called from another source file. Our design requires macros to document which names they can introduce: this provides more information up-front to developers and tools alike to understand the impact that a macro can have on the surrounding program. For developers, this can mean fewer surprises; for tools, this can be used to improve compilation times by avoiding unnecessary macro expansions.

The `@AddCompletionHandler` macro notes that it introduces an *overloaded* name, meaning that it produces a declaration with the same base name as the declaration to which it is attached. A macro that emulated a property wrapper would specify the storage name via `prefixed(_)`, meaning that `_` will be added as a prefix to the name of the declaration to which that macro is attached. Other ways in which macro-generated names are communicated are discussed in the Detailed Design.

### Kinds of attached macros

#### Peer macros

Peer macros produce new declarations alongside the declaration to which they are attached.  The `AddCompletionHandler` macro from earlier was a peer macro. Peer macros are implemented via types that conform to the `PeerMacro` protocol shown earlier. The implementation of `AddCompletionHandlerMacro` looks like the following:

```swift
public struct AddCompletionHandlerMacro: PeerDeclarationMacro {
  public static func expansion(
    of node: CustomAttributeSyntax,
    providingPeersOf declaration: some DeclSyntaxProtocol,
    in context: some MacroExpansionContext
  ) throws -> [DeclSyntax] {
    // make sure we have an async function to start with
    // form a new function "completionHandlerFunc" by starting with that async function and
    //   - remove async
    //   - remove result type
    //   - add a completion-handler parameter
    //   - add a body that forwards arguments
    // return the new peer function
    return [completionHandlerFunc]
  }
}
```

The details of the implementation are left to an Appendix, with a complete version in the [example repository](https://github.com/DougGregor/swift-macro-examples).

#### Member macros

Member macros allow one to introduce new members into the type or extension to which the macro is attached. For example, we can write a macro that defines static members to ease the definition of an [`OptionSet`](https://developer.apple.com/documentation/swift/optionset). Given:

```swift
@OptionSetMembers
struct MyOptions: OptionSet {
  enum Option: Int {
    case a
    case b
    case c
  }
}
```

This struct should be expanded to contain both a `rawValue` field and static properties for each of the options, e.g.,

```swift
// Expands to...
struct MyOptions: OptionSet {
  enum Option: Int {
    case a
    case b
    case c
  }
  
  // Synthesized code below...
  var rawValue: Int = 0
  
  static var a = MyOptions(rawValue: 1 << Option.a.rawValue)
  static var b = MyOptions(rawValue: 1 << Option.b.rawValue)
  static var c = MyOptions(rawValue: 1 << Option.c.rawValue)
}
```

The macro itself will be declared as a member macro that defines an arbitrary set of members:

```swift
/// Create the necessary members to turn a struct into an option set.
@attached(member, names: named(rawValue), arbitrary) macro OptionSetMembers()
```

The `member` role specifies that this macro will be defining new members of the declaration to which it is attached. In this case, while the macro knows it will define a member named `rawValue`, there is no way for the macro to predict the names of the static properties it is defining, so it also specifies `arbitrary` to indicate that it will introduce members with arbitrarily-determined names.

Member macros are implemented with types that conform to the `MemberMacro` protocol:

```swift
protocol MemberMacro: AttachedMacro {
  /// Expand a macro described by the given attribute to
  /// produce additional members of the given declaration to which
  /// the attribute is attached.
  static func expansion(
    of node: AttributeSyntax,
    providingMembersOf declaration: some DeclGroupSyntax,
    in context: some MacroExpansionContext
  ) async throws -> [DeclSyntax]
}
```

#### Accessor macros

Accessor macros allow a macro to add accessors to a property, turning a stored property into a computed property. For example, consider a macro that can be applied to a stored property to instead access a dictionary keyed by the property name. Such a macro could be used like this:

```swift
struct MyStruct {
  var storage: [AnyHashable: Any] = [:]
  
  @DictionaryStorage
  var name: String
  
  @DictionaryStorage(key: "birth_date")
  var birthDate: Date?
}
```

The `DictionaryStorage` attached macro would alter the properties of `MyStruct` as follows, adding accessors to each:

```swift
struct MyStruct {
  var storage: [String: Any] = [:]
  
  var name: String {
    get { 
      storage["name"]! as! String
    }
    
    set {
      storage["name"] = newValue
    }
  }
  
  var birthDate: Date? {
    get {
      storage["birth_date"] as Date?
    }
    
    set {
      if let newValue {
        storage["birth_date"] = newValue
      } else {
        storage.removeValue(forKey: "birth_date")
      }
    }
  }
}
```

The macro can be declared as follows:

```swift
@attached(accessor) macro DictionaryStorage(key: String? = nil)
```

Implementations of accessor macros conform to the `AccessorMacro` protocol, which is defined as follows:

```swift
protocol AccessorMacro: AttachedMacro {
  /// Expand a macro described by the given attribute to
  /// produce accessors for the given declaration to which
  /// the attribute is attached.
  static func expansion(
    of node: AttributeSyntax,
    providingAccessorsOf declaration: some DeclSyntaxProtocol,
    in context: some MacroExpansionContext
  ) async throws -> [AccessorDeclSyntax]
}
```

The implementation of the `DictionaryStorage` macro would create the accessor declarations shown above, using either the `key` argument (if present) or deriving the key name from the property name. The effect of this macro isn't something that can be done with a property wrapper, because the property wrapper wouldn't have access to `self.storage`.

An accessor macro can specify that it produces observers by listing at least one of `willSet` or `didSet ` in the names, e.g., `@attached(accessors, names: named(willSet))`. Such a macro can only produce observers; it cannot change a stored property into a computed property.

The expansion of an accessor macro that does not specify one of `willSet` or `didSet` in its list of names must result in a computed property. A side effect of the expansion is to remove any initializer from the stored property itself; it is up to the implementation of the accessor macro to either diagnose the presence of the initializer (if it cannot be used) or incorporate it in the result.

#### Member attribute macros

Member declaration macros allow one to introduce new member declarations within the type or extension to which they apply. In contrast, member *attribute* macros allow one to modify the member declarations that were explicitly written within the type or extension by adding new attributes to them. Those new attributes could then refer to other macros, such as peer or accessor macros, or builtin attributes. As such, they are primarily a means of *composition*, since they have fairly little effect on their own.

Member attribute macros allow a macro to provide similar behavior to how many built-in attributes work, where declaring the attribute on a type or extension will apply that attribute to each of the members. For example, a global actor `@MainActor` written on an extension applies to each of the members of that extension (unless they specify another global actor or `nonisolated`), an access specifier on an extension applies to each of the members of that extension, and so on.

As an example, we'll define a macro that partially mimics the behavior of the `@objcMembers` attributes introduced long ago in [SE-0160](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0160-objc-inference.md#re-enabling-objc-inference-within-a-class-hierarchy). Our `myObjCMembers` macro is a member-attribute macro:

```swift
@attached(memberAttribute)
macro MyObjCMembers()
```

The implementation of this macro will add the `@objc` attribute to every member of the type or extension, unless that member either already has an `@objc` macro or has `@nonobjc` on it. For example, this:

```swift
@MyObjCMembers extension MyClass {
  func f() { }

  var answer: Int { 42 }

  @objc(doG) func g() { }

  @nonobjc func h() { }
}
```

would result in:

```swift
extension MyClass {
  @objc func f() { }

  @objc var answer: Int { 42 }

  @objc(doG) func g() { }

  @nonobjc func h() { }
}
```

Member-attribute macro implementations should conform to the `MemberAttributeMacro` protocol:

```swift
protocol MemberAttributeMacro: AttachedMacro {
  /// Expand a macro described by the given custom attribute to
  /// produce additional attributes for the members of the type.
  static func expansion(
    of node: AttributeSyntax,
    attachedTo declaration: some DeclGroupSyntax,
    providingAttributesOf member: some DeclSyntaxProtocol,
    in context: some MacroExpansionContext
  ) async throws -> [AttributeSyntax]
}
```

The `expansion` operation accepts the attribute syntax `node` for the spelling of the member attribute macro and the declaration to which that attribute is attached (i.e., the type or extension). The implementation of the macro can walk the members of the `declaration` to determine which members require additional attributes. The returned dictionary will map from those members to the additional attributes that should be added to each of the members.

#### Conformance macros

Conformance macros allow one to introduce new protocol conformances to a type. This would often be paired with other macros whose purpose is to help satisfy the protocol conformance. For example, one could imagine an extended version of the `OptionSetMembers` attributed shown earlier that also adds the `OptionSet` conformance. With it, the mimimal implementation of an option set could be:

```swift
@OptionSet
struct MyOptions {
  enum Option: Int {
    case a
    case b
    case c
  }
}
```

where `OptionSet` is both a member and a conformance macro, providing members (as in `OptionSetMembers`) and the conformance to `OptionSet`. The macro would be declared as, e.g.,

```swift
/// Create the necessary members to turn a struct into an option set.
@attached(member, names: named(rawValue), arbitrary)
@attached(conformance)
macro OptionSet()
```

Conformance macro implementations should conform to the `ConformanceMacro` protocol:

```swift
/// Describes a macro that can add conformances to the declaration it's
/// attached to.
public protocol ConformanceMacro: AttachedMacro {
  /// Expand an attached conformance macro to produce a set of conformances.
  ///
  /// - Parameters:
  ///   - node: The custom attribute describing the attached macro.
  ///   - declaration: The declaration the macro attribute is attached to.
  ///   - context: The context in which to perform the macro expansion.
  ///
  /// - Returns: the set of `(type, where-clause?)` pairs that each provide the
  ///   protocol type to which the declared type conforms, along with
  ///   an optional where clause.
  static func expansion(
    of node: AttributeSyntax,
    providingConformancesOf declaration: some DeclGroupSyntax,
    in context: some MacroExpansionContext
  ) throws -> [(TypeSyntax, WhereClauseSyntax?)]
}
```

The returned array contains the conformances. The `TypeSyntax` describes the protocol to which the enclosing type conforms, and the optional `where` clause provides any additional constraints that would make the conformance conditional.

## Detailed design

### Composing macro roles

A given macro can have several different roles, allowing the various macro features to be composed. Each of the roles is considered independently, so a single use of a macro in source code can result in different macro expansion functions being called. These calls are independent, and could even happen concurrently. As an example, let's define a macro that emulates property wrappers fairly closely.  The property wrappers proposal has an example for a [clamping property wrapper](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0258-property-wrappers.md#clamping-a-value-within-bounds):

```swift
@propertyWrapper
struct Clamping<V: Comparable> {
  var value: V
  let min: V
  let max: V

  init(wrappedValue: V, min: V, max: V) {
    value = wrappedValue
    self.min = min
    self.max = max
    assert(value >= min && value <= max)
  }

  var wrappedValue: V {
    get { return value }
    set {
      if newValue < min {
        value = min
      } else if newValue > max {
        value = max
      } else {
        value = newValue
      }
    }
  }
}

struct Color {
  @Clamping(min: 0, max: 255) var red: Int = 127
  @Clamping(min: 0, max: 255) var green: Int = 127
  @Clamping(min: 0, max: 255) var blue: Int = 127
  @Clamping(min: 0, max: 255) var alpha: Int = 255
}
```

Instead, let's implement this as a macro:

```swift
@attached(peer, prefixed(_))
@attached(accessor)
macro Clamping<T: Comparable>(min: T, max: T) = #externalMacro(module: "MyMacros", type: "ClampingMacro")
```

The usage syntax is the same in both cases. As a macro, `Clamping` both defines a peer (a backing storage property with an `_` prefix) and also defines accessors (to check min/max).  The peer declaration macro is responsible for defining the backing storage, e.g.,

```swift
private var _red: Int = {
  let newValue = 127
  let minValue = 0
  let maxValue = 255
  if newValue < minValue {
    return minValue
  }
  if newValue > maxValue {
    return maxValue
  }
  return newValue
}()
```

Which is implemented by having `ClampingMacro` conform to `PeerMacro`:

```swift
enum ClampingMacro { }

extension ClampingMacro: PeerDeclarationMacro {
  static func expansion(
    of node: CustomAttributeSyntax,
    providingPeersOf declaration: DeclSyntax,
    in context: some MacroExpansionContext
  ) throws -> [DeclSyntax] {
    // create a new variable declaration that is the same as the original, but...
    //   - prepend an underscore to the name
    //   - make it private
  }
}
```

And introduces accessors such as

```swift
    get { return _red }

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

by conforming to `AccessorMacro`:

```swift
extension ClampingMacro: AccessorMacro {
  static func expansion(
    of node: CustomAttributeSyntax,
    providingAccessorsOf declaration: some DeclSyntaxProtocol,
    in context: some MacroExpansionContext
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
        } else if \(newValueName) > maxValue {
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

This formulation of `@Clamping` offers some benefits over the property-wrapper version: we don't need to store the min and max values as part of the backing storage (so the presence of `@Clamping` doesn't add any storage), nor do we need to define a new type. More importantly, it demonstrates how the composition of different macro roles together can produce interesting effects.

Not every macro role applies to every kind of declaration: an `accessor` macro doesn't make sense of a function, nor does a `member` member make sense on a property. If a macro is attached to a declaration, the macro will only be expanded for those roles that are applicable to a declaration. For example, if the `Clamping` macro is applied to a function, it will only be expanded as a peer macro; it's role as an accessor macro is ignored. If a macro is attached to a declaration and none of its roles apply to that kind of declaration, the program is ill-formed.

### Specifying newly-introduced names

Whenever a macro produces declarations that are visible to other Swift code, it is required to declare the names in advance. All of the names need to be specified within the attribute declaring the macro role, using the following forms:

- Declarations with a specific fixed name: `named(<declaration-name>)`.
- Declarations whose names cannot be described statically, for example because they are derived from other inputs: `arbitrary`.

* Declarations that have the same base name as the declaration to which the macro is attached, and are therefore overloaded with it: `overloaded`.
* Declarations whose name is formed by adding a prefix to the name of the declaration to which the macro is attached: `prefixed(_)`. As a special consideration, `$` is permissible as a prefix, allowing macros to produce names with a leading `$` that are derived from the name of the declaration to which the macro is attached. This carve-out enables macros that behavior similarly to property wrappers that introduce a projected value.
* Declarations whose name is formed by adding a suffix to the name of the declaration to which the macro is attached: `suffixed(_docinfo)`. 

A  macro can only introduce new declarations whose names are covered by the kinds provided, or have their names generated via `MacroExpansionContext.createUniqueName`. This ensures that, in most cases (where `arbitrary` is not specified), the Swift compiler and related tools can reason about the set of names that will be introduced by a given use of a macro without having to expand the macro, which can reduce the compile-time cost of macros and improve incremental builds. For example, when the compiler performs name lookup for a name `_x`, it can avoid expanding any macros that are unable to produce a declaration named `_x`. Macros that can produce arbitrary names must always be expanded, as would a macro with `prefixed(_)` that is attached to a declaration named `x`.

The macro is not required to provide a declaration for every name it describes: for example, `OptionSetMembers` will state that it produces a declaration named `rawValue`, but the implementation may opt not to do so if it sees that such a property already exists.

### Visibility of names used and introduced by macros

Macros can introduce new names into the program. Whether and when those names are visible to other parts of the program, and in particular other macros, is a subtle design problem that has a number of interesting constraints.

First, the arguments to a macro are type-checked before it can be determined which macro is being expanded, so they cannot depend on the expansion of that macro. For example, consider an attached macro use spelled `@myMacro(x)`: if it introduced a declaration named `x`, it would potentially invalidate the type-check of its own argument, or cause that macro expansion to choose a different `myMacro` that doesn't produce an `x`!

Second, if the output of one macro expansion (say, `@myMacro1(x)`) introduces a name (say, `y`) that is then used in the argument to another macro expansion (say, `@myMacro2(y)`), then the order in which the macros are expanded can affect the semantics of the resulting program. It is not generally possible to introduce an ordering on macro expansions, nor would it be desirable, because Swift generally tries not to have order dependencies among declarations in a program. Incidental orderings based on the names introduced by the macro don't help, either, because names of declaration macros can be specified as `arbitrary` and therefore cannot be predicated.

Third, macro expansion is a relatively expensive compile-time operation, involving serialization overhead to transfer parts of the program from the compiler/IDE to another program that expands the macro. Therefore, macros are [only expanded once per use site](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0382-expression-macros.md#macro-expansion), so their expansions cannot participate in the type checking of their arguments or of other surrounding statements or expressions. For example, consider a this (intentionally obtuse) Swift code:

```swift
@attached(peer) macro myMacro(_: Int)
@attached(peer, names: named(y)) macro myMacro(_: Double)

func f(_: Int, body: (Int) -> Void)
func f(_: Double, body: (Double) -> Void)

f(1) { x in
  @myMacro(x) func g() { }
  print(y)
}
```

The macro use `@myMacro(x)` provides different names depending on whether the argument is an `Int` or ` Double`. From the perspective of the call to `f`, both overloads of `f` are possible, and the only way to check beyond that macro use to the `print` expression that follows is to try to expand the macro... for the first `myMacro`, the `print(y)` expression will fail to type-check (there is no `y`) whereas the second would find the `y` generated by the second `myMacro` and will succeed. However, this approach does not scale, because it could involve expanding macros a large number of times during type checking. Moreover, the macro expansions might end up getting incomplete information if, for example, macros are someday provided with type information and that type information isn't known yet (because the expansion is happening during type inference).

To address these issues, a name introduced by a macro expansion is not visible within macro arguments for macros the same scope as the macro expansion or any of its enclosing scopes. This is conceptually similar to [outside-in expansion of expression macros](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0382-expression-macros.md#macro-expansion), where one can imagine that all of the macros at a particular scope are type-checked and expanded simultaneously at the top-level scope. Then, macros within each type definition and its extensions are type-checked and expanded simultaneously: they can see the names introduced by top-level macro expansions, but not names introduced at other levels. The following annotated code example shows which names are visible:

```swift
import OtherModule // declares names x, y, z

@macro1(x) // uses OtherModule.x, introduces name y
func f() { }

@macro2(y) // uses OtherModule.y, introduce name x
func g() { }

struct S1 {
  @macro3(x) // uses the name x introduced by #@acro2, introduces the name z
  func h()
}

extension S1 {
  @macro4(z) // uses OtherModule.z
  func g() { }
  
  func f() {
    print(z) // uses z introduced by @macro3
  }
}
```

These name lookup rules, while important for providing a reasonable scheme for macros to be expanded without interfering with each other, do introduce a new kind of name shadowing to the language. If `@macro2` were manually expanded into source code, the declaration `x` it produces would then become visible to the macro argument in `@macro1(x)`, changing the behavior of the program. The compiler might be able to detect such shadowing in practice, when a macro-introduced name at a particular scope would change the meaning of a lookup from that particular scope.

Within function and closure bodies, the fact that names introduced by the macro expansion are not visible within the current scope means that our earlier example will never find a declaration `y` introduced by `@myMacro`:

```swift
f(1) { x in
  @myMacro(x) func g() { }
  print(y)     // does not consider any names introduced by `@myMacro`
}
```

Therefore, a macro used within a closure or function body can only introduce declarations using names produced by `createUniqueName`. This maintains the [two-phase of checking macros](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0382-expression-macros.md#macro-expansion) where type checking and inference is performed without expanding the macro, then the macro is expanded and its result type-checked independently, with no ability to influence type inference further.

### Restrictions on `arbitrary` names

Attached macros that specify `arbitrary` names require expanding the macro in order to determine the exact set of names that the macro generates. This means that name lookup must expand all macros that can produce `arbitrary` names in the given scope in order to determine name lookup results. This is a problem for macros that introduce names at global scope, because any unqualified or module qualified lookup must expand those macros. However, because macro attributes can have arguments that are type-checked prior to macro expansion, type checking those arguments may require unqualified or module qualified name lookup that requires expanding that same macro, which is fundamentally circular. Though macro arguments do not have visibility of macro-generated names in the same scope, macro argument type checking can invoke type checking of other declarations that can use macro-generated names. For example:

```swift
@attached(peer, names: arbitrary)
macro IntroduceArbitraryPeers<T>(_: T) = #externalMacro(...)

@IntroduceArbitraryPeers(MyType().x)
struct S {}

struct MyType {
  var x: AnotherType
}
```

Resolving the macro attribute `@IntroduceArbitraryPeers(MyType().x)` can invoke type-checking of the `var x: AnotherType` property of the `MyType` struct. Unqualified lookup of `AnotherType` must expand `@IntroduceArbitraryPeers` because the macro can introduce arbitrary names; it might introduce a top-level type called `AnotherType`. Resolving the `@IntroduceArbitraryPeers` attribute depends on type checking the `var x: AnotherType` property, and type checking the property depends on expanding the `@IntroduceArbitraryPeers` macro, which results in a circular reference error.

Because `arbitrary` names introduced at global scope are extremely prone to circularity, peer macros attached to top-level declarations cannot introduce `arbitrary` names.


### Ordering of macro expansions

When multiple macros are applied to a single declaration, the order in which macros are expanded could have a significant impact on the resulting program if the the outputs of one macro expansion are treated as inputs to the others. This form of macro composition could be fairly powerful, but it also can have a number of surprising side effects:

1. The order in which attributes are written on a declaration could affect the result.
2. There is no obvious single source of truth for what the inputs to the macro would be.
3. Independently-developed macros could conflict in ways that break the program.

The fine-grained nature of attached macros means that many attached macros have independent effects, e.g., macros can generate distinct members, conformances, and peers. We consider this independence of different macros applying to the same definition to be a feature, because it makes the application of macros more predictable as well as enabling more efficient implementations. 

To ensure the independence of macro invocations, each macro expansion is provided with syntax nodes as they were originally written, and not updated with the effects of other macros. For example, consider the `OptionSet` macro earlier:

```swift
@OptionSet
struct MyOptions {
  enum Option: Int {
    case a
    case b
    case c
  }
} 
```

The `OptionSet` macro is both a member macro (which generates the instance property `rawValue` as well as static properties `a`, `b`, and `c`) and a conformance macro (to add the `OptionSet` conformance). Two different `expansion` functions will be called on the option set macro implementation: `expansion(of:providingMembersOf:in:)` and `expansion(of:providingConformancesOf:in:)`, respectively. In both cases, the syntax tree for `MyOptions` is passed as the middle argument, and both functions provide a result that augments the definition of `MyOptions`, either by adding members or by adding a conformance.

In both cases, the expansion operation is provided with the original definition of `MyOptions`, without the new conformance or added members. That way, each expansion operation operates independently of the other---whether it's different roles for the same macro, or different macros entirely---and the order of expansion does not matter.

### Permitted declaration kinds

A macro can expand to any declaration that is syntactically and semantically well-formed within the context where the macro is expanded, with a few notable exceptions:

* `import` declarations can never be produced by a macro. Swift tooling depends on the ability to resolve import declarations based on a simple scan of the original source files. Allowing a macro expansion to introduce an import declaration would complicate import resolution considerably.
* A type with the `@main` attribute cannot be produced by a macro. Swift tooling should be able to determine the presence of a main entry point in a source file based on a syntactic scan of the source file without expanding macros.
* `extension` declarations can never be produced by a macro. The effect of an extension declaration is wide-ranging, with the ability to add conformances, members, and so on. These capabilities are meant to be introduced in a more fine-grained manner.
* `operator` and `precedencegroup` declarations can never be produced by a macro, because they could allow one to reshape the precedence graph for existing code causing subtle differences in the semantics of code that sees the macro expansion vs. code that does not.
* `macro` declarations can never be produced by a macro, because allowing this would allow a macro to trivially produce infinitely recursive macro expansion.
* Top-level default literal type overrides, including `IntegerLiteralType`,
  `FloatLiteralType`, `BooleanLiteralType`,
  `ExtendedGraphemeClusterLiteralType`, `UnicodeScalarLiteralType`, and
  `StringLiteralType`, can never be produced by a macro.

## Source compatibility

Attached declaration macros use the same syntax introduced for custom attributes (such as property wrappers), and therefore do not have an impact on source compatibility.

## Effect on ABI stability

Macros are a source-to-source transformation tool that have no ABI impact.

## Effect on API resilience

Macros are a source-to-source transformation tool that have no effect on API resilience.

## Alternatives considered

### Mutating declarations rather than augmenting them

Attached declaration macro implementations are provided with information about the macro expansion itself and the declaration to which they are attached. The implementation cannot directly make changes to the syntax of the declaration to which it is attached; rather, it must specify the additions or changes by packaging them in `AttachedDeclarationExpansion`. 

An alternative approach would be to allow the macro implementation to directly alter the declaration to which the macro is attached. This would provide a macro implementation with greater flexibility to affect the declarations it is attached to. However, it means that the resulting declaration could vary drastically from what the user wrote, and would preventing the compiler from making any determinations about what the declaration means before expanding the macro. This could have detrimental effects on compile-time performance (one cannot determine anything about a declaration until the macros have been run on it) and might also prevent macros from accessing information about the actual declaration in the future, such as the types of parameters.

It might be possible to provide a macro implementation API that is expressed in terms of mutation on the original declaration, but restrict the permissible changes to those that can be handled by the implementation. For example, one could "diff" the syntax tree provided to the macro implementation and the syntax tree produced by the macro implementation to identify changes, and return those changes that are acceptable to the compiler.

## Revision History

* After the first pitch:
  * Added conformance macros, to produce conformances
  * Moved the discussion of macro-introduced names from the freestanding macros proposal here.
  * Added a carve-out to allow a `$` prefix on names generated from macros, allowing them to match the behavior of property wrappers.
  * Revisited the design around the ordering of macro expansions, forcing them to be independent.
  * Clarify when accessor macros need to produce observers for a stored property vs. turning it into a computed property.
  * Moved the discussion of "permitted declaration kinds" from the freestanding macros proposal here.
* Originally pitched as "declaration macros"; attached macros were separated into their own proposal after the initial discussion.

## Future directions

### Additional attached macro roles

There are numerous ways in which this proposal could be extended to provide new macro roles. Each new macro role would introduce a new role kind to the `@attached` attribute, along with a corresponding protocol. The macro vision document has a number of such suggestions.

## Appendix

### Implementation of `AddCompletionHandler`

```swift
public struct AddCompletionHandler: PeerDeclarationMacro {
  public static func expansion(
    of node: CustomAttributeSyntax,
    providingPeersOf declaration: some DeclSyntaxProtocol,
    in context: some MacroExpansionContext
  ) throws -> [DeclSyntax] {
    // Only on functions at the moment. We could handle initializers as well
    // with a little bit of work.
    guard let funcDecl = declaration.as(FunctionDeclSyntax.self) else {
      throw CustomError.message("@AddCompletionHandler only works on functions")
    }

    // This only makes sense for async functions.
    if funcDecl.signature.asyncOrReasyncKeyword == nil {
      throw CustomError.message(
        "@AddCompletionHandler requires an async function"
      )
    }

    // Form the completion handler parameter.
    let resultType: TypeSyntax? = funcDecl.signature.output?.returnType.withoutTrivia()

    let completionHandlerParam =
      FunctionParameterSyntax(
        firstName: .identifier("completionHandler"),
        colon: .colonToken(trailingTrivia: .space),
        type: "(\(resultType ?? "")) -> Void" as TypeSyntax
      )

    // Add the completion handler parameter to the parameter list.
    let parameterList = funcDecl.signature.input.parameterList
    let newParameterList: FunctionParameterListSyntax
    if let lastParam = parameterList.last {
      // We need to add a trailing comma to the preceding list.
      newParameterList = parameterList.removingLast()
        .appending(
          lastParam.withTrailingComma(
            .commaToken(trailingTrivia: .space)
          )
        )
        .appending(completionHandlerParam)
    } else {
      newParameterList = parameterList.appending(completionHandlerParam)
    }

    let callArguments: [String] = try parameterList.map { param in
      guard let argName = param.secondName ?? param.firstName else {
        throw CustomError.message(
          "@AddCompletionHandler argument must have a name"
        )
      }

      if let paramName = param.firstName, paramName.text != "_" {
        return "\(paramName.withoutTrivia()): \(argName.withoutTrivia())"
      }

      return "\(argName.withoutTrivia())"
    }

    let call: ExprSyntax =
      "\(funcDecl.identifier)(\(raw: callArguments.joined(separator: ", ")))"

    // FIXME: We should make CodeBlockSyntax ExpressibleByStringInterpolation,
    // so that the full body could go here.
    let newBody: ExprSyntax =
      """

        Task.detached {
          completionHandler(await \(call))
        }

      """

    // Drop the @addCompletionHandler attribute from the new declaration.
    let newAttributeList = AttributeListSyntax(
      funcDecl.attributes?.filter {
        guard case let .customAttribute(customAttr) = $0 else {
          return true
        }

        return customAttr != node
      } ?? []
    )

    let newFunc =
      funcDecl
      .withSignature(
        funcDecl.signature
          .withAsyncOrReasyncKeyword(nil)  // drop async
          .withOutput(nil)                 // drop result type
          .withInput(                      // add completion handler parameter
            funcDecl.signature.input.withParameterList(newParameterList)
              .withoutTrailingTrivia()
          )
      )
      .withBody(
        CodeBlockSyntax(
          leftBrace: .leftBraceToken(leadingTrivia: .space),
          statements: CodeBlockItemListSyntax(
            [CodeBlockItemSyntax(item: .expr(newBody))]
          ),
          rightBrace: .rightBraceToken(leadingTrivia: .newline)
        )
      )
      .withAttributes(newAttributeList)
      .withLeadingTrivia(.newlines(2))

    return [DeclSyntax(newFunc)]
  }
}
```
