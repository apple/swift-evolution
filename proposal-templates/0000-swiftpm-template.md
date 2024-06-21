# Package Manager Feature name

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Author 1](https://github.com/swiftdev), [Author 2](https://github.com/swiftdev)
* Review Manager: TBD
* Status: **Awaiting implementation**

*During the review process, add the following fields as needed:*

* Implementation: [apple/swift-package-manager#NNNNN](https://github.com/apple/swift-package-manager/pull/NNNNN)
* Decision Notes: [Rationale](https://forums.swift.org/), [Additional Commentary](https://forums.swift.org/)
* Bugs: [SR-NNNN](https://bugs.swift.org/browse/SR-NNNN), [SR-MMMM](https://bugs.swift.org/browse/SR-MMMM)
* Previous Revision: [1](https://github.com/swiftlang/swift-evolution/blob/...commit-ID.../proposals/NNNN-filename.md)
* Previous Proposal: [SE-XXXX](XXXX-filename.md)

## Introduction

A short description of what the feature is. Try to keep it to a single-paragraph
"elevator pitch" so the reader understands what problem this proposal is
addressing.

Swift-evolution thread: [Discussion thread topic for that
proposal](https://forums.swift.org/)

## Motivation

Describe the problems that this proposal seeks to address. If the problem is
that some functionality is currently hard to use, show how it is currently used
and describe its drawbacks. If it's completely new functionality that cannot be
emulated, motivate why this new functionality would help Swift developers create
better Swift packages.

## Proposed solution

Describe your solution to the problem. Provide examples and describe how they
work. Show how your solution is better than current workarounds: is it cleaner,
easier, or more efficient?

## Detailed design

Describe the design of the solution in detail. If it involves adding or
modifying functionality in the package manager, explain how the package manager
behaves in different scenarios and with existing features. If it's a new API in
the `Package.swift` manifest, show the full API and its documentation comments
detailing what it does.  The detail in this section should be sufficient for
someone who is *not* one of the author of the proposal to be able to reasonably
implement the feature.

## Security

Does this change have any impact on security, safety, or privacy?

## Impact on existing packages

Explain if and how this proposal will affect the behavior of existing packages.
If there will be impact, is it possible to gate the changes on the tools version
of the package manifest?

## Alternatives considered

Describe alternative approaches to addressing the same problem, and
why you chose this approach instead.
