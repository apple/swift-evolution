# Package Manager Platform Deployment Settings

* Proposal: [SE-0236](0236-package-manager-platform-deployment-settings.md)
* Authors: [Ankit Aggarwal](https://github.com/aciidb0mb3r)
* Review Manager: [Boris Bügling](https://github.com/neonichu)
* Status: **Implemented (Swift 5.0)**
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-with-modifications-se-0236-package-manager-platform-deployment-settings/18420)
* Previous Revisions: [1](https://github.com/swiftlang/swift-evolution/blob/aebb22c0f3e139fd921d14f79c3945af99d0342d/proposals/0236-package-manager-platform-deployment-settings.md)

## Introduction

This is a proposal for adding support for specifying a per-platform minimum required deployment target in the `Package.swift` manifest file.

## Motivation

Packages should be able to declare the minimum required platform deployment target version. SwiftPM currently uses a hardcoded value for the macOS deployment target. This creates friction for packages which want to use APIs that were introduced after the hardcoded deployment target version.

There are two ways to work around this limitation: 1) using availability checks, or 2) passing the deployment target on the command line while building the package. However, these workarounds are not ideal and the package manager should really provide a proper API for setting the required deployment target version.

## Proposed solution

We propose to add the following API to `PackageDescription`:

```swift
/// Represents a supported platform.
struct SupportedPlatform {
    static func macOS(_ version: MacOSVersion) -> SupportedPlatform
    static func macOS(_ versionString: String) -> SupportedPlatform

    static func tvOS(_ version: TVOSVersion) -> SupportedPlatform
    static func tvOS(_ versionString: String) -> SupportedPlatform

    static func iOS(_ version: IOSVersion) -> SupportedPlatform
    static func iOS(_ versionString: String) -> SupportedPlatform

    static func watchOS(_ version: WatchOSVersion) -> SupportedPlatform
    static func watchOS(_ versionString: String) -> SupportedPlatform
}

/// List of known versions.
extension SupportedPlatform.MacOSVersion {
    static let v10_10: MacOSVersion
    static let v10_11: MacOSVersion
    static let v10_12: MacOSVersion
    ...
}

final class Package {

    init(
        name: String,
        platforms: [SupportedPlatform]? = nil,
        ...
    )
}

// Example usage:

let package = Package(
    name: "NIO",
    platforms: [
       .macOS(.v10_13), .iOS(.v12),
    ],
    products: [
        .library(name: "NIO", targets: ["NIO"]),
    ],
    targets: [
        .target(name: "NIO"),
    ]
)
```

A package will be assumed to support all platforms using a predefined minimum deployment version. This predefined deployment version will be the oldest deployment target version supported by the installed SDK for a given platform. One exception to this rule is macOS, for which the minimum deployment target version will start from 10.10. Packages can choose to declare the minimum deployment target version for some platform by using the above APIs. For example:

* This declaration means that the package should use 10.13 on macOS, 12.0 on iOS and the default deployment target when compiled on other platforms:

```swift
    ...
    platforms: [
       .macOS(.v10_13), .iOS(.v12),
    ],
    ...
```

## Detailed design

Changes in deployment target versions should be considered as a major breaking change for the purposes of semantic versioning since the dependees of a library package can break when a library makes such a change.

SwiftPM will emit an error if a dependency is not compatible with the top-level package's deployment version, i.e., the deployment target of dependencies must be lower than or equal to top-level package's deployment target version for a particular platform.

Each package will be compiled with the deployment target specified by it. In theory, SwiftPM can use the top-level package's deployment version to compile the entire package graph since the deployment target versions of dependencies are guaranteed to be compatible. This might even produce more efficient compilation output but it also means that the users might start seeing a lot of warnings due to use of a higher version.

Each platform API has a string overload that can be used to construct versions that are not already provided by PackageDescription APIs. This could be because the new version was recently released or isn't appropriate to be included in the APIs (for e.g. dot versions). The version format for each platform will be documented in the API. Invalid values will be diagnosed and presented as manifest parsing errors.

SwiftPM will emit appropriate errors when an invalid value is provided for supported platforms. For e.g., an empty array, multiple declarations for the same platform, invalid version specification.

The generated Xcode project command will set the deployment target version for each platform.

### Future directions

The Swift compiler supports several platforms like macOS, iOS, Linux, Windows, Android. However, the runtime availability checks currently only work for Apple platforms. It is expected that support for more platforms in the availability APIs will be added gradually as the community and support for Swift compiler grows. We think that the package manager can use a similar direction for declaring the supported platforms. We can start by allowing packages to declare support for Apple platforms with version specifications as proposed by the above APIs. Depending on the need in the community, these APIs can be evolved over time by adding support for declaring the minimum deployment version for other platforms that Swift supports. For e.g., the API can be enhanced to declare the minimum required version for Windows and the API level for Android.

This proposal doesn't handle these problems:

1. **Restricting supported platforms**: Some packages are only meant to work on certain platforms. This proposal doesn't provide ability to restrict the list of supported platforms and only allows customizing the deployment target versions. Restricting supported platforms is orthogonal to customizing deployment target and will be explored separately.

2. **Platform-specific targets or products**: Consider that a package supports multiple platforms but has a product that should be only built for Linux. There is no way to express this intent and the build system may end up trying to build such targets on all platforms. A simple workaround is to use `#if` to conditionalize the source code of the target. Another use case comes up when you want to keep deployment target of a certain target lower than other targets in a package. A workaround is factoring out the target into its own package. A proper solution to this problem will be explored in a separate proposal, which should provide target-level settings.

3. **Platform-specific package dependencies**: Similar to the above issue, a package may want to use a certain dependency only when building for a specific platform. It is currently not possible to declare such a dependency without using `#if os` checks in the manifest file, which doesn't interact nicely with other features like `Package.resolved` and the cross-compilation support. This can be solved by providing APIs to declare platform-specific package dependencies in the manifest file. This will also be explored in a separate proposal.

## Impact on existing packages

Existing packages will not be impacted as the behavior described in this proposal is compatible with SwiftPM's current behavior, i.e., a package is assumed to support all platforms and SwiftPM picks a default deployment target version for the macOS platform.

The new APIs will be guarded against the tools version this proposal is implemented in. Packages that want to use this feature will need to update their tools version.

## Alternatives considered

We considered making the supported platform field mandatory. We think that it would provide little value in practice and cause more friction instead. One advantage would be that SwiftPM could use that information to produce error messages when a package tries to use a dependency which is not tested on some platform. We think that is not really a big enough issue. Since Swift is cross-platform, Swift packages should generally work on all platforms that Swift supports. However, there are platform-specific packages that are only meant for certain platforms and the proposed API does provide an option to declare that intent.

We considered taking the deployment target version into the dependency resolution process to automatically find a compatible version with the top-level package. This too doesn't provide enough value in practice. Library packages generally tend to stick with a deployment target to avoid breaking their dependees. When they do bump the version, it is generally a well-thought decision and may require a semver upgrade anyway due to updates in the API.
