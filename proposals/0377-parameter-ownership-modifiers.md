# `borrowing` and `consuming` parameter ownership modifiers

* Proposal: [SE-0377](0377-parameter-ownership-modifiers.md)
* Authors: [Michael Gottesman](https://github.com/gottesmm), [Joe Groff](https://github.com/jckarter)
* Review Manager: [John McCall](https://github.com/rjmccall)
* Status: **Implemented (Swift 5.9)**
* Implementation: in main branch of compiler
* Review: ([first pitch](https://forums.swift.org/t/pitch-formally-defining-consuming-and-nonconsuming-argument-type-modifiers/54313)) ([second pitch](https://forums.swift.org/t/borrow-and-take-parameter-ownership-modifiers/59581)) ([first review](https://forums.swift.org/t/se-0377-borrow-and-take-parameter-ownership-modifiers/61020)) ([second review](https://forums.swift.org/t/combined-se-0366-third-review-and-se-0377-second-review-rename-take-taking-to-consume-consuming/61904)) ([acceptance](https://forums.swift.org/t/accepted-with-modifications-se-0377-borrowing-and-consuming-parameter-ownership-modifiers/62759)) ([revision and third review](https://forums.swift.org/t/se-0377-revision-make-borrowing-and-consuming-parameters-require-explicit-copying-with-the-copy-operator/64996)) ([revision acceptance](https://forums.swift.org/t/accepted-se-0377-revision-make-borrowing-and-consuming-parameters-require-explicit-copying-with-the-copy-operator/65293))
* Previous Revisions: ([as of first review](https://github.com/swiftlang/swift-evolution/blob/3f984e6183ce832307bb73ec72c842f6cb0aab86/proposals/0377-parameter-ownership-modifiers.md)) ([as of second review](https://github.com/swiftlang/swift-evolution/blob/7e1d16316e5f68eb94546df9241aa6b4cacb9411/proposals/0377-parameter-ownership-modifiers.md))

## Introduction

We propose new `borrowing` and `consuming` parameter modifiers to allow developers to
explicitly choose the ownership convention that a function uses to receive
immutable parameters. Applying one of these modifiers to a parameter causes
that parameter binding to no longer be implicitly copyable, and potential
copies need to be marked with the new `copy x` operator. This allows for
fine-tuning of performance by reducing the number of ARC calls or copies needed
to call a function, and provides a necessary prerequisite feature for
noncopyable types to specify whether a function consumes a noncopyable value or
not.

## Motivation

Swift uses automatic reference counting to manage the lifetimes of reference-
counted objects. There are two broad conventions that the compiler uses to
maintain memory safety when passing an object by value from a caller to a
callee in a function call:

* The callee can **borrow** the parameter. The caller
  guarantees that its argument object will stay alive for the duration of the
  call, and the callee does not need to release it (except to balance any
  additional retains it performs itself).
* The callee can **consume** the parameter. The callee
  becomes responsible for either releasing the parameter or passing ownership
  of it along somewhere else. If a caller doesn't want to give up its own
  ownership of its argument, it must retain the argument so that the callee
  can consume the extra reference count.

These two conventions generalize to value types, where a "retain"
becomes an independent copy of the value, and "release" the destruction and
deallocation of the copy. By default Swift chooses which convention to use 
based on some rules informed by the typical behavior of Swift code:
initializers and property setters are more likely to use their parameters to
construct or update another value, so it is likely more efficient for them to
*consume* their parameters and forward ownership to the new value they construct.
Other functions default to *borrowing* their parameters, since we have found
this to be more efficient in most situations.

These choices typically work well, but aren't always optimal.
Although the optimizer supports "function signature optimization" that can
change the convention used by a function when it sees an opportunity to reduce
overall ARC traffic, the circumstances in which we can automate this are
limited. The ownership convention becomes part of the ABI for public API, so
cannot be changed once established for ABI-stable libraries. The optimizer
also does not try to optimize polymorphic interfaces, such as non-final class
methods or protocol requirements. If a programmer wants behavior different
from the default in these circumstances, there is currently no way to do so.

[SE-0390](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0390-noncopyable-structs-and-enums.md)
introduces noncopyable types into Swift. Since noncopyable types do not have
the ability to be copied, the distinction between these two conventions becomes
an important part of the API contract: functions that *borrow* noncopyable
values make temporary use of the value and leave it valid for further use, like
reading from a file handle, whereas functions that *consume* a noncopyable
value consume it and prevent its further use, like closing a file handle.
Relying on implicit selection of the parameter convention will not suffice for
these types.

## Proposed solution

We give developers direct control over the ownership convention of
parameters by introducing two new parameter modifiers, `borrowing` and
`consuming`.

## Detailed design

### Syntax of parameter ownership modifiers

`borrowing` and `consuming` become contextual keywords inside parameter type
declarations.  They can appear in the same places as the `inout` modifier, and
are mutually exclusive with each other and with `inout`. In a `func`,
`subscript`, or `init` declaration, they appear as follows:

```swift
func foo(_: borrowing Foo)
func foo(_: consuming Foo)
func foo(_: inout Foo)
```

In a closure:

```swift
bar { (a: borrowing Foo) in a.foo() }
bar { (a: consuming Foo) in a.foo() }
bar { (a: inout Foo) in a.foo() }
```

In a function type:

```swift
let f: (borrowing Foo) -> Void = { a in a.foo() }
let f: (consuming Foo) -> Void = { a in a.foo() }
let f: (inout Foo) -> Void = { a in a.foo() }
```

Methods can also use the `consuming` or `borrowing` modifier to indicate
respectively that they consume ownership of their `self` parameter or that they
borrow it. These modifiers are mutually exclusive with each other and with the
existing `mutating` modifier:

```swift
struct Foo {
  consuming func foo() // `consuming` self
  borrowing func foo() // `borrowing` self
  mutating func foo() // modify self with `inout` semantics
}
```

`consuming` cannot be applied to parameters of nonescaping closure type, which by
their nature are always borrowed:

```swift
// ERROR: cannot `consume` a nonescaping closure
func foo(f: consuming () -> ()) {
}
```

`consuming` or `borrowing` on a parameter do not affect the caller-side syntax for
passing an argument to the affected declaration, nor do `consuming` or
`borrowing` affect the application of `self` in a method call. For typical
Swift code, adding, removing, or changing these modifiers does not have any
source-breaking effects. (See "related directions" below for interactions with
other language features being considered currently or in the near future which
might interact with these modifiers in ways that cause them to break source.)

### Ownership convention conversions in protocols and function types

Protocol requirements can also use `consuming` and `borrowing`, and the modifiers will
affect the convention used by the generic interface to call the requirement.
The requirement may still be satisfied by an implementation that uses different
conventions for parameters of copyable types:

```swift
protocol P {
  func foo(x: consuming Foo, y: borrowing Foo)
}

// These are valid conformances:

struct A: P {
  func foo(x: Foo, y: Foo)
}

struct B: P {
  func foo(x: borrowing Foo, y: consuming Foo)
}

struct C: P {
  func foo(x: consuming Foo, y: borrowing Foo)
}
```

Function values can also be implicitly converted to function types that change
the convention of parameters of copyable types among unspecified, `borrowing`,
or `consuming`:

```swift
let f = { (a: Foo) in print(a) }

let g: (borrowing Foo) -> Void = f
let h: (consuming Foo) -> Void = f

let f2: (Foo) -> Void = h
```

These implicit conversions for protocol conformances and function values
are not available for parameter types that are noncopyable, in which case
the convention must match exactly.

### Using parameter bindings with ownership modifiers

Inside of a function or closure body, `consuming` parameters may be mutated, as can
the `self` parameter of a `consuming func` method. These
mutations are performed on the value that the function itself took ownership of,
and will not be evident in any copies of the value that might still exist in
the caller. This makes it easy to take advantage of the uniqueness of values
after ownership transfer to do efficient local mutations of the value:

```swift
extension String {
  // Append `self` to another String, using in-place modification if
  // possible
  consuming func plus(_ other: String) -> String {
    // Modify our owned copy of `self` in-place, taking advantage of
    // uniqueness if possible
    self += other
    return self
  }
}

// This is amortized O(n) instead of O(n^2)!
let helloWorld = "hello ".plus("cruel ").plus("world")
```

`borrowing` and `consuming` parameter values are also **not implicitly copyable**
inside of the function or closure body:

```swift
func foo(x: borrowing String) -> (String, String) {
    return (x, x) // ERROR: needs to copy `x`
}
func bar(x: consuming String) -> (String, String) {
    return (x, x) // ERROR: needs to copy `x`
}
```

And so is the `self` parameter within a method that has the method-level
`borrowing` or `consuming` modifier:

```swift
extension String {
    borrowing func foo() -> (String, String) {
        return (self, self) // ERROR: needs to copy `self`
    }
    consuming func bar() -> (String, String) {
        return (self, self) // ERROR: needs to copy `self`
    }
}
```

A value would need to be implicitly copied if:

- a *consuming operation* is applied to a `borrowing` binding, or
- a *consuming operation* is applied to a `consuming` binding after it has
  already been consumed, or while a *borrowing* or *mutating operation* is simultaneously
  being performed on the same binding

where *consuming*, *borrowing*, and *mutating operations* are as described for
values of noncopyable type in
[SE-0390](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0390-noncopyable-structs-and-enums.md#using-noncopyable-values).
In essence, disabling implicit copying for a binding makes the binding behave
as if it were of some noncopyable type.

To allow a copy to occur, the `copy x` operator may be used:

```swift
func dup(_ x: borrowing String) -> (String, String) {
    return (copy x, copy x) // OK, copies explicitly allowed here
}
```

`copy x` is a *borrowing operation* on `x` that returns an independently
owned copy of the current value of `x`. The copy may then be independently 
consumed or modified without affecting the original `x`. Note that, while
`copy` allows for a copy to occur, it is not a strict
obligation for the compiler to do so; the copy may still be optimized away
if it is deemed semantically unnecessary.

`copy` is a contextual keyword, parsed as an operator if it is immediately
followed by an identifier on the same line, like the `consume x` operator before
it. In all other cases, `copy` is still treated as a reference to a
declaration named `copy`, as it would have been prior to this proposal.

The constraint on implicit copies only affects the parameter binding itself.
The value of the parameter may be passed to other functions, or assigned to
other variables (if the convention allows), at which point the value may 
be implicitly copied through those other parameter or variable bindings.

```swift
func foo(x: borrowing String) {
    let y = x // ERROR: attempt to copy `x`
    bar(z: x) // OK, invoking `bar(z:)` does not require copying `x`
}

func bar(z: String) {
    let w = z // OK, z is implicitly copyable here
}

func baz(a: consuming String) {
    // let aa = (a, a) // ERROR: attempt to copy `a`

    let b = a
    let bb = (b, b) // OK, b is implicitly copyable
}
```

To clarify the boundary within which the no-implicit-copy constraint applies, a
parameter binding's value *is* noncopyable as part of the *call expression* in
the caller, so if forming the call requires copying, that will raise an error,
even if the parameter would be implicitly copyable in the callee. The function
body serves as the boundary for the no-implicit-copy constraint:

```swift
struct Bar {
    var a: String
    var b: String
    init(ab: String) {
        // OK, ab is implicitly copyable here
        a = ab
        b = ab
    }
}

func foo(x: borrowing String) {
    _ = Bar(ab: x) // ERROR: would need to copy `x` to let `Bar.init` consume it
}
```

## Source compatibility

Adding `consuming` or `borrowing` to a parameter in the language today does not
affect source compatibility with existing code outside of that function.
Callers can continue to call the function as normal, and the function body can
use the parameter as it already does. A method with `consuming` or `borrowing`
modifiers on its parameters can still be used to satisfy a protocol requirement
with different modifiers. Although `consuming` parameter bindings become
mutable, and parameters with either of the `borrowing` or `consuming` modifiers
are not implicitly copyable, the effects are localized to the function
adopting the modifiers. This allows for API authors to use
`consuming` and `borrowing` annotations to fine-tune the copying behavior of
their implementations, without forcing clients to be aware of ownership to use
the annotated APIs. Source-only packages can add, remove, or adjust these
annotations on copyable types over time without breaking their clients.

Changing parameter modifiers from `borrowing` to `consuming` may however break
source of any client code that also adopts those parameter modifiers, since the
change may affect where copies need to occur in the caller. Going from
`consuming` to `borrowing` however should generally not be source-breaking
for a copyable type. A change in either direction is source-breaking if the
parameter type is noncopyable.

## Effect on ABI stability

`consuming` or `borrowing` affects the ABI-level calling convention and cannot be
changed without breaking ABI-stable libraries (except on "trivial types"
for which copying is equivalent to `memcpy` and destroying is a no-op; however,
`consuming` or `borrowing` also has no practical effect on parameters of trivial type).

## Effect on API resilience

`consuming` or `borrowing` break ABI for ABI-stable libraries, but are intended to have
minimal impact on source-level API. When using copyable types, adding or
changing these annotations to an API should not affect its existing clients,
except where those clients have also adopted the not-implicitly-copyable
conventions.

## Alternatives considered

### Leaving `consuming` parameter bindings immutable inside the callee

We propose that `consuming` parameters should be mutable inside of the callee,
because it is likely that the callee will want to perform mutations using
the value it has ownership of. There is a concern that some users may find this
behavior unintuitive, since those mutations would not be visible in copies
of the value in the caller. This was the motivation behind
[SE-0003](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0003-remove-var-parameters.md),
which explicitly removed the former ability to declare parameters as `var`
because of this potential for confusion. However, whereas `var` and `inout`
both suggest mutability, and `var` does not provide explicit directionality as
to where mutations become visible, `consuming` on the other hand does not
suggest any kind of mutability to the caller, and it explicitly states the
directionality of ownership transfer. Furthermore, with noncopyable types, the
chance for confusion is moot, because the transfer of ownership means the
caller cannot even use the value after the callee takes ownership anyway.

Another argument for `consuming` parameters to remain immutable is to serve the
proposal's stated goal of minimizing the source-breaking impact of
parameter ownership modifiers. When `consuming` parameters are mutable,
changing a `consuming` parameter to `borrowing`, or removing the
`consuming` annotation altogether, is potentially source-breaking. However,
any such breakage is purely localized to the callee; callers are still
unaffected (as long as copyable arguments are involved). If a developer wants
to change a `consuming` parameter back into a `borrowing`, they can still assign the
borrowed value to a local variable and use that local variable for local
mutation.

### Naming

We have considered several alternative naming schemes for these modifiers:

- The current implementation in the compiler uses `__shared` and `__owned`,
  and we could remove the underscores to make these simply `shared` and
  `owned`. These names refer to the way a borrowed parameter receives a
  "shared" borrow (as opposed to the "exclusive" borrow on an `inout`
  parameter), whereas a consumed parameter becomes "owned" by the callee.
  found that the "shared" versus "exclusive" language for discussing borrows,
  while technically correct, is unnecessarily confusing for explaining the
  model.
- A previous pitch used the names `nonconsuming` and `consuming`. The current
  implementation also uses `__consuming func` to notate a method that takes
  ownership of its `self` parameter. We think it is better to describe
  `borrowing` in terms of what it means, rather than as the opposite of
  the other convention.
- The first reviewed revision used `take` instead of `consume`. Along with
  `borrow`, `take` arose during [the first review of
SE-0366](https://forums.swift.org/t/se-0366-move-function-use-after-move-diagnostic/59202).
  These names also work well as names for operators that explicitly
  transfer ownership of a variable or borrow it in place. However,
  reviewers observed that `take` is possibly confusing, since it conflicts with
  colloquial discussion of function calls "taking their arguments". `consume`
  reads about as well while being more specific.
- Reviewers offered `use`, `own`, or `sink` as alternatives to `consume`.

We think it is helpful to align the naming of these parameter modifiers with
the corresponding `consume` and `borrow` operators (discussed below under
Future Directions), since it helps reinforce the relationship between the
calling conventions and the expression operators: to explicitly transfer
ownership of an argument in a call site to a parameter in a function, use
`foo(consuming x)` at the call site, and use `func foo(_: consuming T)` in the
function declaration. Similarly, to explicitly pass an argument by borrow
without copying, use `foo(borrow x)` at the call site, and `func foo(_: borrowing T)`
in the function declaration.

### `@noImplicitCopy` attribute

Instead of having no-implicit-copy behavior be tied to the ownership-related
binding forms and parameter modifiers, we could have an attribute that can
be applied to any binding to say that it should not be implicitly copyable:

```swift
@noImplicitCopy(self)
func foo(x: @noImplicitCopy String) {
    @noImplicitCopy let y = copy x
}
```

We had [pitched this possibility](https://forums.swift.org/t/pitch-noimplicitcopy-attribute-for-local-variables-and-function-parameters/61506),
but community feedback rightly pointed out the syntactic weight and noise
of this approach, as well as the fact that, as an attribute, it makes the
ability to control copies feel like an afterthought not well integrated
with the rest of the language. We've decided not to continue in this direction,
since we think that attaching no-implicit-copy behavior to the ownership
modifiers themselves leads to a more coherent design.

### `copy` as a regular function

Unlike the `consume x` or `borrow x` operator, copying doesn't have any specific
semantic needs that couldn't be done by a regular function. Instead of an
operator, `copy` could be defined as a regular standard library function:

```swift
func copy<T>(_ value: T) -> T {
    return value
}
```

We propose `copy x` as an operator, because it makes the relation to
`consume x` and `borrow x`, and it avoids the issues of polluting the
global identifier namespace and occasionally needing to be qualified as
`Swift.copy` if it was a standard library function.

### Transitive no-implicit-copy constraint

The no-implicit-copy constraint for a `borrowing` and `consuming` parameter
only applies to that binding, and is not carried over to other variables
or function call arguments receiving the binding's value. We could also
say that the parameter can only be passed as an argument to another function
if that function's parameter uses the `borrowing` or `consuming` modifier to
keep implicit copies suppressed, or that it cannot be bound to `let` or `var`
bindings and must be bound using one of the borrowing bindings once we have
those. However, we think those additional restrictions would only make the
`borrowing` and `consuming` modifiers harder to adopt, since developers would
only be able to use them in cases where they can introduce them bottom-up from
leaf functions.

The transitivity restriction also would not really improve
local reasoning; since the restriction is only on *implicit* copies, but
explicit copies are still possible, calling into another function may lead
to that other function performing copies, whether they're implicit or not.
The only way to be sure would be to inspect the callee's implementation.
One of the goals of SE-0377 is to introduce the parameter ownership modifiers
in a way that minimizes disruption to the the rest of a codebase, allowing
for the modifiers to be easily adopted in spots where the added control is
necessary, and a transitivity requirement would interfere with that goal for
little benefit.

## Related directions

#### `consume` operator

[SE-0366](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0366-move-function.md)
introduced an operator that explicitly ends the lifetime of a
variable before the end of its scope. This allows the compiler to reliably
destroy the value of the variable, or transfer ownership, at the point of its
last use, without depending on optimization and vague ARC optimizer rules.
When the lifetime of the variable ends in an argument to a `consume` parameter,
then we can transfer ownership to the callee without any copies:

```swift
func consume(x: consuming Foo)

func produce() {
  let x = Foo()
  consume(x: consume x)
  doOtherStuffNotInvolvingX()
}
```

#### `borrow` operator

Relatedly, there are circumstances where the compiler defaults to copying
when it is theoretically possible to borrow, particularly when working with
shared mutable state such as global or static variables, escaped closure
captures, and class stored properties. The compiler does
this to avoid running afoul of the law of exclusivity with mutations. In
the example below, if `callUseFoo()` passed `global` to `useFoo` by borrow
instead of passing a copy, then the mutation of `global` inside of `useFoo`
would trigger a dynamic exclusivity failure (or UB if exclusivity checks
are disabled):

```swift
var global = Foo()

func useFoo(x: borrowing Foo) {
  // We need exclusive access to `global` here
  global = Foo()
}

func callUseFoo() {
  // callUseFoo doesn't know whether `useFoo` accesses global,
  // so we want to avoid imposing shared access to it for longer
  // than necessary, and we'll pass a copy of the value. This:
  useFoo(x: global)

  // will compile more like:

  /*
  let globalCopy = copy(global)
  useFoo(x: globalCopy)
  destroy(globalCopy)
   */
}
```

It is difficult for the compiler to conclusively prove that there aren't
potential interfering writes to shared mutable state, so although it may
in theory eliminate the defensive copy if it proves that `useFoo`, it is
unlikely to do so in practice. The developer may know that the program will
not attempt to modify the same object or global variable during a call,
and want to suppress this copy. An explicit `borrow` operator could allow for
this:

```swift
var global = Foo()

func useFooWithoutTouchingGlobal(x: borrowing Foo) {
  /* global not used here */
}

func callUseFoo() {
  // The programmer knows that `useFooWithoutTouchingGlobal` won't
  // touch `global`, so we'd like to pass it without copying
  useFooWithoutTouchingGlobal(x: borrow global)
}
```

If `useFooWithoutTouchingGlobal` did in fact attempt to mutate `global`
while the caller is borrowing it, an exclusivity failure would be raised.

#### Noncopyable types

The `consuming` versus `borrowing` distinction becomes much more important and
prominent for values that cannot be implicitly copied.
[SE-0390](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0390-noncopyable-structs-and-enums.md)
introduces noncopyable types, whose values are never copyable, as well as
attributes that suppress the compiler's implicit copying behavior selectively
for particular variables or scopes. Operations that borrow
a value allow the same value to continue being used, whereas operations that
consume a value destroy it and prevent its continued use. This makes the
convention used for noncopyable parameters a much more important part of their
API contract, since it directly affects whether the value is still available
after the operation:

```swift
struct FileHandle: ~Copyable { ... }

// Operations that open a file handle return new FileHandle values
func open(path: FilePath) throws -> FileHandle

// Operations that operate on an open file handle and leave it open
// borrow the FileHandle
func read(from: borrowing FileHandle) throws -> Data

// Operations that close the file handle and make it unusable consume
// the FileHandle
func close(file: consuming FileHandle)

func hackPasswords() throws -> HackedPasswords {
  let fd = try open(path: "/etc/passwd")
  // `read` borrows fd, so we can continue using it after
  let contents = try read(from: fd)
  // `close` consumes fd, so we can't use it again
  close(fd)

  let moreContents = try read(from: fd) // compiler error: use after consume

  return hackPasswordData(contents)
}
```

As such, SE-0390 requires parameters of noncopyable type to explicitly state
whether they are `borrowing` or `consuming`, since there isn't a clear
default that is always safe to assume.

### `set`/`out` parameter convention

By making the `borrowing` and `consuming` conventions explicit, we mostly round out
the set of possibilities for how to handle a parameter. `inout` parameters get
**exclusive access** to their argument, allowing them to mutate or replace the
current value without concern for other code. By contrast, `borrowing` parameters
get **shared access** to their argument, allowing multiple pieces of code to
share the same value without copying, so long as none of them mutate the
shared value. A `consuming` parameter consumes a value, leaving nothing behind, but
there still isn't a parameter analog to the opposite convention, which would
be to take an uninitialized argument and populate it with a new value. Many
languages, including C# and Objective-C when used with the "Distributed
Objects" feature, have `out` parameter conventions for this, and the Val
programming language calls this `set`.

In Swift up to this point, return values have been the preferred mechanism for
functions to pass values back to their callers. This proposal does not propose
to add some kind of `out` parameter, but a future proposal could.

### `borrowing`, `mutating`, and `consuming` local variables

Swift currently lacks the ability to form local bindings to part of an
aggregate without copying that part, other than by passing the part as
an argument to a function call. We plan to introduce [`borrow` and `inout`
bindings](https://forums.swift.org/t/pitch-borrow-and-inout-declaration-keywords/62366)
that will provide this functionality, with the same no-implicit-copy constraint
described by this proposal applied to these bindings.

### Consistency for `inout` parameters and the `self` parameter of `mutating` methods

`inout` parameters and `mutating` methods have been part of Swift since before
version 1.0, and their existing behavior allows for implicit copying of the
current value of the binding. We can't change the existing language
behavior in Swift 5, but accepting this proposal would leave `inout` parameters
and `mutating self` inconsistent with the new modifiers. There are a few things
we could potentially do about that:

- We could change the behavior of `inout` and `mutating self` parameters to
  make them not implicitly copyable in Swift 6 language mode.
- `inout` is also conspicuous now in not following the `-ing` convention we've
  settled on for `consuming`/`borrowing`/`mutating` modifiers. We could introduce
  `mutating` as a new parameter modifier spelling, with no-implicit-copy
  behavior.

One consideration is that, whereas `borrowing` and `consuming` are strictly
optional for code that works only with copyable types, and is OK with letting
the compiler manage copies automatically, there is no way to get in-place
mutation through function parameters except via `inout`.  Tying
no-implicit-copy behavior to mutating parameters could be seen as a violation
of the "progressive disclosure" goal of these ownership features, since
developers would not be able to avoid interacting with the ownership model when
using `inout` parameters anymore.

## Acknowledgments

Thanks to Robert Widmann for the original underscored implementation of
`__owned` and `__shared`: [https://forums.swift.org/t/ownership-annotations/11276](https://forums.swift.org/t/ownership-annotations/11276).

## Revision history

The [first reviewed revision](https://github.com/swiftlang/swift-evolution/blob/3f984e6183ce832307bb73ec72c842f6cb0aab86/proposals/0377-parameter-ownership-modifiers.md)
of this proposal used `take` and `taking` as the name of the callee-destroy convention.

The [second reviewed revision](https://github.com/swiftlang/swift-evolution/blob/e3966645cf07d6103561454574ab3e2cc2b48ee9/proposals/0377-parameter-ownership-modifiers.md)
used the imperative forms, `consume` and `borrow`, as parameter modifiers,
which were changed to the gerunds `consuming` and `borrowing` in review. The
proposal was originally accepted after these revisions.

The current revision alters the originally-accepted proposal to make it so that
`borrowing` and `consuming` parameter bindings are not implicitly copyable,
and introduces a `copy x` operator that can be used to explicitly allow copies
where needed.
