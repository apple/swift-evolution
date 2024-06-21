# `sending` parameter and result values

* Proposal: [SE-0430](0430-transferring-parameters-and-results.md)
* Authors: [Michael Gottesman](https://github.com/gottesmm), [Holly Borla](https://github.com/hborla), [John McCall](https://github.com/rjmccall)
* Review Manager: [Becca Royal-Gordon](https://github.com/beccadax)
* Status: **Implemented (Swift 6.0)** 
* Previous Proposal: [SE-0414: Region-based isolation](/proposals/0414-region-based-isolation.md)
* Review: ([pitch](https://forums.swift.org/t/pitch-transferring-isolation-regions-of-parameter-and-result-values/70240)) ([first review](https://forums.swift.org/t/se-0430-transferring-isolation-regions-of-parameter-and-result-values/70830)) ([returned for revision](https://forums.swift.org/t/returned-for-revision-se-0430-transferring-isolation-regions-of-parameter-and-result-values/71297)) ([second review](https://forums.swift.org/t/se-0430-second-review-sendable-parameter-and-result-values/71685)) ([acceptance with modifications](https://forums.swift.org/t/accepted-with-modifications-se-0430-second-review-sendable-parameter-and-result-values/71850))


## Introduction

This proposal extends region isolation to enable the application of an explicit
`sending` annotation to function parameters and results. A function parameter
or result that is annotated with `sending` is required to be disconnected at
the function boundary and thus possesses the capability of being safely sent
across an isolation domain or merged into an actor-isolated region in the
function's body or the function's caller respectively.

## Motivation

SE-0414 introduced region isolation to enable non-`Sendable` typed values to be
safely sent over isolation boundaries. In most cases, function argument and
result values are merged together into the same region for any given call. This
means that non-`Sendable` typed parameter values can never be sent:

```swift
// Compiled with -swift-version 6

class NonSendable {}

@MainActor func main(ns: NonSendable) {}

func trySend(ns: NonSendable) async {
  // error: sending 'ns' can result in data races.
  // note: sending task-isolated 'ns' to main actor-isolated 
  //       'main' could cause races between main actor-isolated
  //       and task-isolated uses
  await main(ns: ns)
}
```

Actor initializers have a special rule that requires their parameter values to be
sent into the actor instance's isolation region. Actor initializers are
`nonisolated`, so a call to an actor initializer does not cross an isolation
boundary, meaning the argument values would be usable in the caller after the
initializer returns under the standard region isolation rules. SE-0414 consider
actor initializer parameters as being sent into the actor's region to allow
initializing actor-isolated state with those values:

```swift
class NonSendable {}

actor MyActor {
  let ns: NonSendable
  init(ns: NonSendable) {
    self.ns = ns
  }
}

func send() {
  let ns = NonSendable()
  let myActor = MyActor(ns: ns) // okay; 'ns' is sent into the 'myActor' region
}

func invalidSend() {
  let ns = NonSendable()

  // error: sending 'ns' may cause a data race
  // note: sending 'ns' from nonisolated caller to actor-isolated
  //       'init'. Later uses in caller could race with uses on the actor.
  let myActor = MyActor(ns: ns)

  print(ns) // note: note: access here could race
}
```

In the above code, if the local variable `ns` in the function `send` was instead
a function parameter, it would be invalid to send `ns` into `myActor`'s region
because the caller of `send()` may use the argument value after `send()`
returns:

```swift
func send(ns: NonSendable) {
  // error: sending 'ns' may cause a data race
  // note: task-isolated 'ns' to actor-isolated 'init' could cause races between
  //       actor-isolated and task-isolated uses.
  let myActor = MyActor(ns: ns)
}

func callSend() {
  let ns = NonSendable()
  send(ns: ns)
  print(ns)
}
```

The "sending parameter" behavior of actor initializers is a generally
useful concept, but it is not possible to explicitly specify that functions
and methods can send away specific parameter values. Consider the following
code that uses `CheckedContinuation`:

```swift
@MainActor var mainActorState: NonSendable?

nonisolated func test() async {
  let ns = await withCheckedContinuation { continuation in
    Task { @MainActor in
      let ns = NonSendable()
      // Oh no! 'NonSendable' is passed from the main actor to a
      // nonisolated context here!
      continuation.resume(returning: ns)

      // Save 'ns' to main actor state for concurrent access later on
      mainActorState = ns
    }
  }

  // 'ns' and 'mainActorState' are now the same non-Sendable value;
  // concurrent access is possible!
  ns.mutate()
}
```

In the above code, the closure argument to `withCheckedContinuation` crosses an
isolation boundary to get onto the main actor, creates a non-`Sendable` typed
value, then resumes the continuation with that non-`Sendable` typed value. The
non-`Sendable` typed value is then returned to the original `nonisolated` context,
thus crossing an isolation boundary. Because `resume(returning:)` does not
impose a `Sendable` requirement on its argument, this code does not produce any
data-race safety diagnostics, even under `-strict-concurrency=complete`.

Requiring `Sendable` on the parameter type of `resume(returning:)` is a harsh
restriction, and it's safe to pass a non-`Sendable` typed value as long as the value
is in a disconnected region and all values in that disconnected region are not
used again after the call to `resume(returning:)`.

## Proposed solution

This proposal enables explicitly specifying parameter and result values as
possessing the capability of being sent over an isolation boundary by annotating
the value with a contextual `sending` keyword:

```swift
public struct CheckedContinuation<T, E: Error>: Sendable {
  public func resume(returning value: sending T)
}

public func withCheckedContinuation<T>(
    function: String = #function,
    _ body: (CheckedContinuation<T, Never>) -> Void
) async -> sending T
```

## Detailed design

### Sendable Values and Sendable Types

A type that conforms to the `Sendable` protocol is a thread-safe type: values of
that type can be shared with and used safely from multiple concurrent contexts
at once without causing data races. If a value does not conform to `Sendable`,
Swift must ensure that the value is never used concurrently. The value can still
be sent between concurrent contexts, but the send must be a complete transfer of
the value's entire region implying that all uses of the value (and anything
non-`Sendable` typed that can be reached from the value) must end in the source
concurrency context before any uses can begin in the destination concurrency
context. Swift achieves this property by requiring that the value is in a
disconnected region and we say that such a value is a `sending` value.

Thus a newly-created value with no connections to existing regions is always a
`sending` value:

```swift
func f() async {
  // This is a `sending` value since we can transfer it safely...
  let ns = NonSendable()

  // ... here by calling 'sendToMain'.
  await sendToMain(ns)
}
```

Once defined, a `sending` value can be merged into other isolation
regions. Once merged, such regions, if not disconnected, will prevent the value
from being sent to another isolation domain implying that the value is no longer
a `sending` value:

```swift
actor MyActor {
  var myNS: NonSendable

  func g() async {
    // 'ns' is initially a `sending` value since it is in a disconnected region...
    let ns = NonSendable()

    // ... but once we assign 'ns' into 'myNS', 'ns' is no longer a sending
    // value...
    myNS = ns

    // ... causing calling 'sendToMain' to be an error.
    await sendToMain(ns)
  }
}
```

If a `sending` value's isolation region is merged into another disconnected
isolation region, then the value is still considered to be `sending` since two
disconnected regions when merged form a new disconnected region:

```swift
func h() async {
  // This is a `sending` value.
  let ns = Nonsending()

  // This also a `sending value.
  let ns2 = NonSendable()

  // Since both ns and ns2 are disconnected, the region associated with
  // tuple is also disconnected and thus 't' is a `sending` value...
  let t = (ns, ns2)

  // ... that can be sent across a concurrency boundary safely.
  await sendToMain(ns)
}
```

### sending Parameters and Results

A `sending` function parameter requires that the argument value be in a
disconnected region. At the point of the call, the disconnected region is no
longer in the caller's isolation domain, allowing the callee to send the
parameter value to a region that is opaque to the caller:

```swift
@MainActor
func acceptSend(_: sending NonSendable) {}

func sendToMain() async {
  let ns = NonSendable()

  // error: sending 'ns' may cause a race
  // note: 'ns' is passed as a 'sending' parameter to 'acceptSend'. Local uses could race with
  //       later uses in 'acceptSend'.
  await acceptSend(ns)

  // note: access here could race
  print(ns)
}
```

What the callee does with the argument value is opaque to the caller; the callee
may send the value away, or it may merge the value to the isolation region of
one of the other parameters.

A `sending` result requires that the function implementation returns a value in
a disconnected region:

```swift
@MainActor
struct S {
  let ns: NonSendable

  func getNonSendableInvalid() -> sending NonSendable {
    // error: sending 'self.ns' may cause a data race
    // note: main actor-isolated 'self.ns' is returned as a 'sending' result.
    //       Caller uses could race against main actor-isolated uses.
    return ns
  }

  func getNonSendable() -> sending NonSendable {
    return NonSendable() // okay
  }
}
```

The caller of a function returning a `sending` result can assume the value is
in a disconnected region, enabling non-`Sendable` typed result values to cross
an actor isolation boundary:

```swift
@MainActor func onMain(_: NonSendable) { ... }

nonisolated func f(s: S) async {
  let ns = s.getNonSendable() // okay; 'ns' is in a disconnected region

  await onMain(ns) // 'ns' can be sent away to the main actor
}
```

### Function subtyping

For a given type `T`, `sending T` is a subtype of `T`. `sending` is
contravariant in parameter position; if a function type is expecting a regular
parameter of type `T`, it's perfectly valid to pass a `sending T` value
that is known to be in a disconnected region. If a function is expecting a
parameter of type `sending T`, it is not valid to pass a value that is not
in a disconnected region:

```swift
func sendingParameterConversions(
  f1: (sending NonSendable) -> Void,
  f2: (NonSendable) -> Void
) {
  let _: (sending NonSendable) -> Void = f1 // okay
  let _: (sending NonSendable) -> Void = f2 // okay
  let _: (NonSendable) -> Void = f1 // error
}
```

`sending` is covariant in result position. If a function returns a value
of type `sending T`, it's valid to instead treat the result as if it were
merged with the other parameters. If a function returns a regular value of type
`T`, it is not valid to assume the value is in a disconnected region:

```swift
func sendingResultConversions(
  f1: () -> sending NonSendable,
  f2: () -> NonSendable
) {
  let _: () -> sending NonSendable = f1 // okay
  let _: () -> sending NonSendable = f2 // error
  let _: () -> NonSendable = f1 // okay
}
```

### Protocol conformances

A protocol requirement may include `sending` parameter or result annotations:

```swift
protocol P1 {
  func requirement(_: sending NonSendable)
}

protocol P2 {
  func requirement() -> sending NonSendable
}
```

Following the function subtyping rules in the previous section, a protocol
requirement with a `sending` parameter may be witnessed by a function with a
non-`Sendable` typed parameter:

```swift
struct X1: P1 {
  func requirement(_: sending NonSendable) {}
}

struct X2: P1 {
  func requirement(_: NonSendable) {}
}
```

A protocol requirement with a `sending` result must be witnessed by a function
with a `sending` result, and a requirement with a plain result of type `T` may
be witnessed by a function returning a `sending T`:

```swift
struct Y1: P1 {
  func requirement() -> sending NonSendable {
    return NonSendable()
  }
}

struct Y2: P1 {
  let ns: NonSendable
  func requirement() -> NonSendable { // error
    return ns
  }
}
```

### `sending inout` parameters

A `sending` parameter can also be marked as `inout`, meaning that the argument
value must be in a disconnected region when passed to the function, and the
parameter value must be in a disconnected region when the function
returns. Inside the function, the `sending inout` parameter can be merged with
actor-isolated callees or further sent as long as the parameter is
re-assigned a value in a disconnected region upon function exit.

### Ownership convention for `sending` parameters

When a call passes an argument to a `sending` parameter, the caller cannot
use the argument value again after the callee returns. By default `sending`
on a function parameter implies that the callee consumes the parameter. Like
`consuming` parameters, a `sending` parameter can be re-assigned inside
the callee. Unlike `consuming` parameters, `sending` parameters do not
have no-implicit-copying semantics.

To opt into no-implicit-copying semantics or to change the default ownership
convention, `sending` may also be paired with an explicit `consuming` or
`borrowing` ownership modifier:

```swift
func sendingConsuming(_ x: consuming sending T) { ... }
func sendingBorrowing(_ x: borrowing sending T) { ... }
```

Note that an explicit `borrowing` annotation always implies no-implicit-copying,
so there is no way to change the default ownership convention of a
`sending` parameter without also opting into no-implicit-copying semantics.

### Adoption in the Concurrency library

There are several APIs in the concurrency library that send a parameter across
isolation boundaries and don't need the full guarnatees of `Sendable`.  These
APIs will instead adopt `sending` parameters:

* `CheckedContinuation.resume(returning:)`
* `Async{Throwing}Stream.Continuation.yield(_:)`
* `Async{Throwing}Stream.Continuation.yield(with:)`
* The `Task` creation APIs

Note that this list does not include `UnsafeContinuation.resume(returning:)`,
because `UnsafeContinuation` deliberately opts out of correctness checking.

## Source compatibility

In the Swift 5 language mode, `sending` diagnostics are suppressed under
minimal concurrency checking, and diagnosed as warnings under strict concurrency
checking. The diagnostics are errors in the Swift 6 language mode, as shown in
the code examples in this proposal. This diagnostic behavior based on language
mode allows `sending` to be adopted in existing Concurrency APIs including
`CheckedContinuation`.

## ABI compatibility

This proposal does not change how any existing code is compiled.

## Implications on adoption

Adding `sending` to a parameter is more restrictive at the caller, and
more expressive in the callee. Adding `sending` to a result type is more
restrictive in the callee, and more expressive in the caller.

For libraries with library evolution, `sending` changes name mangling, so
any adoption must preserve the mangling using `@_silgen_name`. Adoping
`sending` must preserve the ownership convention of parameters; no
additional annotation is necessary if the parameter is already (implicitly or
explicitly) `consuming`.

## Future directions

### `Disconnected` types

`sending` requires parameter and result values to be in a disconnected
region at the function boundary, but there is no way to preserve that a value
is in a disconnected region through stored properties, collections, function
calls, etc. To preserve that a value is in a disconnected region through the
type system, we could introduce a `Disconnected` type into the Concurrency
library. The  `Disconnected` type would suppress copying via `~Copyable`, it
would conform to `Sendable`, constructing a `Disconnected` instance would
require the value it wraps to be in a disconnected region, and a value of type
`Disconnected` can never be merged into another isolation region.

This would enable important patterns that take a `sending T` parameter, store
the value in a collection of `Disconnected<T>`, and later remove values from the
collection and return them as `sending T` results. This would allow some
`AsyncSequence` types to return non-`Sendable` typed buffered elements as
`sending` without resorting to unsafe opt-outs in the implementation.

## Alternatives considered

### Use `transferring` or `sending` instead of `sendable`

This proposal originally used the word `transferring` for `sendable`. The idea
was that this would superficially match parameter modifiers like `consuming` and
`borrowing`. But, this ignored that we are not actually `transferring` the
parameter into another isolation domain at the function boundary point. Instead,
we are requiring that the value at that point be in a disconnected region and
thus have the _capability_ to be sent to another isolation domain or merged into
actor isolated state. This is in contrast to `consuming` and `borrowing` which
actively affect the value at the function boundary point by consuming or
borrowing the value. Additionally, by using `transferring` would introduce a new
term of art into the language unnecessarily and contrasts with already
introduced terms like `@Sendable` and the `Sendable` protocol.

It was also suggested that perhaps instead of renaming `transferring` to
`sendable`, it should have been renamed to `sending`. This was rejected by the
authors since it runs into the same problem as `transferring` namely that it is
suggesting that the value is actively being moved to another isolation domain,
when we are expressing a latent capability of the value.
