# Isolated default value expressions

* Proposal: [SE-0411](0411-isolated-default-values.md)
* Authors: [Holly Borla](https://github.com/hborla)
* Review Manager: [Doug Gregor](https://github.com/DougGregor)
* Status: **Implemented (Swift 5.10)**
* Bug: *if applicable* [apple/swift#58177](https://github.com/apple/swift/issues/58177)
* Implementation: [apple/swift#68794](https://github.com/apple/swift/pull/68794)
* Upcoming Feature Flag: `IsolatedDefaultValues` (Enabled in Swift 6 language mode)
* Review: ([acceptance](https://forums.swift.org/t/accepted-se-0411-isolated-default-value-expressions/68806)) ([review](https://forums.swift.org/t/se-0411/68065)) ([pitch](https://forums.swift.org/t/pitch-isolated-default-value-expressions/67714))

## Introduction

Default value expressions are permitted for default arguments and default stored property values. There are several issues with the current actor isolation rules for default value expressions: the rules for stored properties admit data races, the rules for default argument values are overly restrictive, and the rules between the different places you can use default value expressions are inconsistent with each other, making the actor isolation model harder to understand. This proposal unifies the actor isolation rules for default value expressions, eliminates data races, and improves expressivity by safely allowing isolation for default values.

## Motivation

The current actor isolation rules for initial values of stored properties admit data races. For example, the following code is currently valid:

```swift
@MainActor func requiresMainActor() -> Int { ... }
@AnotherActor func requiresAnotherActor() -> Int { ... }

class C {
  @MainActor var x1 = requiresMainActor()
  @AnotherActor var x2 = requiresAnotherActor()

  nonisolated init() {} // okay???
}
```

The above code allows any context to initialize an instance of `C()` through a synchronous, nonisolated `init`. The initializer synchronously calls both `requiresMainActor()` and `requiresAnotherActor()`, which are `@MainActor`-isolated and `@AnotherActor`-isolated, respectively. This violates actor isolation checking because `requiresMainActor()` and `requiresAnotherActor()` may run concurrently with other code on their respective global actors.

The current actor isolation rules for default argument values do not admit data races, but default argument values are always `nonisolated` which is overly restrictive. This rule prohibits programmers from making `@MainActor`-isolated calls in default argument values of `@MainActor`-isolated functions. For example, the following code is not valid even though it is perfectly safe:

```swift
@MainActor class C {}

@MainActor func f(c: C = C()) {} // error: Call to main actor-isolated initializer 'init()' in a synchronous nonisolated context

@MainActor func useFromMainActor() {
  f()
}
```

## Proposed solution

I propose allowing default value expressions to have the same isolation as the enclosing function or the corresponding stored property. As usual, if the caller is not already in the isolation domain of the callee, then the call must be made asynchronously and must be explicitly marked with `await`. For isolated default values of stored properties, the implicit initialization only happens in the body of an `init` with the same isolation.

These rules make the stored property example above invalid at the `nonisolated` initializer:

```swift
@MainActor func requiresMainActor() -> Int { ... }
@AnotherActor func requiresAnotherActor() -> Int { ... }

class C {
  @MainActor var x1 = requiresMainActor()
  @AnotherActor var x2 = requiresAnotherActor()

  nonisolated init() {} // error: 'self.x1' and 'self.x2' are not initialized
}
```

Calling `requiresMainActor()` and `requiresAnotherActor()` explicitly with `await` resolves the issue:

```swift
class C {
  @MainActor var x1 = requiresMainActor()
  @AnotherActor var x2 = requiresAnotherActor()

  nonisolated init() async {
    self.x1 = await requiresMainActor()
    self.x2 = await requiresAnotherActor()
  }
}
```

This rule also makes the default argument example above valid, because the default argument and the enclosing function are both `@MainActor`-isolated.

## Detailed design

### Inference of default value isolation requirements

Default value expressions are always evaluated in a synchronous context, so all calls that are made during the evaluation of the expression must also be synchronous. If the callee is isolated, then the default value expression must already be in the same isolation domain in order to make the call synchronously. So, for a given default value expression, the inferred isolation is the required isolation of its subexpressions. For example:

```swift
@MainActor func requiresMainActor() -> Int { ... }

@MainActor func useDefault(value: Int = requiresMainActor()) { ... }
```

In the above code, the default argument for `value` requires `@MainActor` isolation, because the default value calls `requiresMainActor()` which is isolated to `@MainActor`.

#### Closures

Evaluating a closure literal itself can happen in any isolation domain; the actor isolation of a closure only applies when calling the closure. An actor-isolated closure enables the closure body to make calls within that isolation domain synchronously. For a closure literal in a default value expression that is not explicitly annotated with actor isolation, the inferred isolation of the closure is the union of the isolation of all callees in the closure body for synchronous calls. For example:

```swift
@MainActor func requiresMainActor() -> Int { ... }

@MainActor func useDefaultClosure(
  closure: () -> Void = {
    requiresMainActor()
  }
) {}
```

The above `useDefaultClosure` function has a default argument value that is a closure literal. The closure body calls a `@MainActor`-isolated function synchronously, therefore the closure itself must be `@MainActor` isolated.

Note that the only way for a closure literal in a default argument to be isolated to an actor instance is for the isolation to be written explicitly with an isolated parameter. The inference algorithm will never determine the isolation to be an actor instance based on the following two properties:

1. To be isolated to an actor instance, a closure must either have its own (explicit) isolated parameter or capture an isolated parameter from its enclosing context.
2. Closure literals in default arguments cannot capture values.

#### Restrictions

* If a function or type itself has actor isolation, the required isolation of its default value expressions must share the same actor isolation. For example, a `@MainActor`-isolated function cannot have a default argument that is isolated to `@AnotherActor`. Note that it's always okay to mix isolated default values with `nonisolated` default values.
* If a function or type is `nonisolated`, then the required isolation of its default value expressions must be `nonisolated`.

### Enforcing default value isolation requirements

#### Default argument values

Isolation requirements for default argument expressions are enforced at the caller. If the caller is not in the required isolation domain, the default arguments must be evaluated asynchronously and explicitly  marked with `await`. For example:

```swift
@MainActor func requiresMainActor() -> Int { ... }

@MainActor func useDefault(value: Int = requiresMainActor()) { ... }

@MainActor func mainActorCaller() {
  useDefault() // okay
}

func nonisolatedCaller() async {
  await useDefault() // okay

  useDefault() // error: call is implicitly async and must be marked with 'await'
}
```

In the above example, `useDefault` has default arguments that are isolated to `@MainActor`. The default arguments can be evaluated synchronously from a `@MainActor`-isolated caller, but the call must be marked with `await` from outside the `@MainActor`. Note that these rules already fall out of the semantics of calling actor isolated functions.

#### Argument evaluation

For a given call, argument evaluation happens in the following order:

1. Left-to-right evaluation of explicit r-value arguments
2. Left-to-right evaluation of default arguments and formal access arguments

For example:

```swift
nonisolated var defaultVal: Int { print("defaultVal"); return 0 }
nonisolated var explicitVal: Int { print("explicitVal"); return 0 }
nonisolated var explicitFormalVal: Int {
  get { print("explicitFormalVal"); return 0 }
  set {}
}

func evaluate(x: Int = defaultVal, y: Int = defaultVal, z: inout Int) {}

evaluate(y: explicitVal, z: &explicitFormalVal)
```

The output of the above program is

```
explicitVal
defaultVal
explicitFormalVal
```

Unlike the explicit argument list, isolated default arguments must be evaluated in the isolation domain of the callee. As such, if any of the argument values require the isolation of the callee, argument evaluation happens in the following order:

1. Left-to-right evaluation of explicit r-value arguments
2. Left-to-right evaluation of formal access arguments
3. Hop to the callee's isolation domain
4. Left-to-right evaluation of default arguments

For example:

```swift
@MainActor var defaultVal: Int { print("defaultVal"); return 0 }
nonisolated var explicitVal: Int { print("explicitVal"); return 0 }
nonisolated var explicitFormalVal: Int {
  get { print("explicitFormalVal"); return 0 }
  set {}
}

@MainActor func evaluate(x: Int = defaultVal, y: Int = defaultVal, z: inout Int) {}

nonisolated func nonisolatedCaller() {
  await evaluate(y: explicitVal, z: &explicitFormalVal)
}
```

The output of calling `nonisolatedCaller()` is:

```
explicitVal
explicitFormalVal
defaultVal
```

#### Stored property initial values

Isolation requirements for default initializer expressions for stored properties apply in the body of initializers. If an `init` does not match the isolation of the initializer expression, the initialization of that stored property is not emitted at the beginning of the `init`. Instead, the stored property must be explicitly initialized in the body of the `init`. For example:

```swift
@MainActor func requiresMainActor() -> Int { ... }
@AnotherActor func requiresAnotherActor() -> Int { ... }

class C {
  @MainActor var x1: Int = requiresMainActor()
  @AnotherActor var x2: Int = requiresAnotherActor()

  nonisolated init() {} // error: 'self.x1' and 'self.x2' aren't initialized

  nonisolated init(x1: Int, x2: Int) { // okay
    self.x1 = x1
    self.x2 = x2
  }

  @MainActor init(x2: Int) { // okay
    // 'self.x1' gets assigned to the default value 'requiresMainActor()'
    self.x2 = x2
  }
}
```

In the above example, the no-parameter `nonisolated init()` is invalid, because it does not initialize `self.x1` and `self.x2`. Because the default initializer expressions require different actor isolation, those values are not used in the `nonisolated` initializer. The other two initializers are valid.

### Stored property isolation in initializers

#### Initializing isolated stored properties from across isolation boundaries

It is invalid to initialize an isolated stored property from across isolation boundaries:

```swift
class NonSendable {}

class C {
  @MainActor var ns: NonSendable

  init(ns: NonSendable) {
    self.ns = ns // error: passing non-Sendable value 'ns' to a MainActor-isolated context.
  }
}
```

The above code violates `Sendable` guarantees because the initialization of the `MainActor`-isolated property `self.ns` from a `nonisolated` context is effectively passing a non-`Sendable` value across isolation boundaries. To prevent this class of data races, this proposal requires that any `init` that initializes a global actor isolated stored property must also be isolated to that global actor.

Note that this rule is not specific to default values, but it's necessary to specify the behavior of default values in compiler-synthesized initializers.

#### Default value isolation in synthesized initializers

For structs, default initializer expressions for stored properties are used as default argument values to the compiler-generated memberwise initializer. For structs and classes that have a compiler-generated no-parameter initializer, the default initializer expressions are also used in the synthesized `init()` body.

If any of the type's stored properties with non-`Sendable` type are actor isolated, or if any of the isolated default initializer expressions are actor isolated, then the compiler-synthesized initializer(s) must also be actor isolated. For example:

```swift
class NonSendable {}

@MainActor struct MyModel {
  // @MainActor inferred from annotation on enclosing struct
  var value: NonSendable = .init()

  /* compiler-synthesized memberwise init is @MainActor
  @MainActor
  init(value: NonSendable = .init()) {
    self.value = value
  }
  */
}
```

If none of the type's stored properties are non-`Sendable` and actor isolated, and none of the default initializer expressions require actor isolation, then the compiler-synthesized initializer is `nonisolated`. For example:

```swift
@MainActor struct MyView {
  // @MainActor inferred from annotation on enclosing struct
  var value: Int = 0

  /* compiler-synthesized 'init's are 'nonisolated'

  nonisolated init() {
    self.value = 0
  }

  nonisolated init(value: Int = 0) {
    self.value = value
  }
  */

  // @MainActor inferred from the annotation on the enclosing struct
  var body: some View { ... }
}
```

These rules ensure that the default value expressions in compiler-synthesized initializers are always valid. If a default value expression requires actor isolation, then the enclosing initializer always shares the same actor isolation. It is an error for two different default values to require different actor isolation, because it's not possible to ever use those default values. Initializing an instance of a type using two different initial value expressions with different actor isolation must be done in an `async` initializer, with suspension points explicitly marked with `await`.

## Source compatibility

The actor isolation rules for initial values of stored properties are stricter than what is currently accepted in Swift 5 mode in order to eliminate data races. The isolation rules for stored properties will be staged in under the `IsolatedDefaultValues` upcoming feature identifier.

## ABI compatibility

This is a change to actor isolation checking with no impact on ABI.

## Implications on adoption

This feature can be freely adopted and un-adopted in source code with no deployment constraints and without affecting source or ABI compatibility.

## Alternatives considered

### Remove isolation from all default initializer expressions

SE-0327 originally proposed changing default initializer expressions for stored properties to always be `nonisolated`, matching the current default argument value rules. However, this change was implemented and later reverted because it impacted a lot of code that followed a common pattern: a `@MainActor`-isolated type with stored properties that have default values that call the initializers of other `@MainActor`-isolated types. In some cases, it's possible to make the initializer of a `@MainActor` type `nonisolated`, but many of these cases do access `@MainActor`-isolated properties and functions in the body of the initializer.

## Acknowledgments

Thank you to Kavon Farvardin for implementing the default initializer expression rules originally proposed by SE-0327 and discovering the usability issues outlined in this proposal. Thank you to John McCall for the observation that memberwise initializers can and should be `nonisolated` when possible.

## Revision history

* Changes from the first pitch
  * Require that isolated default arguments share the same isolation as their enclosing function or type.
  * Specify the semantic restrictions on initializing actor isolated properties from across isolation boundaries.
  * Enable using isolated default arguments from across isolation boundaries by changing the argument evaluation between formal access and default arguments.
