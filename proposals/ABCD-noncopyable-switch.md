# Borrowing and consuming pattern matching for noncopyable types

* Proposal: [SE-ABCD](ABCD-noncopyable-switch.md)
* Authors: [Joe Groff](https://github.com/jckarter),
* Review Manager: TBD
* Status: **Awaiting review**
* Implementation: on `main`, using the `BorrowingSwitch` feature flag and `_borrowing x` binding spelling
* Upcoming Feature Flag: `BorrowingSwitch`

## Introduction

Pattern matching over noncopyable types, particularly noncopyable enums, can
be generalized to allow for pattern matches that borrow their subject, in
addition to the existing support for consuming pattern matches.

## Motivation

[SE-0390](https://github.com/apple/swift-evolution/blob/main/proposals/0390-noncopyable-structs-and-enums.md)
introduced noncopyable types, allowing for programs to define
structs and enums whose values cannot be copied. However, it restricted
`switch` over noncopyable values to be a `consuming` operation, meaning that
nothing can be done with a value after it's been matched against. This
severely limits the expressivity of noncopyable enums in particular,
since switching over them is the only way to access their associated values.

## Proposed solution

We lift the restriction that noncopyable pattern matches must consume their
subject value. To enable this, we introduce **borrowing bindings** into
patterns, and formalize the ownership behavior of patterns during matching
and dispatch to case blocks. `switch` statements **infer their ownership
behavior** based on the necessary ownership behavior of the patterns in
the `switch`.

## Detailed design

### `borrowing` bindings

Patterns can currently contain `var` and `let` bindings, which take part
of the matched value and bind it to a new independent variable in the
matching `case` block:

```
enum MyCopyableEnum {
    case foo(String)

    func doStuff() { ... }
}

var x: MyCopyableEnum = ...

switch x {
case .foo(let y):
    // We can pass `y` off somewhere else, or capture it indefinitely
    // in a closure, such as to use it in a detached task
    Task.detached {
        print(y)
    }

    // We can use `x` and update it independently without disturbing `y`
    x.doStuff()
    x = MyEnum.foo("38")

}
```

For copyable types, we can ensure the pattern bindings are independent by
copying the matched part into the new variable, but for noncopyable bindings,
their values can't be copied and need to be moved out of the original value,
consuming the original in the process:

```
struct Handle: ~Copyable {
    var value: Int

    borrowing func access() { ... }

    consuming func close() { ... }
}

enum MyNCEnum: ~Copyable {
    case foo(Handle)

    borrowing func doStuff() { ... }
    consuming func throwAway() { ... }
}

var x: MyNCEnum = ...
switch x {
case .foo(let y):
    // We can pass `y` off somewhere else, or capture it indefinitely
    // in a closure, such as to use it in a detached task
    Task.detached {
        y.access()
    }
    
    // ...but we can't copy `Handle`s, so in order to support that, we have to
    // have moved `y` out of `x`, leaving `x` consumed and unable to be used
    // again
    x.doStuff() // error: 'x' consumed
}

// Since the pattern match had to consume the value, we can't even use it
// after the switch is done.
x.doStuff() // error: 'x' consumed
```

We introduce a new `borrowing` binding modifier. A `borrowing` binding
references the matched part of the value as it currently exists in
the subject value without copying it, instead putting the subject under
a *borrowing access* in order to access the matched part.

```
var x: MyNCEnum = ...
switch x {
case .foo(borrowing y):
    // `y` is now borrowed directly out of `x`. This means we can access it
    // borrowing operations:
    y.access()

    // and we can still access `x` with borrowing operations as well:
    x.doStuff()

    // However, we can't consume `y` or extend its lifetime beyond the borrow
    Task.detached {
        y.access() // error, can't capture borrow `y` in an escaping closure
    }
    y.close() // error, can't consume `y`
    
    // And we also can't consume or modify `x` while `y` is borrowed out of it
    x = .foo(Handle(value: 42)) // error, can't modify x while borrowed
    x.throwAway() // error, can't consume x while borrowed
}

// And now `x` was only borrowed by the `switch`, so we can continue using
// it afterward
x.doStuff()
x.throwAway()
x = .foo(Handle(value: 1738))
```

`borrowing` bindings can also be formed when the subject of the pattern
match and/or the subpattern have `Copyable` types. Like `borrowing` parameter
bindings, a `borrowing` pattern binding is not implicitly copyable in the
body of the `case`, but can be explicitly copied using the `copy` operator.

```
var x: MyCopyableEnum = ...

switch x {
case .foo(borrowing y):
    // We can use `y` in borrowing ways.

    // But we can't implicitly extend its lifetime or perform consuming
    // operations on it, since those would need to copy
    var myString = "hello"
    myString.append(y) // error, consumes `y`
    Task.detached {
        print(y) // error, can't extend lifetime of borrow without copying
    }

    // Explicit copying makes it ok
    Task.detached {[y = copy y] in
        print(y)
    }
    myString.append(copy y)

    // `x` is still copyable, so we can update it independently without
    // disturbing `y`
    x.doStuff()
    x = MyEnum.foo("38")

}
```

To maintain source compatibility, `borrowing` is parsed as a contextual
keyword only when it appears immediately before an identifier name. In
other positions, it parses as a declaration reference as before, forming
an enum case pattern or expression pattern depending on what the name
`borrowing` refers to.

```
switch y {
case borrowing(x): // parses as an expression pattern
    ...
case borrowing(let x): // parses as an enum case pattern binding `x` as a let
    ...
case borrowing.foo(x): // parses as an expression pattern
    ...
case borrowing.foo(let x): // parses as an enum case pattern binding `x` as a let
    ...
case borrowing x: // parses as a pattern binding `x` as a borrow
    ...
case borrowing(borrowing x) // parses as an enum case pattern binding `x` as a borrow
    ...
}
```

This does mean that, unlike `let` and `var`, `borrowing` cannot be applied
over a compound pattern to mark all of the identifiers in the subpatterns
as bindings.

```
case borrowing .foo(x, y): // parses as `borrowing.foo(x, y)`, a method call expression pattern

case borrowing (x, y): // parses as `borrowing(x, y)`, a function call expression pattern
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

```
extension Handle {
    var isReady: Bool { ... }
}

let x: MyNCEnum = ...
switch x {
// OK to have `let y` in multiple patterns because we can delay consuming
// `x` to form bindings until we establish a match
case .foo(let y) where y.isReady:
    y.close()
case .foo(let y):
    y.close()
}
```

However, when a pattern has a `where` clause, variable bindings cannot be
consumed in the where clause even if the binding is consumable in the case
body:

```
extension Handle {
    consuming func tryClose() -> Bool { ... }
}

let x: MyNCEnum = ...
switch x {
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

```
extension Handle {
    static func ~=(identifier: Int, handle: consuming Handle) -> Bool { ... }
}

switch x {
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
result by consuming if necessary. However, for this to be allowed, the
subpattern `p` of the `p as T` pattern would need to be irrefutable, and the
pattern could not have an associated `where` clause, since we would be unable
to back out of the pattern match once a consuming cast is performed.

### Determining the ownership behavior of a `switch` operation

Whether a `switch` borrows or consumes its subject can be determined from
the type of the subject and the patterns involved in the switch. Based on
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
whereas for noncopyable subjects, *borrowing* is the baseline mode. While
looking through the patterns:

- if there is a `borrowing` binding subpattern, then the `switch` behavior is
  at least *borrowing*.
- if there is a `let` or `var` binding subpattern, and the subpattern is of
  a noncopyable type, then the `switch` behavior is *consuming*. If the
  subpattern is copyable, then `let` bindings do not affect the behavior
  of the `switch`, since the binding value can be copied if necessary to
  form the binding.
- if there is an `as T` subpattern, and the type of the value being matched
  is noncopyable, then the `switch` behavior is *consuming*. If the value
  being matched is copyable, there is no effect on the behavior of the
  `switch`.

For example, given the following copyable definition:

```
enum CopyableEnum {
    case foo(Int)
    case bar(Int, String)
}
```

then the following patterns have ownership behavior as indicated below:

```
case let x: // copying
case borrowing x: // borrowing

case .foo(let x): // copying
case .foo(borrowing x): // borrowing

case .bar(let x, let y): // copying
case .bar(borrowing x, let y): // borrowing
case .bar(let x, borrowing y): // borrowing
case .bar(borrowing x, borrowing y): // borrowing
```

And for a noncopyable enum definition:

```
struct NC: ~Copyable {}

enum NoncopyableEnum: ~Copyable {
    case copyable(Int)
    case noncopyable(NC)
}
```

then the following patterns have ownership behavior as indicated below:

```
case let x: // consuming
case borrowing x: // borrowing

case .copyable(let x): // borrowing (because `x: Int` is copyable)
case .copyable(borrowing x): // borrowing

case .noncopyable(let x): // consuming
case .noncopyable(borrowing x): // borrowing
```

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

## Alternatives considered

### Explicit marking of switch ownership

SE-0390 required all `switch` statements on noncopyable bindings to use an
explicit `consume`. Rather than infer the ownership behavior from the 
patterns in a `switch`, as we propose, we could alternatively keep the
requirement that a noncopyable `switch` explicitly mark its ownership. Using
the [`borrow` operator](https://forums.swift.org/t/selective-control-of-implicit-copying-behavior-take-borrow-and-copy-operators-noimplicitcopy/60168)
which has previously been proposed could serve as an explicit marker that
a `switch` should perform a borrow on its subject. This proposal chooses
not to require these explicit markers, though `consume` (and `borrow` when
it's introduced) can still be explicitly used if the developer chooses to
enforce that a particular `switch` either consumes or borrows its subject.

