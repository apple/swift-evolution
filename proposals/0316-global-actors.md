# Global actors

* Proposal: [SE-0316](0316-global-actors.md)
* Authors: [John McCall](https://github.com/rjmccall), [Doug Gregor](https://github.com/DougGregor)
* Review Manager: [Joe Groff](https://github.com/jckarter)
* Status: **Implemented (Swift 5.5)**
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-with-modifications-se-0316-global-actors/50116)

## Table of Contents

* [Introduction](#introduction)
* [Motivation](#motivation)
* [Proposed solution](#proposed-solution)
    * [Defining global actors](#defining-global-actors)
    * [The main actor](#the-main-actor)
    * [Using global actors on functions and data](#using-global-actors-on-functions-and-data)
    * [Global actor function types](#global-actor-function-types)
    * [Closures](#closures)
    * [Global and static variables](#global-and-static-variables)
    * [Using global actors on a type](#using-global-actors-on-a-type)
    * [Global actor inference](#global-actor-inference)
    * [Global actors and instance actors](#global-actors-and-instance-actors)
* [Detailed design](#detailed-design)
    * [`GlobalActor` protocol](#globalactor-protocol)
    * [Closure attributes](#closure-attributes)
* [Source compatibility](#source-compatibility)
* [Effect on ABI stability](#effect-on-abi-stability)
* [Effect on API resilience](#effect-on-api-resilience)
* [Effect on runtime and standard library](#effect-on-runtime-and-standard-library)
* [Future directions](#future-directions)
    * [Restricting global and static variables](#restricting-global-and-static-variables)
    * [Global actor-constrained generic parameters](#global-actor-constrained-generic-parameters)
* [Alternatives considered](#alternatives-considered)
    * [Singleton support](#singleton-support)
    * [Propose only the main actor](#propose-only-the-main-actor)
* [Revision history](#revision-history)

## Introduction

[Actors](0306-actors.md) are a new kind of reference type that protect their instance data from concurrent access. Swift actors achieve this with *actor isolation*, which ensures (at compile time) that all accesses to that instance data go through a synchronization mechanism that serializes execution.

This proposal introduces *global actors*, which extend the notion of actor isolation outside of a single actor type, so that global state (and the functions that access it) can benefit from actor isolation, even if the state and functions are scattered across many different types, functions and modules. Global actors make it possible to safely work with global variables in a concurrent program, as well as modeling other global program constraints such as code that must only execute on the "main thread" or "UI thread".

Global actors also provide a means to eliminate data races on global and static variables, allowing access to such variables to be synchronized via a global actor.

Swift-evolution threads: [Pitch #1](https://forums.swift.org/t/pitch-global-actors/45706), [Pitch #2](https://forums.swift.org/t/pitch-2-global-actors/48332)

## Motivation

Actors are fantastic for isolating instance data, providing a form of reference type that can be used in concurrent programs without introducing data races. However, when the data that needs to be isolated is scattered across a program, or is representing some bit of state that exists outside of the program, bringing all of that code and data into a single actor instance might be impractical (say, in a large program) or even impossible (when interacting with a system where those assumptions are pervasive).

A primary motivator of global actors is to apply the actor model to the state and operations that can only be accessed by the *main thread*. In an application, the main thread is generally responsible for executing the primary event-handling loop that processes events from various sources and delivers them to application code. Graphical applications often deliver user-interaction events (a keyboard press, a touch interaction) on the main thread, and require that any stateful updates to the user interface occur there as well. Global actors provide the mechanism for describing the main thread in terms of actors, utilizing Swift's actor isolation model to aid in correct usage of the main thread.

## Proposed solution

A global actor is a globally-unique actor identified by a type. That type becomes a custom attribute (similar to [property wrapper types](0258-property-wrappers.md) or [result builder types](0289-result-builders.md)). Any declaration can state that it is actor-isolated to that particular global actor by naming the global actor type as an attribute, at which point all of the normal actor-isolation restrictions come into play: the declaration can only be synchronously accessed from another declaration on the same global actor, but can be asynchronously accessed from elsewhere. For example, this proposal introduces `MainActor` as a global actor describing the main thread. It can be used to require that certain functions only execute on the main thread:

```swift
@MainActor var globalTextSize: Int

@MainActor func increaseTextSize() { 
  globalTextSize += 2   // okay: 
}

func notOnTheMainActor() async {
  globalTextSize = 12  // error: globalTextSize is isolated to MainActor
  increaseTextSize()   // error: increaseTextSize is isolated to MainActor, cannot call synchronously
  await increaseTextSize() // okay: asynchronous call hops over to the main thread and executes there
}
``` 

### Defining global actors

A global actor is a type that has the `@globalActor` attribute and contains a `static` property named `shared` that provides a shared instance of an actor. For example:

```swift
@globalActor
public struct SomeGlobalActor {
  public actor MyActor { }

  public static let shared = MyActor()
}
```

A global actor type can be a struct, enum, actor, or `final` class. It is essentially just a marker type that provides access to the actual shared actor instance via `shared`. The shared instance is a globally-unique actor instance that becomes synonymous with the global actor type, and will be used for synchronizing access to any code or data that is annotated with the global actor.

Global actors implicitly conform to the `GlobalActor` protocol, which describes the `shared` requirement. The conformance of a `@globalActor` type to the `GlobalActor` protocol must occur in the same source file as the type definition, and the conformance itself cannot be conditional.


### The main actor

The *main actor* is a global actor that describes the main thread:

```swift
@globalActor
public actor MainActor {
  public static let shared = MainActor(...)
}
```

> **Note**: integrating the main actor with the system's main thread requires support for [custom executors][customexecs], which is the subject of another proposal, as well as specific integration with the system's notion of the main thread. For systems that use the Apple's [Dispatch](https://developer.apple.com/documentation/DISPATCH) library as the underlying concurrency implementation, the main actor uses a custom executor that wraps the [main dispatch queue](https://developer.apple.com/documentation/dispatch/dispatchqueue/1781006-main). It also determines when code is dynamically executing on the main actor to avoid an extra "hop" when performing an asynchronous call to a `@MainActor` function.

### Using global actors on functions and data

As illustrated in our first example, both functions and data can be attributed with a global actor type to isolate them to that global actor. Note that global actors are not restricted to global functions or data as in the first example. One can mark members of types and protocols as belonging to a global actor as well. For example, in a view controller for a graphical UI, we would expect to receive notification of user interactions on the main thread, and must update the UI on the main thread. Therefore want both the methods called on notification and also the data they use to be on the main actor. Here's an small part of a view controller from some [AppKit sample code](https://developer.apple.com/documentation/appkit/cocoa_bindings/navigating_hierarchical_data_using_outline_and_split_views):

```swift
class IconViewController: NSViewController {
  @MainActor @objc private dynamic var icons: [[String: Any]] = []
    
  @MainActor var url: URL?
    
  @MainActor private func updateIcons(_ iconArray: [[String: Any]]) {
    icons = iconArray
        
    // Notify interested view controllers that the content has been obtained.
    // ...
  }
}
```

Note that the data in this view controller, as well as the method that performs the update of this data, is isolated to the `@MainActor`. That ensures that UI updates for this view controller only occur on the main thread, and any attempts to do otherwise will result in a compiler error.

The sample code actually triggers an update when the `url` property is set. With global actors, that would look something like this:

```swift
@MainActor var url: URL? {
  didSet {
    // Asynchronously perform an update
    Task.detached { [url] in                   // not isolated to any actor
      guard let url = url else { return }
      let newIcons = self.gatherContents(url)
      await self.updateIcons(newIcons)         // 'await' required so we can hop over to the main actor
    }
  }
}
```

### Global actor function types

A synchronous function type can be qualified to state that the function is only callable on a specific global actor:

```swift
var callback: @MainActor (Int) -> Void
```

Such a function can only be synchronously called from code that is itself isolated to the same global actor.

A reference to a function that is isolated to a global actor will have a function type with a global actor. The references themselves are not subject to actor-isolation checking, because the actor isolation is described by the resulting function type. For example:

```swift
func functionsAsValues(controller: IconViewController) {
  let fn = controller.updateIcons // okay, type is @MainActor ([[String: Any]]) -> Void
  let fn2 = IconViewController.controller.updateIcons // okay, type is (IconViewController) -> (@MainActor ([[String: Any]]) -> Void)
  fn([]) // error: cannot call main actor-isolated function synchronously from outside the actor
}
```

Values may be converted from a function type with no global actor qualifier to a function with a global actor qualifier. For example:

```swift
func acceptInt(_: Int) { } // not on any actor

callback = acceptInt // okay: conversion to @MainActor (Int) -> Void
```

The opposite conversion is not permitted for synchronous functions, because doing so would allow the function to be called without being on the global actor:

```swift
let fn3: (Int) -> Void = callback // error: removed global actor `MainActor` from function type
```

However, it is permissible for the global actor qualifier to be removed when the result of the conversion is an `async` function. In this case, the `async` function will first "hop" to the global actor before executing its body:

```swift
let callbackAsynchly: (Int) async -> Void = callback   // okay: implicitly hops to main actor
```

This can be thought of as syntactic sugar for the following:

```swift
let callbackAsynchly: (Int) async -> Void = {
  await callback() // `await` is required because callback is `@MainActor`
}
```

A global actor qualifier on a function type is otherwise independent of `@Sendable`, `async`, `throws` and most other function type attributes and modifiers. The only exception is when the function itself is also isolated to an instance actor, which is discussed in the later section on [Global actors and instance actors](#global-actors-and-instance-actors).

### Closures

A closure can be explicitly specified to be isolated to a global actor by providing the attribute prior to the `in` in the closure specifier, e.g.,

```swift
callback = { @MainActor in
  print($0)
}

callback = { @MainActor (i) in 
  print(i)
}
```

When a global actor is applied to a closure, the type of the closure is qualified with that global actor. 

> **Note**: this can be used to replace the common pattern used with Apple's Dispatch library of executing main-thread code via `DispatchQueue.main.async { ... }`. One would instead write:
> ```swift
> Task.detached { @MainActor in
>   // ...
> }
> ```
> This formulation ensures that the closure body is executed on the main actor, and can synchronously use other `@MainActor`-annotated declarations.

If a closure is used to directly initialize a parameter or other value of a global-actor-qualified function type, and the closure itself does not have a global actor explicitly specified on it, the closure will have that global actor inferred. For example:

```swift
@MainActor var globalTextSize: Int

var callback: @MainActor (Int) -> Void
callback = { // closure is inferred to be @MainActor due to the type of 'callback'
  globalTextSize = $0  // okay: closure is on @MainActor
}
```

### Global and static variables

Global and static variables can be annotated with a global actor. Such variables can only be accessed from the same global actor or asynchronously, e.g.,

```swift
@MainActor var globalCounter = 0

@MainActor func incrementGlobalCounter() {
  globalCounter += 1   // okay, we are on the main actor
}

func readCounter() async {
  print(globalCounter)         // error: cross-actor read requires 'await'
  print(await globalCounter)   // okay
}
```

As elsewhere, cross-actor references require the types involved to conform to `Sendable`. 

Global and static variables not annotated with a global actor can effectively be accessed from any concurrency context, and as such are prone to data races. Global actors provide one way to address such data races. The section on [future directions](#future-directions) considers whether to use global actors as a way to address data races for global and static variables comprehensively.

### Using global actors on a type

It is common for entire types (and even class hierarchies) to predominantly require execution on the main thread, and for asynchronous work to be a special case. In such cases, the type itself can be annotated with a global actor, and all of the methods, properties, and subscripts will implicitly be isolated to that global actor. Any members of the type that do not want to be part of the global actor can opt out, e.g., using the [`nonisolated` modifier][isolation]. For example:

```swift
@MainActor
class IconViewController: NSViewController {
  @objc private dynamic var icons: [[String: Any]] = [] // implicitly @MainActor
    
  var url: URL? // implicitly @MainActor
  
  private func updateIcons(_ iconArray: [[String: Any]]) { // implicitly @MainActor
    icons = iconArray
        
    // Notify interested view controllers that the content has been obtained.
    // ...
  }
  
  nonisolated private func gatherContents(url: URL) -> [[String: Any]] {
    // ...
  }
}
```

A non-protocol type that is annotated with a global actor implicitly conforms to `Sendable`. Instances of such types are safe to share across concurrency domains because access to their
state is guarded by the global actor.

A class can only be annotated with a global actor if it has no superclass, the superclass is annotated with the same global actor, or the superclass is `NSObject`. A subclass of a global-actor-annotated class must be isolated to the same global actor.

### Global actor inference

Declarations that are not explicitly annotated with either a global actor or `nonisolated` can infer global actor isolation from several different places:

* Subclasses infer actor isolation from their superclass:

  ```swift
  class RemoteIconViewController: IconViewController { // implicitly @MainActor
      func connect() { ... } // implicitly @MainActor
  }
  ```
  
* An overriding declaration infers actor isolation from the declaration it overrides:

  ```swift
  class A {
    @MainActor func updateUI() { ... }
  }
  
  class B: A {
    override func updateUI() { ... } // implicitly @MainActor
  }
  ```

* A witness that is not inside an actor type infers actor isolation from a protocol requirement that is satisfies, so long as the protocol conformance is stated within the same type definition or extension as the witness:

  ```swift
  protocol P {
    @MainActor func f()
  }
  
  struct X { }
  
  extension X: P {
    func f() { } // implicitly @MainActor
  }
  
  struct Y: P { }
  
  extension Y {
    func f() { } // okay, not implicitly @MainActor because it's in a separate extension
                 // from the conformance to P
  }
  ```

* A non-actor type that conforms to a global-actor-qualified protocol within the same source file as its primary definition infers actor isolation from that protocol:

  ```swift
  @MainActor protocol P {
    func updateUI() { } // implicitly @MainActor
  }
  
  class C: P { } // C is implicitly @MainActor
  
  // source file D.swift
  class D { }
  
  // different source file D-extensions.swift
  extension D: P { // D is not implicitly @MainActor
    func updateUI() { } // okay, implicitly @MainActor
  }
  ```

* A struct or class containing a wrapped instance property with a global actor-qualified `wrappedValue` infers actor isolation from that property wrapper:

  ```swift
  @propertyWrapper
  struct UIUpdating<Wrapped> {
    @MainActor var wrappedValue: Wrapped
  }
  
  struct CounterView { // infers @MainActor from use of @UIUpdating
    @UIUpdating var intValue: Int = 0
  }
  ```

### Global actors and instance actors

A declaration cannot both be isolated to a global actor and isolated to an instance actor.  If an instance declaration within an actor type is annotated with a global actor, it is isolated to the global actor but *not* its enclosing actor instance:

```swift
actor Counter {
  var value = 0
  
  @MainActor func updateUI(view: CounterView) async {
    view.intValue = value  // error: `value` is actor-isolated to `Counter` but we are in a 'MainActor'-isolated context
    view.intValue = await value // okay to asynchronously read the value
  }
}
```

With the `isolated` parameters described in [SE-0313][isolation], no function type can contain both an `isolated` parameter and also a global actor qualifier:

```swift
@MainActor func tooManyActors(counter: isolated Counter) { } // error: 'isolated' parameter on a global-actor-qualified function
```

## Detailed design

Global actor attributes apply to declarations as follows:

* A declaration cannot have multiple global actor attributes.  The rules below say that, in some cases, a global actor attribute is propagated from one declaration to another.  If the rules say that an attribute “propagates by default”, then no propagation is performed if the destination declaration has an explicit global actor attribute.  If the rules say that attribute “propagates mandatorily”, then it is an error if the destination declaration has an explicit global actor attribute that does not identify the same actor.  Regardless, it is an error if global actor attributes that do not identify the same actor are propagated to the same declaration.

* A function declared with a global actor attribute becomes isolated to the given global actor.

* A stored variable or constant declared with a global actor attribute becomes part of the isolated state of the given global actor.

* The accessors of a variable or subscript declared with a global actor attribute become isolated to the given global actor.  (This includes observing accessors on a stored variable.)

* Local variables and constants cannot be marked with a global actor attribute.

* A type declared with a global actor attribute propagates the attribute to all methods, properties, subscripts, and extensions of the type by default. 

* An extension declared with a global actor attribute propagates the attribute to all the members of the extension by default.

* A protocol declared with a global actor attribute propagates the attribute to any type that conforms to it in the primary type definition by default.

* A protocol requirement declared with a global actor attribute requires that a given witness must either have the same global actor attribute or be non-isolated. (This is the same rule observed by all witnesses for actor-isolated requirements).

* A class declared with a global actor attribute propagates the attribute to its subclasses mandatorily.

* An overridden declaration propagates its global actor attribute (if any) to its overrides mandatorily.  Other forms of propagation do not apply to overrides.  It is an error if a declaration with a global actor attribute overrides a declaration without an attribute.

* An actor type cannot have a global actor attribute.  Stored instance properties of actor types cannot have global actor attributes.  Other members of an actor type can have global actor attributes; such members are isolated to the global actor, but not to the enclosing actor. (Per the proposal on [improved control over actor isolation][isolation], the `self` of such methods is not `isolated`).

* A `deinit` cannot have a global actor attribute and is never a target for propagation.

### `GlobalActor` protocol

The `GlobalActor` protocol is defined as follows:

```swift
/// A type that represents a globally-unique actor that can be used to isolate
/// various declarations anywhere in the program.
///
/// A type that conforms to the `GlobalActor` protocol and is marked with the
/// the `@globalActor` attribute can be used as a custom attribute. Such types
/// are called global actor types, and can be applied to any declaration to
/// specify that such types are isolated to that global actor type. When using
/// such a declaration from another actor (or from nonisolated code),
/// synchronization is performed through the \c shared actor instance to ensure
/// mutually-exclusive access to the declaration.
public protocol GlobalActor {
  /// The type of the shared actor instance that will be used to provide
  /// mutually-exclusive access to declarations annotated with the given global
  /// actor type.
  associatedtype ActorType: Actor

  /// The shared actor instance that will be used to provide mutually-exclusive
  /// access to declarations annotated with the given global actor type.
  ///
  /// The value of this property must always evaluate to the same actor
  /// instance.
  static var shared: ActorType { get }
}
```

### Closure attributes

The global actor for a closure is one of a number of potentially-allowable attributes on a closure. The attributes precede the capture-list in the grammar:

```
closure-expression → { closure-signature opt statements opt }
closure-signature → attributes[opt] capture-list[opt] closure-parameter-clause async[opt] throws[opt] function-result[opt] in
closure-signature → attributes[opt] capture-list in
closure-signature → attributes in
```

## Source compatibility

Global actors are an additive feature that have no impact on existing source code.

## Effect on ABI stability

A global actor annotation is part of the type of an entity, and is therefore part of its mangled name. Otherwise, a global actor has no effect on the ABI.

## Effect on API resilience

The `@globalActor` attribute can be added to a type without breaking API.

A global actor attribute (such as `@MainActor`) can neither be added nor removed from an API; either will cause breaking changes for source code that uses the API.

## Effect on runtime and standard library

This proposal introduces a new kind of function type, a global-actor-qualified function type, which requires updates to the Swift runtime, metadata, name mangling scheme, and dynamic-casting machinery. For example, consider the following code:

```swift
@MainActor func f(_ potentialCallback: Any) {
  let Callback = @MainActor () -> Void
  if let callback = potentialCallback as? Callback {
    callback()
  }
}
```

The dynamic cast to a global-actor-qualified function type requires changes to the Swift runtime to represent global-actor-qualified function types and model dynamic casts of unknown values to them. Similar changes for global-actor-qualified function types are required for name mangling, which also has runtime impact.

## Future directions

### Restricting global and static variables

A global actor annotation on a global or static variable synchronizes all access to that variable through that global actor. We could require that *all* mutable global and static variables be annotated with a global actor, thereby eliminating those as a source of data races. Specifically, we can require that every global or static variable do one of the following:

* Explicitly state that it is part of a global actor, or
* Be immutable (introduced via `let`), non-isolated, and of `Sendable` type.

This allows global/static immutable constants to be used freely from any code, while any data that is mutable (or could become mutable in a future version of a library) must be protected by an actor. However, it comes with significant source breakage: every global variable that exists today would require annotation. Therefore, we aren't proposing to introduce this requirement, and instead leave the general data-race safety of global and static variables to a later proposal.

### Global actor-constrained generic parameters

A generic parameter that is constrained to `GlobalActor` could potentially be used as a global actor. For example:

```swift
@T
class X<T: GlobalActor> {
  func f() { ... } // constrained to the global actor T
}

@MainActor func g(x: X<MainActor>, y: X<OtherGlobalActor>) async {
  x.f() // okay, on the main actor
  await y.f() // okay, but requires asynchronous call because y.f() is on OtherGlobalActor
}
```

There are some complications here: without marking the generic parameter `T` with the `@globalActor` attribute, it wouldn't be clear what kind of custom attribute `T` is. Therefore, this might need to be expressed as, e.g.,

```swift
@T
class X<@globalActor T> { ... }
```

which would imply the requirement `T: GlobalActor`. However, doing this would require Swift to also support attributes on generic parameters, which currently don't exist. This is a promising direction for a follow-on proposal.

## Alternatives considered

### Singleton support

Global actors are, effectively, baking a convention for singletons in the language. Singletons are occasionally used in Swift, and if they were to get special language syntax, global actors could be introduced with less boilerplate as "singleton actors", e.g., 

```swift
singleton actor MainActor {
  // integration with system's main thread
}
```

This would eliminate the `@globalActor` attribute from the proposal, but would otherwise leave it unchanged.

### Propose only the main actor

The primary motivation for global actors is the main actor, and the semantics of this feature are tuned to the needs of main-thread execution. We know abstractly that there are other similar use cases, but it's possible that global actors aren't the right match for those use cases. Rather than provide a general feature for global actors now, we could narrow this proposal to `@MainActor` only, then provide global actors (or some other abstraction) at some later point to subsume `@MainActor` and other important use cases.

## Revision history

* Changes to the accepted version:
  * Move global actor-constrained generic parameters to "future directions"
  * Classes that are global actors must be `final`.
* Changes for the second review:
    * Added the `GlobalActor` protocol, to which all global actors implicitly conform.
    * Remove the requirement that all global and static variables be annotated with a global actor.
    * Added a grammar for closure attributes.
    * Clarified the interaction between the main actor and the main thread. Make the main actor a little less "special" in the initial presentation.
* Changes for the first review:
    * Add inference of a global actor for a witness to a global-actor-qualified requirement.
    * Extended inference of global actor-ness from protocols to conforming types to any extension within the same source file as the primary type definition.
* Changes in the second pitch:
    * Clarify that the types of global-actor-qualified functions are global-actor-qualified.
    * State that global-actor-qualified types are Sendable
    * Expand on the implicit conversion rules for function types
    * Require global and static variables to be immutable & non-isolated or global-actor-qualified.
    * Describe the relationship between global actors and instance actors
    * Update inference rules for global actors


[customexecs]: https://github.com/rjmccall/swift-evolution/blob/custom-executors/proposals/0000-custom-executors.md
[isolation]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0313-actor-isolation-control.md

