# Closure isolation control

* Proposal: [SE-NNNN](nnnn-closure-isolation.md)
* Authors: [Sophia Poirier](https://github.com/sophiapoirier), [Matt Massicotte](https://github.com/mattmassicotte), [Konrad Malawski](https://github.com/ktoso), [John McCall](https://github.com/rjmccall)
* Review Manager: TBD
* Status: **Awaiting review**
* Implementation: On `main` gated behind `-enable-experimental-feature ClosureIsolation`
* Previous Proposals: [SE-0313](0313-actor-isolation-control.md), [SE-0316](0316-global-actors.md)
* Review: ([pitch](https://forums.swift.org/t/isolation-assumptions/69514))

## Introduction

This proposal provides the ability to explicitly specify actor-isolation or non-isolation of a closure, as well as providing a parameter attribute to guarantee that a closure parameter inherits the isolation of the context. It makes the isolation inheritance rules more uniform, helps to better express intention at closure-creation time, and also makes integrating concurrency with non-Sendable types less restrictive.

## Table of Contents

* [Introduction](#introduction)
* [Motivation](#motivation)
* [Proposed solution](#proposed-solution)
  + [Explicit closure isolation](#explicit-closure-isolation)
  + [Isolation inheritance](#isolation-inheritance)
* [Detailed design](#detailed-design)
  + [Distributed actor isolation](#distributed-actor-isolation)
* [Source compatibility](#source-compatibility)
* [ABI compatibility](#abi-compatibility)
* [Implications on adoption](#implications-on-adoption)
* [Alternatives considered](#alternatives-considered)
* [Future directions](#future-directions)
* [Acknowledgments](#acknowledgments)

## Motivation

The formal isolation of a closure can be explicitly specified as global actor isolation:

```swift
Task { @MainActor in
  print("global actor isolation")
}
```

Without a global actor isolation annotation, actor-isolation or non-isolation of a closure is inferred but cannot be explicitly specified. This proposal enables closures to be fully explicit about all three types of formal isolation:
* `nonisolated`
* global actor
* specific actor value

Explicit annotation has the benefit of disabling inference rules and the potential that they lead to a formal isolation that is not preferred. For example, there are circumstances where it is beneficial to guarantee that a closure is `nonisolated` therefore knowing that its execution will hop off the current actor. Explicit annotation also offers the ability to identify a mismatch of intention, such as a case where the developer expected `nonisolated` but inference landed on actor-isolated, and the closure is mistakenly used in an isolated context. Using explicit annotation, the developer would receive a diagnostic about a `nonisolated` closure being used in an actor-isolated context which helpfully identifies this mismatch of intention.

Additionally, there is a difference in how isolation inheritance behaves via the experimental attribute `@_inheritActorContext` (as used by `Task.init`) for isolated parameters vs actor isolation: global actor isolation is inherited by `Task`'s initializer closure argument, whereas an actor-isolated parameter is not inherited. This makes it challenging to build intuition around how isolation inheritance works. It also makes it impossible to allow a non-Sendable type to create a new Task that can access self.

```swift
class NonSendableType {
  @MainActor
  func globalActor() {
    Task {
      // accessing self okay
    }
  }

  func isolatedParameter(_ actor: isolated any Actor) {
    Task {
      // not okay to access self
    }
  }
}
``` 

## Proposed solution

### Explicit closure isolation

Enable explicit specification of non-isolation by allowing `nonisolated` to be a modifier on a closure:

```swift
Task { nonisolated in
  print("nonisolated")
}
```

Enable explicit specification of actor-isolation via an isolated parameter in a closure's capture list by using the `isolated` specifier:

```swift
actor A {
  nonisolated func isolate() {
    Task { [isolated self] in
      print("isolated to 'self'")
    }
  }
}
```

### Isolation inheritance

Provide a formal replacement of the experimental parameter attribute `@_inheritActorContext` to resolve its ambiguity with closure isolation. Currently, `@_inheritActorContext` actual context capture behavior is conditional on whether you capture an isolated parameter or isolated capture or actor-isolated function, but unconditional if the context is isolated to a global actor or `nonisolated`. Its replacement `@inheritsIsolation` changes the behavior so that it unconditionally and implicitly captures the isolation context.

```swift
class Old {
  public init(@_inheritActorContext operation: () async)
}

class New {
  public init(@inheritsIsolation operation: () async)
}

class C {
  var value = 0

  @MainActor
  func staticIsolation() {
    Old {
      value = 1 // closure is MainActor-isolated and therefore okay to access self
    }
    New {
      value = 2 // closure is MainActor-isolated and therefore okay to access self
    }
  }

  func dynamicIsolation(_ actor: isolated any Actor) {
    Old {
      // not isolated to actor without explicit capture
    }
    New {
      // isolated to actor through guaranteed implicit capture
    }
  }
}
```

## Detailed design

An isolated parameter in a capture list must be of actor type, or conform to or imply an actor, potentially optional, and there can only be one isolated parameter captured, following the same rules described in [SE-0313](0313-actor-isolation-control.md#actor-isolated-parameters) for actor-isolated parameters.

The contexts in which an isolated parameter is permitted in the capture list of a synchronous closure are when the closure is:

* called immediately
* converted to an `async` function type
* converted to an `@isolated(any)` function type
* converted to a non-Sendable function type and has the correct isolation for the context that does the conversion

Due to the ambiguity between the `nonisolated` modifier and a type-inferred closure parameter, most notably disambiguating `{ nonisolated parameter in ... }` as a modifier followed by a single parameter vs both as a bound pair of tokens, the use of parentheses for a parameter list is required when `nonisolated` is specified.

```swift
{ nonisolated (parameter) in ... }
```

Opting out of `@inheritsIsolation` can be achieved by explicitly annotating the closure argument as `nonisolated`.

`@_inheritActorContext` is currently used by the `Task` initializer in the standard library which should be updated to use `@inheritsIsolation` instead.

One further related clarification of isolation inheritence is that non-`@Sendable` local functions should always inherit their enclosing isolation (unless explicitly `nonisolated` or isolated some other way).

### Distributed actor isolation

`isolated` capture parameter works with distributed actors, however only statically "known to be local" distributed actors may be promoted to `isolated`. Currently, this is achieved only through an `isolated` distributed actor type, meaning that a task can only be made isolated to a distributed actor if the value already was isolated, like this:

```swift
import Distributed

distributed actor D {
  func isolateSelf() {
    // 'self' is isolated
    Task { [isolated self] in print("OK") } // OK: self was isolated
  }

  nonisolated func bad() {
    // 'self' is not isolated
    Task { [isolated self] in print("BAD") } // error: self was not isolated, and may be remote
  }
}

func isolate(d: isolated D) {
  Task { [isolated d] in print("OK") } // OK: d was isolated, thus known-to-be-local
}

func isolate(d: D) {
  Task { [isolated d] in print("OK") } // error: d was not isolated, and may be remote
}
```

While it is technically possible to enqueue work on a remote distributed actor reference, the enqueue on such an actor will always immediately crash. Because of that, we err on the side of disallowing such illegal code. [Future directions](#future-directions) discusses how this can be made more powerful when it is known that an actor is local. It is also worth noting the `da.whenLocal { isolated da in ... }` API which allows dynamically recovering an isolated distributed actor reference after it has dynamically been checked for locality.

## Source compatibility

It is possible that existing code could have a closure that names a type-inferred parameter `nonisolated`:
```swift
{ nonisolated in print(nonisolated) }
```
but with this proposed change, `nonisolated` in this case would instead be interpreted as the contextual keyword specifying the formal isolation of the closure. Such code would then result in a compilation error when trying to use a parameter named `nonisolated`.

The change to `Task.init` in the standard library does have the potential to isolate some closures that previously were inferred to be `nonisolated`. Prior behavior in those cases could be restored, if desired, by explicitly declaring the closure as `nonisolated`.

It is worth noting that this does not affect the isolation semantics for actor-isolated types that make use of isolated parameters. It is currently impossible to access self in these cases, and even with this new inheritance rule that remains true.

```swift
actor MyActor {
  var mutableState = 0

  func isolatedParameter(_ actor: isolated any Actor) {
    self.mutableState += 1 // invalid

    Task {
      self.mutableState += 1 // invalid
    }
  }
}

@MainActor
class MyClass {
  var mutableState = 0

  func isolatedParameter(_ actor: isolated any Actor) {
    self.mutableState += 1 // invalid

    Task {
      self.mutableState += 1 // invalid
    }
  }
}
```

## ABI compatibility

The language change does not add or affect ABI since formal isolation is already part of a closure's type regardless of whether it is explicitly specified. The `Task.init` change does not impact ABI since the function is annotated with `@_alwaysEmitIntoClient` and therefore has no ABI.

## Implications on adoption

none

## Alternatives considered

`@nonisolated` in attribute form was considered to avert the potential for source breakage, but requires an unintuitive inconsistency in the language for when `@` is required vs needs to be avoided.

One alternative to `@inheritsIsolation` is to not use `Task` in combination with non-Sendable types in this way, restructuring the code to avoid needing to rely on isolation inheritance in the first place.

```swift
class NonSendableType {
    private var internalState = 0

    func doSomeStuff(isolatedTo actor: isolated any Actor) async throws {
        try await Task.sleep(for: .seconds(1))
        print(self.internalState)
    }
}
```

Despite this being a useful pattern, it does not address the underlying inheritance semantic differences.

There has also been discussion about the ability to make synchronous methods on actors. The scope of such a change is much larger than what is covered here and would still not address the underlying differences.

## Future directions

### weak isolated

Explore support for explicitly `isolated` closure captures to additionally be specified as `weak`.

### "Known to be local" distributed actors and isolation

Distributed actors have a property that is currently not exposed in the type system that is "known to be local". If a distributed actor is known to be local, code may become isolated to it.

Once the locality of a type is expressed in the type system, the following would become possible:

```swift
let worker: local Worker

// silly example, showcasing isolating on a known-to-be-local distributed actor
func work(item: Item) async {
  await Task { [isolated worker] in
    worker.work(on: item)
  }.value
}
```

## Acknowledgments

Thank you to Franz Busch and Aron Lindberg for looking at the underlying problem so closely and suggesting alternatives. Thank you to Holly Borla for helping to clarify the current behavior, as well as suggesting a path forward that resulted in a much simpler and less-invasive change.
