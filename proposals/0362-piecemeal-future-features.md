# Piecemeal adoption of upcoming language improvements

* Proposal: [SE-0362](0362-piecemeal-future-features.md)
* Authors: [Doug Gregor](https://github.com/DougGregor)
* Review Manager: [Holly Borla](https://github.com/hborla)
* Status: **Implemented (Swift 5.8)**
* Implementation: [apple/swift#59055](https://github.com/apple/swift/pull/59055), [apple/swift-package-manager#5632](https://github.com/apple/swift-package-manager/pull/5632)
* Review: ([pitch](https://forums.swift.org/t/piecemeal-adoption-of-swift-6-improvements-in-swift-5-x/57184)) ([review](https://forums.swift.org/t/se-0362-piecemeal-adoption-of-future-language-improvements/58384)) ([acceptance](https://forums.swift.org/t/accepted-se-0362-piecemeal-adoption-of-future-language-improvements/59076))

## Introduction

Swift 6 is accumulating a number of improvements to the language that have enough source-compatibility impact that they could not be enabled by default in prior language modes (Swift 4.x and Swift 5.x). These improvements are already implemented in the Swift compiler behind the Swift 6 language mode, but they are inaccessible to users, and will remain so until Swift 6 becomes available as a language mode. There are several reasons why we should consider making these improvements available sooner:

* Developers would like to get the benefits from these improvements soon, rather than wait until Swift 6 is available.
* Making these changes available to developers prior to Swift 6 provides more experience, allowing us to tune them further for Swift 6 if necessary.
* The sum of all changes made in Swift 6 might make migration onerous for some modules, and adopting these language changes one-by-one while in Swift 4.x/5.x can smooth that transition path.

A few proposals have already introduced bespoke solutions to provide a migration path: [SE-0337](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0337-support-incremental-migration-to-concurrency-checking.md) adds `-warn-concurrency` to enable warnings for `Sendable`-related checks in Swift 4.x/5.x. [SE-0354](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0354-regex-literals.md) adds the flag `-enable-bare-slash-regex` to enable the bare `/.../` regular expression syntax. And although it wasn't part of the proposal, the discussion of [SE-0335](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0335-existential-any.md) included requests for a compiler flag to require `any` on all existentials. These all have the same flavor, of opting existing Swift 4.x/5.x code into improvements that will come in Swift 6.

This proposal explicitly embraces the piecemeal, intentional adoption of features that were held until Swift 6 for source-compatibility reasons. It establishes a direct path to incrementally adopt Swift 6 features, one-by-one, to gain their benefits in a Swift 4.x/5.x code base and smooth the migration path to a Swift 6 language mode. Developers can use a new compiler flag, `-enable-upcoming-feature X` to enable the specific feature named `X` for that module, and multiple features can be specified in this manner. When the developer moves to the next major language version, `X` will be implied by that language version and the compiler flag will be rejected. This way, upcoming feature flags only accumulate up to the next major Swift language version and are then cleared away, so we don't fork the language into incompatible dialects.

Swift-evolution thread: [Pitch #1](https://forums.swift.org/t/piecemeal-adoption-of-swift-6-improvements-in-swift-5-x/57184)

## Language version and tools version

There are two related kinds of "Swift version" that are distinct, but we often conflate them for convenience. However, both kinds of version have a bearing on this proposal:

- *Swift tools version*: the version number of the compiler itself. For example, the Swift 5.6 compiler was introduced in March 2022.
- *Swift language version*: the language version with which we are providing source compatibility. For example, Swift version 5 is the most current language version supported by Swift tools version 5.6.

The Swift tools support multiple Swift language versions. All recent versions (since Swift tools version 5.0) have supported multiple Swift language versions, of which there are currently only three: 4, 4.2, and 5. As the tools evolve, they try to avoid making source-incompatible changes within a Swift language version, and this is also reflected in the evolution process itself: proposals that change the meaning of existing source code, or make it invalid, are generally not accepted for existing language modes. Many proposals do *extend* the Swift language within an existing language mode. For example, `async`/`await` became available in Swift tools version 5.5, and is available in all language versions (4, 4.2, 5).

This proposal involves source-incompatible changes that are waiting for the introduction of a new Swift language version, e.g., 6. Swift tools version 6.0 will be the first tools to officially allow the use of Swift language version 6. Those tools will continue to support Swift language versions 4, 4.2, and 5. Code does not need to move to Swift language version 6 to use Swift tools version 6.0, or 6.1, and so on, and code written to Swift language version 6 will interoperate with code written to Swift language version 4, 4.2, or 5.

## Proposed solution

Introduce a compiler flag `-enable-upcoming-feature X`, where `X` is a name for the feature to enable. Each proposal will document what `X` is, so it's clear how to enable that feature. For example, SE-0274 could use `ConciseMagicFile`, so that `-enable-upcoming-feature ConciseMagicFile` will enable that change in semantics. One can of course pass multiple `-enable-upcoming-feature` flags to the compiler to enable multiple features. 

Unrecognized upcoming features will be ignored by the compiler. This allows older tools to use the same command lines as newer tools for Swift code that has started adopting new features, but has appropriate workarounds to still work with older tools. Sometimes this is possible because older compilers will still have a reasonable interpretation of the code, other times one will need a way to [detect features in source code](#feature-detection-in-source-code), the subject of a later section.

All "upcoming" features are enabled by default in some language version. The compiler will produce an error if `-enable-upcoming-feature X` is provided and the language version enables the feature `X` by default. This will make it clear to developers when their expectations about when a feature is available, and clean up projects and manifests that have evolved from from earlier language versions, adopted features piecemeal, and then moved to later language versions.

### Proposals define their own feature identifier

Amend the [Swift proposal template](https://github.com/swiftlang/swift-evolution/blob/main/proposal-templates/0000-swift-template.md) with a new, optional field that defines the feature identifier:

* **Feature identifier**: `UpperCamelCaseFeatureName`

Amend the following proposals, which are partially or wholly delayed until Swift 6, with the following feature identifiers:

* [SE-0274 "Concise magic file names"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0274-magic-file.md) (`ConciseMagicFile`) delayed the semantic change to `#file` until Swift 6. Enabling this feature changes `#file` to mean `#fileID` rather than `#filePath`.
* [SE-0286 "Forward-scan matching for trailing closures"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0286-forward-scan-trailing-closures.md) (`ForwardTrailingClosures`) delays the removal of the "backward-scan matching" rule of trailing closures until Swift 6. Enabling this feature removes the backward-scan matching rule.
* [SE-0335 "Introduce existential `any`"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0335-existential-any.md) (`ExistentialAny`) delays the requirement to use `any` for all existentials until Swift 6. Enabling this feature requires `any` for existential types.
* [SE-0337 "Incremental migration to concurrency checking"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0337-support-incremental-migration-to-concurrency-checking.md) (`StrictConcurrency`) delays some checking of the concurrency model to Swift 6 (with a flag to opt in to warnings about it in Swift 5.x). Enabling this feature is equivalent to `-warn-concurrency`, performing complete concurrency checking.
* [SE-0352 "Implicitly Opened Existentials"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0352-implicit-open-existentials.md) (`ImplicitOpenExistentials`) expands implicit opening to more cases in Swift 6, because we didn't want to change the semantics of well-formed code in Swift 5.x. Enabling this feature performs implicit opening in these additional cases.
* [SE-0354 "Regex Literals"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0354-regex-literals.md) (`BareSlashRegexLiterals`) delays the introduction of the `/.../` regex literal syntax until Swift 6. Enabling this feature is equivalent to `-enable-bare-regex-syntax`, making the `/.../` regex literal syntax available. If this proposal and SE-0354 are accepted in the same release, `-enable-bare-regex-syntax` can be completely removed in favor of this approach.

### Swift Package Manager support for upcoming features

SwiftPM targets should be able to specify the upcoming language features they require. Extend `SwiftSetting` with an API to enable an upcoming feature:

```swift
extension SwiftSetting {
  public static func enableUpcomingFeature(
    _ name: String,
    _ condition: BuildSettingCondition? = nil
  ) -> SwiftSetting
}
```

SwiftPM would then pass each of the upcoming features listed there to the compiler via the  `-enable-upcoming-feature` flag when building a module using this setting. Other targets that depend on this one do not need to pass the features when they build, because the effect of upcoming features does not cross module boundaries.

The features are provided as strings here so that SwiftPM's manifest format doesn't need to change each time a new feature is added to the compiler. Package authors can add upcoming features while still supporting older tools without creating a new, versioned manifest.

### Feature detection in source code

When adopting a new feature, it's common to want code to still compile with older tools where that feature is not available. Doing so requires a way to check whether the feature is enabled, either by `-enable-upcoming-feature` or by enabling a suitable language version.

We should extend Swift's `#if` with explicit support for a `hasFeature(X)` check, which evaluates true whenever the feature with identifier `X` is available. Code that needs to check for a specific feature can use `#if hasFeature` like this:

```swift
#if hasFeature(ImplicitOpenExistentials)
  f(aCollectionOfInts)
#else
  f(AnyCollection<Int>(aCollectionOfInts))
#endif
```

The `hasFeature(X)` check indicates the presence of features, but by itself an older compiler will still attempt to parse the `#if` branch even if the feature isn't known. That's fine for this feature (implicitly opened existentials) because it doesn't add any syntax, but other features that add syntax might require something more. `hasFeature` can be composed with the `compiler` directive introduced by [SE-0212](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0212-compiler-version-directive.md), e.g.,

```swift
#if compiler(>=5.7) && hasFeature(BareSlashRegexLiterals)
let regex = /.../
#else
let regex = try NSRegularExpression(pattern: "...")
#endif
```

There is an issue with the above, because `hasFeature` *itself* is not understood by tools that predate this proposal, so the code above will fail to compile with any Swift compiler that predates the introduction of `hasFeature`. It is possible to avoid this problem by nesting the `hasFeature` check like this (assuming that Swift 5.7 introduced `hasFeature`):

```swift
#if compiler(>=5.7)
  #if hasFeature(BareSlashRegexLiterals)
  let regex = /.../
  #else
  let regex = #/.../#
  #endif
#else
let regex = try NSRegularExpression(pattern: "...")
#endif
```

In the worst case, this does involve some code duplication for libraries that need to work on Swift versions that predate the introduction of `hasFeature`, but it is possible to handle those compilers, and over time that limitation will go away.

To prevent this issue for any upcoming extensions to the `#if` syntax, the compiler should not attempt to interpret any "call" syntax on the right-hand side of a `&&` or `||` whose left-hand side disables parsing of the `#if` body, such as `compiler(>=5.7)` or `swift(>=6.0)`, and where the right-hand term is not required to determine the result of the whole expression. For example, if we invent something like `#if hasAttribute(Y)` in the future, one can use this formulation:

```swift
#if compiler(>=5.8) && hasAttribute(Sendable)
...
#endif
```

On Swift 5.8 or newer compilers (which we assume will support `hasAttribute`), the full condition will be evaluated. On prior Swift compilers (i.e., ones that support this proposal but not something newer like `hasAttribute`), the code after the `&&` or `||` will be parsed as an expression, but will not be evaluated, so such compilers will not reject this `#if` condition.

### Embracing experimental features

It is common for language features in the compiler to be staged in behind an "experimental" flag as they are developed. This is usually done in an ad hoc manner, and the flag is removed before the feature finally ships. However, we should embrace the experimental feature model further: when a feature is under development, provide it with a feature identifier that allows it to be enabled with a new flag, `-enable-experimental-feature X`, or its SwiftPM counterpart `enableExperimentalFeature`.

Experimental features are still to be considered unstable, and should not be available in released compilers. However, by unifying the manner in which experimental and upcoming features are introduced, we can rely on the same staging mechanisms: a way to enable the feature and to check for its presence in source code, making it easier to experiment with these features. If a feature then "graduates" to a complete, supported language feature, `hasFeature` can return true for it and, if part of it was delayed until the next major language version, `-enable-upcoming-feature` will work with it, too. 

## Source compatibility

For the language itself, `hasFeature` is the only addition, and it occurs in a constrained syntactic space (`#if`) where there are no user-definable functions. Therefore, there is no source-compatibility issue in the traditional sense, where a newer compiler rejects existing, well-formed code. 

For SwiftPM, the addition of the `enableUpcomingFeature` and `enableExperimentalFeature` functions to `SwiftSetting` represents a one-time break in the manifest file format. Packages that wish to adopt these functions and support tools versions that predate the introduction of `enableUpcomingFeature` and `enableExperimentalFeature` can use versioned manifest, e.g., `Package@swift-5.6.swift`, to adopt the feature for newer tools versions. Once `enableUpcomingFeature` and `enableExperimentalFeature` have been added, adopting additional features this way won't require another copy of the manifest file.

## Alternatives considered

### `$X` instead of `hasFeature(X)`

The original pitch for this proposal used special identifiers `$X` for feature detection instead of `hasFeature(X)`. `$X` has been used in the compiler implementation to help stage in Swift's concurrency features, especially when producing Swift interface files that might need to be understood by older tools versions. The compiler still defines `$AsyncAwait`, for example, which can be used with  `#if` to check for async/await support:

```swift
#if compiler(>=5.3) && $AsyncAwait
func f() async -> String
#endif
```

The primary advantage to the `$` syntax is that all Swift compilers already treat `$` as an acceptable leading character for an identifier. The compiler can define names with a leading `$`, but developers aren't technically supposed to, so it's effectively a reserved space for "magic" names. This means that, unlike the `hasFeature` formulation of the above, older compilers can process the code above without producing an error.

However, this proposal introduces `hasFeature` because it's clearer in the code, and makes the forward-looking changes to the way `#if` conditions are processed to make it easier for additional `hasFeature`-like features to be introduced in the future without having this problem with older compilers.

### Enabling optional features

This proposal narrowly introduces `-enable-upcoming-feature` to only describe accepted features that will be enabled with a newer language version, but that were held back (partially or in full) due to source compatibility concerns. It is not meant to be used to enable "optional" features, which would create permanent dialects, and is designed to be somewhat self-healing: as folks move to newer language modes (e.g., Swift 6), the upcoming feature flags are eliminated with the new baseline.

### Enabling all upcoming features

The set of upcoming features will expand over time, as Swift introduces new features with source-compatibility impact that are staged in via a new major language version. For developers who want to be on the leading edge, it would be more convenient to have a single flag that enables all upcoming features, rather than having to specify each upcoming feature as they get added. However, the introduction of such a flag would create a shifting dialect of Swift: features are only "upcoming" features if they have source-compatibility impact, so code that adopted this flag could break with every new Swift release. That would directly cut against our source-compatibility goals for Swift, so we do not propose such a flag. Instead, we should find a central place to document all upcoming features on swift.org, updated with each release, so that developers know where to go to learn about the new upcoming features they want to enable.

## Revision History

* Changes from first reviewed version:
  * Changed the SwiftPM manifest API to be based on `SwiftSettings` rather than the target.
  * Use the term "upcoming feature" rather than "future feature" to reduce confusion.
  * Don't parse the right-hand side of a `&&` or `||` that doesn't affect the result.
  * Add some discussion of language and tools versions.

## Acknowledgments

Becca Royal-Gordon designed the original `#if compiler(>=5.5) && $AsyncAwait` approach to adopting features without breaking compatibility with older tools, and helped shape this design. Ben Rimmington provided the design for the SwiftPM API, replacing the less-flexible design from the original reviewed proposal.
