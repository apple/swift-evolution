# Structural opaque result types

* Proposal: [SE-0328](0328-structural-opaque-result-types.md)
* Authors: [Benjamin Driscoll](https://github.com/willtunnels), [Holly Borla](https://github.com/hborla)
* Review Manager: [Ben Cohen](https://github.com/airspeedswift)
* Status: **Implemented (Swift 5.7)**
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-with-modifications-se-0328-structural-opaque-result-type/53789)
* Implementation: [apple/swift#38392](https://github.com/apple/swift/pull/38392)
* Toolchain: Any recent [nightly main snapshot](https://swift.org/download/#snapshots) 

## Introduction

An [opaque result type](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0244-opaque-result-types.md) may be used as the result type of a function, the type of a variable, or the result type of a subscript. In all cases, the opaque result type must be the entire type. This proposal recommends lifting that restriction and allowing opaque result types in "structural" positions.

Swift-evolution thread: [Structural opaque result types](https://forums.swift.org/t/structural-opaque-result-types/50998)

## Motivation

The current restriction on opaque result types prevents them from being used in many common API patterns. Some examples are as follows:

```swift
// we cannot express a function that might fail to produce an opaque result type
func f0() -> (some P)? { /* ... */ }

// we cannot use an opaque result type as one of several return values
func f1() -> (some P, some Q) { /* ... */ }

// we cannot return a lazily computed opaque result type
func f2() -> () -> some P { /* ... */ }

// more generally, we cannot embed an opaque result type into a larger structure
func f3() -> S<some P> { /* ... */ }
```

These restrictions are artificial, and lifting them enables more APIs to be expressed using opaque result types.

## Proposed solution

We should allow opaque result types in structural positions in the result type of a function, the type of a variable, or the result type of a subscript.

## Detailed design

### Syntax for Optionals

The `some` keyword binds more loosely than `?` or `!`. An optional of an opaque result type must be written `(some P)?`, and an optional of an unwrapped opaque result type must be written `(some P)!`.

`some P?` gets interpreted as `some Optional<P>` and therefore produces an error because an opaque type must be constrained to `Any`, `AnyObject`, a protocol composition, and/or a base class. The analogous thing is true of `some P!`.

### Higher order functions

If the result type of a function, the type of a variable, or the result type of a subscript is a function type, that function type can only contain structural opaque types in return position. For example, `func f() -> () -> some P` is valid, and `func g() -> (some P) -> ()` produces an error:

```swift
protocol P {}

func g() -> (some P) -> () { ... } // error: 'some' cannot appear in parameter position in result type '(some P) -> ()'
```

### Constraint inference

When a generic parameter type is used in a structural position in the signature of a function, the compiler implicitly constrains the generic parameter based on the context is which it is used. E.g.,

```swift
struct H<T: Hashable> { init(_ t: T) {} }
struct S<T>{ init(_ t: T) {} }

// same as 'f<T: Hashable>' because 'H<T>' implies 'T: Hashable'
func f<T>(_ t: T) -> H<T> {
    var h = Hasher()
    h.combine(t) // OK - we know 'T: Hashable'
    let _ = h.finalize()
    return H(0)
}

// 'S<T>' doesn't imply anything about 'T'
func g<T>(_ t: T) -> S<T> {
    var h = Hasher()
    h.combine(t) // ERROR - instance method 'combine' requires that 'T' conform to 'Hashable'
    let _ = h.finalize()
    return S(0)
}
```

Opaque result types do not feature such inference. E.g.,

```swift
// ERROR - type 'some P' does not conform to protocol 'Hashable'
func f<T>(_ t: T) -> H<some P> { /* ... */ }
```

## Source compatibility

This change is purely additive so has no source compatibility consequences.

As discussed in [SE-0244](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0244-opaque-result-types.md#source-compatibility):

> If opaque result types are retroactively adopted in a library, it would initially break source compatibility [...] but could provide longer-term benefits for both source and ABI stability because fewer details would be exposed to clients. There are some mitigations for source compatibility, e.g., a longer deprecation cycle for the types or overloading the old signature (that returns the named types) with the new signature (that returns an opaque result type).

## Effect on ABI stability

This change is purely additive so has no ABI stability consequences.

## Effect on API resilience

This change is purely additive so has no API resilience consequences. Adopting opaque types in structural positions in a resilient library has the same implications as top-level opaque result types. From [SE-0244](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0244-opaque-result-types.md#effect-on-api-resilience):

> Opaque result types are part of the result type of a function/type of a variable/element type of a subscript. The requirements that describe the opaque result type cannot change without breaking the API/ABI. However, the underlying concrete type can change from one version to the next without breaking ABI, because that type is not known to clients of the API.

## Rust's `impl Trait`

As discussed in [SE-0244](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0244-opaque-result-types.md#rusts-impl-trait), Swift's opaque result types were inspired by `impl Trait` in Rust, which is described in [RFC-1522](https://github.com/rust-lang/rfcs/blob/master/text/1522-conservative-impl-trait.md) and extended in [RFC-1951](https://github.com/rust-lang/rfcs/blob/master/text/1951-expand-impl-trait.md).

Though SE-0244 lists several differences between `some` and `impl Trait`, one difference it does not make explicit is that `impl Trait` is allowed in structural positions, in similar to the manner to that suggested by this proposal. One difference between this proposal and `impl Trait` is that `impl Trait` may not appear in the return type of closure traits or function pointers.

## Alternatives considered

### Syntax for optionals

This proposal recommends that an optional of an opaque result type with a conformance constraint to a protocol `P` be notated `(some P)?`. However, a user's first instinct might be to write `some P?`. This latter syntax is moderately less verbose, and is, in fact, unambiguous since `Optional<P>` is not a valid opaque result type constraint. It would be possible to add a special case that expands `some P?` into `(some P)?`. The analogous thing would be done with `some P!` and `(some P)!`.

However, this is inconsistent with other parts of the language, e.g. the interpretation of `() -> P?` as `() -> Optional<P>` or the fact that `P & Q?` is an invalid construction which is properly written as `(P & Q)?`. Adding special cases to the language can decrease its learnability.

Furthermore, since `P?` is never a correct constraint, it would be possible to (and in fact this proposal's implementation does) provide a "fix it" to the user which suggests that they change `some P?` to `(some P)?`.

### Higher order functions

Consider the function `func f() -> (some P) -> ()`. If this were a valid structural opaque result type, the closure value produced by calling `f` has type `(some P) -> ()`, meaning it takes an opaque result type as an argument. That argument has some concrete type, `T`, determined by the body of the closure. Assuming no special structure on `P`, such as `ExpressibleByIntegerLiteral`, the user cannot call the closure. If they were able to, then they would be depending at the source level on the concrete type of `T` to remain fixed, which is one of the things opaque result types are designed to prevent.

Another reason to disallow returning functions that take opaque result types is that [SE-0341: Opaque Parameter Declarations](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0341-opaque-parameters.md) proposes a different meaning for `some` in parameter position in function declarations, which would cause confusion if opaque parameter types mean something different within a function type.

### Constraint inference

We could infer additional constraints on opaque result types based on context, but this would likely be confusing for the user. Whereas the syntax for generic parameters draws the user's attention to the underlying type itself, i.e. the `T` in `f<T>`, opaque result type syntax draws the user's attention to an explicit list of the protocols which the underlying type satisfies, i.e. the `P` in `some P`. At least one constraining protocol must be specified, unlike with generic parameters. The closest thing to `<T>` one can write is `some Any`.

The decision about what to do in this case seems pretty clear, which is why this section was not included in the original version of this proposal. The main utility of this discussion is in teasing out why opaque result types should function differently than generic parameters because of the implications this has for [named opaque result types](https://forums.swift.org/t/improving-the-ui-of-generics/22814#heading--reverse-generics), which will likely be proposed in the future.

**Though this is outside the scope of this proposal**, named opaque result types have a similar syntactic quality to generic parameters, and therefore should probably be subject to constraint inference in the result of a function, as well as the type of variable or the result type of a subscript.

## Future directions

This proposal is a natural stepping stone to fully generalized reverse generics, as demonstrated by the following code snippet from the [generics UI design document](https://forums.swift.org/t/improving-the-ui-of-generics/22814#heading--reverse-generics):

```swift
func groupedValues<C: Collection>(in collection: C) -> <Output: Collection> (even: Output, odd: Output)
  where C.Element == Int, Output.Element == Int
{
  return (even: collection.lazy.filter { $0 % 2 == 0 },
          odd: collection.lazy.filter { $0 % 2 != 0 })
}
```

This syntax is powerful, but it's unnecessarily verbose for many cases, such as naming an opaque result type with a conformance requirement to `Collection` in order to constrain the `Element` type. We can introduce a more natural syntax for simple associated type constraints, such as writing constraints on associated types names in angle-brackets:

```swift
func concatenate<T>(a: some Collection<.Element == T>, b: some Collection<.Element == T>) -> some Collection<.Element == T>
```

or using a more [light-weight same-type constraint syntax](https://forums.swift.org/t/pitch-light-weight-same-type-constraint-syntax/52889):

```swift
func concatenate<T>(a: some Collection<T>, b: some Collection<T>) -> some Collection<T>
```
