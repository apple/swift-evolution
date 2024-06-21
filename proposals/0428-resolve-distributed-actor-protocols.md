# Resolve DistributedActor protocols

* Proposal: [SE-0428](0428-resolve-distributed-actor-protocols.md)
* Author: [Konrad 'ktoso' Malawski](https://github.com/ktoso), [Pavel Yaskevich](https://github.com/xedin)
* Review Manager: [Freddy Kellison-Linn](https://github.com/Jumhyn)
* Status:  **Implemented (Swift 6.0)**
* Review: ([pitch](https://forums.swift.org/t/pitch-resolve-distributedactor-protocols-for-server-client-apps/69933)) ([review](https://forums.swift.org/t/se-0428-resolve-distributedactor-protocols/70669)) ([acceptance](https://forums.swift.org/t/accepted-with-modifications-se-0428-resolve-distributedactor-protocols/71366))

## Introduction

Swift's distributed actors offer developers a flexible bring-your-own-runtime approach to building distributed systems using the actor paradigm. The initial design of the feature aimed for systems where all nodes of a distributed actor system (such as nodes in a [cluster](https://github.com/apple/swift-distributed-actors)) share the same binary, and therefore all have access to the concrete `distributed actor` declarations which may be resolved and made remote calls on.

Although this works well for peer-to-peer systems, distributed actors are also useful for systems where a client/server split is necessary. Examples of such use-cases include isolating some failure prone logic into another process, or split between a client without the necessary libraries or knowledge how to implement an API and delegating this work to a backend service.

## Motivation

Distributed actors allow to abstract over the location (i.e. in process or not) of an actor -- often referred to as "location transparency". Currently, Swift's distributed actors have a practical limitation in how well this property can be applied to server/client split applications because the only way to obtain a *remote reference* on which a *remote call* can be made, is by invoking the resolve method on a concrete distributed actor type, like this:

```swift
import Distributed
import DistributedCluster // github.com/apple/swift-distributed-actors

protocol Greeter: DistributedActor {
  distributed func greet(name: String) -> String
}

distributed actor EnglishGreeter: Greeter {
  typealias ActorSystem = ClusterSystem
  
  func greet(name: String) -> String {
    "Hello, \(name)!"
  }
}

let system: ClusterSystem = ...

let knownID: EnglishGreeter.ID = /* obtained ID using discovery mechanisms, see ClusterSystem */
let remote: EnglishGreeter = try EnglishGreeter.resolve(id: knownID, using: system)

// remote call (implicitly async and throwing, due to e.g. network errors)
let greeting = try await remote.greet(name: "Caplin")
assert(greeting == "Hello, Caplin!")
```

This is common and acceptable in a **peer-to-peer system**, where all nodes of a cluster share the same types -- or at least a "client side" of a connection can only discover types it knows about, e.g. during version upgrade rollouts. However, this pattern is problematic in a **client/server deployment**, where the two applications do not share the concrete implementation of the `Greeter` type. It is also worth calling out that typical inter-process communications (IPC) use-cases often fall into the category of a client/server setup, where e.g. a daemon process serves as a "server" and an application "client" calls into it.

The goal of this proposal is to allow the following module approach to sharing distributed actor APIs:
- **API** module: allow sharing a `DistributedActor` constrained protocol, which only declares the API surface the server is going to expose
- **Server** module: which depends on API module, and implements the API description using a concrete distributed actor
- **Client** module: which depends on API module, and `$Greeter` type (synthesized by the `@Resolvable` macro) to resolve a remote actor reference it can then invoke distributed methods on 

```swift
                         ┌────────────────────────────────────────┐
                         │                API Module              │
                         │========================================│
                         │ @Resolvable                            │
                         │ protocol Greeter: DistributedActor {   │
                 ┌───────┤   distributed func greet(name: String) ├───────┐
                 │       │ }                                      │       │
                 │       └────────────────────────────────────────┘       │
                 │                                                        │
                 ▼                                                        ▼   
┌────────────────────────────────────────────────┐      ┌──────────────────────────────────────────────┐
│             Client Module                      │      │               Server Module                  │
│================================================│      │==============================================│
│ let g = try $Greeter.resolve(...) /*new*/      │      │ distributed actor EnglishGreeter: Greeter {  │
│ try await greeter.hello(name: ...)             │      │   distributed func greet(name: String) {     │
└────────────────────────────────────────────────┘      │     "Greeting in english, for \(name)!"      │
/* Client cannot know about EnglishGreeter type */      │   }                                          │      
                                                        │ }                                            │
                                                        └──────────────────────────────────────────────┘
```

In this scenario the client module has no knowledge of the concrete distributed actor type or implementation.

In order to achieve this, this proposal improves upon _three_ aspects of distributed actors:
- introduce the `@Resolvable` macro that can be attached to distributed actor protocols, and enable the use of `resolve(id:using:)` with such types,
- allow distributed actors to be generic over their `ActorSystem`
- extend the distributed metadata section such that a distributed method which is witness to a distributed requirement, is also recorded using the protocol method's `RemoteCallTarget.identifier` and not only the concrete method's

## Proposed solution

### The `@Resolvable` macro

At the core of this proposal is the `@Resolvable` macro. It is an attached declaration macro, which introduces a number of declarations which allow the protocol, or rather, a "stub" type for the protocol, to be used on a client without knowledge about the server implementation's concrete distributed actor type.

The macro must be attached to the a `protocol` declaration that is a `DistributedActor` constrained protocol, like this:

```swift
import Distributed 

@Resolvable
protocol Greeter where ActorSystem: DistributedActorSystem<any Codable> {
  distributed func greet(name: String) -> String
}
```

The protocol must specify a constraint on the `ActorSystem` which specifies the kind of `SerializationRequirement` it is able to work with. This serialization requirement must be a protocol, and existing distributed actor functionality already will be verifying this.

Checking of distributed functions works as before, and the compiler will check that the `distributed` declarations all fulfill the `SerializationRequirement` constraint. E.g. in the example above, the parameter type `String` and return type `String` both conform to the `Codable` protocol, so this distributed protocol is well formed.

It is possible to for a distributed actor protocol to contain non-distributed requirements. However in practice, it will be impossible to ever invoke such methods on a remote distributed actor reference. It is possible to call such methods if one were to obtain a local distributed actor reference implementing such protocol, and use the existing `whenLocal(operation:)` method on it.

The `@Resolvable` macro generates a number of internal declarations necessary for the distributed machinery to work, however the only type users should care about is always a `$`-prefixed concrete distributed actor declaration, that is the "stub" type. This stub type can be used to resolve a remote actor using this protocol stub:

E.g. if we knew a remote has a `Greeter` instance for a specific ID we have discovered using some external mechanism, this is how we'd resolve it:

```swift 
let clusterSystem: ClusterSystem // example system

let greeter = try $Greeter.resolve(id: id, using: clusterSystem)
```

As the `ClusterSystem` is using `Codable` as it's serialization requirement, the resolve compiles and produces a valid reference.

### Distributed actors generic over their `ActorSystem`

The previous section made use of a distributed actor that abstracted over a generic `ActorSystem`. Today (in Swift 5.10) this is not possible, and would result in a compile time error.

Previously, the compiler would require that the `ActorSystem` typealias refer to a specific distributed actor system type (not a protocol, a concrete nominal type). For example, the following actor can only be used with the `ClusterSystem`:

```swift
import Distributed
import DistributedCluster // github.com/apple/swift-distributed-actors provides `ClusterSystem`

distributed actor DistributedAsyncSequence<Element> where Element: Sendable & Codable { 
  typealias ActorSystem = ClusterSystem

  // not real implementation; simplified method to showcase introduced capabilities
  distributed func gimmeNextElement() async throws -> Element? { ... }
}
```

And while such generally useful "distributed async sequence" actor can be written generically, to work with the vast majority of actor systems, today's language did not allow to write such generic actor, and the `ActorSystem` type always was forced to be a _concrete_ type. 

To support this new pattern, the `DistributedActorSystem` protocol gains a *primary associated type* for the `SerializationRequirement` associated type:

```swift
// before: 
// protocol DistributedActorSystem: Sendable {
//   associatedtype SerializationRequirement where ...
//   // ...
// }

// now: 
protocol DistributedActorSystem<SerializationRequirement>: Sendable {
  /// The serialization requirement that will be applied to all distributed targets used with this system.
  associatedtype SerializationRequirement
    where SerializationRequirement == InvocationEncoder.SerializationRequirement,
          SerializationRequirement == InvocationDecoder.SerializationRequirement,
          SerializationRequirement == ResultHandler.SerializationRequirement
  // ...
}

// 
```

The `SerializationRequirement` must be specified for all actors and protocols attempting to abstract over an actor system, because it is necessary to compile-time guarantee the correctness of values passed to such distributed actor methods. The compiler uses this associated type to verify all argument types and returned values are able to be serialized when performing remote calls, and will refuse to compile invocations would otherwise would have failed at runtime.

Thanks to this new primary associated type, it is now possible to spell our `DistributedAsyncSequence` as a generic actor, implement it once, and re-use it across any compatible actor system implementation:

```swift
distributed actor DistributedAsyncSequence<Element, ActorSystem> 
  where Element: Sendable & Codable,
        ActorSystem: DistributedActorSystem<any Codable> { 
          
  // not real implementation; simplified method to showcase introduced capabilities
  distributed func exampleNextElement() async throws -> Element? { ... }
}
```

Note that since the `ActorSystem` specifies a concrete `SerializationRequirement` the compiler is still able to check that all types invoked in a distributed function call conform to this protocol, i.e. we're guaranteed to be able to serialize `Element` because the ActorSystem provided must be able to handle this serialization mechanism.

This also extends to distributed protocols, which are now able to abstract over an actor system, while specifying what serialization requirement they support:

```swift
protocol DistributedAsyncSequence: DistributedActor 
    where ActorSystem: DistributedActorSystem<any Codable> {
  associatedtype Element: Sendable & Codable
      
  distributed func exampleNextElement() async throws -> Element? { ... }
}
```

Failing to specify the serialization requirement is a compile time error:

```swift
protocol DistributedAsyncSequence: DistributedActor 
    where ActorSystem: DistributedActorSystem { 
    // error: distributed actor protocol must specify `ActorSystem.SerializationRequirement`,
    // you can provide it like this: DistributedActorSystem<any Codable>
  associatedtype Element: Sendable & Codable
      
  distributed func exampleNextElement() async throws -> Element? { ... }
}
```

The serialization requirement must be a `protocol` type; This was previously enforced, and remains so after this proposal. The important part is that the macro is able to synthesize an stub implementation type, that the existing resolution mechanisms can be invoked on. 

## Detailed design

The `@Resolvable` macro generates a concrete `distributed actor` declaration as well as an extension which implements the protocol's method requirements with "stub" implementations.

>  **NOTE:** The exact details of the macro synthesized code are not guaranteed to remain the same, and may change without notice. The existence of the $-prefixed generated type is guaranteed however, as it is the public API how developers resolve and obtain remote references, I.e. for a `protocol Greeter` annotated with the `@Resolvable` macro, developers may rely on the existence of the `distributed actor $Greeter` with the same access level as the protocol.

This proposal also introduces an empty `DistributedActorStub` protocol:

```swift
public protocol DistributedActorStub where Self: DistributedActor {}
```

The `@Resolvable` macro synthesizes a concrete distributed actor which accepts a generic `ActorSystem`. The generated actor declaration matches access level with the original declaration, and implements the protocol as well as the `DistributedActorStub` protocol:

```swift
protocol Greeter: DistributedActor where ActorSystem: DistributedActorSystem<any Codable> {
  distributed func greet(name: String) -> String
}

// "stub" type
distributed actor $Greeter<ActorSystem>: Greeter, DistributedStubActor
    where DistributedActorSystem<any Codable> {
  private init() {} // cannot initialize, can only resolve(id:using:)
}

extension Greeter where Self: DistributedActorStub {
  // ... stub implementations for protocol requirements ...
}
```

Default implementations for all the protocol's requirements (including non-distributed requirements) are provided by extensions utilizing the `DistributedActorStub` protocol.

It is possible for a protocol type to inherit other protocols, in that case the parent protocol must either have default implementations for all its requirements, or it must also apply the `@Resolvable` protocol which generates such default implementations.

The default method "stub" implementations provided by the `@Resolvable` simply fatal error if they were to ever be invoked. In practice, invoking those methods is not possible, because resolving a stub will always return a remote reference, and therefore calls on these methods are redirected to `DistributedActorSystem`'s `remoteCall` rather than invoking the "local" methods.  

It is recommended to use `some Greeter` or `any Greeter` rather than `$Greeter` when passing around resolved instances of a distributed actor protocol. This way none of your code is tied to the fact of using a specific type of proxy, but rather, can accept any greeter, be it local or resolved through a proxy reference. This can come in handy when refactoring a codebase, and merging modules in such way where the greeter may actually be a local instance in some situations.

### Interaction with the `DefaultDistributedActorSystem`

Since the introduction of distributed actors, it is possible to declare a module-wide `DefaultDistributedActorSystem` type alias, like this:

```swift
typealias DefaultDistributedActorSystem = ClusterSystem
```

This makes it easier to declare distributed actors as the `ActorSystem` type requirement is witnessed by an implicit type alias generated in every concrete distributed actor, like this:

```swift
distributed actor Worker { 
  // synthesized:
  // typealias ActorSystem = ClusterSystem // because 'DefaultDistributedActorSystem = ClusterSystem'
  
  distributed func work()
}
```

The newly introduced ability to abstract over the `ActorSystem` in concrete distributed actors _wins_ over the synthesized typealias, causing the typealias to not be emitted:

```swift
distributed actor Worker<ActorSystem> where ActorSystem: DistributedActorSystem<any Codable> {
  distributed func work()
}
```

The `ActorSystem` type requirement of the `DistributedActorProtocol` in this case is witnessed by the generic parameter, and not by the "default" fallback type.

This is the right behavior because this generic type works with _any_ distributed actor system where the `SerializationRequirement` is Codable, and not only on the `ClusterSystem`.

### Extend Distributed metadata for protocol method identifier lookups

The way distributed method invocations work on the recipient node is that a message is parsed from some incoming transport, and a `RemoteCallTarget` is recovered. The remote call target in currently is a mangled encoding of the concrete distributed method the call was made for, like this:

A shared module introducing the `Capybara` protocol:

```swift
// Module "Shared"
@Resolvable
public protocol Capybara where ActorSystem: DistributedActorSystem<any Codable> {
  distributed var name: String { get }
  distributed func eat()
}
```

The distributed actor runtime stores static metadata about distributed methods, such that the `executeDistributedTarget(on:target:invocationDecoder:handler:)` method is able to turn the mangled `RemoteCallIdentifier` into a concrete method handle that it then invokes. This proposal introduces a way to mangle calls on distributed protocol requirements, in such a way that they refer to the `$`-prefixed name, and invocations on such accessor are performed on the recipient side using the concrete actor's witness tables.

We can illustrate the new `remoteCall` flow like this:

**Caller, i.e. client side:**

```swift
// Module MyClientApp

let discoveredCaplinID = ...
let capybara: some Capybara = try $Caybara.resolve(id: discoveredCaplinID, using: system)

// make the remote call, without knowing the concrete 
// capybara type that we'll invoke on the remote side
let name = try await capybara.name

// invokes selected actor system's remoteCall:
//   DistributedActorSystem.remoteCall(
//     on: someCapybara,
//     target: RemoteCallIdentifier("Shared.$Capybara.name"), <<< PROTOCOL REQUIREMENT MANGLING
//     invocation: <InvocationEncoder>, 
//     throwing: Never.self, 
//     returnType: String.self) 
// -------

assert("Hello, \(name)!" ==
       "Hello, Caplin!")
```

The caller just performed a normal remote call as usual, however the Distributed runtime offered a protocol based remote call identifier (`RemoteCallIdentifier("Shared.$Capybara.name")`) rather than one based on some underlying type that is hidden under the `any Capybara` existential.

The client-side does not need to know or care about what concrete implementation type is used to implement this call on the recipient system, as it will only ever be performing such distributed protocol based calls.

**Recipient, i.e. server side:**

```swift
// Module MyActorSystem

final class MySampleDistributedActorSystem: DistributedActorSystem, ... {
  // ... 
  
  func findById(_ id: ActorID) -> (any DistributedActor)? { ... }
  
  func receiveMessage() async throws {
    let envelope: MyTransportEnvelope = try await readFromNetwork()
    
    guard let actor = findById(envelope.id) else {
      throw TargetActorNotFound(envelope.id)
    }

    // RemoteCallIdentifier("Shared.Capybara.name") <<< 
    let target: RemoteCallTarget = envelope.target 
    try await executeDistributedTarget(
      on: actor,
      target: target, // the protocol method identifier
      invocationDecoder: invocation.makeDecoder(),
      handler: resultHandler
    }
)
```

This logic is exactly the same as any existing `DistributedActorSystem` implementation -- however, changes in the Distributed runtime will handle the protocol method invocation and be able to route it to the concrete resolved actor (returned by `findByID` in this snippet, e.g. the `Caplin` concrete type).

## Source compatibility

The changes proposed are purely additive.

The introduced macros do not introduce any new capabilities to the `DistributedActorSystem` protocol itself, but rather introduce new source generation techniques.

## ABI compatibility

This proposal is purely ABI additive.

We introduce new static, accessible at runtime, metadata necessary for the identification and location of distributed protocol methods.

## Wire compatibility

> Since distributed actors are used across processes, an additional kind of compatibility is necessary to discuss in proposals which may impact how messages are sent or methods identified and invoked.

This proposal is additive and provides additional metadata such that "distributed protocol" methods may be invoked across process. Such calls were previously not supported.

Remote calls are identified using the `RemoteCallTarget` struct, which contains an `identifier` of the target method. In today's distributed actors these identifiers are the mangled name of the target method.

This proposal introduces a special way to mangle calls made on default implementations of distributed protocol requirements, in such a way that the target type identifier of the protocol (e.g. `Greeter`) is replaced with the stub type (e.g. `$Greeter`), and the server performs the invocation on a specific target actor using the concrete types witness and generic accessor thunk when such calls are made.

## Future directions

### Improve tools for non-breaking protocol evolution

While this approach allows sharing protocols as source of "truth" for APIs vended by a server, their capability to evolve over time is limited.

We find that the needs of distributed protocol evolution overlap in some parts with protocol evolution in binary stable libraries. While removing an API would always be breaking, it should be possible to automatically deprecate one method, and delegate to another by adding a new parameter and defaulting it in the deprecated version.

This could be handled with declaration macros, introducing a peer method with the expected new API:

```swift
@Resolvable(deprecatedName: "Greeter")
protocol DeprecatedGreeter: DistributedActor { 
  @Distributed.Deprecated(newVersion: greet(name:), defaults: [name: nil])
  distributed func greet() -> String 
  
  // whoops, we forgot about a name parameter and need to add it...
  // We can delegate from greet() to greet(name: nil) automatically though!
}
```

The deprecation macro could generate the necessary delegation code, like this:

```swift 
protocol Greeter: DistributedActor { 
  @Distributed.Deprecated(newVersion: greet(name:), defaults: [name: nil])
  distributed func greet() -> String
  
/***
  distributed func greet(name: String?) -> String 
 ***/
}

/***
extension Greeter {
  /// Default implementation for deprecated ``greet()``
  /// Delegates to ``greet(name:)``
  distributed func greet() -> String {
    self.greet(name: nil)
  }
}
 ***/
```

This simplified snippet does not solve the problem about introducing the new protocol requirement in a binary compatible way, and we'd have to come up with some pattern for it -- however the general direction of allowing introducing new versions of APIs with easier deprecation of old ones is something we'd like to explore in the future.

Support for renaming methods could also be provided, such that the legacy method can be called `__deprecated_greet()` for example, while maintaining the "legacy name" of "`greet()`". Overall, we believe that the protocol evolution story here is somethign we will have to flesh out in the near future, anf feel we have the tools to do so.

### Consider customization points for distributed call target metadata assignment

Currently the metadata used for remote call targets is based on Swift's mangling scheme. This is sub-optimal as it includes slightly "too much" information, such as the parameters being classes or structs, un-necessarily wire causing wire-incompatible changes when one could handle them more gracefully.

Another downside of using the mangling for the keys of the distributed method accessor identifiers is that the names can be rather long as mangling is pretty verbose. It is possible to avoid sending then complete metadata by using compression schemes such that each identifier can only be sent at-most-once over the wire, and later on a numeric representation is used between the peers. Such scheme would need to be implemented dynamically at runtime and involves some tricky logic. It would be interesting to provide a hook in the actor system to allow for consistent remote call target identifier assignment, such that the identifiers could be both small, and predictable on both sides of a system. 

This would require introducing a dynamic lookup table in the runtime and it would need to interoperate with any given distributed actor system... It remains unclear if this is a net win, or un-necessary complexity since each actor system may handle this slightly differently...

### Utilize distributed method metadata for auditing

Given the information in distributed metadata, we could provide a command line application, or rather extend `swift-inspect` to be able to inspect a binary or running application for the distributed entry points to the application. 

This is useful as it allows auditors to quickly scan for all potential distributed entry points into an application, making auditing easier and more reliable than source scanning as it can be performed on the final artifact of a build.

## Alternatives considered

### Restrict distributed actors only to peer-to-peer systems

Since this feature expands the usefulness of distributed actors to client/server settings, we should discuss wether or not this is a good idea to begin with.

This topic has been one of heated discussions in various ecosystems every time distributed actor systems are compared to source-generation based RPC systems (such as gRPC, or OpenAPI, and others like SOAP and others in earlier days).

Distributed actors in Swift have the unique position of being placed in a language that deeply embraces the actor model. There are valid reasons to NOT use distributed actors in some situations, and e.g. prefer exposing your APIs over OpenAPI or other source generation tools, especially if one wants to treat these as the "source of truth."

At the same time though, Swift is used in many exciting domains where the use of OpenAPI, or gRPC would be deemed problematic. We are interested in supporting IPC and other low-level systems which are tightly integrated with eachother, and frequently even maintained by the same teams or organizations. Deeply integrating the language, with auditing capabilities and control over distributed process boundaries, without having to step out to secondary source generation and tools is a very valuable goal in these scenarios.

### Handle stub synthesis in the compiler

An earlier attempt at implementation of this feature attempted to handle synthesis in the compiler, and emit ad-hoc distributed actor declaration types as triggered by the _call site_ of `resolve(id:using:)` - this is problematic in being a very custom and special path in the compiler, complicating the language and giving distributed actors more "privileges" than normal code.

The idea was as follows:

```swift
protocol Greeter: DistributedActor {
  distributed func greet(name: String) -> String
}

let someSystem: some DistributedActorSystem<any Codable> = ... 
let g: any Greeter = try .resolve(id: id, using: someSystem)
```

This would have to synthesize an ad-hoc created anonymous declaration for a `$Greeter` and at the site of the `resolve` type-check if the declaration can be used with the `someSystem`'s serialization requirement. We would have to check if the distributed greet method's parameters and return type conform to `Codable` etc, and all this would have to happen lazily -- triggered by the existence of a `.resolve` method combining a protocol with a specific actor system.

The only possible spelling of such API would have been this: `let g: any Greeter = try .resolve(...)` as the concrete type that is used to implement this `any Greeter` is not user visible, and cannot be. This is a lot of complexity, to what amounts to just a simple stub type.

We believe that the macro stub approach is a good balance between convenience and lack of "magic" compiler support, as for this specific piece of the design no deep integration in the type system is necessary.


## Revisions

- 1.2
  - Change implementation to not need `#resolve` macro, but rely on generic distributed actors
  - General cleanup
  - Change implementation approach to macros, introduce `#resolve` macro
- 1.0
  - Initial revision
