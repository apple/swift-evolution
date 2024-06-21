# Opaque Parameter Declarations

* Proposal: [SE-0341](0341-opaque-parameters.md)
* Author: [Doug Gregor](https://github.com/DougGregor)
* Review Manager: [Ben Cohen](https://github.com/AirspeedSwift)
* Status: **Implemented (Swift 5.7)**
* Implementation: [apple/swift#40993](https://github.com/apple/swift/pull/40993)

## Introduction

Swift's syntax for generics is designed for generality, allowing one to express complicated sets of constraints amongst the different inputs and outputs of a function. For example, consider an eager concatenation operation that builds an array from two sequences:

```swift
func eagerConcatenate<Sequence1: Sequence, Sequence2: Sequence>(
    _ sequence1: Sequence1, _ sequence2: Sequence2
) -> [Sequence1.Element] where Sequence1.Element == Sequence2.Element
```

There is a lot going on in that function declaration: the two function parameters are of different types determined by the caller, which are captured by `Sequence1` and `Sequence2`, respectively. Both of these types must conform to the `Sequence` protocol and, moreover, the element types of the two sequences must be equivalent. Finally, the result of this operation is an array of the sequence's element type. One can use this operation with many different inputs, so long as the constraints are met: 

```swift
eagerConcatenate([1, 2, 3], Set([4, 5, 6]))  // okay, produces an [Int]
eagerConcatenate([1: "Hello", 2: "World"], [(3, "Swift"), (4, "!")]) // okay, produces an [(Int, String)]
eagerConcatenate([1, 2, 3], ["Hello", "World"]) // error: sequence element types do not match
```

However, when one does not need to introduce a complex set of constraints, the syntax starts to feel quite heavyweight. For example, consider a function that composes two SwiftUI views horizontally:

```swift
func horizontal<V1: View, V2: View>(_ v1: V1, _ v2: V2) -> some View {
  HStack {
    v1
    v2
  }
}
```

There is a lot of boilerplate to declare the generic parameters `V1` and `V2` that are only used once, making this function look far more complex than it really is. The result, on the other hand, is able to use an [opaque result type](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0244-opaque-result-types.md) to hide the specific returned type (which would be complicated to describe), describing it only by the protocols to which it conforms.

This proposal extends the syntax of opaque result types to parameters, allowing one to specify function parameters that are generic without the boilerplate associated with generic parameter lists. The `horizontal` function above can then be expressed as:

```swift
func horizontal(_ v1: some View, _ v2: some View) -> some View {
  HStack {
    v1
    v2
  }
}
```

Semantically, this formulation is identical to the prior one, but is simpler to read and understand because the inessential complexity from the generic parameter lists has been removed. It takes two views (the concrete type does not matter) and returns a view (the concrete type does not matter).

Swift-evolution threads: [Pitch for this proposal](https://forums.swift.org/t/pitch-opaque-parameter-types/54914), [Easing the learning curve for introducing generic parameters](https://forums.swift.org/t/discussion-easing-the-learning-curve-for-introducing-generic-parameters/52891), [Improving UI of generics pitch](https://forums.swift.org/t/improving-the-ui-of-generics/22814)

## Proposed solution

This proposal extends the use of the `some` keyword to parameter types for function, initializer, and subscript declarations. As with opaque result types, `some P` indicates a type that is unnamed and is only known by its constraint: it conforms to the protocol `P`. When an opaque type occurs within a parameter type, it is replaced by an (unnamed) generic parameter. For example, the given function:

```swift
func f(_ p: some P) { }
```

is equivalent to a generic function described as follows, with a synthesized (unnamable) type parameter `_T`:

```swift
func f<_T: P>(_ p: _T)
```

Note that, unlike with opaque result types, the caller determines the type of the opaque type via type inference. For example, if we assume that both `Int` and `String` conform to `P`,  one can call or reference the function with either `Int` or `String`:

```swift
f(17)      // okay, opaque type inferred to Int
f("Hello") // okay, opaque type inferred to String

let fInt: (Int) -> Void = f       // okay, opaque type inferred to Int
let fString: (String) -> Void = f // okay, opaque type inferred to String
let fAmbiguous = f                // error: cannot infer parameter for `some P` parameter
```

[SE-0328](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0328-structural-opaque-result-types.md) extended opaque result types to allow multiple uses of `some P` types within the result type, in any structural position. Opaque types in parameters permit the same structural uses, e.g.,

```swift
func encodeAnyDictionaryOfPairs(_ dict: [some Hashable & Codable: Pair<some Codable, some Codable>]) -> Data
```

This is equivalent to:

```swift
func encodeAnyDictionaryOfPairs<_T1: Hashable & Codable, _T2: Codable, _T3: Codable>(_ dict: [_T1: Pair<_T2, _T3>]) -> Data
```

Each instance of `some` within the declaration represents a different implicit generic parameter.

## Detailed design

Opaque parameter types can only be used in parameters of a function, initializer, or subscript declaration. They cannot be used in (e.g.) a typealias or any value of function type. For example:

```swift
typealias Fn = (some P) -> Void    // error: cannot use opaque types in a typealias
let g: (some P) -> Void = f        // error: cannot use opaque types in a value of function type
```

There are additional restrictions on the use of opaque types in parameters where they may conflict with future language features.

### Variadic generics

An opaque type cannot be used in a variadic parameter:

```swift
func acceptLots(_: some P...)
```

This restriction is in place because the semantics implied by this proposal might not be the appropriate semantics if Swift gains variadic generics. Specifically, the semantics implied by this proposal itself (without variadic generics) would be equivalent to:

```swift
func acceptLots<_T: P>(_: _T...)
```

where `acceptLots` requires that all of the arguments have the same type:

```swift
acceptLots(1, 1, 2, 3, 5, 8)          // okay
acceptLots("Hello", "Swift", "World") // okay
acceptLots("Swift", 6)                // error: argument for `some P` could be either String or Int
```

With variadic generics, one might instead make the implicit generic parameter a generic parameter pack, as follows:

```swift
func acceptLots<_Ts: P...>(_: _Ts...)
```

In this case, `acceptLots` accepts any number of arguments, all of which might have different types:

```swift
acceptLots(1, 1, 2, 3, 5, 8)          // okay, Ts contains six Int types
acceptLots("Hello", "Swift", "World") // okay, Ts contains three String types
acceptLots(Swift, 6)                  // okay, Ts contains String and Int
```

### Opaque parameters in "consuming" positions of function types

The resolution of [SE-0328](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0328-structural-opaque-result-types.md) prohibited the use of opaque parameters in "consuming" positions of function types. For example:

```swift
func f() -> (some P) -> Void { ... } // error: cannot use opaque type in parameter of function type
```

The result of function `f` is fairly hard to use, because there is no way for the caller to easily create a value of an unknown, unnamed type:

```swift
let fn = f()
fn(/* how do I create a value here? */)
```

The same prohibition applies to opaque types that occur within parameters of function type, e.g.,

```swift
func g(fn: (some P) -> Void) { ... } // error: cannot use opaque type in parameter of function type
```

The reasoning for this prohibition is similar. In the implementation of `g`, it's hard to produce a value of the type `some P` when that type isn't named anywhere else.

## Source compatibility

This is a pure language extension with no backward-compatibility concerns, because all uses of `some` in parameter position are currently errors.

## Effect on ABI stability

This proposal has no effect on the ABI or runtime because it is syntactic sugar for generic parameters.

## Effect on API resilience

This feature is purely syntactic sugar, and one can switch between using opaque parameter types and the equivalent formulation with explicit generic parameters without breaking either the ABI or API. However, the complete set of constraints must be the same in such cases.

## Future Directions

### Constraining the associated types of a protocol

This proposal composes well with an idea that allows the use of generic syntax to specify the associated type of a protocol, e.g., where `Collection<String>`is "a `Collection` whose `Element` type is `String`". Combined with this proposal, one can more easily express a function that takes an arbitrary collection of strings:

```swift
func takeStrings(_: some Collection<String>) { ... }
```

Recall the complicated `eagerConcatenate` example from the introduction:

```swift
func eagerConcatenate<Sequence1: Sequence, Sequence2: Sequence>(
    _ sequence1: Sequence1, _ sequence2: Sequence2
) -> [Sequence1.Element] where Sequence1.Element == Sequence2.Element
```

With opaque parameter types and generic syntax on protocol types, one can express this in a simpler form with a single generic parameter representing the element type:

```swift
func eagerConcatenate<T>(
    _ sequence1: some Sequence<T>, _ sequence2: some Sequence<T>
) -> [T]
```

And in conjunction with opaque result types, we can hide the representation of the result, e.g.,

```swift
func lazyConcatenate<T>(
    _ sequence1: some Sequence<T>, _ sequence2: some Sequence<T>
) -> some Sequence<T>
```

### Enabling opaque types in consuming positions

The prohibition on opaque types in "consuming" positions could be lifted for opaque types both in parameters and in return types, but they wouldn't be useful with their current semantics because in both cases the wrong code (caller vs. callee) gets to choose the parameter. We could enable opaque types in consuming positions by "flipping" who gets to choose the parameter. To understand this, think of opaque result types as a form of "reverse generics", where there is a generic parameter list after a function's `->` and for which the function itself (the callee) gets to choose the type. For example:

```swift
func f1() -> some P { ... }
// translates to "reverse generics" version...
func f1() -> <T: P> T { /* callee implementation here picks concrete type for T */ }
```

The problem with opaque types in consuming positions of the return type is that the callee picks the concrete type, and the caller can't reason about it. We can see this issue by translating to the reverse-generics formulation:

```swift
func f2() -> (some P) -> Void { ... }
// translates to "reverse generics" version...
func f2() -> <T: P> (T) -> Void { /* callee implementation here picks concrete type for T */}
```

We could "flip" the caller/callee choice here by translating opaque types in consuming positions to the other side of the `->`. For example, `f2` would be translated into

```swift
// if we "flip" opaque types in consuming positions
func f2() -> (some P) -> Void { ... }
// translates to
func f2<T: P>() -> (T) -> Void { ... }
```

This is a more useful translation, because the caller picks the type for `T` using type context, and the callee provides a closure that can work with whatever type the caller picks, generically. For example:

```swift
let fn1: (Int) -> Void == f2 // okay, T == Int
let fn2: (String) -> Void = f2 // okay, T == String
```

Similar logic applies to opaque types in consuming positions within parameters. Consider this function:

```swift
func g2(fn: (some P) -> Void) { ... }
```

If this translates to "normal" generics, i.e., then the parameter isn't readily usable:

```swift
// if we translated to "normal" generics
func g2<T: P>(fn: (T) -> Void) { /* how do we come up with a T to call fn with? */}
```

Again, the problem here is that the caller gets to choose what `T` is, but then the callee cannot use it effectively. We could again "flip" the generics, moving the implicit type parameter for an opaque type in consuming position to the other side of the function's `->`:

```swift
// if we "flip" opaque types in consuming positions
func g2(fn: (some P) -> Void) { ... }
// translates to
func g2(fn: (T) -> Void) -> <T> Void { ... }
```

Now, the implementation of  `g2` (the callee) gets to choose the type of `T`, which is appropriate because it will be providing values of type `T` to `fn`. The caller will need to provide a closure or generic function that's able to accept any `T` that conforms to `P`. It cannot write the type out, but it can certainly make use of it via type inference, e.g.:

```swift
g2 { x in x.doSomethingSpecifiedInP() }
```
