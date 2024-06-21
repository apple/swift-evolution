# Opaque Result Types

* Proposal: [SE-0244](0244-opaque-result-types.md)
* Authors: [Doug Gregor](https://github.com/DougGregor), [Joe Groff](https://github.com/jckarter)
* Review Manager: [Ben Cohen](https://github.com/airspeedswift)
* Status: **Implemented (Swift 5.1)**
* Implementation: [apple/swift#22072](https://github.com/apple/swift/pull/22072)
* Toolchain: https://github.com/apple/swift/pull/22072#issuecomment-483495849
* Previous revisions: ([1](https://github.com/swiftlang/swift-evolution/commit/e60bac23bf0d6f345ddb48fbf64ea8324fce79a9))
* Previous review threads: https://forums.swift.org/t/se-0244-opaque-result-types/21252
* Decision Notes: [Rationale](https://forums.swift.org/t/se-0244-opaque-result-types-reopened/22942/57)

## Introduction

This proposal is the first part of a group of changes we're considering in a [design document for improving the UI of the generics model](https://forums.swift.org/t/improving-the-ui-of-generics/22814). We'll try to make this proposal stand alone to describe opaque return types, their design, and motivation, but we also recommend reading the design document for more in-depth exploration of the relationships among other features we're considering. We'll link to relevant parts of that document throughout this proposal.

This specific proposal addresses the problem of [type-level abstraction for returns](https://forums.swift.org/t/improving-the-ui-of-generics/22814#heading--missing-type-level-abstraction). Many libraries consist of composable generic components. For example, a graphics library might provide primitive types for basic shapes:

```swift
protocol Shape {
  func draw(to: Surface)

  func collides<Other: Shape>(with: Other) -> Bool
}

struct Rectangle: Shape { /* ... */ }
struct Circle: Shape { /* ... */ }
```

along with composable transformations to combine and modify primitive shapes into more complex ones:

```swift
struct Union<A: Shape, B: Shape>: Shape {
  var a: A, b: B
  // ...
}

struct Intersect<A: Shape, B: Shape>: Shape {
  var a: A, b: B
  // ...
}

struct Transformed<S: Shape>: Shape {
  var shape: S
  var transform: Matrix3x3
  // ...
}
```

One could compose these transformations by using the existential type `Shape` instead of generic arguments, but doing so would imply more dynamism and runtime overhead than may be desired. If we directly compose the generic containers, maintaining the concrete types, then generic specialization can more readily optimize the composed operations together, and the type system can also be used. A game or graphics app may want to define objects in terms of their shapes:

```swift
protocol GameObject {
  // The shape of the object
  associatedtype Shape: Shapes.Shape

  var shape: Shape { get }
}
```

However, users of the `GameObject` protocol would now be burdened with writing out long, explicit types for their shapes:

```swift
struct EightPointedStar: GameObject {
  var shape: Union<Rectangle, Transformed<Rectangle>> {
    return Union(Rectangle(), Transformed(Rectangle(), by: .fortyFiveDegrees)
  }
}
```

This is unsightly because it's verbose, but it's also not very helpful for someone reading this declaration. The exact return type doesn't really matter, only the fact that it conforms to `Shape`. Spelling out the return type also effectively reveals most of the implementation of `shape`, making the declaration brittle; clients of `EightPointedStar` could end up relying on its exact return type, making it harder if the author of `EightPointedStar` wants to change how they implement its shape, such as if a future version of the library provides a generic `NPointedStar` primitive. Right now, if you want to abstract the return type of a declaration from its signature, existentials or manual type erasure are your only options, and these [come with tradeoffs](https://forums.swift.org/t/improving-the-ui-of-generics/22814#heading--limits-of-existentials) that are not always acceptable.

## Proposed solution

Instead of declaring the specific return type of `EightPointedStar.shape`'s current implementation,
all we really want to say is that it returns something that conforms to `Shape`. We propose
the syntax `some Protocol`:

```swift
struct EightPointedStar: GameObject {
  var shape: some Shape {
    return Union(Rectangle(), Transformed(Rectangle(), by: .fortyFiveDegrees)
  }
}
```

to declare that an `EightPointedStar` has some `Shape` without having to specify exactly what shape that is. The underlying concrete type is hidden, and can even change from one version of the library to the next without breaking those clients, because the underlying type identity is never exposed to clients. Unlike an existential, though, clients still have access to the type identity. This allows the library to provide a potentially-more-efficient design that leverages Swift's type system, without expanding the surface area of the library or making implementors of the library's protocols rely on exposing verbose implementation types.

An opaque type behaves like a ["reverse generic"](https://forums.swift.org/t/reverse-generics-and-opaque-result-types/21608). In a traditional generic function, the caller decides what types get bound to the callee's generic arguments:

```swift
func generic<T: Shape>() -> T { ... }

let x: Rectangle = generic() // T == Rectangle, chosen by caller
let x: Circle = generic() // T == Circle, chosen by caller
```

An opaque return type can be thought of as putting the generic signature "to the right" of the function arrow; instead of being a type chosen by the caller that the callee sees as abstracted, the return type is chosen by the callee, and comes back to the caller as abstracted:

```swift
// Strawman syntax
func reverseGeneric() -> <T: Shape> T { return Rectangle(...) }

let x = reverseGeneric() // abstracted type chosen by reverseGeneric's implementation
```

Reverse generics are a great mental model for understanding opaque return types, but the notation
is admittedly awkward. We expect the common use case for this feature to be a single return value behind a set of protocol conformances, so we're proposing to start with the more concise `some Shape` syntax:

```swift
// Proposed syntax
func reverseGeneric() -> some Shape { return Rectangle(...) }

let x = reverseGeneric() // abstracted type chosen by reverseGeneric's implementation
```

Following the `some` keyword is a set of constraints on the implicit generic type variable: a class, protocol, `Any`, `AnyObject`, or some composition thereof (joined with `&`).
This `some Protocol` sugar [can be generalized to generic arguments and structural positions in return types](https://forums.swift.org/t/improving-the-ui-of-generics/22814#heading--directly-expressing-constraints) in the future, and we could also eventually support a fully general generic signature for opaque returns. To enable incremental progress on the implementation, we propose starting by only supporting the `some` syntax in return position.

### Type identity

Generics give us an idea of what to expect from opaque return types. The return values from different calls to the same function have the same return type, like two variables of the same generic argument type:

```swift
func foo<T: Equatable>(x: T, y: T) -> some Equatable {
  let condition = x == y // OK to use ==, x and y are the same generic type T
  return condition ? 1738 : 679
}

let x = foo("apples", "bananas")
let y = foo("apples", "some fruit nobody's ever heard of")

print(x == y) // also OK to use ==, x and y are the same opaque return type
```

If the opaque type exposes associated types, those associated types' identities are also maintained. This allows the full API of protocols like the `Collection` family to be used:

```swift
func makeMeACollection<T>(with: T) -> some RangeReplaceableCollection & MutableCollection { ... }

var c = makeMeACollection(with: 17)
c.append(c.first!) // ok: it's a RangeReplaceableCollection
c[c.startIndex] = c.first! // ok: it's a MutableCollection
print(c.reversed()) // ok: all Collection/Sequence operations are available

func foo<C: Collection>(_ : C) { }
foo(c) // ok: C inferred to opaque result type of makeMeACollection<Int>
```

Moreover, opaque result types preserve their identity when composed into other types, such as when forming a collection of the results:

```swift
var cc = [c]
cc.append(c) // ok: cc's Element == the result type of makeMeACollection<Int>
var c2 = makeMeACollection(with: 38)
cc.append(c2) // ok: Element == the result type of makeMeACollection<Int>
```

The opaque return type can however depend on the generic arguments going into the function when
it's called, so the return types of the same function invoked with different generic arguments are
different:

```swift
var d = makeMeACollection(with: "seventeen")
c = d // error: types of makeMeACollection<Int> and makeMeACollection<String> are different
```

Like a generic argument, the static type system does not consider the opaque type to be statically equivalent to the type it happens to be bound to:

```swift
func foo() -> some BinaryInteger { return 219 }
var x = foo()
let i = 912
x = i // error: Int is not known to be the same as the return type as foo()
```

However, one can inspect an opaque type's underlying type at runtime using dynamic casting:

```swift
if let x = foo() as? Int {
  print("It's an Int, \(x)\n")
} else {
  print("Guessed wrong")
}
```

In other words, like generic arguments, opaque result types are only opaque to the static type system. They don't have an independent existence at runtime.

### Implementing a function returning an opaque type

The implementation of a function returning an opaque type must return a value of the same concrete type `T` from each `return` statement, and `T` must meet all of the constraints stated on the opaque type. For example:

```swift
protocol P { }
extension Int : P { }
extension String : P { }

func f1() -> some P {
  return "opaque"
}

func f2(i: Int) -> some P { // ok: both returns produce Int
  if i > 10 { return i }
  return 0
}

func f2(flip: Bool) -> some P {
  if flip { return 17 }
  return "a string" // error: different return types Int and String
}

func f3() -> some P {
  return 3.1419 // error: Double does not conform to P
}

func f4() -> some P {
  let p: P = "hello"
  return p // error: protocol type P does not conform to P
}

func f5() -> some P {
  return f1() // ok: f1() returns an opaque type that conforms to P
}

protocol Initializable { init() }

func f6<T: P & Initializable>(_: T.Type) -> some P {
  return T() // ok: T will always be a concrete type conforming to P
}
```

These rules guarantee that there is a single concrete type produced by any call to the function. The concrete type can depend on the generic type arguments (as in the `f6()` example), but must be consistent across all `return` statements.

Note that recursive calls *are* allowed, and are known to produce a value of the same concrete type, but the concrete type itself is not known:

```swift
func f7(_ i: Int) -> some P {
  if i == 0 {
    return f7(1) // ok: returning our own opaque result type
  } else if i < 0 {
    let result: Int = f7(-i) // error: opaque result type of f7() is not convertible to Int
    return result
  } else {
    return 0 // ok: grounds the recursion with a concrete type
  }
}
```

Of course, there must be at least one `return` statement that provides a concrete type.

Note that a function cannot call itself recursively in a way that forms a type parameterized on the function's own opaque result type, since this would mean that the opaque type is infinitely recursive:

```swift
struct Wrapper<T: P>: P { var value: T }

func f8(_ i: Int) -> some P {
   // invalid; this binds the opaque result type to Wrapper<return type of f8>,
   // which is Wrapper<Wrapper<return type of f8>>, which is
   // Wrapper<Wrapper<Wrapper<...>>>...
  return Wrapper(f8(i + 1))
}
```

A function with an opaque result type is also required to have a `return` statement even if it does not terminate:

```swift
func f9() -> some P {
  fatalError("not implemented")

  // error: no return statement to get opaque type
}
```

This requirement is necessary because, even though `f9`'s return value cannot be reached, the return type of `f9` can still be propagated by local type inference or generic instantiation in ways that don't require evaluating `f9`, so a type for `f9` must be available:

```swift
let delayedF9 = { f9() } // closure has type () -> return type of f9
```

We can't necessarily default the underlying type to `Never`, since `Never` may not
conform to the constraints of the opaque type. If `Never` does conform, and it
is desired as the underlying return type, that can be written explicitly as a
`return` statement:

```swift
extension Never: P {}
func f9b() -> some P {
  return fatalError("not implemented") // OK, explicitly binds return type to Never
}
```

This restriction on non-terminating functions could be something we lift in the future, by synthesizing bottom-type conformances to the protocols required by the opaque type. We leave that as a future extension.

### Properties and subscripts

Opaque result types can also be used with properties and subscripts:

```swift
struct GameObject {
  var shape: some Shape { /* ... */ }
}
```

For computed properties, the concrete type is determined by the `return` statements in the getter. Opaque result types can also be used in stored properties that have an initializer, in which case the concrete type is the type of the initializer:

```swift
let strings: some Collection = ["hello", "world"]
```

Properties and subscripts of opaque result type can be mutable. For example:

```swift
// Module A
public protocol P {
  mutating func flip()
}

private struct Witness: P {
  mutating func flip() { /* ... */ }
}

public var someP: some P = Witness()

// Module B
import A
someP.flip() // ok: flip is a mutating function called on a variable
```

With a subscript or a computed property, the type of the value provided to the setter (e.g., `newValue`) is determined by the `return` statements in the getter, so the type is consistent and known only to the implementation of the property or subscript. For example:

```swift
protocol P { }
private struct Impl: P { }

public struct Vendor {
  private var storage: [Impl] = [/* ... */]

  public var count: Int {
    return storage.count
  }

  public subscript(index: Int) -> some P {
    get {
      return storage[index]
    }
    set (newValue) {
      storage[index] = newValue
    }
  }
}

var vendor = Vendor()
vendor[0] = vendor[2] // ok: can move elements around
```

### Associated type inference

While one can use type inference to declare variables of the opaque result type of a function, there is no direct way to name the opaque result type:

```swift
func f1() -> some P { /* ... */ }

let vf1 = f1() // type of vf1 is the opaque result type of f1()
```

However, type inference can deduce an opaque result type as the associated type of the protocol:

```swift
protocol GameObject {
  associatedtype ObjectShape: Shape

  var shape: ObjectShape
}

struct Player: GameObject {
  var shape: some Shape { /* ... */ }

  // infers typealias Shape = opaque result type of Player.shape
}

let pos: Player.ObjectShape // ok: names the opaque result type of S.someValue()
pos = Player().shape // ok: returns the same opaque result type
```

Note that having a name for the opaque result type still doesn't give information about the underlying concrete type. For example, the only way to create an instance of the type `S.SomeType` is by calling `S.someValue()`.

Associated type inference can only infer an opaque result type for a non-generic requirement, because the opaque type is parameterized by the function's own generic arguments. For instance, in:

```swift
protocol P {
  associatedtype A: P
  func foo<T: P>(x: T) -> A
}

struct Foo: P {
  func foo<T: P>(x: T) -> some P {
    return x
  }
}
```

there is no single underlying type to infer `A` to, because the return type
of `foo` is allowed to change with `T`.

## Detailed design

### Grammar of opaque result types

The grammatical production for opaque result types is straightforward:

```
type ::= opaque-type

opaque-type ::= 'some' type
```

The `type` following the `'some'` keyword is semantically restricted to be a class or existential type, meaning it must consist only of `Any`, `AnyObject`, protocols, or base classes, possibly composed using `&`. This type is used to describe the constraints on the implicit "reverse generic" parameter.

### Restrictions on opaque result types

Opaque result types can only be used as the result type of a function, the type of a variable, or the result type of a subscript. The opaque type must be the entire return type of the function, For example, one cannot return an optional opaque result type:

```swift
func f(flip: Bool) -> (some P)? { // error: `some P` is not the entire return type
  // ...
}
```

This restriction could be lifted in the future.

More fundamentally, opaque result types cannot be used in the requirements of a protocol:

```swift
protocol Q {
  func f() -> some P // error: cannot use opaque result type within a protocol
}
```

Associated types provide a better way to model the same problem, and the requirements can then be satisfied by a function that produces an opaque result type. (There might be an interesting shorthand feature here, where using `some` in a protocol requirement implicitly introduces an associated type, but we leave that for future language design to explore.)

Similarly to the restriction on protocols, opaque result types cannot be used for a non-`final` declaration within a class:

```swift
class C {
  func f() -> some P { /* ... */ } // error: cannot use opaque result type with a non-final method
  final func g() -> some P { /* ... */ } // ok
}
```

This restriction could conceivably be lifted in the future, but it would mean that override implementations would be constrained to returning the same type as their super implementation, meaning they must call `super.method()` to produce a valid return value.

### Uniqueness of opaque result types

Opaque result types are uniqued based on the function/property/subscript and any generic type arguments. For example:

```swift
func makeOpaque<T>(_: T.Type) -> some Any { /* ... */ }
var x = makeOpaque(Int.self)
x = makeOpaque(Double.self) // error: "opaque" type from makeOpaque<Double> is distinct from makeOpaque<Int>
```

This includes any generic type arguments from outer contexts, e.g.,

```swift
extension Array where Element: Comparable {
  func opaqueSorted() -> some Sequence { /* ... */ }
}

var x = [1, 2, 3]. opaqueSorted()
x = ["a", "b", "c"].opaqueSorted() // error: opaque result types for [Int].opaqueSorted() and [String].opaqueSorted() differ
```

### Implementation strategy

From an implementation standpoint, a client of a function with an opaque result type needs to treat values of that result type like any other resilient value type: its size, alignment, layout, and operations are unknown.

However, when the body of the function is known to the client (e.g., due to inlining or because the client is in the same compilation unit as the function), the compiler's optimizer will have access to the specific concrete type, eliminating the indirection cost of the opaque result type.

## Source compatibility

Opaque result types are purely additive. They can be used as a tool to improve long-term source (and binary) stability, by not exposing the details of a result type to clients.

If opaque result types are retroactively adopted in a library, it would initially break source compatibility (e.g., if types like `EnumeratedSequence`, `FlattenSequence`, and `JoinedSequence` were removed from the public API) but could provide longer-term benefits for both source and ABI stability because fewer details would be exposed to clients. There are some mitigations for source compatibility, e.g., a longer deprecation cycle for the types or overloading the old signature (that returns the named types) with the new signature (that returns an opaque result type).

## Effect on ABI stability

Opaque result types are an ABI-additive feature, so do not in and of themselves impact existing ABI. However, additional runtime support is however needed to support instantiating opaque result types across ABI boundaries, meaning that *a Swift 5.1 runtime will be required to deploy code that uses opaque types in public API*. Also, changing an existing API to make use of opaque result types instead of returning concrete types would be an ABI-breaking change, so one of the source compatibility mitigations mentioned above would also need to be deployed to maintain ABI compatibility with existing binary clients.

## Effect on API resilience

Opaque result types are part of the result type of a function/type of a variable/element type of a subscript. The requirements that describe the opaque result type cannot change without breaking the API/ABI. However, the underlying concrete type *can* change from one version to the next without breaking ABI, because that type is not known to clients of the API.

One notable exception to the above rule is `@inlinable`: an `@inlinable` declaration with an opaque result type requires that the underlying concrete type be `public` or `@usableFromInline`. Moreover, the underlying concrete type *cannot be changed* without breaking backward compatibility, because it's identity has been exposed by inlining the body of the function. That makes opaque result types somewhat less compelling for the `compactMap` example presented in the introduction, because one cannot have `compactMap` be marked `@inlinable` with an opaque result type, and then later change the underlying concrete type to something more efficient.

We could allow an API originally specified using an opaque result type to later evolve to specify the specific result type. The result type itself would have to become visible to clients, and this might affect source compatibility, but (mangled name aside) such a change would be resilient.

## Rust's `impl Trait`

The proposed Swift feature is largely inspired by Rust's `impl Trait` language feature, described by [Rust RFC 1522](https://github.com/rust-lang/rfcs/blob/master/text/1522-conservative-impl-trait.md) and extended by [Rust RFC 1951](https://github.com/rust-lang/rfcs/blob/master/text/1951-expand-impl-trait.md). There are only a small number of differences between this feature as expressed in Swift vs. Rust's `impl Trait` as described in RFC 1522:

* Swift's need for a stable ABI necessitates translation of opaque result types as resilient types, which is unnecessary in Rust's model, where the concrete type can always be used for code generation.
* Swift's opaque result types are fully opaque, because Swift doesn't have pass-through protocols like Rust's `Send` trait, which simplifies the type checking problem slightly.
* Due to associated type inference, Swift already has a way to "name" an opaque result type in some cases.

## Alternatives considered

### Return type inference

Part of the motivation of this feature is to avoid having to spell out elaborate return types. This proposal achieves that for types that can be abstracted behind protocols, but in doing so introduces complexity in the form of a new kind of "reverse generic" type. Meanwhile, there are kinds of verbose return types that can't be effectively hidden behind protocol interfaces, like deeply nested collections:

```swift
func jsonBlob() -> [String: [String: [[String: Any]]]] { ... }
```

We could theoretically address the verbosity problem in its full generality and without introducing new type system features  by allowing return types to be inferred, like C++14's or D's `auto` return types:

```swift
func jsonBlob() -> auto { ... }
```

Although this would superficially address the problem of verbose return types, that isn't really the primary problem this proposal is trying to solve, which is to allow for *more precise description of interfaces*. The case of a verbose composed generic adapter type is fundamentally different from a deeply nested collection; in the former case, the concrete type is not only verbose, but it's largely irrelevant, because clients should only care about the common protocols the type conforms to. For a nested collection, the verbose type *is* the interface: the full type is necessary to fully describe how someone interacts with that collection.

Return type inference also has several undesirable traits as a language feature:

- It violates separate compilation of function bodies, since the body of the function must be
  type-checked to infer the return type. Swift already has type inference for
  stored property declarations, and we consider this a mistake, since it has
  been an ongoing source of implementation complexity and performance problems
  due to the need to type-check the property initializer across files. This is
  not a problem for opaque return types, because callers only interface with the
  declaration through the opaque type's constraints. Code can be compiled against a
  function with an opaque return type without having to know what the
  underlying type is; it is "just an optimization" to specialize away the
  opaque type when the underlying type is known.
- Similarly, inferred return types wouldn't provide any semantic abstraction once the type is inferred. Code that calls the function would still see the full concrete type, allowing clients to rely on unintentional details of the concrete type, and the implementation would be bound by ABI and source compatibility constraints if it needed to change the return type. Module interfaces and documentation would also still expose the full return type, meaning they don't get the benefit of the shorter notation.

We see opaque return types as not only sugar for syntactically heavy return types, but also a tool for writing clearer, more resilient APIs. Return type inference would achieve the former but not the latter, while also introducing another compile-time performance footgun into the language.

### Syntax for opaque return types

This proposal suggests the word `some` to introduce opaque return types, since it has the right connotation by analogy to `Any` which is used to describe dynamically type-erased containers (and has been proposed to be a general way of referring to existential types)--a function that has an opaque return type returns *some* specific type
that conforms to the given constraints, whereas an existential can contain *any* type dynamically at any point in time. The spelling would also work well if [generalized to implicit generic arguments](https://forums.swift.org/t/improving-the-ui-of-generics/22814#heading--directly-expressing-constraints) in the future. There are nonetheless reasonable objections to this keyword:

- `some` is already used in the language as one of the case names for `Optional`; it is rare to need to spell `Optional.some` or `.some` explicitly, but it could nonetheless introduce confusion.
- In spoken language, `some type` and `sum type` sound the same.
- `swift some` could be a difficult term to search for. (However, on Google it currently gives [reasonable results](https://www.google.com/search?client=safari&rls=en&q=swift+some&ie=UTF-8&oe=UTF-8) about Optional in Swift.)

Another obvious candidate is `opaque`, following the title of this very proposal. The word "opaque" is itself an overloaded term with existing meaning in many domains, which is unfortunate:

```swift
protocol Shape {}

func translucentRectangle() -> opaque Shape { /* ... */ }
```

The term "opaque" is also fairly heavily overloaded in the Swift implementation (albeit not so much the source-level programming model). It may be that there is a better term, such as "abstract return types", to refer to this feature in its entirety. `opaque` is also not as good a fit for generalization to implicit generic arguments.

In swift-evolution discussion, several other names came up, including:

```
func translucentRectangle() -> unspecified Shape { ... }

func translucentRectangle() -> anyOld Shape { ... }

func translucentRectangle() -> hazyButSpecific Shape { ... }

func translucentRectangle() -> someSpecific Shape { ... }

func translucentRectangle() -> someConcrete Shape { ... }

func translucentRectangle() -> nonspecific Shape { ... }

func translucentRectangle() -> unclear Shape { ... }

func translucentRectangle() -> arbitrary Shape { ... }

func translucentRectangle() -> someThing Shape { ... }

func translucentRectangle() -> anonymized Shape { ... }

func translucentRectangle() -> nameless Shape { ... }
```

In our opinion, most of these are longer and not much clearer, and many wouldn't work well as generalized sugar for arguments and returns.

### Opaque type aliases

As proposed, opaque result types are tied to a specific declaration. They offer no way to state that two related APIs with opaque result types produce the *same* underlying concrete type. The idea of "reverse generics" could however be decoupled from function declarations, if you could write an "opaque typealias" that describes the abstracted interface to a type, while giving it a name, then you could express that relationship across declarations. For example:

```swift
public typealias LazyCompactMapCollection<Elements, ElementOfResult>
  -> <C: Collection> C where C.Element == ElementOfResult
  = LazyMapSequence<LazyFilterSequence<LazyMapSequence<Elements, ElementOfResult?>>, ElementOfResult>
```

In this strawman syntax, the "reverse generic" signature following the `->` is how clients see `LazyCompactMapCollection`. The underlying concrete type, spelled after the `=`, is visible only to the implementation (in some way that would have to be designed). With this feature, multiple APIs could be described as returning a `LazyCompactMapCollection`:

```swift
extension LazyMapCollection {
  public func compactMap<U>(_ transform: @escaping (Element) -> U?) -> LazyCompactMapCollection<Base, U> {
    // ...
 }

  public func filter(_ isIncluded: @escaping (Element) -> Bool) -> LazyCompactMapCollection<Base, Element> {
    // ...
  }
}
```

From the client perspective, both APIs would return the same type, but the specific underlying type would not be known.

```swift
var compactMapOp = values.lazy.map(f).compactMap(g)
if Bool.random() {
  compactMapOp = values.lazy.map(f).filter(h)  // ok: both APIs have the same type
}
```

This would be a great feature to explore as a future direction, since it has some important benefits relative to opaque result types. However, we don't think it's the best place to start with this feature. Compare a declaration using opaque typealiases like:

```swift
func foo<T>() -> ReturnTypeOfFoo<T> { return 1 }

opaque typealias ReturnTypeOfFoo<T> -> <U: P> P = Int
```

to one using opaque return types:

```swift
func foo<T>() -> some P { return 1 }
```

The one using opaque typealiases requires an intermediate name, which one must read and follow to its definition to understand the interface of `foo`. The definition of `ReturnTypeOfFoo` also needs to spell out the underlying concrete return type of `foo`, and the two declarations are tightly coupled; a change to `foo` will likely require a lockstep change to `ReturnTypeOfFoo`. We expect that, in the common use case for this feature, the types being abstracted are going to be tied to specific declarations, and there wouldn't be any better name to really give than "return type of (decl)," so making opaque type aliases the only way of expressing return type abstraction would introduce a lot of obscuring boilerplate.

## Future Directions

As noted in the introduction, this proposal is the first part of a group of changes we're considering in a [design document for improving the UI of the generics model](https://forums.swift.org/t/improving-the-ui-of-generics/22814). That design document lays out a number of related directions we can go based on the foundation establised by this proposal, including:

- allowing [fully generalized reverse generics](https://forums.swift.org/t/improving-the-ui-of-generics/22814#heading--reverse-generics)
- [generalizing the `some` syntax](https://forums.swift.org/t/improving-the-ui-of-generics/22814#heading--directly-expressing-constraints) as shorthand for generic arguments, and allowing structural use in generic returns
- [more compact constraint syntax](https://forums.swift.org/t/improving-the-ui-of-generics/22814#heading--directly-expressing-constraints) that can also work with generalized existentials
- introducing [`any` as a dual to `some`](https://forums.swift.org/t/improving-the-ui-of-generics/22814#heading--clarifying-existentials) for explicitly spelling existential types

We recommend reading that document for a more in-depth exploration of these related ideas.
