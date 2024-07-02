# Metatype KeyPaths

* Proposal: [SE-0438](0438-metatype-keypath.md)
* Authors: [Amritpan Kaur](https://github.com/amritpan), [Pavel Yaskevich](https://github.com/xedin)
* Review Manager: [Joe Groff](https://github.com/jckarter)
* Status: **Active review (May 30th...June 13th, 2024)** 
* Implementation: [apple/swift#73242](https://github.com/apple/swift/pull/73242)
* Review: ([Pitch](https://forums.swift.org/t/pitch-metatype-keypaths/70767))

## Introduction

KeyPath expressions access properties dynamically. They are declared with a concrete root type and one or more KeyPath components that define a path to a resulting value via the type’s properties, subscripts, optional-chaining expressions, forced unwrapped expressions, or self. This proposal expands KeyPath expression access to include static properties of a type, i.e., metatype KeyPaths.

## Motivation

Metatype keypaths were briefly explored in the pitch for [SE-0254](https://forums.swift.org/t/pitch-static-and-class-subscripts/21850) and the [proposal](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0254-static-subscripts.md#metatype-key-paths) later recommended them as a future direction. Allowing key path expressions to directly refer to static properties has also been discussed on the Swift Forums for database lookups when used [in conjunction with @dynamicMemberLookup](https://forums.swift.org/t/dynamic-key-path-member-lookup-cannot-refer-to-static-member/30212) and as a way to avoid verbose hacks like [referring to a static property through another computed property](https://forums.swift.org/t/key-path-cannot-refer-to-static-member/28055). Supporting metatype keypaths in the Swift language will address these challenges and improve language semantics.

## Proposed solution

We propose to allow KeyPath expressions to define a reference to static properties. The following usage, which currently generates a compiler error, will be allowed as valid Swift code.

```swift
struct Bee {
  static let name = "honeybee"
}

let kp = \Bee.Type.name
```

## Detailed design

### Metatype syntax

KeyPath expressions where the first component refers to a static property will include `.Type` on their root types stated in the KeyPath contextual type or in the KeyPath literal. For example:

```swift
struct Bee {
  static let name = "honeybee"
}

let kpWithContextualType: KeyPath<Bee.Type, String> = \.name // KeyPath contextual root type of Bee.Type
let kpWithLiteral = \Bee.Type.name // KeyPath literal \Bee.Type
```

Attempting to write the above metatype KeyPath without including `.Type` will trigger an error diagnostic with a fix-it recommending the addition of `Type` after the root type `Bee`:

```swift
let kpWithLiteral = \Bee.name // error: static member 'name' cannot be used on instance of type 'Bee'
```

KeyPath expressions where the component referencing a static property is not the first component do not require `.Type`:
```swift
struct Species {
  static let isNative = true
}

struct Wasp {
  var species: Species.Type {Species.self}
}

let kpSecondComponentIsStatic = \Wasp.species.isNative
```
### Access semantics

Immutable static properties will form the read-only KeyPaths just like immutable instance properties.
```swift
struct Tip {
  static let isIncluded = True
  let isVoluntary = False
}

let kpStaticImmutable: KeyPath<Tip.Type, Bool> = \.isIncluded 
let kpInstanceImmutable: KeyPath<Tip, Bool> = \.isVoluntary 
```
However, unlike instance members, KeyPaths to mutable static properties will always conform to `ReferenceWritableKeyPath` because metatypes are reference types.
```swift
struct Tip {
  static var total = 0
  var flatRate = 20
}

let kpStaticMutable: ReferenceWriteableKeyPath<Tip.Type, Int> = \.total 
let kpInstanceMutable: WriteableKeyPath<Tip, Int> = \.flatRate 
```
## Effect on source compatibility

This feature breaks source compatibility for KeyPath expressions that reference static properties after subscript overloads. For example, the compiler cannot differentiate between subscript KeyPath components by return type in the following:

```swift
struct S {
  static var count: Int { 42 }
}

struct Test {
  subscript(x: Int) -> String { "" }
  subscript(y: Int) -> S.Type { S.self }
}

let kpViaSubscript = \Test.[42] // fails to typecheck
```

This KeyPath does not specify a contextual type, without which the KeyPath value type is unknown. To form a KeyPath to the metatype subscript and return an `Int`, we can specify a contextual type with a value type of `S.Type` and chain the metatype KeyPath: 

```swift
let kpViaSubscript: KeyPath<Test, S.Type> = \Test.[42]
let kpAppended = kpViaSubscript.appending(path: \.count)
```

## ABI compatibility

This feature does not affect ABI compatibility.

## Implications on adoption

This feature is back-deployable but it requires emission of new (property descriptors) symbols for static properties.

The type-checker wouldn't allow to form key paths to static properties of types that come from modules that are built by an older compiler that don't support the feature because dynamic or static library produced for such module won't have all of the required symbols.

Attempting to form a key path to a static property of a type from a module compiled with a complier that doesn't yet support the feature will result in the following error with a note to help the developers:

```swift
error: cannot form a keypath to a static property <Property> of type <Type>
note: rebuild <Module> to enable the feature
```

## Future directions

### Key Paths to Enum cases

Adding language support for read-only KeyPaths to enum cases has been widely discussed on the [Swift Forums](https://forums.swift.org/t/enum-case-key-paths-an-update/68436) but has been left out of this proposal as this merits a separate discussion around [syntax design and implementation concerns](https://forums.swift.org/t/enum-case-keypaths/60899/32).

Since references to enum cases must be metatypes, extending KeyPath expressions to include references to metatypes will hopefully bring the Swift language closer to adopting KeyPaths to enum cases in a future pitch.

## Acknowledgments

Thank you to Joe Groff for providing pivotal feedback on this pitch and its possible implementation and to Becca Royal-Gordon for an insightful discussion around the anticipated hurdles in implementing this feature.
