# Borrowing and consuming pattern matching for noncopyable types

* Proposal: [SE-0432](0432-noncopyable-switch.md)
* Authors: [Joe Groff](https://github.com/jckarter)
* Review Manager: [Ben Cohen](https://github.com/airspeedswift)
* Status: **Implemented (Swift 6.0)**
* Implementation: on `main`, using the `BorrowingSwitch` experimental feature flag and `_borrowing x` binding spelling
* Experimental Feature Flag: `BorrowingSwitch`
* Previous Revision: [1](https://github.com/swiftlang/swift-evolution/blob/86cf6eadcdb35a09eb03330bf5d4f31f2599da02/proposals/ABCD-noncopyable-switch.md)
* Review: ([review](https://forums.swift.org/t/se-0432-borrowing-and-consuming-pattern-matching-for-noncopyable-types/71158)) ([acceptance](https://forums.swift.org/t/accepted-with-modifications-se-0432-borrowing-and-consuming-pattern-matching-for-noncopyable-types/71656))

## Introduction

Pattern matching over noncopyable types, particularly noncopyable enums, can
be generalized to allow for pattern matches that borrow their subject, in
addition to the existing support for consuming pattern matches.

## Motivation

[SE-0390](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0390-noncopyable-structs-and-enums.md)
introduced noncopyable types, allowing for programs to define
structs and enums whose values cannot be copied. However, it restricted
`switch` over noncopyable values to be a `consuming` operation, meaning that
nothing can be done with a value after it's been matched against. This
severely limits the expressivity of noncopyable enums in particular,
since switching over them is the only way to access their associated values.

## Proposed solution

We lift the restriction that noncopyable pattern matches must consume their
subject value, and  formalize the ownership behavior of patterns during
matching and dispatch to case blocks. `switch` statements **infer their
ownership behavior** based on a combination of whether the subject expression
refers to storage or a temporary value, in addition to the necessary ownership
behavior of the patterns in the `switch`. We also introduce **borrowing
bindings** into patterns, as a way of explicitly declaring a binding as a
borrow that doesn't allow for implicit copies. 

## Detailed design

### Determining the ownership behavior of a `switch` operation

Whether a `switch` borrows or consumes its subject can be determined from
the subject expression and the patterns involved in the switch. Based on
the criteria below, a switch may be one of:

- **copying**, meaning that the subject is semantically copied, and additional
  copies of some or all of the subject value may be formed to execute the
  pattern match.
- **borrowing**, meaning that the subject is borrowed for the duration of the
  `switch` block.
- **consuming**, meaning that the subject is consumed by the `switch` block.

These modes can be thought of as being increasing in strictness. The compiler
looks recursively through the patterns in the `switch` and increases the
strictness of the `switch` behavior when it sees a pattern requiring stricter
ownership behavior. For copyable subjects, *copying* is the baseline mode, 
whereas for noncopyable subjects, the baseline mode depends on the subject
expression:

- If the expression refers to a variable or stored property, and is not
  explicitly consumed using the `consume` operator, then the baseline
  mode is *borrowing*. (Properties and subscripts which use the experimental
  `_read`, `_modify`, or `unsafeAddress` accessors also get a baseline mode
  of borrowing.)
- Otherwise, the baseline mode is *consuming*.

For example, given the following copyable definition:

```swift
enum CopyableEnum {
    case foo(Int)
    case bar(Int, String)
}
```

then the following patterns have ownership behavior as indicated below:

```swift
case let x: // copying
case .foo(let x): // copying
case .bar(let x, let y): // copying
```

And for a noncopyable enum definition:

```swift
struct NC: ~Copyable {}

enum NoncopyableEnum: ~Copyable {
    case copyable(Int)
    case noncopyable(NC)
}
```

then the following patterns have ownership behavior as indicated below:

```swift
var foo: NoncopyableEnum // stored variable

switch foo {
case let x: // borrowing

case .copyable(let x): // borrowing (because `x: Int` is copyable)

case .noncopyable(let x): // borrowing
}

func bar() -> NoncopyableEnum {...} // function returning a temporary

switch bar() {
case let x: // consuming
case .copyable(let x): // borrowing (because `x: Int` is copyable)
case .noncopyable(let x): // consuming
}
```

### Refining the ownership behavior of `switch`

The order in which `switch` patterns are evaluated is unspecified in Swift,
aside from the property that when multiple patterns can match a value,
the earliest matching `case` condition takes priority. Therefore, it is
important that matching dispatch **cannot mutate or consume the subject**
until a final match has been chosen. For copyable values, this means that
pattern matching operations can't mutate the subject, but they can be copied
as necessary to keep an instance of the subject available throughout the
pattern match even if a match operation wants to consume an instance of
part of the value.

Copying isn't an option for noncopyable types, so
**noncopyable types strictly cannot undergo `consuming` operations until 
the pattern match is complete**. For many kinds of pattern matches, this
doesn't need to affect their expressivity, since checking whether a type
matches the pattern criteria can be done nondestructively separate from
consuming the value to form variable bindings. Matching enum cases and tuples
(when noncopyable tuples are supported) for instance is still possible
even if they contain consuming `let` or `var` bindings as subpatterns:

```swift
extension Handle {
    var isReady: Bool { ... }
}

let x: MyNCEnum = ...
switch consume x {
// OK to have `let y` in multiple patterns because we can delay consuming
// `x` to form bindings until we establish a match
case .foo(let y) where y.isReady:
    y.close()
case .foo(let y):
    y.close()
}
```

However, when a pattern has a `where` clause, variable bindings cannot be
consumed in the `where` clause even if the binding is consumable in the `case`
body:

```swift
extension Handle {
    consuming func tryClose() -> Bool { ... }
}

let x: MyNCEnum = ...
switch consume x {
// error: cannot consume `y` in a "where" clause
case .foo(let y) where y.tryClose():
    // OK to consume in the case body
    y.close()
case .foo(let y):
    y.close()
}
```

Similarly, an expression subpattern whose `~=` operator consumes the subject
cannot be used to test a noncopyable subpattern.

```swift
extension Handle {
    static func ~=(identifier: Int, handle: consuming Handle) -> Bool { ... }
}

switch consume x {
// error: uses a `~=` operator that would consume the subject before
// a match is chosen
case .foo(42):
    ....
case .foo(let y):
    ...
}
```

Noncopyable types do not yet support dynamic casting, but it is worth
anticipating how `is` and `as` patterns will work given this restriction.
An `is T` pattern only needs to determine whether the value being matched can
be cast to `T` or not, which can generally be answered nondestructively.
However, in order to form the value of type `T`, many kinds of casting,
including casts that bridge or which wrap the value in an existential
container, need to consume or copy parts of the input value in order to form
the result. The cast can still be separated into a check whether the type
matches, using a borrowing access, followed by constructing the actual cast
result by consuming if necessary. To do this, the switch would have already
be a consuming switch. But also, for a consuming `as T` pattern to work, the
subpattern `p` of the `p as T` pattern would need to be irrefutable, and the
pattern could not have an associated `where` clause, since we would be unable
to back out of the pattern match once a consuming cast is performed.

### `case` conditions in `if`, `while`, `for`, and `guard`

Patterns can also appear in `if`, `while`, `for`, and `guard` forms as part
of `case` conditions, such as `if case <pattern> = <subject> { }`. These behave
just like `switch`es with one `case` containing the pattern, corresponding
to a true condition result with bindings, and a `default` branch corresponding
to a false condition result. Therefore, the ownership behavior of the `case`
condition on the subject follows the behavior of that one pattern.

## Source compatibility

SE-0390 explicitly required that a `switch` over a noncopyable variable
use the `consume` operator. This will continue to work in most cases, forcing
the lifetime of the binding to end regardless of whether the `switch` actually
consumes it or not. In some cases, the formal lifetime of the value or parts
of it may end up different than the previous implementation, but because
enums cannot yet have `deinit`s, noncopyable tuples are not yet supported,
and structs with `deinit`s cannot be partially destructured and must be
consumed as a whole, it is unlikely that this will be noticeable in real
world code.

Previously, it was theoretically legal for noncopyable `switch`es to use
consuming `~=` operators, or to consume pattern bindings in the `where`
clause of a pattern. This proposal now expressly forbids these formulations.
We believe it is impossible to exploit these capabilities in practice under the
old implementation, since doing so would leave the value partially or fully
consumed on the failure path where the `~=` match or `where` clause fails,
leading to either mysterious ownership error messages, compiler crashes, or
both.

## ABI compatibility

This proposal has no effect on ABI.

## Future directions

### `inout` pattern matches

With this proposal, pattern matches are able to *borrow* and *consume* their
subjects, but they still aren't able to take exclusive `inout` access to a
value and bind parts of it for in-place mutation. This proposal lays the
groundwork for supporting this in the future; we could introduce `inout`
bindings in patterns, and introducing **mutating** switch behavior as a level
of ownership strictness between *borrowing* and *consuming*.

### Automatic borrow deduction for `let` bindings, and explicitly `consuming` bindings

When working with copyable types, although `let` and `var` bindings formally
bind independent copies of their values, in cases where it's semantically
equivalent, the compiler optimizes aways the copy and borrows the original
value in place, with the idea that developers do not need to think about
ownership if the compiler does an acceptable job of optimizing their code.
By similar means, we could say that `let` pattern bindings for noncopyable types
borrow rather than consume their binding automatically if the binding is
not used in a way that requires it to consume the binding. This would
give developers a "do what I mean" model for noncopyable types closer to the
convenience of copyable types. This should be a backward compatible change
since it would allow for strictly more code to compile than does currently
when `let` bindings are always consuming.

Conversely, performance-minded developers would also like to have explicit
control over ownership behavior and copying, while working with either
copyable or noncopyable types. To that end, we could add explicitly `consuming`
bindings to patterns as well, which would not be implicitly copyable, and
which would force the switch behavior mode on the subject to become *consuming*
even if the subject is copyable.

### enum `deinit`

SE-0390 left `enum`s without the ability to have a `deinit`, based on the fact
that the initial implementation of noncopyable types only supported consuming
`switch`es. Noncopyable types with `deinit`s generally cannot be decomposed,
since doing so would bypass the `deinit` and potentially violate invariants
maintained by `init` and `deinit` on the type, so an `enum` with a `deinit`
would be completely unusable when the only primitive operation supported on it
is consuming `switch`. Now that this proposal allows for `borrowing` switches,
we could allow `enum`s to have `deinit`s, with the restriction that such
enums cannot be decomposed by a consuming `switch`.

### Explicit `borrow` operator

The [`borrow` operator](https://forums.swift.org/t/selective-control-of-implicit-copying-behavior-take-borrow-and-copy-operators-noimplicitcopy/60168)
could be used in the future to explicitly mark the subject of a switch as
being borrowed, even if it is normally copyable or would be a consumable
temporary, as in:

```swift
let x: String? = "hello"

switch borrow x {
case .some(let y): // ensure y is bound from a borrow of x, no copies
    ...
}
```

### `borrowing` bindings in patterns

In the future, we want to support `borrowing` and `inout` local bindings
in functions and potentially even as fields in nonescapable types. It might
also be useful to specify explicitly `borrowing` bindings within patterns.
Although the default behavior for a `let` binding within a noncopyable
borrowing `switch` pattern is to borrow the matched value, an explicitly
`borrowing` binding could be used to indicate that a copyable binding should
have its local implicit copyability suppressed, like a `borrowing` parameter
binding.

## Alternatives considered

### Determining pattern match ownership wholly from patterns

The [first pitched revision](https://github.com/swiftlang/swift-evolution/blob/86cf6eadcdb35a09eb03330bf5d4f31f2599da02/proposals/ABCD-noncopyable-switch.md)
of this proposal kept `let` bindings in patterns as always being consuming
bindings, and required the use of `borrowing` bindings in every pattern in order
for a `switch` to act as a borrow. Early feedback using the feature found this
tedious; `borrowing` is more often a better default for accessing values
stored in variables and stored properties. This led us to the design now
proposed, where `let` behaves as a copying, consuming, or borrowing binding
based on the subject expression.
