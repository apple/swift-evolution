# `@isolated(any)` Function Types

* Proposal: [SE-0431](0431-isolated-any-functions.md)
* Authors: [John McCall](https://github.com/rjmccall)
* Review Manager: [Doug Gregor](https://github.com/DougGregor)
* Status: **Implemented (Swift 6.0)**
* Previous revision: [1](https://github.com/swiftlang/swift-evolution/blob/b35498bf6f198477be50809c0fec3944259e86d0/proposals/0431-isolated-any-functions.md)
* Review: ([pitch](https://forums.swift.org/t/isolated-any-function-types/70562))([review](https://forums.swift.org/t/se-0431-isolated-any-function-types/70939))([acceptance](https://forums.swift.org/t/accepted-with-modifications-se-0431-isolated-any-function-types/71611))

[SE-0316]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0316-global-actors.md
[SE-0392]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0392-custom-actor-executors.md
[isolated-captures]: https://forums.swift.org/t/closure-isolation-control/70378
[generalized-isolation]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0420-inheritance-of-actor-isolation.md#generalized-isolation-checking
[regions]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0414-region-based-isolation.md
[region-transfers]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0430-transferring-parameters-and-results.md

## Introduction

The actor isolation of a function is an important part of how it's
used.  Swift can reason precisely about the isolation of a specific
function *declaration*, but when functions are passed around as
*values*, Swift's function types are not expressive enough to keep up.
This proposal adds a new kind of function type that carries its function's
actor isolation dynamically.  This solves a variety of expressivity
problems in the language.  It also allows features such as the standard
library's task-creation APIs to be implemented more efficiently and
with stronger semantic guarantees.

## Motivation

The safety of Swift concurrency relies on understanding the isolation
requirements of functions.  The caller of an isolated synchronous
function must run it in an appropriately-isolated context or else the
function will almost certainly introduce data races.

Function declarations and closures in Swift support three different
forms of actor isolation:

- They can be non-isolated.
- They can be isolated to a specific [global actor][SE-0316] type.
- They can be isolated to a specific parameter or captured value.

A function's isolation can be specified or inferred in many ways.
Non-isolation is the default if no other rules apply, and it can also
be specified explicitly with the `nonisolated` modifier.  Global actor
isolation can be expressed explicitly with a global actor attribute,
such as `@MainActor`, but it can also be inferred from context, such
as in the methods of main-actor-isolated types.  A function can
explicitly declare one of its parameters as `isolated` to isolate
itself to the value of that parameter; this is also done implicitly to
the `self` parameter of an actor method if the method doesn't explicitly
use some other isolation.  Closure expressions can be declared with a
global actor attribute, and there is a [proposal currently being
developed][isolated-captures] to also allow them to have an explicit
`isolated` capture or to be explicitly non-isolated.  Additionally,
when you pass a closure expression directly to the `Task` initializer,
that closure is inferred to have the isolation of the enclosing context.[^1]
These rules are fairly complex, but at the end of the day, they all
boil down to this: every function is assigned one of the three kinds of
actor isolation above.

[^1]: Currently, if the enclosing context is isolated to a value, the
closure is only isolated to it if it actually captures that value (by
using it somewhere in its body).  This is often seen as confusing, and
the `isolated` captures proposal is considering lifting this restriction
by unconditionally capturing the value.

When a function is called directly, Swift's isolation checker can
analyze its isolation precisely and compare that to the isolation of the
calling context.  However, when a call expression calls an opaque value
of function type, Swift is limited by what can be expressed in the type
system:

- A function type with no isolation specifiers, such as `() -> Int`,
  represents a non-isolated function.

- A function type with a global actor attribute, such as
  `@MainActor () -> Int`, represents a function that's isolated to that
  global actor.

- A function type with an `isolated` parameter, such as
  `(isolated MyActor) - > Int`, represents a function that's isolated to
  that parameter.

But there's a very important case that can't be expressed in the type
system like this: a closure can be isolated to one of its captures.  In
the following example, the closure is isolated to its captured `self`
value:

```swift
actor WorldModelObject {
  var position: Point3D

  func linearMove(to finalPosition: Point3D, over time: Duration) {
    let originalPosition = self.position
    let motion = finalPosition - originalPosition

    gradually(over: time) { [isolated self] progressProportion in
      self.position = originalPosition + progressProportion * motion
    }
  }

  func updateLater() {
    Task {
      // This closure doesn't have an explicit isolation
      // specification, and it's being passed to the `Task`
      // initializer, so it will be inferred to have the same
      // isolation as its enclosing context.  The enclosing
      // context is isolated to its `self` parameter, which this
      // closure captures, so this closure will also be isolated
      // that value.
      self.update()
    }
  }
}
```

This inexpressible case also arises with a partial application of an
actor method, such as `myActor.methodName`: the resulting function
value captures `myActor` and is isolated to it.  For now, these are
the only two cases of isolated captures.  However, the upcoming
[closure isolation control][isolated-captures] proposal is expected
to give this significantly greater prominence and importance.  Under
that proposal, isolated captures will become a powerful general tool
for controlling the isolation of a specific piece of code.  But there
will still not be a way to express the isolation of that closure in
the type system.[^2]

[^2]: Expressing this exactly would require the use of value-dependent
types.  Value dependence is an advanced type system feature that we
cannot easily add to Swift.  This is discussed in greater depth in
the Future Directions section.

This is a very unfortunate limitation, because it actually means that
there's no way for a function to accept a function argument with
arbitrary isolation without completely erasing that isolation.  Swift
does allow functions with arbitrary isolation to be converted to a
non-isolated function type, but this comes with three severe drawbacks.
The first is that the resulting function type must be `async` so that
it can switch to the right isolation internally.  The second is that,
because the function changes isolation internally, it is limited in its
ability to work with non-`Sendable` values because any argument or return
value must cross an isolation boundary.  And the third is that the
isolation is completely dynamically erased: there is no way for the
recipient of the function value to recover what isolation the function
actually wants, which often puts the recipient in the position of doing
unnecessary work.

Here's an example of that last problem.  The `Task` initializer receives
an opaque value of type `() async throws -> ()`.  Because it cannot
dynamically recover the isolation from this value, the initializer has
no choice but to start the task on the global concurrent executor.  If
the function passed to the initializer is actually isolated to an actor,
it will immediately switch to that actor on entry.  This requires
additional synchronization and may require re-suspending the task.
Perhaps more importantly, it means that the order in which tasks are
actually enqueued on the actor is not necessarily the same as the order
in which they were created.  It would be much better --- both semantically
and for performance --- if the initializer could immediately enqueue the
task on the right executor to begin with.

The straightforward solution to these problems is to add a type which
is capable of expressing a function with an arbitrary (but statically
unknown) isolation.  That is what we propose to do.

## Proposed solution

This proposal adds a new attribute that can be placed on function types:

```swift
func gradually(over: Duration, operation: @isolated(any) (Double) -> ())
```

A function value with this type dynamically carries the isolation of
the function that was used to initialize it.

When such a function is called from an arbitrary context, it must be
assumed to always cross an isolation boundary.  This means, among other
things, that the call is effectively asynchronous and must be `await`ed.

```swift
await operation(timePassed / overallDuration)
```

The isolation can be read using the special `isolation` property
of these types:

```swift
func traverse(operation: @isolated(any) (Node) -> ()) {
  let isolation = operation.isolation
}
```

The isolation checker knows that the value of this special property
matches the isolation of the function, so calls to the function from
contexts that are isolated to the `isolation` value do not cross
an isolation boundary.

Finally, every task-creation API in the standard library will be updated
to take a `@isolated(any)` function value and synchronously enqueue the
new task on the appropriate executor.

## Detailed design

### Grammar and structural rules

`@isolated(any)` is a new type attribute that can only be applied to
function types.  It is an isolation specification, and it is an error
to combine it with other isolation specifications such as a global
actor attribute or an `isolated` parameter.

`@isolated(any)` is not a *concrete* isolation specification and cannot
be directly applied to a declaration or a closure.  That is, you cannot
declare a function *entity* as having `@isolated(any)` isolation,
because Swift needs to know what the actual isolation is, and
`@isolated(any)` does not provide a rule for that.

### Conversions

Let `F` and `G` be function types, and let `F'` and `G'` be the corresponding
function types with any isolation specifier removed (including but not
limited to `@isolated(any)`.  If either `F` or `G` specifies
`@isolated(any)` then a value of type `F` can be converted to type `G`
if a value of type `F'` could be converted to type `G'` *and* the
following conditions apply:

- If `F` and `G` both specify `@isolated(any)`, there are no further
  conditions.  The resulting function is dynamically isolated to the
  same value as the original function.

- If only `G` specifies `@isolated(any)`, then the behavior depends on the
  specified isolation of `F`:

  - If `F` has an `isolated` parameter, the conversion is invalid.
  - Otherwise, the conversion is valid, and the dynamic isolation of
    the resulting function is determined as follows:
    - If the converted value is the result of an expression that is a
      closure expression (including an implicit autoclosure), a function
      reference, or a partial application of a method reference, the
      resulting function is dynamically isolated to the isolation of the
      function or closure.  This looks through syntax that has no impact
      on the value produced by the expression, such as parentheses; the
      list is the same as in [SE-0420][generalized-isolation].
    - Otherwise, if `F` is non-isolated, the resulting function is
      dynamically non-isolated.
    - Otherwise, `F` must be isolated to a global actor, and the resulting
      function is dynamically isolated to that global actor.

- If only `F` specifies `@isolated(any)`, then `G` must be an `async` function
  type.  `G` may have any isolation specifier, but it will be ignored and the
  function will run with the isolation of the original value.  The arguments
  and result must be sendable across an isolation boundary.  It is unspecified
  whether the task will dynamically suspend when calling or returning from
  the resulting value.

### Effects of intermediate conversions

In general, all of the isolation semantics and runtime behaviors laid out
here are affected by intermediate conversions to non-`@isolated(any)`
function types.  For example, if you coerce a non-isolated function or
closure to the type `@MainActor () -> ()`, the resulting function will
thereafter be treated as a `MainActor`-isolated function; the fact that
it was originally a non-isolated function is both statically and
dynamically erased.

### Runtime behavior

Calling a `@isolated(any)` function value behaves the same way as a direct
call to a function with that isolation would:

- If the function is `async`, it will run with its formal isolation.  This
  includes leaving isolated contexts if the function is dynamically
  non-isolated, as specified by [SE-0338](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0338-clarify-execution-non-actor-async.md).

- If the function is synchronous, it will run with its formal isolation
  only if it is dynamically isolated.  If it is dynamically non-isolated,
  it will simply run synchronously in the current context, even if that
  is isolated, just like an ordinary call to a non-isolated synchronous
  function would.

### `isolation` property

Values of `@isolated(any)` function type have a special `isolation`
property.  The property is read-only and has type `(any Actor)?`.  The
value of the property is determined by the dynamic isolation of the
function value:

- If the function is dynamically non-isolated, the value of `isolation`
  is `nil`.
- If the function is dynamically isolated to a global actor type `G`,
  the value of `isolation` is `G.shared`.
- If the function is dynamically isolated to a specific actor reference,
  the value of `isolation` is that actor reference.

### Distributed actors

Function values cannot generally be isolated to a distributed actor
unless the actor is known to be local.  When a distributed actor *is*
local, function values isolated to the actor can be converted to
`@isolated(any)` type as above.  The `isolation` property presents
the distributed actor as an `(any Actor)?` using the same mechanism
as [`#isolation`](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0420-inheritance-of-actor-isolation.md#isolated-distributed-actors).

### Isolation checking

Since the isolation of an `@isolated(any)` function value is
statically unknown, calls to it typically cross an isolation boundary.
This means that the call must be `await`ed even if the function is
synchronous, and the arguments and result must satisfy the usual
sendability restrictions for cross-isolation calls.  The function
value itself must satisfy a slightly less restrictive rule: it must
be a sendable value only if it is `async` and the current
context is not statically known to be non-isolated.[^4]

[^4]: The reasoning here is as follows.  All actor-isolated functions
are inherently `Sendable` because they will only use their captures from
an isolated context.[^5]  There is only a data-race risk for the
captures of a non-`Sendable` `@isolated(any)` function in the case
where the function is dynamically non-isolated.  The sendability
restrictions therefore boil down to the same restrictions we would
impose on calling a non-isolated function.  A call to a non-isolated
function never crosses an isolation boundary if the function is
synchronous or if the current context is non-isolated.

[^5]: Sending an isolated function value may cause its captures to be
*destroyed* in a different context from the function's formal isolation.
Swift pervasively assumes this is okay: copies of non-`Sendable` values
must still be managed in a thread-safe manner.  This is a significant
departure from Rust, where non-`Send` values cannot necessarily be safely
managed concurrently, and it means that `Sendable` is not sufficient
to enable optimizations like non-atomic reference counting.  Swift
accepts this in exchange for being more permissive, as long as the code
avoids "user-visible" data races.  Note that this assumption is not new
to this proposal.

In order for a call to an `@isolated(any)` function to be treated as
not crossing an isolation boundary, the caller must be known to have
the same isolation as the function.  Since the isolation of an
`@isoalted(any)` parameter is necessarily an opaque value, this would
require the caller to be declared with value-specific isolation.  It
is currently not possible for a local function or closure to be
isolated to a specific value that isn't already the isolation of the
current context.[^6]  The following rules lay out how `@isolated(any)`
should interact with possible future language support for functions
that are explicitly isolated to a captured value.  In order to
present these rules, this proposal uses the syntax currently proposed
by the [closure isolation control pitch][isolated-captures], where
putting `isolated` before a capture makes the closure isolated to
that value.  This should not be construed as accepting the terms of
that pitch.  Accepting this proposal will leave most of this
section "suspended" until a feature with a similar effect is added
to the language.

[^6]: Technically, it is possible to achieve this effect in Swift
today in a way that Swift could conceivably look through: the caller
could be a closure with an `isolated` parameter, and that closure
could be called with an expression like `fn.isolation` as the argument.
Swift could analyze this to see that the parameter has the value of
`fn.isolation` and then understand the connection between the caller's
isolation and `fn`.  This would be very cumbersome, though, and it
would have significant expressivity gaps vs. an isolated-captures
feature.

If `f` is an immutable binding of `@isolated(any)` function type,
then a call to `f` does not cross an isolation boundary if the
current context is isolated to a *derivation* of the expression
`f.isolation`.

In the isolated captures pitch, a closure can be isolated to a specific
value by using the `isolated` modifier on an entry in its capture list.
So this question would reduce to whether that capture was initialized
to a derivation of `f.isolation`.

An expression is a derivation of some expression form `E` if:

- it has the exact form required by `E`;
- it is a reference to a capture or immutable binding immediately
  initialized with a derivation of `E`;
- it is the result of `?` (the optional-chaining operator) or `!`
  (the optional-forcing operator) applied to a derivation of `E`; or
- it is a reference to a non-optional binding (an immutable binding
  initialized by a successful pattern-match which removes optionality,
  such as `x` in `if let x = E`) of a derivation of `E`.

The term *immutable binding* in the rules above means a `let` constant
or immutable (non-`inout`) parameter that is neither `weak` nor
`unowned`.  The analysis ignores syntax that has no effect on the
value of an expression, such as parentheses; the exact set of cases
are the same as described in [SE-0420][generalized-isolation].

For example:

```swift
func delay(operation: @isolated(any) () -> ()) {
  let isolation = operation.isolation
  Task { [isolated isolation] in // <-- tentative syntax from the isolated captures pitch
    print("waking")
    operation() // <-- does not cross an isolation barrier and so is synchronous
    print("finished")
  }
}
```

In this example, the expression `operation()` calls `operation`,
which is an immutable binding (a parameter) of `@isolated(any)`
function type.  The call therefore does not cross an isolation
boundary if the calling context is isolated to a derivation of
`operation.isolation`.  The calling context is the closure passed
to `Task.init`, which has an explicit `isolated` capture named
`isolation` and so is isolated to that value of that capture.
The capture is initialized with the value of the enclosing
variable `isolation`, which is an immutable binding (a `let`
constant) initialized to `operation.isolation`.  As such, the
calling context is isolated to a derivation of `operation.isolation`,
so the call does not cross an isolation boundary.

The primary intent of the rules above is simply to extend the
generalized isolation checking rules laid out in
[SE-0420][generalized-isolation] to work with an underlying
expression like `fn.isolation`.  However, the rules above go
beyond the SE-0420 rules in some ways, most importantly by looking
through local `let`s.  Looking through such bindings was not especially
important for SE-0420, but it is important for this proposal.  In
order to keep the rules consistent, the isolation checking rules from
SE-0420 will be "rebased" on top of the rules in this proposal,
as follows:

- When calling a function with an `isolated` parameter `calleeParam`,
  if the current context also has an `isolated` parameter or capture
  `callerIsolation`, the function has the same isolation as the current
  context if the argument expression corresponding to `calleeParam` is
  a derivation of either:

  - a reference to `callerIsolation` or
  - a call to `DistributedActor.asAnyActor` applied to a derivation of
    `calleeIsolation`.

As a result, the following code is now well-formed:

```swift
func operate(actor1: isolated MyActor) {
  let actor2 = actor1
  actor2.isolatedMethod() // Swift now knows that actor2 is isolated
}
```

There is no reason to write this code instead of just using `actor1`,
but it's good to have consistent rules.

### Adoption in task-creation routines

There are a large number of functions in the standard library that create
tasks:
- `Task.init`
- `Task.detached`
- `TaskGroup.addTask`
- `TaskGroup.addTaskUnlessCancelled`
- `ThrowingTaskGroup.addTask`
- `ThrowingTaskGroup.addTaskUnlessCancelled`
- `DiscardingTaskGroup.addTask`
- `DiscardingTaskGroup.addTaskUnlessCancelled`
- `ThrowingDiscardingTaskGroup.addTask`
- `ThrowingDiscardingTaskGroup.addUnlessCancelled`

This proposal modifies all of these APIs so that the task function has
`@isolated(any)` function type.  These APIs now all synchronously enqueue
the new task directly on the appropriate executor for the task function's
dynamic isolation.

Swift reserves the right to optimize the execution of tasks to avoid
"unnecessary" isolation changes, such as when an isolated `async` function
starts by calling a function with different isolation.[^3] In general, this
includes optimizing where the task initially starts executing:

```swift
@MainActor class MyViewController: UIViewController {
  @IBAction func buttonTapped(_ sender : UIButton) {
    Task {
      // This closure is implicitly isolated to the main actor, but Swift
      // is free to recognize that it doesn't actually need to start there.
      let image = await downloadImage()
      display.showImage(image)
    }
  }
}
```

[^3]: This optimization doesn't change the formal isolation of the functions
involved and so has no effect on the value of either `#isolation` or
`.isolation`.

As an exception, in order to provide a primitive scheduling operation with
stronger guarantees, Swift will always start a task function on the
appropriate executor for its formal dynamic isolation unless:
- it is non-isolated or
- it comes from a closure expression that is only *implicitly* isolated
  to an actor (that is, it has neither an explicit `isolated` capture
  nor a global actor attribute).  This can currently only happen with
  `Task {}`.

As a result, in the following code, these two tasks are guaranteed
to start executing on the main actor in the order in which they were
created, even if they immediately switch away from the main actor without
having done anything that requires isolation:[^4]

```swift
func process() async {
  Task { @MainActor in
    ...
  }

  // do some work

  Task { @MainActor in
    ...
  }
}
```


[^4]: This sort of guarantee is important when working with a FIFO
"pipeline", which is a common pattern when working with explicit queues.
In a pipeline, code responds to an event by performing work on a series
of queues, like so:

    ```swift
    func handleEvent(event: Event) {}
      queue1.async {
        let x = makeX(event)
        queue2.async {
          let y = makeY(event)
          queue3.async {
            handle(x, y)
          }
        }
      }
    }
    ```
    
    As long as execution always goes through the exact same sequence of FIFO
    queues, each queue will execute its stage of the overall pipeline in
    the same order as the events were originally received.  This can be a
    difficult property to maintain --- concurrency at any stage will destroy
    it, as will skipping any stages of the pipeline --- but it's not uncommon
    for systems to be architected around it.

The exception here to allow more optimization for implicitly-isolated
closures is an effort to avoid turning `Task {}` into a surprising
performance bottleneck.  Programmers often reach for `Task {}` just to
do something concurrently with the current context, such as downloading
a file from the internet and then storing it somewhere.  However, if
`Task {}` is used from an isolated context (such as from a `@MainActor`
event handler), the closure passed to `Task` will implicitly formally
inherit that isolation.  A strict interpretation of the scheduling
guarantee in this proposal would require the closure to run briefly
on the current actor before it could do anything else.  That would mean
that the task could never begin the download immediately; it would have
to wait, not just for the current operation on the actor to finish, but
for the actor to finish processing everything else currently in its
queue.  If this is needed, it is not unreasonable to ask programmers
to state it explicitly, just as they would have to from a non-isolated
context.

## Source compatibility

Most of this proposal is additive.  The exception is the adoption
in the standard library, which changes the types of certain API
parameters.  Calls to these APIs should continue to work, as any
function that could be passed to the current parameter should also
be convertible to an `@isolated(any)` type.  The observed type of
the API will change, however, if anyone does an abstract reference
such as `Task.init`.  Contravariant conversion should allow these
unapplied references to work in any concrete type context that
would accept the current function, but references in other contexts
can lead to source breaks (such as `var fn = Task.init`).  This is
unlikely to be an issue in practice.  More importantly, I believe
Swift has a general policy of declining to guarantee stable types
for unapplied function references in the standard library this way.
Doing so would prevent a wide variety of reasonable code evolution
for the library, such as generalizing the type of a parameter (as
this proposal does) or adding a new defaulted parameter.

## ABI compatibility

This feature does not change the ABI of any existing code.

## Implications on adoption

The basic functionality of `@isolated(any)` function types is
implemented directly in generated code and does not require runtime
support.

Using a type as a generic argument generally requires runtime type
metadata support for the type.  For `@isolated(any)` function types,
that metadata support requires a new Swift runtime.  It will therefore
not possible to use a type such as `[@isolated(any) () -> ()]` when
back-deploying code on a platform with ABI stability.  However,
wrapping the function in a `struct` with a single field will generally
work around this problem.  (It also generally allows the function to
be stored more efficiently.)

The task-creation APIs in the standard library have been implemented
in a way that allows their signatures to be changed without ABI
considerations.  Direct enqueuing on the isolated actor does require
runtime support, but fortunately that support has present in the
concurrency runtime since the first release.  Therefore, there should
not be any back-deployment problems supporting the proposed changes.

Adopters of `@isolated(any)` function types will generally face the
same source-compatibility considerations as this proposal does with
the task-creation APIs: it requires generalizing some parameter types,
which generally should not cause incompatibilities with direct callers
but can introduce problems in the somewhat unlikely case that anyone
is using those function as values.

### When to use `@isolated(any)`

It is recommended that APIs which take functions that are likely to run
concurrently and don't have a predetermined isolation take those functions
as `@isolated(any)`.  This allows the API to make more intelligent
scheduling decisions about the function.

Examples that should usually use `@isolated(any)` include:
- functions that wrap the creation of a task
- algorithms that call a function multiple times in parallel, such as a
  parallel `map`

Examples that should usually not use `@isolated(any)` include:
- algorithms that preserve the current isolation, such as a non-parallel
  `map`; these functions should usually take a non-`Sendable` function
  instead
- APIs that intend to call the function with a specific isolation, such
  as UI frameworks that expect their event handlers to be `@MainActor`
  or actor functions that run an operation on the actor

## Future directions

### Interaction with `assumeIsolated`

It would be convenient in some cases to be able to assert that the
current synchronous context is already isolated to the isolation of
an `@isolated(any)` function, allowing the function to be called without
crossing isolation.  Similar functionality is provided by the
`assumeIsolated` function introduced by [SE-0392][SE-0392].
Unfortunately, the current `assumeIsolated` function is inadequate
for this purpose for several reasons.

The first problem is that `assumeIsolated` only works on a
non-optional actor reference.  We could add a version of this API
which does work on optional actors, but it's not clear what it
should actually do if given a `nil` reference.  A `nil` isolation
represents non-isolation, which of course does not actually isolate
anything.  Should `assumeIsolated` check that the current context
has exactly the given isolation, or should it check that it is safe
to use something with the given isolation requirement from the current
context?  The first rule is probably the one that most people would
assume when they first heard about the feature.  However, it implies
that `assumeIsolated(nil)` should check that no actors are currently
isolated, and that is not something we can feasibly check in general:
Swift's concurrency runtime does track the current isolation of a task,
but outside of a task, arbitrary things can be isolated without Swift
knowing about them.  It is also needlessly restrictive, because there
is nothing that is unsafe to do in an isolated context that would be
safe if done in a non-isolated context.[^7]  The second rule is less
intuitive but more closely matches the safety properties that static
isolation checking tests for.  It implies that `assumeIsolated(nil)`
should always succeed.  This is notably good enough for `@isolated(any)`:
since `assumeIsolated` is a synchronous function, only synchronous
`@isolated(any)` functions can be called within it, and calling a
synchronous non-isolated function always runs immediately without
changing the current isolation.

[^7]: As far as data-race safety goes, at least.  A specific actor
could conceivably have important semantic restrictions against doing
certain operations in its isolated code.  Of course, such an actor should
generally not be calling arbitrary functions that are handed to it.

The second problem is that `assumeIsolated` does not currently establish
a link back to the original expression passed to it.  Code such as
the following is invalid:

```swift
myActor.assumeIsolated {
  myActor.property += 1   // invalid: Swift doesn't know that myActor is isolated
}
```

The callback passed to `assumeIsolated` is isolated because it takes
an `isolated` parameter, and while this parameter is always bound to
the actor that `assumeIsolated` was called on, Swift's isolation checking
doesn't know that.  As a result, it is necessary to use the parameter
instead of the original actor reference, which is a persistent annoyance
when using this API:

```swift
myActor.assumeIsolated { myActor2 in
  myActor2.property += 1
}
```

For `@isolated(any)`, we would naturally want to write this:

```swift
myFn.isolation.assumeIsolated {
  myFn()
}
```

However, since Swift doesn't understand the connection between the
closure's `isolated` parameter and `myFn`, this call will not work,
and there is no way to make it work.

One way to fix this would be to add some new way to assert that an
`@isolated(any)` function is currently isolated.  This could even
destructure the function value, giving the program access it to as
a non-`@isolated(any)` function.  But it seems like a better approach
to allow isolation checking to understand that the `isolated` parameter
and the `self` argument of `assumeIsolated` are the same value.
That would fix both the usability problem with actors and the
expressivity problem with `@isolated(any)`.  Decomposition could
be done as a general rule that permits isolation to be removed from
a function value as long as that isolation matches the current
context and the resulting function is non-`Sendable`.

This is all sufficiently complex that it seems best to leave it for
a future direction.  However, it should be relatively approachable.

### Statically-isolated function types

`@isolated(any)` function types are effectively an "existential
erasure" of the isolation of the function, removing the type system's
static knowledge of the isolation while dynamically preserving it.
This is directly analogous to how `Any` erases the type of the value
you store into it: the type system no longer knows statically what
type is stored there, but it's still possible to recover it dynamically.
This analogy is why this proposal uses the keyword `any` in the
attribute name.

Where there's an existential, there's also a generic.  The generic
analogue to `@isolated(any)` would be a type that expressed that it
was isolated to a specific value, like so:

```swift
func delay<A: Actor>(on operationActor: A,
                     operation: @isolated(to: operationActor) () async -> ())
```

This is a kind of value-dependent type.  Value-dependent types add a
lot of complexity to a type system.  Consider how the arguments interact
in the example above: both value and type information from the first
argument flows into the second.  This is not something to do lightly,
and we think Swift is relatively unlikely to ever add such a feature
as `@isolated(to:)`.

Fortunately, it is unlikely to be necessary.  We believe that
`@isolated(any)` function types are superior from a usability perspective
for all the dominant patterns of higher-order APIs.  The main thing that
`@isolated(to:)` can express in an API signature that `@isolated(any)`
cannot is multiple functions that share a common isolation.  It is
quite uncommon for APIs to take multiple closely-related functions
this way, especially `@Sendable` functions where there's an expected
isolation change from the current context.  When only a single function
is required in an API, `@isolated(any)` allows its isolation to bound
up with it in a single value, which is both more convenient and likely
to have a more performant representation.

If Swift ever does explore in the direction of `@isolated(to:)`,
nothing in this proposal would interfere with it.  In fact, the
features would support each other well.  Erasing the isolation of
an `@isolated(to:)` function into an `@isolated(any)` type would
be straightforward, much like erasing an `Int` into an `Any`.
Similarly, an `@isolated(any)` function could be "opened" into a
pair of an `@isolated(to:)` function and its known isolation.
Since the common cases will still be more convenient to express
with `@isolated(any)`, the community is unlikely to regret having
added this proposal first.

## Alternatives considered

### Other spellings

`isolated` and `nonisolated` are used as bare-word modifiers in several
places already in Swift: you can declare a parameter as `isolated`, and
you can declare methods and properties as `nonisolated`.  Using `@isolated`
as a function type attribute therefore risks confusion about whether
`isolated` should be written with an `@` sign.

One alternative would be to drop the `@` sign and spell these function
types as e.g. `isolated(any) () -> ()`.  However, this comes with its own
problems.  Modifiers typically affect a specific entity without changing
its type; for example, the `weak` modifier makes a variable or property
a weak reference, but the type of that reference is unchanged (although
it is required to be optional).  This wouldn't be too confusing if
modifiers and types were written in fundamentally different places, but
it's expected that `@isolated(any)` will usually be used on parameter
functions, and parameter modifiers are written immediately adjacent to
the parameter type.  As a result, removing the `@` would create this
unfortunate situation:

```swift
// This means `foo` is isolated to the actor passed in as `actor`.
func foo(actor: isolated MyActor) {}

// This means `operation` is a value of isolated(any) function type;
// it has no impact on the isolation of `bar`.
func bar(operation: isolated(any) () -> ())
```

It is better to preserve the current rule that type modifiers are
written with an `@` sign.

Another alternative would be to not spell the attribute `@isolated(any)`.
For example, it could be spelled `@anyIsolated` or `@dynamicallyIsolated`.
The spelling `@isolated(any)` was chosen because there's an expectation
that this will be one of a family of related isolation-specifying
attributes.  For example, if Swift wanted to make it easier to inherit
actor isolation from one's caller, it could add an `@isolated(caller)`
attribute.  Another example is the `@isolated(to:)` future direction
listed above.  There's merit in having these attributes be closely
related in spelling.  Using a common `Isolated` suffix could serve as
that connection, but in the author's opinion, `@isolated` is much
clearer.

If programmers do end up confused about when to use `@` with `isolated`,
it should be relatively straightforward to provide a good compiler
experience that corrects misuses.

### Implying `@Sendable`

An earlier version of this proposal made `@isolated(any)` imply `@Sendable`.
The logic behind this implication was that `@isolated(any)` is only
really useful if the function is going to be passed to a different
concurrent context.  If a function cannot be passed to a different
concurrent context, the reasoning goes, there's really no point in
it carrying its isolation dynamically, because it can only be used
if that isolation is compatible with the current context.  There's
therefore no reason not to eliminate the redundant `@Sendable` attribute.

However, this logic subtly misunderstands the meaning of `Sendable`
in a world with [region-based isolation][regions].  A type conforming
to `Sendable` means that its values are intrinsically thread-safe and
can be used from multiple concurrent contexts *concurrently*.
Values of non-`Sendable` type are still safe to use from different
concurrent contexts as long as those uses are well-ordered: if the
value is properly [transferred][region-transfers] between contexts,
everything is fine.  Given that, it is sensible for a non-`Sendable`
function to be `@isolated(any)`: if the function can be transferred
to a different concurrent context, it's still useful for it to carry
its isolation dynamically.

In particular, something like a task-creation function ought to declare
the initial task function as a non-`@Sendable` but still transferrable
`@isolated(any)` function.  This permits closures passed in to capture
non-`Sendable` state as long as that state can be transferred into the
closure.  (Ideally, the initial task function would then be able to
transfer that captured state out of the closure.  However, this would
require the compiler to understand that the task function is only
called once.)

## Acknowledgments

I'd like to thank Holly Borla and Konrad Malawski for many long
conversations about the design and implementation of this feature.
