# Package Manager Allow Targets to Depend on Products in the Same Package

* Proposal: [SE-NNNN](NNNN-swiftpm-same-package-product-dependencies.md)
* Authors: [stackotter](https://github.com/stackotter), [Tammo Freese](https://github.com/tammofreese)
* Review Manager: TBD
* Status: **Awaiting implementation**
* Implementation: [apple/swift-package-manager#7331](https://github.com/apple/swift-package-manager/pull/7331)

## Introduction

This proposal allows targets to depend on products within the same package; solving issues faced by packages vending multiple dynamic library products (such as code duplication and type casting related issues).

Swift-evolution thread: [discussion thread](https://forums.swift.org/t/pitch-swiftpm-allow-targets-to-depend-on-products-in-the-same-package/57717)

## Motivation

Consider a package `Library` with the following package manifest:

```swift
// Library/Package.swift
// swift-tools-version: 5.9

import PackageDescription

let package = Package(
    name: "Library",
    products: [
        .library(name: "API", type: .dynamic, targets: ["API"]),
        .library(name: "Auth", type: .dynamic, targets: ["Auth"]),
    ],
    targets: [
        .target(name: "API", dependencies: ["Auth"]),
        .target(name: "Auth"),
    ]
)
```

A consumer of `Library` may import both `LibAPI` and `LibAuth` leading to duplication of the `Auth` target in the final bundle of products (due to static linking between `API` and `Auth`). This increases code-size, and more importantly, breaks type casting in many scenarios (see [the example](#type-casting-example)).

### Existing workarounds

The current workaround for these issues is to separate the leaf product (i.e. `LibAuth`) into a separate package (often in a subdirectory of the root package) to allow dynamic linking between the offending targets/products (i.e. `API` would now depend on `LibAuth`). This has its own issues; it complicates what could be a very simple project structure, and it hides `LibAuth` from consumers of `Library` (because SwiftPM doesn't support vending multiple packages from a single repository, see [Rust's Cargo Workspaces](https://doc.rust-lang.org/book/ch14-03-cargo-workspaces.html)). In order for users of `LibAPI` to be able to use `LibAuth`, `LibAuth` would have to be imported with  `@_exported import LibAuth` in `API` (or in a dedicated re-exporting target). Alternatively, the two packages can be placed in separate repositories, however that often doesn't make sense for closely tied targets (e.g. an API client and its authentication implementation), and can hurt maintainability.

### Another motivating use-case (plugin systems)

Another motivating use-case is building plugin systems for tools/apps built with SwiftPM. The plugin API must be imported by both the app and the plugin, which requires dynamic linking between the app and the plugin API ([in order for type casting to work](#type-casting-example)), and means that the plugin API cannot be in the same package as the app. For the same reasons as above, this is unwieldy. The proposed changes would mean that such tools/apps would be able to be developed as a single package (which [greatly improves maintainability](https://forums.swift.org/t/pitch-swiftpm-allow-targets-to-depend-on-products-in-the-same-package/57717/12)).

This use-case may become increasingly common as people start making cross-platform apps with Swift (using SwiftPM instead of Xcode projects).

### Type casting example

This example demonstrates the type casting issue mentioned above. The full demo is hosted in [the `type-casting-issue-demo` repository](https://github.com/stackotter/type-casting-issue-demo) so that readers can see the issue in action.

When reading the following code, developers would expect that `isGitHubAccessToken(githubAccessToken)` is true, but it turns out that isn't! This is due to `LibAuth` and `LibAPI` being dynamic library products — changing them to static library products (and performing a clean build) restores the assumption.

```swift
// Library/Sources/Auth/Auth.swift
public protocol AccessToken { /* ... */ }

public struct MicrosoftAccessToken: AccessToken { /* ... */ }

public struct GitHubAccessToken: AccessToken { /* ... */ }

// Library/Sources/API/API.swift
import Auth

public func isGitHubAccessToken(_ accessToken: any AccessToken) -> Bool {
    return accessToken is GitHubAccessToken
}

// App/Sources/main.swift
import LibAPI
import LibAuth

let microsoftToken = MicrosoftAccessToken(/* ... */)
let githubToken = GitHubAccessToken(/* ... */)

print("microsoftToken is GitHubAccessToken == ", microsoftToken is GitHubAccessToken)
print("isGitHubAccessToken(microsoftToken) == ", isGitHubAccessToken(microsoftToken))
print("githubToken is GitHubAccessToken == ", githubToken is GitHubAccessToken)
print("isGitHubAccessToken(githubToken) == ", isGitHubAccessToken(githubToken))

// Output:
// > microsoftToken is GitHubAccessToken == false
// > isGitHubAccessToken(microsoftToken) == false
// > githubToken is GitHubAccessToken == true
// > isGitHubAccessToken(githubToken) == false
```

## Proposed solution

The proposed solution is to introduce a new method `Target.Dependency.product(name:condition:)` so that targets can depend on products in the same package. This enables dynamic linking between code within the same package fixing both code-size concerns and type-casting issues.

## Detailed design

A new `innerProductItem(name:condition:)` case will be added to `Target.Dependency` as the underlying representation for same-package product dependencies. It will be accompanied by a `product(name:condition:)` method — just as all existing enum cases of `Target.Dependency` have accompanying methods.

```swift
extension Target {
    public enum Dependency {
        /// A dependency on a product in the same package.
        ///
        /// - Parameters:
        ///    - name: The name of the product.
        ///    - condition: A condition that limits the application of the target dependency. For example, only apply a dependency for a specific platform.
        case innerProductItem(name: String, condition: TargetDependencyCondition?)
    }
}

extension Target.Dependency {
    /// Creates a dependency on a product from the same package.
    ///
    /// - Parameters:
    ///   - name: The name of the product.
    ///   - condition: A condition that limits the application of the target dependency. For example, only apply a
    ///       dependency for a specific platform.
    /// - Returns: A `Target.Dependency` instance.
    public static func product(
        name: String,
        condition: TargetDependencyCondition? = nil
    ) -> Target.Dependency {
        return .innerProductItem(name: name, condition: condition)
    }
}
```

SwiftPM's package graph and package builder will be updated to accomodate the new dependency type — the changes required are relatively self-contained.

The existing `Target.Dependency.product(name:package:moduleAliases:condition)` method will not be able to be used to depend on products in the current package as that would cause the meaning of the package manifest to change depending on the name of the package's root directory now that a package's name is no longer tied to the package manifest's `name` field. 

Similarly, products within the same package as a target will still be ignored when evaluating the target's by-name dependencies (otherwise this proposal would be introducing a breaking change).

## Impact on existing packages

This isn't a breaking change for users of SwiftPM, and won't affect existing packages.

## Alternatives considered

### Rust's Cargo Workspaces

Rust's Cargo package manager has a feature called [Cargo Workspaces](https://doc.rust-lang.org/book/ch14-03-cargo-workspaces.html) which allows multiple packages to be vended from a single repository. This is very useful for keeping subsystems of a large project separate, including being able to keep their dependencies separate. SwiftPM could introduce a similar system which would make the existing workaround much more manageable, by essentially making it a supported use-case. However this would be a massive undertaking and would likely require much stronger motivation to be worthwhile. Additionally, this can already be somewhat achieved using [a custom Swift package registry implementation designed to vend subdirectories of a repository as separate packages](https://github.com/stackotter/swiftpm-workspaces), however that has issues of its own for local development (the separate packages can't depend on eachother via paths which means that this solution is basically useless).

### Doing nothing

There is already a workaround, so one option would be to do nothing, but the workaround is sufficiently inconvenient and often requires use of the unofficial `@_exported` attribute. This was enough for multiple people to vocally want a better solution.
