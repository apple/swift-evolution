# Rename Literal Syntax Protocols

* Proposal: [SE-0115](0115-literal-syntax-protocols.md)
* Author: [Matthew Johnson](https://github.com/anandabits)
* Review Manager: [Chris Lattner](https://github.com/lattner)
* Status: **Implemented (Swift 3.0)**
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-se-0115-rename-literal-syntax-protocols/3358)
* Bug: [SR-2054](https://bugs.swift.org/browse/SR-2054)

## Introduction

This proposal renames the `*LiteralConvertible` protocols to `ExpressibleBy*Literal`.  

Swift-evolution thread: [Literal Syntax Protocols](https://forums.swift.org/t/proposal-draft-literal-syntax-protocols/3109)

An earlier thread that resulted in this proposal: [Revisiting SE-0041 Names](https://forums.swift.org/t/revisiting-se-0041-names/3084)

## Motivation

The standard library currently has protocols that use the term `Convertible` in two different ways.  The `*LiteralConvertible` protocols use the meaning of converting *from* a literal.  The `Custom(Debug)StringConvertible` protocols use the meaning of converting *to* a `String`.  This causes confusion for developers attempting to name their own protocols following the precedence established by the standard library.

Further, the standard library team has observed:

> The "literal" protocols are not about conversion, they are about adopting
> a certain syntax provided by the language.  "Convertible" in the name is 
> a red herring: a type can't be convertible from an integer literal because 
> there is no "IntegerLiteral" entity in the type system.  
> The literal *becomes* typed as the corresponding literal type 
> (e.g., Int or String), and as far as the user at the call site is concerned, 
> there is no visible conversion (even if one is happening behind the scenes).

[An earlier proposal](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0041-conversion-protocol-conventions.md) was intended to address the first problem by introducing strong naming conventions for three kinds of conversion protocols (*from*, *to*, and *bidirectional*).  The review highlighted the difficulty in establishing conventions that everyone is happy with.  This proposal takes a different approach to solving the problem that originally inspired that proposal while also solving the awkwardness of the current names described by the standard library team.

## Proposed solution

This proposal addresses both problems by renaming the protocols to `ExpressibleBy*Literal`.  The proposal **does not** make any changes to the requirements of the protocols.

## Detailed design

All of the `*LiteralConvertible` protocols will receive new `ExpressibleBy*Literal` names.  

This proposal does not change any requirements of these protocols.  All requirements of all `*LiteralConvertible` protocols will remain exactly the same.

The following protocol declarations and names:

```swift
public protocol NilLiteralConvertible { ... }
public protocol BooleanLiteralConvertible { ... }
public protocol FloatLiteralConvertible { ... }
public protocol IntegerLiteralConvertible { ... }
public protocol UnicodeScalarLiteralConvertible { ... }
public protocol ExtendedGraphemeClusterLiteralConvertible { ... }
public protocol StringLiteralConvertible { ... }
public protocol StringInterpolationConvertible { ... }
public protocol ArrayLiteralConvertible { ... }
public protocol DictionaryLiteralConvertible { ... }
```

Are changed as follows:

```swift
public protocol ExpressibleByNilLiteral { ... }
public protocol ExpressibleByBooleanLiteral { ... }
public protocol ExpressibleByFloatLiteral { ... }
public protocol ExpressibleByIntegerLiteral { ... }
public protocol ExpressibleByUnicodeScalarLiteral { ... }
public protocol ExpressibleByExtendedGraphemeClusterLiteral { ... }
public protocol ExpressibleByStringLiteral { ... }
public protocol ExpressibleByStringInterpolation { ... }
public protocol ExpressibleByArrayLiteral { ... }
public protocol ExpressibleByDictionaryLiteral { ... }
```

## Impact on existing code

All code that references any of the `*LiteralConvertible` protocols will need to be modified to reference the protocol via the new `ExpressibleBy*Literal` name.

## Alternatives considered

Discussion of the pros and cons of the proposed and alternative naming schemes is encouraged.  The core team should feel free to choose names they deem best suited for Swift after community discussion and review if they decide to accept this proposal.

The discussion thread for this proposal includes abundant bike shedding on the names.  This section includes selected examples to highlight different directions that have been discussed.  Reviewers are encouraged to read the discussion thread if they wish to see all of the alternatives.  The thread includes abundant discussion of the pros and cons of many naming ideas.

Some of the names that have been suggested have been inaccurate due to a misunderstanding of what the protocols do.  Dave Abrahams explained during the discussion:

> No, it's exactly the opposite, as I keep saying.  Conformance to this
> protocol does *not* mean you can initialize the type with a literal.
> Proof:
>
> ```swift
> func f<T: IntegerLiteralConvertible>() -> T {
>    return T(integerLiteral: 43) // Error
>    return T(43)                 // Also an Error
> }
>
> // It means an instance of the type can be *written* as a literal:
>
> func f<T: IntegerLiteralConvertible>() -> T {
>    return 43   // OK
> }
>```
>
> Everybody's confused about the meaning of the protocol, and doesn't like
> the proposed names because they imply exactly the actual meaning of the
> protocol, which they misunderstand.

### Previous Version

The original version of this proposal introduced a `Syntax` "namespace" (using an empty `enum`) and placed the protocols in that namespace with a `*Literal` naming scheme like this:

```swift

public protocol _NilLiteralSyntax { ... }

public /* closed */ enum Syntax {
  public typealias NilLiteral = _NilLiteralSyntax
}
```

Several commenters suggested that this naming scheme is confusing at the site of use.  The ensuing discussion led to the approach in the current proposal.

Nate Cook provided the best explanation of the potential confusion:

> Primarily, the new names read like we're saying that a conforming type is a 
> literal, compounding a common existing confusion between literals and types 
> that can be initialized with a literal. Swift's type inference can sometimes 
> make it seem like dark magic is afoot with literal conversions—for example, 
> you need to understand an awful lot about the standard library to figure out 
> why line 1 works here but not line 2:

>```swift
>var x = [1, 2, 3, 4, 5]
>let y = [10, 20]
>
>x[1..<2] = [10, 20]     // 1
>x[1..<2] = y            // 2
>```

(Note: The comment above is still valid if it is corrected to say "types that can have instances *written as* a literal" rather than "types that can be *initialized with* a literal".)

> These new names are a (small) step in the wrong direction. While it's true 
> that the type system doesn't have an IntegerLiteral type, the language does 
> have integer literals. If someone reads:
>
>```swift
>extension MyInt : Syntax.IntegerLiteral { ... }
>```
>
> the implication is that MyInt is an integer literal, and therefore instances 
> of MyInt should be usable wherever an integer literal is usable. 
> The existing "Convertible" wording may be a red herring, but it at least 
> suggests that there's a difference between a literal and a concrete type.


### Namespace names

David Sweeris suggested `Compiler` as an alternative to `Syntax`.  

Adrian Zubarev suggested a convention using a nested namespace `Literal.*Protocol`.

### Protocol names

An alternative naming scheme suggested by Xiaodi Wu emphasizes that the type *conforms to a protocol* rather than *is a literal* is:

```swift
struct Foo: Syntax.IntegerLiteralProtocol { ... }
```

### Rename the protocols without placing them in a namespace

Adrian Zubarev suggests that we could use the `*LiteralProtocol` naming scheme for these protocols (replacing `Convertible` with `Protocol`) without placing them in a namespace:

```swift
public protocol NilLiteralProtocol { ... }
public protocol BooleanLiteralProtocol { ... }
public protocol IntegerLiteralProtocol { ... }
public protocol FloatLiteralProtocol { ... }
public protocol UnicodeScalarLiteralProtocol { ... }
public protocol ExtendedGraphemeClusterProtocol { ... }
public protocol StringLiteralProtocol { ... }
public protocol StringInterpolationLiteralProtocol { ... }
public protocol ArrayLiteralProtocol { ... }
public protocol DictionaryLiteralProtocol { ... }
```

### Previous proposal

This proposal is a follow up to [Updating Protocol Naming Conventions for Conversions](0041-conversion-protocol-conventions.md).  Many related alternatives were explored during the discussion and review of that proposal.

## Acknowledgements

The name used in the final proposal was first suggested by Sean Heber.  Dave Abrahams suggested moving it out of the `Syntax` namespace that was used in the original draft of this proposal.  

The design in the original draft was suggested by Dave Abrahams, Dmitri Gribenko, and Maxim Moiseev.  

Dave Abrahams, Nate Cook, David Sweeris, Adrian Zubarev and Xiaodi Wu contributed ideas to the alternatives considered section.
