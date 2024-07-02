# Nonescapable Types

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Andrew Trick](https://github.com/atrick), [Tim Kientzle](https://github.com/tbkka)
* Review Manager: TBD
* Status: **Awaiting implementation**
* Roadmap: [BufferView Language Requirements](https://forums.swift.org/t/roadmap-language-support-for-bufferview)
* Implementation: **Pending**
* Upcoming Feature Flag: `NonescapableTypes`
* Review: ([pitch](https://forums.swift.org/t/pitch-non-escapable-types-and-lifetime-dependency/69865))

## Introduction

We propose adding a new type constraint `~Escapable` for types that can be locally copied but cannot be assigned or transferred outside of the immediate context.
This complements the `~Copyable` types added with SE-0390 by introducing another set of compile-time-enforced lifetime controls that can be used for safe, highly-performant APIs.

In addition, these types will support lifetime-dependency constraints (being tracked in a separate proposal), that allow them to safely hold pointers referring to data stored in other types.

This feature is a key requirement for the proposed `StorageView` type.

**See Also**

* [SE-0390: Noncopyable structs and enums](https://github.com/apple/swift-evolution/blob/main/proposals/0390-noncopyable-structs-and-enums.md)
* [Language Support for Bufferview](https://forums.swift.org/t/roadmap-language-support-for-bufferview/66211)
* [Roadmap for improving Swift performance predictability: ARC improvements and ownership control](https://forums.swift.org/t/a-roadmap-for-improving-swift-performance-predictability-arc-improvements-and-ownership-control/54206)
* [Ownership Manifesto](https://forums.swift.org/t/manifesto-ownership/5212)
* [Draft StorageView Proposal](https://github.com/apple/swift-evolution/pull/2307)
* [Draft Lifetime Dependency Annotations Proposal](https://github.com/apple/swift-evolution/pull/2305)

## Motivation

Swift's current notion of an "iterator" has several weaknesses that become apparent when you try to use it in extremely performance-constrained environments.
These weaknesses arise from the desire to ensure safety while simultaneously allowing iterator values to be arbitrarily copied to support multi-iterator algorithms.

For example, the standard library iterator for Array logically creates a copy of the Array when it is constructed; this ensures that changes to the array cannot affect the iteration.
This is implemented by having the iterator store a reference-counted pointer to the array storage in order to ensure that the storage cannot be freed while the iterator is active.
These safety checks all incur runtime overhead.

In addition, the use of reference counting to ensure correctness at runtime makes types of this sort unusable in highly constrained embedded environments.

## Proposed solution

Currently, the notion of "escapability" appears in the Swift language as a feature of closures.
Nonescapable closures can use a very efficient stack-based representation;
closures that are `@escapable` store their captures on the heap.

By allowing Swift developers to mark various types as nonescapable, we provide a mechanism for them to opt into a specific set of usage limitations that:

* Can be automatically verified by the compiler. In fact, the Swift compiler internally already makes heavy use of escapability as a concept.
* Are strict enough to permit high performance. The compiler uses this concept precisely because values that do not escape can be managed much more efficiently.
* Do not interfere with common uses of these types.

For example, if an iterator type were marked as nonescapable, the compiler would produce an error message whenever the user of that type tried to copy or store the value in a way that might limit efficient operation.
These checks would not significantly reduce the usefulness of iterators which are almost always created, used, and destroyed in a single local context.
These checks would also still allow local copies for multi-iterator uses, with the same constraints applied to those copies as well.

A separate proposal will show how we can further improve safety by allowing library authors to impose additional constraints that bind the lifetime of the iterator to the object that produced it.
These "lifetime dependency" constraints can also be verified at compile time to ensure that the source of the iterator is not modified and that the iterator specifically does not outlive its source.

**Note**: We are using iterators here to illustrate the issues we are considering.
We are not at this time proposing any changes to Swift's current `Iterator` protocol.

## Detailed design

#### New Escapable Concept

We add a new suppressible protocol `Escapable` to the standard library and implicitly apply it to all current Swift types (with the sole exception of nonescapable closures).
`Escapable` types can be assigned to global variables, passed into arbitrary functions, or returned from the current function or closure.
This matches the existing semantics of all Swift types prior to this proposal.

Specifically, we will add this declaration to the standard library:

```swift
// An Escapable type may or may not be Copyable
protocol Escapable: ~Copyable {}
```

#### In concrete contexts, `~Escapable` indicates nonescapability

Using the same approach as used for `~Copyable` and `Copyable`, we use `~Escapable` to suppress the `Escapable` conformance on a type.

```swift
// Example: A type that is not escapable
struct NotEscapable: ~Escapable {
  ...
}
```

A nonescapable value is not allowed to escape the local context:
```swift
// Example: Basic limits on ~Escapable types
func f() -> NotEscapable {
  let ne = NotEscapable()
  borrowingFunc(ne) // OK to pass to borrowing function
  let another = ne // OK to make local copies
  globalVar = ne // 🛑 Cannot assign ~Escapable type to a global var
  return ne // 🛑 Cannot return ~Escapable type
}
```

**Note**:
The section "Returned nonescapable values require lifetime dependency" explains the implications for how you must write initializers.

Without `~Escapable`, the default for any type is to be escapable.  Since `~Escapable` suppresses a capability, you cannot put this in an extension.

```swift
// Example: Escapable by default
struct Ordinary { }
extension Ordinary: ~Escapable // 🛑 Extensions cannot remove a capability
```

Classes cannot be declared `~Escapable`.

#### In generic contexts, `~Escapable` suppresses the default Escapable requirement

When used in a generic context, `~Escapable` allows you to define functions or types that can work with values that might or might not be escapable.
That is, `~Escapable` indicates the default escapable requirement has been suppressed.
Since the values might not be escapable, the compiler must conservatively prevent the values from escaping:

```swift
func f<MaybeEscapable: ~Escapable>(_ value: MaybeEscapable) {
  // `value` might or might not be Escapable
  globalVar = value // 🛑 Cannot assign possibly-nonescapable type to a global var
}
f(NotEscapable()) // Ok to call with nonescapable argument
f(7) // Ok to call with escapable argument
```

[SE-0427 Noncopyable Generics](https://github.com/apple/swift-evolution/blob/main/proposals/0427-noncopyable-generics.md) provides more detail on
how suppressible protocols such as `Escapable` are handled in the generic type system.

**Note:**  There is no relationship between `Copyable` and `Escapable`.
Copyable or noncopyable types can be escapable or nonescapable.

#### Constraints on nonescapable local variables

A nonescapable value can be freely copied and passed into other functions, including async and throwing functions, as long as the usage can guarantee that the value does not persist beyond the current scope:

```swift
// Example: Local variable with nonescapable type
func borrowingFunc(_: borrowing NotEscapable) { ... }
func consumingFunc(_: consuming NotEscapable) { ... }
func inoutFunc(_: inout NotEscapable) { ... }
func asyncBorrowingFunc(_: borrowing NotEscapable) async -> ResultType { ... }

func f() {
  var value: NotEscapable
  let copy = value // OK to copy as long as copy does not escape
  globalVar = value // 🛑 May not assign to global
  SomeType.staticVar = value // 🛑 May not assign to static var
  async let r = asyncBorrowingFunc(value) // OK to pass borrowing
  borrowingFunc(value) // OK to pass borrowing
  inoutFunc(&value) // OK to pass inout
  consumingFunc(value) // OK to pass consuming
  // `value` was consumed above, but NotEscapable
  // is Copyable, so the compiler can insert
  // a copy to satisfy the following usage:
  borrowingFunc(value) // OK
}
```

#### Constraints on nonescapable arguments

A value of nonescapable type received as an argument is subject to the same constraints as any other local variable.
In particular, a nonescapable `consuming` argument (and all direct copies thereof) must actually be destroyed during the execution of the function.
This is in contrast to an _escapable_ `consuming` argument which can be disposed of by being returned or stored to an instance property or global variable.

#### Values that contain nonescapable values must be nonescapable

Stored struct properties and enum payloads can have nonescapable types if the surrounding type is itself nonescapable.
(Equivalently, an escapable struct or enum can only contain escapable values.)
Nonescapable values cannot be stored as class properties, since classes are always inherently escaping.

```swift
// Example
struct OuterEscapable {
  // 🛑 Escapable struct cannot have nonescapable stored property
  var nonesc: Nonescapable
}

enum EscapableEnum {
 // 🛑 Escapable enum cannot have a nonescapable payload
  case nonesc(Nonescapable)
}

struct OuterNonescapable: ~Escapable {
  var nonesc: Nonescapable // OK
}

enum NonescapableEnum: ~Escapable {
  case nonesc(Nonescapable) // OK
}
```

#### Returned nonescapable values require lifetime dependency

As mentioned earlier, a simple return of a nonescapable value is not permitted:
```swift
func f() -> NotEscapable { // 🛑 Cannot return a nonescapable type
  var value: NotEscapable 
  return value // 🛑 Cannot return a nonescapable type
}
```

A separate proposal describes “lifetime dependency annotations” that can relax this requirement by tying the lifetime of the returned value to the lifetime of some other object, either an argument to the function or `self` in the case of a method or computed property returning a nonescapable type.
In particular, struct and enum initializers (which build a new value and return it to the caller) cannot be written without some mechanism similar to that outlined in our companion proposal.

#### Globals and static variables cannot be nonescapable

Nonescapable values must be constrained to some specific local execution context.
This implies that they cannot be stored in global or static variables.

#### Closures and nonescapable values

Escaping closures cannot capture nonescapable values.
Nonescaping closures can capture nonescapable values subject only to the usual exclusivity restrictions.

Returning a nonescapable value from a closure requires explicit lifetime dependency annotations, as covered in the companion proposal.

#### Nonescapable values and concurrency

All of the requirements on use of nonescapable values as function arguments and return values also apply to async functions, including those invoked via `async let`.

The closures used in `Task.init`, `Task.detached`, or `TaskGroup.addTask` are escaping closures and therefore cannot capture nonescapable values.

#### Conditionally `Escapable` types

You can define types whose escapability varies depending on their generic arguments.
As with other conditional behaviors, this is expressed by using an extension to conditionally add a new capability to the type:

```swift
// Example: Conditionally Escapable generic type
// By default, Box is itself nonescapable
struct Box<T: ~Escapable>: ~Escapable {
  var t: T
}

// Box gains the ability to escape whenever its
// generic argument is Escapable
extension Box: Escapable where T: Escapable { }
```

This can be used in conjunction with other suppressible protocols.
For example, many general library types will need to be copyable and/or escapable following their contents.
Here's a compact way to declare such a type:
```swift
struct Wrapper<T: ~Copyable & ~Escapable> { ... }
extension Wrapper: Copyable where T: ~Escapable {}
extension Wrapper: Escapable where T: ~Copyable {}
```

The above declarations all in a single source file will result in a type `Wrapper` that is `Escapable` exactly when `T` is `Escapable` and `Copyable` exactly when `T` is `Copyable`.
To see why, first note that the explicit `extension Wrapper: Escapable` in the same source file implies that the original `struct Wrapper` must be `~Escapable` and similarly for `Copyable`, exactly as if the first line had been
```swift
struct Wrapper<T: ~Copyable & ~Escapable>: ~Copyable & ~Escapable {}
```

Now recall from SE-427 that suppressible protocols must be explicitly suppressed on type parameters in extensions.
This means that
```swift
extension Wrapper: Copyable where T: ~Escapable {}
```
is exactly the same as
```swift
extension Wrapper: Copyable where T: Copyable & ~Escapable {}
```
which implies that `Wrapper` becomes `Copyable` when `T` is `Copyable`.
Finally, remember that `~Escapable` means that `Escapable` is not required, so
this condition on `Copyable` applies regardless of whether `T` is `Escapable` or not.

Similarly,
```swift
extension Wrapper: Escapable where T: ~Copyable {}
```
is exactly the same as
```swift
extension Wrapper: Escapable where T: Escapable & ~Copyable {}
```
which means that `Wrapper` is `Escapable` whenever `T` is `Escapable` regardless of whether `T` is `Copyable` or not.

## Source compatibility

The compiler will treat any type without explicit `~Escapable` as escapable.
This matches the current behavior of the language.

Only when new types are marked as `~Escapable` does this have any impact.

Adding `~Escapable` to an existing concrete type is generally source-breaking because existing source code may rely on being able to escape values of this type.
Removing `~Escapable` from an existing concrete type is not generally source-breaking since it effectively adds a new capability, similar to adding a new protocol conformance.

## ABI compatibility

As above, existing code is unaffected by this change.
Adding or removing a `~Escapable` constraint on an existing type is an ABI-breaking change.

## Implications on adoption

Manglings and interface files will only record the lack of escapability.
This means that existing interfaces consumed by a newer compiler will treat all types as escapable.
Similarly, an old compiler reading a new interface will have no problems as long as the new interface does not contain any `~Escapable` types.

These same considerations ensure that escapable types can be shared between previously-compiled code and newly-compiled code.

Retrofitting existing generic types so they can support both escapable and nonescapable type arguments is possible with care.

## Future directions

#### `StorageView` type

This proposal is being driven in large part by the needs of the `StorageView` type that has been discussed elsewhere.
Briefly, this type would provide an efficient universal “view” of array-like data stored in contiguous memory.
Since values of this type do not own any data but only refer to data stored elsewhere, their lifetime must be limited to not exceed that of the owning storage.
We expect to publish a sample implementation and proposal for that type very soon.

#### Lifetime dependency annotations

Nonescapable types have a set of inherent restrictions on how they can be passed as arguments, stored in variables, or returned from functions.
A companion proposal builds on this by supporting more detailed annotations that link the lifetimes of different objects.
This would allow, for example, a container to vend an iterator value that held a direct unmanaged pointer to the container's contents.
The lifetime dependency would ensure that such an iterator could not outlive the container to whose contents it referred.

```swift
// Example: Nonescaping iterator
struct NEIterator {
  // `dependsOn(container)` indicates that the constructed value
  // cannot outlive the `container` argument.
  init(over container: MyContainer) -> dependsOn(container) Self {
    ... initialize an iterator suitable for `MyContainer` ...
  }
}
```

#### Expanding standard library types

We expect that many standard library types will need to be updated to support possibly-nonescapable types, including `Optional`, `Array`, `Set`, `Dictionary`, and the `Unsafe*Pointer` family of types.

Some of these types will require first exploring whether it is possible for the `Collection`, `Iterator`, `Sequence`, and related protocols to adopt these concepts directly or whether we will need to introduce new protocols to complement the existing ones.

The more basic protocols such as `Equatable`, `Comparable`, and `Hashable` should be easier to update.

#### Refining `with*` closure-taking APIs

The `~Escapable` types can be used to refine common `with*` closure-taking APIs by ensuring that the closures cannot save or hold their arguments beyond their own lifetime.
For example, this can greatly improve the safety of locking APIs that expect to unlock resources upon completion of the closure.

#### Nonescapable classes

We’ve explicitly excluded class types from being nonescapable.  In the future, we could allow class types to be declared nonescapable as a way to avoid most reference-counting operations on class objects.

#### Concurrency

Structured concurrency implies lifetime constraints similar to those outlined in this proposal.
It may be appropriate to incorporate `~Escapable` into the structured concurrency primitives.

For example, the current `TaskGroup` type is supposed to never be escaped from the local context;
making it `~Escapable` would prevent this type of abuse and possibly enable other optimizations.

#### Global nonescapable types with immortal lifetimes

This proposal currently prohibits putting values with nonescapable types into global or static variables.
We expect to eventually allow this by explicitly annotating a “static” or “immortal” lifetime.

## Alternatives considered

#### Require `Escapable` to indicate escapable types without using `~Escapable`

We could avoid using `~Escapable` to mark types that lack the `Escapable` property by requiring `Escapable` on all escapable types.
However, it is infeasible to require updating all existing types in all existing Swift code with a new explicit capability.

Apart from that, we expect almost all types to continue to be escapable in the future, so the negative marker reduces the overall burden.
It is also consistent with progressive disclosure:
Most new Swift programmers should not need to know details of how escapable types work, since that is the common behavior of most data types in most programming languages.
When developers use existing nonescapable types, specific compiler error messages should guide them to correct usage without needing to have a detailed understanding of the underlying concepts.
With our current proposal, the only developers who will need detailed understanding of these concepts are library authors who want to publish nonescapable types.

#### `Nonescapable` as a marker protocol

We considered introducing `Nonescapable` as a marker protocol indicating that the values of this type required additional compiler checks.
With that approach, you would define a conditionally-escapable type such as `Box` above in this fashion:

```swift
// Box does not normally require additional escapability checks
struct Box<T> {
  var t: T
}

// But if T requires additional checks, so does Box
extension Box: Nonescapable where T: Nonescapable { }
```

However, this would imply that any `Nonescapable` type was a
subtype of `Any` and could therefore be placed within an `Any` existential box.
An `Any` existential box is both `Copyable` and `Escapable`,
so it cannot be allowed to contain a nonescapable value.

#### Rely on `~Copyable`

As part of the `StorageView` design, we considered whether it would suffice to use `~Copyable` instead of introducing a new type concept.
Andrew Trick's analysis in [Language Support for Bufferview](https://forums.swift.org/t/roadmap-language-support-for-bufferview/66211) concluded that making `StorageView` be non-copyable would not suffice to provide the full semantics we want for that type.
Further, introducing `StorageView` as `~Copyable` would actually preclude us from later expanding it to be `~Escapable`.

The iterator example in the beginning of this document provides another motivation:
Iterators are routinely copied in order to record a particular point in a collection.
Thus we concluded that non-copyable was not the correct lifetime restriction for types of this sort, and it was worthwhile to introduce a new lifetime concept to the language.

#### Returns and initializers

This proposal does not by itself provide any way to initialize a nonescapable value, requiring the additional proposed lifetime dependency annotations to support that mechanism.
Since those annotations require that the lifetime of the returned value be bound to that of one of the arguments, this implies that our current proposal does not permit nonescapable types to have trivial initializers:

```swift
struct NE: ~Escapable {
  init() {} // 🛑 Initializer return must depend on an argument
}
```

We considered introducing an annotation that would specifically allow this and related uses:

```swift
struct NE: ~Escapable {
  @_unsafeNonescapableResult
  init() {} // OK because of annotation
}
```

We omitted this annotation from our proposal because there is more than one possible interpretation of such a marker. And we did not see a compelling reason for preferring one particular interpretation because we have yet to find a use case that actually requires this.

In particular, the use cases we’ve so far considered have all been resolvable by adding an argument specifically for the purpose of anchoring a lifetime dependency:

```swift
struct NE: ~Escapable {
  // Proposed lifetime dependency notation;
  // see separate proposal for details.
  init(from: SomeType) -> dependsOn(from) Self {}
}
```

We expect that future experience with nonescapable types will clarify whether additional lifetime modifiers of this sort are justified.

## Acknowledgements

Many people discussed this proposal and gave important feedback, including:  Kavon Farvardin, Meghana Gupta, John McCall, Slava Pestov, Joe Groff, Guillaume Lessard, and Franz Busch.
