# Package Manager Target Specific Build Settings

* Proposal: [SE-0238](0238-package-manager-build-settings.md)
* Decision Notes: [Draft Thread](https://forums.swift.org/t/draft-proposal-target-specific-build-settings/18031)
* Authors: [Ankit Aggarwal](https://github.com/aciidb0mb3r)
* Review Manager: [Boris Bügling](https://github.com/neonichu)
* Status: **Implemented (Swift 5.0)**
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-with-modifications-se-0238-package-manager-target-specific-build-settings/18590)
* Previous Revisions: [1](https://github.com/swiftlang/swift-evolution/blob/1a2801a3dc912b093f2cda13eafd54f0d98b3c8e/proposals/0238-package-manager-build-settings.md)

## Introduction

This is a proposal for adding support for declaring some commonly used target-specific build settings in the `Package.swift` manifest file. As the name suggests, target-specific build settings are only applied to a particular target. SwiftPM also aims to support cross-target build settings that go across the target boundary and impart certain settings on a target's dependees, but this proposal is only concerned with the former type of build settings and the latter will be explored with a future proposal.

## Motivation

SwiftPM currently has little facility for customizing how the build tools (compilers, linker, etc.) are invoked during a build. This causes a lot of friction for package authors who want to do some basic customizations in order to build their targets. They often have to resort to awkward workarounds like creating custom modulemaps for linking system libraries, symlinking private headers inside the include directory, changing the include statements, and so on.

We think most of these workarounds can be removed by providing support for some common build settings at the target level. This proposal will also set the stage for a richer build settings API in the future that has support for various conditional expressions, deployment options, inheritance of build settings, etc.

## Proposed solution

We propose to add four new arguments to the target factory method: `cSettings`, `cxxSettings`, `swiftSettings` and `linkerSettings`. The build settings specified in these arguments will be used to compile a particular target and the settings will not affect any other target in the package or the package graph. The API will also allow conditionalization using a `.when` modifier on two parameters: platforms and build configuration.

We propose to add the following build settings in this proposal:

*Note: `<BuildSettingType>` represents the concrete type of a certain setting. Possible types are `CSetting`, `CXXSetting`, `SwiftSetting` or `LinkerSetting`. Each build setting in the upcoming section contains the method signature that will be available in their corresponding <BuildSettingType>.*

### Header search path (C/CXX)

```swift
static func headerSearchPath(_ path: String, _ condition: BuildSettingCondition? = nil) -> <BuildSettingType>
```

Many C-family projects are structured in a way that requires adding header search paths to different directories of the project. Currently, SwiftPM only adds a search path to the `include` directory which makes it difficult for many C projects to add support for building with SwiftPM. This specified path should be relative to the target and should not escape the package boundary. Absolute paths are disallowed.

*Note: It is not recommended to use this setting for adding search paths of public headers as the target-specific settings are not imparted onto other targets.*

### Define (C/CXX)

```swift
static func define(_ name: String, to value: String? = nil, _ condition: BuildSettingCondition? = nil) -> <BuildSettingType>
```

This setting will add the `-D<name>=<value>` flag during a target's compilation. This is useful for projects that want to specify a compile-time condition.

*Note: It is not recommended to use this setting for public headers as the target-specific settings are not imparted.*

### Define (Swift)

```swift
static func define(_ name: String, _ condition: BuildSettingCondition? = nil) -> SwiftSetting
```

This setting enables the specified compilation condition for Swift targets. Unlike C/CXX's define, it doesn't have an associated value.

### Link library (Linker)

```swift
static func linkedLibrary(_ libraryName: String, _ condition: BuildSettingCondition? = nil) -> <BuildSettingType>
```

This is useful for packages that want to link against a library present in the system. The current approach requires them to create a module map using system library targets or a fake C target in order to achieve this effect. There is also no provision for conditionalization based on the platform in the existing approach, which is valuable when writing cross-platform packages.

### Link framework (Linker)

```swift
static func linkedFramework(_ frameworkName: String, _ condition: BuildSettingCondition? = nil) -> <BuildSettingType>
```

Frameworks are autolinked for Swift and C/ObjC targets so most packages shouldn't require this build setting. However, packages that contain C++ files can't autolink the frameworks. Since frameworks are widely used on Apple platforms, it is recommended to use this setting with a platform conditional.

### Unsafe flags (All)

```swift
static func unsafeFlags(_ flags: [String], _ condition: BuildSettingCondition? = nil) -> <BuildSettingType>
```

This is an escape hatch that will allow targets to pass arbitrary command-line flags to the corresponding build tool. The "unsafe" here implies that SwiftPM can't safely determine if the build flags will have any negative side-effect to the build since certain flags can change the behavior of how a build is performed. It is similar to how the `-Xcc`, `-Xswiftc`, `-Xlinker` option work in the command-line SwiftPM tools.

The primary purpose of this escape hatch is to enable experimentation and exploration for packages that currently use a makefile or script to pass the `-X*` flags. Products that contain a target which uses an unsafe flag will be ineligible to act as a dependency for other packages.

We have several such conditions (use of local dependencies, branch-based dependencies etc) that makes a package (individual products in this case) ineligible for acting as a dependency. This feature would be one more in that category. SwiftPM could provide a "pre-publish" command to detect and report such cases. RFC: https://forums.swift.org/t/rfc-swift-package-publish-precheck/15398

### Conditionalization

`static func when(platforms: [Platform]? = nil, configuration: BuildConfiguration? = nil) -> BuildSettingCondition`

By default, build settings will be applicable for all platforms and build configurations. The `.when` modifier can be used to conditionalize a build setting. SwiftPM will diagnose invalid usage of `.when` and emit a manifest parsing error. For e.g., it is invalid to specify a when condition with both parameter as `nil`.

### Example

Here is an example usage of the proposed APIs:

```swift
...
.target(
    name: "MyTool",
    dependencies: ["Yams"],
    cSettings: [
        .define("BAR"),
        .headerSearchPath("path/relative/to/my/target"),

        .define("DISABLE_SOMETHING", .when(platforms: [.iOS], configuration: .release)),
        .define("ENABLE_SOMETHING", .when(configuration: .release)),

        // Unsafe flags will be rejected by SwiftPM when a product containing this 
        // target is used as a dependency.
        .unsafeFlags(["-B=imma/haxx0r"]),
    ],
    swiftSettings: [
        .define("API_VERSION_5"),
    ],
    linkerSettings: [
        .linkLibrary("z"),
        .linkFramework("CoreData"),

        .linkLibrary("openssl", .when(platforms: [.linux])),
        .linkFramework("CoreData", .when(platforms: [.macOS], configuration: .debug)),

        // Unsafe flags will be rejected by SwiftPM when a product containing this
        // target is used as a dependency.
        .unsafeFlags(["-L/path/to/my/library", "-use-ld=gold"], .when(platforms: [.linux])),
    ]
),
...
```

## Detailed design

#### Use of a declarative model

Using a declarative model for build settings (and, in general, all PackageDescription APIs) allows SwiftPM to understand the complete package manifest, including the conditionals that may currently evaluate to false. This information can be used to build some advanced features like mechanically editing the manifest file and it also allows possibility for a “migrator” feature for upgrading the APIs as they evolve.

It is important to consider the impact of each build setting that is allowed to be used in a package. Certain build flags can be unsafe when configured without a more expressive build settings model, which can lead to non-hermetic builds. They can also cause bad interaction with the compilation process as certain flags can have a large impact on how the build is performed (for e.g. Swift compiler's `-wmo`). Some flags can even be exploited to link pre-compiled binaries without being officially supported by the package manager, which can be a huge security issue. Other flags (like `-B`) can be used to change the directory where the tools are looked up by the compiler. In the future, we can enhance the build system to perform builds in a highly sandboxed environment and potentially loosen the restrictions from unsafe flags as such vulnerabilities will no longer be possible. The package author will immediately run into build errors in such a sandbox.

#### Sharing build settings between tools

If a Swift target specifies both Swift and C settings, the flags produced by C settings will be added to Swift compiler by prefixing each flag with `-Xcc`. Similarly, if a C-family target specifies both C and CXX settings, the flags produced by C settings will be added to C++ compiler by prefixing the flags with `-Xcc`. This behavior is similar to what the command-line SwiftPM does for the `-X*` overrides. This strategy doesn't allow passing C flags that should be only passed to the C++ compiler but that is a very rare case.

## Future direction

One of the major goal of this proposal is to introduce the infrastructure for build settings with some frequently used ones. There are many other build settings that can be safely added to the proposed API. Such additional build settings can be explored in a separate proposal.

In the long term, SwiftPM aims to have a more complex build settings model with a rich API that allows expressing conditionalization on the various parameter, macro expansion, etc. We believe that the experience we gain with the proposed API will help us in fleshing out the future build settings API.

## Impact on existing packages

There is no impact on existing packages as this is an additive feature. Packages that want to use the new build settings APIs will need to upgrade their manifest's tools version to the version this proposal implemented in.

## Alternatives considered

We considered making the API to be just an array of `String` that can take arbitrary build flags. However, that requires SwiftPM to implement parsing logic in order to determine if the flags are in the whitelist or not. The compiler flags are very difficult to parse and there are several variations accepted by the compiler for different flags. The other option was standardizing on the syntax of each whitelisted flag but that would require package authors to lookup SwiftPM's documentation to figure out which variation is accepted.

We considered another spelling for the proposal API but rejected it because we expect that most packages that need to add a build settings will require only one of the four type. It seems unnecessary to have package authors do nesting in order to add a single flag.

```swift
...
.target(
    name: "foo",
    dependencies: ["Yams"],
    settings: [
        .swift([
            .define("BAR"),
        ]),
        .linker([
            .linkLibrary("z"),
        ]),
    ]
),
...
```
