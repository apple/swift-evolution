## Usability of global-actor-isolated types

* Proposal: [SE-NNNN](NNNN-global-actor-isolated-types-usability.md)
* Authors: [Sima Nerush](https://github.com/simanerush), [Matt Massicotte](https://github.com/mattmassicotte), [Holly Borla](https://github.com/hborla)
* Review Manager: TBD
* Status: **Awaiting implementation**
* Implementation: TBD
* Review: ([pitch](https://forums.swift.org/t/pitch-usability-of-global-actor-isolated-types/70799))

## Introduction

This proposal encompasses a collection of changes to concurrency rules concerning global-actor-isolated types to improve their usability. 

## Motivation

Currently, there exist limitations in the concurrency model around global-isolated-types. 

First, let's consider rules for properties of global-isolated value types. The first limitation is that `var` properties of such value types cannot be declared `nonisolated`. This poses a number of problems, for example when implementing a protocol conformance. The current workaround is to use the `nonisolated(unsafe)` keyword on the property:

```swift
@MainActor struct S {
  nonisolated(unsafe) var x: Int = 0
}

extension S: Equatable {
  static nonisolated func ==(lhs: S, rhs: S) -> Bool {
    return lhs.x == rhs.x
  }
}
```

The above code is perfectly safe and should not require an unsafe opt-out. Since `S` is a value type and `x` is of a `Sendable` type `Int`, it should not be unsafe to declare `x` non-isolated, because when accessing `x` from different concurrency domains, we will be operating on a copy of the value type, and the result type is `Sendable` so it's safe to return the same value across different isolation domains. Because access to `x` across concurrency domains is always safe, `nonisolated` should be implicit within the module, similar to actor-isolated `let` constants with `Sendable` type.

Next, under the current concurrency rules, globally isolated functions and closures do not implicitly conform to `Sendable`. This impacts usability, because these closures cannot themselves be captured by `@Sendable` closures, which makes them unusable with `Task`:

```swift 
func test() {
  let closure: @MainActor () -> Void = {
    print("hmmmm")
  }

  Task {
    // error: capture of 'closure' with non-sendable type '@MainActor () -> Void' in a `@Sendable` closure
    await closure()
  }
}
```

In the above code, the closure is global-actor-isolated, so it cannot be called concurrently. The compiler should be able to infer the `@Sendable` attribute. Because of the same reason, globally isolated closures should be allowed to capture non-`Sendable` values.


Finally, the current diagnostic for a global-actor-isolated subclass of a non-isolated superclass is too restrictive:

```swift
class NotSendable {}


@MainActor
class Subclass: NotSendable {} // error: main actor-isolated class 'Subclass' has different actor isolation from nonisolated superclass 'NotSendable'
```

Because global actor isolation on a class implies a `Sendable` conformance, adding isolation to a subclass of a non-`Sendable` superclass can circumvent `Sendable` checking:

```swift
func computeCount() async -> Int { ... }

class NotSendable {
  var mutableState = 0
  func mutate() async {
    let count = await computeCount()
    mutableState += count
  }
}

@MainActor
class Subclass: NotSendable {}

func test() async {
  let c = Subclass()
  await withDiscardingTaskGroup { group in
    group.addTask {
      await c.mutate()
    }

    group.addTask { @MainActor in
      await c.mutate()
    }
  }
}
```

In the above code, an instance of `Subclass` can be passed across isolation boundaries because `@MainActor` implies that the type is `Sendable`. However, `Subclass` inherits non-isolated, mutable state from the superclass, so this `Sendable` conformance allows smuggling unprotected shared mutable state across isolation boundaries to create potential for concurrent access. For this reason, the warning about adding isolation to a subclass was added in Swift 5.10, but this restriction could be lifted by instead preventing the subclass from being `Sendable`.

## Proposed solution

We propose that:

- `Sendable` properties of a global-actor-isolated value type would be treated `nonisolated` as inferred within the module.
- `@Sendable` would be inferred for global-actor-isolated functions and closures. Additionally, globally isolated closures would be allowed to capture non-`Sendable` values.
- The programmer would be able to suppress the automatic conformance inferred via the above rule using the new `@~Sendable` attribute. By analogy, introduce a new `~Sendable` protocol to indicate that a nominal type is not `Sendable`.
- Require the global-actor-isolated subclass of a `nonisolated`, non-`Sendable` to be non-`Sendable`.


## Detailed design


### Inference of `nonisolated` for `var` properties of globally isolated value types

Let's look at the first problem with usability of a `var` property of a main-actor-isolated struct:

```swift
@MainActor 
struct S {
  var x: Int = 0 // okay ('nonisolated' is inferred within the module)
}

extension S: Equatable {
  static nonisolated func ==(lhs: S, rhs: S) -> Bool {
    return lhs.x == rhs.x // okay
  }
}
```

In the above code, `x` is implicitly `nonisolated` within the module. Under this proposal, `nonisolated` is inferred for within the module access of `Sendable` properties of a global-actor-isolated value type. This is data-race safe because the property belongs to a value type, meaning it will be copied every time it crosses an isolation boundary.

The programmer can still choose to mark the property `nonisolated` to allow synchronous access from outside the module. Requiring asynchronous access from outside the module preserves the ability for library authors to change a stored property to a computed property without breaking clients, and the library author may explicitly write `nonisolated` to opt-into synchronous access as part of the API contract.

### `@Sendable` inference for global-actor-isolated functions and closures

To improve usability of globally isolated functions and closures, under this proposal `@Sendable` is inferred:

```swift
func test() {
  let closure: @MainActor () -> Void = {
    print("hmmmm")
  }

  Task {
    await closure() // okay
  }
}
```

The closure in the above code is global-actor isolated via the `@MainActor`. Thus, it can never operate on the same reference concurrently at the same time, making it safe to be invoked from different isolation domains. This means that for such global-actor-isolated closures and functions, the `@Sendable` attribute is implicit.

#### Non-`Sendable` captures in isolated closures

Under this proposal, globally-isolated closures are allowed to capture non-`Sendable` values:

```swift
class NonSendable {}

func test() {
  let ns = NonSendable()

  let closure { @MainActor in
    print(ns)
  }

  Task {
    await closure() // okay
  }
}
```

The above code is data-race safe, since a globally isolated closure will never operate on the same instance of `NonSendable` concurrently.

Note that under region isolation in SE-0414, capturing a non-`Sendable` value in an actor-isolated closure will transfer the region into the actor, so it is impossible to have concurrent access on non-`Sendable` captures even if the isolated closure is formed outside the actor.


### `@~Sendable` and `~Sendable`

This proposal also adds a way to "opt-out" of the implicit `@Sendable` inference introduced by the above change by using the new `@~Sendable` attribute:

```swift
func test() {
  let closure: @~Sendable @MainActor () -> Void = {
    print("hmmmm")
  }

  Task {
    await closure() // error
  }
}
```

In the above code, we use `@~Sendable` to explicitly indicate that the closure is not `Sendable` and suppress the implicit `@Sendable`. This change will mostly help with possible ABI compatibility issues, but will not necessarily improve usability.

By analogy, this proposal inroduces the new `~Sendable` syntax for explicitly suppressing a conformance to `Sendable`:

```swift
class C { ... }

@available(*, unavailable)
extension C: @unchecked Sendable {}

// instead

class C: ~Sendable {}
```

In the above code, we use `~Sendable` instead of an unavailable `Sendable` conformance to indicate that the type `C` is not `Sendable`. Suppressing a `Sendable` conformance in a superclass still allows a conformance to be added in subclasses:

```swift
class Sub: C, @unchecked Sendable { ... }
```

Previously, an unavailable `Sendable` conformance would prevent `@unchecked Sendable` conformances from being added to subclasses because types can only conform to protocols in one way, and the unavailable `Sendable` conformance was inherited in all subclasses.

### Global actor isolation and inheritance

Subclasses may add global actor isolation when inheriting from a nonisolated, non-`Sendable` superclass. In this case, an implicit conformance to `Sendable` will not be added, and explicitly specifying a `Sendable` conformance is an error:

```swift
class NonSendable {
  func test() {}
}

@MainActor
class IsolatedSubclass: NonSendable {
  func trySendableCapture() {
    Task.detached {
      self.test() // error: Capture of 'self' with non-sendable type 'IsolatedSubclass' in a `@Sendable` closure
    }
  }
}
```

Inherited and overridden methods still must respect the isolation of the superclass method:

```swift
class NonSendable {
  func test() { ... }
}

@MainActor
class IsolatedSubclass: NonSendable {
  var mutable = 0
  override func test() {
    super.test()
    mutable += 0 // error: Main actor-isolated property 'isolated' can not be referenced from a non-isolated context
  }
}
```

Matching the isolation of the superclass method is necessary because the superclass method implementation may internally rely on the static isolation, such as when hopping back to the isolation after any asynchronous calls, and because there are a variety of ways to call the subclass method that don't preserve its isolation, including:

* Upcasting to the superclass type
* Erasing to an existential type based on conformances of the superclass type
* Passing the isolated subclass as a generic argument to a type parameter that requires a conformance implemented by the superclass

## Source compatibility

The introduced changes are additive, so the proposal does not impact source compatibility.

## ABI compatibility

This proposal should have no impact on ABI compatibility.


## Implications on adoption

This feature can be freely adopted and un-adopted in source code with no deployment constraints and without affecting source or ABI compatibility.

## Acknowledgments

Thank you to Frederick Kellison-Linn for surfacing the problem with global-actor-isolated function types, and to Kabir Oberai for exploring the implications more deeply.
