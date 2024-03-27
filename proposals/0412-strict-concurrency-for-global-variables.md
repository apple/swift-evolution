# Strict concurrency for global variables

* Proposal: [SE-0412](0412-strict-concurrency-for-global-variables.md)
* Authors: [John McCall](https://github.com/rjmccall), [Sophia Poirier](https://github.com/sophiapoirier)
* Review Manager: [Holly Borla](https://github.com/hborla)
* Status: **Implemented (Swift 5.10)**
* Upcoming Feature Flag: `GlobalConcurrency` (Enabled in Swift 6 language mode)
* Implementation: On `main` gated behind `-enable-experimental-feature GlobalConcurrency`
* Previous Proposals: [SE-0302](0302-concurrent-value-and-concurrent-closures.md), [SE-0306](0306-actors.md), [SE-0316](0316-global-actors.md), [SE-0337](0337-support-incremental-migration-to-concurrency-checking.md), [SE-0343](0343-top-level-concurrency.md)
* Review: ([pitch](https://forums.swift.org/t/pitch-strict-concurrency-for-global-variables/66908)), ([review](https://forums.swift.org/t/se-0412-strict-concurrency-for-global-variables/68352)), ([acceptance](https://forums.swift.org/t/accepted-se-0412-strict-concurrency-for-global-variables/69004))

## Introduction

This proposal defines options for the usage of global variables free of data races. Within this proposal, global variables encompass any storage of static duration: `let`s and stored `var`s that are either declared at global scope or as static member variables.

## Motivation

Global state poses a challenge within concurrency because it is memory that can be accessed from any program context. Global variables are of particular concern in data isolation checking because they defy other attempts to enforce isolation. Variables that are local and un-captured can only be accessed from that local context, which implicitly isolates them. Stored properties of value types are already isolated by the exclusivity rules. Stored properties of reference types can be isolated by isolating their containing object with sendability enforcement or using actor restrictions. But global variables can be accessed from anywhere, so these tools do not work.

```swift
var value = 1

func f() {
  value = 2 // warning: reference to var 'value' is not concurrency-safe because it involves shared mutable state
}
```

## Proposed solution

Under strict concurrency checking, require every global variable to either be isolated to a global actor or be both:

1. immutable
2. of `Sendable` type

Global variables that are immutable and `Sendable` can be safely accessed from any context, and otherwise, isolation is required.

Top-level global variables are already implicitly isolated to `@MainActor` and therefore automatically meet these proposed requirements.

## Detailed design

These requirements can be enforced in the type checker at declaration time.

Although global variables are lazily initialized, the initialization is already guaranteed to be thread-safe and therefore requires no further specification under strict concurrency checking.

There may be need in some circumstances to opt out of static checking to enable the developer to rely upon their own data isolation management, such as with an associated global lock serializing data access. The attribute `nonisolated(unsafe)` can be used to annotate the global variable (or any form of storage). Though this will disable static checking of data isolation for the global variable, note that without correct implementation of a synchronization mechanism to achieve data isolation, dynamic run-time analysis from exclusivity enforcement or tools such as Thread Sanitizer could still identify failures.

```swift
nonisolated(unsafe) var global: String
```

The same annotation on a local variable can be used to suppress a static diagnostic being generated when the local variable is referenced asynchronously:

```swift
func f() async {
  nonisolated(unsafe) var value = 1
  let task = Task {
    value = 2
    return value
  }
  print(await task.value)
}
```

Because `nonisolated` is a contextual keyword, there is ambiguity when using `nonisolated(unsafe)` on a separate line immediately preceding a top-level variable declaration in script mode as it could also be the invocation of a function named `nonisolated` with argument `unsafe`. This ambiguity can be resolved by favoring the interpretation of `nonisolated` as a keyword if it has a single unlabeled argument of `unsafe` and precedes a variable declaration.

Importing a module via `@preconcurrency import` suppresses any potential errors resulting from data isolation checking of imported global variables that lack explicit concurrency annotations. Any use of a `@preconcurrency import`ed concurrency-unsafe global variable will produce a warning at the use site.

Note that imports from other languages are implicitly `@preconcurrency`. There remain tools for enforcing safety for imported global variables from other languages, such as isolating to a global actor using for example `__attribute__((swift_attr("@MainActor")))` in C or Obj-C, or wrapping access within a safer API that declares the correct isolation or locks appropriately.

## Source compatibility

Due to the addition of restrictions, this could require changes to some type declaration when strict concurrency checking is in use. Such source changes however would still be backwards compatible to any version of Swift with concurrency features.

Resolving the ambiguity of `nonisolated(unsafe)` in a top-level variable declaration would break existing top-level script code that invokes a function named `nonisolated` with a single unlabeled argument `unsafe` when immediately preceding a variable declaration by eliminating that function invocation in favor of its interpretation as an isolation specification.

## ABI compatibility

This proposal does not add or affect ABI in and of itself, however type declaration changes that it may instigate upon an adopting project could impact that project's ABI.

## Implications on adoption

Some global variable types may need to be modified in a project adopting strict concurrency checking.

## Alternatives considered

For isolation, rather than requiring a global actor, we could implicitly lock around accesses of the variable. While providing memory safety, this can be problematic for thread safety, because developers can easily write non-atomic use patterns:

```swift
// value of global may concurrently change between
// the read for the multiplication expression
// and the write for the assignment
global = global * 2
```

Though we could consider implicit locking if we needed to do something source-compatible in old language modes, generally our approach has just been to say that old language modes are concurrency-unsafe. It also would not work for non-`Sendable` types unless we force the value to remain isolated while accessing it. We potentially could accomplish that with the proposed [Safely sending non-Sendable values across isolation domains](https://forums.swift.org/t/pitch-safely-sending-non-sendable-values-across-isolation-domains/66566) feature, but that is probably too advanced a feature to push as a solution for such a basic problem.

We could default all global variables that require isolation to `@MainActor`. It is arguably better to make developers think about the choice (e.g. perhaps it should just be a `let` constant).

Access control is theoretically useful here: for example, we could know that a global variable is concurrency-safe because it is private to a file and all of the accesses in that file are from within a single global actor context, or because it is never mutated. That is a more global analysis than we usually want to do in the compiler, though; we would have to check everything in the context, and then it might be hard for the developer to understand why it works.

## Future directions

We do not necessarily need to require isolation to a global actor to be _explicit_; there is room for inferring the right global actor. A global mutable variable of global-actor-constrained type could be inferred to be constrained to that global actor (though unnecessary if the variable is immutable, since global-actor-constrained class types are `Sendable`).

## Revision history

Post-review changes:
* removed implicit `nonisolated(unsafe)` import of C global variables in favor of `@preconcurrency import` as the mechanism to suppress static isolation checking of global variables
* clarifed `nonisolated(unsafe)` for local variables
