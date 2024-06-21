# Distributed Actor Runtime

* Proposal: [SE-0344](0344-distributed-actor-runtime.md)
* Authors: [Konrad 'ktoso' Malawski](https://github.com/ktoso), [Pavel Yaskevich](https://github.com/xedin), [Doug Gregor](https://github.com/DougGregor), [Kavon Farvardin](https://github.com/kavon), [Dario Rexin](https://github.com/drexin), [Tomer Doron](https://github.com/tomerd)
* Review Manager: [Joe Groff](https://github.com/jckarter/)
* Status: **Implemented (Swift 5.7)**
* Implementation:
  * Partially available in [recent `main` toolchain snapshots](https://swift.org/download/#snapshots) behind the `-enable-experimental-distributed` feature flag.
  * This flag also implicitly enables `-enable-experimental-concurrency`.
* Review threads
  * [First Review](https://forums.swift.org/t/se-0344-distributed-actor-runtime/55525) ([summary](https://forums.swift.org/t/returned-for-revision-se-0344-distributed-actor-runtime/55836))
  * [Second Review](https://forums.swift.org/t/se-0344-second-review-distributed-actor-runtime/56002) ([summary](https://forums.swift.org/t/accepted-se-0344-distributed-actor-runtime/56416))


## Table of Contents

- [Distributed Actor Runtime](#distributed-actor-runtime)
  - [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
      - [Useful links](#useful-links)
  - [Motivation](#motivation)
      - [Example scenario](#example-scenario)
    - [Caveat: Low-level implementation details](#caveat-low-level-implementation-details)
  - [Detailed design](#detailed-design)
    - [The `DistributedActorSystem` protocol](#the-distributedactorsystem-protocol)
    - [Implicit distributed actor properties](#implicit-distributed-actor-properties)
    - [Initializing distributed actors](#initializing-distributed-actors)
    - [Distributed actor initializers](#distributed-actor-initializers)
      - [Initializing `actorSystem` and `id` properties](#initializing-actorsystem-and-id-properties)
    - [Ready-ing distributed actors](#ready-ing-distributed-actors)
      - [Ready-ing distributed actors, exactly once](#ready-ing-distributed-actors-exactly-once)
    - [Resigning distributed actor IDs](#resigning-distributed-actor-ids)
    - [Resolving distributed actors](#resolving-distributed-actors)
    - [Invoking distributed methods](#invoking-distributed-methods)
      - [Sender: Invoking a distributed method](#sender-invoking-a-distributed-method)
      - [Sender: Serializing and sending invocations](#sender-serializing-and-sending-invocations)
      - [Recipient: Receiving invocations](#recipient-receiving-invocations)
      - [Recipient: Deserializing incoming invocations](#recipient-deserializing-incoming-invocations)
      - [Recipient: Resolving the recipient actor instance](#recipient-resolving-the-recipient-actor-instance)
      - [Recipient: The `executeDistributedTarget` method](#recipient-the-executedistributedtarget-method)
      - [Recipient: Executing the distributed target](#recipient-executing-the-distributed-target)
      - [Recipient: Collecting result/error from invocations](#recipient-collecting-resulterror-from-invocations)
  - [Amendments](#amendments)
    - [Initializers no longer need to accept a single DistributedActorSystem](#initializers-no-longer-need-to-accept-a-single-distributedactorsystem)
  - [Future work](#future-work)
    - [Variadic generics removing the need for `remoteCallVoid`](#variadic-generics-removing-the-need-for-remotecallvoid)
    - [Identifying, evolving and versioning remote calls](#identifying-evolving-and-versioning-remote-calls)
      - [Default distributed call target identification scheme](#default-distributed-call-target-identification-scheme)
      - [Compression techniques to avoid repeatedly sending large identifiers](#compression-techniques-to-avoid-repeatedly-sending-large-identifiers)
      - [Overlap with general ABI and versioning needs in normal Swift code](#overlap-with-general-abi-and-versioning-needs-in-normal-swift-code)
      - [Discussion: User provided target identities](#discussion-user-provided-target-identities)
    - [Resolving `DistributedActor` protocols](#resolving-distributedactor-protocols)
    - [Passing parameters to `assignID`](#passing-parameters-to-assignid)
  - [Alternatives considered](#alternatives-considered)
    - [Define `remoteCall` as protocol requirement, and accept `[Any]` arguments](#define-remotecall-as-protocol-requirement-and-accept-any-arguments)
    - [Constraining arguments, and return type with of `remoteCall` with `SerializationRequirement`](#constraining-arguments-and-return-type-with-of-remotecall-with-serializationrequirement)
    - [Hardcoding the distributed runtime to make use of `Codable`](#hardcoding-the-distributed-runtime-to-make-use-of-codable)
  - [Acknowledgments & Prior art](#acknowledgments--prior-art)
  - [Source compatibility](#source-compatibility)
  - [Effect on ABI stability](#effect-on-abi-stability)
  - [Effect on API resilience](#effect-on-api-resilience)
  - [Changelog](#changelog)

## Introduction

With the recent introduction of [actors](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0306-actors.md) to the language, Swift gained powerful and foundational building blocks for expressing *thread-safe* concurrent programs. Actors guarantee thread-safety thanks to actor-isolation of mutable state they encapsulate.

In [SE-0336: Distributed Actor Isolation][isolation] we took it a step further, guaranteeing complete isolation of state with distributed actor-isolation, and setting the stage for `distributed` method calls to be performed across process and node boundaries.

This proposal focuses on the runtime aspects of making such remote calls possible, their exact semantics and how developers can provide their own `DistributedActorSystem` implementations to hook into the same language mechanisms, extending Swift's distributed actor model to various environments (such as cross-process communication, clustering, or even client/server communication).

#### Useful links

It is recommended, though not required, to familiarize yourself with the prior proposals before reading this one:

- [SE-0336: Distributed Actor Isolation][isolation] — a detailed proposal
- Distributed Actor Runtime (this proposal)

Feel free to reference the following library implementations which implement this proposal's library side of things:

- [Swift Distributed Actors Library](https://www.swift.org/blog/distributed-actors/) — a reference implementation of a *peer-to-peer cluster* for distributed actors. Its internals depend on the work in progress language features and are dynamically changing along with these proposals. It is a realistic implementation that we can use as reference for these design discussions.

## Motivation

With distributed actor-isolation checking laid out in [SE-0336: Distributed Actor Isolation][isolation], we took the first step towards enabling remote calls being made by invoking `distributed func` declarations on distributed actors. The isolation model and serialization requirement checks in that proposal outline how we can guarantee the soundness of such distributed actor model at compile time.

Distributed actors enable developers to build their applications and systems using the concept of actors that may be "local" or "remote", and communicate with them regardless of their location. Our goal is to set developers free from having to re-invent ad-hoc approaches to networking, serialization and error handling every time they need to embrace distributed computing.

Instead, we aim to embrace a co-operative approach to the problem, in which:

1. the Swift language, compiler, and runtime provide the necessary isolation checks and runtime hooks for distributed actor lifecycle management, and distributed method calls that can be turned into "messages" that can be sent to remote peers,
2. `DistributedActorSystem` library implementations, hook into the language provided cut-points, take care of the actual message interactions, e.g. by sending messages representing remote distributed method calls over the network,
3. `distributed actor` authors, who want to focus on getting things done, express their distributed API boundaries and communicate using them. They may have opinions about serialization and specifics of message handling, and should be able to configure and use the `DistributedActorSystem` of their choice to get things done.

In general, we propose to embrace the actor style of communication for typical distributed system development, and aim to provide the necessary tools in the language, and runtime to make this a pleasant and nice default go-to experience for developers.

Distributed actors may not serve *all* possible use-cases where networking is involved, but we believe a large group of applications and systems will benefit from them, as the ecosystem gains mature `DistributedActorSystem` implementations.

#### Example scenario

In this proposal we will focus only on the runtime aspects of distributed actors and methods, i.e. what happens in order to create, send, and receive messages formed when a distributed method is called on a remote actor. For more details on distributed actor isolation and other compile-time checks, please refer to [SE-0336: Distributed Actor Isolation][isolation].

We need to pass around distributed actors in order to invoke methods on them at some later point in time. We need those actors to declare `distributed` methods such that we have something we can message them with, and there must be some lifecycle and registration mechanisms related to them.

One example use case we can keep in mind is a simple turn-based `Game` which showcases most of the capabilities we come to expect of distributed actors:

```swift
distributed actor Player {
  // ...

  distributed func makeMove() -> Move { ... }

  distributed func gameFinished(result: GameResult) {
    if result.winner == self {
      print("I WON!")
    } else {
      print("Player \(result.winner) won the game.")
    }
  }
}

distributed actor Game {
  var state: GameState = ...

  // players can be located on different nodes
  var players: Set<Player> = []

  distributed func playerJoined(_ player: Player) {
    others.append(player)
    if others.count >= 2 { // we need 2 other players to start a game
      Task { try await self.start() }
    }
  }

  func start() async throws {
    state = .makeNewGameState(with: players)
    while !state.finished {
      for player in players {
        let move = try await p.makeMove() // TODO: handle failures, e.g. "move timed-out"
        state.apply(move, by: player)
      }
    }

    let winner = state.winner
    try await game.finishedResult
  }
}
```

This code snippet showcases what kind of distributed actors one might want to implement – they represent addressable identities in a system where players may be hosted on different hosts or devices, and we'd like to communicate with any of them from the `Game` actor which manages the entire game's state. Players may be on the same host as the `Game` actor, or on different ones, but we never have to change the implementation of `Game` to deal with this – thanks to distributed actors and the concept of location transparency, we can implement this piece of code once, and run it all locally, or distributed without changing the code specifically for either of those cases.

### Caveat: Low-level implementation details

This proposal includes low-level implementation details in order to showcase how one can use to build a real, efficient, and extensible distributed actor system using the proposed language runtime. It is primarily written for distributed actor system authors, which need to understand the underlying mechanisms which distributed actors use.

End users, who just want to use _distributed actors_, and not necessarily _implement_ a distributed actor system runtime, do not need to dive deep as deep into this proposal, and may be better served by reading [SE-0366: Distributed Actor Isolation][isolation] which focuses on how distributed actors are used. Reading this — runtime — proposal, however, will provide additional insights as to why distributed actors are isolated the way they are.

This proposal focuses on how a distributed actor system runtime can be implemented. Because this language feature is extensible, library authors may step in and build their own distributed actor runtimes. It is expected that there will be relatively few, but solid actor system implementations eventually, yet their use would apply to many many more end-users than actor system developers.

## Detailed design

This section is going to deep dive into the runtime details and its interaction with user provided `DistributedActorSystem` implementations. Many of these aspects are not strictly necessary to internalize by end-user/developer, who only wants to write some distributed actors and have them communicate using *some* distributed actor system.

### The `DistributedActorSystem` protocol

At the core of everything distributed actors do, is the `DistributedActorSystem` protocol. This protocol is open to be implemented by anyone, and can be used to extend the functionality of distributed actors to various environments.

Building a solid actor system implementation is not a trivial task, and we only expect a handful of mature implementations to take the stage eventually.

> At the time of writing, we–the proposal authors–have released a work in progress [peer-to-peer cluster actor system implementation](https://www.swift.org/blog/distributed-actors/) that is tracking this evolving language feature. It can be viewed as a reference implementation for the language features and `DistributedActorSystem` protocol discussed in this proposal.

Below we present the full listing of the `DistributedActorSystem` protocol, and we'll be explaining the specific methods one by one as we go:

```swift
// Module: _Distributed

protocol DistributedActorSystem: Sendable {
  /// The type of `ID` assigned to a distributed actor while initializing with this actor system.
  /// The identity should be meaningfully unique, in the sense that ID equality should mean referring to the
  /// same distributed actor.
  ///
  /// A distributed actor created using a specific actor system will use the system's `ActorID` as
  /// the `ID` type it stores and for its `Hashable` implementation.
  ///
  /// ### Implicit distribute actor `Codable` conformance
  /// If the `ActorID` (and therefore also the `DistributedActor.ID`) conforms to `Codable`,
  /// the `distributed actor` will gain an automatically synthesized conformance to `Codable` as well.
  associatedtype ActorID: Sendable & Hashable

  /// The specific type of the invocation encoder that will be created and populated
  /// with details about the invocation when a remote call is about to be made.
  ///
  /// The populated instance will be passed to the `remoteCall` from where it can be
  /// used to serialize into a message format in order to perform the remote invocation.
  associatedtype InvocationEncoder: DistributedTargetInvocationEncoder

  /// The specific type of invocation decoder used by this actor system.
  ///
  /// An instance of this type must be passed to `executeDistributedTarget` which
  /// extracts arguments and applies them to the local target of the invocation.
  associatedtype InvocationDecoder: DistributedTargetInvocationDecoder

  /// The serialization requirement that will be applied to all distributed targets used with this system.
  ///
  /// An actor system is still allowed to throw serialization errors if a specific value passed to a distributed
  /// func violates some other restrictions that can only be checked at runtime, e.g. checking specific types
  /// against an "allow-list" or similar. The primary purpose of the serialization requirement is to provide
  /// compile time hints to developers, that they must carefully consider evolution and serialization of
  /// values passed to and from distributed methods and computed properties.
  associatedtype SerializationRequirement
    where SerializationRequirement == InvocationEncoder.SerializationRequirement,
          SerializationRequirement == InvocationDecoder.SerializationRequirement

  // ==== ---------------------------------------------------------------------
  // - MARK: Actor Lifecycle

  /// Called by a distributed when it begins its initialization (in a non-delegating init).
  ///
  /// The system should take special care to not assign two actors the same `ID`, and the `ID`
  /// must remain valid until it is resigned (see `resignID(_:)`).
  func assignID<Actor>(_ actorType: Actor.Type) -> ActorID
      where Actor: DistributedActor,
            Actor.ID == ActorID

  /// Automatically called by in every distributed actor's non-delegating initializer.
  ///
  /// The call is made specifically before the `self` of such distributed actor is about to
  /// escape, e.g. via a function call, closure or otherwise. If no such event occurs the
  /// call is made at the end of the initializer.
  ///
  /// The passed `actor` is the `self` of the initialized actor, and its `actor.id` is expected
  /// to be of the same value that was assigned to it in `assignID`.
  ///
  /// After the ready call returns, it must be possible to resolve it using the 'resolve(_:as:)'
  /// method on the system.
  func actorReady<Actor>(_ actor: Actor)
      where Actor: DistributedActor,
            Actor.ID == ActorID

  /// Called when the distributed actor is deinitialized (or has failed to finish initializing).
  ///
  /// The system may release any resources associated with this actor id, and should not make
  /// further attempts to deliver messages to the actor identified by this identity.
  func resignID(_ id: ActorID)

  // ==== ---------------------------------------------------------------------
  // - MARK: Resolving distributed actors

  /// Resolve a local or remote actor address to a real actor instance, or throw if unable to.
  /// The returned value is either a local actor or proxy to a remote actor.
  ///
  /// Resolving an actor is called when a specific distributed actors `init(from:)`
  /// decoding initializer is invoked. Once the actor's identity is deserialized
  /// using the `decodeID(from:)` call, it is fed into this function, which
  /// is responsible for resolving the identity to a remote or local actor reference.
  ///
  /// If the resolve fails, meaning that it cannot locate a local actor managed for
  /// this identity, managed by this transport, nor can a remote actor reference
  /// be created for this identity on this transport, then this function must throw.
  ///
  /// If this function returns correctly, the returned actor reference is immediately
  /// usable. It may not necessarily imply the strict *existence* of a remote actor
  /// the identity was pointing towards, e.g. when a remote system allocates actors
  /// lazily as they are first time messaged to, however this should not be a concern
  /// of the sending side.
  ///
  /// Detecting liveness of such remote actors shall be offered / by transport libraries
  /// by other means, such as "watching an actor for termination" or similar.
  func resolve<Actor>(_ id: ActorID, as actorType: Actor.Type) throws -> Actor?
      where Actor: DistributedActor,
            Actor.ID: ActorID,
            Actor.SerializationRequirement == Self.SerializationRequirement

  // ==== ---------------------------------------------------------------------
  // - MARK: Remote Target Invocations

  /// Invoked by the Swift runtime when a distributed remote call is about to be made.
  ///
  /// The returned `InvocationEncoder` will be populated with all
  /// generic substitutions, arguments, and specific error and return types
  /// that are associated with this specific invocation.
  func makeInvocationEncoder() -> InvocationEncoder

  // We'll discuss the remoteCall method in detail in this proposal.
  // It cannot be declared as protocol requirement, and remains an ad-hoc
  // requirement like this:
  /// Invoked by the Swift runtime when making a remote call.
  ///
  /// The `invocation` is the arguments container that was previously created
  /// by `makeInvocationEncoder` and has been populated with all arguments.
  ///
  /// This method should perform the actual remote function call, and await for its response.
  ///
  /// ## Errors
  /// This method is allowed to throw because of underlying transport or serialization errors,
  /// as well as by re-throwing the error received from the remote callee (if able to).
  ///
  /// Ad-hoc protocol requirement.
  func remoteCall<Actor, Failure, Success>(
      on actor: Actor,
      target: RemoteCallTarget,
      invocation: inout InvocationEncoder,
      throwing: Failure.Type,
      returning: Success.Type
  ) async throws -> Success
      where Actor: DistributedActor,
            Actor.ID == ActorID,
            Failure: Error,
            Success: Self.SerializationRequirement

  /// Invoked by the Swift runtime when making a remote call to a `Void` returning function.
  ///
  /// ( ... Same as `remoteCall` ... )
  ///
  /// Ad-hoc protocol requirement.
  func remoteCallVoid<Actor, Error>(
      on actor: Actor,
      target: RemoteCallTarget,
      invocation: inout InvocationEncoder,
      throwing: Failure.Type
  ) async throws
      where Actor: DistributedActor,
            Actor.ID == ActorID,
            Failure: Error
}

/// A distributed 'target' can be a `distributed func` or `distributed` computed property.
///
/// The actor system should encode the identifier however it sees fit,
/// and transmit it to the remote peer in order to invoke identify the target of an invocation.
public struct RemoteCallTarget: Hashable {
  /// The mangled name of the invoked distributed method.
  ///
  /// It contains all information necessary to lookup the method using `executeDistributedActorMethod(...)`
  var mangledName: String { ... }

  /// The human-readable "full name" of the invoked method, e.g. 'Greeter.hello(name:)'.
  var fullName: String { ... }
}
```

In the following sections, we will be explaining how the various methods of a distributed system are invoked by the Swift runtime.

### Implicit distributed actor properties

Distributed actors have two properties that are crucial for the inner workings of actors that we'll explore during this proposal: the `id` and `actorSystem`.

These properties are synthesized by the compiler, in every `distributed actor` instance, and they witness the `nonisolated` property requirements defined on the `DistributedActor` protocol.

The `DistributedActor` protocol (defined in [SE-0336][isolation]), defines those requirements:

```swift
protocol DistributedActor {
  associatedtype ActorSystem: DistributedActorSystem

  typealias ID = ActorSystem.ActorID
  typealias SerializationRequirement: ActorSystem.SerializationRequirement

  nonisolated var id: ID { get }
  nonisolated var actorSystem: ActorSystem { get }

  // ...
}
```

which are witnessed by *synthesized properties* in every specific distributed actor instance.

Next, we will discuss how those properties get initialized, and used in effectively all aspects of a distributed actor's lifecycle.

### Initializing distributed actors

At runtime, a *local* `distributed actor` is effectively the same as a local-only `actor`. The allocated `actor` instance is a normal `actor`. However, its initialization is a little special, because it must interact with its associated actor system to make itself available for remote calls.

We will focus on non-delegating initializers, as they are the ones where distributed actors cause additional things to happen.

> Please note that **initializing** a distributed actor with its `init` always returns a **local** reference to a new actor. The only way to obtain a a remote reference is by using the `resolve(id:using:)` method, which is discussed in [Resolving Distributed Actors](#resolving-distributed-actors).

Distributed actor initializers inject a number of calls into specific places of the initializer's body. These calls allow for the associated actor system to manage the actor's identity, and availability to remote calls. Before we dive into the details, the following diagram outlines the various calls that will be explained in this section:

```
┌────────────────────────────┐               ┌──────────────────────────┐
│  distributed actor MyActor │               │ MyDistributedActorSystem │
└────────────────────────────┘               └──────────────────────────┘
      │                                                    │
 init(...)                                                 │
      │                                                    │
      │── // self.id = actorSystem.assignID(Self.self) ───▶│ Generate and reserve ID
      │                                                    │
      │   // self.actorSystem = system                     │
      │                                                    │
      │   <initialize other properties...>                 │
      │                                                    │
      │── // actorSystem.actorReady(self) ────────────────▶│ Store a mapping (ID -> some DistributedActor)
      │                                                    │
     ...                                                  ...
      │                                                    │
      ◌ deinit ─ // actorSystem.resignID(self.id) ────────▶│ Remove (ID -> some DistributedActor) mapping
                                                           │
                                                          ...
```

### Distributed actor initializers

A non-delegating initializer of a type must *fully initialize* it. The place in code where an actor becomes fully initialized has important and specific meaning to actor isolation which is defined in depth in [SE-0327: On Actors and Initialization](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0327-actor-initializers.md). Not only that, but once fully initialized it is possible to escape `self` out of a (distributed) actor's initializer. This aspect is especially important for distributed actors, because it means that once fully initialized they _must_ be registered with the actor system as they may be sent to other distributed actors and even sent messages to.

All non-delegating initializers must store an instance of a `DistributedActorSystem` conforming type into their `self.actorSystem` property. 

This is an improvement over the initially proposed semantics in [SE-0336: Distributed Actor Isolation][isolation], where the initializers must have accepted a _single_ distributed actor system argument and would initialize the synthesized stored property automatically. 

This proposal amends this behavior to the following semantics:

- the default initializer gains a `actorSystem: Self.ActorSystem` parameter.
- other non-delegating initializers must initialize the `self.actorSystem` property explicitly.
  - it is recommended to accept an actor system argument and store it 
  - technically it is also possible to store a global actor system to this property, however this is generally an anti-pattern as it hurts testability of such actor (i.e. it becomes impossible to swap the actor system for a "for testing" one during test execution).

```swift
distributed actor DA {
  // synthesized:
  // init(actorSystem: Self.ActorSystem) {} // ✅ no user defined init, so we synthesize a default one
}

distributed actor DA2 {
  init(system: Self.ActorSystem) { // ⚠️ ok declaration, but self.actorSystem was not initialized
    // ❌ error: implicit stored property 'actorSystem' was not initialized
  }

  init(other: Int, actorSystem system: Self.ActorSystem) { // ✅ ok
    self.actorSystem = system
  }
}
```

Now in the next sections, we will explore in depth why this parameter was necessary to enforce to begin with.

#### Initializing `actorSystem` and `id` properties

The two properties (`actorSystem` and `id`) are synthesized _stored_ properties in the body of every actor.

Users must initialize the `actorSystem`, however management of the `id` is left up to the compiler to synthesize. All properties of an actor have to be initialized for the type to become fully initialized, same as with any other type. However, the initialization of `id` is left to the compiler in order to streamline the initialization, as well as keep the _strict_ contract between the stored actor system and that the `ID` _must_ be assigned by that exact actor system. It would be illegal to just assign an `id` from some other source because of how tightly it relates to the actors lifecycle.

The compiler synthesizes code that does this in any designated initializer of distributed actors:

```swift
distributed actor DA {
  // let id: ID
  // let actorSystem: ActorSystem

  init(actorSystem: Self.ActorSystem) {
    self.actorSystem = actorSystem
    // ~~~ synthesized ~~~
    // ...
    // ~~~ end of synthesized ~~~
  }
}
```

Synthesizing the id assignment means that we need to communicate with the `system` used to initialize the distributed actor, for it is the `ActorSystem` that allocates and manages identifiers. In order to obtain a fresh `ID` for the actor being initialized, we need to call `system`'s `assignID` method. This is done before any user-defined code is allowed to run in the actors designated initializer, like this:

```swift
distributed actor DA {
  let number: Int

  init(system: ActorSystem) {
    self.actorSystem = system
    // ~~ injected property initialization ~~
    // self.id = system.assignID(Self.self)
    // ~~ end of injected property initialization ~~

    // user-defined code follows...
    self.number = 42

    // ...
  }
}
```

### Ready-ing distributed actors

So far, the initialization process was fairly straightforward. We only needed to find a way to initialize the stored properties, and that's it. There is one more step though that is necessary to make distributed actors work: "ready-ing" the actor.

As the actor becomes fully initialized, the type system allows escaping its `self` through method calls or closures. There are a number of rules which govern the isolation state of `self` of any actor during its initializer, which are fully explained in: [SE-0327: On Actor Initialization](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0327-actor-initializers.md). Distributed actor initializers are subject to the same rules, and in addition to that they inject an `actorReady(self)` at the point where self would have become nonisolated under the rules explained in SE-0327.

This call is necessary in order for the distributed actor system to be able to resolve an incoming `ID` (that it knows, since it assigned it) to a specific distributed actor instance (which it does not know, until `actorReady` is called on the system). This means that there is a state between the `assignID` and `actorReady` calls, during which the actor system cannot yet properly resolve the actor.

A distributed actor becomes "ready", and transparently invokes `actorSystem.ready(self)`, during its non-delegating initializer just _before_ the actor's `self` first escaping use, or at the end of the initializer if no explicit escape is found.

This rule is not only simple to remember, but also consistent between synchronous and asynchronous initializers. The rule plays also very well with the flow-isolation treatment of self in plain actors.

The following snippets illustrate where the ready call is emitted by the compiler:

```swift
distributed actor DA {
  let number: Int

  init(sync system: ActorSystem) {
    self.actorSystem = system
    // << self.id = system.assignID(Self.self)
    self.number = 42
    // << system.actorReady(self)
  }

  init(sync system: ActorSystem) {
    self.actorSystem = system
    // << self.id = system.assignID(Self.self)
    self.number = 42
    // << system.actorReady(self)
    Task.detached { // escaping use of `self`
      await self.hello()
    }
  }
}
```

If the self of the actor were to be escaped on multiple execution paths, the ready call is injected in all appropriate paths, like this:

```swift
distributed actor DA {
  let number: Int
  init(number: Int, system: ActorSystem) async {
    self.actorSystem = system
    // << self.id = system.assignID(Self.self)
    if number % 2 == 0 {
      print("even")
      self.number = number
      // << system.actorReady(self)
      something(self)
    } else {
      print("odd")
      self.number = number
      // << system.actorReady(self)
      something(self)
    }
  }
}
```

Special care needs to be taken about the distributed actor and actor system interaction in the time between the `assignID` and `actorReady` calls, because during this time the system is unable to *deliver* an invocation to the target actor. However, it is always able to recognize that an ID is known, but just not ready yet – the system did create and assign the ID after all.

This should not be an issue for developers using distributed actors, but actor system authors need to be aware of this interval between the actor ID being reserved and readies. We suggest "reserving" the ID immediately in `assignID` in order to avoid issuing the same ID to multiple actors which can yield unexpected behavior when handling incoming messages.

Another thing to be aware of is "long" initializers, which take a long time to complete which may sometimes be the case with asynchronous initializers. For example, consider this initializer which performs a lot of work on the passed in items before returning:

```swift
init(items: [Item], system: ActorSystem) async {
  self.actorSystem = system
  // << self.id = system.assignID(Self.self)
  for await item in items {
    await compute(item)
  }
  // ...
  // ...
  // ?? what if init "never" returns" ??
  // ...
  // ...
  // << system.actorReady(self)
}
```

This is arguably problematic for any class, struct or actor, however for distributed actors this also means that the period of time during an ID was assigned and will finally be readied can be potentially quite long. In general, we discourage such "long running" initializers as they make use of the actor in distribution impossible until it is readied. On the other hand, though, it can only be used in distribution once the initializer returns in any case so this is a similar problem to any long running initializer.

#### Ready-ing distributed actors, exactly once

Another interesting case the `actorReady` synthesis in initializers needs to take care of is triggering the `actorReady` call only *once*, as the actor first becomes fully initialized. The following snippet does a good job showing an example of where it can manifest:

```swift
distributed actor DA {
  var int: Int
  init(system: ActorSystem) async {
    self.actorSystem = system
    var loops = 10
    while loops > 0 {
      self.int = loops
      // ~ AT THE FIRST ITERATION ~
      // ~ become fully initialized ~
      // ...
      escape(self)

      loops -= 1
    }
  }
}
```

This actor performs a loop during which it assigns values to `self.int`, the actor becomes fully initialized the first time this loop runs.

We need to emit the `actorReady(self)` call, only once, and we should not repeatedly call the actor system's `actorReady` method which would force system developers into weirdly defensive implementations of this method. Thankfully, this is possible to track in the compiler, and we can emit the ready call only once, based on internal initialization marking mechanisms (that store specific bits for every initialized field).

The synthesized (pseudo-)code therefore is something like this:

```swift
distributed actor DA {
  var int: Int
  init(system: ActorSystem) {
     self.actorSystem = system
    // << self.id = system.assignID(Self.self)
    // MARK INITMAP[id] = INITIALIZED

    var loops = 10
    while loops > 0 {
      self.int = loops
      // MARK INITMAP[int] = INITIALIZED
      // INITMAP: FULLY INITIALIZED
      //
      // IF INITMAP[IMPLICIT_HOP_TO_SELF] != DONE {
      //   << <hop to self executor>
      //   MARK INITMAP[IMPLICIT_HOP_TO_SELF] = DONE
      // }
      //
      // IF INITMAP[ACTOR_READY] != INITIALIZED {
      //   << system.actorReady(self)
      //   MARK INITMAP[ACTOR_READY] = INITIALIZED
      // }
      escape(self)

      loops -= 1
    }
  }
}
```

Using this technique we are able to emit the ready call only once, and put off the complexity of dealing with repeated ready calls from distributed actor system library authors.

> The same technique is used to avoid hopping to the self executor 10 times, and the implicit hop-to-self is only performed once, on the initial iteration where the actor became fully initialized.

Things get more complex in the face of failable as well as throwing initializers. Specifically, because we not only have to assign identities, we also need to ensure that they are resigned when the distributed actor is deallocated. In the simple, non-throwing initialization case this is simply done in the distributed actor's `deinit`. However, some initialization semantics make this more complicated.

### Resigning distributed actor IDs

In addition to assigning `ID` instances to specific actors as they get created, we must also *always* ensure the `ID`s assigned are resigned as their owning actors get destroyed.

Resigning an `ID` allows the actor system to release any resources it might have held in association with this actor. Most often this means removing it from some internal lookup table that was used to implement the `resolve(ID) -> Self` method of a distributed actor, but it could also imply tearing down connections, clearing caches, or even dropping any in-flight messages addressed to the now terminated actor.

In the simple case this is trivially solved by deinitialization: we completely initialize the actor, and once it deinitializes, we invoke `resignID` in the actor's deinitializer:

```swift
deinit {
  // << self.actorSystem.resignID(self.id)
}
```

This also works with user defined deinitializers, where the resign call is injected as the *first* operation in the deinitializer:

```swift
// user-defined deinit
deinit {
  // << self.actorSystem.resignID(self.id)
  print("deinit \(self.id)")
}
```

Things get more complicated once we take into account the existence of *failable* and *throwing* initializers. Existing Swift semantics around those types of initializers, and their effect on if and when `deinit` is invoked mean that we need to take special care of them.

Let us first discuss [failable initializers](https://docs.swift.org/swift-book/LanguageGuide/Initialization.html#ID224), i.e. initializers which are allowed to assign `nil` during their initialization. As actors allow such initializers, distributed actors should too, in order to make the friction of moving from local-only to distributed actors as small as possible.

```swift
distributed actor DA {
  var int: Int

  init?(int: Int, system: ActorSystem) {
    self.actorSystem = system
    // << self.id = actorSystem.assignID(Self.self)
    // ...
    if int < 10 {
      // ...
      // << self.actorSystem.resignID(self.id)
      return nil
    }
    self.int = int
    // << self.actorSystem.actorReady(self)
  }

  // deinit {
  //   << self.actorSystem.resignID(self.id)
  // }
}
```

Due to rules about actor and class init/deinit, when we `return nil` from a failable initializer, its deinitializer *does not run* (!). Because of this, we cannot rely on the deinit to resign the ID as we'd leave an un-used, but still registered identity hanging in the actor system, and the `resignID` is injected just before the "failing return" from such an initializer. This is done transparently, and neither distributed actor developers nor actor system developers need to worry about this: the ID is always resigned properly.

> This does mean that `resignID` may be called without `actorReady` having ever been called! The system should react to this as it would to any usual resignID and free any resources associated with the identifier.

Next, we need to discuss *throwing* initializers, and their multiple paths of execution. Again, rules about class and actor deinitialization, are tightly related to whether a type's `deinit` will be executed or not, so let us analyze the following example:

```swift
distributed actor DA {
  var int: Int

  init(int: Int, system: ActorSystem) throws {
    self.actorSystem = system
    // << self.id = system.assignID(Self.self)
    // ...
    if int <= 1 {
      // ...
      // << self.actorSystem.resignID(self.id)
      throw Boom() // [1]
    }

    if int <= 2 {
      self.int = int
      // ~ become fully initialized ~
      throw Boom() // [2]
    }

    throw Boom() // Boom for good measure... (same as [2] though)
    // theoretically, the ready call is inserted at the end of the init:
    // << system.actorReady(self)
  }

  init(int: Int, system: ActorSystem) async throws {
    // << self.id = system.assignID(Self.self)
    // ...
    if int <= 1 {
      // ...
      // << self.actorSystem.resignID(self.id)
      throw Boom() // [1]
    }

    if int <= 2 {
      self.int = int
      // ~ become fully initialized ~
      // << system.actorReady(self)
      throw Boom() // [2]
    }

    throw Boom() // Boom for good measure... (same as [2] though)
  }

  // deinit {
  //   << self.actorSystem.resignID(self.id)
  // }
}
```

The actor shown above both has state that it needs to initialize, and it is going to throw. It will throw either before becoming fully initialized `[1]`, or after it has initialized all of its stored properties `[2]`. Swift handles those two executions differently. Only a fully initialized reference type's `deinit` is going to be executed. This means that if the `init` throws at `[1]` we need to inject a `resignID` call there, while if it throws after becoming fully initialized, e.g. on line `[2]` we do not need to inject the `resignID` call, because the actor's `deinit` along with the injected-there `resignID` will be executed instead.

Both the synchronous and asynchronous initializers deal with this situation well, because the resign call must be paired with the assign, and if the actor was called ready before it calls `resignID` it does not really impact the resignation logic.

To summarize, the following are rules that distributed actor system implementors can rely on:

- `assignID(_:)` will be called exactly-once, at the very beginning of the initialization of a distributed actor associated with the system.
- `actorReady(_:)` will be called exactly-once, after all other properties of the distributed actor have been initialized, and it is ready to receive messages from other peers. By construction, it will also always be called after `assignID(_:)`.
- `resignID(_:)` will be called exactly-once as the actor becomes deinitialized, or fails to finish its initialization. This call will always be made after an `assignID(_:)` call. While there may be ongoing racy calls to the transport as the actor invokes this method, any such calls after `resignID(_:)` was invoked should be handled as if actor never existed to begin with.

Note that the system usually should not hold the actor with a strong reference, as doing so inhibits its ability to deinit until the system lets go of it.

### Resolving distributed actors

Every distributed actor type has a static "resolve" method with the following signature:

```swift
extension DistributedActor {
  public static func resolve(id: Self.ID, using system: Self.ActorSystem) throws -> Self {
    ...
  }
}
```

This method will return a distributed actor reference, or throw when the actor system is unable to resolve the reference.

The `resolve(id:using:)` method on distributed actors is an interesting case of the Swift runtime collaborating with the `DistributedActorSystem`. The Swift runtime implements this method as calling the passed-in actor system to resolve the ID, and if the system claims that this is a _remote_ reference, the Swift runtime will allocate a _remote_ distributed actor reference, sometimes called a "proxy" instance.

Its implementation can be thought of as follows:

```swift
extension DistributedActor {
  // simplified implementation
  static func resolve(id: Self.ID, using system: ActorSystem) throws -> Self {
    switch try system.resolve(id: id, as: Self.self) {
      case .some(let localInstance):
        return localInstance
      case nil:
        return <<make proxy instance of type Self>>(id: id, system: system)
    }
  }
}
```

Specifically, this calls into the `ActorSystem`'s `resolve(id:as:)` method which has a slightly different signature than the one defined on actors, specifically it can return `nil` to signal the instance is not found in this actor system, but we're able to proxy it.

> The **result** of `resolve(id:using:)` may be a **local instance** of a distributed actor, or a **reference to a remote** distributed actor.

The resolve implementation should be fast, and should be non-blocking. Specifically, it should *not* attempt to contact the remote peer to confirm whether this actor really exists or not. Systems should blindly resolve remote identifiers assuming the remote peer will be able to handle them. Some systems may after all spin up actor instances lazily, upon the first message sent to them etc.

Allocating the remote reference is implemented by the Swift runtime, by creating a fixed-size object that serves only the purpose of proxying calls into the `system.remoteCall`. The `_isDistributedRemoteActor()` function always returns `true` for such a reference.

If the system entirely fails to resolve the ID, e.g. because it was ill-formed or the system is unable to handle proxies for the given ID, it must throw with an error conforming to `DistribtuedActorSystemError`, rather than returning `nil`. An example implementation could look something like this:

```swift
final class ClusterSystem: DistributedActorSystem {
  private let lock: Lock
  private var localActors: [ActorID: AnyWeaklyHeldDistributedActor] // stored into during actorReady

  // example implementation; more sophisticated ones can exist, but boil down to the same idea
  func resolve<ID, Actor>(id: ID, as actorType: Actor.Type)
      throws -> Actor? where Actor: DistributedActor,
                           Actor.ID == Self.ActorID,
                           Actor.SerializationRequirement == Self.SerializationRequirement {
    if validate(id) == .illegal {
      throw IllegalActorIDError(id)
    }

    return lock.synchronized {
      guard let known = self.localActors[id] else {
        return nil // not local actor, but we can allocate a remote reference for it
      }

      return try known.as(Actor.self) // known managed local instance
    }
  }
}
```

The types work out correctly since it is the specific actor system that has _assigned_ the `ID`, and stored the specific distributed actor instance for the specific `ID`.

> **Note:** Errors thrown by actor systems should conform to the `protocol DistributedActorSystemError: Error {}` protocol. While it is just a marker protocol, it helps end users understand where an error originated.

Attempting to ready using one type, and resolve using another will cause a throw to happen during the resolve, e.g. like this:

```swift
distributed actor One {}
distributed actor Two {}

let one = One(system: cluster)
try Two.resolve(id: one.id, using: cluster)
// throws: DistributedActorResolveError.wrongType(found: One) // system specific error
```

This is only the case for local instances, though. For remote instances, by design, the local actor system does not track any information about them and as any remote call can fail anyway, the failures surface at call-site (as the remote recipient will fail to be resolved).

### Invoking distributed methods

Invoking a distributed method (or distributed computed property) involves a number of steps that occur on two "sides" of the call.

The local/remote wording which works well with actors in general can get slightly confusing here, because every call made "locally" on a "remote reference", actually results in a "local" invocation execution on the "remote system". Instead, we will be using the terms "**sender**" and "**recipient**" to better explain which side of a distributed call we are focusing on.

As was shown earlier, invoking a `distributed func` essentially can follow one of two execution paths:

- if the distributed actor instance was **local**:
  - the call is made directly, as if it was a plain-old local-only `actor`
- if the distributed actor was **remote**:
  - the call must be transformed into an invocation that will be offered to the `system.remoteCall(...)` method to execute

The first case is governed by normal actor execution rules. There might be a context switch onto the actor's executor, and the actor will receive and execute the method call as usual.

In this section, we will explain all the steps involved in the second, remote, case of a distributed method call. The invocations will be using two very important types that represent the encoding and decoding side of such distributed method invocations.

The full listing of those types is presented below:

```swift
protocol DistributedActorSystem: ... {
  // ...
  associatedtype InvocationEncoder: DistributedTargetInvocationEncoder
  associatedtype InvocationDecoder: DistributedTargetInvocationDecoder

  func makeInvocationEncoder() -> InvocationEncoder
}
```



```swift
public protocol DistributedTargetInvocationEncoder {
  associatedtype SerializationRequirement

  /// Record a type of generic substitution which is necessary to invoke a generic distributed invocation target.
  ///
  /// The arguments must be encoded order-preserving, and once `decodeGenericSubstitutions`
  /// is called, the substitutions must be returned in the same order in which they were recorded.
  mutating func recordGenericSubstitution<T>(_ type: T.Type) throws

  /// Record an argument of `Argument` type in this arguments storage.
  ///
  /// The argument name is provided to the `name` argument and can either be stored or ignored.
  /// The value of the argument is passed as the `argument`.
  ///
  /// Ad-hoc protocol requirement.
  mutating func recordArgument<Value: SerializationRequirement>(
    _ argument: RemoteCallArgument<Value>
  ) throws

  /// Record the error type thrown by the distributed invocation target.
  /// If the target does not throw, this method will not be called and the error type can be assumed `Never`.
  mutating func recordErrorType<E: Error>(_ type: E.Type) throws

  /// Record the return type of the distributed method.
  /// If the target does not return any specific value, this method will not be called and the return type can be assumed `Void`.
  ///
  /// Ad-hoc protocol requirement.
  mutating func recordReturnType<R: SerializationRequirement>(_ type: R.Type) throws

  /// All values and types have been recorded.
  /// Optionally "finalize" the recording, if necessary.
  mutating func doneRecording() throws
}

/// Represents an argument passed to a distributed call target.
public struct RemoteCallArgument<Value> {
  /// The "argument label" of the argument.
  /// The label is the name visible name used in external calls made to this
  /// target, e.g. for `func hello(label name: String)` it is `label`.
  ///
  /// If no label is specified (i.e. `func hi(name: String)`), the `label`,
  /// value is empty, however `effectiveLabel` is equal to the `name`.
  ///
  /// In most situations, using `effectiveLabel` is more useful to identify
  /// the user-visible name of this argument.
  let label: String?
  var effectiveLabel: String {
    return label ?? name
  }

  /// The internal name of parameter this argument is accessible as in the
  /// function body. It is not part of the functions API and may change without
  /// breaking the target identifier.
  ///
  /// If the method did not declare an explicit `label`, it is used as the
  /// `effectiveLabel`.
  let name: String

  /// The value of the argument being passed to the call.
  /// As `RemoteCallArgument` is always used in conjunction with
  /// `recordArgument` and populated by the compiler, this Value will generally
  /// conform to a distributed actor system's `SerializationRequirement`.
  let value: Value
}
```



```swift
public protocol DistributedTargetInvocationDecoder {
  associatedtype SerializationRequirement

  mutating func decodeGenericSubstitutions() throws -> [Any.Type]

  /// Attempt to decode the next argument from the underlying buffers into pre-allocated storage
  /// pointed at by 'pointer'.
  ///
  /// This method should throw if it has no more arguments available, if decoding the argument failed,
  /// or, optionally, if the argument type we're trying to decode does not match the stored type.
  ///
  /// Ad-hoc protocol requirement.
  mutating func decodeNextArgument<Argument: SerializationRequirement>() throws -> Argument

  mutating func decodeErrorType() throws -> Any.Type?

  mutating func decodeReturnType() throws -> Any.Type?
}
```

#### Sender: Invoking a distributed method

A call to a distributed method (or computed property) on a remote distributed actor reference needs to be turned into a runtime introspectable representation which will be passed to the `remoteCall` method of a specific distributed actor system implementation.

In this section, we'll see what happens for the following `greet(name:)` distributed method call:

```swift
// distributed func greet(name: String) -> String { ... }

try await greeter.greet(name: "Alice")
```

Such invocation is calling the method via a "distributed thunk" rather than directly. The "distributed thunk" is synthesized by the compiler for every `distributed func`, and can be illustrated by the following snippet:

```swift
// ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ SYNTHESIZED ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
extension Greeter {
  // synthesized; not user-accessible thunk for: greet(name: String) -> String
  nonisolated func greet_$distributedThunk(name: String) async throws -> String {
    guard _isDistributedRemoteActor(self) else {
      // the local func was not throwing, but since we're nonisolated in the thunk,
      // we must hop to the target actor here, meaning the 'await' is always necessary.
      return await self.greet(name: name)
    }

    // [1] prepare the invocation object:
    var invocation = self.actorSystem.makeInvocationEncoder()

    // [1.1] if method has generic parameters, record substitutions
    // e.g. for func generic<A, B>(a: A, b: B) we would get two substitutions,
    // for the generic parameters A and B:
    //
    // << invocation.recordGenericSubstitution(<runtime type of a>)
    // << invocation.recordGenericSubstitution(<runtime type of b>)

    // [1.2] for each argument, synthesize a specialized recordArgument call:
    try invocation.recordArgument(RemoteCallArgument(label: nil, name: "name", value: name))

    // [1.3] if the target was throwing, record Error.self,
    // otherwise do not invoke recordErrorType at all.
    //
    // << try invocation.recordErrorType(Error.self)

    // [1.4] we also record the return type; it may or may not be necessary to
    // transmit over the wire but if necessary, the system may choose to do so.
    //
    // This call is not made when the return type is Void.
    try invocation.recordReturnType(String.self)

    // [1.5] done recording arguments
    try invocation.doneRecording()

    // [2] invoke the `remoteCall` method of the actor system
    return try await self.actorSystem.remoteCall(
      on: self,
      target: RemoteCallTarget(...),
      invocation: invocation,
      throwing: Never.self, // the target func was not throwing
      returning: String.self
    )
  }
}
```

The synthesized thunk is always throwing and asynchronous. This is correct because it is only invoked in situations where we might end up calling the `actorSystem.remoteCall(...)` method, which by necessity is asynchronous and throwing.

The thunk is `nonisolated` because it is a method that can actually run on a *remote* instance, and as such is not allowed to touch any other state than other nonisolated stored properties. This is specifically designed such that the thunk and actor system are able to access the `id` of the actor (and the `actorSystem` property itself) which is necessary to perform the actual remote message send.

The `nonisolated` aspect of the method has another important role to play: if this invocation happens to be on a local distributed actor, we do not want to "hop" executors twice. If this invocation were on a local actor, only accessing `nonisolated` state, or for other reasons the hop could be optimized away, we want to keep this ability for the optimizer to do as good of a job as it would for local only actors. If the instance was remote, we don't need to suspend early at all, and we leave it to the `ActorSystem` to decide when exactly the task will suspend. For example, the system may only suspend the call after it has sent the bytes synchronously over some IPC channel, etc. The semantics of to suspend are highly dependent on the specific underlying transport, and thanks to this approach we allow system implementations to do the right thing, whatever that might be: they can suspend early, late, or even not at all if the call is known to be impossible to succeed.

Note that the compiler will pass the `self` of the distributed *known-to-be-remote* actor to the `remoteCall` method on the actor system. This allows the system to check the passed type for any potential, future, customization points that the actor may declare as static properties, and/or conformances affecting how a message shall be serialized or delivered. It is impossible for the system to access any of that actor's state, because it is remote after all. The one piece of state it will need to access though is the actor's `id` because that is signifying the *recipient* of the call.

The thunk creates the `invocation` container `[1]` into which it records all arguments. Note that all these APIs are using only concrete types, so we never pay for any existential wrapping or other indirections. Arguments are wrapped in a `RemoteCallArgument` which also provides additional information about the argument, such as the labels used in its declaration. This can be useful for transports which wish to encode values in named dictionaries, e.g. such as JSON dictionaries. It is important to recognize that storing names or labels is _not_ required or recommended for the majority of transports, however for those which need it, they are free to use this additional information.

Since the strings created are string literals, known at compile time, there is no allocation impact for this argument wrapper type and this additional label information.

> It may seem tempting to simply record arguments into a dictionary using their effective labels, however this can lead to issues due to labels being allowed to be reused. For example, `func sum(a a1: Int, a a2: Int)` is a legal, although not very readable, function declaration. A naive encoding scheme could risk overriding the "first `a`" value with the "second `a` if it strongly relied on the labels only. A distributed actor system implementation should decide how to deal with such situations and may choose to throw at runtime if such a risky signature is detected, or apply some mangling, e.g. use the parameter names to clarify which value was encoded.

The `record...` calls are expected to serialize the values, using any mechanism they want to, and thanks to the fact that the type performing the recording is being provided by the specific `ActorSystem`, it also knows that it can rely on the arguments to conform to the system's `SerializationRequirement`.

The first step in the thunk is to record any "generic substitutions" `[1.1]` if they are necessary. This makes it possible for remote calls to support generic arguments, and even generic distributed actors. The substitutions are not recorded for call where the generic context is not necessary for the invocation. For a generic method, however, the runtime will invoke the `recordGenericTypeSubstitution` with _concrete_ generic arguments that are necessary to perform the call. For example, if we declared a generic `echo` method like this:

```swift
distributed func echo<T: SerializationRequirement>(_ value: T) -> T
```

and call it like this:

```swift
try await greeter.echo("Echo!") // typechecks ok; String: SerializationRequirement
```

The Swift runtime would generate the following call:

```swift
try invocation.recordGenericTypeSubstitution(String.self)
```

This method is implemented by a distributed actor system library, and can use this information to double-check this type against an allow-list of types allowed to be transmitted over the wire, and then store and send it over to the recipient such that it knows what type to decode the argument as.

Next, the runtime will record all arguments of the invocation `[1.2]`. This is done in a series of `recordArgument` calls. If the type of actor the target is declared on also includes a generic parameter that is used by the invocation, this also is recorded.

As the `recordArgument(_:)` method is generic over the argument type (`<Argument: SerializationRequirement>`), and requires the argument to conform to `SerializationRequirement` (which in turn was enforced at compile time by [SE-0336][isolation]), the actor system implementation will have an easy time to serialize or validate this argument. For example, if the `SerializationRequirement` was codable — this is where one could invoke `SomeEncoder().encode(argument)` because `Argument` is a concrete type conforming to `Codable`!

Finally, the specific error `[1.3]` and return types `[1.4]` are also recorded. If the function is not throwing, `recordErrorType` is not called. Likewise, if the return type is `Void` the `recordReturnType` is not called.

Recording the error type is mostly future-proofing and currently will only ever be invoked with the `Error.self` or not at all. It allows informing the system if a throw from the remote side is to be expected, and technically, if Swift were to gain typed throws this method could record specific expected error types as well — although we have no plans with regards to typed throws at this point in time.

The last encoder call `doneRecording()` is made, to signal to the invocation encoder that no further record calls will be made. This is useful since with the optional nature of some of the calls, it would be difficult to know for a system implementation when the invocation is fully constructed. Operations which may want to be delayed until completion could include serialization, de-duplicating values or similar operations which benefit from seeing the whole constructed invocation state in the encoder.

Lastly, the populated encoder, along with additional type and function identifying information is passed to the `remoteCall`, or `remoteCallVoid` method on the actor system which should actually perform the message request/response interaction with the remote actor.

#### Sender: Serializing and sending invocations

The next step in making a remote call is serializing a representation of the distributed method (or computed property) invocation. This is done through a series of compiler, runtime, and distributed actor system interactions. These interactions are designed to be highly efficient and customizable. Thanks to the `DistributedTargetInvocationEncoder`, we are able to never resort to existential boxing of values, allow serializers to manage and directly write into their destination buffers (i.e. allowing for zero copies to be performed between the message serialization and the underlying networking layer), and more.

Let us consider a `ClusterSystem` that will use `Codable` and send messages over the network. Most systems will need to form some kind of "Envelope" (easy to remember as: "the thing that contains the **message** and also has knowledge of the **recipient**"). For the purpose of this proposal, we'll define a a `WireEnvelope` and use it in the next snippets to showcase how a typical actor system would work with it. This type is not pre-defined or required by this proposal, but it is something implementations will frequently do on their own:

```swift
// !! ClusterSystem or WireEnvelope are NOT part of the proposal, but serves as illustration how actor systems might !!
// !! implement the necessary pieces of the DistributedActorSystem protocol.                                         !!

final struct ClusterSystem: DistributedActorSystem {
  // ...
  typealias SerializationRequirement = Codable
  typealias InvocationEncoder = ClusterTargetInvocationEncoder
  typealias InvocationDecoder = ClusterTargetInvocationDecoder

  // Just an example, we can implement this more efficiently if we wanted to.
  private struct WireEnvelope: Codable, Sendable {
    var recipientID: ClusterSystem.ActorID // is Codable

    /// Mangled method/property identifier, e.g. in a mangled format
    var identifier: String

    // Type substitutions matter only for distributed methods which use generics:
    var genericSubstitutions: [String]

    // For illustration purposes and simplicity of code snippets we use `[Data]` here,
    // but real implementations can be much more efficient here — packing all the data into exact
    // byte buffer that will be passed to the networking layer, etc.
    var arguments: [Data] // example is using Data, because that's what Codable coders use

    // Metadata can be used by swift-distributed-tracing, or other instrumentations to carry extra information:
    var metadata: [String: [Data]] // additional metadata, such as trace-ids
  }
}
```

Note that `method` property is enough to identify the target of the call, we do not need to carry any extra type information explicitly in the call. The method identifier is sufficient to resolve the target method on the recipient, however in order to support generic distributed methods, we need to carry additional (mangled) type information for any of the generic parameters of this specific method invocation. Thankfully, these are readily provided to us by the Swift runtime, so we'll only need to store and send them over.

> **Note:** An implementation may choose to define any shape of "envelope" (or none at all) that suits its needs. It may choose to transport mangled names of involved types for validation purposes, or choose to not transfer them at all and impose other limitations on the system and its users for the sake of efficiency.
>
> While advanced implementations may apply compression and other techniques to minimize the overhead of these envelopes — this is a deep topic by itself, and we won't be going in depth on it in this proposal — rest assured though, we have focused on making different kinds of implementations possible with this approach.

Next, we will discuss how the `InvocationEncoder` can be implemented in order to create such `WireEnvelope`.

> Note on ad-hoc requirements: Some of the protocol requirements on the encoder, as well as actor system protocols, are so-called "ad-hoc" requirements. This means that they are not directly expressed in Swift source, but instead the compiler is aware of the signatures and specifically enforces that a type conforming to such protocol implements these special methods.
>
> Specifically, methods which fall into this category are functions which use the `SerializationRequirement` as generic type requirement. This is currently not expressible in plain Swift, due to limitations in the type system which are difficult to resolve immediately, but in time as this could become implementable these requirements could become normal protocol requirements.
>
> This tradeoff was discussed at length and we believe it is worth taking, because it allows us to avoid numerous un-necessary type-casts, both inside the runtime and actor system implementations. It also allows us to avoid any existential boxing  and thus lessens the allocation footprint of making remote calls which is an important aspect of the design and use cases we are targeting.

The following listing illustrates how one _could_ implement a `DistributedTargetInvocationEncoder`:

```swift
extension ClusterSystem {
  typealias InvocationEncoder = ClusterTargetInvocationEncoder

  func makeInvocationEncoder() -> Self.InvocationEncoder {
    return ClusterTargetInvocation(system: system)
  }
}

struct ClusterTargetInvocationEncoder: DistributedTargetInvocationEncoder {
  typealias SerializationRequirement = ClusterSystem.SerializationRequirement

  let system: ClusterSystem
  var envelope: Envelope

  init(system: ClusterSystem) {
    self.system = system
    self.envelope = .init() // new "empty" envelope
  }

  /// The arguments must be encoded order-preserving, and once `decodeGenericSubstitutions`
  /// is called, the substitutions must be returned in the same order in which they were recorded.
  mutating func recordGenericSubstitution<T>(_ type: T.Type) throws {
    // NOTE: we are showcasing a pretty simple implementation here...
    //       advanced systems could use mangled type names or registered type IDs.
    envelope.genericSubstitutions.append(String(reflecting: T.self))
  }

  mutating func recordArgument<Argument: SerializationRequirement>(_ argument: RemoteCallArgument<Argument>) throws {
    // in this implementation, we just encode the values one-by-one as we receive them:
    let argData = try system.encoder.encode(argument) // using whichever Encoder the system has configured
    envelope.arguments.append(argData)
  }

  mutating func recordErrorType<E: Error>(_ errorType: E.Type) throws {
    envelope.errorType = String(reflecting: errorType)
  }

  mutating func recordReturnType<R: SerializationRequirement>(_ returnType: R.Type) throws {
    envelope.returnType = String(reflecting: returnType)
  }

  /// Invoked when all the `record...` calls have been completed and the `DistributedTargetInvocation`
  /// will be passed off to the `remoteCall` to perform the remote call using this invocation representation.
  mutating func doneRecording() throws {
    // our impl does not need to do anything here
  }
}
```

The above encoder is going to be called by the Swift runtime as was explained in the previous section.

Once that is complete, the runtime will pass the constructed `InvocationEncoder` to the `remoteCall`:

```swift
 extension ClusterSystem {
  // 'remoteCall' is not a protocol requirement, however its signature is well known to the compiler,
  // and it will invoke the method. We also are guaranteed that the 'Success: Codable' requirement is correct,
  // since the type-system will enforce this conformance thanks to the type-level checks on distributed funcs.
  func remoteCall<Actor, Failure, Success>(
      on actor: Actor,
      target: RemoteCallTarget,
      invocation: Self.InvocationEncoder,
      throwing: Failure.Type,
      returning: Success.Type
  ) async throws -> Success
      where Actor: DistributedActor,
            Actor.ID == ActorID.
            Failure: Error,
            Success: Self.SerializationRequirement {
    var envelope = invocation.envelope

    // [1] the recipient is transferred over the wire as its id
    envelope.recipient = recipient.id

    // [2] the method is a mangled identifier of the 'distributed func' (or var).
    //     In this system, we just use the mangled name, but we could do much better in the future.
    envelope.target = target.identifier

    // [3] send the envelope over the wire and await the reply:
    let responseData = try await self.underlyingTransport.send(envelope, to: actor.id)

    // [4] decode the response from the response bytes
    // in our example system, we're using Codable as SerializationRequirement,
    // so we can decode the response like this (and never need to cast `as? Codable` etc.):
    return try self.someDecoder.decode(as: Success.self, from: responseData)
  }
}
```

The overall purpose of this `remoteCall` implementation is to create some form of message representation of the invocation and send it off to the remote node (or process) to receive and invoke the target method on the remote actor.

In our example implementation, the `Invocation` already serialized the arguments and stored them in the `Envelope`, so the `remoteCall` only needs to add the information about the call recipient `[1]`, and the target (method or computed property) of the call `[2]`. In our example implementation, we just store the target's mangled name `[2]`, which is simple, but it has its challenges in regard to protocol evolution.

One notable issue that mangled names have is that any change in the method signature will result in not being able to resolve the target method anymore. We are very much aware of the issues this may cause to protocol evolution, and we lay out plans in [Future Work](#future-work) to improve the lookup mechanisms in ways that will even allow adding parameters (with default values), in wire (and ABI) compatible ways.

The final step is handing over the envelope containing the encoded arguments, recipient information, etc., to the underlying transport mechanism `[3]`. The transport does not really have to concern itself with any of the specifics of the call, other than transmitting the bytes to the callee and the response data back. As we get the response data back, we have the concrete type of the expected response and can attempt to decode it `[4]`.

> Note on `remoteCallVoid`: One limitation in the current implementation approach is that a remote call signature cannot handle void returning methods, because of the `Success: SerializationRequirement` requirement on the method.
>
> This will be possible to solve using the incoming [Variadic Generics](https://forums.swift.org/t/variadic-generics/54511) language feature that is being currently worked on and pitched. With this feature, the return type could be represented as variadic generic and the `Void` return type would be modeled as "empty" tuple, whereas a value return would contain the specific type of the return, this way we would not violate the `Success: SerializationRequirement` when we needed to model `Void` calls.

#### Recipient: Receiving invocations

On the remote side, there usually will be some receive loop or similar mechanism that is implemented in the transport layer of the actor system. In practice this often means binding a port and receiving TCP (or UDP) packets, applying some form of framing and eventually decoding the incoming message envelope.

Since the communication of the sending and receiving side is going to be implemented by the same type of transport and actor system, receiving the envelopes is straightforward: we know the wire protocol, and follow it to receive enough bytes to decode the `Envelope` which we sent a few sections above.

This part does not have anything specific prescribed in the `DistributedActorSystem` protocol. It is up to every system to implement whichever transport mechanism works for it. While not a "real" snippet, this can be thought of a simple loop over incoming connections, like this:

```swift
// simplified pseudo code for illustration purposes
func receiveLoop(with node: Node) async throws {
  for try await envelopeData in connection(node).receiveEnvelope {
    await self.receive(envelopeData)
  }
}
```

In a real server implementation we'd likely use a [Swift NIO](https://github.com/apple/swift-nio) `ChannelPipeline` to perform this networking, framing and emitting of `Envelope`s, but this is beyond the scope of what we need to explain in this proposal to get the general idea of how this is going to work.

#### Recipient: Deserializing incoming invocations

Now that we have received all the bytes for one specific envelope, we need to perform a two-step deserialization on it.

First, we'll need to decode the target identifier (e.g. method name, mangled method name, or some other form of target identifier), and the actor `ID` of the recipient. These are necessary to decode always, as we need to locate both the method and actor we're trying to invoke.

Next, the deserialization of the actual message representation of our invocation will take place. However, this is done lazily. Rather than just decoding the values and storing them somewhere in our system implementation, these will be requested by the Swift runtime when it is about to perform the method call.

Before we dive deeper into this, let us visualize how this two-step process is intended to work, by looking at what might be a typical envelope format on the wire:

```c
+------------------------------- ENVELOPE --------------------------------------+
| +---------- HEADER --------++-------------------- MESSAGE ------------------+ |
| | target | recipient | ... || [ ... lazy decoded section: types, args ... ] | |
| +--------------------------++-----------------------------------------------+ |
+-------------------------------------------------------------------------------+
```

We see that as we decode our wire envelope, we are able to get the header section, and all values contained within it eagerly and leave the remaining slice of the buffer untouched. It will be consumed during performing of the invocation soon enough. The nice thing about this design is that we're still able to hold onto the actual buffer handed us from the networking library, and we never had to copy the buffer to our own local copies.

Next, we need to prepare for the decoding of the message section. This is done by implementing the remaining protocol requirements on the `ClusterTargetInvocation` type we defined earlier, as well as implementing a decoding iterator of type `DistributedTargetInvocationArgumentDecoder`, as shown below:

```swift
struct ClusterTargetInvocationDecoder: DistributedTargetInvocationDecoder {
  typealias SerializationRequirement = Codable

  let system: ClusterSystem
  var bytes: ByteBuffer

  mutating func decodeGenericSubstitutions() throws -> [Any.Type] {
    let subCount = try self.bytes.readInt()

    var subTypes: [Any.Type] = []
    for _ in 0..<subCount {
      let length = try self.bytes.readInt() // read the length of the next substitution
      let typeName = try self.bytes.readString(length: length)
      try subTypes.append(self.system.summonType(byName: typeName))
    }

    return subTypes
  }

  /// Attempt to decode the next argument from the underlying buffers into pre-allocated storage
  /// pointed at by 'pointer'.
  ///
  /// This method should throw if it has no more arguments available, if decoding the argument failed,
  /// or, optionally, if the argument type we're trying to decode does not match the stored type.
  ///
  /// The result of the decoding operation must be stored into the provided 'pointer' rather than
  /// returning a value. This pattern allows the runtime to use a heavily optimized, pre-allocated
  /// buffer for all the arguments and their expected types. The 'pointer' passed here is a pointer
  /// to a "slot" in that pre-allocated buffer. That buffer will then be passed to a thunk that
  /// performs the actual distributed (local) instance method invocation.
  mutating func decodeNextArgument<Argument: SerializationRequirement>() throws -> Argument {
    try nextDataLength = try bytes.readInt()
    let nextData = try bytes.readData(bytes: nextDataLength)
    // since we are guaranteed the values are Codable, so we can just invoke it:
    return try system.decoder.decode(as: Argument.self, from: bytes)
  }

  mutating func decodeErrorType() throws -> Any.Type? {
    let length = try self.bytes.readInt() // read the length of the type
    guard length > 0 {
      return nil // we don't always transmit it, 0 length means "none"
    }
    let typeName = try self.bytes.readString(length: length)
    return try self.system.summonType(byName: typeName)
  }

  mutating func decodeReturnType() throws -> Any.Type? {
    let length = try self.bytes.readInt() // read the length of the type
    guard length > 0 {
      return nil // we don't always transmit it, 0 length means "none"
    }
    let typeName = try self.bytes.readString(length: length)
    return try self.system.summonType(byName: typeName)
  }
}
```

The general idea here is that the `InvocationDecoder` is *lazy* in its decoding and just stores the remaining bytes of the envelope. All we need to do for now is to implement the Invocation in such way that it expects the decoding methods be invoked in the following order (which is the same as the order on the sending side):

- 0...1 invocation of `decodeGenericArguments`,
- 0...n invocation(s) of `decoder.decodeNextArgument<Argument>`,
- 0...1 invocation of `decodeReturnType`,
- 0...1 invocation of `decodeErrorType`.

Decoding arguments is the most interesting here. This is another case where the compiler and Swift runtime enable us to implement things more easily. Since the `Argument` generic type of the `decodeNextArgument` is ensured to conform to the `SerializationRequirement`, actor system implementations can rely on this fact and have a simpler time implementing the decoding steps. For example, with `Codable` the decoding steps becomes a rather simple task of invoking the usual `Decoder` APIs.

This decoder must be prepared by the actor system and eventually passed to the `executeDistributedTarget` method which we'll discuss next. That, Swift runtime provided, function is the one which will be calling the `decode...` methods and will is able to ensure all the type requirements are actually met and form the correct generic method invocations.

> **Note:** This proposal does not include an implementation for the mentioned `summonType(byName:)` function, it is just one way systems may choose to implement these functions. Possible implementations include: registering all "trusted" types, using mangled names, or something else entirely. This proposal has no opinion about how these types are recovered from the transmitted values.

#### Recipient: Resolving the recipient actor instance

Now that we have prepared our `InvocationDecoder` we are ready to make the next step, and resolve the recipient actor which the invocation shall be made on.

We already discussed how resolving actors works in [Resolving distributed actors](#resolving-distributed-actors), however in this section we can tie it into the real process of invoking the target function as well.

In the example we're following so far, the recipient resolution is simple because we have the recipient ID available in the `Envelope.recipientID`, so we only need to resolve that using the system that is receiving the message:

```swift
guard let actor: any DistributedActor = try self.knownActors[envelope.recipientID] else {
  throw ClusterSystemError.unknownRecipient(envelope.recipientID)
}
```

This logic is the same as the internal implementation of the `resolve(id:as:)` method only that we don't have a need to validate the specific type of the actor — this will be handled by the Swift runtime in `executeDistributedTarget`'s implementation the target of the call which we'll explain in the next section.

#### Recipient: The `executeDistributedTarget` method

Invoking a distributed method is a tricky task, and involves a lot of type demangling, opening existential types, forming specific generic invocations and tightly managing all of that in order to avoid un-necessary heap allocations to pass the decoded arguments to the target function, etc. After iterating over multiple designs, we decided to expose a single `DistributedActorSystem.executeDistributedTarget` entry point which efficiently performs all the above operations.

Thanks to abstracting the decoding logic into the `DistributedTargetInvocationDecoder` type, all deserialization can be made directly from the buffers that were received from the underlying network transport. The `executeDistributedTarget` method has no opinion about what serialization mechanism is used either, and any mechanism — be it `Codable` or other external serialization systems — can be used, allowing distributed actor systems developers to implement whichever coding strategy they choose, potentially directly from the buffers obtained from the transport layer.

The `executeDistributedTarget` method is defined as:

```swift
extension DistributedActorSystem {
  /// Prepare and execute a call to the distributed function identified by the passed arguments,
  /// on the passed `actor`, and collect its results using the `ResultHandler`.
  ///
  /// This method encapsulates multiple steps that are invoked in executing a distributed function,
  /// into one very efficient implementation. The steps involved are:
  ///
  /// - looking up the distributed function based on its name;
  /// - decoding, in an efficient manner, all arguments from the `Args` container into a well-typed representation;
  /// - using that representation to perform the call on the target method.
  ///
  /// The reason for this API using a `ResultHandler` rather than returning values directly,
  /// is that thanks to this approach it can avoid any existential boxing, and can serve the most
  /// latency sensitive-use-cases.
  func executeDistributedTarget<Actor, ResultHandler>(
      on actor: Actor,
      mangledName: String,
      invocation: inout Self.InvocationDecoder,
      handler: ResultHandler
  ) async throws where Actor: DistributedActor,
                       Actor.ID == ActorID,
                       ResultHandler: DistributedTargetInvocationResultHandler {
    // implemented by the _Distributed library
  }
}
```

This method encapsulates all the difficult and hard to implement pieces of the target invocation, and it accepts the base actor the call should be performed on, along with a `DistributedTargetInvocationResultHandler`.

Rather than having the `executeDistributedTarget` method return an `Any` result, we use the result handler in order to efficiently, and type-safely provide the result value to the actor system library implementation. This technique is the same as we did with the `recordArgument` method before, and it allows us to provide the _specific_ type including its `SerializationRequirement` conformance making handling results much simpler, and without having to resort to any casts which can be unsafe if used wrongly, or have impact on runtime performance.

The `DistributedTargetInvocationResultHandler` is defined as follows:

```swift
protocol DistributedTargetInvocationResultHandler {
  associatedtype SerializationRequirement

  func onReturn<Success>(value: Success) async throws
    where Success: SerializationRequirement
  func onThrow<Error>(error: Error) async throws
    where Failure: Error
}
```

In a way, the `onReturn`/`onThrow` methods can be thought of as the counterparts of the `recordArgument` calls on the sender side. We need to encode the result and send it _back_ to the sender after all. This is why providing the result value along with the appropriate SerializationRequirement conforming type is so important — it makes sending back the reply to a call, as simple as encoding the argument of the call.

Errors must be handled by informing the sender about the failed call. This is in order to avoid senders waiting and waiting for a reply, and eventually triggering a timeout; rather, they should be informed as soon as possible that a call has failed. Treat an error the same way as you would a valid return in terms of sending the reply back. However, it is not required to actually send back the actual error, as it may not be safe, or a good idea from a security and information exposure perspective, to send back entire errors. Instead, systems are encouraged to send back a reasonable amount of information about a failure, and e.g. optionally, only if the thrown error type is `Codable` and allow-listed to be sent over the wire, transport it directly.

#### Recipient: Executing the distributed target

Now that we have completed all the above steps, all building up to actually invoking the target of a remote call: it is finally time to do so, by calling the `executeDistributedTarget` method:

```swift
// inside recipient actor system
let envelope: IncomingEnvelope = // receive & decode ...
let recipient: DistributedActor = // resolve ...

let invocationDecoder = InvocationDecoder(system: self, bytes: envelope.bytes)

try await executeDistributedTarget(
  on: recipient, // target instance for the call
  mangledName: envelope.targetName, // target func/var for the call
  invocation: invocationDecoder // will be used to perform decoding arguments,
  handler: ClusterTargetInvocationResultHandler(system, envelope) // handles replying to the caller (omitted in proposal)
)
```

This call triggers all the decoding that we discussed earlier, and if any of the decoding, or distributed func/var resolution fails this call will throw. Otherwise, once all decoding has successfully been completed, the arguments are passed through the buffer to a distributed method accessor that actually performs the local method invocation. Once the method returns, its results are moved into the handler where the actor system takes over in order to send a reply to the remote caller — completing the remote call!

Internally, the execute distributed thunk heavily relies on the lookup and code generated by the compiler for every `distributed func` which we refer to as **distributed method accessor thunk**. This thunk is able to decode incoming arguments using the `InvocationDecoder` and directly apply the target function, all while properly handling generics and other important aspects of function invocations. It is the distributed method accessor thunk that must be located using the "target identifier" when we handle an incoming the remote call, the thunk then calls the actual target function.

For sake of completeness, the listing below shows the distributed method accessor thunk that is synthesized by the compiler. The thunk contains compiler synthesized logic specific to every distributed function to locate the target function, obtain the expected parameter types and use the passed in decoder to decode the arguments to finally pass them to the final function application.

The thunk can be thought of in terms of this abstract example. However, it cannot be implemented like this because of various interactions with the generic system as well as how emissions (function calls) actually work. Distributed method accessor thunks are implemented directly in IR as it would not be possible to synthesize the necessary emissions in any higher level part of the compiler (!). Thankfully, the logic contained in those accessors is fairly straightforward and can be imagined as:

```swift
distributed actor DA {
  func myCompute(_ i: Int, _ s: String, _ d: Double) async throws -> String  {
    "i:\(i), s:\(s), d:\(d)"
  }
}

extension DA {
  // Distributed accessor thunk" for 'myCompute(_:_:_:) -> String'
  //
  // PSEUDO CODE FOR ILLUSTRATION PURPOSES; NOT IMPLEMENTABLE IN PLAIN SWIFT;
  // Implemented in directly in IR for expressibility reasons, and not user-accessible.
  nonisolated func $distributedFuncAccessor_myCompute(
    decoder: UnsafeMutableRawPointer,
    argumentTypes: UnsafeRawPointer,
    resultBuffer: UnsafeRawPointer,
    genericSubstitutions: UnsafeRawPointer,
    witnessTables: UnsafeRawPointer,
    numWitnessTables: Int,
    actorSelf: UnsafeRawPointer) async {

    // - get generic signature of 'myCompute'
    // - create storage 'args' for all the parameters; it will be used directly
    // - for every argument, get the argumentType
    //   - invoke 'decoder.decodeArgument<Argument>()'
    //   - store in 'args'
    // - deal with the generic substitutions, witness tables and prepare the call
    // invoke 'myCompute' with 'args', and the prepared 'result' and 'error' buffers
  }
}
```

As we can see, this thunk is "just" taking care of converting the heterogeneous parameters into the well typed counterparts, and finally performing a plain-old method invocation using those parameters. The actual code emission and handling of generics for all this to work is rather complex and can only be implemented in the IR layer of the compiler. The good part about it is that the compiler is able to prepare and emit good errors in case the types or witness tables seem to be mismatched with the target or other issues are found. Allocations are also kept to a minimum, as no intermediate allocations need to be made for the arguments and they are stored and directly emitted into the call emission of the target.

The thunk again uses the indirect return, so we can avoid any kind of implicit existential boxing even on those layers. Errors are always returned indirectly, so we do not need to do it explicitly.

#### Recipient: Collecting result/error from invocations

Now that the distributed method has been invoked, it eventually returns or throws an error.

Collecting the return (or error) value is also implemented using the `DistributedMethodInvocationHandler` we passed to the `executeDistributedTarget(...)` method. This is done for the same reason as parameters: we need a concrete type in order to efficiently pass the values to the actor system, so it can encode them without going through existential wrappers. As we cannot implement the `invoke()` method to be codable over the expected types — we don't know them until we've looked up the actual method we were about to invoke (and apply generic substitutions to them).

The implementation could look as follows:

```swift
extension ExampleDistributedMethodInvocationHandler {
  func onReturn<Success: SerializationRequirement>(result: Success) throws {
    do {
      let replyData = system.encoder.encode(result)
      self.reply(replyData)
    } catch {
      self.replyError("Failed to encode reply: \(type(of: error))")
    }
  }

  func onError<Failure: Error>(error: Failure) {
    guard Failure is Encodable else {
      // best effort error reporting just sends back the type string
      // we don't want to send back string representation since it could leak sensitive information
      self.replyError("\(Failure.self)")
    }

    // ... if possible, `as?` cast to Encodable and return an actual error,
    //     but only if it is allow-listed, as we don't want to send arbitrary errors back.
  }
}
```

We omit the implementations of `replyError` and `reply` because they are more of the same patterns that we have already discussed here, and this is a proposal focused on illustrating the language feature, not a complete system implementation after all.

The general pattern here is the same as with decoding parameters, however in the opposite direction.

Once the `onError` or `onReturn` methods complete, the `executeDistributedTarget` method returns, and its caller knows the distributed request/response has completed – at least, as far as this peer is concerned. We omit the implementation of the `reply` and `replyError` methods that the actor system would implement here, because they are pretty much the same process as sending the request, except that the message must be sent as a response to a specific request, rather than target a specific actor and method. How this is achieved can differ wildly between transport implementations: some have built-in request/reply mechanisms, while others are uni-directional and rely on tagging replies with identifiers such as "this is a reply for request 123456".

## Amendments

### Initializers no longer need to accept a single DistributedActorSystem

During this review, feedback was received and addressed with regards to the previous implementation limitation that user defined designated actor initializers must have always accepted a single `DistributedActorSystem` conforming parameter. This was defined in [SE-0336: Distributed Actor Isolation](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0336-distributed-actor-isolation.md).

The limitation used to be that one had to accept such parameter or the system would fail to synthesize the initializer an offer a compile-time error:

```swift
// PREVIOUSLY, as defined by SE-0336
init(x: String) {}
// ❌ error: designated distributed actor initializer 'init(x:)' is missing required 'DistributedActorSystem' parameter

init(system: AnyDistributedActorSystem) {} 
// ✅ used to be ok; though the "unused" parameter looks confusing and is actually necessary
```

This system argument, picked by type, would have been picked up by the compiler and used in synthesis of necessary calls during a distributed actor's initialization. This led to a weird "dangling" parameter that is not used by visible code, but is crucial for the correctness of the program.

This implementation restriction is now lifted, and is rephrased as requiring an assignment to the `self.actorSystem` property in non-delegating initializers. This is in line with how normal Swift works, and can be explained to users more intuitively, because every `distributed actor` has a synthesized `let actorSystem: ActorSystem` property, it clearly must be initialized to something as implied by normal initializer rules.

The checks therefore are now plain property assignment checks, and the compiler is able to pick up the assigned property and synthesize the necessary `assignID` and `actorReady` calls based on the property, e.g.:

```swift
distributed actor Example {  
  init(x: String) {
    // ❌ error: property 'actorSystem' was not initialized
    // more educational notes to be produced here (e.g. what type it expects etc)
  }
  
  init (x: String, cluster: ClusterSystem) {
    self.actorSystem = cluster // ✅ ok
  }
}
```

This means that it is also possible to assign a "global" or static actor system, however it is **strongly discouraged** to do so because it hurts reusability and testability, e.g. by swapping a test instance of an actor system during unit tests.

```swift
// Possible, but considered an anti-pattern as it hurts testability of such actor
// DON'T DO THIS.
distributed actor NotGoodIdea {
  init() {
    self.actorSystem = SomeSystem.global // anti-pattern, always accept an actor system via initializer.
  }
}
```

## Future work

### Variadic generics removing the need for `remoteCallVoid`

Once [variadic generics](https://forums.swift.org/t/variadic-generics/54511/2) are fully implemented, we will be able to remove the limitation that we cannot express the `remoteCall<..., Success: SerializationRequirement>(..., returning returnType: Success.Type)` function for the `Void` type, since it cannot always conform to `SerializationRequirement`.

With variadic generics, it would be natural to conform an "empty tuple" to the `SerializationRequirement` and we'd this way be able to implement only a single method (`remoteCall`) rather than having to provide an additional special case implementation for `Void` return types.

### Identifying, evolving and versioning remote calls

At present remote calls use an opaque string identifier (wrapped as `RemoteCallTarget`) to refer to, and invoke remote call targets. This identifier is populated by default using the mangled name of the distributed thunk of the target function. 

#### Default distributed call target identification scheme

Distributed actor system implementations shall treat the identifier as opaque value that they shall ship from the sender to the recipient node, without worrying too much about their contents. The identification scheme though has a large and important impact on the versioning story of distributed calls, so in this section we'd like to discuss and outline the general plans we foresee here as we will introduce proper and more versioning friendly schemes in the future.

Swift's default mangling scheme is used for distributed methods is problematic for API evolution, however, it is a great "default" to pick until we devise a different scheme in the future. The mangling scheme is a good default because:

- it allows callers to use all of the richness of Swift's calling semantics, including:
  - overloads by type (e.g. we can call `receive(_:Int)` as well as `receive(_:String)`) and invoke the correct target based on the type.
- it allows us to evolve APIs manually by adding overloads, deprecate methods etc, and we will continue to work towards a feature rich versioning story while adopting limited use-cases where these limitations are not a problem.

This is also avoids a well-known problem from objective-c, where selectors must not accidentally be the same, otherwise bad-things-happen™.

One potential alternative would be to use the full names of methods, i.e. `hello(name:surname:)`, similar to objective-c selectors to identify methods. However, this means the loss of any and all type-safety and _intent_ of target methods being used according to their appropriate types. It also means identifiers must be unique and overloads must be banned at compile time, or we risk reusing identifiers and not knowing which function to invoke. This again could be solved by separately shipping type identifiers, but this would mean reinventing what Swift's mangling scheme already does, but in a worse way. Instead, we propose to start with the simple mangling scheme, and in a subsequent proposal, address the versioning story more holistically, in a way that addresses ABI concerns of normal libraries, as well as distributed calls.

We are aware that the mangling scheme makes the following, what should be possible to evolve without breaking APIs situations not work, that a different scheme could handle well:

- adding parameters even with default parameters,
- changing a type of argument from struct to class (or enum etc), is also breaking though it need not be.
  - We could address this by omitting the kind information from the mangled names.

The robust versioning and evolution scheme we have in mind for the future must be able to handle these cases, and we will be able to roll out a new identification scheme that handles those in the future, without breaking API or breaking existing remote calls. The metadata to lookup distributed method accessors would still be available using the "most precise" mangled format, even as we introduce a lossy, more versioning friendly format. APIs would remain unchanged, because from the perspective of `DistributedActorSystem` all it does is ship around an opaque String `identifier` of a remote call target.

There are numerous other versioning and rollout topics to cover, such as "rolling deployments", "green/blue deployments" and other techniques that both the versioning scheme _and_ the specific actor system implementation must be able to handle eventually. However, based on experience implementing other actor systems in the past, we are confident that those are typical to add in later phases of such an endeavor, not as the first thing in the first iteration of the project.

#### Compression techniques to avoid repeatedly sending large identifiers

One of the worries brought up with regards to mangled names and string identifiers in general during the review has been that they can impose a large overhead when the actual messages are small. Specifically, as the `RemoteCallTarget.identifier` often can be quite large, and for distributed methods which e.g. accept only a few integers, or even no values at all, the identifiers often can dominate the entire message payload.

This is not a novel problem – we have seen and solved such issues in the past in other actor runtimes (i.e. Akka). One potential solution is to establish a compression mechanism between peers where message exchanges establish shared knowledge about "long identifiers" and mapping them to unique numbers. The sender in such a system at first pessimistically sends the "long" String-based identifier, and in return may get additional metadata in the response "the next time you want to call this target, send the ID 12345". The sender system then stores in a cache that when invoking the "X...Z" target it does not need to send the long string identifier, but instead can send the number 12345. This solves the long string identifiers problem, as they need not be sent repeatedly over the wire. (There are a lot of details to how such schemes can be implemented that we do not need to dive into here, however we are confident those work well, because we have seen them solve this exact issue before).

Given the need, and assuming other shared knowledge, we could even implement other identification schemes, which can also avoid that "initial" long string identifier sending. We will be exploring those as specific use-cases and requirements from adopters as they arise. We are confident in our ability to fit such techniques into this design because of the flexibility that APIs based around `RemoteCallTarget` gives us, without requiring us to actually send the entire target object over the wire.

Such optimization techniques, are entirely implementable in concrete distributed actor system implementations. However, if necessary, we would be able to even extend the capabilities of `executeDistributedTarget` to accommodate other concrete needs and designs.

#### Overlap with general ABI and versioning needs in normal Swift code

The general concept binary/wire-compatible evolution of APIs by _adding_ new parameters to methods is not unique to distribution. In fact, this is a feature that would benefit ABI-stable libraries like the ones shipping with the SDK. It is often the case that new APIs are introduced that accept _more_ arguments, perhaps to enable or extend functionality of an existing feature. Developers today have to add new functions with "one more argument", like this:

```swift
public func f() {
  f(x: 0)
}

@available(macOS 9999)
public func f(x: Int) {
  // new implementation
}
```

Where the new function is going to ship with a new OS, yet actually contains an implementation that is compatible with the old definition of `f()`. In such situations, developers are forced to manually write forwarding methods. While this seems simple on small APIs, it can easily become rather complex and easy to introduce subtle bugs in the forwarding logic.

Instead, developers would want to be able to introduce new parameters, with a default value that would be used when the function is invoked on older platforms, like this:

```swift
public func f(@available(macOS 9999) x: Int = 0) {
  // new implementation
}

// compiler synthesizes:
//
//   public func f() { self.f(x: 0) }
//
//   @available(macOS 9999)
//   public func f( x: Int = 0) {
```

Where the compiler would synthesize versions of the methods for the various availabilities, and delegate, in an ABI-compatible way to the new implementation. This is very similar to what would be necessary to help wire-compatible evolution of distributed methods. So the same mechanisms could be used for distributed calls, if instead of OS versions, we could also allow application/node versions in the availability perhaps? This is not completely designed, but definitely a direction we are interested in exploring.

#### Discussion: User provided target identities

We also are considering, though are not yet convinced that these would be the optimal way to address some of the evolution concerns, the possibility to use "stable names". This mechanism would allow users to give full control over target identity to developers, where the `RemoteCallTarget.identifier` offered to the `remoteCall` implemented by a distributed actor system library, could be controlled by end users of the library, i.e. those who define distributed methods.

On one hand, this gives great power to end-users, as they may use specific and minimal identifiers, even using `"A"` and other short names to minimize the impact of the identifiers to the payload size. On the other hand though, it opens up developers to a lot of risks, including conflicts in identifier use – a problem all to well known to objective-c developers where selectors could end up in similar problematic situations.

We want to take time to design a proper system rather than just open up full control over the remote call target `identifier` to developers. We will aim to provide a flexible, and powerful mechanism, that does not risk giving developers easy ways to get shoot themselves in the foot. The design proposed here, is flexible enough to evolve.

### Resolving `DistributedActor` protocols

We want to be able to publish only protocols that contain distributed methods, and allow clients to resolve remote actors based on protocols alone, without having any knowledge about the specific `distributed actor` type implementing the protocol. This allows binary, closed-source frameworks to offer distributed actors as way of communication. Of course, for this to be viable we also need to solve the above ABI and wire-compatible evolution of distributed methods, assuming we solve those though, publishing distributed actor protocols is very useful and interesting for client/server scenarios, where the peers of a communication are not exact mirrors of the same process, but exhibit some asymmetry.

Currently, we resolve distributed actors using the static method defined on the `DistributedActor` protocol, sadly this method is not possible to invoke on just a protocol:

```swift
protocol Greeter: DistributedActor {
  func greet() -> String
}

let greeter: any Greeter = try Greeter.resolve(id: ..., using: websocketActorSystem)
// ❌ error: static member 'resolve' cannot be used on protocol metatype 'Greeter.Protocol'
```

A "client" peer does not have to know what distributed actor exactly implements this protocol, just that we're able to send a "greet" message to it, we should be able to obtain an existential `any Greeter` and be able to invoke `greet()` on it.

In order to facilitate this capability, we need to:

- implement ad-hoc synthesis of a type that effectively works like a "stub" that other RPC systems generally source-generate, yet thanks to our actor model we're able to synthesize it in the compiler on demand;
- find a way to invoke `resolve` on such protocol, for example we could offer a global function `resolveDistributedActorProtocol(Greeter.self, using: websocketActorSystem)`

The `resolveDistributedActorProtocol` method has to be able to check the serialization requirement at compile-time where we invoke the resolve, because the distributed actor protocols don't have to declare a serialization requirement — they can, but they don't have to (and this is by design).

It should be possible to resolve the following examples:

```swift
protocol Greeter: DistributedActor {
  distribute func greet() -> String
}

final class WebsocketActorSystem: DistributedActorSystem {
  typealias ActorID: WebsocketID // : Codable
  typealias SerializationRequirement: Codable
}

... = try resolveDistributedActorProtocol(id: ..., as: Greeter.self, using: websocketActorSystem)
```

The resolve call would use the types defined for the `ActorID` and `SerializationRequirement` to see if this `Greeter` protocol is even implementable using these, i.e. if its distributed method parameters/return types do indeed conform to `Codable`, and if there isn't a conflict with regards to the actor ID.

This means that we should reject at compile-time any attempts to resolve a protocol that clearly cannot be implemented over the actor system in question, for example:

```swift
protocol Greeter: DistributedActor {
  distributed func greet() -> NotCodableResponse
}

final class WebsocketActorSystem: DistributedActorSystem {
  typealias ActorID: WebsocketID // : Codable
  typealias SerializationRequirement: Codable
}

... = try resolveDistributedActorProtocol(id: ..., as: Greeter.self, using: websocketActorSystem)
// ❌ error: 'Greeter' cannot be resolved using 'WebsocketActorSystem'
// ❌ error: result type 'NotCodableResponse' of distributed instance method does not conform to 'Codable'
```

These are the same checks that are performed on `distributed actor` declarations, but they are performed on the type. We can think of these checks running whenever distributed methods are "combined" with a specific actor system: this is the case in `distributed actor` declarations, as well as this protocol resolution time, because we're effectively creating a not-user-visible actor declaration that combines the given `DistributedActorSystem` with the synthesized "stub" distributed actor, so we need to run the checks here. Thankfully, we can run them at compile time, disallowing any ill-formed and impossible to implement combinations.

### Passing parameters to `assignID`

Sometimes, transports may need to get a little of configuration for a specific actor being initialized.

Since `DistributedActorSystem.assignID` accepts the actor *type* it can easily access any configuration that is static for some specific actor type, e.g. like this:

```swift
protocol ConfiguredDistributedActor: DistributedActor {
  static var globalServiceName: String { get }
}

distributed actor Cook: DistributedActorConfiguration {
  static var globalServiceName: String { "com.apple.example.CookingService" }
}
```

This way the `assignID` can detect the static property and e.g. ensure this actor is possible to look up by this static name:

```swift
extension SpecificDistributedActorSystem {
  func assignID<Actor>(_ type: Actor.Type) -> Actor.ID where Actor: DistributedActor {
    let id = <<make up some id>>
    if let C = type as ConfiguredDistributedActor.Type {
      // for example, we could make sure the actor is discoverable using the service name:
      let globalServiceName = C.globalServiceName
      self.ensureAccessibleAs(id: id, as: globalServiceName)
    }

    return id
  }
}
```

Or similar configuration patterns. However, it is hard to implement a per instance configuration to be passed to the system.

One way we could solve this is by introducing an `assignID` overload that accepts the `ActorConfiguration` that may be
passed to the actor initializer, and would be passed along to the actor system like this:

```swift
extension SpecificDistributedActorSystem {
  // associatedtype ActorConfiguration
  func assignID<Actor>(_ type: Actor.Type, _ properties: Self.ActorConfiguration) -> Actor.ID where Actor: DistributedActor {
    if let name = properties.name {
      return makeID(withName: name)
    }

    return makeRandomID()
  }
}
```

The creation of the actor would then be able to be passed at-most one `ActorConfiguration` instance, and that would be then passed down to this method:

```swift
distributed actor Worker {...}

Worker(actorSystem: system, actorProperties: .name("worker-1234"))
```

Which can be *very* helpful since now IDs can have user provided information that are meaningful in the user's domain.

## Alternatives considered

This section summarizes various points in the design space for this proposal that have been considered, but ultimately rejected from this proposal.

### Define `remoteCall` as protocol requirement, and accept `[Any]` arguments

The proposal includes the fairly special `remoteCall` method that is expected to be present on a distributed actor system, however is not part of the protocol requirements because it cannot be nicely expressed in today's Swift, and it suffers from the lack of variadic generics (which are being worked on, see: [Pitching The Start of Variadic Generics](https://forums.swift.org/t/pitching-the-start-of-variadic-generics/51467)), however until they are complete, expressing `remoteCall` in the type-system is fairly painful, and we resort to providing multiple overloads of the method:

```swift
  func remoteCall<Actor, P1, Failure, Success>(
    on recipient: Actor,
    method: DistributedMethodName,
    _ arg1: P1,
    throwing errorType: Failure.Type,
    returning returnType: Success.Type
  ) async throws -> Success where Actor: DistributedActor, Actor.ID = ActorID { ... }

  func remoteCall<Actor, P1, P2, Failure, Success>(
    on recipient: Actor,
    method: DistributedMethodName,
    _ arg1: P1, _ arg2: P2,
    throwing errorType: Failure.Type,
    returning returnType: Success.Type
  ) async throws -> Success where Actor: DistributedActor, Actor.ID = ActorID { ... }

  // ...
```

This is annoying for the few distributed actor system developers. However, it allows us to completely avoid any existential boxing that shuttling values through `Any` would imply. We are deeply interested in offering this system to systems that are very concerned about allocations, and runtime overheads, and believe this is the right tradeoff to make, while we await the arrival of variadic generics which will solve this system implementation annoyance.

We are also able to avoid any heap allocations during the `remoteCall` thanks to this approach, as we do not have to construct type erased `arguments: [Any]` which would have been the alternative:

```swift
  func remoteCall<Actor, Failure, Success>(
    on recipient: Actor,
    method: DistributedMethodIdentifier,
    _ args: [Any], // BAD
    throwing errorType: Failure.Type,
    returning returnType: Success.Type
  ) async throws -> Success where Actor: DistributedActor, Actor.ID = ActorID { ... }
```

Not only that, but passing arguments as `[Any]` would force developers into using internal machinery to open the existentials (the not officially supported `_openExistential` feature), in order to obtain their specific types, and e.g. use `Codable` with them.

### Constraining arguments, and return type with of `remoteCall` with `SerializationRequirement`

Looking at the signature, one might be tempted to also include a `where` clause to statically enforce that all parameters and return type, conform to the `Self.SerializationRequirement`, like so:

```swift
  func remoteCall<Actor, P1, Failure, Success>(
    on recipient: Actor,
    method: DistributedMethodName,
    _ arg1: P1,
    throwing errorType: Failure.Type,
    returning returnType: Success.Type
  ) async throws -> Success where Actor: DistributedActor,
                    Actor.ID = ActorID,
                    P1: SerializationRequirement { ... }
// ❌ error: type 'P1' constrained to non-protocol, non-class type 'Self.R'
```

However, this is not expressible today in Swift, because we cannot prove the `associatedtype SerializationRequirement` can be used as constraint.

Fixing this would require introducing new very advanced type system features, and after consultation with the core team we decided to accept this as a current implementation limitation.

In practice this is not a problem, because the parameters are guaranteed to succeed being cast to `SerializationRequirement` at runtime thanks to the compile-time guarantee about parameters of distributed methods.

### Hardcoding the distributed runtime to make use of `Codable`

`Codable` is a great, useful, and relatively flexible protocol allowing for serialization of Swift native types. However, it may not always be the best serialization system available. For example, we currently do not have a great binary serialization format that works with `Codable`, or perhaps some developers just really want to use a 3rd party serialization format such as protocol buffers, SBE or something entirely custom.

The additional complexity of the configurable `SerializationRequirement` is pulling its weight, and we are not interested in closing down the system to just use Codable.

## Acknowledgments & Prior art

We would like to acknowledge the prior art in the space of distributed actor systems which have inspired our design and thinking over the years. Most notably we would like to thank the Akka and Orleans projects, each showing independent innovation in their respective ecosystems and implementation approaches. As these are library-only solutions, they have to rely on wrapper types to perform the hiding of information, and/or source generation; we achieve the same goal by expanding the actor-isolation checking mechanisms  already present in Swift.

We would also like to acknowledge the Erlang BEAM runtime and Elixir language for a more modern take built upon the on the same foundations, which have greatly inspired our design, however take a very different approach to actor isolation (i.e. complete isolation, including separate heaps for actors).

## Source compatibility

This change is purely additive to the source language.

The language impact has been mostly described in the Distributed Actor Isolation proposal

## Effect on ABI stability

TODO

## Effect on API resilience

None.

## Changelog

- 2.0 Adjustments after first round of review
  - Expanded discussion on current semantics, and **future directions for versioning** and wire compatibility of remote calls.
  - Amendment to **non-delegating distributed actor initializer semantics**
    - Non-delegating Initializers no longer require a single `DistributedActorSystem` conforming argument to be passed, and automatically store and initialize the `self.id` with it.
    - Instead, users must initialize the `self.actorSystem` property themselves. 
      - This should generally be done using the same pattern, by passing _in_ an actor system from the outside which helps in testing and stubbing out systems, however if one wanted to.
      - However the actorSystem property is initialized, from an initializer parameter or some global value, task-local etc, the general initialization logic remains the same, and the compiler will inject the assignment of the `self.id` property. 
      - Useful error messages explaining this decision are reported if users attempt to assign to the `id` property directly.
  
    - Thanks to YR Chen for recognizing this limitation and helping arrive at a better design here.
  
  - **Removal of explicit use of mangled names** in the APIs, and instead all APIs use an opaque `RemoteCallTarget` that carries an opaque String `identifier`
    - The `RemoteCallTarget` is populated by the compiler using mangled names by default, but other schemes can be used in the future
    - The `RemoteCallTarget` now pretty prints as "`hello(name:surname:)`" if able to decode the target from the identifier, otherwise it prints the identifier directly. This can be used to include pretty names of call targets in tracing systems etc.
    - This design future-proofs the APIs towards potential new encoding schemed we might come up in the future as we tackle a proper and feature complete versioning story for distributed calls.
  
  - `recordArgument` is passed a **`RemoteCallArgument<Value>`** which carries additional information about the argument in question
    - This parameter can be either ignored, or stored along the serialized format which may be useful for non swift targets of invocations. E.g. it is possible to store an invocation as JSON object where the argument names are used as labels or similar patterns.
    - Thanks to Slava Pestov for suggesting this improvement.
  
- 1.3 Larger revision to match the latest runtime developments
  - recording arguments does not need to write into provided pointers; thanks to the calls being made in IRGen, we're able to handle things properly even without the heterogenous buffer approach. Thank you, Pavel Yaskevich
  - simplify rules of readying actors across synchronous and asynchronous initializers, we can always ready "just before `self` is escaped", in either situation; This is thanks to the latest developments in actor initializer semantics. Thank you, Kavon Farvardin
  - express recording arguments and remote calls as "ad-hoc" requirements which are invoked directly by the compiler
  - various small cleanups to reflect the latest implementation state
- 1.2 Drop implicitly distributed methods
- 1.1 Implicitly distributed methods
- 1.0 Initial revision
- [Pitch: Distributed Actors](https://forums.swift.org/t/pitch-distributed-actors/51669)
  - Which focused on the general concept of distributed actors, and will from here on be cut up in smaller, reviewable pieces that will become their own independent proposals. Similar to how Swift Concurrency is a single coherent feature, however was introduced throughout many interconnected Swift Evolution proposals.

[isolation]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0336-distributed-actor-isolation.md
