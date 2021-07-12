# Retroactive Protocol Refinement

* Proposal: [SE-NNNN](0000-retroactive-protocol-refinement.md)
* Authors: [John Holdsworth](https://github.com/johnno1962)
* Review Manager: TBD
* Status: **Awaiting review**
* Implementation: [apple/swift#32228](https://github.com/apple/swift/pull/32228)
* Decision Notes: TBD
* Bugs: [SR-11013](https://bugs.swift.org/browse/SR-11013)

## Introduction

Protocol extensions are a powerful feature of the Swift language but it is not currently possible to specify that an extension provides conformance to a particular protocol. This is mentioned as "Retroactive protocol refinement" in the [Generics Manifesto](https://github.com/apple/swift/blob/master/docs/GenericsManifesto.md#retroactive-protocol-refinement). The PR mentioned above explores as a proof of concept how difficult it would be to implement this feature and whether it could perhaps be considered as an additive change to the Swift Language.

Swift-evolution thread: [Protocol extensions inheriting protocols](https://forums.swift.org/t/protocol-extensions-inheriting-protocols/25491/8)

## Motivation

Extensions to classes or structs (nominals) are able to specify conformances that they implement. When one extends a protocol however, it seems reasonable that specifying conformance would add that functionality to that an entire class of nominals i.e. those that already adopt the protocol being extended or protocols that adopt that protocol, etc. One example that was discussed was something that should be a simple case, where one wants to make all fixed width integers expressible by a UnicodeScalarLiteral (i.e. Strings). On the face of it one might assume the following would be possible:

```Swift
extension FixedWidthInteger: ExpressibleByUnicodeScalarLiteral {
  @_transparent
  public init(unicodeScalarLiteral value: Unicode.Scalar) {
    self = Self(value.value)
  }
}
```
In practical terms, for the Swift compiler, this effectively means creating ad-hoc extensions on Int8, Int16 and all types which conform to FixedWidthInteger along with their associated witnesses. It's more powerful however as an idea and in more abstract terms (from the Generics Manifesto):

```
protocol P {
  func foo()
}

protocol Q {
  func bar()
}

extension Q : P { // Make every type that conforms to Q also conforms to P
  func foo() {    // Implement `P.foo` requirement in terms of `Q.bar`
    bar()
  }
}

func f<T: P>(t: T) { ... }

struct X : Q {
  func bar() { ... }
}

f(X()) // okay: X conforms to P through the conformance of Q to P
```
```
This is an extremely powerful feature: it allows one
to map the abstractions of one domain into another
domain (e.g., every Matrix is a Graph).
```
It continues though:

```
However, similar to private conformances, it puts a
major burden on the dynamic-casting runtime to chase
down arbitrarily long and potentially cyclic chains
of conformances, which makes efficient implementation
nearly impossible.
```
The PR above strives to demonstrate that it may be possible to avoid these pitfalls and refine protocols retroactively at compile time, providing summary conformance data to the existing runtime for an efficient implementation of dynamic casts.

## Proposed solution

Allow conformances to be added to protocol extensions with all that implies in terms of all nominals conforming to the protocol being extended acquiring the added conformances. It would be useful if this also worked for synthesized conformances (and indeed it does).

## Detailed design

The PR above makes changes in three principal areas internal to the compiler.

1)  The ConformanceLookupTable — a cache of the protocols a nominal conforms to has been adapted to traverse the inherited conformances of protocol extensions.

2) The "Generic Signature" builder and module deserializing code has been adapted to traverse these extended sources of conformances.

3) While the above takes place, a registry of extended conformances is kept in the compiler. This is used to emit the ad-hoc witness tables used by extended conformances in a manner that is compatible across modules.

In terms of what is surfaced to the language, extending conformances of a protocol is fully functional including across modules.

## Source compatibility

This is an additive feature that has a syntax which while it follows the norms of extensions in the Swift language it is currently not permitted. Therefore, there is no effect on source compatibility.

## Effect on ABI stability

This is a compiler level addition to the language that does not require any changes to the Swift runtime in order to work so it has no effect on ABI stability.

## Effect on API resilience

This proposal does not affect API resilience as it does not modify an API 
but it's more appropriate here to talk about how api resilience affects
this proposal. In order that Swift nominals in the standard library can be
evolved in future, some are declared "resilient" which introduces extra
indirection so that implementations are not dependent on a particular
memory layout at run-time. This applies to protocols in that the final
layout of their witness tables is not known at compile time so as not to be
depended on at run time. This prevents emitting the ad-hoc witness tables
used by this proposal for newly conforming types and the prototype has
disabled the assertion which would normally fire in the compiler. This
erodes the utility of the proposal as often the protocols one is looking
to extend are those defined in the standard library.

## Alternatives considered

There is a more ambitious and well advanced proposal: "[parameterised extensions](https://github.com/apple/swift/pull/25263)" of which it has been said this is a subset. This proposal has more modest goals which are hopefully more accessible to the average programmer and requires only targeted surgical changes to the compiler.
