# Remove associated type inference

* Proposal: [SE-0108](0108-remove-assoctype-inference.md)
* Authors: [Douglas Gregor](https://github.com/DougGregor), Austin Zheng
* Review Manager: [Chris Lattner](https://github.com/lattner)
* Status: **Rejected**
* Decision Notes: [Rationale](https://forums.swift.org/t/rejected-se-0108-remove-associated-type-inference/3304)

## Introduction

In Swift, a type `T` may choose to conform to a protocol `P`, where `P` has [associated types](https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/Generics.html#//apple_ref/doc/uid/TP40014097-CH26-ID189) that may be used in the protocol requirements. If the associated types are used in the requirements, the types that `T` chooses to bind those associated types to can currently be inferred by the type checker by examining how `T` chooses to implement `P`'s requirements:

```swift
// This protocol is used in subsequent examples throughout the document.
protocol SimpleCollection {
	associatedtype Element
	func object(at index: Int) -> Element?
}

class StringBag : SimpleCollection {
	func object(at index: Int) -> String? {
		// ...
	}
}
```

In this example, the typechecker deduces that `StringBag.Element` is `String` through the way the `object(at:)` requirement is implemented.

In order to simplify the compiler and typechecker, we propose to **remove associated type witness inference**.

swift-evolution thread: [pre-proposal](https://forums.swift.org/t/pitch-remove-type-inference-for-associated-types/3135)

## Motivation

According to *[Completing Generics](https://github.com/apple/swift/blob/master/docs/GenericsManifesto.md#associated-type-inference)*:

> [...] associated type inference is the only place in Swift where we have a global type inference problem: it has historically been a major source of bugs, and implementing it fully and correctly requires a drastically different architecture to the type checker.

The main advantage of removing associated type witness inference is that it decreases the complexity of the type checker. Doing so removes the only aspect of Swift that depends upon global type inference. Simplifying the type checker makes it easier to improve the performance and correctness of the type checker code. Given that both are widely acknowledged issues with current versions of Swift, any opportunity for improvement should be carefully considered.

As Douglas Gregor (original author of the relevant type inference code) [puts it](https://forums.swift.org/t/pitch-remove-type-inference-for-associated-types/3135/23):

> Because this is the only place we do global type inference, it’s put tremendous pressure on the type checker that caused a huge number of bugs, crashes, and outright incomprehensible behavior. [...] [The re-implementation is] still not global *enough* to actually be predictable, and the legacy of this mis-feature manifests in a number of weird ways (e.g., typealiases in protocol extensions cannot be used to satisfy associated type requirements, weird rules for when a defaulted associated type gets used).

## Proposed solution

Associated type witness inference will be removed. A type implementing one or more protocols with associated types will have to explicitly spell out how those associated types are bound using one of the following methods.

### Explicit binding using `typealias`

A type may bind an associated type to a specific type using a `typealias` declaration, whether in the primary definition or retroactively through an extension:

```swift
class StringBag : SimpleCollection {
	typealias Element = String

	func object(at index: Int) -> String? { /* ... */ }	
}
```

### Explicit binding using nested type

A type may bind an associated type to a specific type by defining a nested type with the name of that associated type:

```swift
class FooBag : SimpleCollection {
	struct Element { /* ... */ }

	func object(at index: Int) -> Element? { /* ... */ }
}
```

### Default type for associated type

A type may adopt the default type specified for an associated type without any explicit annotation:

```swift
protocol P {
	associatedtype A = Int
}

class C : P {
	// C.A is Int
	// ...
}
```

Removing the associated type witness inference machinery will allow typealiases to be defined in protocol extensions, which can also be used to define default type values for associated types:

```swift
protocol P {
	associatedtype A
	associatedtype B
}

extension P where A : Fooable {
	typealias B = Int
}

class C1 : P {
	// C1.A is not Fooable
	struct A { /* ... */ }
	
	// Must bind 'C1.B' explicitly
	typealias B = String
}

class C2 : P {
	// C2.A is Fooable
	struct A : Fooable { /* ... */ }

	// 'C2.B' is implicitly Int
	// No need for explicit binding
}
```

## Detailed design

There currently exists a possible issue where a requirement on a protocol might be implemented both by a protocol extension (default implementation), and by a conforming type, but the implementation considered by the compiler to fulfill the protocol requirement is surprising to the programmer. The following example illustrates this issue:


```swift
protocol P {
	associatedtype A = Int

	func doSomething() -> A
}

extension P {
	func doSomething() -> Int {
		return 50
	}
}

class C : P {
	func doSomething() -> String {
		return "hello"
	}
}

func myMethod<T : P>(_ x: T) -> T.A {
  return x.doSomething()
}
```

Currently, `C.A` for the previous example would be inferred to be `String`, and the `doSomething()` implementation returning `String` would be considered to fulfill the protocol requirement.

If associated type inference were to be removed, `C.A` would be bound as `Int` (since there would be no explicit `typealias` declaration overriding the default type value), and the `doSomething()` implementation returning `Int` would be considered to fulfill the protocol requirement. Thus, the semantics of the code listing above would change even though the source itself remained unchanged.

To some extent, this is an issue inherent to any design which makes no distinctions at the site of implementation between members intended to satisfy protocol requirements and members that are explicitly not intended to satisfy protocol requirements. Rather than adding keywords to create this distinction, Douglas Gregor has [proposed and implemented type checker heuristics](https://forums.swift.org/t/warning-when-overriding-an-extension-method-thats-not-in-the-protocol/861/2) that will generate warnings when a programmer implements a member that "looks like" it should fulfill a protocol requirement but does not actually do so. This is one possible mitigation strategy that should be revisited as a way to decrease the possible impact of removing associated type witness inference from the compiler.

## Impact on existing code

Swift source code containing types conforming to protocols with associated types will need to explicitly define their associated types using the syntax detailed in the proposal in some cases. This is a source-breaking change.

## Alternatives considered

A couple of alternatives follow.

### Keep the current behavior

The current behavior is kept. Swift will continue to allow associated types to be inferred.

There are some advantages to this approach. Brevity is slightly improved. A type's associated types don't "stand out" in the type declaration, being unobtrusively and implicitly defined through the implementation of protocol requirements.

As well, Dave Abrahams expresses a [potential issue](https://forums.swift.org/t/pitch-remove-type-inference-for-associated-types/3135/17):

> Finally, I am very concerned that there are protocols such as `Collection`, with many inferrable associated types, and that conforming to these protocols could become *much* uglier.

As with many proposals, there is a tradeoff between the status quo and the proposed behavior. As *Completing Generics* puts it,

> Is the value of this feature worth keeping global type inference in the Swift language [...]?

### Require explicit declaration using `associatedtype`

An [earlier draft of this proposal](https://github.com/swiftlang/swift-evolution/blob/18a1781d930034583ffc0325a180099f15fbb834/proposals/XXXX-remove-assoctype-inference.md) detailed a design in which types would explicitly bind their associated types using an `associatedtype` declaration. It is presented as an alternative for consideration.
