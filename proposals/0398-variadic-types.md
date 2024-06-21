# Allow Generic Types to Abstract Over Packs

* Proposal: [SE-0398](0398-variadic-types.md)
* Authors: [Slava Pestov](https://github.com/slavapestov), [Holly Borla](https://github.com/hborla)
* Review Manager: [Frederick Kellison-Linn](https://github.com/Jumhyn)
* Status: **Implemented (Swift 5.9)**
* Previous Proposal: [SE-0393](0393-parameter-packs.md)
* Review: ([pitch](https://forums.swift.org/t/pitch-variadic-generic-types-abstracting-over-packs/64377)) ([review](https://forums.swift.org/t/se-0398-allow-generic-types-to-abstract-over-packs/64661)) ([acceptance](https://forums.swift.org/t/accepted-se-0398-allow-generic-types-to-abstract-over-packs/64998))

## Introduction

Previously [SE-0393](0393-parameter-packs.md) introduced type parameter packs and several related concepts, allowing generic function declarations to abstract over a variable number of types. This proposal generalizes these ideas to generic type declarations.

## Motivation

Generic type declarations that abstract over a variable number of types arise naturally when attempting to generalize common algorithms on collections. For example, the current `zip` function returns a `Zip2Sequence`, but it's not possible from SE-0393 alone to define an equivalent variadic `zip` function because the return type would need an arbitrary number of type parameters—one for each input sequence:

```swift
func zip<each S>(_ seq: repeat each S) -> ??? 
  where repeat each S: Sequence
```

## Proposed solution

In the generic parameter list of a generic type, the `each` keyword declares a generic parameter pack, just like it does in the generic parameter list of a generic function. The types of stored properties can contain pack expansion types, as in `let seq` and `var iter` below.

This lets us define the return type of the variadic `zip` function as follows:

```swift
struct ZipSequence<each S: Sequence>: Sequence {
  typealias Element = (repeat (each S).Element)

  let seq: (repeat each S)

  func makeIterator() -> Iterator {
    return Iterator(iter: (repeat (each seq).makeIterator()))
  }

  struct Iterator: IteratorProtocol {
    typealias Element = (repeat (each S).Element)

    var iter: (repeat (each S).Iterator)

    mutating func next() -> Element? {
      return ...
    }
  }
}

func zip<each S>(_ seq: repeat each S) -> ZipSequence<repeat each S>
  where repeat each S: Sequence
```

## Detailed design

Swift has the following kinds of generic type declarations:

- Structs
- Enums
- Classes (and actors)
- Type aliases

A generic type is _variadic_ if it directly declares a type parameter pack with `each`, or if it is nested inside of another variadic type. In this proposal, structs, classes, actors and type aliases can be variadic. Enums will be addressed in a follow-up proposal.

### Single parameter

A generic type is limited to declaring at most one type parameter pack. The following are allowed:

```swift
struct S1<each T> {}
struct S2<T, each U> {}
struct S3<each T, U> {}
```

But this is not:

```swift
struct S4<each T, each U> {}
```

However, by virtue of nesting, a variadic type can still abstract over multiple type parameter packs:

```swift
struct Outer<each T> {
  struct Inner<each U> {
    var fn: (repeat each T) -> (repeat each U)
  }
}
```

### Referencing a variadic type

When used with a variadic type, the generic argument syntax `S<...>` allows a variable number of arguments to be specified. Since there can only be one generic parameter pack, the non-pack parameters are always specified with a fixed prefix and suffix of the generic argument list.

```swift
struct S<T, each U, V> {}

S<Int, Float>.self  // T := Int, U := Pack{}, V := Float
S<Int, Bool, Float>.self  // T := Int, U := Pack{Bool}, V := Float
S<Int, Bool, String, Float>.self  // T := Int, U := Pack{Bool, String}, V := Float
```

Note that `S<Int, Float>` substitutes U with the empty pack type, which is allowed. The minimum number of generic arguments is equal to the number of non-pack generic parameters. In our above example, the minimum argument count is 2, because `T` and `V` must always be specified:

```swift
S<Int>.self // error: expected at least 2 generic arguments
```

If the generic parameter list of a variadic type consists of a single generic parameter pack and nothing else, it is possible to reference it with an empty generic argument list:

```swift
struct V<each T> {}

V< >.self
```
Note that `V< >` is not the same as `V`. The former substitutes the generic parameter pack `T` with the empty pack. The latter does not constrain the pack at all and is only permitted in contexts where the generic argument can be inferred (or within the body of `V` or an extension thereof, where it is considered identical to `Self`).

A placeholder type in the generic argument list of a variadic generic type is always understood as a single pack element. For example:

```swift
struct V<each T> {}

let x: V<_> = V<Int>()  // okay
let x: V<_, _> = V<Int, String>()  // okay
let x: V<_> = V<Int, String>()  // error
```

### Stored properties

In a variadic type, the type of a stored property can contain pack expansion types. The type of a stored property cannot _itself_ be a pack expansion type. Stored properties are limited to having pack expansions nested inside tuple types, function types and other named variadic types:

```swift
struct S<each T> {
  var a: (repeat each Array<T>)
  var b: (repeat each T) -> (Int)
  var c: Other<repeat each T>
}
```
This is in contrast with the parameters of generic function declarations, which can have a pack expansion type. A [future proposal](#future-directions) might lift this restriction and introduce true "stored property packs."

### Requirements

The behavior of generic requirements on type parameter packs is mostly unchanged between generic functions and generic types. However, allowing types to abstract over parameter packs introduces _requirement inference_ of [generic requirement expansions](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0393-parameter-packs.md#generic-requirements). Requirement expansion inference follows these rules:

1. If a generic type that imposes an inferred scalar requirement is applied to a pack element inside a pack expansion, the inferred requirement is a requirement expansion.
2. If a generic type imposes an inferred requirement expansion, the requirement is expanded for each of the concrete generic arguments.
3. If a generic type that imposes an inferred requirement expansion is applied to a pack element inside a pack expansion:

    1. The inferred requirement is invalid if it contains multiple pack elements captured by expansions at different depths.
    2. Otherwise, the nested requirement expansion is semantically equivalent to the innermost requirement expansion.
  
  The below code demonstrates each of the above rules:

  ```swift
  protocol P {
      associatedtype A
  }
  struct ImposeRequirement<T> where T: P {}
  struct ImposeRepeatedRequirement<each T> where repeat each T: P {}
  struct ImposeRepeatedSameType<T1: P, each T2> where repeat T1.A == each T2 {}
  
  // Infers 'repeat each U: P'
  func demonstrate1<each U>(_: repeat ImposeRequirement<each U>)
  
  // Infers 'Int: P, V: P, repeat each U: P'
  func demonstrate2<each U, V>(_: ImposeRepeatedRequirement<Int, V, repeat each U>)
  
  // Error. Would attempt to infer 'repeat <U' = each U> repeat <V' = each V> U'.A == V' which is not a supported requirement in the language
  func demonstrate3a<each U, each V>(_: repeat ImposeRepeatedSameType<each U, repeat each V>))
  
  // Infers 'Int: P, repeat each V: P'
  func demonstrate3b<each U, each V>(_: repeat (each U, ImposeRepeatedRequirement<Int, repeat each V>))
```

### Conformances

Variadic structs, classes and actors can conform to protocols. The associated type requirements of the protocol may be fulfilled by a type alias whose underlying type contains pack expansions.

### Type aliases

As with the other variadic types, a variadic type alias either has a generic parameter pack of its own, or can be nested inside of another variadic generic type.

The underlying type of a variadic type alias can reference pack expansion types in the same manner as the type of a stored property. That is, the pack expansions must appear in nested positions, but not at the top level.

```swift
typealias Element = (repeat (each S).Element)
typealias Callback = (repeat each S) -> ()
typealias Factory = Other<repeat each S>
```

Like other type aliases, variadic type aliases can be nested inside of generic functions (and like other structs and classes, variadic structs and classes cannot).

### Classes

While there are no restrictions on non-final classes adopting type parameter packs, for the time being the proposal restricts such classes from being the superclass of another class.

An attempt to inherit from a variadic generic class outputs an error. The correct behavior of override checking and initializer inheritance in variadic generic classes will be dealt with in a follow-up proposal:

```swift
class Base<each T> {
  func foo(t: repeat each T) {}
}

// error: cannot inherit from a class with a type parameter pack
class Derived<U, V>: Base<U, V> {
  override func foo(t: U, _: V) {}
}
```

## Source compatibility

Variadic generic types are a new language feature which does not impact source compatibility with existing code.

## ABI compatibility

Variadic type aliases are not part of the binary interface of a module and do not require runtime support.

All other variadic types make use of new entry points and other behaviors being added to the Swift runtime. Since the runtime support requires extensive changes to the type metadata logic, backward deployment to older runtimes is not supported.

Replacing a non-variadic generic type with a variadic generic type is **not** binary-compatible in either direction. When adopting variadic generic types, binary-stable frameworks must introduce them as wholly-new symbols.

## Future directions

A future proposal will address variadic generic enums, and complete support for variadic generic classes.

Another possible future direction is stored property packs, which would eliminate the need to wrap a pack expansion in a tuple type in order to store a variable number of values inside of a variadic type:

```swift
struct S<each T> {
  var a: repeat each Array<T>
}
```

However, there is no expressivity lost in requiring the tuple today, since the contents of a tuple can be converted into a value pack.

## Alternatives considered

The one-parameter limitation could be lifted if we introduced labeled generic parameters at the same time. The choice to allow only a single (unlabeled) generic parameter pack does not preclude this possibility from being explored in the future.

Another alternative is to not enforce the one-parameter limitation at all. There would then exist variadic generic types which cannot be spelled explicitly, but can still be constructed by type inference:

```swift
struct S<each T, each U> {
  init(t: repeat each T, u: repeat each U) {}
}

S(t: 1, "hi", u: false)
```

It was felt that in the end, the single-parameter model is the simplest.

We could require that variadic classes are declared `final`, instead of rejecting subclassing at the point of use. However, since adding or removing `final` on a class is an ABI break, this would preclude the possibility of publishing APIs which work with the existing compiler but can allow subclassing in the future.
