# Distributed Actor Isolation

* Proposal: [SE-0336](0336-distributed-actor-isolation.md)
* Authors: [Konrad 'ktoso' Malawski](https://github.com/ktoso), [Pavel Yaskevich](https://github.com/xedin), [Doug Gregor](https://github.com/DougGregor), [Kavon Farvardin](https://github.com/kavon)
* Review Manager: [Joe Groff](https://github.com/jckarter)
* Status: **Implemented (Swift 5.7)**
* Decision Notes: [Acceptance](https://forums.swift.org/t/accepted-se-0336-distributed-actor-isolation/54726)
* Implementation: 
  * Partially available in [recent `main` toolchain snapshots](https://swift.org/download/#snapshots) behind the `-enable-experimental-distributed` feature flag. 
  * This flag also implicitly enables `-enable-experimental-concurrency`.
* Sample app:
  * A sample app, showcasing how the various "pieces" work together is available here:
    [https://github.com/apple/swift-sample-distributed-actors-transport](https://github.com/apple/swift-sample-distributed-actors-transport)

## Table of Contents

- [Distributed Actor Isolation](#distributed-actor-isolation)
  - [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
      - [Useful links](#useful-links)
  - [Motivation](#motivation)
    - [Location Transparency](#location-transparency)
    - [Remote and Local Distributed Actors](#remote-and-local-distributed-actors)
  - [Proposed solution](#proposed-solution)
    - [Distributed Actors](#distributed-actors)
    - [Complete isolation of state](#complete-isolation-of-state)
    - [Distributed Methods](#distributed-methods)
  - [Detailed design](#detailed-design)
    - [Distributed Actors and Distributed Actor Systems](#distributed-actors-and-distributed-actor-systems)
    - [Distributed Actor Initializers](#distributed-actor-initializers)
    - [Distributed Actors implicitly conform to Codable](#distributed-actors-implicitly-conform-to-codable)
    - [Distributed Methods](#distributed-methods-1)
      - [Distributed Method Serialization Requirements](#distributed-method-serialization-requirements)
      - [Distributed Methods and Generics](#distributed-methods-and-generics)
      - [Distributed Methods and Existential Types](#distributed-methods-and-existential-types)
      - [Implicit effects on Distributed Methods](#implicit-effects-on-distributed-methods)
      - [Isolation states and Implicit effects on Distributed Methods](#isolation-states-and-implicit-effects-on-distributed-methods)
    - [Distributed Actor Properties](#distributed-actor-properties)
      - [Stored properties](#stored-properties)
      - [Computed properties](#computed-properties)
    - [Protocol Conformances](#protocol-conformances)
      - [The `DistributedActor` protocol and protocols inheriting from it](#the-distributedactor-protocol-and-protocols-inheriting-from-it)
    - [Breaking through Location Transparency](#breaking-through-location-transparency)
  - [Future Directions](#future-directions)
    - [Versioning and Evolution of Distributed Actors and Methods](#versioning-and-evolution-of-distributed-actors-and-methods)
      - [Evolution of parameter values only](#evolution-of-parameter-values-only)
      - [Evolution of distributed methods](#evolution-of-distributed-methods)
    - [Introducing the `local` keyword](#introducing-the-local-keyword)
  - [Alternatives Considered](#alternatives-considered)
    - [Implicitly `distributed` methods / "opt-out of distribution"](#implicitly-distributed-methods--opt-out-of-distribution)
    - [Introducing "wrapper" type for `Distributed<SomeActor>`](#introducing-wrapper-type-for-distributedsomeactor)
    - [Creating only a library and/or source-generation tool](#creating-only-a-library-andor-source-generation-tool)
  - [Acknowledgments & Prior Art](#acknowledgments--prior-art)
  - [Source compatibility](#source-compatibility)
  - [Effect on ABI stability](#effect-on-abi-stability)
  - [Effect on API resilience](#effect-on-api-resilience)
  - [Changelog](#changelog)

## Introduction

With the recent introduction of [actors](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0306-actors.md) to the language, Swift gained powerful and foundational building blocks for expressing *thread-safe* concurrent programs. This proposal is the first in a series of proposals aiming to extend Swift's actor runtime with the concept of *distributed actors*, allowing developers leverage the actor model not only in local, but also distributed settings.

With distributed actors, we acknowledge that the world we live in is increasingly built around distributed systems, and that we should provide developers with better tools to work within those environments. We aim to simplify and push the state-of-the-art for distributed systems programming in Swift as we did with concurrent programming with local actors and Swift’s structured concurrency approach embedded in the language.

> The distributed actor proposals will be structured similarly to how Swift Concurrency proposals were: as a series of interconnected proposals that build on top of each other.

This proposal focuses on the extended actor isolation and type-checking aspects of distributed actors. 

#### Useful links

Swift Evolution:

- [Distributed Actors: Pitch #1](https://forums.swift.org/t/pitch-distributed-actors/51669) - a comprehensive, yet quite large, pitch encompassing all pieces of the distributed actor feature; It will be split out into smaller proposals going into the details of each subject, such that we can focus on, and properly review, its independent pieces step by step.

While this pitch focuses _only_ on the actor isolation rules, we have work-in-progress transport implementations for distributed actors available as well. While they are work-in-progress and do not make use of the complete model described here, they may be useful to serve as reference for how distributed actors might be used.

- [Swift Distributed Actors Library](https://www.swift.org/blog/distributed-actors/) - a reference implementation of a *peer-to-peer cluster* for distributed actors. Its internals depend on the work in progress language features and are dynamically changing along with these proposals. It is a realistic implementation that we can use as reference for these design discussions.
- "[Fishy Transport](https://github.com/apple/swift-sample-distributed-actors-transport)" Sample - a simplistic example transport implementation that is easier to follow the basic integration pieces than the realistic cluster implementation. Feel free to refer to it as well, while keeping in mind that it is very simplified in its implementation approach.

## Motivation

Distributed actors are necessary to expand Swift's actor model to distributed environments. The new `distributed` keyword offers a way for progressively disclosing the additional complexities that come with multiprocess or multi-node environments, into the local-only actor model developers are already familiar with.

Distributed actors need stronger isolation guarantees than those that are offered by Swift's "local-only" actors. This was a conscious decision, as part of making sure actors are convenient to use in the common scenario where they are only used as concurrency isolation domains. This convenience though is too permissive for distributed programming. 

This proposal introduces the additional isolation checks necessary to allow a distributed runtime to utilize actors as its primary building block, while keeping the convenience and natural feel of such actor types.

### Location Transparency

The design of distributed actors intentionally does not provide facilities to easily determine whether an instance is local or remote. The programmer should not _need_ to think about where the instance is located, because Swift will make it work in either case. There are numerous benefits to embracing location transparency:

- The programmer can write a complex distributed systems algorithm and test it locally. Running that program on a cluster becomes merely a configuration and deployment change, without any additional source code changes.
- Distributed actors can be used with multiple transports without changing the actor's implementation.
- Actor instances can be balanced between nodes once capacity of a cluster changes, or be passivated when not in use, etc. There are many more advanced patterns for allocating instances, such as the "virtual actor" style as popularized by Orleans or Akka's cluster sharding.

Swift's take on location transparency is expressed and enforced in terms of actor isolation. The same way as actors isolate their state to protect from local race conditions, distributed actors must isolate their state because the state "might not actually be available locally" while we're dealing with a remote distributed actor reference.

It is also possible to pass distributed actors to distributed methods, if the actor is able to conform to the serialization requirements imposed on it by the actor system.

### Remote and Local Distributed Actors

For the purpose of this proposal, we omit the implementation details of a remote actor reference, however as the purpose of actor isolation is to erase the observable difference between a local and remote instance (to achieve location transparency), we need to at least introduce the general concept.

It is, by design, not possible to *statically* determine if a distributed actor instance is remote or local, therefore all programming against a distributed actor must be done as-if it was remote. This is the root reason for most of the isolation rules introduced in this proposal. For example, the following snippet illustrates location transparency in action, where in our tests we use a local instance, but in a real deployment they would be remote instances communicating:

```swift
distributed actor TokenRange {
  let range: (Token, Token)
  var storage: [Token: Data]
  
  init(...) { ... }
  
  distributed func read(at loc: Token) -> Data? {
    return storage[loc]
  }

  distributed func write(to loc: Token, data: Data) -> Data? {
    let prev = storage[loc]
    storage[loc] = data
    return prev
  }
}
```

Which can be used in a local test:

```swift
func test_distributedTokenRange() async throws {}
  let range = TokenRange(...)
  try await assert(range.read(at: testToken) == nil)
  
  try await write(to: testToken, someData)
  try await assert(range.read(at: testToken) == someData)
}
```

Distributed functions must be marked with `try` and `await` because they imply asynchronous network calls which may fail. While the `await` rule is the same as with local-only actors, the rule about distributed methods throwing is unique to them because of the assumption that underlying transport mechanisms can fail (i.e. network or serialization errors), regardless if the called function is able to throw or not.

Note that the even though this test is strictly local -- there are no remote actors involved here at all -- the call-sites of distributed methods have implicitly gained the async and throwing effects, which means that we must invoke them with `try await dist.<invocation>` This is an important aspect of the design, as it allows us to surface any potential network issues that might occur during these calls, such as timeouts, network failures or other issues that may have caused these calls to fail. This failure is a natural consequence of the calls potentially having to cross process or network boundaries. The asynchronous effect is similar, because we might be waiting for a long time for a response to arrive, distributed calls must be potential suspension points.

We could write the same unit-test using a distributed remote actor, and the test would remain exactly the same:

```swift
func test_distributedTokenRange() async throws {}
  // the range is actually 'remote' now
  let range: TokenRange = <obtain remote instance, using e.g. test-system>
  try await assert(range.read(at: testToken) == nil)
  
  try await write(to: testToken, someData)
  try await assert(range.read(at: testToken) == someData)
}
```

During this proposal, we will be using the following phrases which have well-defined meanings, so in order to avoid confusion, let us define them explicitly up-front:

- _distributed actor type_ - any `distributed actor` declaration, or `protocol` declaration that also conforms to `DistributedActor` because they can only be implemented by specific distributed actors, e.g. `protocol Worker: DistributedActor` as well as `distributed actor Worker`, both, can be referred to as "distributed actor type"
- _distributed actor reference_ - any variable, or parameter referring to a distributed actor instance (regardless if remote or local),
- _known-to-be-local distributed actor_, or "_distributed local actor_" for short - a specific known to be local instance of a distributed actor. A distributed actor reference can be checked at runtime if it is remote or local, but in certain situations it is also known in the type system that an actor is "definitely local" and not all isolation checks need to be applied,
- "_distributed remote actor_" - an instance of a distributed actor type, that is actually "remote" and therefore does not have any storage allocated and effectively functions like a "proxy" object. This state does not exist anywhere explicitly in the type-system explicitly, and is what we assume every distributed actor is, unless proven to be "known to be local".

Keeping this in mind, let us proceed to discussing the specific isolation rules of distributed actors.

## Proposed solution

### Distributed Actors

Distributed actors are a flavor of the `actor` type that enforces additional rules on the type and its instances in order to enable location transparency. Thanks to this, it is possible to program against a `distributed actor` without *statically* knowing if a specific instance is remote or local. All calls are made to look as-if they were remote, and in the local case simply no networking s performed and the calls execute the same as if they were a normal local-only actor.

Distributed actors are declared by prepending `distributed` to an `actor` declaration:

```swift
distributed actor Player {
  // ...
  let name: String
}
```

While we do not deep dive into the runtime representation in this proposal, we need to outline the general idea behind them: a `distributed actor` is used to represent an actor which may be either *local* or *remote*. 

This property of hiding away information about the location of the actual instance is called _location transparency_. Under this model, we must program against such location transparent type as-if it was remote, even when it might not be. This allows us to develop and test distributed algorithms locally, without having to resort to networking (unless we want to), vastly simplifying the testing of such systems.

> **Note:** This is not the same as making "remote calls look like local ones" which has been a failure of many RPC systems. Instead, it is the opposite! Pessimistically assuming that all calls made cross-actor to a distributed actor may be remote, and offering specific ways to guarantee that some calls are definitely local (and thus have the usual, simpler isolation rules).

Distributed actor isolation checks introduced by this proposal serve the purpose of enforcing the property of location transparency, and helping developers not accidentally break it. For example, the above `Player` actor could be used to represent an actor in a remote host, where the same game state is stored and references to player's devices are managed. As such, the _state_ of a distributed actor is not known locally. This brings us to the first of the additional isolation checks: properties.

### Complete isolation of state

Because a distributed actor, along with its actual state, may be located on a remote host, some conveniences local-only actors allow cannot be allowed for distributed ones. Let's consider the following `Player` type:

```swift
public distributed actor Player {
  public let name: String
  public var score: Int
}
```

Such actor may be running on some remote host, meaning that if we have a "remote reference" to it we _do not_ have its state available, and any attempt to get it would involve network communication. Because of that, stored properties are not accessible across distributed actors:

```swift
let player: Player = // ... get remote reference to Player
player.name // ❌ error: distributed actor state is only available within the actor instance
```

Developers should think carefully about operations that cross into the actor's isolation domain, because the cost of each operation can be very expensive (e.g., if the actor is on a machine across the internet). Properties make it very easy to accidentally make multiple round-trips:

```swift
func example1(p: Player) async throws -> (String, Int) {
  try await (p.name, p.score) // ❌ might make two slow network round-trips to `p`
}
```

Instead, the use of methods to perform a batched read is strongly encouraged.

Stored properties can only be accessed when the actor is known-to-be-local, a property that is possible to check at runtime using the `whenLocal` function that we'll discuss later during this proposal. The following snippet illustrates one example of such known-to-be-local actor access, though there can be different situations where this situation occurs:

```swift
distributed actor Counter {
  var count = 0
  
  func publishNextValue() {
    count += 1
    Task.detached { @MainActor in
       ui.countLabel.text = "Count is now \(await self.count)"
     }
  }
}
```

Stored properties cannot be declared `distributed` nor `nonisolated`. Computed properties however can be either of the two. However, computed properties can only be `distributed` if they are `get`-only due to limitations in how effectful properties work, in which case they function effectively the same as distributed methods which we'll discuss next.

### Distributed Methods

In order to enforce the distributed "*maybe remote*" nature of distributed actors, this proposal introduces a new flavor of method declaration called a *distributed method*. Other than a few special cases (such as `nonisolated` members), distributed methods are the only members that can be invoked cross-actor on distributed actors.

It is necessary to give developers tight control over the distributed nature of methods they write, and it must be a conscious opt-in step.  It is also possible to declared computed properties as `distributed`. A distributed method or property is defined within a distributed actor type by writing `distributed` in front of the method's declaration:

```swift
distributed actor Player { 
  
  distributed func yourTurn() -> Move { 
    return thinkOfNextMove() 
  }
  
  func thinkOfNextMove() -> Move {
    // ... 
  }
  
  distributed var currentTurn: Int { 
    // ... 
  }
}
```

It is not possible to invoke the `thinkOfNextMove()` method cross-actor, because the target of the invocation may be remote, and it was not "exposed" for distribution using the `distributed func` keywords. This is checked at compile time and is a more restrictive form of actor-isolation checking:

```swift
func test(p: Player) async throws { 
  try await p.yourTurn() 
  // ✅ ok, distributed func
  
  try await p.currentTurn
  // ✅ ok, distributed computed property
  
  try await p.thinkOfNextMove() 
  // ❌ error: only 'distributed' instance methods can be called on a potentially remote distributed actor
}
```

Distribution must not be simply inferred from access-control, because the concept of distribution is orthogonal to access control. For example, it is very much common to have `internal distributed func` (or even `private distributed func`) declarations, which are useful for actors within a module communicating with each other (remotely), however those methods should be invoked be end-users of such library. 

Distributed methods may be subject to additional type-checking, specifically a distributed actor infers a `SerializationRequirement` from the ActorSystem it is associated with. One common serialization requirement is `Codable`.

Such `SerializationRequirement` typealias defined on the actor system the actor is associated with causes additional type-checks to be enforced on distributed methods: all parameter types and return type of such method must be or conform to the SerializationRequirement type. This allows the compiler to fail compilation early, rather than leaving serialization crashes to the runtime, easing development and analysis of distributed actor systems:

```swift
distributed actor Player { 
  typealias ActorSystem = CodableMessagingSystem 
  // inferred: typealias SerializationRequirement = Codable
  
  distributed func test(not: NotCodable) {} 
  // ❌ error: parameter 'not' of type 'NotCodable' in distributed instance method
  //           does not conform to 'Codable'
}
```



## Detailed design

Unless otherwise specified in this proposal, the semantics of a distributed actor are the same as a regular actor, as described in [SE-0306](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0306-actors.md).

### Distributed Actors 

Distributed actors can only be declared using the `distributed actor` keywords. Such types automatically conform to the `DistributedActor` protocol. The protocol is defined in the `_Distributed` module as follows:

```swift
/// Common protocol to which all distributed actors conform.
///
/// The `DistributedActor` protocol generalizes over all distributed actor types.
/// All distributed actor types implicitly conform to this protocol.
/// 
/// It is not possible to explicitly conform to this protocol using any other declaration 
/// other than a 'distributed actor', e.g. it cannot be conformed to by a plain 'actor' or 'class'.
/// 
/// ### Implicit Codable conformance
/// If the 'ID' conforms to `Codable` then the concrete distributed actor adopting this protocol 
/// automatically gains a synthesized Codable conformance as well. This is because the only reasonable
/// way to implement coding of a distributed actor is to encode it `ID`, and decoding can make use of
/// decoding the same ID, and resolving it using an actor system found in the Decoder's `userInfo`.
///
/// This works well with `Codable` serialization requirements, and allows actor references to be 
/// sent to other distributed actors.
protocol DistributedActor: AnyActor, Identifiable, Hashable 
  where ID == ActorSystem.ActorID {
    
  /// Type of the distributed actor system this actor is able to operate with.
  /// It can be a type erased, or existential actor system (through a type-eraser wrapper type),
  /// if the actor is able to work with different ones.
  associatedtype ActorSystem: DistributedActorSystem
  
  /// The serialization requirement to apply to all distributed declarations inside the actor.
  typealias SerializationRequirement = ActorSystem.SerializationRequirement

  /// Unique identity of this distributed actor, used to resolve remote references to it from other peers,
  /// and also enabling the Hashable and (optional) Codable conformances of a distributed actor.
  /// 
  /// The id may be freely shard across tasks and processes, and resolving it should return a reference
  /// to the actor where it originated from.
  nonisolated override var id: ID { get }
  
  /// Distributed Actor System responsible for managing this distributed actor.
  ///
  /// It is responsible for assigning and managing the actor's id, 
  /// as well as delivering incoming messages as distributed method invocations on the actor.
  nonisolated var actorSystem: ActorSystem { get }
}
```

All distributed actors are *explicitly* part of some specific distributed actor system. The term "actor system" originates from both early, and current terminology relating to actor runtimes and loosely means "group of actors working together", which carries a specific meaning for distributed actors, because it implies they must be able to communicate over some (network or ipc) protocol they all understand. In Swift's local-only actor model, the system is somewhat implicit, because it simply is "the runtime", as all local objects can understand and invoke each other however they see fit. In distribution this needs to become a little more specific: there can be different network protocols and "clusters" to which actors belong, and as such, they must be explicit about their actor system use. We feel this is an expected and natural way to introduce the concept of actor systems only once we enter distribution, because previously (in local only actors) the concept would not have added much value, but in distribution it is the *core* of everything distributed actors do.

The protocol also includes two nonisolated property requirements: `id` and `actorSystem`. Witnesses for these requirements are nonisolated computed properties that the compiler synthesizes in specific distributed actor declarations. They store the actor system the actor was created with, and its id, which is crucial to its lifecycle and messaging capabilities. We will not discuss in depth how the id is assigned in this proposal, but in short: it is created and assigned by the actor system during the actor's initialization.

Note, that the `DistributedActor` protocol does *not* refine the `Actor` protocol, but instead it refines the `AnyActor` protocol, also introduced in this proposal. This detail is very important to upholding the soundness of distributed actor isolation. 

Sadly, just refining the Actor protocol results in the following unsound isolation behavior: 

```swift
// Illustrating isolation violation, IF 'DistributedActor' were to refine 'Actor':
extension Actor {
  func f() -> SomethingSendable { ... }
}
func g<A: Actor>(a: A) async {
  print(await a.f())
}

// given any distributed actor:
actor MA: DistributedActor {} // : Actor implicitly (not proposed, for illustration purposes only)

func h(ma: MA) async {
  await g(ma) // 💥 would be allowed because a MA is an Actor, but can't actually work at runtime 
}
```

The general issue here is that a distributed actor type must uphold its isolation guarantees, because the actual instance of such type may be remote, and therefore cannot be allowed to have non-distributed calls made on it. One could argue for the inverse relationship, that `Actor: DistributedActor` as the Actor is more like "`LocalActor`", however this idea also breaks down rather quickly, as one would expect "any IS-A distributed actor type, to have distributed actor isolation", however we definitely would NOT want `actor Worker {}` suddenly exhibit distributed actor isolation. In a way, this way of inheritance breaks the substitution principle in weird ways which could be hacked together to make work, but feel fragile and would lead to hard to understand isolation issues.

In order to prevent this hole in the isolation model, we must prevent `DistributedActor` from being downcast to `Actor` and the most natural way of doing so, is introducing a shared super-type for the two Actor-like types: `AnyActor`.

```swift
@_marker
@available(SwiftStdlib 5.6, *)
public protocol AnyActor: Sendable, AnyObject {}

public protocol Actor: AnyActor { ... }
public protocol DistributedActor: AnyActor, ... { ... }
```

Thanks to this protocol we gain an understandable, and complete, type hierarchy for all actor-like behaviors, that is, types that perform a kind of isolation checking and guarantee data-race freedom to invocations on them by serializing them through an actor mailbox. This does not incur much implementation complexity in practice because functionality wise, distributed actors mirror actors exactly, however their customization of e.g. executors only applies to local instances.

## Distributed Actor Systems

Libraries aiming to implement distributed actor systems, and act as the runtime for distributed actors must implement the `DistributedActorSystem`. We will expand the definition of this protocol with important lifecycle functions in the runtime focused proposal, however for now let us focus on its aspects which affect type checking and isolation of distributed actors. The protocol is defined as:

```swift
public protocol DistributedActorSystem: Sendable {
  associatedtype ActorID: Hashable & Sendable // discussed below
  
  /// The serialization requirement that will be applied to all distributed targets used with this system.
  typealias SerializationRequirement = // (simplified, actually an associatetype)
  
  // ... many lifecycle related functions, to be defined in follow-up proposals ... 
} 
```

Every distributed actor must declare what distributed actor system it is able to work with, this is expressed as an `associatedtype` requirement on the `DistributedActor` protocol, to which all `distributed actor` declarations conform implicitly. For example, this distributed actor works with some `ClusterSystem`:

```swift
distributed actor Worker { 
  typealias ActorSystem = ClusterSystem
}
```

The necessity of declaring this statically will become clear as we discuss the serialization requirements and details of the typechecking mechanisms in the sections below.

Please note that it is possible to use a protocol or type eraser as the actor system, which allows actors to swap-in completely different actor system implementations, as long as their serialization mechanisms are compatible. Using existential actor systems though comes at a slight performance penalty (as do all uses of existentials).

It is possible to declare a module-wide `typealias DefaultDistributedActorSystem` in order to change this "default" actor system type, for all distributed actor types declared within a module:

```swift
// in 'Cluster' module:
typealias DefaultDistributedActorSystem = ClusterSystem

// in 'Cluster' module, clearly we want to use the 'ClusterSystem'
distributed actor Example {
  // synthesized:
  // typealias DistributedActorSystem = DefaultDistributedActorSystem // ClusterSystem

  // synthesized initializers (discussed below) also accept the expected type then:
  // init(system: DefaultDistributedActorSystem) { ... }
}
```

It is also possible to declare protocols which refine the general `DistributedActor` concept to some specific transport, such as:

```swift
protocol ClusterActor: DistributedActor where DistributedActorSystem == ClusterSystem {}

protocol XPCActor: DistributedActor where DistributedActorSystem == XPCSystem {	}
```

Those protocols, because they refine the `DistributedActor` protocol, can also only be conformed to by other distributed actors. It allows developers to declare specific requirements to their distributed actor's use, and even provide extensions based on the actor system type used by those actors, e.g.:

```swift
extension DistributedActor where DistributedActorSystem == ClusterSystem {
  /// Returns the node on which this distributed actor instance is located.
  nonisolated var node: Cluster.Node? { ... }
}
```

>  **Note:** We refer to `distributed actor` declarations or protocols refining the `DistributedActor` protocol as any "distributed actor type" - wherever this phrase is used, it can apply to a specific actor or such protocol.

### Distributed Actor Initializers

Distributed actor initializers are always _local_, therefore no special rules are applied to their isolation checking.

Distributed actor initializers are subject to the same isolation rules as actor initializers, as outlined in [SE-0327: On Actors and Initialization](https://forums.swift.org/t/se-0327-on-actors-and-initialization/53053). Please refer to that proposal for details about when it is safe to escape `self` out of an actor initializer, as well as when it is permitted to call other functions on the actor during its initialization.

A distributed actor's *designated initializer* must always contain exactly one `DistributedActorSystem` parameter. This is because the lifecycle and messaging of a distributed actor is managed by the system. It also assigns every newly initialized distributed actor instance an identity, that the actor then stores and makes accessible via the compiler-synthesized computed property `id`. The system is similarly available to the actor via the compiler synthesized computed property `actorSystem`.

Similar to classes and local-only actors, a distributed actor gains an implicit default designated initializer when no user-defined initializer is found. This initializer accepts an actor system as parameter, in order to conform to the requirement stated above:

```swift
// default system for this module:
typealias DefaultDistributedActorSystem = SomeSystem

distributed actor Worker { 
  // synthesized default designated initializer:
  // init(system: DefaultDistributedActorSystem)
}
```

if no module-wide `DefaultDistributedActorSystem` is defined, such declaration would request the developer to provide one at compile time:

```swift
distributed actor Worker { 
  typealias ActorSystem = SomeSystem

  // synthesized default designated initializer:
  // init(system: SomeSystem)
}
```

Alternatively, we can infer this typealias from a user-defined initializer, like this:

```swift
distributed actor Worker { 
  // inferred typealias from explicit initializer declaration
  // typealias ActorSystem = SomeSystem
  
  init(system: SomeSystem) { self.name = "Alice" }
}
```

The necessity to pass an actor system to each newly created distributed actor is because the system is the one assigning and managing identities. While we don't discuss those details in depth in this proposal, here is a short pseudocode of why passing this system is necessary:

```swift
// Lifecycle interactions with the system during initialization
// NOT PART OF THIS PROPOSAL; These will be discussed in-depth in a forthcoming proposal focused on the runtime.
distributed actor Worker { 
  init(system: SomeSystem) {
    // self._system = system
    // the actor is assigned an unique identity as it initializes:
    // self._id = system.assignID(Self.self)
    self.name = "Alice" 
    // once fully initialized, the actor is ready to receive remote calls:
    // system.actorReady(self)
  }
}
```

Having that said, here are a few example of legal and illegal initializer declarations:

```swift
distributed actor InitializeMe { 
  init() 
  // ❌ error: designated distributed actor initializer 'init()' is missing required 'DistributedActorSystem' parameter
  
  init(x: String)
  // ❌ error: designated distributed actor initializer 'init(x:)' is missing required 'DistributedActorSystem' parameter

  init(system: AnyDistributedActorSystem, too many: AnyDistributedActorSystem)
  // ❌ error: designated distributed actor initializer 'init(system:too:)' must accept exactly one DistributedActorSystem parameter, found 2
  
  // --------
  
  
  init(system: AnyDistributedActorSystem) // ✅ ok
  init(y: Int, system: AnyDistributedActorSystem) // ✅ ok
  init(canThrow: Bool, system: AnyDistributedActorSystem) async throws // ✅ ok, effects are ok too
  
  // 'convenience' may or may not be necessary, depending on SE-0327 review outcome.
  convenience init() { 
    self.init(system: SomeSystem(...)) // legal, but not recommended
  }
}
```

*Remote* distributed actor references are not obtained via initializers, but rather through a static `resolve(_:using:)` function that is available on any distributed type:

```swift
extension DistributedActor { 
  
  /// Resolves the passed in `id` using the passed distributed actor `system`, 
  /// returning either a local or remote distributed actor reference.
  ///
  /// The system will be asked to `resolve` the identity and return either
  /// a local instance or request a "proxy" to be created for this identity.
  ///
  /// A remote distributed actor reference will forward all invocations through
  /// the system, allowing it to take over the remote messaging with the
  /// remote actor instance.
  ///
  /// - Parameter id: identity uniquely identifying a, potentially remote, actor in the system
  /// - Parameter system: distributed actor system which must resolve and manage the returned distributed actor reference
  static func resolve(id: ID, using system: DistributedActorSystem) throws -> Self
}
```

The specifics of resolving, and remote actor runtime details will be discussed in a follow-up proposal focused on the runtime aspects of distributed actors. We mention it here to share a complete picture how Identities, systems, and remote references all fit into the picture.

### Distributed Actors implicitly conform to Codable

If a distributed actor's `ID` conforms to `Codable`, the distributed actor automatically gains a `Codable` conformance as well.

This conformance is synthesized by the compiler, for every specific `distributed actor` declaration. It is not possible to express such conformance using the conditional conformances. 

> **Note:** It is not possible to implement such conformance semantics on the DistributedActor protocol using conditional conformances (like this `extension DistributedActor: Codable where ID: Codable`), and it is unlikely to be supported in the future. As such, we currently opt to synthesize the conformance for specific distributed actor declarations.

```swift
distributed actor Player /*: DistributedActor, Codable */ { 
  // typealias ID = SomeCodableID
}
```

The synthesized `Codable` conformance strictly relies on the implementation of the actors' identity `Codable` conformance. When we "encode" a distributed actor, we never encode "the actor", but rather only its identity:  

```swift
// distributed actor Player: Codable, ... {
  nonisolated public func encode(to encoder: Encoder) throws {
    var container = encoder.singleValueContainer()
    try container.encode(self.id)
  }
// }
```

And similarly, decoding a distributed actor has the specific meaning of attempting to `resolve(_:using:)` a reference of the specific actor type, using the decoded id:

```swift
// distributed actor Player: Codable, ... {
  nonisolated public init(from decoder: Decoder) throws {
    // ~~~ pseudo code for illustration purposes ~~~ 
    guard let system = decoder.userInfo[.distributedActorSystemKey] as? Self.ActorSystem else {
      throw DistributedActorCodingError(message:
        "Missing DistributedActorSystem (for key .distributedActorSystemKey) " +
        "in \(decoder).userInfo, while decoding \(Self.self)!")
    }

    // [1] decode the identity
    let id: ID = try Self.ID(from: decoder)
    // [2] resolve the identity using the current system; this usually will return a "remote reference"
    self = try Self.resolve(id: id, using: system) // (!)
  }
// }
```

The Decodable's `init(from:)` implementation is actually not possible to express in plain Swift today, because the restriction on self assignment in class initializers (and therefore also actor initializers). 

> **Note:** We could eventually generalize this more mutable `self` in class/actor initializer mechanism, however that would be done as separate Swift Evolution proposal. We are aware [this feature was requested before](https://forums.swift.org/t/allow-self-x-in-class-convenience-initializers/15924), and feels like a natural follow up to this proposal to generalize this capability.

Note also that, realistically, there is only one correct way to implement a distributed actor's codability (as well as `Hashable` and `Equatable` conformances), because the only property that is related to its identity, and is known to both local and remote "sides" is the identity, as such implementations of those protocols must be directly derived from the `id` property of a distributed actor.

The capability, to share actor references across to other (potentially remote) distributed actors, is crucial for location-transparency and the ability to "send actor references around" which enables developers to implement "call me later" style patterns (since we cannot do so with closures, as they are not serializable). In a way, this is similar to the delegate pattern, known to developers on Apple platforms: where we offer an instance to some other object, that will call lifecycle or other types of methods on the delegate whenever certain events happen.

To illustrate how this capability is used in practice, let us consider the following turn-based distributed `Game` example, which waits until it has enough players gathered, and then kicks off the game by notifying all the players (regardless _where_ they are located) that the game is now starting.

```swift
typealias DefaultDistributedActorSystem = SomeCodableDistributedActorSystem
struct SomeCodableDistributedActorSystem: DistributedActorSystem {
  typealias ActorID = SomeCodableID
  typealias SerializationRequirement = Codable
}

distributed actor Player {
  distributed func play(turn: Int) -> Move { ... }
  distributed func opponentMoved(_ move: Move) { ... }
}

distributed actor Game { 
  let minPlayers = 2
  var players: Set<Player> = []
  
  distributed func join(player: Player) async throws {
    guard players.count < 2 else {
      throw ...
    }
    
    players.insert(player)
    
    if players.count == 2 { 
      await play() // keep asking players for their move via 'play(turn:)' until one of them wins
    }
  }
  
  func play() async throws {
    // keep asking players for their move via 'play(turn:)' until one of them wins
  }
  
  distributed var result: GameResult { 
     ... 
  }
}

func play(game: Game) async throws { 
  try await game.join(player: Player(system: ...))
  try await game.join(player: Player(system: ...))
  // the game begins, players are notified about it

  let result =  try await game.result
  print("Winner of \(game) was: \(result.winner)")
}
```

The `Player` distributed actor automatically gained a Codable conformance, because it is using the `SomeCodableDistributedActorSystem` that assigns it a `SomeCodableID`. Other serialization mechanisms are also able to implement this "encode the ID" and "decode the ID, and resolve it" pattern, so this pattern is equally achievable using Codable, or other serialization mechanisms.

### Distributed Methods

The primary way a distributed actor can be interacted with are distributed methods. Most notably, invoking a non-distributed method (i.e. those declared with *just* the `func` keyword by itself), is not allowed as it may be potentially violating distributed actor isolation rules, that is unless the target of the invocation is known to be a *local* distributed actor - a topic we'll explore later on in this proposal:

```swift
distributed actor IsolationExample { 
  func notDistributed() {}
  distributed func accessible() {}
  distributed var computed: String { "" }
}

func test(actor: IsolationExample) async throws {
  try await actor.notDistributed() 
  // ❌ error: only 'distributed' instance methods can be called on a potentially remote distributed actor
  
  try await actor.accessible()
  // ✅ ok, method is distributed
  
  try await actor.computed
  // ✅ ok, distributed get-only computed property
}
```

Distributed methods are declared by writing the `distributed` keyword in the place of a declaration modifier, under the `actor-isolation-modifier` production rule as specified by [the grammar in TSPL](https://docs.swift.org/swift-book/ReferenceManual/Declarations.html#grammar_declaration-modifiers). Only methods can use `distributed` as a declaration modifier, and no order is specified for this modifier. 

It is also possible to declare distributed get-only properties, and they obey the same rules as a parameter-less `distributed func` would. It is not permitted to make get/set computed properties, or stored properties `distributed`.

Distributed actor types are the only types in which a distributed method declaration is allowed. This is because, in order to implement a distributed method, an actor system and identity must be associated with the values carrying the method. Distributed methods can synchronously refer to any of the state isolated to the distributed actor instance.

The following distributed method declarations are not allowed:

```swift
actor/class/enum/struct NotDistributedActor {
  distributed func test() {} 
  // ❌ error: 'distributed' function can only be declared within 'distributed actor'
}

protocol NotDistributedActorProtocol {
  distributed func test()
  // ❌ error: 'distributed' function can only be declared within 'distributed actor'
  // 💡 fixit: add ': DistributedActor' to protocol inheritance clause
}
```

While these are all proper declarations:

```swift
distributed actor Worker { 
  distributed func work() { ... }
}

extension Worker { 
  distributed func reportWorkedHours() -> Duration { ... }
}

protocol TypicalGreeter: DistributedActor {
  distributed func greet()
}
```

The last example, the `TypicalGreeter` protocol, can *only* be implemented by a `distributed actor`, because of the `DistributedActor` requirement. We will discuss distributed actors conforming to protocols in great detail below.

It is not allowed to combine `distributed` with `nonisolated`, as a distributed function is _always_ isolated to the actor in which it is defined.

```swift
distributed actor Charlie {
  distributed nonisolated func cantDoThat() {}
  // ❌ error: 'distributed' function must not be 'nonisolated'
  // 💡 fixit: remove 'nonisolated' or 'distributed'
}
```

It is possible to declare a nonisolated method though. Such function can only access other `nonisolated` members of the instance. Two important members which are such nonisolated computed properties are the actor's identity, and associated actor system. Those are synthesized by the compiler, however they just follow the same isolation rules as laid out in this proposal:

```swift
distributed actor Charlie: CustomStringConvertible { 
  // synthesized: nonisolated var id: Self.ID { get }
  // synthesized: nonisolated var actorSystem: Self.ActorSystem { get }
  
  nonisolated var description: String { 
    "Charlie(\(self.id))" // ok to refer to `self.id` since also nonisolated
  }
}
```

Distributed methods may be declared explicitly `async` or `throws` and this has the usual effect on the declaration and method body. It has no effect on cross distributed actor calls, because such calls are implicitly asynchronous and throwing to begin with.

The `distributed` nature of a method is completely orthogonal to access control. It is even possible to declare a `private distributed func` because the following pattern may make it an useful concept to have:

```swift
distributed actor Robot {
  
  nonisolated async throws isHuman(caller: Caller) -> String {
    guard isTrustworthy(caller) else {
      return "It is a mystery!" // no remote call needs to be performed
    }
    
    return try await self.checkHumanity()
  }
  
  private distributed func checkHumanity() -> String { 
    "Human, after all!" 
  }
}
```

Such methods allow us avoiding remote calls if some local validation already can short-circuit them. While not a common pattern, it definitely can have its uses. Note that the ability to invoke distributed methods remotely, also directly translates into such methods being "effectively public", even if access control wise they are not. This makes sense, and distributed methods must always be audited and carefully checked if they indeed should be allowed to execute when invoked remotely, e.g. they may need to perform caller authentication – a feature we do not provide out of the box yet, but are definitely interested in exploring in the future.

It is not allowed to declare distributed function parameters as `inout` or varargs:

```swift
distributed actor Charlie {
  distributed func varargs(int: Int...) {}
  // ❌ error: cannot declare variadic argument 'int' in distributed instance method 'varargs(int:)'
  
  distributed func noInout(inNOut burger: inout String) {}
  // ❌ error: cannot declare 'inout' argument 'burger' in distributed instance method 'noInout(inNOut:)'
  // 💡 fixit: remove 'inout'
}
```

While subscripts share many similarities with methods, they can lead to complex and potentially impossible to support invocations, meaning that they are currently also not allowed to be `distributed`. Such subscripts' usefulness would, in any case, be severely limited by both their lack of support for being `async` (e.g., could only support read-only subscripts, because no coroutine-style accessors) and their lightweight syntax can lead to the same problems as properties.

Distributed functions _may_ be combined with property wrappers to function parameters (which were introduced by [SE-0293: Extend Property Wrappers to Function and Closure Parameters](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0293-extend-property-wrappers-to-function-and-closure-parameters.md)), and their semantics are what one would expect: they are a transformation on the syntactical level, meaning that the actual serialized parameter value is what the property wrapper has wrapped the parameter in. This is especially interesting for implementing eager validation of specific parameters, such that calls with illegal argument values can be synchronously prevented before even sending the message. Of course, the recipient should still validate the incoming arguments using the same logic, but thanks to this we are able to avoid sending wrong values in non-adversarial situations, and just validate some values on the client side eagerly.

#### Distributed Method Serialization Requirements

An important goal of the distributed actor design is being able to enforce some level of compile time safety onto distributed methods calls, which helps prevent unexpected runtime failures, and aides developers make conscious decisions which types should be exposed to remote peers and which not.

This feature is applied to `distributed` methods, and configured by declaring a `SerializationRequirement` typealias on the actor system, from which specific actors infer it. This type alias informs the type-checker to ensure that all parameters, as well as return type of distributed methods must conform to the type that is provided as `SerializationRequirement`. This is in addition to the usual `Sendable` conformance requirements enforced on any values passed to/from actors).

Another interesting capability this unlocks is being able to confine actors to sending only well-known types, if we wanted to enforce such closed-world assumptions onto the permissible messages exchanged between actors.

Most frequently, the serialization requirement is going to be `Codable`, so for the rest of this proposal we'll focus mostly on this use-case. It is equally possible and supported to provide e.g. an external serialization systems top-level protocol as requirement here, e.g. a Protocol Buffer `Message`. The following snippet illustrates how this can work in practice:

```swift
protocol CodableDistributedActorSystem: DistributedActorSystem { 
  typealias SerializationRequirement = Codable  
}

distributed actor Worker { 
  typealias ActorSystem = CodableDistributedActorSystem
  typealias SerializationRequirement = SpecificActorSystem.SerializationRequirement
  //																 = Codable
}
```

It is possible, albeit not recommended, to disable this checking by setting the `SerializationRequirement` to `Any` in which case no additional checks are performed on distributed methods.

This section will discuss the implications of the `SerializationRequirement` on distributed method declarations. 

A serialization requirement means that all parameter types and return type of distributed method must conform to the requirement. With the `CodableDistributedActorSystem` in mind, let us write a few methods and see how this works:

```swift
distributed actor Worker { 
  typealias ActorSystem = CodableDistributedActorSystem
  
  distributed func ok() // ✅ ok, no parameters
  distributed func greet(name: String) -> String // ✅ ok, String is Codable

  struct NotCodable {}
  
  distributed func reject(not: NotCodable)
  // ❌ error: parameter 'not' of type 'NotCodable' in distributed instance method
  //           does not conform to 'Codable'
  // 💡 fixit: add ': Codable' to 'struct NotCodable'
}
```

This also naturally extends to closures without any the need of introducing any special rules, because closures do not conform to protocols (such as `Codable`), the following is naturally ill-formed and rejected:

```swift
distributed actor Worker { 
  typealias ActorSystem = CodableDistributedActorSystem
  
  distributed func take(_ closure: (String) -> String) 
  // ❌ error: parameter 'closure' of type '(String) -> String' in distributed instance method
  //           does not conform to 'Codable'
}
```

Thrown errors are not enforced to be `Codable`, however a distributed actor system may detect that an error is Codable at runtime, and attempt to transfer it back entirely. For throws of non-Codable types, systems should attempt some form of best-effort description of the error, while keeping in mind privacy of error descriptions. I.e. errors should never be sent back to the caller by just getting their description, as that may leak sensitive information from the server system. A recommended approach here is to send back the type of the thrown error and throwing some generic `NotCodableError("\(type(of: error))")` or similar.

Distributed actors may also witness protocol requirements (discussed in more detail below), however their method declarations must then also conform to the `SerializationRequirement`:

```swift
protocol Greetings { 
  func greet(name: String) async throws
  func synchronous()
}

distributed actor Greeter: Greetings { 
  // typealias SerializationRequirement = Codable
  distributed func greet(name: String) { // may or may not be async/throws, it always is when cross-actor
    // ✅ ok, String is Codable
  }
  
  nonisolated func synchronous() {} // nonisolated func may be used the same as on normal actors 
}
```

Note that while every `distributed actor` must be associated with some specific distributed actor system, protocols need not be so strict and we are allowed to specify a distributed actor protocol like this:

```swift
protocol Greetings: DistributedActor {
  // no specific ActorSystem requirement (!)
  func greet(name: String)
}
```

At the declaration site of such protocol the distributed functions are *not* subject to any `SerializationRequirement` checks. However once it is implemented by a distributed actor, that actor will be associated with a specific actor system, and thus also a specific SerializationRequirement, and could potentially not be able to implement such protocol because of the serializability checks, e.g.:

```swift
protocol Greetings: DistributedActor {
  // no specific ActorSystem requirement (!)
  func greet(name: String)
}

distributed actor Greeter { 
  // typealias SerializationRequirement = MagicMessage
  distributed func greet(name: String) {} 
  // ❌ error: parameter 'name' of type 'String' in distributed instance method 
  //   				 does not conform to 'MagicMessage'
}
```

A similar mechanism will exist for resolving remote actor references only based on a protocol.

#### Distributed Methods and Generics

It is possible to declare and use distributed methods that make use of generics. E.g. we could define an actor that picks an element out of a collection, yet does not really care about the element type:

```swift
distributed actor Picker { 
  func pickOne<Item>(from items: [Item]) -> Item? { // Is this ok? It depends...
     ... 
  }
}
```

This is possible to implement in general, however the `Item` parameter will be subject to the same `SerializableRequirement` checking as any other parameter. Depending on the associated distributed actor system's serialization requirement, this declaration may fail to compile, e.g. because `Item` was not guaranteed to be `Codable`:

```swift
distributed actor Picker { 
  // typealias ActorSystem = CodableMessagingSystem
  func pickOne<Item>(from items: [Item]) -> Item? { nil } 
  // ❌ error: parameter 'items' of type '[Item]' in distributed instance method 
  //   				 does not conform to 'Codable'
  // ❌ error: return type 'Item' in distributed instance method does not conform to 'Codable'
  
  func pickOneFixed<Item>(from items: [Item]) -> Item? 
    where Item: Codable { nil } // ✅ ok, we declared that the generic 'Item' is 'Codable'
}
```

This is the same rule about serialization requirements really, but spelled out explicitly.

 The runtime implementation of such calls is more complicated than non-generic calls, and does incur a slight wire envelope size increase, because it must carry the *specific type identifier* that was used to perform the call (e.g. that it was invoked using the *specific* `struct MyItem: Item` and not just some item). Generic distributed function calls will perform the deserialization using the *specific type* that was used to perform the remote invocation. 

As with any other type involved in message passing, actor systems may also perform additional inspections at run time of the types and check if they are trusted or not before proceeding to decode them (i.e. actor systems have the possibility to inspect incoming message envelopes and double-check involved types before proceeding tho decode the parameters).

It is also allowed to make distributed actors themselves generic, and it works as one would expect:

```swift
distributed actor Worker<Item: Codable> { // ✅ ok
  func work() -> Item { ... }
}
```



#### Distributed Methods and Existential Types

It is worth calling out that due to existential types not conforming to themselves, it is not possible to just pass a `Codable`-conforming existential as parameter to distributed functions. It will result in the following compile time error:

```swift
protocol P: Codable {} 

distributed actor TestExistential {
  typealias ActorSystem = CodableMessagingSystem
  
  distributed func compute(s: String, i: Int, p: P) {}
  // ❌ error: parameter 'p' of type 'P' in distributed instance method does not conform to 'Codable'
}
```

The way to deal with this, as with usual local-only Swift programming, is to make the `P` existential generic, like this:

```swift
protocol P: Codable {} 

distributed actor TestExistential {
  typealias ActorSystem = CodableMessagingSystem
  
  distributed func compute<Param: P>(s: String, i: Int, p: Param) {}
  // ✅ ok, the generic allows us getting access to the specific underlying type
}
```

which will compile, and work as expected.

#### Implicit effects on Distributed Methods

Local-only actor methods can be asynchronous , throwing or both, however invoking them cross-actor always causes them to become implicitly asynchronous:

```swift
// Reminder about implicit async on actor functions
actor Greeter { 
  func greet() -> String { "Hello!" } 
  func inside() { 
    greet() // not asynchronous, we're not crossing an actor boundary
  }
}

Task {
  await Greeter().hi() // implicitly asynchronous
}
```

The same mechanism is extended to the throwing behavior of distributed methods. Distributed cross-actor calls may fail not only because of the remote side actively throwing an error, but also because of transport errors such as network issues or serialization failures. Therefore, distributed cross-actor calls also implicitly gain the throwing effect, and must be marked with `try` when called:

```swift
distributed actor Greeter { 
  distributed func greet() -> String { "Hello!" }
  
  func inside() { 
    greet() // not asynchronous or throwing, we're inside the actual local instance
  }
}

Task {
  try await Greeter().greet() // cross-actor distributed function call: implicitly async throws
}
```

It is also possible to declare distributed functions as either `throws` or `async` (or both). The implicitly added effect is a no-op then, as the function always was, respectively, throwing or asynchronous already.

The following snippets illustrate all cases how effects are applied to distributed actor methods:

```swift
distributed actor Worker {
  distributed func simple() {}
  distributed func funcAsync() async {}
  distributed func funcThrows() throws {} 
  distributed func funcAsyncThrows() async throws {}
}
```

Cross distributed-actor calls behave similar to cross actor calls, in the sense that they gain those implicit effects. This is because we don't know if the callee is remote or local, and thus assume that it might be remote, meaning that there may be transport errors involved in the call, making the function call implicitly throwing:

```swift
func outside(worker: Worker) async throws { 
  // wrong invocation:
  worker.simple()
  // ❌  error: expression is 'async' but is not marked with 'await'
  // ❌  error: call can throw but is not marked with 'try'
  // 💡 note: calls to distributed instance method 'simple()' from outside of its actor context are implicitly asynchronous
  
  // proper invocations:
  try await worker.simple()
  try await worker.funcAsync()
  try await worker.funcThrows()
  try await worker.funcAsyncThrows()
}
```

These methods may be also be called from *inside* the actor, as well as on an `isolated` parameter of that actor type, without any implicit effects applied to them. This is the same idea applies that actor methods becoming implicitly asynchronous but only during cross-actor calls.

```swift
extension Worker { 
  distributed func inside() async throws { 
    self.simple()
    await self.funcAsync()
    try self.funcThrows()
    try await self.funcAsyncThrows()
  }
}

func isolatedFunc(worker: isolated Worker) async throws { 
  worker.simple()
  await worker.funcAsync()
  try worker.funcThrows()
  try await worker.funcAsyncThrows()
}
```

The isolated function parameter works because the only way to offer an `isolated Worker` to a function, is for a real local actor instance to offer its `self` to `isolatedFunc`, and because of that it is known that it is a real local instance (after all, only a real local instance has access to `self`).

It is not allowed to declare `isolated` parameters on distributed methods, because distributed methods _must_ be isolated to the actor they are declared on. This can be thought of always using an `isolated self: Self` parameter, and in combination of a func only being allowed to be isolated to a single actor instance, this means that there cannot be another isolated parameter on such functions. Following this logic a `nonisolated func` declared on a distributed actor, _is_ allowed to accept `isolated` parameters, however such call will not be crossing process boundaries.

It is also worth calling out the interactions with `Task` and `async let`. Their context may be the same asynchronous context as the actor, in which case we also do not need to cause the implicit asynchronous effect. When it is known the invocation is performed on an `isolated` distributed actor reference, we infer the fact that it indeed is "known to be local", and do not need to apply the implicit throwing effect either:

```swift
extension Worker {
  func test(other: Philosopher) async throws {
    // self --------------------------------------------------------------------
    async let alet = self.simple() // implicitly async; async let introduced concurrent context
    _ = await alet // not throwing, but asynchronous!

    Task {
      _ = self.hi() // no implicit effects, Task inherited the Actor's execution context
    }

    Task.detached {
      _ = await self.hi() // implicitly async, different Task context than the actor
      // however not implicitly throwing; we know there is no networking involved in a call on self
    }

    // other -------------------------------------------------------------------
    async let otherLet = other.hi() // implicitly async and throws; other may be remote
    _ = try await otherLet // forced to 'try await' here, as per usual 'async let' semantics

    Task {
      _ = try await other.hi() // implicitly async and throws
    }

    Task.detached {
      _ = try await other.hi() // implicitly async and throws
    }
  }
}
```

#### Isolation states and Implicit effects on Distributed Methods

A distributed actor reference. such as a variable or function parameter, effectively can be in one of three states:

- `isolated` – as defined by Swift's local-only actors. The `isolated` also implies the following "local" state, because it is not possible to pass isolated members across distributed boundaries,
- "local" – not explicitly modeled in the type-system in this proposal, though we might end up wanting to do so (see Future Directions), or
- "potentially remote" – which is the default state of any distributed actor variable.

These states determine the implicit effects that function invocations, and general distributed actor isolation checking, need to apply when checking accesses through the distributed actor reference. 

Let us discuss the implications of these states on the effects applied to method calls on such distributed actor references, starting from the last "potentially remote" state, as it is the default and most prominent state which enables location-transparency.

By default, any call on a ("potentially remote") distributed actor must be assumed to be crossing network boundaries. Thus, the type system pessimistically applies implicit throwing and async effects to such call-sites:

```swift
func test(actor: Greeter) async throws { 
  try await actor.greet(name: "Asa") // ✅ call could be remote
}
```

In special circumstances, a reference may be "known to be local", even without introducing a special "local" keyword in the language this manifests itself for example in closures which capture `self`. For example, we may capture `self` in a detached task, meaning that the task's closure will be executing on some different execution context than the actor itself -- and thus `self` is *not* isolated, however we *know* that it definitely is local, because there is no way we could ever refer to `self` from a remote actor:

```swift
distributed actor Closer { 
  distributed func check() -> Bool { true }

  func test() {
    Task.detached { 
      await self.check() // ✅ call is definitely local, but it must be asynchronous
    }
  }
}
```

In the above situation, we know for sure that the `self.check()` will not be crossing any process boundaries, and therefore there cannot be any implicit errors emitted by the underlying distributed actor system transport. This manifests in the type-system by the `distributed func` call not being throwing (!), however it remains asynchronous because of the usual local-only actor isolation rules.

The last case is `isolated` distributed actor references. This is relatively simple, because it just reverts all isolation checking to the local-only model. Instance members of actors are effectively methods which take an `isolated Self`, and in the same way functions which accept an `isolated Some(Distributed)Actor` are considered to be isolated to that actor. For the purpose of distributed actor isolation checking it effectively means there are no distributed checks at all, and we can even access stored properties synchronously on such reference:

```swift
distributed actor Namer { 
  let baseName: String = ...
}

func bad(n: Namer) {
  n.baseName // ❌ error, as expected we cannot access the distributed actor-isolated state
}

func good(n: isolated Namer) {
  n.baseName // ✅ ok; we are isolated to the specific 'n' Namer instance
}
```

### Distributed Actor Properties

#### Stored properties

Distributed actors may declare any kind of stored property, and the declarations themselves are *not restricted in any way*. This is important and allows distributed actors to store any kind of state, even if it were not serializable. Access to such state from the outside though is only allowed through distributed functions, meaning that cross-network access to such non-serializable state must either be fully encapsulated or "packaged up" into some serializable format that leans itself to transporting across the network. 

One typical example of this is a distributed actor storing a live database connection, and being unable to send this connection across to other nodes, it should send the results of querying the database to its callers. This is a very natural way to think about actor storage, and will even be possible to enforce at compile time, which we'll discuss in follow-up proposals discussing serialization and runtime aspects of distributed actor messages.

To re-state the rule once again more concisely: It is not possible to reach a distributed actors stored properties cross-actor. This is because stored properties may be located on a remote host, and we do not want to subject them to the same implicit effects, and serialization type-checking as distributed methods.

```swift
distributed actor Properties { 
  let fullName: String
  var age: Int
}
```

Trying to access those properties results in isolation errors at compile time:

```swift
Properties().fullName 
// ❌ error: distributed actor-isolated property 'fullName' can only be referenced inside the distributed actor
Properties().age 
// ❌ error: distributed actor-isolated property 'age' can only be referenced inside the distributed actor
```

Unlike with local-only actors, it is *not* allowed to declare `nonisolated` *stored properties*, because a nonisolated stored property implies the ability to access it without any synchronization, and would force the remote "proxy" instance to have such stored property declared and initialized, however there is no meaningful good way to initialize such variable, because a remote reference is _only_ the actor's identity and associated transport (which will be explored in more depth in a separate proposal):

```swift
distributed actor Properties { 
  nonisolated let fullName: String // ❌ error: distributed actor cannot declare nonisolated stored properties
}
```

It is allowed to declare static properties on distributed actors, and they are not isolated to the actor. This is the same as static properties on local-only actors. 

```swift
distributed actor Worker { 
  static let MAX_ITEMS: Int = 12 // ⚠️ static properties always refer to the value in the *local process*
  var workingOnItems: Int = 0
  
  distributed func work(on item: Item) throws { 
    guard workingOnItems < Self.MAX_ITEMS else {
      throw TooMuchWork(max: Self.MAX_ITEMS)
    }
    
    workingonItems += 1
  }
}
```

Be aware though that any such `static` property on a `distributed actor` always refers to whatever the property was initialized with _locally_ (in the current process). i.e. if the remote node is running a different version of the software, it may have the `MAX_ITEMS` value set to something different. So keep this in mind when debugging code while rolling out new versions across a cluster. Static properties are useful for things like constants, so feel free to use them in the same manner as you would with local-only actors.

It is permitted, same as with local-only actors, to declare `static` methods and even `static` variables on distributed actors, although please be advised that currently static variables are equally thread-*unsafe* as global properties and Swift Concurrency currently does not perform any checks on those. 

```swift
// Currently allowed in Swift 5.x, but dangerous (for now)
[distributed] actor Glass {
  var contents: String = Glass.defaultContents
  
  static var defaultContents: String { "water" } // ⚠️ not protected from data-races in Swift 5.x
}
```

As such, please be very careful with such mutable declarations. Swift Concurrency will eventually also check for shared global and static state, and devise a model preventing races in such declarations as well. Static properties declared on distributed actors will be subject to the same checks as any other static properties or globals once this has been proposed and implemented (via a separate Swift Evolution proposal).

#### Computed properties

Distributed _computed properties_ are possible to support in a very limited fashion because of the effectful nature of the distributed keyword. It is only possible to make *read-only* properties distributed, because only such properties may be effectful (as introduced by [SE-0310: Effectful Read-only Properties](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0310-effectful-readonly-properties.md)). 

```swift
distributed actor Chunk { 
  let chunk: NotSerializableDataChunk
  
  distributed var size: Int { self.chunk.size }
}
```

A distributed computed property is similar to a method accepting zero arguments, and returning a value. 

Distributed computed properties are subject to the same isolation rules, and implicit async and throwing effects. As such, accessing such variable (even across the network) is fairly explicitly telling the developer something is going on here, and they should re-consider if e.g. doing this in a loop truly is a good idea:

```swift
var i = 0
while i < (try await chunk.size) { // very bad idea, don't do this
  // logic here
  i += 1
}

// better, only check the size once:
var i = 0
let max = try await chunk.size // implicitly 'async throws', same as distributed methods
while i < max {
  // logic here
  i += 1
}
```

Because distributed methods and properties are statically known, we could envision IDEs giving explicit warnings, and even do some introspection and analysis detecting such patterns if they really wanted to. 

Any value returned by such computed property needs to be able to be serialized, similarly to distributed method parameters and return values, and would be subject to the same checks.

It is not possible to declare read/write computed properties, because of underlying limitations of effectful properties.

### Protocol Conformances

Distributed actors can conform to protocols in the same manner as local-only actors can.

As calls "through" protocols are always cross-actor, requirements that are possible to witness by a `distributed actor` must be `async throws`. The following protocol shows a few examples of protocol requirements, and whether they are possible to witness using a distributed actor's distributed function:

```swift
protocol Example { 
  func synchronous()
  func justAsync() async -> Int
  func justThrows() throws -> Int
  func asyncThrows() async throws -> String
}
```

We can attempt to conform to this protocol using a distributed actor:

```swift
distributed actor ExampleActor: Example { 
  distributed func synchronous() {} 
  // ❌ error: actor-isolated instance method 'synchronous()' cannot be used to satisfy a protocol requirement
    // cross-actor calls to 'justThrows()' are 'async throws' yet protocol requirement is synchronous
  
  distributed func justAsync() async -> Int { 2 }
  // ❌ error: actor-isolated instance method 'justAsync()' cannot be used to satisfy a protocol requirement
  // cross-actor calls to 'justAsync()' are 'async throws' yet protocol requirement is only 'async'
  
  distributed func justThrows() throws -> Int { 2 }
  // ❌ error: actor-isolated instance method 'justThrows()' cannot be used to satisfy a protocol requirement}}
  // cross-actor calls to 'justThrows()' are 'async throws' yet protocol requirement is only 'throws'
  
  distributed func asyncThrows() async throws -> String { "two" } // ✅
}
```

Let us focus on the last example, `asyncThrows()` which is declared as a throwing and asynchronous protocol requirement, and returns a `String`. We are able to witness this requirement, but we should mention the future direction of compile time serialization checking while discussing this function as well. 

If we recall the previously mentioned serialization conformance checking mechanism, we could imagine that the `ExampleActor` configured itself to use e.g. `Codable` for its message serialization. This means that the method declarations are subject to `Codable` checking:

```swift
distributed actor CodableExampleActor: Example { 
  typealias SerializationRequirement = Codable
  
  distributed func asyncThrows() async throws -> String { "two" } // ✅ ok, String is Codable
}
```

As we can see, we were still able to successfully witness the `asyncThrows` protocol requirement, since the signature matches our serialization requirement. This allows us to conform to existing protocol requirements with distributed actors, without having to invent complicated wrappers.

If we used a different serialization mechanism, we may have to provide a `nonisolated` witness, that converts the types expected by the protocol, to whichever types we are able to serialize (e.g. protocol buffer messages, or anything else, including custom serialization formats). Either way, we are able to work our way through and conform to protocols if necessary.

It is possible to utilize `nonisolated` functions to conform to synchronous protocol requirements, however those have limited use in practice on distributed actors since they cannot access any isolated state. In practice such functions are implementable by accessing the actor's identity or actor system it belongs to, but not much else.

```swift
protocol CustomStringConvertible {
  var description: String { get }
}

distributed actor Example: CustomStringConvertible { 
  nonisolated var description: String { 
    "distributed actor Example: \(self.identity)"
  }
}
```

The above example conforms a distributed actor to the well-known `CustomStringConvertible` protocol, and we can use similar techniques to implement protocols like `Hashable`, `Identifiable`, and even `Codable`. We will discuss these in the following proposals about distributed actor runtime details though.

#### The `DistributedActor` protocol and protocols inheriting from it

This proposal mentioned the `DistributedActor` protocol a few times, however without going into much more depth about its design. We will leave this to the *actor runtime* focused proposals, however in regard to isolation we would like do discuss its relation to protocols and protocol conformances:

The `DistributedActor` protocol cannot be conformed to explicitly by any other type other than a `distributed actor` declaration. This is similar to the `Actor` protocol and `actor` declarations.

It is possible however to express protocols that inherit from the `DistributedActor` protocol, like this:

```swift
protocol Worker: DistributedActor { 
  distributed func work(on: Item) -> Int
  
  nonisolated func same(as other: Worker) -> Bool
  
  static func isHardWorking(_ worker: Worker) -> Bool
}
```

Methods definitions inside distributed actor inheriting protocols must be declared either:`distributed`, `static`or `nonisolated`. Again, we value the explicitness of the definitions, and the compiler will guide and help you decide how the method shall be isolated.

Note that it is always possible to conform to a distributed protocol requirement with a witness with "more" effects, since the cross-actor API remains the same - thanks to the implicit effects caused by the distributed keyword.

```swift
protocol Arnold: Worker { 
  distributed func work(on: Item) async -> Int {
    // turns out we need this to be async internally, this is okay
  }
}
```

This witness works properly, because the `distributed func` requirement in the protocol is always going to be `async throws` due to the `distributed func`'s effect on the declaration. Therefore the declaration "inside the actor" can make use of `async` or `throws` without changing how the protocol can be used.

### Breaking through Location Transparency

Programs based on distributed actors should always be written to respect location transparency, but sometimes it is useful to break through that abstraction. The most common situation where breaking through location transparency can be useful is when writing unit tests. Such tests may need to inspect state, or call non-distributed methods, of a distributed actor instance that is known to be local.

To support this kind of niche circumstance, all distributed actors offer a `whenLocal` method, which executes a provided closure based on whether it is a local instance:

```swift
extension DistributedActor {
  /// Runs the 'body' closure if and only if the passed 'actor' is a local instance.
  /// 
  /// Returns `nil` if the actor was remote.
  @discardableResult
  nonisolated func whenLocal<T>(
    _ body: (isolated Self) async throws -> T
  ) async rethrows -> T?

  /// Runs the 'body' closure if and only if the passed 'actor' is a local instance.
  /// 
  /// Invokes the 'else' closure if the actor instance was remote.
  @discardableResult
  nonisolated func whenLocal<T>(
    _ body: (isolated Self) async throws -> T,  
    else whenRemote: (Self) async throws -> T
  ) async rethrows -> T 
```

When the instance is local, the `whenLocal` method exposes the distributed actor instance to the provided closure, as if it were a regular actor instance. This means you can invoke non-distributed methods when the actor instance is local, without relying on hacks that would trigger a crash if invoked on a remote instance.

> **Note:** We would like to explore a slightly different shape of the `whenLocal` functions, that would allow _not_ hopping to the actor unless necessary, however we are currently lacking the implementation ability to do so. So this proposal for now shows the simple, `isolated` based approach. The alternate API we are considering would have the following shape:
>
> ```swift
> @discardableResult
> nonisolated func whenLocal<T>(
> _ body: (local Self) async throws -> T
> ) reasync rethrows -> T?
> ```
>
> This API could enable us to treat such `local DistActor` exactly the same as a local-only actor type; We could even consider allowing nonisolated stored properties, and allow accessing them synchronously like that:
>
> ```swift
> // NOT part of this proposal, but a potential future direction
> distributed actor FamousActor { 
> let name: String = "Emma"
> }
> 
> FamousActor().whenLocal { fa /*: local FamousActor*/ in
> fa.name // OK, known to be local, distributed-isolation does not apply
> }
> ```

## Future Directions

### Versioning and Evolution of Distributed Actors and Methods

Versioning and evolution of exposed `distributed` functionality is a very important, and quite vast topic to tackle. This proposal by itself does not include new capabilities - we are aware this might be limiting adoption in certain use-cases. 

#### Evolution of parameter values only

In today's proposal, it is possible to evolve data models *inside* parameters passed through distributed method calls. This completely relies on the serialization mechanism used for the individual parameters. Most frequently, we expect Codable, or some similar mechanism, to be used here and this evolution of those values relies entirely on what the underlying encoders/decoders can do. As an example, we can define a `Message` struct like this:

```swift
struct Message: Codable { 
  let oldVersion: String
  let onlyInNewVersion: String
}

distributed func accept(_: Message) { ... }
```

and the usual backwards / forwards evolution techniques used with `Codable` can be applied here. Most coders are able to easily ignore new unrecognized fields when decoding. It is also possible to improve or implement a different decoder that would also store unrecognized fields in some other container, e.g. like this:

```swift
struct Message: Codable { 
  let oldVersion: String
  let unknownFields: [String: ...] 
}

JSONDecoderAwareOfUnknownFields().decode(Message.self, from: ...)
```

and the decoder could populate the `unknownFields` if necessary. There are various techniques to perform schema evolution here, and we won't be explaining them in more depth here. We are aware of limitations and challenges related to `Codable` and might revisit it for improvements. 

#### Evolution of distributed methods

The above-mentioned techniques apply only for the parameter values themselves though. With distributed methods we need to also take care of the method signatures being versioned, this is because when we declare

```swift
distributed actor Greeter { 
  distributed func greet(name: String)
}
```

we exposed the ability to invoke `greet(name:)` to other peers. Such normal, non-generic signature will *not* cause the transmission of `String`, over the wire. They may be attempting to invoke this method, even as we roll out a new version of the "greeter server" which now has a new signature:

```swift
distributed actor Greeter { 
  distributed func greet(name: String, in language: Language)
}
```

This is a breaking change as much in API/ABI and of course also a break in the declared wire protocol (message) that the actor is willing to accept. 

Today, Swift does not have great facilities to move between such definitions without manually having to keep around the forwarder methods, so we'd do the following:

```swift
distributed actor Greeter { 
  
  @available(*, deprecated, renamed: "greet(name:in:)")
  distributed func greet(name: String) {
    self.greet(name: name, in: .defaultLanguage)
  }
  
  distributed func greet(name: String, in language: Language) {
    print("\(language.greeting), name!")
  }
}
```

This manual pattern is used frequently today for plain old ABI-compatible library evolution, however is fairly manual and increasingly annoying to use as more and more APIs become deprecated and parameters are added. It also means we are unable to use Swift's default argument values, and have to manually provide the default values at call-sites instead.

Instead, we are interested in extending the `@available` annotation's capabilities to be able to apply to method arguments, like this:

```swift
distributed func greet(
  name: String,
  @available(macOS 12.1, *) in language: Language = .defaultLanguage) {
    print("\(language.greeting), name!")
}

// compiler synthesized:
// // "Old" API, delegating to `greet(name:in:)`
// distributed func greet(name: String) {
//   self.greet(name: name, in: .defaultLanguage)
// }
```

This functionality would address both ABI stable library development, and `distributed` method evolution, because effectively they share the same concern -- the need to introduce new parameters, without breaking old API. For distributed methods specifically, this would cause the emission of metadata and thunks, such that the method `greet(name:)` can be resolved from an incoming message from an "old" peer, while the actual local invocation is performed on `greet(name:in:)`.

Similar to many other runtimes, removing parameters is not going to be supported, however we could look into automatically handling optional parameters, defaulting them to `nil` if not present incoming messages.

In order to serve distribution well, we might have to extend what notion of "platform" is allowed in the available annotation, because these may not necessarily be specific to "OS versions" but rather "version of the distributed system cluster", which can be simply sem-ver numbers that are known to the cluster runtime:

```swift
distributed func greet(
  name: String,
  @available(distributed(cluster) 1.2.3, *) in language: Language = .defaultLanguage) {
    print("\(language.greeting), name!")
}
```

During the initial handshake peers in a distributed system exchange information about their runtime version, and this can be used to inform method lookups, or even reject "too old" clients. 

## Introducing the `local` keyword

It would be possible to expand the way distributed actors can conform to protocols which are intended only for the actor's "local side" if we introduced a `local` keyword. It would be used to taint distributed actor variables as well as functions in protocols with a local bias.

For example, `local` marked distributed actor variables could simplify the following (surprisingly common in some situations!) pattern:

```swift
distributed actor GameHost { 
  let myself: local Player
  let others: [Player]
  
  init(system: GameSystem) {
    self.myself = Player(system: GameSystem)
    self.others = []
  }
  
  distributed func playerJoined(_ player: Player) { 
    others.append(player)
    if others.count >= 2 { // we need 2 other players to start a game
      self.start()
    }
  }
  
  func start() {
    // start the game somehow, inform the local and all remote players
    // ... 
    // Since we know `myself` is local, we can send it a closure with some logic 
    // (or other non-serializable data, like a connection etc), without having to use the whenLocal trick.
    myself.onReceiveMessage { ... game logic here ... }
  }
}
```

The above example makes use of the `myself: local Player` stored property, which propagates the knowledge that the player instance stored in this property *definitely* is local, and therefore we can call non-distributed methods on it, which is useful when we need to pass it closures or other non-serializable state -- as we do in the `start()` method.

An `isolated Player` where Player is a `distributed actor` would also automatically be known to be `local`, and the `whenLocal` function could be expressed more efficiently (without needing to hop to the target actor at all):

```swift
// WITHOUT `local`:
// extension DistributedActor {
//   public nonisolated func whenLocal<T>(_ body: @Sendable (isolated Self) async throws -> T)
//     async rethrows -> T? where T: Sendable

// WITH local, we're able to not "hop" when not necessary:
extension DistributedActor {
  public nonisolated func whenLocal<T>(_ body: @Sendable (local Self) async throws -> T)
    reasync rethrows -> T? where T: Sendable // note the reasync (!)
}
```

This version of the `whenLocal` API is more powerful, since it would allow accessing actor state without hops, if we extended the model to allow this. This would allow treating `local AnyDistributedActor` the same way as we treat any local-only actor, and can be very useful in testing. 

We would not have to wrap APIs in `whenLocal` or provide wrapper APIs that are `nonisolated` but actually invoke things on self, like this real problem example, from implementing a Cluster "receptionist" actor where certain calls shall only be made by the "local side", however the entire actor is accessible remotely for other peers to communicate with:

```swift
distributed actor Receptionist { 
  distributed func receiveGossip(...) { ... }
  
  // only to be invoked by "local" actors
  func registerLocalActor<Act>(actor: Act) where Act: DistributedActor { ... }
}
```

Since it is too annoying to tell end-users to "always use `whenLocal` to invoke the local receptionist", library developers are forced to provide the following wrapper:

```swift
extension Receptionist { 
  
  // annoying forwarder/wrapper func; potentially unsafe, intended only for local use.
  nonisolated func register<Act>(actor: Act) async where Act: DistributedActor { 
    await self.whenLocal { myself in 
      myself.registerLocalActor(actor: actor)
    } else: {
			fatalError("\(#function) must only be called on the local receptionist!")
    }
  }
}

// ------------------------------------
final class System: DistributedActorSystem {
  // ...
  let receptionist: Receptionist
}

distributed actor Worker { 
  init(system: System) async { 
    receptionist.register(self) // ✅ OK
  }
}
```

This mostly works, but the implementation of the `nonisolated func register` leaves much to be desired. Rather, we want to express the following:

```swift
final class System: DistributedActorSystem {
  // ...
  let receptionist: local Receptionist
}

distributed actor Worker { 
  init(system: System) async { 
    await receptionist.registerLocalActor(self) // ✅ OK
  }
}
```

Without the need of manually implementing the "discard the distributed nature" of such actors.

We see this as a natural follow up and future direction, which may take a while to implement, but would vastly improve the ergonomics of distributed actors in those special yet common enough few cases where such actors make an appearance.

## Alternatives Considered

This section summarizes various points in the design space for this proposal that have been considered, but ultimately rejected from this proposal.

### Implicitly `distributed` methods / "opt-out of distribution"

After initial feedback that `distributed func` seems to be "noisy", we actively explored the idea of alternative approaches which would reduce this perceived noise. We are convinced that implicitly distributed functions are a bad idea for the overall design, understandability, footprint and auditability of systems expressed using distributed actors.

A promising idea, described by Pavel Yaskevich in the [Pitch #1](https://forums.swift.org/t/pitch-distributed-actors/51669/129) thread, was to inverse the rule, and say that _all_ functions declared on distributed actors are `distributed` by default (except `private` functions), and introduce a `local` keyword to opt-out from the distributed nature of actors. This listing exemplifies the idea:

```swift
distributed actor Worker { 
  func work(on: Item) {} // "implicitly distributed"
  private func actualWork() {} // not distributed
  
  local func shouldWork(on item: Item) -> Bool { ... } // NOT distributed
}
```

However, this turns out to complicate the understanding of such a system rather than simplify it. 

[1] We performed an analysis of a real distributed actor runtime (that we [open sourced recently](https://swift.org/blog/distributed-actors/)), and noticed that complex distributed actors have by far more non-distributed functions, than distributed ones. It is typical for a single distributed function, to invoke multiple non distributed functions in the same actor - simply because good programming style causes the splitting out of small pieces of logic into small functions with good names; Special care would have to be taken to mark those methods local. It is easy to forget doing so, since it is not a natural concept anywhere else in Swift to have to mark things "local" -- everything else is local after all.

For example, the [distributed actor cluster implementation](https://github.com/apple/swift-distributed-actors) has a few very complex actors, and their sizes are more or less as follows:

- ClusterShell - a very complex actor, orchestrating node connections etc.
  - 14 distributed methods (it's a very large and crucial actor for the actor system)
  - ~25 local methods
- SWIMShell, thee actor orchestrating the SWIM failure detection mechanism,
  - 5 distributed methods
  - 1 public local-only methods used by local callers
  - ~12 local methods

- ClusterReceptionist, responsible for discovering and gossiping information about actors
  - 2 distributed methods
  - 3 public local-only methods
  - ~30 internal and private methods (lots of small helpers)

- NodeDeathWatcher, responsible for monitoring node downing, and issuing associated actor termination events,
  - 5 distributed functions
  - no local-only methods

[2] We are concerned about the auditability and review-ability of implicit distributed methods. In a plain text review it is not possible to determine whether the following introduces a distributed entry point or not. Consider the following diff, that one might be reviewing when another teammate submits a pull request:

```swift
+ extension Worker { 
+   func runShell(cmd: String) { // did this add a remotely invocable endpoint? we don't know from this patch!
+     // execute in shell
+   }
+ }
```

Under implicit `distributed func` rules, it is impossible to know if this function is possible to be invoked remotely. And if it were so, it could be a potential exploitation vector. Of course transports do and will implement their own authentication and authorization mechanisms, however nevertheless the inability to know if we just added a remotely invokable endpoint is worrying.

In order to know if we just introduced a scary security hole in our system, we would have to go to the `Worker` definition and check if it was an `actor` or `distributed actor`.

The accidental exposing can have other, unintended, side effects such as the following declaration of a method which is intended only for the actor itself to invoke it when some timer tick is triggered:

```swift
// inside some distributed actor
func onPeriodicAckTick() { ... }
```

The method is not declared `private`, because in tests we want to be able to trigger the ticks manually. Under the implicit `distributeed func` rule, we would have to remember to make it local, as otherwise we accidentally made a function that is only intended for our own timers as remotely invocable, which could be misunderstood and/or be abused by either mistake, or malicious callers. 

Effectively, the implicitly-distributed rule causes more cognitive overhead to developers, every time having to mark and think about local only functions, rather than only think about the few times they actively want to _expose_ methods.

[3] We initially thought we could delay additional type checks of implicit distributed functions until their first use. This would be similar to `Sendable` checking, where one can define a function accepting not-Sendable values, and only once it is attempted to be used in a cross-actor situation, we get compile errors.

With distribution this poses a problem though: For example, should we allow the following conformance:

```swift
struct Item {} // NOT Codable

protocol Builder { 
  func build(_: Item) async throws
}

distributed actor Bob: Builder {
  typealias SerializationRequirement = Codable
  func build(_: Item) async throws { ... }
}
```

Under implicit distributed rules, we should treat this function as distributed, however that means we should be checking `Item` for the `Codable` conformance. We know at declaration time that this conformance is faulty. While in theory we could delay the error until someone actually invoked the build function:

``` swift
let bob: Bob
try await bob.build(Item()) // ❌ error: parameter type 'Item' does not conform to 'Bob.SerializationRequirement'
```

so we have declared a method that is impossible to invoke... however if we attempted to erase `Bob` to `Builder`...

```swift
let builder: Builder = bob
try await builder.build(Item())
```

there is nothing preventing this call from happening. There is no good way for the runtime to handle this; We would have to invent some defensive throwing modes, throwing in the distributed remote thunk, if the passed parameters do not pass what the type-system should have prevented from happening. 

In other words, the Sendable-like conformance model invites problematic cases which may lead to unsoundness.

Thus, the only type-checking model of distributed functions, implicit or not, is an eager one. Where we fail during type checking immediately as we see the illegal declaration:

```swift
struct Item {} // NOT Codable

protocol Builder { 
  func build(_: Item) async throws
}

distributed actor Bob: Builder {
  typealias SerializationRequirement = Codable
  func build(_: Item) async throws { ... } 
  // ❌ error: function 'build(_:)' cannot be used to satisfy protocol requirement
  // ❌ error: parameter type 'Item' does not conform to 'Bob.SerializationRequirement'
}
```

By itself this is fine, however this has a painful effect on common programming patterns in Swift, where we are encouraged to extract small meaningful functions that are re-used in places by the actor. We are forced to annotate _more_ APIs as `local` than we would have been with the _explicit_ `distributed` annotation model (see observation that real world distributed actors often have many small functions, not intended for distribution)

[4] Since almost all functions are distributed by default in the implicit model, we need to create and store metadata for all of them, regardless if they are used or not. This may cause unnecessary binary size growth, and seems somewhat backwards to Swift's approach to be efficient and minimal in metadata produced. 

We are aware of runtimes where every byte counts, and would not want to prevent them from adopting distributed actors for fear of causing accidental binary size growth. In practice, we would force developers to always write `local func` unless proven that it needs to be distributed, then removing the keyword – this model feels backwards from the explicit distributed marking model, in which we make a conscious decision that "yes, this function is intended for distribution" and mark it as `distributed func` only once we actively need to.

[5] While it may seem simplistic, an effective method for auditing a distributed "attack surface" of a distributed actor system is enabled by the ability search the codebase for `distributed func` and make sure all functions perform the expected authorization checks. These functions are as important as "service endpoints" and should be treated with extra care. This only works when distributed functions are explicit.

We should also invest in transport-level authentication and authorization techniques, however some actions are going to be checked action-per-action, so this additional help of quickly locating distributed functions is a feature, not an annoyance.

Summing up, the primary benefit of the implicit `distributed func` rule was to attempt to save developers a few keystrokes, however it fails to deliver this in practice because frequently (verified by empirical data) actors have many local methods which they do not want to expose as well. The implicit rule makes these more verbose, and results in more additional annotations. Not only that, but it causes greater mental overhead for having to remember if we're in the context of a distributed actor, and if a `func` didn't just accidentally get exposed as remotely accessible endpoint. We also noticed a few soundness and additional complexity in regard to protocol conformances that we found quite tricky.

We gave this alternative design idea significant thought and strongly favor the explicit distributed rule.

### Declaring actors and methods as "`distributable`"

Naming of distributed actors has been debated and while it is true that `distributed` means "may be distributed (meaning 'remote') or not", this is not really the mindset we want to promote with distributed actors. The mental mindset should be that these are distributed and we must treat them this way, and they may happen to be local. Locality is the special case, distribution is the capability we're working with while designing location transparent actors. While we do envision the use of "known to be local" distributed actors, this is better solved with either a `worker.whenLocal { ...` API or allowing marking types with a `local` keyword - either approaches are not part of this proposal and will be pitched in dependently.

The `distributed` keyword functions the same way as `async` on methods. Async methods are not always asynchronous. The `async` keyword merely means that such method _may suspend_. Similarly, a `distributed func` may or may not perform a remote call, as such the semantics follow the same "beware, the more expensive thing may happen" style of marking methods. 

### Unconditionally conforming `DistributedActor` to `Codable`

This was part of an earlier design, where the distributed actor protocol was declared as:

```swift
protocol DistributedActor: AnyActor, Sendable, Codable, ... { ... }
```

forcing all implementations of distributed actors to implement the Codable `init(from:)` initializer and `encode(to:)` method. 

While we indeed to expect `Codable` to play a large role in some distributed actor implementations, we have specific use-cases in mind where:

- Codable might not be used _at all_, thus the re-design and strong focus on being serialization mechanism agnostic in the proposal, by introducing the `SerializationRequirement` associated type.
- Some distributed actor runtimes may behave more like "services" which are _not_ meant to be "passed around" to other nodes. This capability has been explicitly requested by some early adopters in IPC scenarios, where it will help to clean up vague and hacky solutions today, with a clear model where some distributed actors are Codable and thus "pass-around-able" and some are not, depending on the specifics how they were created.

As such, we are left with no other implementation approach other than the implicit conformance, because it is not possible to add the `Codable` conformance to types managed by a distributed actor system that _wants to_ make distributed actors Codable otherwise (i.e. it is not possible to express `extension DistributedActor: Codable where ID: Codable {}` in today's Swift). Alternative approaches force implementations into casting and doing unsafe tricksy and lose out on the type-safety of only passing Codable actors to distributed methods.

For distributed actor systems which _do not_ use `Codable`, forcing them to implement Codable methods and initializers would be quite a problem and the implementations would likely be implemented as just crashing. Implementations may force actors to conform to some other protocol, like `IPCServiceDistributedActor` which conforms to the `SerializationRequirement` and attempts to initialize an actor which does not conform to this protocol can crash eagerly, at initialization time. This way actor system authors gain the same developer experience as using `Codable` for passing distributed actors through distributed methods, but the initialization can be specialized -- as it is intended to, because libraries may require specific things from actor types after all.

### Introducing "wrapper" type for `Distributed<SomeActor>`

We did consider (and have implemented, assisted by swift-syntax based source-generation) the idea of wrapping distributed actors using some "wrapper" type, that would delegate calls to all distributed functions, but prevent access to e.g. stored properties wrapped by such instance. 

This loses the benefit that a proper nominal type distributed actor offers though: the easy to incrementally move actors to distribution as it becomes necessary. The complexity of forming the "call forwarding" functions is also problematic, and extensions to such types would be confusing, would we have to do extensions like this?

```swift
extension Distributed where Actor == SomeActor { 
  func hi() { ... }
}
```

while _also_ forwarding to functions extended on the `SomeActor` itself?

```swift
extension SomeActor  {
  func hi() { ... } // conflict?
}
```

What would that mean for when we try to call `hi()` on a distributed actor? It also does not really simplify testing, as we want to test the actual actor, but also the distributed functions actually working correctly (i.e. enforcing serialization constraints on parameters).

### Creating only a library and/or source-generation tool

While this may be a highly subjective and sensitive topic, we want to tackle the question up-front, so why are distributed actors better than "just" some RPC library?

The answer lies in the language integration and the mental model developers can work with when working with distributed actors. Swift already embraces actors for its local concurrency programming, and they will be omni-present and become a familiar and useful tool for developers. It is also important to notice that any async function may be technically performing work over network, and it is up to developers to manage such calls in order to not overwhelm the network etc. With distributed actors, such calls are more _visible_ because IDEs have the necessary information to e.g. underline or otherwise highlight that a function is likely to hit the network and one may need to consider its latency more, than if it was just a local call. IDEs and linters can even use this statically available information to write hints such as "hey, you're doing this distributed actor call in a tight loop - are you sure you want to do that?"

Distributed actors, unlike "raw" RPC frameworks, help developers think about their distributed applications in terms of a network of collaborating actors, rather than having to think and carefully manage every single serialization call and network connection management between many connected peers - which we envision to be more and more important in the future of device and server programming et al. You may also refer to the [Swift Concurrency Manifesto; Part 4: Improving system architecture](https://gist.github.com/lattner/31ed37682ef1576b16bca1432ea9f782#part-4-improving-system-architecture) section for some other ideas on the topic.

This does _not_ mean that we shun RPC style libraries or plain-old HTTP clients and libraries similar to them, which may rather be expressed as non-actor types with asynchronous functions. They still absolutely have their place, and we do not envision distributed actors fully replacing them - they are fantastic for cross-language communication, however distributed actors offer a vastly superior programming model, while we remain mostly within Swift and associated actor implementations (we *could*, communicate with non-swift actors over the network, however have not invested into this yet). We do mean however that extending the actor model to its natural habitat (networking) will enable developers to build some kinds of interactive multi-peer/multi-node systems far more naturally than each time having to re-invent a similar abstraction layer, never quite reaching the integration smoothness as language provided integration points such as distributed actors can offer.

## Acknowledgments & Prior Art

We would like to acknowledge the prior art in the space of distributed actor systems which have inspired our design and thinking over the years. Most notably we would like to thank the Akka and Orleans projects, each showing independent innovation in their respective ecosystems and implementation approaches. As these are library-only solutions, they have to rely on wrapper types to perform the hiding of information, and/or source generation; we achieve the same goal by expanding the already present in Swift actor-isolation checking mechanisms.

We would also like to acknowledge the Erlang BEAM runtime and Elixir language for a more modern take built upon the on the same foundations, which have greatly inspired our design, however take a very different approach to actor isolation (i.e. complete isolation, including separate heaps for actors).

## Source compatibility

This change is purely additive to the source language. 

The additional use of the keyword `distributed` in `distributed actor` and `distributed func` applies more restrictive requirements to the use of such an actor, however this only applies to new code, as such no existing code is impacted.

Marking an actor as distributed when it previously was not is potentially source-breaking, as it adds additional type checking requirements to the type.

## Effect on ABI stability

None.

## Effect on API resilience

None.

## Changelog

- 1.3.1 Minor cleanups
  - Allow `private distributed func`
  - Allow generic distributed actor declarations
- 1.3 More about serialization typechecking and introducing mentioned protocols explicitly 
  - Revisions Introduce `DistributedActor` and `DistributedActorSystem` protocols properly
  - Discuss future directions for versioning and evolving APIs
  - Introduce conditional Codable conformance of distributed actors, based on ID
  - Discuss `SerializationRequirement` driven typechecking of distributed methods
  - Discuss `DistributedActorSystem` parameter requirement in required initializers
  - Discuss isolation states in depth "isolated", "known to be local", "potentially remote" and their effect on implicit effects on call-sites
- 1.2 Drop implicitly distributed methods
- 1.1 Implicitly distributed methods
- 1.0 Initial revision
- [Pitch: Distributed Actors](https://forums.swift.org/t/pitch-distributed-actors/51669)
  - Which focused on the general concept of distributed actors, and will from here on be cut up in smaller, reviewable pieces that will become their own independent proposals; Similar to how Swift Concurrency is a single coherent feature, however was introduced throughout many interconnected Swift Evolution proposals.
