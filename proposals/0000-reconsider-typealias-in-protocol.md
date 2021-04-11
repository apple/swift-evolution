# Reconsider the semantics of type aliases in protocol extensions

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Anthony Latsis](https://github.com/AnthonyLatsis)
* Review Manager: TBD
* Status: Awaiting review
* Pull Request: [apple/swift#NNNNN]()

[Swift-evolution pitch thread](https://forums.swift.org/t/disallow-type-aliases-in-protocols/11965?u=anthonylatsis)

## Introduction

> This proposal faces only type aliases that have a corresponding associated type.
> For convenience:
> * AT - Associated Type
> * PE - Protocol Extension

The current semantics of type aliases in protocol extensions and their interaction with associated types are incomplete and contradict existing language conventions.

The declaration of a `typealias` with an identifier equal to that of an `associatedtype` in a PE currently yields a same-type constraint. This behavior is based on accident of evolution, and was often used as a workaround for same-type constraints on associated types in protocol declarations ([SE-0142](https://github.com/apple/swift-evolution/blob/master/proposals/0142-associated-types-constraints.md)). Furthermore, it is also contrary to an important semantic convention: declarations in protocol extensions with a corresponding requirement act as default values. This deviation exposes counterproductive situations that have no practical value and considerably holds back type aliases as part of the evolving type system:

```swift
protocol P {
    associatedtype E
    func foo(_ arg: E)
}

class Foo: P {
    typealias E = String

    func foo(_ arg: E) {}
}

extension P { typealias E = Bool } // This will break Foo and any other conformances to P and its descendants.
```

With the introduction of where clauses in protocol declarations, the current behavior of type aliases in relation to ATs is no longer necessary. This gives the opportunity to finally reconsider the semantics of type aliases in protocol extensions and make them catch up with the type system.

## Proposed solution

A type alias in a PE that has a corresponding associated type requirement shall be the default value for that AT. 

### Constrained extensions

As in the case with method requirements, the conventional default value semantics will allow to customize default values for associated types in constrained extensions:

``` swift
protocol P {
  associatedtype A = Int
  associatedtype B
}

extension P where B: Collection {
  typealias A = B.Element
}
```
This is a feature the standard library has been using internal hack-arounds for to get defaulting behaviour for the `Indices` type of a constrained `RandomAccessCollection`: [RandomAccessCollection.swift](https://github.com/apple/swift/blob/master/stdlib/public/core/RandomAccessCollection.swift#L270-L273)

### Edge Cases

Defining a default value for an AT in an extension that refines it's constrains is illegal. Doing so would mean we can define a default for a value that has already been implemented. This would have no effect, but it's preferrable to disallow cases that aren't backed up by source compatibility and cannot show practical value to avoid supporting them in the future.

``` swift
protocol P {
  associatedtype R
}
extension P where R: Collection { 
  typealias R = [Int] // error: default value has no effect; extension constrined with 'R' implies an existing value for 'R'.
}
```

Default value collisions are a ambiguity error.
``` swift
protocol A {
  associatedtype R: Collection = [Bool] 
}
extension A where R: Collection { 
  typealias R = [Int]
}

protocol B {
  associatedtype R: Collection = [Bool] // note: default value '[Bool]' was previously defined here
}
extension B { 
  typealias R = [Int] // error: cannot define default value for associated type 'A' with existing default value '[Bool]'.
}
```


## Detailed design

## Source compatibility

Code that relies on same-type constraints that type aliases produce will break. Nevertheless, the impact will not be major: a `typealias` in a PE can only be used as a same-type constraint on an AT when there are no requirements using that type. Otherwise, an error is raised, which is in fact a bug:

``` swift
protocol P {
  associatedtype R
  func foo() -> R // error: 'R' is ambiguous for type lookup in this context
}
extension P {
  typealias R = Int
}
```

## Effect on ABI stability

The changes this proposal implies will likely extend the ABI. 

## Alternatives considered

Since we already have an equivalent way to provide a default value for an AT, the existing syntax can be enforced on type aliases in unconstrained extensions to avoid situations when default values conflict:

``` swift
protocol P {
  associatedtype R = Int
}
extension P {
  typealias R = String // Enforce existing syntax
}
```
