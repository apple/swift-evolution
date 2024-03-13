# Reflection

* Proposal: [SE-NNNN](NNNN-reflection.md)
* Authors: [Alejandro Alonso](https://github.com/Azoy)
* Review Manager: TBD
* Status: **Awaiting implementation**

## Introduction

I propose adding a new module to the Swift toolchain named `Reflection` that
provides high level APIs for reflection in Swift. This makes use of reflective
information that has always been available, but was never exposed as API until
now.

Swift-evolution pitch thread: [Reflection](https://forums.swift.org/t/pitch-reflection/61438)

## Motivation

Reflection is a powerful capability of any programming language allowing
developers to create APIs that weren't previously possible. It enables a
powerful sense of dynamism in a language. With enough reflection support, it
can even alter the way the language looks and feels without changing the
language design itself.

Swift's current introspective abilities are all behind `Mirror`, an API that was
designed before Evolution, providing some very basic information about an
instance's field labels and values. However, `Mirror` has some basic
restrictions that make working with it quite hard and sometimes unusable. Let's
take a look at a typical usage of this API:

```swift
struct Dog {
  var name: String
  var age: Int
}

let sparky = Dog(name: "Sparky", age: 5)

for child in Mirror(reflecting: sparky).children {
  let fieldName = child.label
  let fieldValue = child.value
  let fieldType = type(of: fieldValue)
}
```

This works great, but there is a serious issue here in that it requires an
instance of the type you want to introspect over. Say you want to register types
and for each field who conforms to my special protocol `CustomRegister`, cache
it and send that as part of the types data when say my server requests that
type. This wouldn't be that hard if you had an instance of the type you're
registering, but if you're abstracting over metatypes, you quickly realize you
simply can't do this at all:

```swift
func register(_ type: Any.Type) {
  // How do I iterate the stored properties in 'type'?
}
```

The fact of the matter is with `Mirror` it's impossible. Obviously with just a
metatype we wouldn't get back a field's value (because there is none), but it
would be nice to have access to its name, type, etc. abstractly.

Another simple example is getting enum case names. In the following example, we
have an enum with some cases that define a raw string value for their proper
capitalized name.

```swift
enum Food: String {
  case burger = "Hamburger"
  case fries = "French Fries"
  case tots = "Tater tots"
  case milkshake = "Milkshake"
}

let favoriteFood = Food.tots
print(String(describing: favoriteFood)) // "tots"
```

We can easily access the string of "tots" by calling the `String(describing:)`
initializer on a value of the enum case, but what if we didn't have a value of
the case? Or maybe we later add a conformance that implicitly adds a
`CustomStringConvertible` conformance and overrides the default printing
behavior of your enum:

```swift
enum Food: String, CodingKey {
  ...
}

let favoriteFood = Food.tots

// "Food(stringValue: "Tater tots", intValue: nil)"
print(String(describing: favoriteFood))
```

and there's just no way of getting the enum case name "tots" now.

Consider you're writing an algorithm that is generic over any sequence, but you
have a specific optimization that works only for arrays and the dictionary keys
collection like the following:

```swift
func sequenceAlgorithm<T: Sequence>(_ seq: T) {
  if seq is [Any] {
    // How do I get 'Element'?
  }

  // This doesn't work!
  if seq is Dictionary<AnyHashable, Any>.Keys {
    // How do I get 'Key' and 'Value'?
  }

  // Do slow thing
}
```

The first check works because the runtime will convert whatever concrete
element type is in the array to `Any`, but the second check doesn't work at all
because our key will not get converted to `AnyHashable` implicitly. However in
both cases, we still have no fundamental way of getting our generic arguments to
say create a specialized wrapper, get their size, alignment, etc. We could alter
the language to support these use cases, but the information needed to perform
these queries are already available and present in Swift binaries.

The unfortunate truth with all of these examples is that the compiler emits the
data required to achieve what we want in every case. Our current public API just
doesn't surface any of that vast amount of information that is available to us.

## Proposed solution

Swift introduces a new module, `Reflection`, with a whole new suite of high
level APIs that make working with reflection easier, more ergonomic, and
provides developers with much more information than what they had.

We can take each of the motivating examples and see how we can achieve our
result using these new `Reflection` APIs.

Our registration example can finally iterate a type's fields without even
having access to an instance of the type by using the new `Type` API:

```swift
import Reflection

func register(_ type: Any.Type) {
  for prop in Type(type).storedProperties {
    if prop.type.swiftType is CustomRegister.Type {
      cache[type].append(prop)
    }
  }
}
```

In our enum case name example, we can use a new type called `Case` and use our
enum instance to initialize a value of this `Case` and grab the name out:

```swift
import Reflection

enum Food: String, CodingKey {
  case burger = "Hamburger"
  case fries = "French Fries"
  case tots = "Tater tots"
  case milkshake = "Milkshake"
}

let favoriteFood = Food.tots
let caze = Case(from: favoriteFood)!

print(caze.name) // "tots"
```

Finally, we can use a new type called `PartialType` that lets us do these sort
of generic erased queries and utilize another new API on `Type` called
`genericArguments`:

```swift
import Reflection

func sequenceAlgorithm<T: Sequence>(_ seq: T) {
  let genericArgs = Type(T.self).genericArguments

  // Void doesn't matter here as you'll see later on
  if Type(T.self).partial == Type([Void].self).partial {
    let elementType = genericArgs[0]
  }

  // This dictionary type doesn't matter here as you'll see later on
  if Type(T.self).partial == Type([AnyHashable: Any].Keys.self).partial {
    let keyType = genericArgs[0]
    let valueType = genericArgs[1]
  }

  // Do slow thing
}
```

## Detailed design

`Reflection` will be a new module included in the Swift toolchain. In order to
use this module, one must explicitly declare an `import Reflection`.

### `Type`

This is our main entry point into the reflection module. `Type` represents all
possible concrete types that can occur in Swift. Ranging from structs, enums,
tuples, functions, existentials, etc. Anything that can be on the right hand
side of an `as` is represented here in `Type`. 

```swift
@frozen
public struct Type {
  @usableFromInline
  let metadata: Metadata

  /// Converts a Swift metatype to a 'Type'.
  @inlinable
  public init(_ type: Any.Type)

  /// Gets the runtime type of the instance and makes a 'Type'.
  @inlinable
  public init(_ instance: Any)
}

extension Type {
  /// Returns a collection wrapper over 'Case'
  @inlinable
  public var cases: Cases { get }

  /// Returns a collection wrapper over 'StoredProperty'
  @inlinable
  public var storedProperties: StoredProperties { get }

  /// Returns a collection wrapper over the generic arguments needed to
  /// construct this concrete 'Type'. The collection will always be empty for
  /// types who are not structs, classes, or enums as well as types who are not
  /// generic.
  @inlinable
  public var genericArguments: GenericArguments { get }
}

extension Type {
  /// Returns the 'PartialType' associated with this type, if it has one. A type
  /// always has a partial type if it's representing a struct, enum, or class
  /// type. Any other type never has a partial type.
  @inlinable
  public var partial: PartialType? { get }

  /// If this type is representing a class type, return the superclass type if
  /// it has one.
  @inlinable
  public var superclass: Type? { get }

  /// Converts the type back to a Swift metatype.
  @inlinable
  public var swiftType: Any.Type { get }
}

extension Type {
  /// Returns a collection wrapper over the function parameter types.
  @inlinable
  public var functionParameters: FunctionParameters

  /// Returns the function result type, if this type is representing a
  /// function.
  @inlinable
  public var functionResult: Type?
}

extension Type: Equatable {}
extension Type: Hashable {}
extension Type: CustomStringConvertible {}
```

### `StoredProperty`

A `StoredProperty` can represent one of three things: a struct stored property,
a class stored property, or a tuple element. By default, we return all of the
physical stored properties of types and return read-only keypaths to read values
out of. We could allow returning mutable keypaths to these, but there's no way
to opt out of that with this current API for your types which is a potential
security concern.

```swift
@frozen
public struct StoredProperty {
  @usableFromInline
  let index: Int

  @usableFromInline
  let parent: Metadata
}

extension StoredProperty {
  /// If this stored property is a struct or class stored property, this will
  /// return true if the field in question was syntactically marked 'var' and
  /// false if it marked 'let'. For a tuple element, this will always be false.
  @inlinable
  public var isVar: Bool { get }

  /// A read-only key path to get the value of this field out from an instance
  /// of its parent type.
  @inlinable
  public var keyPath: AnyKeyPath { get }

  /// The name of the stored property or the tuple element label.
  @inlinable
  public var name: String { get }

  /// The offset in bytes to this stored property for structs or tuple element.
  /// This will always return 'nil' for class properties.
  @inlinable
  public var offset: Int? { get }

  /// The type of the property or tuple element.
  @inlinable
  public var type: Type { get }
}

extension StoredProperty: Equatable {}
extension StoredProperty: Hashable {}
extension StoredProperty: CustomStringConvertible {}
```

### `Case`

A `Case` represents a single enum case. Note that cases that look like the
following: `case red, green, blue` define 3 enum cases and a value of a `Case`
represents either `.red`, `.green`, or `.blue`.

```swift
@frozen
public struct Case {
  @usableFromInline
  let parent: EnumMetadata

  @usableFromInline
  let tag: Int

  /// Given an enum case value, produce the 'Case' value that represents said
  /// case.
  @inlinable
  public init?(from: Any)
}

extension Case {
  /// Whether or not this enum case has a payload.
  @inlinable
  public var hasPayload: Bool { get }

  /// Whether this enum case was marked 'indirect'.
  @inlinable
  public var isIndirect: Bool { get }

  /// A read-only key path to get the case payload out of an instance of this
  /// enum type.
  @inlinable
  public var keyPath: AnyKeyPath { get }

  /// The name of the case as it appears in source.
  @inlinable
  public var name: String { get }

  /// The type of the case's payload. If the case is empty (it has no payload),
  /// then this will return 'Type(Void.self)'. If the case does have a payoad,
  /// it is either the single unlabeled type 'Type(Payload.self)' or a tuple of
  /// the payload elements 'Type((PayloadArg0, PayloadArg1, ..).self)'.
  @inlinable
  public var payloadType: Type { get }
}

extension Case: Equatable {}
extension Case: Hashable {}
extension Case: CustomStringConvertible {}
```

### `PartialType`

`PartialType` represents a somewhat new concept to Swift developers. Imagine you
want to compare two types who are generic over `<T>`, but you don't care if both
types share the same type for `T`, and only care if the enclosing type is the
same. That's what a `PartialType` helps you achieve. It represents `Array`, not
`Array<Int>` for example. You can get a value of this `PartialType` by calling
`Type.partial`. Here's an example of this in action:

```swift
func takeSequence<T: Sequence>(_: T) {
  // We don't really care about the 'Void' here, it's just a placeholder to get
  // the partial type.
  let arrayPartialTy = Type([Void].self).partial

  if Type(T.self).partial == arrayPartialType {
    print("Array type!")
    return
  }

  print("Something else")
}

takeSequence([1, 2, 3]) // Array type!
takeSequence(["hello", "world"]) // Array type!
takeSequence(Set([1, 2, 3])) // Something else
takeSequence(["key": "value"]) // Something else
```

Another interesting aspect of partial types is that you can create a full
fledged `Type` from them as well. Say we have the partial type for `Dictionary`.
In order for one to create a full dictionary `Type` from the partial one, we
need 2 generic arguments to be passed for our `Key` and `Value`.

```swift
// Again, we don't really care about the dictionary type here, it's just a
// placeholder.
let dictionaryPartial = Type([AnyHashable: Any].self).partial

let keyType = Type(String.self)
let valueType = Type(Set<Int>.self)

let newDictionaryType = dictionaryPartial?.create(with: keyType, valueType)

print(newDictionaryType!) // Dictionary<String, Set<Int>>
```

Now remember that `Dictionary`'s `Key` generic argument has a conformance
requirement to `Hashable`, yet our API doesn't mention requirements at all.
That's why this `create(with:)` method returns an optional type. We know what
requirements said partial type requires for its generic arguments and we will
dynamically lookup protocol conformances for the specific arguments that require
them. In the example above, we lookup `String`'s `Hashable` conformance and use
that when creating the `Dictionary<String, Set<Int>>` type. If you passed in a
type that doesn't conform to `Hashable`, we will bail out and return nil.

The order of generic arguments start from the outermost type and works its way
inwards. Consider the following scenario:

```swift
struct ABC<A, B: Equatable> {
  struct XYZ<C: Hashable, D> {}
}
```

Effectively `XYZ` has 4 generic arguments needed to fully realize it. Given
the following list of types: `[Int, String, Double, Array<Float>]`, we start
with the outermost type, `ABC`, and assign the following generic arguments:
`<A = Int, B = String>` and lookup the `Equatable` conformance for `String`. Now
for `XYZ`, we finish the generic assignment with the following:
`<C = Double, D = Array<Float>>` and lookup `Double`'s `Hashable`.

Full detailed proposed API:

```swift
@frozen
public struct PartialType {
  @usableFromInline
  let descriptor: TypeDescriptor
}

extension PartialType {
  /// Whether or not this type has generic arguments. Note that this type
  /// specifically may not have direct generic arguments, but it may be nested
  /// in a generic type who do contribute to this type's generic arguments.
  @inlinable
  public var isGeneric: Bool { get }
}

extension PartialType {
  /// Instantiates a new fully realized 'Type' of this partial type using no
  /// arguments. Used to disambiguate the variadic cases.
  @inlinable
  public func create() -> Type?

  /// Use the given list of types as generic arguments used to instantiate a
  /// new fully realized 'Type' of this partial type.
  @inlinable
  public func create(with: Any.Type...) -> Type?

  /// Use the given list of types as generic arguments used to instantiate a
  /// new fully realized 'Type' of this partial type.
  @inlinable
  public func create(with: [Any.Type]) -> Type?

  /// Use the given list of types as generic arguments used to instantiate a
  /// new fully realized 'Type' of this partial type.
  @inlinable
  public func create(with: Type...) -> Type?

  /// Use the given list of types as generic arguments used to instantiate a
  /// new fully realized 'Type' of this partial type.
  @inlinable
  public func create(with: [Type]) -> Type?
}

extension PartialType: Equatable {}
extension PartialType: Hashable {}

/// This will return the non-generic name of this type. For example, 'Array' or
/// 'Dictionary'.
extension PartialType: CustomStringConvertible {}

/// This will return the non-generic, fully qualified, and placeholder name of
/// this type. For example, 'Swift.Array<_>' or
/// 'Swift.Dictionary<_: Hashable, _>'.
extesnion PartialType: CustomDebugStringConvertible {}
```

## Source compatibility

All of this is new API in a new standalone module. The only source
compatibility concerns are for existing types that share the same name as the
ones being proposed, but in order to see these types one must explicitly import
the reflection module. Thus, there is no source breakage.

## Effect on ABI stability

ABI is not broken with this proposal due to all of this being new API.

## Effect on API resilience

This proposal only introduces new API, thus API resilience should be unaffected.

## Alternatives considered

### Why not just add this functionality to `Mirror`?

We could augment `Mirror` to support all of this functionality, but there are a
few reasons why I don't think this is a great idea.

1. `Mirror` is in the `Swift` module, so any new additions here must go into
that module as well. I don't think it's a great idea to add all of this new
functionality to that module because not everyone needs/should use reflection
and breaking these APIs out into a dedicated module makes it clear when one is
using reflection. I'd like to leave the `Swift` module dedicated for core APIs
that everyone gets for free by default, whereas this should be separated into
its own dedicated module.

2. I think the current design of `Mirror` lends itself to some performance
issues that designing a new interface like the proposed API doesn't have.
`Mirror` eagerly demangles each field type as you iterate its children as well
as ripping out the value from the instance, but one may not care about each
field and only needs to look for 1. In this case, the new API lazily retrieves
this information as you need it (as well as doing other performance
optimizations to make it much faster than `Mirror`).

3. It's quite old and was designed before evolution. Having the chance to go
through evolution and gain feedback from community members about how to best
shape the future reflection APIs is invaluable. A lot of the frustrations with
`Mirror` might have been avoided if evolution existed at the time of creation.

### `Type<T>` instead of `Type`

Having a generic argument represent the type that we're reflecting over has some
benefits over the erased version. APIs like `StoredProperty.keyPath` could
instead be `StoredProperty<T>.keyPath` where we can now return a
`PartialKeyPath<T>` instead of the `AnyKeyPath` return type.

There are other benefits too like `Type<T>` doesn't need to have any properties
and can perform all of its current operations by just referencing the generic
parameter `T` and using static methods instead:

```swift
for genericArg in Type<[Int]>.genericArguments {
  // ...
}

for prop in Type<[Int]>.storedProperties {
  // ...
}
```

Another big issue with this design is that we'd still need a way to reference
an erased type when returning a type whose generic argument we don't know at
compile time. Consider the `Type.superclass` API. What does this return if
`Type` is generic? It could return `Any.Type?`, but now you've lost the ability
to seamlessly optional chain this with `Type` APIs. E.g.

```swift
// Before
someType.superclass?.storedProperties

// After
guard let superclassType = someType.superclass else {
  // ...
}

func opened<T>(_: T.Type) {
  Type<T>.storedProperties
}

_openExistential(superclassType, do: opened(_:))
```

### `Type` as a protocol and various conforming types

One can imagine a design where `Type` is instead a protocol and we define types
like `StructType` who represent only struct types. 

```swift
protocol Type {
  // ...
}

struct StructType: Type {}
struct EnumType: Type {}
struct ClassType: Type {}
struct TupleType: Type {}

// ... and so on

```

This approach has some advantages such as the fact that the types themselves are
the discriminators when determining if say a `any Type` is a tuple type or not
by `as? TupleType`. It would also allow for more fine grained APIs, like
`genericArguments` or `partial`, to only be the respected conforming types.

However, I think there are a few quirks with this design that the generic `Type`
doesn't have. Writing generic code over a `Type` protocol is pretty useless.
If we had separate types for each kind of type, the base protocol wouldn't have
any useful operations that work for all kinds of types. It would require
downcasting to the respectful type to do any meaningful operation. We also don't
want people conforming to `Type` at all. That could be mitigated by not having
a base protocol at all here and just defining the concrete types.

There is another issue of failable initialization. All of these types can fail
to be initialized with an `Any.Type` because it may or may not be the kind of
type. Whereas `Type` can provide a non-failable initializer, but on the flip
side, APIs like `functionResult` have to return `Type?` for the erased `Type`,
but could be non-optional for something like `FunctionType`.

```swift
struct Type {
  init(_: Any.Type)

  var functionResult: Type?
}

// Vs.

struct FunctionType: Type {
  init?(_: Any.Type)

  var result: any Type // or Any.Type
}
```

### Why not put the API on `Type` directly on `Any.Type`?

Another option was to stick all of the new APIs proposed for `Type` directly on
the `Any.Type` and concrete cases. This has the advantage of not having a top
level type named `Type` and reuses the metatype values we've grown accustomed
to.

```swift
for prop in Dog.self.storedProperties {
  // ...
}

for prop in Dog.storedProperties {
  // ...
}

func printCases<T>(of type: T.Type) {
  for caze in type.cases {
    print(caze)
  }
}
```

The biggest reason I didn't go for this approach was because static members of
the underlying type would be present in things like code completion. It would be
less of an issue if these APIs were only available on `Any.Type` instead of
that and concrete types like `Int.Type`. Should these APIs be available when one
types `String.` and/or only `String.self.`? If a type already has a static
member named after one of these new APIs, then this alternative design would be
much more source breaking than gating them in a completely new type.

## Future Directions

### Runtime module

Powering the reflection APIs and this module is the ability to interact and
communicate with the Swift runtime. There are a bunch of runtime routines and
data structures needed to retrieve the information to be able to provide these
reflective capabilities. I think another module dedicated to interacting with
the runtime makes a lot of sense to 1. help build higher level APIs like this
`Reflection` module does, and 2. prevent people from reimplementing and
potentially misusing lots of these low level facilities. This new module would
provide very low level APIs that most developers should hopefully never need.

In fact, [the previous demangle function proposal review notes](https://forums.swift.org/t/returned-for-revision-se-0262-demangle-function/28186)
mentioned this new runtime module as a place this `demangle` function could
call its home:

> Independent of this proposal, the Core Team would be interested in starting discussion of a "Runtime" module, akin to `<objc/runtime.h>` in Objective-C or `libc++abi`, that provides access to low-level functionality and data structures in the Swift runtime that can be used for reflection and diagnostic purposes but should not be part of the standard library. The Core Team thinks that the proposed demangle function makes sense as a standalone, top-level function. However, it would be a natural candidate for such a Runtime module if it existed.

### More reflection information

With the reflection data we have now, we can do some pretty cool queries on
types, fields, and cases, but there's nothing to support attributes, functions,
computed properties, etc. One can imagine new opt in reflective data to say be
able to invoke methods dynamically or lookup the attributes on a type or field.
This new data would let us add more and more powerful reflection APIs, but we'd
need to weigh the cost of how much code size they'd add and making it
completely opt in.
