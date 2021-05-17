# Introducing Configuration Tests to detect Development and Testing Conditions

* Proposal: SE-TBD
* Author(s): [Erica Sadun](http://github.com/erica), [John Holdsworth](https://github.com/johnno1962)
* Review manager: TBD
* Status: **Preliminary Implementation** ([preliminary diffs](https://github.com/apple/swift/compare/master...erica:optcheck))

<!---
* Implementation: [apple/swift#NNNNN](https://github.com/apple/swift/pull/NNNNN)
* Decision Notes: [Rationale](https://lists.swift.org/pipermail/swift-evolution/), [Additional Commentary](https://lists.swift.org/pipermail/swift-evolution/)
* Bugs: [SR-NNNN](https://bugs.swift.org/browse/SR-NNNN), [SR-MMMM](https://bugs.swift.org/browse/SR-MMMM)
* Previous Revision: [1](https://github.com/apple/swift-evolution/blob/...commit-ID.../proposals/NNNN-filename.md)
* Previous Proposal: [SE-XXXX](XXXX-filename.md)
* -->

## Introduction

This proposal introduces configuration tests to detect development and testing. This supports code exclusion at build time and branched selection statements at run-time based on whether a project has been built for internal development or external release.

This proposal was discussed on-list in the [Introducing a Debug Build Configuration Test](https://forums.swift.org/t/draft-introducing-a-debug-build-configuration-test/1794) thread and in-forum in the [Support debug-only code](https://forums.swift.org/t/support-debug-only-code/11037/7) thread.

## Motivation

Conditional compilation differentiate requirements for various build conditions. Typical language conditions include the target platforms the code will run on (their architectures, operating system, vendor, and other qualities like endianness and bitwidth), API availability, language versions, and more. Using "development" conditions detects in-house builds and beta builds not meant for general release.

Implementation details can vary based on whether a code base is prepared for pre-release use or deployment. For example, you may need to use a sandbox server, incorporate verbose logging, or run expensive tests when building a "debug" version of a project. These features may be entirely excluded from a release build for reasons of security or code size. Similar concerns can support unit testing and development staging. 

For example, you may need to select an appropriate URL resources for sites that may not yet be live. Some code may not be App Store safe (like [SimulatorStatusMagic](https://github.com/shinydevelopment/SimulatorStatusMagic)) and should be entirely removed for release builds. This ensures that App Store static analysis will not flag vestigial development elements that could trigger an automatic rejection. Or you may need to change parameters during in-house [emissions testing](https://en.wikipedia.org/wiki/Volkswagen_emissions_scandal) that can be bypassed on release.

Optimizations and assertions can be orthogonal to each other. For example, you many need to test development code using high load conditions, which requires both optimizations and asserts. Some builds may need optimization for usable performance but require diagnostics, logging, alternative assets, or App Store unsafe code. Using `precondition` to retain assertions and sanity checks in test code does not sufficiently cover all in-house and beta-build features for "development" conditions.

Using buildtime configuration conditions are an industry-standard approach to handling these needs. They distinguish development behavior from their release counterparts.

The exact nature of conditional build testing varies by language. Swift currently allows you to set custom flags at the command line. 

* `-D <#value#>` (for example, `-D debug`) lets you conditionally compile using the `#if <#value#>` test, such as `#if debug`. There is no way to access these flags for runtime tests and no public way to differentiate "debug" from "release" at runtime. 
* You can manually set asssert firing conditions by supplying the `-assert-config <#value#>` flag. Supply Debug, Release, Unchecked, or DisableReplacement.

### Defining "Development"

This proposal treats the notion of "development" to mean:

* A compilation or execution environment not meant for general release.
* Code meant for in-house or beta distribution.

It offers two axes of development conditions:

* It checked for optimized compilation. When code is compiled with optimization flags (-O, -Ounchecked, _specifically excluding -Onone/-Oplayground from these tests_), then the code is "optimized"
* It checks whether asserts can fire. When an assert is included in the build product, then the code supports assert firing.

These definitions are distinct from any tooling including the "Debug" and "Release" schemes used in IDEs like Xcode. Schemes do not play into the proposed Swift checks outside of their specific inclusion of optimization flags.

## Detail Design

This proposal introduces both run-time and compile-time checks. Here is how each check is designed.

#### Run-Time Checks

The swift standard library includes three hidden helper functions: `_isDebugAssertConfiguration`, `_isReleaseAssertConfiguration`, and `_isFastAssertConfiguration`. 

#### Compile-Time Checks

Compile-time checks use the standard `#if <#condition#>` form and include code on each successful branch. To test for "optimized" conditions, the conditional compilation configuration test must wait until any optimization flag is found and trigger for any flag that is not -Onone or -Oplayground (treated as the equivalent to -Onone). Including a command-line `-O` flag such as `-O` or `-Ounchecked` excludes a "debug" compilation. The "assertion" condition waits slightly longer and then uses the value stored in `Opts.RemoveRuntimeAsserts`.

These build configuration tests allow you to use conditional build configurations to incorporate or exclude code specific to development builds as in the following examples:

```swift
#if !configuration(optimized)
    // code for un-optimized builds only
#endif

#if configuration(assertsWillFire)
    // code where assertions can fire
#endif
```

These names and approaches have been selected because:

* They reflect the underlying implementation mechanism.
* They avoid any suggestion that the configuration test is tied to a selected Xcode scheme using the names "Debug" and "Release".

Implementing "optimization" and "assert" tests rather than "debug" conditions arises from practical considerations. The differentiation from scheme names is an added bonus.

#### Run-Time vs Compile-Time checks

This proposal distinguishes between run-time checks (code is not removed, it's simply a visible boolean function) and compile-time checks by giving each approach a distinct name and flavor.

#### Moving Beyond Custom Conditional Compilation Flag

Custom conditional compilation flags are specified using `-D <#value#>`. This proposal introduces a built-in way to test for development builds that does not rely on scheme configurations in Xcode or other design environments. Doing so:

* Decouples code from schemes, allowing code re-use that may rely on development conditions that may not transfer when looking only at a flat code file.
* Provides a helpful affordance for those who use development checks regularly in their code to provide App Store safety or safety for other release conditions.
* Promotes checks in standard Swift code beyond condition build configuration.
* Unifies "development" flags to a single opinionated Swift declaration with two variations, avoiding the many possible ways of naming the `-D` flag: `DEBUG`, `debug`, `Debug`, `Optimized`, `OPTIMIZED`, `OPTIMIZED_BUILD`, `RELEASE`, `BETA`, `INHOUSE`, etc.
* Avoids the common `DEBUG` name, which feels at odds with accepted Swift style. Swift avoids uppercase and underscored naming conventions. This tooling uses a function name and build configuration test that conform to standard Swift naming.

#### Background

Joe Groff writes about early considerations:

> "We specifically avoided making debug/release an #if condition because we considered #if to be the wrong point at which to start conditionalizing code generation for assertions. Though the final executable image's behavior is unavoidably dependent on whether asserts are enabled, we didn't want the SIL for inlineable code to be, since that would mean libraries with inlineable code would need to ship three times the amount of serialized SIL to support the right behavior in -Onone, -O, and -Ounchecked builds. Instead, the standard library has some hidden helper functions, `_isDebugAssertConfiguration`, `_isReleaseAssertConfiguration`, and `_isFastAssertConfiguration`, which are guaranteed to be constant-folded away before final code generation." 
 
Swift has matured sufficiently that incorporating a conditional build configuration and promoting one of the helper functions (albeit in negative form) makes sense for the language moving forward. [SE-0075](https://github.com/apple/swift-evolution/blob/master/proposals/0075-import-test.md) laid the groundwork for tests with [complex timing issues](https://bugs.swift.org/browse/SR-1560) in implementation. This proposal's prototype checks optimization and assertions after SIL options are populated.

## Source compatibility

This proposal is strictly additive.

## Effect on ABI stability

This proposal does not affect ABI stability.

## Effect on API resilience

This proposal does not affect ABI resilience.

## Alternatives Considered

It is possible to mark symbols with a `@condition(development)` flag that omits those symbols from inclusion in release builds. This is a compile-time solution that enhances readability and simplifies end-coder implementation. However it acts on a symbol-by-symbol basis. You cannot use it for specific lines where APIs differ.
