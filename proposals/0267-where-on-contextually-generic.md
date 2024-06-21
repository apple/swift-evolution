# `where` clauses on contextually generic declarations

* Proposal: [SE-0267](0267-where-on-contextually-generic.md)
* Author: [Anthony Latsis](https://github.com/AnthonyLatsis)
* Review Manager: [Joe Groff](https://github.com/jckarter)
* Status: **Implemented (Swift 5.3)**
* Implementation: [apple/swift#23489](https://github.com/apple/swift/pull/23489)
* Decision Notes: [Review](https://forums.swift.org/t/se-0267-where-clauses-on-contextually-generic-declarations/30051/49), [Announcement](https://forums.swift.org/t/accepted-se-0267-where-clauses-on-contextually-generic-declarations/30474)

## Introduction

This proposal aims to lift the restriction on attaching `where` clauses to member declarations that can reference only outer generic parameters. Simply put, this means the `'where' clause cannot be attached` error will be relaxed for most declarations nested inside generic contexts:

```swift
struct Box<Wrapped> {
    func boxes() -> [Box<Wrapped.Element>] where Wrapped: Sequence { ... }
}

```

> The current proposal only covers member declarations that already support a generic parameter list, that is, properties and unsupported constraints on protocol requirements are out of scope.
> To illustrate, the following remains invalid:
> ```swift
> protocol P {
>     // error: Instance method requirement 'foo(arg:)' cannot add constraint 'Self: Equatable' on 'Self'
>     func foo() where Self: Equatable  
> }
>
> class C {
>     // error: type 'Self' in conformance requirement does not refer to a generic parameter or associated type
>     func foo() where Self: Equatable  
> }
> ```
> Whereas placing constraints on an extension member rather than the extension itself becomes possible:
> ```swift
> extension P {
>     func bar() where Self: Equatable { ... }
> }
> ```

[Swift-evolution thread](https://forums.swift.org/t/where-clauses-on-contextually-generic-declaractions/22449)

## Motivation

Today, a contextual where clause on a member declaration can only be expressed indirectly by placing the member inside a dedicated constrained extension. Unless constraints are identical, every such member requires a separate extension.
This lack of flexibility is a potential obstacle to grouping semantically related APIs, stacking up constraints, and the legibility of heavily generic interfaces.

It is reasonable to expect a `where` clause to "work" anywhere a constraint is meaningful, which is to say both of these structuring styles should be available to the user:

```swift
// 'Foo' can be any kind of nominal type declaration.
// For a protocol, 'T' would be Self or an associatedtype.
struct Foo<T>  

extension Foo where T: Sequence, T.Element: Equatable {
    func slowFoo() { ... }
}
extension Foo where T: Sequence, T.Element: Hashable {
    func optimizedFoo() { ... }
}
extension Foo where T: Sequence, T.Element == Character {
    func specialCaseFoo() { ... }
}

extension Foo where T: Sequence, T.Element: Equatable {
    func slowFoo() { ... }

    func optimizedFoo() where T.Element: Hashable { ... }

    func specialCaseFoo() where T.Element == Character { ... }
}
```
A step towards generalizing `where` clause use is an obvious and farsighted improvement to the generics
system with numerous future applications, including generic properties, [opaque types](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0244-opaque-result-types.md), [generalized
existentials](https://github.com/apple/swift/blob/master/docs/GenericsManifesto.md#generalized-existentials) and constrained protocol requirements. 

## Detailed design
### Overrides

Because contextual `where` clauses are effectively visibility constraints, overrides adopting this feature must be at least as visible as the overridden method:

```swift 
class Base<T> {
    func foo() where T == Int { ... }
}

class Derived<T>: Base<T> {
    // OK, the substitutions for <T: Equatable> are a superset of those for <T == Int>
    override func foo() where T: Equatable { ... } 
}
```

## Source compatibility and ABI stability

This is an additive change with no impact on the ABI and existing code.

## Effect on API resilience

For public declarations in resilient libraries such as the Standard Library, moving a constraint from an extension to a member and vice versa will break the ABI due to subtle mangling differences as of the current implementation.
