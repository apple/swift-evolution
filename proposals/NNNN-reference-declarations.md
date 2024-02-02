# Add `borrow` and `inout` reference bindings

* Proposal: [SE-NNNN](NNNN-reference-declarations.md)
* Authors: [Tim Kientzle](https://github.com/tbkka)
* Review Manager: TBD
* Status: **Awaiting implementation**
<!--- * Implementation: [apple/swift#NNNNN](https://github.com/apple/swift/pull/NNNNN) or [apple/swift-evolution-staging#NNNNN](https://github.com/apple/swift-evolution-staging/pull/NNNNN) -->
<!--- * Decision Notes: [Rationale](https://forums.swift.org/), [Additional Commentary](https://forums.swift.org/) -->
<!--- * Bugs: [SR-NNNN](https://bugs.swift.org/browse/SR-NNNN), [SR-MMMM](https://bugs.swift.org/browse/SR-MMMM) -->
<!--- * Previous Revision: [1](https://github.com/apple/swift-evolution/blob/...commit-ID.../proposals/NNNN-filename.md) -->
<!--- * Previous Proposal: [SE-XXXX](XXXX-filename.md) -->

## Introduction

We propose adding new `borrow` and `inout` keywords that
can be used by themselves and in pattern-matching
operations.
Like `let`, these constructs define a new symbol
and must be initialized exactly once.
Unlike `let` and `var` _value bindings_ which
create a copy of their initial value,
these _reference bindings_
instead create a _reference_ to an existing
value that allows the value to be used or updated
"in-place."

These new keywords can be used to selectively
improve performance of critical code,
enable convenient idioms for "in-place"
access and modification of values stored in
containers, and
can be used with values and types that do not
support copying.

Pitch discussion thread: [Pitch: `borrow` and `inout` declaration keywords](https://forums.swift.org/t/pitch-borrow-and-inout-declaration-keywords/62366)


## Motivation

The Swift language has two core keywords used to define
new names that can be used to refer to values:

* `let` introduces a name that must be initialized
  exactly once

* `var` introduces a name that must be initialized
  before it is used and can be updated any number of times

These keywords are used standalone, as integral components
of pattern-matching constructs, and are implied by yet
other constructs:
```swift
// Define mutable symbol `x`, initialize with new `Foo`
var x = Foo()

// Copy the payload of the optional into a new symbol `x`
if let x = optional_value {
  // `x` is a copy of the optional payload
}

// Copy payload values out of an enum
switch enum_value {
case .a(let x):
  // `x` is a non-modifiable copy of the payload
case .b(var y):
  // `y` is a modifiable copy of the payload
}
```

Note that in most of the examples above, the `let` or `var`
causes an implicit copy of its initial value.
This copying causes a number of problems in practice:

* Although the optimizer is frequently able to remove unnecessary copy
  operations introduced in these cases, this can change in non-obvious
  ways as the code changes over time and as the compiler optimizer
  itself evolves.

* When the optimizer cannot remove the copy, it can lead to a variety
  of performance issues.  For example, it can lead to redundant
  references to copy-on-write collections which can in turn cause
  small mutations to become quite expensive.

* When the intent is to modify data in-place, it is easy to forget to
  write the updated value back into the original container.

* Ongoing work on C++ interoperability and Swift noncopyable
  types will soon allow Swift programs to work with values that
  do not support copying at all.
  Such values cannot easily be used with the existing `let` and
  `var` constructs.


## Proposed solution

We propose adding new keywords
that define an alternate name
for accessing an existing property or variable.
These names define _references_ to an
existing value in memory.
We'll refer to the original value as the _target_
of the reference.
The reference declaration keywords are

* `borrow` introduces a read-only reference;
  reading the reference produces the same result as
  reading the target

* `inout` introduces a read/write reference;
  read or write operations on the reference affect
  the target

```swift
// Non-copying version of `if let y = optional`
// `inout` here requires marking the input with
// `&` to signify that it may be mutated
if inout y = &optional {
  // `y` is a read/write reference to the payload
  // of `optional`
  // So we can mutate the payload in place
  y += 1
}

// Non-copying version of `let z = ...`
borrow z = long.and.complex.path.to.uncopyable
// `z` is a reference to the uncopyable object
// that provides a simple shorthand
z.operation()
print(z)
```


## Detailed design

These new keywords can be used in a number of contexts as described
in the following sections.

### Standalone reference declarations

These keywords can be used to declare
new names initialized as a reference to
some existing value:
```swift
borrow x = a.b
inout y = &c.d
```
These names are valid until the end of the enclosing
lexical scope (the trailing `}` of the innermost function,
`do` statement, or closure).

Note the use of the `&` sigil for the `inout`
initializer to reflect that the target
might be mutated by the following code.
This parallels the use of `&` to mark
values being passed to `inout` function parameters.

Each of these constructs must be initialized
at the point of declaration to reference a specific
target.
The target for `borrow` or `inout`
must be a local variable, function argument,
closure capture, stored property, computed
property, subscript.
For a `borrow`, the target must be readable;
for an `inout`, it must be both readable and
writable.
Once initialized, the target cannot be changed.

Note that the target for a `borrow` or `inout`
reference cannot be a global variable,
the temporary result of an operator or function call,
or initializer result.
Among other constructs, this rules out the following:
```
inout z = use_a ? &a : &b  // Illegal
borrow y = a ?? b  // Illegal
```


### Exclusivity and scope

A `borrow` or `inout` binding is an access (read or read/write, respectively)
of the target that lasts for the lifetime of the reference.
As such, Swift's usual exclusivity rules (as laid out in
[SE-0176](https://github.com/apple/swift-evolution/blob/main/proposals/0176-enforce-exclusive-access-to-memory.md))
affect how these
references can be used and what happens if you try to access
the underlying value through other means.

For `borrow` references, the read access ensures that
the original value cannot be mutated within the lifetime
of the reference, though it can be read through any valid
`borrow` reference or directly:
```swift
var t: T
borrow b = t
borrow c = t.x
t = T() // Illegal: Exclusivity violation
b = T() // Illegal: `borrow` cannot be modified
t.foo() // Legal: borrow does not prevent direct access
b.foo() // Legal: access via the borrow 
c.foo() // Legal: access via another borrow
```

In contrast, `inout` references introduce a read/write
access that does prohibit any other read or write
to the target:
```swift
var t: T
do {
  inout i = &t
  i = T() // Legal: inout reference allows modification
  i.foo() // Legal: inout reference allows reading target
  t.foo() // Illegal: Exclusivity violation
  t = T() // Illegal: Exclusivity violation
}
// `t` now reflects changes made via `i` above
t.foo() // Legal: `i` is no longer in scope
t = T() // Legal: `i` is no longer in scope
```

It is possible to `borrow` the same
value multiple times or `borrow` separate
properties of an aggregate,
even in the presence of an `inout`.
But multiple `inout` bindings are not
legal.
These interact as specified in SE-0176.
To summarize:
* Binding any property of an aggregate is the same as binding the entire aggregate for purposes of exclusivity checking
* A new `inout` binding cannot be formed in the scope of an existing `inout` or `borrow` binding.
* A `borrow` binding can follow another `borrow` or `inout` binding

For example:
```
var a = A()
do {
  inout x = a.b
  a.foo() // Illegal exclusivity violation
  inout y = a.c // Illegal exclusivity violation
  borrow z = a // Legal
  print(x.z) // Legal
  x.z = 1 // Illegal exclusivity violation
  print(z) // Legal
}
```

### Visibility of mutations

When a `borrow` or `inout` reference is initialized
to a simple stored property or local variable,
the compiler is expected to implement it with
a pointer to the underlying storage if
no more efficient form is possible.

For a computed property implemented with `get`/`set`
accessors, the property will be read at the point
the name is declared and the changed value will be
written back (for `inout` references) when the name goes out
of scope.
The following outlines the expected behavior:
```swift
struct A {
  var b: B { get { ... } set { ... } }
}
var a = A()
do {
  inout x = &a.b
  // * `get` method is called
  // * value is stored in a temporary 
  // * `x` is a reference to that temporary
  x += 1
  x.someMutatingMethod()
  x.someOtherMethod()
  // * `set` method is called with the mutated value
  // * `x` goes out of scope
}
```

### Not implicitly copyable

These new binding keywords express a programmer's
intent to reduce the copying of certain values.
As such, the symbols expressed by the keywords
prohibit implicit copying.

For example, if `a` is a copyable value of type `A`, then:
```swift
borrow x = a    // Borrow `a`
let b = a       // Legal; `a` is still copyable
let y = x       // Illegal; `x` is not implicitly copyable
func use(_ a: A) { ... }
use(x)          // Legal; argument can be borrowed
func consuming_use(_ a: consuming A) { ... }
consuming_use(x) // Illegal; requires copying x 
func borrowing_use(_ a: borrow A) { ... }
borrowing_use(x) // Legal; `borrow` argument marker avoids copy
let z = copy x  // Legal way to explicitly copy x
```

### Explicit `copy` operator

As illustrated above, `borrow` and `inout` bindings
create a new name that is not implicitly copyable.
But if the underlying type is in fact copyable, there
can be a need to make an explicit copy (for example,
to preserve the original value in order to rollback
a failed operation).
For that reason, we also propose a new `copy` keyword
operator that allows an explicit copy operation of
its argument.

For implicitly-copyable types, this new keyword
has no effect:
```
struct CopyableStruct { ... }
let a = CopyableStruct()
let b = copy a // Same as `let b = a`
```

Also note that this new keyword does not provide a way
to copy values that are inherently noncopyable:
```
@noncopyable struct NoncopyableStruct { ... }
let a = NoncopyableStruct()
let b = a // Illegal: `a` is noncopyable
let b = copy a // Illegal: `a` is noncopyable
```

This keyword only has an effect on values that
are inherently copyable (according to their type)
but are currently restricted from being implicitly
copied:
```
struct CopyableStruct { ... }
let a = CopyableStruct()
borrow b = a
let c = b // Illegal: `b` is not implicitly copyable
let d = a // Legal: `a` is implicitly copyable
let e = copy b // Legal: Explicit copy is okay
```

### Optional unwrapping

These new keywords can be used in place of `let` in
`if let` constructs in order to test and unwrap an
optional value.
As with `if let`, the initializer must be an optional
value:
```swift
if borrow x = optional_expression {
  // If the value produced by optional_expression
  // is non-nil, `x` will be a reference to the payload of
  // that value.
}

if inout y = &optional_expression {
  // If the value produced by optional_expression
  // is non-nil, `y` will be a read/write reference
  // to the payload of that value.
  // Assignments to `y` will update the
  // payload of the optional:
  y = new_value // Set optional to .some(new_value)
}
```


### `switch` and `if case` bindings

The `borrow` and `inout` keywords can be used to
access the payload of an enum used in a `switch`
statement:
```swift
switch enum_value {
case .a(borrow x):
   // `x` is a reference to the payload of the
   // enum that allows you to access it without
   // the copy implied by a `let` or `var`
   x.method()
   ...
}

switch &enum_value {
case .a(inout x):
   // `x` is a read/write reference to the enum payload
   // that allows you to mutate it in-place
   x += 1 // Directly modify `enum_value`
   ...
}
```

Note that the `&` sigil is required on the argument to the
`switch` statement if there are any `inout` references within
that `switch` statement.
As with `inout` function parameters, the `&` sigil indicates
at the point of use that the value is subject to possible mutation.


## Source compatibility

These new keywords add new syntax that does not conflict with
existing syntax.
In particular, the parser can distinguish the `borrow` keyword
in `borrow x` from a call to a function `borrow(x)`,
so this new functionality should not break existing source.


## Effect on ABI stability and API resilience

These are purely additive language features with
no effect on existing ABI.
They do not change how code is invoked across
module boundaries, and so have no effect on API resilience.


## Alternatives considered

**Special-casing noncopyable values:**
An alternative way to support noncopyable types would
be for the compiler to change the behavior of
the existing `switch`, `if case`, `if let`, and `for..in`
statements whenever noncopyable values were involved.
But we feel this could cause considerable confusion at the point of
use, for example, if you have an enum where case `a` has a
non-copyable payload and case `b` has a copyable payload:
```swift
switch foo {
case .a(var uncopyable): // Alternative to `inout uncopyable`
   // `uncopyable` here is a reference?
   // So this modifies `foo`
   uncopyable += 1
case .b(var copyable):
   // `copyable` here is a copy of the payload?
   // So this does not modify `foo`
   copyable += 1
}
```
Also note that with such an approach, the behavior
of code such as the above would change if the types
in question were changed to be copyable or noncopyable,
leading to action-at-a-distance effects.
By making a clear distinction between copying and referencing
behavior, the different keywords described in this proposal
avoid such confusion.

**Avoiding `&` sigils:**
The `&` sigil is technically redundant in many of the
above constructs.
For example, we could have omitted it for standalone
`inout` bindings:
```
inout y = a  // Possible alternate syntax
```
But for `switch` statements in particular,
there can be a considerable distance between
the point where the underlying value is named
and the point where the binding is selected:
```
switch &enum_value {
  // ...  many, many lines of code ...
  case .a(inout x):
    x += 1
  // ... many, many lines of code ...
}
```
In these cases, the `&` sigil serves the
important purpose of making it obvious in
the `switch` statement introduction that
the value may be mutated by one or more of
the cases.

Retaining the sigil for the other `inout` syntaxes
described in this proposal is less critical
for legibility, but we felt it was important
for consistency.

**Alternate Spellings:**
Several alternate spellings have
been suggested in various discussions.

The term "borrow" for the read-only reference has
generally been well-received by commentators,
although there has been some concern about
proliferation of declaration keywords and
whether "borrow" is sufficient by itself to
convey the read-only nature.

In place of our `borrow`, various commentators have suggested
`borrowing`, `borrowed`, `borrow let`, `borrow(let)`, `let(borrow)`,
`let &`, `&let`, `ref`, `ref let`, `rlet`, `hold`, and `alias`.
The possibility of annotating the reference name has also
been put forward: `let &x = a`, `for &x in collection`, 

The use of `inout` here has been more controversial with
extensive debate about whether the functionality was
similar enough to the argument-passing use to merit the
same keyword and concerns about the readability of this
specific word in these contexts.

Alternatives suggested have included `borrow var`,
`borrow(var)`, `var(borrow)`, `var &`, `ref`, `ref var`, `rvar`,
`mutate`, `mutating`, and `mutating alias`.

There has also been some concern about the
length of the proposed `borrow` and `inout`
keywords compared to the more succinct `var` and `let`,
especially for systems programmers who expect to
make heavy use of these new constructs.

Several people also observed the need to ensure these
are aligned with similar concepts being proposed for `for..in`
loops and possibly elsewhere.

We invite readers to substitute these terms into
the various examples presented earlier in this
proposal and give us feedback about how well each
option works in each context.


### Future directions

**Binding keywords with `for..in` loops:**
The `borrow` and `inout` binding keywords have obvious
advantages for `for..in` loops:
```swift
for borrow x in array {
  // This does not involve copying each element
  x.something()
}

for inout y in &array {
  // The following updates the array elements "in place"
  y += 1
}
```
This is more subtle than it might appear, so we are
deferring this to a separate proposal.

**Mixing copying and non-copying switch case bindings:**
There are some subtleties involved when both copying and non-copying
semantics are used for the same value in different cases of a switch
statement:
```
switch foo {
case .int(inout i):
  i += 1
  fallthrough
case .int(var i):
  i += 1
  print(i)
default: ...
}
```
In the example above, the two `i` are fundamentally different;
the first is a _reference_ for the payload of the enum, the second
is a _copy_ of that payload.
As such, the first `i += 1` updates the payload,
the second does not.

To reduce confusion in such cases, and to simplify the initial
implementation, we proposed above that no single switch
statement can contain both copying (`let`, `var`) and referencing
(`borrow`, `inout`) keywords.
This restriction makes all of the following invalid:
```
let tuple : (Int, Float)
switch &tuple {
case (let i, inout f): ...  // Unsupported: Cannot mix `let` and `inout` bindings
}

switch &foo {
case .int(inout i):
  print(i)
case .int(let i): // Unsupported:  Cannot mix `let` and `inout` bindings
  i += 1
}
```
We understand this is likely to be restrictive
in practice and look forward to revisiting this behavior
in a future proposal once we have more experience with
this feature.

**Definite initialization:**
Each `borrow` or `inout` declaration creates a reference to
a single value
and cannot be changed once it is initialized.
As with other Swift constructs, they must be initialized before
being used.

The syntax for `if inout`, `if borrow`, `if consume`, `switch` case bindings,
and `for..in` loops requires that the symbol be initialized
immediately.

For implementation simplicity, we also require in this proposal
that standalone bindings be initialized immediately:
```swift
borrow x = a.b
```

But it would be entirely consistent to allow bindings to
be handled similarly to `let`, requiring only that they
are initialized exactly once before any use:
```swift
borrow x
if condition {
  x = a
} else {
  x = b
}
print(x)
```
Again, we look forward to relaxing this requirement in
a future proposal.

**Use with modify and read accessors:**
We've described how these references should behave for computed properties
defined with `get` and `set` accessors.
We expect them to work similarly for properties defined
with `modify` and `read` accessors,
once such accessors are formally proposed and accepted
into the language.

**First-class references:**
The `borrow` and `inout` keywords would also be useful for
struct properties that hold references:
```swift
struct Foo {
   borrow x: Int // Read-only reference to an Int
}
```
This requires additional tools to ensure that
such references never outlive their targets.

**`consume` keyword:**
The `borrow` and `inout` constructs define names
that cannot be used to implicitly copy the associated
value,
in contrast to `let` and `var` which do allow copying
of their value.
This distinction alone seems useful and suggests that
we also include a `consume` definition that provides
complementary support by taking direct ownership of
a value rather than creating a reference:
```
// Ends lifetime of `a`
// Moves value to `x`
// `x` is not implicitly copyable
consume x = a

// Constructs reference `y` to value stored in `b`
// `y` is not implicitly copyable
borrow y = b

// Constructs reference `z` to storage `c`
// `z` is not implicitly copyable
inout z = &c
```
This would provide an alternative to the `@noImplicitCopy` attribute being discussed elsewhere.


## Acknowledgments

Thanks to Joe Groff, Michael Gottesman, Kavon Farvardin, Guillaume Lessard, and Andrew Trick for extensive discussions about this feature.
