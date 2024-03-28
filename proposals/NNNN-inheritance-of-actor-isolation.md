# Inheritance of actor isolation

* Proposal: [SE-NNNN](NNNN-inheritance-of-actor-isolation.md)
* Authors: [John McCall](https://github.com/rjmccall)
* Review Manager: TBD
* Status: **Awaiting implementation**
* Implementation: TBD
* Review: ([pitch](https://forums.swift.org/t/pitch-inheriting-the-callers-actor-isolation/68391))  

**This is a draft design which has not been implemented.  It's very
possible that some of the ideas in it simply do not work.**

[SE-0302]: https://github.com/apple/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md
[SE-0304-propagation]: https://github.com/apple/swift-evolution/blob/main/proposals/0304-structured-concurrency.md#actor-context-propagation
[SE-0306]: https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md
[SE-0313]: https://github.com/apple/swift-evolution/blob/main/proposals/0313-actor-isolation-control.md
[SE-0316]: https://github.com/apple/swift-evolution/blob/main/proposals/0316-global-actors.md
[SE-0338]: https://github.com/apple/swift-evolution/blob/main/proposals/0338-clarify-execution-non-actor-async.md
[SE-0392]: https://github.com/apple/swift-evolution/blob/main/proposals/0392-custom-actor-executors.md

## Introduction

Under Swift's [actors design][SE-0306], every function in Swift has
an actor isolation: it is either isolated to some specific actor or
non-isolated.  It is sometimes useful to be able to give a function
the same actor isolation as its caller, either to give it access to
actor-isolated data or just to avoid unnecessary suspensions.  This
proposal allows `async` functions to opt in to this behavior.

## Motivation

The actor isolation of a function controls whether and how the
function can access actor-isolated data.  An isolated function
can synchronously access the isolated storage of its actor, such
as the isolated properties of an [`actor` declaration][SE-0306]
or a global variable annotated with a [global actor attribute][SE-0316].
When called from another function with the same isolation, it can
also pass and return non-[`Sendable`][SE-0302] values that are
isolated to the actor.  A non-isolated function cannot do these
things, so making sure that functions share the same formal actor
isolation is sometimes important in order to safely express certain
patterns.

Actor isolation also affects how the function is executed.  Calls
and returns between functions with different actor isolations
may require the task to be suspended and then enqueued on a
different executor.[^1]  Even when this is not required, there is
typically some overhead associated with the switch.  Programmers
trying to optimize `async` code often find that avoiding these
overheads is important.  Avoiding extra suspensions from
actor-isolated code can also be semantically important because
code from other tasks can interleave on the actor during suspensions,
potentially changing the values stored in isolated storage;
this is guaranteed not to happen at the moments of call and return
between functions with the same isolation.

[^1]: This always happens when one of the functions is isolated
to an actor with a [custom actor executor][SE-0392], such as the
main actor (which uses a custom executor to ensure that execution
always happens on the main thread).  For other actors, it typically
only happens when the actor is contended.

Non-isolated synchronous functions dynamically inherit the isolation
of their caller.  For example, an `actor` method can call a non-isolated
synchronous function, and the function will behave dynamically as if it
is isolated to the actor.  While the function cannot directly access
actor-isolated storage --- it would need to be statically isolated to
the actor to do that --- it can be passed and return non-`Sendable`
values that are isolated to the actor.  Among other things, this means
that you can call a function like `map` on an actor-isolated `Array`
of non-`Sendable` values; you can even pass it an actor-isolated
function, and everything will run synchronously and without suspension.

However, there is currently no way to get this same effect from an
asynchronous function.  [SE-0338][] clarified that non-isolated
`async` functions do not inherit isolation in this same way; instead,
they reset isolation.[^2]  This means that these functions cannot
get passed and return non-`Sendable` data when called from an isolated
context, which can be a serious expressivity restriction, especially
for higher-order functions.  It may also cause unwanted suspensions.

[^2]: Prior to SE-0338, non-isolated asynchronous functions still
didn't properly inherit their caller's isolation: they just didn't
actively switch away.  As a result, they ran with whatever isolation
they happened to the called or resumed with.  That is not good enough
to allow them to safely be passed actor-isolated data or to make
strong guarantees of a lack of suspensions.  This is now an ABI
constraint: even if we wanted to change the language to make these
functions inherit their caller's isolation by default, they aren't
passed that information reliably and have no way to implement those
semantics.

For example, suppose you wanted to write a version of `map` that
allows the transform to be `async`:[^3]

```swift
extension Collection {
  func sequentialMap<R>(transform: (Element) async -> R) async -> [R] {
    var results: [R] = []
    for elt in self {
      results.append(await transform(elt))
    }
    return results
  }
}
```

[^3]: The prospect of an `async` `map` function naturally raises the
question of whether it should be a *parallel* map, i.e. whether the
operation should be run concurrently for different elements.
Parallel maps are a useful but different operation from the one
we're discussing here, which still runs the transform sequentially
for each element.  Inheriting isolation isn't as important for a
parallel map because the transform probably isn't isolated itself
and certainly has to be `@Sendable`.

As written above, `sequentialMap` is a non-isolated asynchronous
function.  If it is called from another non-isolated asynchronous
function, everything's okay: it can be passed an arbitrary function
and work with arbitrary types.  But if it's called from an *isolated*
asynchronous function, Swift will treat the call as crossing an
isolation barrier and enforce three restrictions:

- First, the result of the call (`[R]`, i.e. `Array<R>`) must be
  `Sendable`.  `Array` conditionally conforms to `Sendable` if its
  `Element` type does, so this means `R` must be `Sendable`.
  In short, this restriction prevents `sequentialMap` from being
  used from an actor to produce an array of values isolated to the
  actor.

- Second, the `self` argument to the call (the collection) must
  be `Sendable`.  Collections are generally `Sendable` if their
  element types are `Sendable`, so in short, this restriction
  prevents `sequentialMap` from being used from an actor to work
  on a collection of values isolated to the actor.

- Finally, the `transform` argument to the call must be `Sendable`.
  Functions are generally `Sendable` if they don't capture any
  `Sendable` values, so in short, this restriction prevents
  `sequentialMap` from using any other data that's isolated to
  the actor.

In summary, these restrictions unnecessarily limit the capability of
the API when used from an isolated context.[^4]  Furthermore, even
if the API is usable (e.g. because all the types involved are
`Sendable`), it may be unexpectedly inefficient if, say, the
function argument is an isolated function.

[^4]: To be sure, there are some reasons to be concerned about using
this sort of API from an isolated context --- for example, because
actors are re-entrant, other operations on the actor can interleave
whenever `sequentialMap` suspends.  An API author might reasonably
decide that this consideration advises against making `sequentialMap`
try to inherit actor isolation.  But the example concisely demonstrates
the isolation problems this proposal is trying to solve.

This proposal addresses this problem by giving programmers better
tools for formally inheriting isolation from their caller, allowing
non-`Sendable` data to be safely passed back and forth and
avoiding unnecessary suspensions.

## Proposed solution

This proposal makes three changes to the language:

- First, [SE-0313][]'s `isolated` parameters can now have optional
  type.  This is required in order for them to express that the
  function should be dynamically non-isolated.

- Second, default argument expressions can now have the special form
  `#isolation`, which will be filled in with the actor isolation of
  the caller.  If the default argument is for an `isolated` parameter,
  this allows isolation to be implicitly passed down.

- Third, a function can now be declared `@inheritsIsolation` to specify
  that it formally inherits the actor isolation of its caller.  This
  is equivalent to receiving an `isolated` parameter with the same
  value as would be produced by `#isolated`, but it's easier to write,
  can be used in a few places that can't add arbitrary parameters, and
  may be more efficiently implementable.

## Detailed design

### Design approach

The basic design approach of this proposal is to first enable
polymorphism over actor isolation, so that a function can declare
itself to have an arbitrary dynamic isolation, then add features
to allow that to be implicitly propagated in calls to the function.
The isolation logic can then recognize calls that propagate the
caller's isolation in sufficiently obvious ways and know that the
callee will share the current context's isolation.

A function can be non-isolated, isolated to a specific actor instance,
or isolated to a global actor type.  Dynamically, however, global actor
isolation is really just isolation to the `shared` instance of the
global actor, so a function's isolation can actually be dynamically
represented as just an optional actor reference, with `nil`
representing non-isolation.

Since isolation is unavoidably value-dependent (an actor method is
isolated to a *specific* actor reference, not just any actor of that
type), polymorphism over it can't be expressed with just generics.
The natural next choice is to just use a parameter of polymorphic
type, such as `(any Actor)?`.  This matches [SE-0313][]'s `isolated`
parameter feature, except that `isolated` parameters are currently
required to be non-optional actor types: either a concrete `actor`
type or a protocol type which implies `Actor`.  Generalizing this
is straightforward and gives us the ability to make functions
explicitly polymorphic over an arbitrary isolation.

Allowing arbitrary isolation to implicitly propagate from caller to
callee is a little trickier.  If isolation is specified as a parameter,
then the caller must implicitly provide an argument to it; the most
obvious way to do that is to create a new special form for default
arguments, like `#line`, which expands to an expression that
evaluates to the isolation of the caller.  This is attractively
general, but it has three downsides.

The first downside is that the use pattern we expect to dominate ---
declaring a function to inherit its caller's isolation --- is pretty
cumbersome:

```swift
func foo(isolation: isolated (any Actor)? = #isolation)
```

The second downside is we can only do this if we can add formal
parameters to a function.  Unfortunately, there are several places
in the language where we really can't do that, most importantly
accessors for computed properties:

```swift
var count: Int {
  get { // How do we add an isolated parameter here?
    ...
  }
}
```

The third downside is minor in comparison, but this pattern naturally
turns into passing an actor reference, which isn't the most efficient
way of passing down actor isolation because it still requires dynamic
dispatch in order to extract the executor.  It would be better for the
implementation if we could pass down the exact `UnownedSerialExecutor?`
value that's needed at runtime.  While we do not currently want to
encourage programmers to work with values of this type directly because
of its tricky lifetime semantics, the compiler can manage it fairly
easily.

All three of these downsides are addressed by just adding an attribute
which causes an entity (including an accessor) to inherit its caller's
isolation.  The behavior of this can be defined in terms of the more
general features, but its implementation can be tailored more carefully.

### Generalized `isolated` parameters

The type of an `isolated` parameter must be an *isolation type*.
Currently, the only kind of isolation is a possibly-optional actor type,
which is to say, either `T` or `Optional<T>`, where `T` either conforms
to `Actor` or is a protocol type that implies `Actor`.

If a function's `isolated` parameter has an optional actor type, then
the dynamic isolation of the function depends on whether the argument
value is `nil`.  If it is `nil`, then the function behaves dynamically
as it were non-isolated; for example, if the function is `async`, it
resets isolation on entry under [SE-0338][] just as a non-isolated
function would.  Otherwise, the function behaves dynamically as it
were isolated to the unwrapped actor reference.

According to [SE-0304][SE-0304-propagation], closures passed directly
to the `Task` initializer (i.e. `Task { /*here*/ }`) inherit the
statically-specified isolation of the current context if:

- the current context is non-isolated,
- the current context is isolated to a global actor, or
- the current context has an `isolated` parameter (including the
  implicit `self` of an actor method) and that parameter is strongly
  captured by the closure.

The third clause is modified by this proposal to say that isolation
is also inherited if a non-optional binding of an isolated parameter
is captured by the closure (see below).

### Generalized isolation checking

When calling a function with an `isolated` parameter, the function
shares the same isolation as the current context if:

- the current context is non-isolated, the parameter type is optional,
  and the argument expression is `nil` or a reference to `Optional.none`;

- the current context has an `isolated` parameter (including the
  implicitly-`isolated` `self` parameter of an actor function) and
  the argument expression is a reference to that parameter or a
  non-optional derivation of it (see below); or

- the current context is isolated to a global actor type `T` and the
  argument expression is `T.shared`, where `shared` is `GlobalActor`'s
  protocol requirement or the concrete declaration which provides it
  in `T`'s conformance to `GlobalActor`.

An expression is a non-optional derivation of an isolated parameter
`param` if it is:
- `param?` (the optional-chaining operator);
- `param!` (the force-unwrapping operator); or
- a reference to a *non-optional binding* of `param`, i.e. a `let`
  constant initialized by a successful pattern-match which removes
  the optionality from `param`, such as `ref` in `if let ref = param`.

When analyzing an argument expression in all cases above, certain
non-instrumental differences in expression syntax and behavior must
be ignored:
- parentheses;
- the effect-marking operators `try`, `try?`, `try!`, and `await`;[^5]
- the type coercion operator `as` (in the cases where it doesn't
  perform a dynamic bridging conversion); and
- implicit type conversions such as promotion to `Optional` type.

[^5]: The restrictions on the underlying expression should make it
pointless to use these operators, but they must be ignored anyway.

Note that the special `#isolation` default argument form should
always be replaced by something matching the rule above, so calls
using this default argument for an isolated parameter will always be
to a context that shares isolation.

Calls to `@inheritsIsolation` functions and initializers, and uses of
`@inheritsIsolation` variables and subscripts, always share the isolation
of the current context.

For example:

```swift
/// This class type is not Sendable.
class Counter {
  var count = 0
}

extension Counter {
  /// Since this is an async function, if it were just declared
  /// non-isolated, calling it from an isolated context would be
  /// forbidden because it requires sharing a non-Sendable value
  /// between concurrency domains.  Inheriting isolation makes it
  /// okay.  This is a contrived example chosen for its simplicity.
  func incrementAndSleep(isolation: isolated (any Actor)?) async {
    count += 1
    await Task.sleep(nanoseconds: 1_000_000)
  }
}

actor MyActor {
  var counter = Counter()
}

extension MyActor {
  func testActor(other: MyActor) {
    // allowed
    await counter.incrementAndSleep(isolation: self)

    // not allowed
    await counter.incrementAndSleep(isolation: other)

    // not allowed
    await counter.incrementAndSleep(isolation: MainActor.shared)

    // not allowed
    await counter.incrementAndSleep(isolation: nil)
  }
}

@MainActor func testMainActor(counter: Counter) {
  // allowed
  await counter.incrementAndSleep(isolation: MainActor.shared)

  // not allowed
  await counter.incrementAndSleep(isolation: nil)
}

func testNonIsolated(counter: Counter) {
  // allowed
  await counter.incrementAndSleep(isolation: nil)

  // not allowed
  await counter.incrementAndSleep(isolation: MainActor.shared)
}
```

### `#isolation` default argument

The special expression form `#isolation` can be used as a default
argument:

```swift
extension Collection {
  func sequentialMap<R>(isolation: isolated (any Actor)? = #isolation,
                        transform: (Element) async -> R) async -> [R] {
    var results: [R] = []
    for elt in self {
      results.append(await transform(elt))
    }
    return results
  }
}
```

When a call uses this default argument, it behaves as if the argument
was an expression representing the static actor isolation of the
current context:

- if the current context is statically non-isolated, the parameter
  must have optional type, and the argument is `nil`;
- if the current context is isolated to a global actor `T`, the argument
  is `T.shared`;
- if the current context has an `isolated` parameter (including the
  implicit `self` parameter of an actor method), the argument is a
  reference to that parameter;
- otherwise, the current context must be a closure which captures
  an `isolated` parameter or a non-optional binding of it, and the
  argument is a reference to that capture.

Except where noted above, a parameter using `#isolation` as a default
argument can have any type.  When type-checking considers a candidate
function for a call that would use this default argument for a parameter,
it assumes that the notional argument expression above can be coerced
to the parameter type.  If the call is actually resolved to use that
candidate, the coercion must succeed or the call is ill-formed.
This rule is necessary in order to avoid the need to decide the isolation
of the calling context before resolving calls from it.

The parameter does not have to be an `isolated` parameter.

### `@inheritsIsolation` attribute

The attribute `@inheritsIsolation` can be applied to a `func`,
`init`, `subscript`, or computed `var`:

```swift
extension Collection {
  @inheritsIsolation
  func sequentialMap<R>(transform: (Element) async -> R) async -> [R] {
    var results: [R] = []
    for elt in self {
      results.append(await transform(elt))
    }
    return results
  }
}
```

`@inheritsIsolation` cannot be used on a variable that requires storage.
When it is written on a `subscript` or `var`, it applies as if written
on every accessor of that declaration.

`@inheritsIsolation` is a form of explicit isolation specification and
cannot be combined with other isolation specifications, such as an
`isolated` parameter or a global actor attribute.

`@inheritsIsolation` can be deduced for a member declaration from a
protocol requirement in the same situations that other forms of isolation
can be deduced.[^6]

[^6]: When a member declaration does not have an explicit isolation
specification, and it appears in a type body or extension that also
declares conformance to one or more protocols, and those protocols
(or the protocols they derive from) declare requirements with the
same name as the member, and those requirements all have the same
actor isolation, the member will be deduced to have that isolation.

An `@inheritsIsolation` declaration is isolated to an optional actor
reference that's dynamically provided to the declaration and which
always matches the isolation of the context which is directly accessing
or calling the declaration.  For functions and initializers, this
exactly matches the behavior of adding an implicit parameter
`_: isolated (any Actor)? = #isolation`, except that it is not reflected
in the type of the entity and the default argument is always used.

`@inheritsIsolation` is not part of the type of the declaration.
When an `@inheritsIsolation` function or initializer is referenced
without being called, its isolation is determined from its type
context; if there is no context, it is non-isolated.  For example:

```swift
@inheritsIsolation func inheritingFunction() async {}

func runOnMainActor(operation: @MainActor () async -> ()) { ... }

// `inheritingFunction` will be dynamically isolated to the main actor.
runOnMainActor(operation: inheritingFunction)

func runAnywhere(operation: () async -> ()) { ... }

// `inheritingFunction` will be dynamically non-isolated.
runAnywhere(operation: inheritingFunction)
```

Unlike most function conversions that change actor isolation, there
are no restrictions against converting synchronous `@inheritsIsolation`
functions because the actor isolation does not actually change.

Similarly, an `@inheritsIsolation` declaration can be used to fulfill
a protocol requirement with any isolation.  The isolation of the protocol
requirement determines the dynamic isolation of the `@inheritsIsolation`
function.

It is likely that `@inheritsIsolation` can be implemented more efficiently
than just passing down an actor reference, such as by passing down an
`UnownedSerialExecutor`.  This has no semantic impact on the language,
however, and is not a question for Swift Evolution.

## Source compatibility

This proposal is largely additive and should not affect the behavior
of existing code.

The new rules for isolation checking permit more calls to be
recognized as sharing isolation.  This should strictly allow
more code to be compiled; it cannot cause source-compatibility
regressions by allowing different overloads to be picked because
isolation checking is performed separately from type-checking.

## ABI compatibility

This proposal does not change how any existing code is compiled.

## Implications for adoption

This proposal does not add any new types and does not require new
runtime or library support.  It can be implemented purely in the compiler.

Adding `@inheritsIsolation` to an existing declaration is an ABI-breaking
change: the ABI of the declaration must be changed to allow the isolation
information to be provided to it dynamically.

Adding `#isolation` as a default argument to an existing parameter is not
ABI-breaking, but this is probably an uncommon situation.  Adding a new
parameter to an existing declaration is ABI-breaking, of course.

Making a library function inherit isolation is effectively a promise that
it can work when called from any isolated context.  While this might seem
superficially like a pretty strong guarantee, it's not very different
in practice from just making the library function non-isolated: in both
cases, the function does not have any isolation preconditions that it can
rely on.  Library authors should not be reserved about adopting this
proposal on that account.

A better reason to be cautious about adopting this feature is that it
can cause more work to be done while actor-isolated, potentially creating
significant "hangover" on the actor lock and a less effective use of
concurrency.  It may be better for the whole system if functions that do
significant computational work, including doing a lot of object
allocation and initialization, stay non-isolated rather than
isolation-inheriting.  On the other hand, `async` functions with "fast
paths" --- functions that usually return quickly and only occasionally
need to set up more expensive work --- may see real benefits from
extracting the fast path into a function that inherits isolation and
then leaving the slow path in a non-isolated function.

## Future directions

### Composition with other isolation control features

This feature composes well with features that improve control over
formal actor isolation.  For example, if a programmer wants to make
an `@inheritsIsolation` function run isolated to a specific actor, they
can do that by calling it from a function that's explicitly isolated
to that actor.  That can of course be done with an `extension` on the
actor, but it could also be done in a more one-off way with e.g. a
feature that allows closures to precisely state their isolation.
This is a very interesting topic for a follow-up proposal.

### Isolated function types

This proposal is focused on propagating isolation information *into*
functions, but it's also interesting to look at propagating isolation
*out* of functions.  Currently, the Swift type system only allows
function isolation to be expressed in limited ways: functions can be
declared as isolated to a global actor (e.g. `@MainActor () -> ()`), but
all other kinds of isolation must be "type-erased", leaving a value
whose type appears to be non-isolated.  This forces some of the same
awkward `Sendable` restrictions that this proposal discusses with the
`sequentialMap` example in the Motivation section.

One way to solve this would be to introduce value-dependent isolated
function types.  With such a feature, you could declare a value to have
type, say, `@isolated(myActor) () -> ()`, where `myActor` is a `let`
constant in the local scope.  This kind of value dependence, however,
is a large step in complexity for a type system, and it's not a likely
path for Swift in the foreseeable future.

A more promising approach would be to allow the isolation to be
statically erased but still make it dynamically recoverable by carrying
it along in the function value, essentially as an extra value of type
`(any Actor)?`.  This would look something like `@isolated () -> ()`,
and it could be used to e.g. dynamically propagate the isolation of
a function into something like the `Task` initializer so that the task
can immediately start on the right executor.  This would compose well
with the features in this proposal because it would be natural to allow
such functions to be used as `isolated` parameters.  This would be very
nice for functions like `sequentialMap` that should probably be isolated
not to their *caller* but to the *function they've been passed*.

## Alternatives Considered

### Allowing isolation to `SerialExecutor` types

This proposal observes that it is more efficient to pass down an
`UnownedSerialExecutor` value instead of an actor reference and suggests
that `@inheritsIsolation` be implemented this way.  However, if a function
wants to use the explicit isolation-polymorphism pattern, it cannot use
this more efficient pattern because an `isolated` parameter must be an
actor type.  This is an intentional decision.

Philosophically, Swift programmers should be encouraged to think about
actors in terms of isolation rather than execution policy.  There are many
ways for actors to provide isolation, many of which don't require taking
over execution; in fact, Swift's actors use one such approach by default.
Keeping the focus on actors rather than executors supports this.

Putting that aside, there also just isn't a reasonable type that could
be used here:

- `UnownedSerialExecutor` is an unsafe type that requires the compiler
to implicitly manage a dependency on the underlying actor or executor
reference in order to safely use.  While this is not difficult for the
compiler, we do not want to encourage programmers to use this type
directly.  If Swift introduces a safe replacement in the future, possibly
using future language support for value dependencies, we can consider
allowing that to be used as an `isolated` parameter type at that time.

- A managed serial executor reference such as `any SerialExecutor`
would be a safe alternative, but it's a surprisingly complex one.
For one, normal isolated contexts would not to be able to implement
`#isolation` forwarding to such a parameter, because there's currently no
way to get a managed serial executor reference from an actor, only an
`UnownedSerialExecutor`.  For another, actor types can (and often do)
also conform to `SerialExecutor`, but there's nothing in the language
requiring those actors to always use `self` as their executor.  This
greatly complicates the logic for both establishing and forwarding
isolation; e.g. an isolated *actor* parameter must not be forwarded
directly as an isolated *executor* (as opposed to extracting the
correct executor reference) even if the actor's type would normally
implicitly convert.  Furthermore, the decision logic for whether a call
crosses isolation would have to recognize expressions that extract serial
executors, as well as appropriately reasoning about actor/executor
differences.  And finally, getting an `UnownedSerialExecutor` from an
`any SerialExecutor` still requires calling a protocol method, so it's
not really enabling any sort of optimization.

## Acknowledgments

I'd like to especially thank Konrad Malawski, Doug Gregor, and
Holly Borla for their help in developing the ideas in this proposal.
