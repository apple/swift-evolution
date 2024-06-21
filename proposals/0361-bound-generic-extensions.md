# Extensions on bound generic types

* Proposal: [SE-0361](0361-bound-generic-extensions.md)
* Authors: [Holly Borla](https://github.com/hborla)
* Review Manager: [John McCall](https://github.com/rjmccall)
* Status: **Implemented (Swift 5.7)**
* Implementation: [apple/swift#41172](https://github.com/apple/swift/pull/41172), gated behind the frontend flag `-enable-experimental-bound-generic-extensions`
* Review: ([pitch](https://forums.swift.org/t/pitch-extensions-on-bound-generic-types/57535/)) ([review](https://forums.swift.org/t/se-0361-extensions-on-bound-generic-types/58366)) ([acceptance](https://forums.swift.org/t/accepted-se-0361-extensions-on-bound-generic-types/58716))

## Contents
  - [Introduction](#introduction)
  - [Motivation](#motivation)
  - [Proposed solution](#proposed-solution)
  - [Detailed design](#detailed-design)
  - [Source compatibility](#source-compatibility)
  - [Effect on ABI stability](#effect-on-abi-stability)
  - [Effect on API resilience](#effect-on-api-resilience)
  - [Alternatives considered](#alternatives-considered)
    - [Reserving syntax for parameterized extensions](#reserving-syntax-for-parameterized-extensions)
  - [Future directions](#future-directions)
    - [Parameterized extensions](#parameterized-extensions)

## Introduction

Specifying the type arguments to a generic type in Swift is almost always written in angle brackets, such as `Array<String>`. Extensions are a notable exception, and if you attempt to extend `Array<String>`, the compiler reports the following error message:

```swift
extension Array<String> { ... } // error: Constrained extension must be declared on the unspecialized generic type 'Array' with constraints specified by a 'where' clause
```

As the error message suggests, this extension must instead be written using a `where` clause:

```swift
extension Array where Element == String { ... }
```

This proposal removes this limitation on extensions, allowing you to write bound generic extensions the same way you write bound generic types everywhere else in the language.

Swift evolution discussion thread: [[Pitch] Extensions on bound generic types](https://forums.swift.org/t/pitch-extensions-on-bound-generic-types/57535).

## Motivation

Nearly everywhere in the language, you write bound generic types using angle brackets after the generic type name. For example, you can write a typealias to an array of strings using angle brackets, and extend that type using the typealias:

```swift
typealias StringArray = Array<String>

extension StringArray { ... }
```

With [SE-0346](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0346-light-weight-same-type-syntax.md), we can also declare a primary associated type, and bind it in an extension using angle-brackets:

```swift
protocol Collection<Element> {
  associatedtype Element
}

extension Collection<String> { ... }
```

Not allowing this syntax directly on generic type extensions is clearly an artificial limitation, and even the error message produced by the compiler suggests that the compiler understood what the programmer was trying to do:

```swift
extension Array<String> { ... } // error: Constrained extension must be declared on the unspecialized generic type 'Array' with constraints specified by a 'where' clause
```

This limitation is confusing, because programmers don’t understand why they can write `Array<String>` everywhere *except* to extend `Array<String>`, as evidenced by the numerous questions about this limitation here on the forums, such as [this thread](https://forums.swift.org/t/why-doesnt-eg-extension-array-int-compile-even-though-using-a-typealias-does/56049).

## Proposed solution

I propose to allow extending bound generic types using angle-brackets for binding type arguments, or using sugared types such as `[String]` and `Int?`.

The following declarations all express an extension over the same type:

```swift
extension Array where Element == String { ... }

extension Array<String> { ... }

extension [String] { ... }
```

## Detailed design

A generic type name in an extension can be followed by a comma-separated type argument list in angle brackets. The type argument list binds the type parameters of the generic type to each of the specified type arguments. This is equivalent to writing same-type requirements in a `where` clause. For example:

```swift
struct Pair<T, U> {}

extension Pair<Int, String> {}
```

is equivalent to

```swift
extension Pair where T == Int, U == String {}
```

A type argument list applied to an extended type name must specify all type arguments; constraining only a subset of the type parameters in an extension must still use a `where` clause:

```swift
struct Pair<T, U> {}

extension Pair<Int> {} // error: Generic type 'Pair' specialized with too few type parameters (got 1, but expected 2)

extension Pair where T == Int {} // okay
```

The types specified in the type argument list must be concrete types. For example, you cannot extend a generic type with placeholders as type arguments:

```swift
extension Pair<Int, _> {} // error: Cannot extend a type that contains placeholders
```

> **Rationale**: When `_` is used as a type placeholder, it directs the compiler to infer the type at the position of the underscore. Using `_` in a bound generic extension would introduce a subtly different meaning of `_`, which is to leave the type at that position unconstrained, so `Pair<Int, _>` would mean different things in different contexts.

Name lookup of the type arguments is performed outside the extension context, so the type parameters of the generic type cannot appear in the type argument list:

```swift
extension Array<Element> {} // error: Cannot find type 'Element' in scope
```

If a generic type has a sugared spelling, the sugared type can also be used to extend the generic type:

```swift
extension [String] { ... } // Extends Array<String>

extension String? { ... } // Extends Optional<String>
```

## Source compatibility

This change has no impact on source compatibility.

## Effect on ABI stability

This is a syntactic sugar change with no impact on ABI.

## Effect on API resilience

This change has no impact on API resilience. Changing an existing bound generic extension using a where clause to the sugared syntax and vice versa is a resilient change.

## Alternatives considered

### Reserving syntax for parameterized extensions

Using angle brackets after an extended type name as sugar for same-type requirements prevents this syntax from being used to declare a parameterized extension. Alternatively, `extension Array<T, U> { ... }` could  mean an extension that declares two new type parameters `T` and `U`, rather than an (invalid) application of type arguments to `Array`'s type parameters. However, SE-0346 already introduced this syntax as sugar for same-type requirements on associated types:

```swift
protocol Collection<Element> {
  associatedtype Element
}

// Already sugar for `extension Collection where Element == String`
extension Collection<String> { ... }
```

Instead of reserving this syntax for parameterized extensions, type parameters could be declared in angle brackets after the `extension` keyword, which will help indicate that the type parameters belong to the extension:

```swift
// Introduces new type parameters `T` and `U` for the APIs
// in this extension.
extension <T, U> Array { ... }
```

## Future directions

### Parameterized extensions

This proposal does not provide parameterized extensions, but a separate proposal could build upon this proposal to allow extending a generic type with more sophisticated constraints on the type parameters:

```swift
extension <Wrapped> Array<Optional<Wrapped>> { ... }

extension <Wrapped> [Wrapped?] { ... }
```

Parameterized extensions could also allow using the shorthand `some` syntax to write generic extensions where a type parameter has a conformance requirement:

```swift
extension Array<some Equatable> { ... }

extension [some Equatable] { ... }
```

Writing the type parameter list after the `extension` keyword applies more naturally to extensions over structural types. With this syntax, an extension over all two-element tuples could be spelled

```swift
extension <T, U> (T, U) { ... }
```

This syntax also generalizes to variadic type parameters, e.g. to extend all tuple types to provide a protocol conformance:

```swift
extension <T...> (T...): Hashable where T: Hashable { ... }
```
