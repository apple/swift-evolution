# Dynamic actor isolation enforcement from non-strict-concurrency contexts

* Proposal: [SE-0423](0423-dynamic-actor-isolation.md)
* Authors: [Holly Borla](https://github.com/hborla), [Pavel Yaskevich](https://github.com/xedin)
* Review Manager: [Ben Cohen](https://github.com/airspeedswift)
* Status: **Implemented (Swift 6.0)**
* Upcoming Feature Flag: `DynamicActorIsolation`
* Review: ([pitch](https://forums.swift.org/t/pitch-dynamic-actor-isolation-enforcement/68354)) ([first review](https://forums.swift.org/t/se-0423-dynamic-actor-isolation-enforcement-from-non-strict-concurrency-contexts/70155)) ([second review](https://forums.swift.org/t/se-0423-second-review-dynamic-actor-isolation-enforcement-from-non-strict-concurrency-contexts/71159)) ([acceptance](https://forums.swift.org/t/accepted-se-0423-dynamic-actor-isolation-enforcement-from-non-strict-concurrency-contexts/71540))

## Introduction

Many Swift programs need to interoperate with frameworks written in C/C++/Objective-C whose implementations cannot participate in static data race safety. Similarly, many Swift programs have dependencies that have not yet adopted strict concurrency checking. A `@preconcurrency import` statement downgrades concurrency-related error messages that the programmer cannot resolve because the fundamental issue is in one of the dependencies. To strengthen Swift's data-race safety guarantees while working with preconcurrency dependencies, this proposals adds actor isolation checking at runtime for synchronous isolated functions.

## Motivation

The ecosystem of Swift libraries has a vast surface area of APIs that predate strict concurrency checking, relying on carefully calling APIs from the appropriate thread or dispatch queue to avoid data races. Migrating all of these libraries to strict concurrency checking will happen incrementally, motivating [SE-0337: Incremental migration to concurrency checking](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0337-support-incremental-migration-to-concurrency-checking.md) which introduced the `@preconcurrency import` statement to suppress concurrency warnings from APIs that programmers do not control.

If an actor isolation violation exists in the implementation of a preconcurrency library, the bug is only surfaced to clients as hard-to-debug data races on isolated state. `@preconcurrency` also does not apply to protocol conformances; there is no way to suppress concurrency diagnostics when conforming to a protocol from a preconcurrency library. This is unfortunate, because it's common for protocols to have a dynamic invariant that all requirements are called on the main thread or a specific dispatch queue provided by the client.

For example, consider the following protocol in a library called `NotMyLibrary`, which provides a guarantee that its requirements are always called from the main thread:

```swift
public protocol ViewDelegateProtocol {
  func respondToUIEvent()
}
```

and a client of `NotMyLibrary` that contains a conformance to `ViewDelegateProtocol`:

```swift
import NotMyLibrary

@MainActor
class MyViewController: ViewDelegateProtocol {
  func respondToUIEvent() { // error: @MainActor function cannot satisfy a nonisolated requirement
      // implementation...   
  }
}
```

The above code is invalid because `MyViewController.respondToUIEvent()` is `@MainActor`-isolated, but it satisfies a `nonisolated` protocol requirement that can be called from generic code off the main actor. If the library provides a dynamic guarantee that the requirement is always called on the main actor, a sensible workaround is to resort to dynamic actor isolation checking by marking the function as `nonisolated` and wrapping the implementation in `MainActor.assumeIsolated`:

```swift
import NotMyLibrary

@MainActor
class MyViewController: ViewDelegateProtocol {
  nonisolated func respondToUIEvent() {
    MainActor.assumeIsolated {
      // implementation...   
    }
  }
}
```

With this workaround, the programmer must annotate every witness with `nonisolated` and wrap the implementation in `MainActor.assumeIsolated`. More importantly, the programmer loses static data-race safety in their own code, because internal callers of `respondToUIEvent()` are free to invoke it from any isolation domain without compiler errors.

## Proposed solution

This proposal adds dynamic actor isolation checking to:

  - Witnesses of synchronous `nonisolated` protocol requirements when the witness is isolated and the protocol conformance is annotated as `@preconcurrency`. For example:

    If `respondToUIEvent` is a witness to a synchronous `nonisolated` protocol requirement, the protocol conformance error can be suppressed using a `@preconcurrency` annotation on the protocol to indicate that the protocol itself predates concurrency:

    ```swift
    import NotMyLibrary

    @MainActor
    class MyViewController: @preconcurrency ViewDelegateProtocol {
     func respondToUIEvent() {
       // implementation...
     }
    }
    ```

    The witness checker diagnostic will be suppressed, the actor isolation assertion will fail if `respondToUIEvent()` is called inside `NonMyLibrary` from off the main actor, and the compiler will continue to emit diagnostics inside the module when called from off the main actor.

    These dynamic checks apply to any situation where a synchronous `nonisolated` requirement is implemented by an isolated method, including synchronous actor methods.

  - `@objc` thunks of synchronous actor-isolated members of classes.

    Similarly to the previous case if a class or its individual synchronous members are actor-isolated and marked as either `@objc` or `@objcMembers`, the thunks, synthesized by the compiler to make them available from Objective-C, would have a new precondition check to make sure that use always happens on the right actor.

  - Synchronous actor-isolated function values passed to APIs that erase actor isolation and haven't yet adopted strict concurrency checking.

    When API comes from a module that doesn't have strict concurrency checking enabled it's possible that it could introduce actor isolation violations that would not be surfaced to a client. In such cases actor isolation erasure should be handled defensively by introducing a runtime check at each position for granular protection.

    ```swift
    @MainActor
    func updateUI(view: MyViewController) {
        NotMyLibrary.track(view.renderToUIEvent)
    }
    ```

    The use of `track` here would be considered unsafe if it accepts a synchronous nonisolated function type due to loss of `@MainActor` from `renderToUIEvent` and compiler would transform the call site into a function equivalent of:

    ```swift
    @MainActor
    func updateUI(view: MyViewController) {
        NotMyLibrary.track({
            MainActor.assumeIsolated {
                view.renderToUIEvent()
            }
        })
    }
    ```

  - Call-sites of synchronous actor-isolated functions imported from Swift 6 libraries.

    When importing a module that was compiled with the Swift 6 language mode into code that is not, it's possible to call actor-isolated functions from outside the actor using `@preconcurrency`. For example:

    ```swift
    // ModuleA built with -swift-version 6
    @MainActor public func onMain() { ... }

    // ModuleB built with -swift-version 5 -strict-concurrency=minimal
    import ModuleA

    @preconcurrency @MainActor func callOnMain() {
      onMain()
    }

    func notIsolated() {
      callOnMain()
    }
    ```

    In the above code, `onMain` from ModuleA can be called from outside the main actor via a call to `notIsolated()`. To close this safety hole, a dynamic check is inserted at the call-site of `onMain()` when ModuleB is recompiled against ModuleA after ModuleA has migrated to the Swift 6 language mode.

These are the most common circumstances when losing actor isolation could be problematic and restricting runtime checking to them significantly limits negative performance impact of the new checks. The strategy of only emitting runtime checks when there’s potential for the function to be called from unchecked code is desirable, because it means the dynamic checks will be eliminated as more of the Swift ecosystem transitions to Swift 6.


## Detailed design

### Runtime actor isolation checking

For all of the situations described in the previous section the compiler will emit a runtime check to assert that the current executor matches the expected executor of the isolated actor. Calling an isolated synchronous function from outside the isolation domain will result in a runtime error that halts program execution.

Runtime checking for actor isolation is not necessary for `async` functions, because switching to the callee's actor is always performed by the callee. `async` functions cannot be unsafely called from non-Swift code because they are not available directly in C/C++/Objective-C.

### `@preconcurrency` conformances

A `@preconcurrency` protocol conformance is scoped to the implementation of the protocol requirements in the conforming type. A `@preconcurrency` conformance can be written at the primary declaration or in an extension, and witness checker diagnostics about actor isolation will be suppressed. Like other `@preconcurrency` annotations, if no diagnotsics are suppressed, a warning will be emitted at the `@preconcurrency` annotation stating that the annotation has no effect and it should be removed.

### Disabling dynamic actor isolation checking

The dynamic actor isolation checks can be disabled using the flag `-disable-dynamic-actor-isolation`. Disabling dynamic actor isolation is discouraged, but it may be necessary if code that you don't control violates actor isolation in a way that causes the program to crash, such as by passing a non-`Sendable` function argument outside of a main actor context. `-disable-dynamic-actor-isolation` is similar to the `-enforce-exclusivity=unchecked` flag, which was a tool provided when staging in dynamic memory exclusivity enforcement under the Swift 5 lanugage mode.

## Source compatibility

Dynamic actor isolation checking can introduce new runtime assertions for existing programs. Therefore, dynamic actor isolation is only performed for synchronous functions that are witnesses to an explicitly annotated `@preconcurrency` protocol conformance, or that are compiled under the Swift 6 language mode.

## ABI compatibility

This proposal has no impact on ABI compatibility of existing code. There are runtime implications for code that explicitly adopts this feature; see the following section.

## Implications on adoption

This feature can be freely adopted and un-adopted in source code with no deployment constraints and without affecting source or ABI compatibility. However, as noted in the Source compatibility section, adoption of this feature has runtime implications, because actor-isolated code called incorrectly from preconcurrency code will crash instead of race.

## Alternatives considered

### Always emit dynamic checks upon entry to synchronous isolated functions

A previous iteration of this proposal specified that dynamic actor isolation checks are always emitted upon entry to a synchronous isolated function. This approach is foolproof; there's little possiblity for missing a dynamic check for code that can be called from another module that does not have strict concurrency checking at compile time. However, the major downside of this approach is that code will be paying the price of runtime overhead for actor isolation checking even when actor isolation is fully enforced at compile time in Swift 6.

The current approach in this proposal has a very desirable property of eliminated more runtime overhead as more of the Swift ecosystem transitions to Swift 6 at the cost of introducing the potential for missing dynamic checks where synchronous functions can be called from not-statically-checked code. We believe this is the right tradeoff for the long term arc of data race safety in Swift 6 and beyond, but it may require more special cases when we discover code patterns that are not covered by the specific set of rules in this proposal.

### `@preconcurrency(unsafe)` to downgrade dynamic actor isolation violations to warnings

If adoption of this feature exposes a bug in existing binaries because actor-isolated code is run outside the actor, a `@preconcurrency(unsafe)` annotation (or similar) could be provided to downgrade assertion failures to warnings. However, it's not clear whether allowing a known data race exhibited at runtime is the right approach to solving such a problem.

## Revision history

* Changes from the first review
  * Insert dynamic checks at direct calls to synchronous actor-isolated functions imported from Swift 6 libraries.
  * Add a flag to disable all dynamic actor isolation checking.

## Acknowledgments

Thank you to Doug Gregor for implementing the existing dynamic actor isolation checking gated behind `-enable-actor-data-race-checks`.
