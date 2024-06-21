# Inferring `Sendable` for methods and key path literals

* Proposal: [SE-0418](0418-inferring-sendable-for-methods.md)
* Authors: [Angela Laar](https://github.com/angela-laar), [Kavon Farvardin](https://github.com/kavon), [Pavel Yaskevich](https://github.com/xedin)
* Review Manager: [Becca Royal-Gordon](https://github.com/beccadax)
* Status: **Implemented (Swift 6.0)**
* Upcoming Feature Flag: `InferSendableFromCaptures`
* Review: ([pitch](https://forums.swift.org/t/pitch-inferring-sendable-for-methods/66565)) ([review](https://forums.swift.org/t/se-0418-inferring-sendable-for-methods-and-key-path-literals/68999)) ([acceptance](https://forums.swift.org/t/accepted-se-0418-inferring-sendable-for-methods-and-key-path-literals/69242))

## Introduction

This proposal is focused on a few corner cases in the language surrounding functions as values and key path literals when using concurrency. We propose Sendability should be inferred for partial and unapplied methods. We also propose to lift a Sendability restriction placed on key path literals in [SE-0302](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md#key-path-literals) by allowing the developers to control whether key path literal is Sendable or not. The goal is to improve flexibility, simplicity, and ergonomics without significant changes to Swift.

## Motivation

The partial application of methods and other first-class uses of functions have a few rough edges when combined with concurrency.

Let’s look at partial application on its own before we combine it with concurrency.  In Swift, you can create a function-value representing a method by writing an expression that only accesses (but does not call) a method using one of its instances. This access is referred to as a "partial application" of a method to one of its (curried) arguments - the object instance.

```swift
struct S {
  func f() { ... }
}

let partial: (() -> Void) = S().f 
```


When referencing a method *without* partially applying it  to the object instance, using the expression NominalType.method, we call it "unapplied."


```swift
let unapplied: (S) -> (() -> Void) = S.f
```


Suppose we want to create a generic method that expects an unapplied function method conforming to Sendable as a parameter. We can create a protocol `P` that conforms to the `Sendable` protocol and tell our generic function to expect some generic type that conforms to `P`. We can also use the `@Sendable` attribute, introduced for closures and functions in [SE-302](https://github.com/kavon/swift-evolution/blob/sendable-functions/proposals/0302-concurrent-value-and-concurrent-closures.md), to annotate the closure parameter.


```swift
protocol P: Sendable {
  init()
}

func g<T>(_ f: @escaping @Sendable (T) -> (() -> Void)) where T: P {
  Task {
    let instance = T()
    f(instance)()
  }
}
```

Now let’s call our method and pass our struct type `S` . First we should make `S` conform to Sendable, which we can do by making `S` conform to our new Sendable type `P` . 

This should make `S` and its methods Sendable as well. However, when we pass our unapplied function `S.f`  to our generic function `g`, we get a warning that `S.f` is not Sendable as `g()` is expecting.  


```swift
struct S: P {
  func f() { ... }
}

g(S.f) // Converting non-sendable function value to '@Sendable (S) -> (() -> Void)' may introduce data races
```


We can work around this by wrapping our unapplied function in a Sendable closure.  

```swift
// S.f($0) == S.f()
g({ @Sendable in S.f($0) })
```


However, this is a lot of churn to get the expected behavior. The compiler should preserve `@Sendable` in the type signature instead.

**Key Paths**

[SE-0302](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md#key-path-literals) makes an explicit mention that all key path literals are treated as implicitly `Sendable` which means that they are not allowed to capture any non-`Sendable` values. This behavior is justified when key path values are passed across concurrency domains or otherwise involved in concurrently executed code but is too restrictive for non-concurrency related code.

```swift
class Info : Hashable {
  // some information about the user
}

public struct Entry {}

public struct User {
  public subscript(info: Info) -> Entry {
    // find entry based on the given info
  }
}

let entry: KeyPath<User, Entry> = \.[Info()]
```

With sendability checking enabled this example is going to produce the following warning:

```
warning: cannot form key path that captures non-sendable type 'Info'
let entry: KeyPath<User, Entry> = \.[Info()]
                                     ^
```

Use of the key path literal is currently being diagnosed because all key path literals should be Sendable. In actuality, this code is concurrency-safe, there are no data races here because key path doesn’t actually cross any isolation boundary. The compiler should instead verify and diagnose situations when key path is actually passed across an isolation boundary otherwise a warning like that would be confusing for the developers unfamiliar with Swift concurrency, might not always be actionable when type is declared in a different module, and goes against the progressive disclosure principle of the language.

## Proposed solution

We propose the compiler should automatically employ `Sendable`  on functions and key paths that cannot capture non-Sendable values. This includes partially-applied and unapplied instance methods of `Sendable` types, as well as non-local functions. Additionally, it should be disallowed to utilize `@Sendable` on instance methods of non-`Sendable` types.

**Functions**

For a function, the `@Sendable` attribute primarily influences the kinds of values that can be captured by the function. But methods of a nominal type do not capture anything but the object instance itself. Semantically, a method can be thought of as being represented by the following functions:


```swift
// Pseudo-code declaration of a Nominal Type:
type NominalType {
  func method(ArgType) -> ReturnType { /* body of method */ }
}

// Can desugar to these two global functions:
func NominalType_method_partiallyAppliedTo(_ obj: NominalType) -> ((ArgType) -> ReturnType) {
  let inner = { [obj] (_ arg1: ArgType) -> ReturnType in
    return NominalType_method(obj, arg1)
  }
  return inner
}
// The actual method call
func NominalType_method(_ self: NominalType, _ arg1: ArgType) -> ReturnType {
  /* body of method */
}
```

Thus, the only way a partially-applied method can be `@Sendable` is if the `inner` closure were `@Sendable`, which is true if and only if the nominal type conforms to `Sendable`.


```swift
type NominalType : Sendable {
  func method(ArgType) -> ReturnType { /* body of method */ }
}
```

For example, by declaring the following type `Sendable`, the partial and unapplied function values of the type would have implied Sendability and the following code would compile with no errors.

```swift
struct User : Sendable {
  func updatePassword (new: String, old: String) -> Bool {
    /* update password*/ 
    return true
  }
}

let unapplied: @Sendable (User) -> ((String, String) → Bool) = User.updatePassword // no error

let partial: @Sendable (String, String) -> Bool = User().updatePassword // no error
```

**Key paths**

Key path literals are very similar to functions, their sendability could be influenced by sendability of the values they capture in their arguments and isolation of the referenced properties and subscripts.  Instead of requiring key path literals to always be sendable and warning about cases where key path literals capture non-Sendable types, let’s flip that requirement and allow the developers to explicitly state when a key path is required to be Sendable via `& Sendable` type composition and employ type inference to infer sendability in the same fashion as functions when no contextual type is specified. [The key path hierarchy of types is non-Sendable].

Let’s extend our original example type `User` with a new property and a subscript to showcase the change in behavior:

```swift
struct User {
  var name: String

  @MainActor var age: Int

  subscript(_ info: Info) -> Entry { ... }
}
```

A key path to reference a property `name` does not capture any non-Sendable types which means the type of such key path literal could either be inferred as `WritableKeyPath<User, String> & Sendable` or stated to have a sendable type via `& Sendable` composition:

```swift
let name = \User.name // WritableKeyPath<User, String> **& Sendable**
let name: KeyPath<User, String> & Sendable = \.name // 🟢
```

It is also allowed to use `@Sendable` function type and `& Sendable` key path interchangeably:

```swift
let name: @Sendable (User) -> String = \.name 🟢
```

It is important to note that **under the proposed rule all of the declarations that do not explicitly specify a Sendable requirement alongside key path type are treated as non-Sendable** (see Source Compatibility section for further discussion):

```swift
let name: KeyPath<User, String> = \.name // 🟢 but key path is **non-Sendable**
```

Since Sendable is a marker protocol it should be possible to adjust all declarations where `& Sendable` is desirable without any ABI impact.

Existing APIs that use key path in their parameter types or default values can add `Sendable` requirement in a non-ABI breaking way by marking existing declarations as @preconcurrency and adding `& Sendable` at appropriate positions:

```swift
public func getValue<T, U>(_: KeyPath<T, U>) { ... }
```

becomes

```swift
@preconcurrency public func getValue<T, U>(_: KeyPath<T, U> & Sendable) { ... }
```

Explicit sendability annotation does not override sendability checking and it would still be incorrect to state that the key path literal is Sendable when it captures non-Sendable values:

```swift
let entry: KeyPath<User, Entry> & Sendable = \.[Info()] 🔴 Info is a non-Sendable type
```

Such `entry` declaration would be diagnosed by the sendability checker:

```swift
warning: cannot form key path that captures non-sendable type 'Info'
```

In the same fashion key path that references `age` (i.e. `\User.age`), which is a global actor isolated property, is non-Sendable.

## Detailed design

This proposal includes five changes to `Sendable` behavior.

The first two are what we just discussed regarding partial and unapplied methods.

```swift
struct User : Sendable {
  var address: String
  var password: String

  func changeAddress (new: String, old: String) {/*do work*/ }
}
```

1. The inference of `@Sendable` for unapplied references to methods of a Sendable type.

```swift
let unapplied : @Sendable (User)-> ((String, String) -> Void) = User.changeAddress // no error
```

2. The inference of `@Sendable` for partially-applied methods of a Sendable type.

```swift
let partial : @Sendable (String, String) -> Void = User().changeAddress // no error
```


These two rules include partially applied and unapplied static methods but do not include partially applied or unapplied mutable methods. Unapplied references to mutable methods are not allowed in the language because they can lead to undefined behavior.  More details about this can be found in [SE-0042](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0042-flatten-method-types.md).


3. A key path literal without non-Sendable type captures and references to actor-isolated properties and/or subscripts is going to be inferred as key path type with a `& Sendable` requirement or a function type with `@Sendable` attribute.

```swift
extension User {
  @MainActor var age: Int { get { 0 } }
}

let ageKP = \User.age
let infoKP = \User.[Info()]
```

The type of `ageKP` is `KeyPath<User, Int>` because `age` is isolated to a global actor. Similarly `infoKP` is a non-Sendable key path because `Info()` argument to a subscript reference has a non-Sendable type.

Key path types respect all of the existing sub-typing rules related to Sendable protocol which means a key path that is not marked as Sendable cannot be assigned to a value that is Sendable.

```swift
let name: KeyPath<User, String> = \.name
let otherName: KeyPath<User, String> & Sendable = \.name 🔴
```

The conversion between key path and a `@Sendable` function doesn’t actually require the key path itself to be `Sendable` because it’s not captured by the closure but wrapped by it.

```swift
let name: @Sendable (User) -> String = \.name 🟢
```

 The example above is accepted and  is transformed by the compiler into:

```swift
let name: @Sendable (User) -> String = { $0[keyPath: \.name] }
```

But any subscript arguments that are non-Sendable would preclude the conversion because they’d be captured by the implicitly synthesized closure which makes the closure non-Sendable:

```swift
let value: NonSendable = NonSendable()
let _: @Sendable (User) -> String = \.[value] 🔴
```

This is an error because `value` has a non-Sendable type and the compiler synthesized closure that wraps the key path - `{ $0[keyPath: \.[value]] }` is going to be inferred as non-Sendable (because it captures `value`) hence non-convertible to a `@Sendable` function type.

Similarly if the conversion captures a key path that has a reference to an isolated property or subscript the implicitly generated closure is not inferred to be non-Sendable.

Key path literals are allowed to infer Sendability requirements from the context i.e. when a key path literal is passed as an argument to a parameter that requires a Sendable type:

```swift
func getValue<T: Sendable>(_: KeyPath<User, T> & Sendable) -> T {}

getValue(name) // 🟢 both parameter & argument match on sendability requirement
getValue(\.name) // 🟢 use of '& Sendable' by the parameter transfers to the key path literal
getValue(\.[NonSendable()]) // 🔴 This is invalid because key path captures a non-Sendable type

func filter<T: Sendable>(_: @Sendable (User) -> T) {}
filter(name) // 🟢 use of @Sendable applies a sendable key path
```

Next is:

4. The inference of `@Sendable`  when referencing non-local functions.

Unlike closures, which retain the captured value, global functions can't capture any variables - because global variables are just referenced by the function without any ownership. With this in mind there is no reason not to make these `Sendable` by default. This change will also include static global functions.

```swift
func doWork() -> Int {
  Int.random(in: 1..<42)
}

Task<Int, Never>.detached(priority: nil, operation: doWork) // Converting non-sendable function value to '@Sendable () async -> Void' may introduce data races
```

Currently, trying to start a `Task` with the global function `doWork` will cause an error complaining that the function is not `Sendable`. This should compile with no issue.  

5. Prohibition of marking methods `@Sendable` when the type they belong to is not `@Sendable`.

```swift
class C {
    var random: Int = 0 // random is mutable so `C` can't be checked sendable

    @Sendable func generateN() async -> Int { //error: adding @Sendable to function of non-Senable type prohibited
         random = Int.random(in: 1..<100)
         return random
    }
}

func test(x: C) { x.generateN() }

let num = C()
Task.detached {
  test(num)
}
test(num) // data-race
```

If we move the previous work we wanted to do into a class that stores the random number we generate as a mutable value, we could be introducing a data race by marking the function responsible for this work `@Sendable` . Doing this should be prohibited by the compiler.

Since `@Sendable` attribute will be automatically determined with this proposal, you will no longer have to explicitly write it on function and method declarations.

### Extending key path merging functionality to preserve sendability

Existing Key path API provides a way to join two key paths together via using instance method `appending(...)` . Overloads of this method take key path types of varying mutability as their parameters and produce a new “joined” key path of a desired mutability (read-only, writable, or reference writable).

Under the proposed semantics all overloads of this method become non-Sendable but it is possible and desirable to alleviate that and support/propagate sendability if both “base” and “appended” key paths are `Sendable`.

Such could be archived by introducing new overloads to `func appending(...)` that utilize `& Sendable` for their parameter and result in an extension of `Sendable` protocol. For example:

```swift
extension Sendable where Self: AnyKeyPath {
  @inlinable
  public func appending<Root, Value, AppendedValue>(
        path: KeyPath<Value, AppendedValue> & Sendable
  ) -> KeyPath<Root, AppendedValue> & Sendable where Self : KeyPath<Root, Value> {
    ...
  }
}
```

This overload would be selected if both “base” key path and the argument are `Sendable` and would produce a new `Sendable` key path:

```swift
func makeUTF8CountKeyPath<Root>(from base: KeyPath<Root, String> & Sendable) -> KeyPath<Root, Int> & Sendable {
  // Both `base` and `\String.utf8.count` are Sendable key paths,
  // so `appending(path:)` returns a Sendable key path too.
  return base.appending(path: \.utf8.count) 🟢
}
```

Standard library would have to introduce a variety of new overloads to keep `Sendable` capable  `appending(...)` on par with existing non-Sendable functionality.

## Source compatibility

As described in the Proposed Solution section, some of the existing property and variable declarations **without explicit types** could change their type but the impact of the inference change should be very limited. For example, it would only be possible to observe it when a function or key path value which is inferred as Sendable is passed to an API which is overloaded on Sendable capability:

```swift
func callback(_: @Sendable () -> Void) {}
func callback(_: () -> Void) {}

callback(MyType.f) // if `f` is inferred as @Sendable first `callback` is preferred

func getValue(_: KeyPath<String, Int> & Sendable) {}
func getValue(_: KeyPath<String, Int>) {}

getValue(\.utf8.count) // prefers first overload of `getValue` if key path is `& Sendable`
```

Such calls to `callback` and `getValue` are currently ambiguous but under the proposed rules the type-checker would pick the first overload of `callback` and `getValue` as a solution if `f` is inferred as `@Sendable` and `\String.utf8.count` would be inferred as having a type of `KeyPath<String, Int> & Sendable` instead of just `KeyPath<String, Int>`.

## Effect on ABI stability

When you remove an explicit `@Sendable` from a method, the mangling of that method will change. Since `@Sendable` will now be inferred, if you choose to remove the explicit annotation to "adopt" the inference, you may need to consider the mangling change.

Adding or removing `& Sendable` from type doesn’t have any ABI impact because `Sendable` is a marker protocol that can be added transparently.

## Effect on API resilience

N/A

## Future Directions 

Accessors are not currently allowed to participate with the `@Sendable` system in this proposal. It would be straightforward to allow getters to do so in a future proposal if there was demand for this.

## Alternatives Considered 

Swift could forbid explicitly marking function declarations with the `@Sendable` attribute, since under this proposal there’s no longer any reason to do this.

```swift
/*@Sendable*/ func alwaysSendable() {}
```

However, since these attributes are allowed today, this would be a source breaking change. Swift 6 could potentially include fix-its to remove `@Sendable` attributes to ease migration, but it’d still be disruptive. The attributes are harmless under this proposal, and they’re still sometimes useful for code that needs to compile with older tools, so we have chosen not to make this change in this proposal. We can consider deprecation at a later time if we find a good reason to do so.
