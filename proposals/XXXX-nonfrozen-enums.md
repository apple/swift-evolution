# Non-Frozen Enumerations

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Karl Wagner](https://github.com/karwa)
* Review Manager: TBD
* Status: **Implementation In Progress**
* Previous Pitch: [pitch](https://forums.swift.org/t/extensible-enumerations-for-non-resilient-libraries/35900)
<!-- 
* Previous Proposal: *if applicable* [SE-XXXX](XXXX-filename.md)
* Previous Revision: *if applicable* [1](https://github.com/apple/swift-evolution/blob/...commit-ID.../proposals/NNNN-filename.md)
* Review: ([pitch](https://forums.swift.org/...))
-->

## Introduction

Swift's enumerations provide powerful expressive capabilities for developers. However, it is currently not possible for source libraries (such as Swift Packages) to expose enums that can evolve while maintaining source compatibility. This proposal would allow them to opt-in to that stability.

## Motivation

In some contexts, knowing all of an enum's cases is highly desirable; unhandled cases in a `switch` statement lead to a compilation error, requiring developers to examine every `switch` and proactively consider how the omitted cases apply to their operation. This helps ensure robustness in systems which evolve together (for instance, because they are part of the same module).

In other contexts, we want to leave room for an enum to grow and add new cases. This is particularly useful for enums which are exposed as part of a library's public interface, as it allows the library to evolve without breaking existing clients. When writing a `switch` statement involving an enum from a foreign library, clients should have to consider future cases so their code continues to compile as their dependencies evolve.

To illustrate, consider a library with a public data type. The type exposes a formatting function, taking as a parameter an enum describing the desired output:

```swift
public struct MyDataType {

  // ...

  public enum FormatStyle {
    case short
    case verbose
  }

  public func format(style: FormatStyle) -> String {
    switch style {
    case .short:
      // ...
    case .verbose:
      // ...
    }
  }
}
```

Because `MyDataType.FormatStyle` is a public enum, it is _possible_ that some client library has written an exhaustive `switch` over it. If the library were to add another case - say, `.ultraCompact`, `.medium`, or `.extraVerbose`, that would _technically_ be a source-breaking change, and require incrementing the library's major version (e.g. 2.x.y -> 3.x.y). Incrementing a library's major version is a highly disruptive process that requires extensive coordination with downstream packages, and is entirely disproportionate to the modest change being made here.

This is not a novel insight. Such changes could also alter an enum's ABI, which is why  [SE-0192 - Handling Future Enum Cases](https://github.com/apple/swift-evolution/blob/main/proposals/0192-non-exhaustive-enums.md) introduced the idea of _frozen_ and _non-frozen_ enums to Swift, and established that enums compiled in library-evolution mode would be non-frozen by default. When compiling in this mode, enums may be marked `@frozen` to opt-in to allowing exhaustive switching by clients.

SE-0192 did not address enums outside of library-evolution mode, instead leaving it for future discussion. However, the fragility of enums is not _only_ a concern for ABI-stable libraries -- as discussed above, it is also a major concern for libraries distributed as source packages. Without the ability to add cases to an enum _and_ preserve source compatibility, major libraries have decided to not expose enums in their interfaces, even when they would be the best, most expressive tool for the task.


## Proposed solution, Detailed design

A new attribute will be introduced to the language, `@nonfrozen`.

- Only `public enum`s may be marked `@nonfrozen`
- An enum may not be marked both `@frozen` and `@nonfrozen`
- When compiling with library-evolution mode enabled, the `@nonfrozen` attribute has no effect

An enum with the `@nonfrozen` attribute is formally non-exhaustive. That means `switch` statements in other modules (including `@inlinable` code exposed to such modules) which involve the enum must include a 'catch-all' clause to handle future cases.

When library-evolution mode is not enabled, `@nonfrozen` enums remain _effectively_ exhaustive to later stages of the compiler. This means source packages do not incur any performance penalty for marking an enum `@nonfrozen`; they should perform identically to unannotated (implicitly `@frozen`) enums. A `@nonfrozen` public enum compiled without library-evolution mode is **not** ABI stable.

A `@nonfrozen` enum behaves as the non-frozen enums described by SE-0192 do, with two minor alterations:

### 1. Switch statements MUST contain a 'catch-all'

To ease the rollout of SE-0192, it was softened so that omitting the 'catch-all' clause which handles future values only prompts a warning from the compiler, rather than an error. If an unknown value is encountered at runtime, the program reliably traps.

```swift
// Note that only warnings are produced here.
// The program still compiles successfully.

func test(_ x: FloatingPointRoundingRule) {
    switch x {
    // ^ warning: switch covers known cases, but 'FloatingPointRoundingRule' may have additional unknown values, possibly added in future versions
    // ^ note: handle unknown values using "@unknown default"
    case .up,
         .down,
         .toNearestOrEven,
         .toNearestOrAwayFromZero,
         .towardZero,
         .awayFromZero:
      print("...")
    }
}
```

However, this only applies to `switch` statements which are exhaustive. If we omit a case (simulating code that was written against some version of the standard library where `.towardZero` had not yet been introduced), we find the compiler is no longer willing to synthesise a catch-all clause for us, and instead refuses to compile the code:

```swift
func test(_ x: FloatingPointRoundingRule) {
    switch x {
    // ^ error: switch must be exhaustive
    // ^ note: add missing case: '.towardZero'
    // ^ note: handle unknown values using "@unknown default"
    case .up,
         .down,
         .toNearestOrEven,
         .toNearestOrAwayFromZero,
         .awayFromZero:
      print("...")
    }
}
```

This serves the narrow goal of providing ABI stability to `switch` statements which were exhaustive when they were compiled, but does _not_ provide source stability because the addition of an enum case means our client code no longer compiles. For source stability, we **must** include a catch-all clause, such as an `@unknown default`. Once we do so, the switch exhaustiveness error is downgraded to a warning.

```swift
func test(_ x: FloatingPointRoundingRule) {
    switch x {
    // ^ warning: switch must be exhaustive
    // note: add missing case: '.towardZero'
    case .up,
         .down,
         .toNearestOrEven,
         .toNearestOrAwayFromZero,
         .awayFromZero:
      print("...")
    @unknown default:
      print("???")
    }
}
```

The `@nonfrozen` enums being discussed in this proposal are motivated by source stability, therefore we will **insist** that all `switch` statements involving them include a catch-all clause. Failure to do so will be an error.

### 2. Modules in the same package may continue to treat the enum as frozen.

As previously mentioned, it is often desirable to treat enums as frozen and exhaustively switch over them. The line where it becomes desirable or undesirable can approximately be described as "things which evolve together"; if a usage site evolves together with the enum's declaration (e.g. because they are in the same module) we can ensure they are always in sync, but if they evolve separately the usage site needs to consider that evolution.

[SE-0386 New access modifier: package](https://github.com/apple/swift-evolution/blob/main/proposals/0386-package-access-modifier.md) introduced the concept of packages in to the language. Packages are a unit of code distribution which may encompass several modules, and the modules inside a package indeed evolve together and are version-locked with respect to each other. 

Therefore, when switching over a `@nonfrozen public enum`, if the declaration and usage modules belong to the same package, no catch-all is required and the enum's cases may be reasoned about exhaustively.

## Source compatibility

- If library-evolution mode is enabled:
  - Adding/removing the `@nonfrozen` attribute has no effect, since it is already the default and is mutually-exclusive with `@frozen`.

- If library-evolution mode is disabled:
  - Adding the `@nonfrozen` attribute to an existing `public enum` is a source-breaking change.
  - Removing the `@nonfrozen` attribute from a `public enum` when the set of cases stabilise is a source-compatible change.

## ABI compatibility

- If library-evolution mode is enabled:
  - Adding/removing the `@nonfrozen` attribute has no effect, since it is already the default and is mutually-exclusive with `@frozen`.

- If library-evolution mode is disabled:
  - Adding/removing the `@nonfrozen` attribute has no effect on the enum's ABI. Importantly, it does not confer ABI stability.

## Implications on adoption

This is an additive language feature which does not require support from the runtime or standard library.

## Future directions

### Version-locked dependencies

If a package is a collection of version-locked modules, perhaps there is room to introduce another organisational unit for a collection of version-locked packages. For instance, an App developer might split their project up in to a number of packages, for reuse in various internal projects:

- MyApp
- SharedUtilityViews
- SharedNetworkRequests
- (Possibly also 3rd-party packages which the developer updates manually)
- ...etc

The reason it would be attractive to model this is that `@nonfrozen` enums declared anywhere in this collection could be treated as exhaustive by every other package in the collection. This may allow us to make enums `@nonfrozen` by default even when library-evolution is disabled, with minimal source breakage and inconvenience.

This is a complex mix of several related features, and deserves extensive investigation. It is separable from the idea of giving source packages the ability to express non-frozen enums.


## Alternatives considered

- Do nothing.

  Package developers are avoiding exposing public enums because it is not possible to evolve them. That's not great.

- Wait for version-locked dependencies.

  The only way version-locked dependencies would satisfy the evolution requirements of source packages is if we _also_ switched the default behaviour of enums to be `@nonfrozen`.

  If that ever happens (which isn't clear), it's going to be a significant undertaking and _deinfitely, massively_ source-breaking. It's definitely interesting but it's also unreasonable to make package developers wait for such an enormous change to maybe happen one day.

## Acknowledgments

@lukasa pitched a version of this feature before.