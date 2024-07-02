# Incremental migration to concurrency checking

* Proposal: [SE-0337](0337-support-incremental-migration-to-concurrency-checking.md)
* Authors: [Doug Gregor](https://github.com/DougGregor), [Becca Royal-Gordon](https://github.com/beccadax)
* Review Manager: [Ben Cohen](https://github.com/AirspeedSwift)
* Status: **Implemented (Swift 5.6)**
* Upcoming Feature Flag: `StrictConcurrency` (Implemented in Swift 6.0) (Enabled in Swift 6 language mode)
* Implementation: [Pull request](https://github.com/apple/swift/pull/40680), [Linux toolchain](https://ci.swift.org/job/swift-PR-toolchain-Linux/761//artifact/branch-main/swift-PR-40680-761-ubuntu16.04.tar.gz), [macOS toolchain](https://ci.swift.org/job/swift-PR-toolchain-osx/1256//artifact/branch-main/swift-PR-40680-1256-osx.tar.gz)

## Introduction

Swift 5.5 introduced mechanisms to eliminate data races from the language, including the `Sendable` protocol ([SE-0302](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md)) to indicate which types have values that can safely be used across task and actor boundaries, and global actors ([SE-0316](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0316-global-actors.md)) to help ensure proper synchronization with (e.g.) the main actor. However, Swift 5.5 does not fully enforce `Sendable` nor all uses of the main actor because interacting with modules which have not been updated for Swift Concurrency was found to be too onerous. We propose adding features to help developers migrate their code to support concurrency and interoperate with other modules that have not yet adopted it, providing a smooth path for the Swift ecosystem to eliminate data races.

Swift-evolution threads: [[Pitch] Staging in `Sendable` checking](https://forums.swift.org/t/pitch-staging-in-sendable-checking/51341), [Pitch #2](https://forums.swift.org/t/pitch-2-staging-in-sendable-checking/52413), [Pitch #3](https://forums.swift.org/t/pitch-3-incremental-migration-to-concurrency-checking/53610)

## Motivation

Swift Concurrency seeks to provide a mechanism for isolating state in concurrent programs to eliminate data races. The primary mechanism is `Sendable` checking. APIs which send data across task or actor boundaries require their inputs to conform to the `Sendable` protocol; types which are safe to send declare conformance, and the compiler checks that these types only contain `Sendable` types, unless the type's author explicitly indicates that the type is implemented so that it uses any un-`Sendable` contents safely.

This would all be well and good if we were writing Swift 1, a brand-new language which did not need to interoperate with any existing code. Instead, we are writing Swift 6, a new version of an existing language with millions of lines of existing libraries and deep interoperation with C and Objective-C. None of this code specifies any of its concurrency behavior in a way that `Sendable` checking can understand, but until it can be updated, we still want to use it from Swift.

There are several areas where we wish to address adoption difficulties.

### Adding retroactive concurrency annotations to libraries

Many existing APIs should be updated to formally specify concurrency behavior that they have always followed, but have not been able to describe to the compiler until now. For instance, it has always been the case that most UIKit methods and properties should only be used on the main thread, but before the `@MainActor` attribute, this behavior could only be documented and asserted in the implementation, not described to the compiler.

Thus, many modules should undertake a comprehensive audit of their APIs to decide where to add concurrency annotations. But if they try to do so with the tools they currently have, this will surely cause source breaks. For instance, if a method is marked `@MainActor`, projects which have not yet adopted Swift Concurrency will be unable to call it even if they are using it correctly, because the project does not yet have the annotations to *prove to the compiler* that the call will run in the main actor.

In some cases, these changes can even cause ABI breaks. For instance, `@Sendable` attributes on function types and `Sendable` constraints on generic parameters are incorporated into mangled function names, even though `Sendable` conformances otherwise have no impact on the calling convention (there isn't an extra witness table parameter, for instance). A mechanism is needed to enforce these constraints during typechecking, but generate code as though they do not exist.

Here, we need:

* A formal specification of a "compatibility mode" for pre-concurrency code which imports post-concurrency modules

* A way to mark declarations as needing special treatment in this "compatibility mode" because their signatures were changed for concurrency

### Adopting `Sendable` checking before the modules you use have been updated

The process of auditing libraries to add concurrency annotations will take a long time. We don't think it's realistic for each module to wait until all of its libraries have been updated before they can start adopting `Sendable` checking.

This means modules need a way to work around incomplete annotations in their imports--either by tweaking the specifications of imported declarations, or by telling the compiler to ignore errors. Whatever mechanism we use, we don't want it to be too verbose, though; for example, marking every single variable of a non-`Sendable` type which we want to treat as `Sendable` would be pretty painful.

We must also pay special attention to what happens when the library finally *does* add its concurrency annotations, and they reveal that a client has made a mistaken assumption about its concurrency behavior. For instance, suppose you import type `Point` from module `Geometry`. You enable `Sendable` checking before `Geometry`'s maintainers have added concurrency annotations, so it diagnoses a call that sends a `Point` to a different actor. Based on the publicly-known information about `Point`, you decide that this type is probably `Sendable`, so you silence this diagnostic. However, `Geometry`'s maintainers later examine the implementation of `Point` and determine that it is *not* safe to send, so they mark it as non-`Sendable`. What should happen when you get the updated version of `Geometry` and rebuild your project?

Ideally, Swift should not continue to suppress the diagnostic about this bug. After all, the `Geometry` team has now marked the type as non-`Sendable`, and that is more definitive than your guess that it would be `Sendable`. On the other hand, it probably shouldn't *prevent* you from rebuilding your project either, because this bug is not a regression. The updated `Geometry` module did not add a bug to your code; your code was already buggy. It merely *revealed* that your code was buggy. That's an improvement on the status quo--a diagnosed bug is better than a hidden one.

But if Swift reacts to this bug's discovery by preventing you from building a module that built fine yesterday, you might have to put off updating the `Geometry` module or even pressure `Geometry`'s maintainers to delay their update until you can fix it, slowing forward progress. So when your module assumes something about an imported declaration that is later proven to be incorrect, Swift should emit a *warning*, not an error, about the bug, so that you know about the bug but do not have to correct it just to make your project build again.

Here, we need:

* A mechanism to silence diagnostics about missing concurrency annotations related to a particular declaration or module

* Rules which cause those diagnostics to return once concurrency annotations have been added, but only as warnings, not errors

## Proposed solution

We propose a suite of features to aid in the adoption of concurrency annotations, especially `Sendable` checking. These features are designed to enable the following workflow for adopting concurrency checking:

1. Enable concurrency checking, by adopting concurrency features (such as `async/await` or actors), enabling Swift 6 mode, or adding the `-warn-concurrency` flag. This causes new errors or warnings to appear when concurrency constraints are violated.

2. Start solving those problems. If they relate to types from another module, a fix-it will suggest using a special kind of import, `@preconcurrency import`, which silences these warnings.

3. Once you've solved these problems, integrate your changes into the larger build.

4. At some future point, a module you import may be updated to add `Sendable` conformances and other concurrency annotations. If it is, and your code violates the new constraints, you will see warnings telling you about these mistakes; these are latent concurrency bugs in your code. Correct them.

5. Once you've fixed those bugs, or if there aren't any, you will see a warning telling you that the `@preconcurrency import` is unnecessary. Remove the `@preconcurrency` attribute. Any `Sendable`-checking failures involving that module from that point forward will not suggest using `@preconcurrency import` and, in Swift 6 mode, will be errors that prevent your project from building.

Achieving this will require several features working in tandem:

* In Swift 6 mode, all code will be checked completely for missing `Sendable` conformances and other concurrency violations, with mistakes generally diagnosed as errors. The `-warn-concurrency` flag will diagnose these violations as warnings in older language versions.

* When applied to a nominal declaration, the `@preconcurrency` attribute specifies that a declaration was modified to update it for concurrency checking, so the compiler should allow some uses in Swift 5 mode that violate concurrency checking, and generate code that interoperates with pre-concurrency binaries.

* When applied to an `import` statement, the `@preconcurrency` attribute tells the compiler that it should only diagnose `Sendable`-requiring uses of non-`Sendable` types from that module if the type explicitly declares a `Sendable` conformance that is unavailable or has constraints that are not satisifed; even then, this will only be a warning, not an error.


## Detailed design

### Recovery behavior

When this proposal speaks of an error being emitted as a warning or suppressed, it means that the compiler will recover by behaving as though (in order of preference):

* A nominal type that does not conform to `Sendable` does.

* A function type with an `@Sendable` or global actor attribute doesn't have it.

### Concurrency checking modes

Every scope in Swift can be described as having one of two "concurrency checking modes":

* **Strict concurrency checking**: Missing `Sendable` conformances or global-actor annotations are diagnosed. In Swift 6, these will generally be errors; in Swift 5 mode and with nominal declarations visible via  `@preconcurrency import` (defined below), these diagnostics will be warnings.

* **Minimal concurrency checking**: Missing `Sendable` conformances or global-actor annotations are diagnosed as warnings; on nominal declarations, `@preconcurrency` (defined below) has special effects in this mode which suppress many diagnostics.

The top level scope's concurrency checking mode is:

* **Strict** when the module is being compiled in Swift 6 mode or later, when the `-warn-concurrency` flag is used with an earlier language mode, or when the file being parsed is a module interface.

* **Minimal** otherwise.

A child scope's concurrency checking mode is:

* **Strict** if the parent's concurrency checking mode is **Minimal** and any of the following conditions is true of the child scope:

  * It is a closure with an explicit global actor attribute.

  * It is a closure or autoclosure whose type is `async` or `@Sendable`. (Note that the fact that the parent scope is in Minimal mode may affect whether the closure's type is inferred to be `@Sendable`.)

  * It is a declaration with an explicit `nonisolated` or global actor attribute.

  * It is a function, method, initializer, accessor, variable, or subscript which is marked `async` or `@Sendable`.

  * It is an `actor` declaration.

* Otherwise, the same as the parent scope's.

> Implementation note: The logic for determining whether a child scope is in Minimal or Strict mode is currently implemented in `swift::contextRequiresStrictConcurrencyChecking()`.

Imported C declarations belong to a scope with Minimal concurrency checking.

### `@preconcurrency` attribute on nominal declarations

To describe their concurrency behavior, maintainers must change some existing declarations in ways which, by themselves, could be source-breaking in pre-concurrency code or ABI-breaking when interoperating with previously-compiled binaries. In particular, they may need to:

* Add `@Sendable` or global actor attributes to function types
* Add `Sendable` constraints to generic signatures
* Add global actor attributes to declarations

When applied to a nominal declaration, the `@preconcurrency` attribute indicates that a declaration existed before the module it belongs to fully adopted concurrency, so the compiler should take steps to avoid these source and ABI breaks. It can be applied to any `enum`, enum `case`, `struct`, `class`, `actor`, `protocol`, `var`, `let`, `subscript`, `init` or `func` declaration.

When a nominal declaration uses `@preconcurrency`:

* Its name is mangled as though it does not use any of the listed features.

* At use sites whose enclosing scope uses Minimal concurrency checking, the compiler will suppress any diagnostics about mismatches in these traits.

* The ABI checker will remove any use of these features when it produces its digests.

Objective-C declarations are always imported as though they were annotated with `@preconcurrency`.

For example, consider a function that can only be called on the main actor, then runs the provided closure on a different task:

```swift
@MainActor func doSomethingThenFollowUp(_ body: @Sendable () -> Void) {
  // do something
  Task.detached {
    // do something else
    body()
  }
}
```

This function could have existed before concurrency, without the `@MainActor` and `@Sendable` annotations. After adding these concurrency annotations, code that worked previously would start producing errors:

```swift
class MyButton {
  var clickedCount = 0
  
  func onClicked() { // always called on the main thread by the system
    doSomethingThenFollowUp { // ERROR: cannot call @MainActor function outside the main actor
      clickedCount += 1 // ERROR: captured 'self' with non-Sendable type `MyButton` in @Sendable closure
    }
  }
}
```

However, if we add `@preconcurrency` to the declaration of `doSomethingThenFollowUp`, its type is adjusted to remove both the `@MainActor` and the `@Sendable`, eliminating the errors and providing the same type inference from before concurrency was adopted by `doSomethingThenFollowUp`. The difference is visible in the type of `doSomethingThenFollowUp` in a minimal vs. a strict context:

```swift
func minimal() {
  let fn = doSomethingThenFollowUp // type is (( )-> Void) -> Void
}

func strict() async {
  let fn = doSomethingThenFollowUp // type is @MainActor (@Sendable ( )-> Void) -> Void
}
```

### `Sendable` conformance status

A type can be described as having one of the following three `Sendable` conformance statuses:

* **Explicitly `Sendable`** if it actually conforms to `Sendable`, whether via explicit declaration or because the `Sendable` conformance was inferred based on the rules specified in [SE-0302](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md).

* **Explicitly non-`Sendable`** if a `Sendable` conformance has been declared for the type, but it is not available or has constraints the type does not satisfy, *or* if the type was declared in a scope that uses Strict concurrency checking.[2]

* **Implicitly non-`Sendable`** if no `Sendable` conformance has been declared on this type at all.

> [2] This means that, if a module is compiled with Swift 6 mode or the `-warn-concurrency` flag, all of its types are either explicitly `Sendable` or explicitly non-`Sendable`.

A type can be made explicitly non-`Sendable` by creating an unavailable conformance to `Sendable`, e.g.,

```swift
@available(*, unavailable)
extension Point: Sendable { }
```

Such a conformance suppresses the implicit conformance of a type to `Sendable`.

### `@preconcurrency` on `Sendable` protocols

Some number of existing protocols describe types that should all be `Sendable`. When such protocols are updated for concurrency, they will likely inherit from the `Sendable` protocol. However, doing so will break existing types that conform to the protocol and are now assumed to be `Sendable`. This problem was [described in SE-0302](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md#thrown-errors) because it affects the `Error` and `CodingKey` protocols from the standard library:

```swift
protocol Error: /* newly added */ Sendable { ... }

class MutableStorage {
  var counter: Int
}
struct ProblematicError: Error {
  var storage: MutableStorage // error: Sendable struct ProblematicError has non-Sendable stored property of type MutableStorage
}
```

To address this, SE-0302 says the following about the additional of `Sendable` to the `Error` protocol:

> To ease the transition, errors about types that get their `Sendable` conformances through `Error` will be downgraded to warnings in Swift < 6.

We propose to replace this bespoke rule for `Error` and `CodingKey` to apply to every protocol that is annotated with `@preconcurrency` and inherits from `Sendable`. These two standard-library protocols will use `@preconcurrency`:

```swift
@preconcurrency protocol Error: Sendable { ... }
@preconcurrency protocol CodingKey: Sendable { ... }
```

### `@preconcurrency` attribute on `import` declarations

The `@preconcurrency` attribute can be applied to an `import` declaration to indicate that the compiler should reduce the strength of some concurrency-checking violations caused by types imported from that module. You can use it to import a module which has not yet been updated with concurrency annotations; if you do, the compiler will tell you when all of the types you need to be `Sendable` have been annotated. It also serves as a temporary escape hatch to keep your project compiling until any mistaken assumptions you had about that module are fixed.

When an import is marked `@preconcurrency`, the following rules are in effect:

* If an implicitly non-`Sendable` type is used where a `Sendable` type is needed:

  * If the type is visible through a `@preconcurrency import`, the diagnostic is suppressed (prior to Swift 6) or emitted as a warning (in Swift 6 and later).
  
  * Otherwise, the diagnostic is emitted normally, but a separate diagnostic is provided recommending that `@preconcurrency import` be used to work around the issue.

* If an explicitly non-`Sendable` type is used where a `Sendable` type is needed:

  * If the type is visible through an `@preconcurrency import`, a warning is emitted instead of an error, even in Swift 6.

  * Otherwise, the diagnostic is emitted normally.

* If the `@preconcurrency` attribute is unused[3], a warning will be emitted recommending that it be removed.

> [3] We don't define "unused" more specifically because we aren't sure if we can refine it enough to, for instance, recommend removing one of a pair of `@preconcurrency` imports which both import an affected type.

## Source compatibility

This proposal is largely motivated by source compatibility concerns. Correct use of `@preconcurrency` should prevent source breaks in code built with Minimal concurrency checking, and `@preconcurrency import` temporarily weakens concurrency-checking rules to preserve source compatibility if a project adopts Full or Strict concurrency checking before its dependencies have finished adding concurrency annotations.

## Effect on ABI stability

By itself, `@preconcurrency` does not change the ABI of a declaration. If it is applied to declarations which have already adopted one of the features it affects, that will create an ABI break. However, if those features are added at the same time or after `@preconcurrency` is added, adding those features will *not* break ABI.

`@preconcurrency`'s tactic of disabling `Sendable` conformance errors is compatible with the current ABI because `Sendable` was designed to not emit additional metadata, have a witness table that needs to be passed, or otherwise impact the calling convention or most other parts of the ABI. It only affects the name mangling.

This proposal should not otherwise affect ABI.

## Effect on API resilience

`@preconcurrency` on nominal declarations will need to be printed into module interfaces. It is effectively a feature to allow the evolution of APIs in ways that would otherwise break resilience.

`@preconcurrency` on `import` statements will not need to be printed into module interfaces; since module interfaces use the Strict concurrency checking mode, where concurrency diagnostics are warnings, they have enough "wiggle room" to tolerate the missing conformances. (As usual, compiling a module interface silences warnings by default.)

## Alternatives considered

### A "concurrency epoch"

If the evolution of a given module is tied to a version that can be expressed in `@available`, it is likely that there will be some specific version where it retroactively adds concurrency annotations to its public APIs, and that thereafter any new APIs will be "born" with correct concurrency annotations. We could take advantage of this by allowing the module to specify a particular version when it started ensuring that new APIs were annotated and automatically applying `@preconcurrency` to APIs available before this cutoff.

This would save maintainers from having to manually add `@preconcurrency` to many of the APIs they are retroactively updating. However, it would have a number of limitations:

1. It would only be useful for modules used exclusively on Darwin. Non-Darwin or cross-platform modules would still need to add `@preconcurrency` manually.

2. It would only be useful for modules which are version-locked with either Swift itself or a Darwin OS. Modules in the package ecosystem, for instance, would have little use for it.

3. In practice, version numbers may be insufficiently granular for this task. For instance, if a new API is added at the beginning of a development cycle and it is updated for concurrency later in that cycle, you might mistakenly assume that it will automatically get `@preconcurrency` when in fact you will need to add it by hand.

Since these shortcomings significantly reduce its applicability, and you only need to add `@preconcurrency` to declarations you are explicitly editing (so you are already very close to the place where you need to add it), we think a concurrency epoch is not worth the trouble.

### Objective-C and `@preconcurrency`

Because all Objective-C declarations are implicitly `@preconcurrency`, there is no way to force concurrency APIs to be checked in Minimal-mode code, even if they are new enough that there should be no violating uses. We think this limitation is acceptable to simplify the process of auditing large, existing Objective-C libraries.
