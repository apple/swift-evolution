# Implicitly Opened Existentials

* Proposal: [SE-0352](0352-implicit-open-existentials.md)
* Authors: [Doug Gregor](https://github.com/DougGregor)
* Review Manager: [Joe Groff](https://github.com/jckarter)
* Status: **Implemented (Swift 5.7)**
* Upcoming Feature Flag: `ImplicitOpenExistentials` (Implemented in Swift 6.0) (Enabled in Swift 6 language mode)
* Implementation: [apple/swift#41996](https://github.com/apple/swift/pull/41996), [macOS toolchain](https://ci.swift.org/job/swift-PR-toolchain-macos/120/artifact/branch-main/swift-PR-41996-120-osx.tar.gz)
* Decision Notes: [Acceptance](https://forums.swift.org/t/accepted-se-0352-implicitly-opened-existentials/57553)
* Previous Revision: [1](https://github.com/swiftlang/swift-evolution/blob/77374319a7d70c866bd197faada46ecfce461645/proposals/0352-implicit-open-existentials.md)
* Previous Review: [First review](https://forums.swift.org/t/se-0352-implicitly-opened-existentials/56557/52)

## Table of Contents

   * [Introduction](#introduction)
   * [Proposed solution](#proposed-solution)
     * [Moving between any and some](#moving-between-any-and-some)
   * [Detailed design](#detailed-design)
     * [When can we open an existential?](#when-can-we-open-an-existential)
     * [Type-erasing resulting values](#type-erasing-resulting-values)
     * ["Losing" constraints when type-erasing resulting values](#losing-constraints-when-type-erasing-resulting-values)
     * [Contravariant erasure for parameters of function type](#contravariant-erasure-for-parameters-of-function-type)
     * [Order of evaluation restrictions](#order-of-evaluation-restrictions)
     * [Avoid opening when the existential type satisfies requirements (in Swift 5)](#avoid-opening-when-the-existential-type-satisfies-requirements-in-swift-5)
     * [Suppressing explicit opening with as any P / as! any P](#suppressing-explicit-opening-with-as-any-p--as-any-p)
   * [Source compatibility](#source-compatibility)
   * [Effect on ABI stability](#effect-on-abi-stability)
   * [Effect on API resilience](#effect-on-api-resilience)
   * [Alternatives considered](#alternatives-considered)
     * [Explicitly opening existentials](#explicitly-opening-existentials)
     * [Value-dependent opening of existentials](#value-dependent-opening-of-existentials)
   * [Revisions](#revisions)
   * [Acknowledgments](#acknowledgments)

## Introduction

Existential types in Swift allow one to store a value whose specific type is unknown and may change at runtime. The dynamic type of that stored value, which we refer to as the existential's *underlying type*, is known only by the set of protocols it conforms to and, potentially, its superclass. While existential types are useful for expressing values of dynamic type, they are necessarily restricted because of their dynamic nature. Recent proposals have made [existential types more explicit](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0335-existential-any.md) to help developers understand this dynamic nature, as well as [making existential types more expressive](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0309-unlock-existential-types-for-all-protocols.md) by removing a number of limitations. However, a fundamental issue with existential types remains, that once you have a value of existential type it is *very* hard to use generics with it. Developers usually encounter this via the error message "protocol 'P' as a type cannot conform to itself":

```swift
protocol P {
  associatedtype A
  func getA() -> A
}

func takeP<T: P>(_ value: T) { }

func test(p: any P) {
  takeP(p) // error: protocol 'P' as a type cannot conform to itself
}
```

This interaction with the generics system makes existentials a bit of a trap in Swift: it's easy to go from generics to existentials, but once you have an existential it is very hard to go back to using it generically. At worst, you need to go back through many levels of functions, changing their parameters or results from `any P` to being generic over `P`, or writing a custom [type eraser](https://www.swiftbysundell.com/articles/different-flavors-of-type-erasure-in-swift/). 

This proposal addresses this existential trap by allowing one to "open" an existential value, binding a generic parameter to its underlying type. Doing so allows us to call a generic function with an existential value, such that the generic function operates on the underlying value of the existential rather than on the existential box itself, making it possible to get out of the existential trap without major refactoring. This capability already exists in the language when accessing a member of an existential (e.g., `p.getA()`), and this proposal extends that behavior to all call arguments in a manner that is meant to be largely invisible: calls to generic functions that would have failed (like `takeP(p)` above) will now succeed. Smoothing out this interaction between existentials and generics can simplify Swift code and make the language more approachable.

Swift-evolution thread: [Pitch #1](https://forums.swift.org/t/pitch-implicitly-opening-existentials/55412), [Pitch #2](https://forums.swift.org/t/pitch-2-implicitly-opening-existentials/56360)

## Proposed solution

To make it easier to move from existentials back to the more strongly-typed generics, we propose to implicitly *open* an existential value when it is passed to a parameter of generic type. In such cases, the generic argument refers to the *underlying* type of the existential value rather than the existential "box". Let's start with a protocol `Costume` that involves `Self` requirements, and write a generic function that checks some property of a costume:

```swift
protocol Costume {
  func withBells() -> Self
  func hasSameAdornments(as other: Self) -> Bool
}

// Okay: generic function to check whether adding bells changes anything
func hasBells<C: Costume>(_ costume: C) -> Bool {
  return costume.hasSameAdornments(as: costume.withBells())
}
```

This is fine. However, let's write a function that makes sure every costume has bells for the big finale. We run into problems at the boundary between the array of existential values and our generic function:

```swift
func checkFinaleReadiness(costumes: [any Costume]) -> Bool {
  for costume in costumes {
    if !hasBells(costume) { // error: protocol 'Costume' as a type cannot conform to the protocol itself
      return false
    }
  }
  
  return true
}
```

In the call to `hasBells`, the generic parameter `C` is getting bound to the type `any Costume`, i.e., a box that contains a value of some unknown underlying type. Each instance of that box type might have a different type at runtime, so even though the underlying type conforms to `Costume`, the box does not. That box itself does not conform to `Costume` because it does not meet the requirement for `hasSameAdornments`., i.e., two boxes aren't guaranteed to store the same the same underlying type.

This proposal introduces implicitly opened existentials, which allow one to use a value of existential type (e.g., `any Costume`) where its underlying type can be captured in a generic parameter. For example, the call  `hasBells(costume)` above would succeed, binding the generic parameter `C` to the underlying type of that particular instance of `costume`. Each iteration of the loop could have a different underlying type bound to `C`:

```swift
func checkFinaleReadiness(costumes: [any Costume]) -> Bool {
  for costume in costumes {
    if !hasBells(costume) { // okay with this proposal: C is bound to the type stored inside the 'any' box, known only at runtime
      return false
    }
  }
  
  return true
}
```

Implicitly opening existentials allows one to take a dynamically-typed value and give its underlying type a name by binding it to a generic parameter, effectively moving from a dynamically-typed value to a more statically-typed one. This notion isn't actually new: calling a member of a protocol on a value of existential type implicitly "opens" the `Self` type. In the existing language, one could implement a shim for `hasBells` as a member of a protocol extension:

```swift
extension Costume {
  var hasBellsMember: Bool {
    hasBells(self) 
  }
}

func checkFinaleReadinessMember(costumes: [any Costume]) -> Bool {
  for costume in costumes {
    if !costume.hasBellsMember { // okay today: 'Self' is bound to the type stored inside the 'any' box, known only at runtime
      return false
    }
  }
  
  return true
}
```

In that sense, implicitly opening existentials for calls to generic functions is a generalization of this existing behavior to all generic parameters. It isn't strictly more expressive: as the `hasBellsMember` example shows, one *can* always write a member in a protocol extension to get this opening behavior. This proposal aims to make implicit opening of existentials more uniform and more ergonomic, by making it more general.

Let's consider one last implementation of our "readiness" check, where want to "open code" the check for bells without putting the logic into a separate generic function `hasBells`:

```swift
func checkFinaleReadinessOpenCoded(costumes: [any Costume]) -> Bool {
  for costume in costumes {
    let costumeWithBells = costume.withBells() // returned type is 'any Costume'
    if !costume.hasSameAdornments(costumeWithBells) { // error: 'any Costume' isn't necessarily the same type as 'any Costume'
      return false
    }
  }
  
  return true
}
```

There are two things to notice here. First, the method `withBells()` returns type `Self`. When calling that method on a value of type `any Costume`, the concrete result type is not known, so it is type-erased to `any Costume` (which becomes the type of `costumeWithBells`). Second, on the next line, the call to `hasSameAdornments` produces a type error because the function expects a value of type `Self`, but there is no statically-typed link between `costume` and `costumeWithBells`: both are of type `any Costume`. Implicit opening of existential arguments only occurs in calls, so that its effects can be type-erased at the end of the call. To have the effects of opening persist over multiple statements, factor that code out into a generic function that gives a name to the generic parameter, as with `hasBells`.

### Moving between `any` and `some`

One of the interesting aspects of this proposal is that it allows one to refactor `any` parameters into `some` parameters (as introduced by [SE-0341](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0341-opaque-parameters.md)) without a significant effect on client code. Let's rewrite our generic `hasBells` function using `some`:

```swift
func hasBells(_ costume: some Costume) -> Bool {
  return costume.hasSameAdornments(as: costume.withBells())
}
```

With this proposal, we can now call `hasBells` given a value of type `any Costume`:

```swift
func isReadyForFinale(_ costume: any Costume) -> Bool {
  return hasBells(costume) // implicit opening of the existential value
}
```

It's always the case that one can go from a statically-typed `some Costume` to an `any Costume`. This proposal also allows one to go the other way, opening up an `any Costume` into a `some Costume` parameter. Therefore, with this proposal, we could refactor `isReadyForFinale` to make it generic via `some`:

```swift
func isReadyForFinale(_ costume: some Costume) -> Bool {
  return hasBells(costume) // okay, `T` binds to the generic argument
}
```

Any callers to `isReadyForFinale` that provided concrete types now avoid the overhead of "boxing" their type in an `any Costume`, and any callers that provided an `any Costume` will now implicitly open up that existential in the call to `isReadyForFinale`. This allows existential operations to be migrated to generic ones without having to also make all clients generic at the same time, offering an incremental way out of the "existential trap".

## Detailed design

Fundamentally, opening an existential means looking into the existential box to find the dynamic type stored within the box, then giving a "name" to that dynamic type. That dynamic type name needs to be captured in a generic parameter somewhere, so it can be reasoned about statically, and the value with that type can be passed along to the generic function being called. The result of such a call might also refer to that dynamic type name, in which case it has to be erased back to an existential type. The After the call, any values described in terms of that dynamic type opened existential type has to be type-erased back to an existential so that the opened type name doesn't escape into the user-visible type system. This both matches the existing language feature (opening an existential value when accessing one of its members) and also prevents this feature from constituting a major extension to the type system itself.

This section describes the details of opening an existential and then type-erasing back to an existential. These details of this change should be invisible to the user, and manifest only as the ability to use existentials with generics in places where the code would currently be rejected. However, there are a *lot* of details, because moving from dynamically-typed existential boxes to statically-typed generic values must be carefully done to maintain type identity and the expected evaluation semantics.

### When can we open an existential?

To open an existential, the argument (or source) must be of existential type (e.g., `any P`) or existential metatype (e.g., `any P.Type`) and must be provided to a parameter (or target) whose type involves a generic parameter that can bind directly to the underlying type of the existential. This means that, for example, we can open an existential when its underlying type would directly bind to a generic parameter:

```swift
protocol P {
  associatedtype A
  
  func getA() -> A
}

func openSimple<T: P>(_ value: T) { }

func testOpenSimple(p: any P) {
  openSimple(p) // okay, opens 'p' and binds 'T' to its underlying type
}
```

It's also possible to open an `inout` parameter. The generic function will operate on the underlying type, and can (e.g.) call `mutating` methods on it, but cannot change its *dynamic* type because it doesn't have access to the existential box:

```swift
func openInOut<T: P>(_ value: inout T) { }
func testOpenInOut(p: any P) {
  var mutableP: any P = p
  openInOut(&mutableP) // okay, opens to 'mutableP' and binds 'T' to its underlying type
}
```

However, we cannot open when there might be more than one value of existential type or no values at all, because we need to be guaranteed to have a single underlying type to infer. Here are several such examples where the generic parameter is used in multiple places in a manner that prevents opening the existential argument:

```swift
func cannotOpen1<T: P>(_ array: [T]) { .. }
func cannotOpen2<T: P>(_ a: T, _ b: T) { ... }
func cannotOpen3<T: P>(_ values: T...) { ... }

struct X<T> { }
func cannotOpen4<T: P>(_ x: X<T>) { }

func cannotOpen5<T: P>(_ x: T, _ a: T.A) { }

func cannotOpen6<T: P>(_ x: T?) { }

func testCannotOpenMultiple(array: [any P], p1: any P, p2: any P, xp: X<any P>, pOpt: (any P)?) {
  cannotOpen1(array)         // each element in the array can have a different underlying type, so we cannot open
  cannotOpen2(p1, p2)        // p1 and p2 can have different underlying types, so there is no consistent binding for 'T'
  cannotOpen3(p1, p2)        // similar to the case above, p1 and p2 have different types, so we cannot open them
  cannotOpen4(xp)            // cannot open the existential in 'X<any P>' there isn't a specific value there.
  cannotOpen5(p1, p2.getA()) // cannot open either argument because 'T' is used in both parameters
  cannotOpen6(pOpt)          // cannot open the existential in '(any P)?' because it might be nil, so there would not be an underlying type
}
```

The case of optionals is somewhat interesting. It's clear that the call `cannotOpen6(pOpt)` cannot work because `pOpt` could be `nil`, in which case there is no type to bind `T` to. We *could* choose to allow opening a non-optional existential argument when the parameter is optional, e.g.,

```swift
cannotOpen6(p1) // we *could* open here, binding T to the underlying type of p1, but choose not to 
```

but this proposal doesn't allow this because it would be odd to allow this call but not the `cannotOpen6(pOpt)` call.

A value of existential metatype can also be opened, with the same limitations as above.

```swift
func openMeta<T: P>(_ type: T.Type) { }

func testOpenMeta(pType: any P.Type) {
  openMeta(pType) // okay, opens 'pType' and binds 'T' to its underlying type
}
```

### Type-erasing resulting values

The result type of a generic function can involve generic parameters and their associated types. For example, here's a generic function that returns the original value and some values of its associated types: 

```swift
protocol Q { 
  associatedtype B: P
  func getB() -> B
}

func decomposeQ<T: Q>(_ value: T) -> (T, T.B, T.B.A) {
  (value, value.getB(), value.getB().getA())
}
```

When calling `decomposeQ` with an existential value, the existential is opened and `T` will bind to its underlying type. `T.B` and `T.B.A` are types derived from that underlying type. Once the call completes, however, the types `T`, `T.B`, and `T.B.A` are *type-erased* to their upper bounds, i.e., the existential type that captures all of their requirements. For example:

```swift
func testDecomposeQ(q: any Q) {
  let (a, b, c) = decomposeQ(q) // a is any Q, b is any P, c is Any
}
```

This is identical to the [covariant erasure of associated types described in SE-0309](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0309-unlock-existential-types-for-all-protocols.md#covariant-erasure-for-associated-types), and the rules specified there apply equally here. We can restate those requirements more generally for an arbitrary generic parameter as:

When binding a generic parameter `T` to an opened existential, `T`, `T` and `T`-rooted associated types that

- are **not** bound to a concrete type, and
- appear in covariant position within the result type of the generic function

will be type-erased to their upper bounds as per the generic signature of the existential that is used to access the member. The upper bounds can be either a class, protocol, protocol composition, or `Any`, depending on the *presence* and *kind* of generic constraints on the associated type. 

When `T` or a `T`-rooted associated type appears in a non-covariant position in the result type, `T` cannot be bound to the underlying type of an existential value because there would be no way to represent the type-erased result. This is essentially the same property as descibed for the parameter types that prevents opening of existentials, as described above. For example:

```swift
func cannotOpen7<T: P>(_ value: T) -> X<T> { /*...*/ }
```

However, because the return value is permitted a conversion to erase to an existential type, optionals, tuples, and even arrays *are* permitted:

```swift
func openWithCovariantReturn1<T: Q>(_ value: T) -> T.B? { /*...*/ }
func openWithCovariantReturn2<T: Q>(_ value: T) -> [T.B] { /*...*/ }

func covariantReturns(q: any Q){
  let r1 = openWithCovariantReturn1(q)  // okay, 'T' is bound to the underlying type of 'q', resulting type is 'any P'
  let r2 = openWithCovariantReturn2(q)  // okay, 'T' is bound to the underlying type of 'q', resulting type is '[any Q]'
}
```

### "Losing" constraints when type-erasing resulting values

When the result of a call involving an opened existential is type-erased, it is possible that some information about the returned type cannot be expressed in an existential type, so the "upper bound" described above will lose information. For example, consider the type of `b` in this example:

```swift
protocol P {
  associatedtype A
}

protocol Q {
  associatedtype B: P where B.A == Int
}

func getBFromQ<T: Q>(_ q: T) -> T.B { ... }

func eraseQAssoc(q: any Q) {
  let b = getBFromQ(q)
}
```

When type-erasing `T.B`, the most specific upper bound would be "a type that conforms to `P` where the associated type `A` is known to be `Int`". However, Swift's existential types cannot express such a type, so the type of `b` will be the less-specific `any P`.

It is likely that Swift's existentials will grow in expressivity over time. For example, [SE-0353 "Constrained Existential Types"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0353-constrained-existential-types.md) allows one to express existential types that involve bindings for [primary associated types](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0346-light-weight-same-type-syntax.md). If we were to adopt that feature for protocol `P`, the most specific upper bound would be expressible:

```swift
// Assuming SE-0353...
protocol P<A> {
  associatedtype A
}

// ... same as above ...
```

Now, `b` would be expected to have the type `any P<Int>`. Future extensions of existential types might make the most-specific upper bound expressible even without any source code changes, and one would expect that the type-erasure after calling a function with an implicitly-opened existential would become more precise when those features are added.

However, this kind of change presents a problem for source compatibility, because code might have come to depend on the type of `b` being the less-precise `any P` due to, e.g., overloading:

```swift
func f<T: P>(_: T) -> Int { 17 }
func f<T: P>(_: T) -> Double where T.A == Int { 3.14159 }

// ...
func eraseQAssoc(q: any Q) {
  let b = getBFromQ(q)
  f(b)
}
```

With the less-specific upper bound (`any P`), the call `f(b)` would choose the first overload that returns an `Int`. With the more-specific upper bound (`any P` where `A` is known to be `Int`), the call `f(b)` would choose the second overload that returns a `Double`. 

Due to overloading, the source-compatibility impacts of improving the upper bound cannot be completely eliminated without (for example) holding the upper bound constant until a new major language version. However, we propose to mitigate the effects by requiring a specific type coercion on any call where the upper bound is unable to express some requirements due to limitations on existentials. Specifically, the call `getBFromQ(q)` would need to be written as:

```swift
getBFromQ(q) as any P
```

This way, if the upper bound changes due to increased expressiveness of existential types in the language, the overall expression will still produce a value of the same type---`any P`---as it always has. A developer would be free to remove the `as any P` at the point where Swift can fully capture all of the information known about the type in an existential.

Note that this requirement for an explicit type coercion also applies to all type erasure due to existential opening, including ones that existed prior to this proposal. For example, `getBFromQ` could be written as a member of a protocol extension. The code below has the same issues (and the same resolution) as our example, as was first made well-formed with [SE-0309](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0309-unlock-existential-types-for-all-protocols.md):

```swift
extension Q {
  func getBFromQ() -> B { ... }
}

func eraseQAssocWithSE0309(q: any Q) {
  let b = q.getBFromQ()
}
```

### Contravariant erasure for parameters of function type

While covariant erasure applies to the result type of a generic function, the opposite applies to other parameters of the generic function. This affects parameters of function type that reference the generic parameter binding to the opened existential, which will be type-erased to their upper bounds. For example:

```swift
func acceptValueAndFunction<T: P>(_ value: T, body: (T) -> Void) { ... }

func testContravariantErasure(p: any P) {
  acceptValueAndFunction(p) { innerValue in        // innerValue has type 'any P'
    // ... 
  }
}
```

Like the covariant type erasure applied to result types, this type erasure ensures that the "name" assigned to the dynamic type doesn't escape into the user-visible type system through the inferred closure parameter. It effectively maintains the illusion that the generic type parameter `T` is binding to `any P`, while in fact it is binding to the underlying type of that specific value.

There is one exception to this rule: if the argument to such a parameter is a reference to a generic function, the type erasure does not occur. In such cases, the dynamic type name is bound directly to the generic parameter of this second generic function, effectively doing the same implicit opening of existentials again. This is best explained by example:

```swift
func takeP<U: P>(_: U) -> Void { ... }

func implicitOpeningArguments(p: any P) {
  acceptValueAndFunction(p, body: takeP) // okay: T and U both bind to the underlying type of p
}
```

This behavior subsumes most of the behavior of the hidden `_openExistential` operation, which specifically only supports opening one existential value and passing it to a generic function. `_openExistential` might still have a few scattered use cases when opening an existential that doesn't have conformance requirements on it.

 ### Order of evaluation restrictions

Opening an existential box requires evaluating that the expression that produces that box and then peering inside it to extract its underlying type. The evaluation of the expression might have side effects, for example, if one calls the following `getP()` function to produce a value of existential box type `any P`:

```swift
extension Int: P { }

func getP() -> any P {
  print("getP()")
  return 17
}
```

Now consider a generic function for which we want open an existential argument:

```swift
func acceptFunctionStringAndValue<T: P>(body: (T) -> Void, string: String, value: T) { ... }

func hello() -> String {
  print("hello()")
}

func implicitOpeningArgumentsBackwards() {
  acceptFunctionStringAndValue(body: takeP, string: hello(), value: getP()) // will be an error, see later
}
```

Opening the argument to the `value` parameter requires performing the call to `getP()`. This has to occur *before* the argument to the `body` parameter can be formed, because `takeP`'s generic type parameter `U` is bound to the underlying type of that existential box. Doing so means that the program would produce side effects in the following order:

```
getP()
hello()
```

However, this would contradict Swift's longstanding left-to-right evaluation order. Rather than do this, we instead place another limitation on the implicit opening of existentials: an existential argument cannot be opened if the generic type parameter bound to its underlying type is used in any function parameter preceding the one corresponding to the existential argument. In the `implicitOpeningArgumentsBackwards` above, the call to `acceptFunctionStringAndValue` does not permit opening the existential argument to the `value` parameter because its generic type parameter, `T`, is also used in the `body` parameter that precedes `value`. This ensures that the underlying type  is not needed for any argument prior to the opened existential argument, so the left-to-right evaluation order is maintained.

### Avoid opening when the existential type satisfies requirements (in Swift 5)

As presented thus far, opening of existential values can change the behavior of existing programs that relied on passing the existential box to a generic function. For example, consider the effect of passing an existential box to an unconstrained generic function that puts the parameter into the returned array:

```swift
func acceptsBox<T>(_ value: T) -> Any { [value] }

func passBox(p: any P) {
  let result = acceptsBox(p) // currently infers 'T' to be 'any P', returns [any P]
      // unrestricted existential opening would infer 'T' to be the underlying type of 'p', returns [T]
}
```

Here, the dynamic type of the result of `acceptsBox` would change if the existential box is opened as part of the call. The change itself is subtle, and would not be detected until runtime, which could cause problems for existing Swift programs that rely on binding generic parameters. Therefore, in Swift 5, this proposal prevents opening of existential values when the existential types themselves would satisfy the conformance requirements of the corresponding generic parameter, making it a strictly additive change: calls to generic functions with existential values that previously worked will continue to work with the same semantics, but calls that didn't work before will open the existential and can therefore succeed.

Most of the cases in today's Swift where a generic parameter binds to an existential type succeed because there are no conformance requirements on the generic parameter, as with the `T` generic parameter to `acceptsBox`. For most protocols, an existential referencing the corresponding type does not conform to that protocol, i.e., `any Q` does not conform to `Q`. However, there are a small number of exceptions:

* The existential type `any Error` conforms to the `Error` protocol, as specified in [SE-0235](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0235-add-result.md#adding-swifterror-self-conformance).
* An existential type `any Q` of an `@objc` protocol `Q`, where `Q` contains no `static` requirements, conforms to `Q`.

For example, consider an operation that takes an error. Passing a value of type `any Error` to it succeeds without opening the existential:

```swift
func takeError<E: Error>(_ error: E) { }

func passError(error: any Error) {
  takeError(error)  // okay without opening: 'E' binds to 'any Error' because 'any Error' conforms to 'Error'
}
```

This proposal preserves the semantics of the call above by not opening the existential argument in cases where the existential type satisfies the corresponding generic parameter's conformance requirements according to the results above. Should Swift eventually grow a mechanism to make existential types conform to protocols (e.g., so that `any Hashable` conforms to `Hashable`), then such conformances will **not** suppress implicit opening, because any code that made use of these conformances would be newly-valid code and would start with implicit-opening semantics.

Swift 6 will be a major language version change that can incorporate some semantics- and source-breaking changes. In Swift 6, the suppression mechanism described in this section will *not* apply, so the `passBox` example above would open the value of `p` and bind `T` to that opened existential type. This provides a more consistent semantics that, additionally, subsumes all of the behavior of `type(of:)` and the hidden `_openExistential` operation.

### Suppressing explicit opening with `as any P` / `as! any P`

If for some reason one wants to suppress the implicit opening of an existential value, one can explicitly write a coercion or forced cast to an existential type directly on the call argument. For example:

```swift
func f1<T: P>(_: T) { }   // #1
func f1<T>(_: T) { }      // #2

func test(p: any P) {
  f1(p)            // opens p and calls #1, which is more specific
  f1(p as any P)   // suppresses opening of 'p', calls #2 which is the only valid candidate
  f1((p as any P)) // parentheses disable this suppression mechanism, so this opens p and calls #1
}
```

Given that implicit opening of existentials is defined to occur in those cases where a generic function would not otherwise be callable, this suppression mechanism should not be required often in Swift 5. In Swift 6, where implicit opening will be more eagerly performed, it can be used to provide the Swift 5 semantics.

An extra set of parentheses will disable this suppression mechanism, which can be important when `as any P` is required for some other reason. For example, because it acknowledges when information is lost from the result type due to type erasure. This can help break ambiguities when both meanings of `as` could apply:

```swift
protocol P {
  associatedtype A
}
protocol Q {
  associatedtype B: P where B.A == Int
}

func getP<T: P>(_ p: T)
func getBFromQ<T: Q>(_ q: T) -> T.B { ... }

func eraseQAssoc(q: any Q) {
  getP(getBFromQ(q))          // error, must specify "as any P" due to loss of constraint T.B.A == Int
  getP(getBFromQ(q) as any P) // suppresses error above, but also suppresses opening, so it produces
                              // error: now "any P does not conform to P" and op
  getP((getBFromQ(q) as any P)) // okay! original error message should suggest this
}

```

## Source compatibility

This proposal is defined specifically to avoid most impacts on source compatibility, especially in Swift 5. Some calls to generic functions that would previously have been ill-formed (e.g., they would fail because `any P` does not conform to `P`) will now become well-formed, and existing code will behavior in the same manner as before. As with any such change, it's possible that overload resolution that would have succeeded before will continue to succeed but will now pick a different function. For example:

```swift
protocol P { }

func overloaded1<T: P, U>(_: T, _: U) { } // A
func overloaded1<U>(_: Any, _: U) { }     // B

func changeInResolution(p: any P) {
  overloaded1(p, 1) // used to choose B, will choose A with this proposal
}
```

Such examples are easy to construct in the abstract for any feature that makes ill-formed code well-formed, but these examples rarely cause problems in practice.

## Effect on ABI stability

This proposal changes the type system but has no ABI impact whatsoever.

## Effect on API resilience

This proposal changes the use of APIs, but not the APIs themselves, so it doesn't impact API resilience per se.

## Alternatives considered

This proposal opts to open existentials implicitly and locally, type-erasing back to existentials after the immediate call, as a generalization of opening when using a member of an existential value. There are alternative designs that are explicit or open the existential more broadly, with different tradeoffs.

### Explicitly opening existentials

This proposal implicitly opens existentials at call sites. Instead, we could provide an explicit syntax for opening an existential, e.g., via [an `as` coercion to `some P`](https://forums.swift.org/t/pitch-implicitly-opening-existentials/55412/8). For example,

```swift
protocol P {
  associatedtype A
}

func takesP<T: P>(_ value: T) { }

func hasExistentialP(p: any P) {
  takesP(p) // error today ('any P' does not conform to 'P'), would be well-formed with implicit opening
}
```

could be written to explicitly open the existential, e.g.,

```swift
func hasExistentialP(p: any P) {
  takesP(p)               // error today ('any P' does not conform to 'P'), would still be an error
  takesP(p as some P)     // explicitly open the existential
}
```

There are two advantages to this approach over the implicit opening in this proposal. The first is that it is a purely additive feature and completely opt-in feature, which one can read and reason about when it is encountered in source code. The second is that the opened existential could persist throughout the body of the function. This would allow one to write the "open-coded" finale check from earlier in the proposal without having to factor the code into a separate (generic) function:

```swift
func checkFinaleReadinessOpenCoded(costumes: [any Costume]) -> Bool {
  for costume in costumes {
    let openedCostume = costume as some Costume             // type is "opened type of costume at this point"
    let costumeWithBells = openedCostume.withBells()        // returned type is the same as openedCostume
    if !openedCostume.hasSameAdornments(costumeWithBells) { // okay, both types are known to be the same
      return false
    }
  }

  return true
}
```

The type of `openedCostume` is based on the dynamic type of the the value in the variable `costume` at the point where the `as some Costume` expression occurred. That type must not be allowed to "escape" the scope where the value is created, which implies several restrictions:

* Only non-`static` local variables can have opened existential type. Any other kind of variable can be referenced at some later point in time where the dynamic type might have changed.
* A value of opened existential type cannot be returned from a function that has an opaque result type (e.g., `some P`), because then the underlying type of the opaque type would be dependent on runtime values provided to the function.

Additionally, having an explicit opening expression means that opened existential types become part of the user-visible type system: the type of `openedCostume` can only be reasoned about based on its constraints (`P`) and the location in the source code where the expression occurred. Two subsequent openings of the same variable would produce two different types:

```swift
func f(eq: any Equatable) {
  let x1 = eq as some Equatable
  if x1 == x1 { ... }  // okay

  let x2 = eq as some Equatable
  if x1 == x2 { ... } // error: "eq as some Equatable" produces different types in x1 and x2
}
```

An explicit opening syntax is more expressive within a single function than the proposed implicit opening, because one can work with different values that are statically known to be derived from the same opened existential without having to introduce a new generic function to do so. However, this explicitness comes with a corresponding increase in the surface area of the language: not only the expression that performs the explicit opening (`as some P`), but the notion of opened types in the type system, which has heretofore been an implementation detail of the compiler not exposed to users.

In contrast, the proposed implicit opening improves the expressivity of the language without increasing it's effective surface area. The opening is implicit, and the opened types remain an implementation detail.

This "alternative Considered" could perhaps be expressed as a potential future direction. Nothing in this proposal prevents us from adding explicitly opened existentials in the future, should they prove to be useful, and we would still want the implicitly opening with type erasure as described in this proposal. Should that happen, the implicit behavior in this proposal could be retroactively understood as inferring something that could be written in the explicit syntax:

```swift
protocol Q { }

protocol P {
  associatedtype A: Q
}

func getA<T: P>(_ value: T) -> T.A { ... }

func unwrap(p: any P) {
  let a = getA(p) // implicitly the same as "getA(p as some P) as any Q"
}
```

### Value-dependent opening of existentials

Implicit opening in this proposal is always scoped to a particular binding of a specific generic parameter (`T`) and is erased thereafter. For example, this means that two invocations of the same generic function on the same existential value will return values of existential type that are not (statically) known to be equivalent:

```swift
func identity<T: Equatable>(_ value: T) -> T { value }
func testIdentity(p: any Equatable) {
  let p1 = identity(p)   // p1 gets type-erased type 'any Equatable'
  let p2 = identity(p)   // p2 gets type-erased type 'any Equatable'
  if p1 == p2 { ... }    // error: p1 and p2 aren't known to have the same concrete type
  
  let openedP1: some P = identity(p)   // openedP1 has an opaque type binding to the underlying type of the call
  let openedP2: some P = identity(p)   // openedP2 has an opaque type binding to the underlying type of the call
  if openedP1 == openedP2 { ... }      // error: openedP1 and openedP2 aren't known to have the same concrete type
}
```

One could imagine tying the identity of the opened existential type to the *value* of the existential. For example, the two calls to `identity(p)` could produce opaque types that are identical because they are based on the underlying type of the value `p`. This is a form of dependent typing, because the (static) types of some entities are determined by their values. It begins to break down if there is any way in which the value can change, e.g.,

```swift
func identityTricks(p: any Equatable) {
  let openedP1 = identity(p)      // openedP1 has the underlying type of 'p'
  let openedP2 = identity(p)      // openedP2 has the underlying type of 'p'
  if openedP1 == openedP2 { ... } // okay because both values have the underlying type of 'p'
  
  var q = p                          // q has the underlying type of 'p'
  let openedQ1: some P = identity(q) // openedQ1 has the underlying type of 'q' and therefore 'p'
  if openedP1 == openedQ1 { ... }    // okay because both values have the underlying type of 'p'

  if condition {
    q = 17   // different underlying type for 'q'
  }
  
  let openedQ2: some P = identity(q)
  if openedQ1 == openedQ2 { }  // error: openedQ1 has the underlying type of 'p', but
                               // openedQ2 has the underlying type of 'q', which now might be different from 'p'
}
```

This approach is much more complex because it introduces value tracking into the type system (where was this existential value produced?), at which point mutations to variables can affect the static types in the system. 

## Revisions

Fifth revision:

* Note that parentheses disable the `as any P` suppression mechanism, avoiding the problem where `as any P` is both required (because type erasure lost information from the return type) and also has semantic effect (suppressing opening).

Fourth revision:

* Add discussion about type erasure losing constraints and the new requirement to introduce an explicit `as` coercion when the upper bound loses information.

Third revision:

* Only apply the source-compatibility rule, which avoids opening an existential argument when the existential box would have sufficed, in Swift 5. In Swift 6, we will open the existential argument whenever we can, providing a consistent and desirable semantics.
* Re-introduce `as any P` and `as! any P` , now that they will be useful in Swift 6.
* Clarify more about the relationship to the explicit opening syntax, which could also be a future direction.

Second revision:

* Remove the discussion about `type(of:)`, whose special behavior is no longer subsumed by this proposal. Weaken statements about fully subsuming `_openExistential`.
* Removed `as any P` and `as! any P` as syntaxes to suppress the implicit opening of an existential value. It isn't needed given that we only open when the existential type doesn't meet the generic function's constraints.

First revision:

* Describe contravariant erasure for parameters
* Describe the limitation on implicit existential opening to maintain order of evaluation
* Avoid opening an existential argument when the existential type already satisfies the conformance requirements of the corresponding generic parameter, to better maintain source compatibility 
* Introduce `as any P` and `as! any P` as syntaxes to suppress the implicit opening of an existential value.
* Added discussion on the relationship with `some` parameters ([SE-0341](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0341-opaque-parameters.md)).
* Expand discussion of an explicit opening syntax.

## Acknowledgments

This proposal builds on the difficult design work of [SE-0309](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0309-unlock-existential-types-for-all-protocols.md), which charted most of the detailed semantics for working with values of existential type and dealing with (e.g.) covariant erasure and the restrictions that must be placed on opening existentials. Moreover, the implementation work from one of SE-0309's authors, [Anthony Latsis](https://github.com/AnthonyLatsis), formed the foundation of the implementation work for this feature, requiring only a small amount of generalization. Ensan highlighted the issue with losing information in upper bounds and [suggested an approach](https://forums.swift.org/t/se-0352-implicitly-opened-existentials/56557/7) similar to what is used here.
