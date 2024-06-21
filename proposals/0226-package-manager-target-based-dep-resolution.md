# Package Manager Target Based Dependency Resolution

* Proposal: [SE-0226](0226-package-manager-target-based-dep-resolution.md)
* Authors: [Ankit Aggarwal](https://github.com/aciidb0mb3r)
* Review Manager: [Boris Bügling](https://github.com/neonichu)
* Status: **Partially implemented (Swift 5.2):** Implemented the manifest API to disregard targets not concerned by any dependency products, which avoids building dependency test targets.
* Bug: [SR-8658](https://bugs.swift.org/browse/SR-8658)
* Previous Revision: [1](hhttps://github.com/swiftlang/swift-evolution/blob/e833c4d00bef253452b7d546e1303565ba584b58/proposals/0226-package-manager-target-based-dep-resolution.md)
* Review: [Review](https://forums.swift.org/t/se-0226-package-manager-target-based-dependency-resolution/), [Acceptance](https://forums.swift.org/t/accepted-se-0226-package-manager-target-based-dependency-resolution/15616), [Amendment](https://forums.swift.org/t/amendment-se-0226-package-manager-target-based-dependency-resolution/), [Amendment Acceptance](https://forums.swift.org/t/accepted-se-0226-amendment-package-manager-target-based-dependency-resolution/)

## Introduction

This is a proposal for enhancing the package resolution process to resolve
the minimal set of dependencies that are used in a package graph.

## Motivation

The current package resolution process resolves all declared dependencies in
the package graph. Some of the declared dependencies may not be required by the
products that are being used in the package graph. For e.g., a package may be
using some additional dependencies for its test targets. The packages that
depend on this package doesn't need to resolve such additional dependencies. These
dependencies increase the overall constraint in the dependency resolution process
that can otherwise be avoided. It can cause more cases of dependency hell if two
packages want to use incompatible versions of a dependency that they only use
for their unexported products. Cloning unnecessary dependencies also impacts the
performance of the resolution process.

Another example of packages requiring additional dependencies is for sample code
targets. A library package may want to create an executable target which
demonstrates some functionality of the library. This executable may require
other dependencies such as a command-line argument parser.

## Proposed solution

We propose to enhance the dependency resolution process to resolve only the
dependencies that are actually being used in the package graph. The resolution
process can examine the target dependencies to figure out which package
dependencies require resolution. Since we will only resolve what is required, it
may reduce the odds of dependency hell situations.

To achieve this, the package manager needs to associate the product dependencies
with the packages which provide those products without cloning them. We propose
to make the `package` parameter in the product dependency declaration
non-optional. This is necessary because if the package is not specified,
the package manager is forced to resolve all package dependencies just to figure
out which products are vended by which packages.

```swift
extension Target.Dependency {
    static func product(name: String, package: String) -> Target.Dependency
}
```

e.g.

```
.target(
    name: "foo",
    dependencies: [.product(name: "bar", package: "bar-package")]
),
```

SwiftPM will retain support for the simplified `byName` declaration for products
that are named after the package.
This provides a shorthand for the common case of small packages that vend just
one product, and the product and package names are aligned.

```swift
extension Target.Dependency {
    static func byName(_ name: String) -> Target.Dependency
}
```

e.g.

```
.target(
    name: "foo",
    dependencies: ["bar"]
),
```

To correlate between the package referenced in the target and the declared dependency,
SwiftPM will need to compute an identity for the declared dependencies without
cloning them.

When this feature was first implemented in SwiftPM version 5.2, the package identity needed
to be specified using the `name` attribute on the package dependency declaration.
Such `name` attribute had to also align to the name declared in the dependency's manifest,
which led to adoption friction given that the manifest name is rarely known by
the users of the dependency.

**5.2 version**

```swift
extension Package.Dependency {
    static func package(
        name: String? = nil,    // Proposed
        url: String,
        from version: Version
    ) -> Package.Dependency

    static func package(
        name: String? = nil,    // Proposed
        url: String,
        _ requirement: Package.Dependency.Requirement
    ) -> Package.Dependency

    static func package(
        name: String? = nil,    // Proposed
        url: String,
        _ range: Range<Version>
    ) -> Package.Dependency

    static func package(
        name: String? = nil,    // Proposed
        url: String,
        _ range: ClosedRange<Version>
    ) -> Package.Dependency
}
```

Starting with the SwiftPM version following 5.4 (exact number TBD), SwiftPM will actively discourage the use of the
`name` attribute on the package dependency declaration (will emit warning when used with tools-version >= TBD)
and instead will compute an identity for the declared dependency by using the last path
component of the dependency URL (or path in the case of local dependencies) in the dependencies section.
The dependency URL used for this purpose is as-typed (no percent encoding or other transformation), and regardless of any configured mirrors.
With this change, the name specified in the dependency manifest will have no bearing
over target based dependencies (other than for backwards compatibility).

Note: [SE-0292] (when accepted and implemented) will further refine how package identities are computed.

  [SE-0292]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0292-package-registry-service.md

**TBD version**

```swift
extension Package.Dependency {
    static func package(
        name: String? = nil,    // Usage will emit warning and eventually be deprecated
        url: String,
        from version: Version
    ) -> Package.Dependency

    static func package(
        name: String? = nil,    // Usage will emit warning and eventually be deprecated
        url: String,
        _ requirement: Package.Dependency.Requirement
    ) -> Package.Dependency

    static func package(
        name: String? = nil,    // Usage will emit warning and eventually be deprecated
        url: String,
        _ range: Range<Version>
    ) -> Package.Dependency

    static func package(
        name: String? = nil,    // Usage will emit warning and eventually be deprecated
        url: String,
        _ range: ClosedRange<Version>
    ) -> Package.Dependency
}
```

## Detailed design

The resolution process will start by examining the products used in the target
dependencies and figure out the package dependencies that vend these products.
For each dependency, the resolver will only clone what is necessary to build
the products that are used in the dependees.

The products declared in the target dependencies will need to provide their
package identity unless the package and product have the same name. SwiftPM will
diagnose the invalid product declarations and emit an error.

As an example, consider the following package manifests:

```swift
// IRC package
let package = Package(
    name: "irc",
    products: [
        .library(name: "irc", targets: ["irc"]),
        .executable(name: "irc-sample", targets: ["irc-sample"]),
    ],
    dependencies: [
       .package(url: "https://github.com/apple/swift-nio.git", from: "1.0.0"),

       .package(url: "https://github.com/swift/ArgParse.git", from: "1.0.0"),
       .package(url: "https://github.com/swift/TestUtilities.git", from: "1.0.0"),
    ],
    targets: [
        .target(
            name: "irc",
            dependencies: [.product(name: "NIO", package: "swift-nio"),]
        ),

        .target(
            name: "irc-sample",
            dependencies: ["irc", "ArgParse"]
        ),

        .testTarget(
            name: "ircTests",
            dependencies: [
                "irc",
                .product(name: "Nimble", package: "TestUtilities"),
            ]
        )
    ]
)

// IRC Client package
let package = Package(
    name: "irc-client",
    products: [
        .executable(name: "irc-client", targets: ["irc-client"]),
    ],
    dependencies: [
       .package(url: "https://github.com/swift/irc.git", from: "1.0.0"),
    ],
    targets: [
        .target(name: "irc-client", dependencies: ["irc"]),
    ]
)
```

When the package "irc-client" is resolved, the package manager will only create
checkouts for the packages "irc" and "swift-nio" as "ArgParse" is used by
"irc-sample" but that product is not used in the "irc-client" package and
"Nimble" is used by the test target of the "irc" package.

## Impact on existing packages

There will be no impact on the existing packages. All changes, both behavioral
and API, will be guarded against the tools version this proposal is implemented
in. It is possible to form a package graph with mix-and-match of packages
with different tools versions. Packages with the older tools version will
resolve all package dependencies, while packages using the newer tools version
will only resolve the package dependencies that are required to build the used
products.

As described in the proposal, the package manager will stop resolving the unused
dependencies. There will be no `Package.resolved` entries and checkouts for such
dependencies.

Declaring target dependency on a product from an already resolved dependency
could potentially trigger the dependency resolution process, which in turn could
lead to cloning more repositories or even dependency hell. Note that such
dependency hell situations will always happen in the current implementation.

## Alternatives considered

We considered introducing a way to mark package dependencies as "development" or
"test-only". Adding these types of dependencies would have introduced new API
and new concepts, increasing package manifest complexity. It could also require
complicated rules, or new workflow options, dictating when these dependencies
would be resolved. Instead, we rejected adding new APIs as the above proposal
can handle these cases without any new API, and in a more intuitive manner.
