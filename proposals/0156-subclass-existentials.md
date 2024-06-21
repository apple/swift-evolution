# Class and Subtype existentials

* Proposal: [SE-0156](0156-subclass-existentials.md)
* Authors: [David Hart](https://github.com/hartbit), [Austin Zheng](https://github.com/austinzheng)
* Review Manager: [Doug Gregor](https://github.com/DougGregor)
* Status: **Implemented (Swift 4.0)**
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-se-0156-class-and-subtype-existentials/5477)
* Bug: [SR-4296](https://bugs.swift.org/browse/SR-4296)

## Introduction

This proposal brings more expressive power to the type system by allowing Swift to represent existentials of classes and subtypes which conform to protocols.

[Mailing list discussion](https://forums.swift.org/t/subclass-existentials/5024)

## Motivation

Currently, the only existentials which can be represented in Swift are conformances to a set of protocols, using the `&` protocol composition syntax:

```swift
Protocol1 & Protocol2
```

On the other hand, Objective-C is capable of expressing existentials of classes and subclasses conforming to protocols with the following syntax:

```objc
id<Protocol1, Protocol2>
Base<Protocol>*
```

We propose to provide similar expressive power to Swift, which will also improve the bridging of those types from Objective-C.

## Proposed solution

The proposal keeps the existing `&` syntax but allows one of the elements to be either `AnyObject` or of class type. The equivalent to the above Objective-C types would look like this:

```swift
AnyObject & Protocol1 & Protocol2
Base & Protocol
```

As in Objective-C, the first line is an existential of classes which conform to `Protocol1` and `Protocol2`, and the second line is an existential of subtypes of `Base` which conform to `Protocol`.

Here are the new proposed rules for what is valid in a existential conjunction syntax:

### 1. An element in the protocol composition syntax can be the `AnyObject` keyword to enforce a class constraint:

```swift
protocol P {}
struct S : P {}
class C : P {}
class D { }
let t: AnyObject & P = S() // Compiler error: S is not of class type
let u: AnyObject & P = C() // Compiles successfully
let v: P & AnyObject = C() // Compiles successfully
let w: P & AnyObject = D() // Compiler error: class D does not conform to protocol P
```

### 2. An element in the protocol composition syntax can be a class type to enforce the existential to be a subtype of the class:

```swift
protocol P {}
struct S {}
class C {}
class D : P {}
class E : C, P {}
let u: S & P // Compiler error: S is not of class type
let v: C & P = D() // Compiler error: D is not a subtype of C
let w: C & P = E() // Compiles successfully
```

### 3. If a protocol composition contains both a class type and `AnyObject`, the class type supersedes the `AnyObject` constraint:

```swift
protocol P {}
class C {}
class D : C, P { }
let u: AnyObject & C & P = D() // Okay: D is a subclass of C and conforms to P 
let v: C & P = u               // Okay: C & P is equivalent to AnyObject & C & P
let w: AnyObject & C & P = v   // Okay: AnyObject & C & P is equivalent to C & P
```

### 4. If a protocol composition contains two class types, either the class types must be the same or one must be a subclass of the other. In the latter case, the subclass type supersedes the superclass type:

```swift
protocol P {}
class C {}
class D : C { }
class E : C { }
class F : D, P { }
let t: C & D & P = F() // Okay: F is a subclass of D and conforms to P
let u: D & P = t       // Okay: D & P is equivalent to C & D & P
let v: C & D & P = u   // Okay: C & D & P is equivalent to D & P
let w: D & E & P       // Compiler error: D is not a subclass of E or vice-versa
```

### 5. When a protocol composition type contains one or more typealiases, the validity of the type is determined by expanding the typealiases into their component protocols, class types, and `AnyObject` constraints, then following the rules described above:

```swift
class C {}
class D : C {}
class E {}
protocol P1 {}
protocol P2 {}
typealias TA1 = AnyObject & P1
typealias TA2 = AnyObject & P2
typealias TA3 = C & P2
typealias TA4 = D & P2
typealias TA5 = E & P2

typealias TA5 = TA1 & TA2
// Expansion: typealias TA5 = AnyObject & P1 & AnyObject & P2
// Normalization: typealias TA5 = AnyObject & P1 & P2 
// TA5 is valid

typealias TA6 = TA1 & TA3
// Expansion: typealias TA6 = AnyObject & P1 & C & P2 
// Normalization (AnyObject < C): typealias TA6 = C & P1 & P2 
// TA6 is valid

typealias TA7 = TA3 & TA4
// Expansion: typealias TA7 = C & P2 & D & P2
// Normalization (C < D): typealias TA7 = D & P2
// TA7 is valid

typealias TA8 = TA4 & TA5
// Expansion: typealias TA8 = D & P2 & E & P2
// Normalization: typealias TA8 = D & E & P2
// TA8 is invalid because the D and E constraints are incompatible
```

## `class` and `AnyObject`

This proposal merges the concepts of `class` and `AnyObject`, which now have the same meaning: they represent an existential for classes. To get rid of the duplication, we suggest only keeping `AnyObject` around. To reduce source-breakage to a minimum, `class` could be redefined as `typealias class = AnyObject` and give a deprecation warning on `class` for the first version of Swift this proposal is implemented in. Later, `class` could be removed in a subsequent version of Swift.

## Inheritance clauses and `typealias`

To improve readability and reduce confusion, a class conforming to a typealias which contains a class type constraint does not implicitly inherit the class type: inheritance should stay explicit. Here are a few examples to remind what the current rules are and to make the previous sentence clearer:

The proposal does not change the rule which forbids using the protocol composition syntax in the inheritance clause:

```swift
protocol P1 {}
protocol P2 {}
class C {}

class D : P1 & P2 {} // Compiler error
class E : C & P1 {} // Compiler error
```

Class `D` in the previous example does not inherit a base class so it can be expressed using the inheritance/conformance syntax or through a typealias:

```swift
class D : P1, P2 {} // Valid
typealias P12 = P1 & P2
class D : P12 {} // Valid
```

Class `E` above inherits a base class. The inheritance must be explicitly declared in the inheritance clause and can't be implicitly derived from a typealias:

```swift
class E : C, P1 {} // Valid
typealias CP1 = C & P1
class E : CP1 {} // Compiler error: class 'E' does not inherit from class 'C'
class E : C, CP1 {} // Valid: the inheritance is explicitly declared
```

## Source compatibility

This change will not break Swift 3 compatibility mode because Objective-C types will continue to be imported as before. But in Swift 4 mode, all types bridged from Objective-C which use the equivalent Objective-C existential syntax could break code which does not meet the new protocol requirements. For example, the following Objective-C code:

```objc
@interface MyViewController
- (void)setup:(nonnull UIViewController<UITableViewDataSource,UITableViewDelegate>*)tableViewController;
@end
```

is imported into Swift-3 mode as:

```swift
class MyViewController {
    func setup(tableViewController: UIViewController) {}
}
```

which allows calling the function with an invalid parameter:

```swift
let myViewController = MyViewController()
myViewController.setup(UIViewController())
```

The previous code continues to compile but still crashs if the Objective-C code calls a method of `UITableViewDataSource` or `UITableViewDelegate`. But if this proposal is accepted and implemented as-is, the Objective-C code will be imported in Swift 4 mode as:

```swift
class MyViewController {
    func setup(tableViewController: UIViewController & UITableViewDataSource & UITableViewDelegate) {}
}
```

That would then cause the Swift code run in version 4 mode to fail to compile with an error which states that `UIViewController` does not conform to the `UITableViewDataSource` and `UITableViewDelegate` protocols.

## Alternatives considered

An alternative solution to the `class`/`AnyObject` duplication was to keep both, redefine `AnyObject` as `typealias AnyObject = class` and favor the latter when used as a type name.

The [reviewed version of the
proposal](https://github.com/swiftlang/swift-evolution/blob/78da25ec4acdc49ad9b68fb58300e49c33bc6355/proposals/0156-subclass-existentials.md)
included rules that required the class type (or `AnyObject`) to be
first within the protocol composition, e.g., `AnyObject & Protocol1`
was well-formed but `Protocol1 & AnyObject` would produce a compiler
error. When accepting this proposal, the core team removed these
rules; see the decision notes at the top for more information.

## Acknowledgements

Thanks to [Austin Zheng](https://github.com/austinzheng) and [Matthew Johnson](https://github.com/anandabits) who brought a lot of attention to existentials in this mailing-list and from whom most of the ideas in the proposal come from.
