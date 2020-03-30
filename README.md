# Swift Programming Language Evolution
[![Swift](https://img.shields.io/badge/Swift%205-Open%20to%20requests-brightgreen.svg)](#swift_stage)


**Before you initiate a pull request**, please read the process document. Ideas should be thoroughly discussed on the [swift-evolution forums](https://swift.org/community/#swift-evolution) first.

This repository tracks the ongoing evolution of Swift. It contains:

* Goals for upcoming Swift releases (this document).
* The [Swift evolution review status][proposal-status] tracking proposals to change Swift.
* The [Swift evolution process](process.md) that governs the evolution of Swift.
* [Commonly Rejected Changes](commonly_proposed.md), proposals that have been denied in the past.

This document describes goals for the Swift language on a per-release
basis. These releases include minor releases that add to the currently shipping
version plus one major release out.  Each release will have many
smaller features or changes independent of these larger goals, and not
all goals will be reached for each release.

For historical purposes, the bottom of the document includes goals for past versions. These goals do not necessarily indicate which
features actually shipped for a given version. Those are documented in each version’s release notes.

<a name="swift_stage"></a>

## Development major version:  Swift 5.0

Released on March 25, 2019

### Primary Focus: ABI Stability

The Swift 5 release **will** provide [ABI stability](https://github.com/apple/swift/blob/master/docs/ABIStabilityManifesto.md#what-is-abi-stability) for the Swift Standard Library.  ABI stability enables OS vendors to embed a Swift Standard Library and runtime in the OS that is compatible with applications built with Swift 5 or later.  Progress towards achieving ABI stability will be tracked at a high level on the [ABI Dashboard](https://swift.org/abi-stability/).

ABI stability is only one of two pieces needed to support binary frameworks. The second half is *module stability* (see "[The Big Picture](https://github.com/apple/swift/blob/master/docs/ABIStabilityManifesto.md#the-big-picture)" of the [ABI Stability Manifesto](https://github.com/apple/swift/blob/master/docs/ABIStabilityManifesto.md) for more information).  While we’d like to support this for Swift 5, it will be a stretch goal, and may not make it in time.

The need to achieve ABI stability in Swift 5 will guide most of the priorities for the release.  In addition, there are important goals to complete that carry over from Swift 4 that are prerequisites to locking down the ABI of the standard library:

- **Generics features needed for standard library**.  We will finish implementing [conditional conformances](https://github.com/apple/swift-evolution/blob/master/proposals/0143-conditional-conformances.md) and [recursive protocol requirements](https://github.com/apple/swift-evolution/blob/master/proposals/0157-recursive-protocol-constraints.md), which are needed for the standard library to achieve ABI stability.  Both of these have gone through the evolution proposal process and there are no known other generics enhancements needed for ABI stability.

- **API resilience**. We will implement the essential pieces needed to support API resilience, in order to allow public APIs for a library to evolve over time while maintaining a stable ABI.

- **Memory ownership model**. An (opt-in) Cyclone/Rust-inspired memory [ownership model](https://github.com/apple/swift/blob/master/docs/OwnershipManifesto.md) is strongly desirable for systems programming and for other high-performance applications that require predictable and deterministic performance.  Part of this model was introduced in Swift 4 when we began to [ enforce exclusive access to memory](https://github.com/apple/swift-evolution/blob/master/proposals/0176-enforce-exclusive-access-to-memory.md).  In Swift 5 our goal is to tackle the [pieces of the ownership model that are key to ABI stability](https://github.com/apple/swift/blob/master/docs/OwnershipManifesto.md#priorities-for-abi-stability).

### Other Improvements

Beyond ABI stability (which focuses mostly on getting a bunch of low-level implementation details of the language finalized), in Swift 5 the evolution process welcomes additions that improve the overall usability of the language and standard library, including but not restricted to:

- **String ergonomics**. We will complete more of the work outlined in the [String Manifesto](https://github.com/apple/swift/blob/master/docs/StringManifesto.md) to make `String` easier to use and more performant.  This work may include the addition of new text processing affordances to the language and standard library, and language-level support for regular expressions.  In addition to ergonomic changes, the internal implementation of `String` offers many opportunities for enhancing performance which we would like to exploit.

- **Improvements to existing standard library facilities**. We will consider other minor additions to existing library features, but are not open for significant new facilities outside of supporting the primary focuses of this release.

- **Foundation improvements**. We anticipate proposing some targeted improvements to Foundation API to further the goal of making the Cocoa SDK work seamlessly in Swift.

- **Syntactic additions**. Syntactic changes do not increase the expressive power of the language but do increase its complexity.  Consequently, such changes must be extremely well-motivated and will be subject to additional scrutiny.  We will expect proposals to include concrete data about how widespread the positive impact will be.

- **Laying groundwork for a new concurrency model**. We will lay groundwork for a new concurrency model, especially as needed for ABI stability.  Finalizing such a model, however, is a *non-goal* for Swift 5.  A key focus area will be on designing language affordances for creating and using asynchronous APIs and dealing with the problems created by callback-heavy code.

### Source Stability

Similar to [Swift 4](releases/swift-4_0.md) , the Swift 5 compiler will provide a source compatibility mode to allow source code written using some previous versions of Swift to compile with the Swift 5 compiler.  The Swift 5 compiler will at least support code written in Swift 4, but may also extend back to supporting code written in Swift 3.  The final decision on the latter will be made in early 2018.

Source-breaking changes in Swift 5 will have an even higher bar than in Swift 4, following these guidelines:

* The current syntax/API must be shown to actively cause problems for users.
* The new syntax/API must be clearly better and must not conflict with existing Swift syntax.
* There must be a reasonably automated migration path for existing code.

### Evolution Process for Swift 5

Unlike [Swift 4](releases/swift-4_0.md), there will be no "stage 1" and "stage 2" for the evolution process.  Proposals that fit within the general focus of the release are welcome until **March 1, 2018**.  Proposals will still be considered after that, but the bar will be increasingly high to accept changes for Swift 5.

The broader range of proposals for Swift 5 compared to Swift 4 incurs the risk of diluting the focus on ABI stability.
To mitigate that risk, **every evolution proposal will need a working implementation, with test cases, in order to be considered for review**.  An idea can be pitched and a proposal written prior to providing an implementation, but a pull request for a proposal will not be accepted for review until an implementation is available.

More precisely:

1. Once a proposal is written, the authors submit the proposal via a pull request to the `swift-evolution` repository.

2. The Core Team will regularly review `swift-evolution` pull requests, and provide feedback to the authors in the pull request on whether or not the proposal looks within reason of something that might be accepted.

3. If a proposal gets a positive indicator from the Core Team for later review, the authors must provide an implementation prior to the proposal being formally reviewed.  An implementation should be provided in the form of a pull request against the impacted repositories (e.g., `swift`, `swift-package-manager`), and the proposal should be updated with a link to that pull request.  It is expected that before review begins the pull request implementation runs at least the following tests:

    a. A full test on Linux and macOS. (e.x. "@swift-ci test").

    b. A full test of the Source Compatibility Suite to identify source breakage. (e.x. "@swift-ci test source compatibility").

    c. A full run of Swift Compiler Performance. (e.x. "@swift-ci test compiler performance").

    d. A full run of the Swift Benchmark Suite. (e.x. "@swift-ci benchmark").

    > _NOTE: The existence of an implementation does not guarantee that the proposal will be accepted, but it is instrumental in evaluating the quality and impact of the proposal._

We want to strike a balance between encouraging open discussion of potential changes to the language and standard library while also providing more focus when changes are actually reviewed.  Further, having implementations on hand allow the changes to be more easily tried out before they are officially accepted as part of the language.  In particular, development of the initial pull request for a proposal remains a very open review process that everyone in the community can contribute a lot to.  Similarly, members of the community can help craft implementations for a proposal even if they aren't the authors of the proposal.

## Previous releases

* [Swift 4.0](releases/swift-4_0.md) - Released on September 19, 2017
* [Swift 3.0](releases/swift-3_0.md) - Released on September 13, 2016
* [Swift 2.2](releases/swift-2_2.md) - Released on March 21, 2016

[proposal-status]: https://apple.github.io/swift-evolution/
