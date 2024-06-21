# Usability of global-actor-isolated types

* Proposal: [SE-0434](0434-global-actor-isolated-types-usability.md)
* Authors: [Sima Nerush](https://github.com/simanerush), [Matt Massicotte](https://github.com/mattmassicotte), [Holly Borla](https://github.com/hborla)
* Review Manager: [John McCall](https://github.com/rjmccall)
* Status: **Implemented (Swift 6.0)**
* Upcoming Feature Flag: `GlobalActorIsolatedTypesUsability`
* Review: ([pitch](https://forums.swift.org/t/pitch-usability-of-global-actor-isolated-types/70799)) ([review](https://forums.swift.org/t/se-0434-usability-of-global-actor-isolated-types/71187))

## Introduction

This proposal encompasses a collection of changes to concurrency rules concerning global-actor-isolated types to improve their usability. 

## Motivation

Currently, there exist limitations in the concurrency model around types that are isolated to global actors.

First, let's consider the stored properties of `struct`s isolated to global actors. `let` properties of such types are implicitly treated as `nonisolated` within the current module if they have `Sendable` type, but `var` properties are not.  This poses a number of problems, such as when implementing a protocol conformance.  Currently, the only solution is to declare the property `nonisolated(unsafe)`:

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

However, there is nothing unsafe about treating `x` as `nonisolated`.  The general rule is that concurrency is safe as long as there aren't data races.  The type of `x` conforms to `Sendable`, and using a value of `Sendable` type from multiple concurrent contexts shouldn't ever introduce a data race, so any data race involved with an access to `x` would have to be on memory in which `x` is stored.  But `x` is part of a value type, which means any access to it is always also an access to the containing `S` value.  As long as Swift is properly preventing data races on that larger access, it's always safe to access the `x` part of it.  So, first off, there's no reason for Swift to require `(unsafe)` when marking `x` `nonisolated`.

We can do better than that, though.  It should be possible to treat a `var` stored property of a global-actor-isolated value type as *implicitly* `nonisolated` under the same conditions that a `let` property can be.  A stored property from a different module can be changed to a computed property in the future, and those future computed accessors may need to be isolated to the global actor, so allowing access across module boundaries would not be okay for source or binary compatibility without an explicit `nonisolated` annotation.  But within the module that defines the property, we know that hasn't happened, so it's fine to use a more relaxed rule.

Next, under the current concurrency rules, it is possible for a function type to be both isolated to a global actor and yet not required to be `Sendable`:

```swift
func test(globallyIsolated: @escaping @MainActor () -> Void) {
  Task {
    // error: capture of 'globallyIsolated' with non-sendable type '@MainActor () -> Void' in a `@Sendable` closure
    await globallyIsolated()
  }
}
```

This is not a useful combination: such a function can only be used if the current context is isolated to the global actor, and in that case the global actor annotation is unnecessary because *all* non-`Sendable` functions will run with global actor isolation. It would be better for a global actor attribute to always imply `@Sendable`.

Because a globally-isolated closure cannot be called concurrently, it's safe for it to capture non-`Sendable` values even if it's implicitly `@Sendable`.  Such values just need to be transferred to the global actor's region (if they aren't there already).  The same logic also applies to closures that are isolated to a specific actor reference, although it isn't currently possible to write such a closure in a context that isn't isolated to that actor.


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

- Stored properties of `Sendable` type in a global-actor-isolated value type can be declared as `nonisolated` without using `(unsafe)`.
- Stored properties of `Sendable` type in a global-actor-isolated value type are treated as `nonisolated` when used within the module that defines the property.
- `@Sendable` is inferred for global-actor-isolated functions and closures.
- Global-actor-isolated closures are allowed to capture non-`Sendable` values despite being `@Sendable`.
- A global-actor-isolated subclass of a non-isolated, non-`Sendable` class is allowed, but it must be non-`Sendable`.


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

In the above code, `x` is implicitly `nonisolated` within the module. Under this proposal, `nonisolated` is inferred for in-module access to `Sendable` properties of a global-actor-isolated value type. A `var` with `Sendable` type within a value type can also have an explicit `nonisolated` modifier to allow synchronous access from outside the module. Once added, `nonisolated` cannot later be removed without potentially breaking clients. The programmer can still convert the property to a computed property, but it has to be a `nonisolated` computed property.

Because `nonisolated` access only applies to stored properties, wrapped properties and `lazy`-initialized properties with `Sendable` type still must be isolated because they are computed properties:

```swift
@propertyWrapper
struct MyWrapper<T> { ... }

@MainActor
struct S {
  @MyWrapper var x: Int = 0
}

extension S: Equatable {
  static nonisolated func ==(lhs: S, rhs: S) -> Bool {
    return lhs.x == rhs.x // error
  }
}
```

### `@Sendable` inference for global-actor-isolated functions and closures

To improve usability of globally-isolated functions and closures, under this proposal `@Sendable` is inferred:

```swift
func test(globallyIsolated: @escaping @MainActor () -> Void) {
  Task {
    await globallyIsolated() //okay
  }
}
```

The `globallyIsolated` closure in the above code is global-actor isolated because it has the `@MainActor` attribute. Because it will always run isolated, it's fine for it to capture and use values that are isolated the same way. It's also safe to share it with other isolation domains because the captured values are never directly exposed to those isolation domains. This means that there's no reason not to always treat these functions as `@Sendable`.

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

The above code is data-race safe, since a globally-isolated closure will never operate on the same instance of `NonSendable` concurrently.

Note that under region isolation in SE-0414, capturing a non-`Sendable` value in an actor-isolated closure will transfer the region into the actor, so it is impossible to have concurrent access on non-`Sendable` captures even if the isolated closure is formed outside the actor:

```swift
class NonSendable {}

func test(ns: NonSendable) async {
  let closure { @MainActor in
    print(ns) // error: task-isolated value 'ns' can't become isolated to the main actor
  }

  await closure()
}
```

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
    mutable += 0 // error: Main actor-isolated property 'mutable' can not be referenced from a non-isolated context
  }
}
```

Matching the isolation of the superclass method is necessary because the superclass method implementation may internally rely on the static isolation, such as when hopping back to the isolation after any asynchronous calls, and because there are a variety of ways to call the subclass method that don't preserve its isolation, including:

* Upcasting to the superclass type
* Erasing to an existential type based on conformances of the superclass type
* Passing the isolated subclass as a generic argument to a type parameter that requires a conformance implemented by the superclass

## Source compatibility

This proposal changes the interpretation of existing code that uses global-actor-isolated function types that are not already marked with `@Sendable`. This can cause minor changes in type inference and overload resolution. However, the proposal authors have not encountered any such issues in source compatibility testing, so this proposal does not gate the inference change behind an upcoming feature flag.

An alternative choice would be to introduce an upcoming feature flag that's enabled by default in the Swift 6 language mode, but this flag could not be enabled by default under `-strict-concurrency=complete` without risk of changing behavior in existing projects that adopt complete concurrency checking. Gating the `@Sendable` inference change behind a separate upcoming feature flag may lead to more code churn than necessary when migrating to complete concurrency checking unless the programmer knows to enable the flags in a specific order.

## ABI compatibility

`@Sendable` is included in name mangling, so treating global-actor-isolated function types as implicitly `@Sendable` changes mangling. This change only impacts resilient libraries that use global-actor-isolated-but-not-`Sendable` function types in effectively-public APIs. However, as noted in this proposal, such a function type is not useful, and the proposal authors expect that any API that uses a global-actor-isolated function type either already has `@Sendable`, or should add `@Sendable`. Because the only ABI impact of `@Sendable` is mangling, `@_silgen_name` can be used to preserve ABI in cases where `@Sendable` should be added, and the API is not already `@preconcurrency` (in which case the mangling will strip both the global actor and `@Sendable`).

## Implications on adoption

The existing adoption implications of `@Sendable` and global actor isolation adoption apply when making use of the rules in this proposal. For example, `@Sendable` and `@MainActor` can be staged into existing APIs using `@preconcurrency`. See [SE-0337: Incremental migration to concurrency checking](/proposals/0337-support-incremental-migration-to-concurrency-checking.md) for more information.

## Acknowledgments

Thank you to Frederick Kellison-Linn for surfacing the problem with global-actor-isolated function types, and to Kabir Oberai for exploring the implications more deeply.
