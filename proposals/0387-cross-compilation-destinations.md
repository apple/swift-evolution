# Swift SDKs for Cross-Compilation

* Proposal: [SE-0387](0387-cross-compilation-destinations.md)
* Authors: [Max Desiatov](https://github.com/MaxDesiatov), [Saleem Abdulrasool](https://github.com/compnerd), [Evan Wilde](https://github.com/etcwilde)
* Review Manager: [Mishal Shah](https://github.com/shahmishal)
* Status: **Accepted**
* Implementation: [apple/swift-package-manager#5911](https://github.com/apple/swift-package-manager/pull/5911),
  [apple/swift-package-manager#5922](https://github.com/apple/swift-package-manager/pull/5922),
  [apple/swift-package-manager#6023](https://github.com/apple/swift-package-manager/pull/6023), 
  [apple/swift-package-manager#6186](https://github.com/apple/swift-package-manager/pull/6186)
* Review: ([pitch](https://forums.swift.org/t/pitch-cross-compilation-destination-bundles/61777))
  ([first review](https://forums.swift.org/t/se-0387-cross-compilation-destination-bundles/62875))
  ([second review](https://forums.swift.org/t/second-review-se-0387-cross-compilation-destination-bundles/64660))

## Table of Contents

- [Introduction](#introduction)
- [Motivation](#motivation)
- [Proposed Solution](#proposed-solution)
- [Detailed Design](#detailed-design)
  - [Swift SDK Bundles](#swift-sdk-bundles)
  - [`toolset.json` Files](#toolsetjson-files)
  - [`swift-sdk.json` Files](#swift-sdkjson-files)
  - [Swift SDK Installation and Configuration](#swift-sdk-installation-and-configuration)
  - [Using a Swift SDK](#using-a-swift-sdk)
  - [Swift SDK Bundle Generation](#swift-sdk-bundle-generation)
- [Security](#security)
- [Impact on Existing Packages](#impact-on-existing-packages)
- [Prior Art](#prior-art)
  - [Rust](#rust)
  - [Go](#go)
- [Alternatives Considered](#alternatives-considered)
  - [Extensions Other Than `.artifactbundle`](#extensions-other-than-artifactbundle)
  - [Building Applications in Docker Containers](#building-applications-in-docker-containers)
  - [Alternative Bundle Formats](#alternative-bundle-formats)
- [Making Swift SDK Bundles Fully Self-Contained](#making-swift-sdk-bundles-fully-self-contained)
- [Future Directions](#future-directions)
  - [Identifying Platforms with Dictionaries of Properties](#identifying-platforms-with-dictionaries-of-properties)
  - [SwiftPM Plugins for Remote Running, Testing, Deployment, and Debugging](#swiftpm-plugins-for-remote-running-testing-deployment-and-debugging)
  - [`swift sdk select` Subcommand](#swift-sdk-select-subcommand)
  - [SwiftPM and SourceKit-LSP Improvements](#swiftpm-and-sourcekit-lsp-improvements)
  - [Source-Based Swift SDKs](#source-based-swift-sdks)
  - [Swift SDK Bundles and Package Registries](#swift-sdk-bundles-and-package-registries)

## Introduction

Cross-compilation is a common development use case. When cross-compiling, we need to refer to these concepts:

- a **toolchain** is a set of tools used to build an application or a library;
- a **triple** describes features of a given machine such as CPU architecture, vendor, OS etc, corresponding to LLVM's
  triple;
- a **host triple** describes a machine where application or library code is built;
- a **target triple** describes a machine where application or library code is running;
- an **SDK** is a set of dynamic and/or static libraries, headers, and other resources required to generate code for the
  target triple.

When a triple of a machine on which the toolchain is built is different from the host triple, we'll call it a **build triple**.
The cross-compilation configuration itself that involves three different triples is called
[the Canadian Cross](https://en.wikipedia.org/wiki/Cross_compiler#Canadian_Cross).

Let’s call a Swift toolchain and an SDK bundled together in an artifact bundle a **Swift SDK**.

## Motivation

In Swift 5.8 and earlier versions users can cross-compile their code with so called "destination files" passed to 
SwiftPM invocations. These destination files are produced on an ad-hoc basis for different combinations of
host and target triples. For example, scripts that produce macOS → Linux destinations were created by both
[the Swift
team](https://github.com/apple/swift-package-manager/blob/swift-5.8-RELEASE/Utilities/build_ubuntu_cross_compilation_toolchain)
and [the Swift community](https://github.com/SPMDestinations/homebrew-tap). At the same time, the distribution process
of assets required for cross-compiling is cumbersome. After building a destination tree on the file system, required 
metadata files rely on hardcoded absolute paths. Adding support for relative paths in destination's metadata and 
providing a unified way to distribute and install required assets as archives would clearly be an improvement for the 
multi-platform Swift ecosystem.

The primary audience of this pitch are people who cross-compile from macOS to Linux. When deploying to single-board
computers supporting Linux (e.g. Raspberry Pi), building on such hardware may be too slow or run out of available
memory. Quite naturally, users would prefer to cross-compile on a different machine when developing for these platforms.

In other cases, building in a Docker container is not always the best solution for certain development workflows. For
example, when working with Swift AWS Lambda Runtime, some developers may find that installing Docker just for building a
project is a daunting step that shouldn’t be required.

The solution described below is general enough to scale for any host/target triple combination.

## Proposed Solution

Since a Swift SDK is a collection of binaries arranged in a certain directory hierarchy, it makes sense to distribute
it as an archive. We'd like to build on top of
[SE-0305](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0305-swiftpm-binary-target-improvements.md) and
extend the `.artifactbundle` format to support this.

Additionally, we propose introducing a new `swift sdk` CLI command for installation and removal of Swift SDKs on the 
local filesystem.

We introduce a notion of a top-level toolchain, which is the toolchain that handles user’s `swift sdk`
invocations. Parts of this top-level toolchain (linker, C/C++ compilers, and even the Swift compiler) can be overridden
with tools supplied in `.artifactbundle` s installed by `swift sdk` invocations.

When the user runs `swift build` with the selected Swift SDK, the overriding tools from the corresponding bundle
are invoked by `swift build` instead of tools from the top-level toolchain.

The proposal is intentionally limited in scope to build-time experience and specifies only configuration metadata, basic
directory layout for proposed artifact bundles, and some CLI helpers to operate on those.

## Detailed Design

### Swift SDK Bundles

As a quick reminder for a concept introduced in
[SE-0305](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0305-swiftpm-binary-target-improvements.md), an
**artifact bundle** is a directory that has the filename suffix `.artifactbundle` and has a predefined structure with
`.json` manifest files provided as metadata.

The proposed structure of artifact bundles containing Swift SDKs looks like:

```
<name>.artifactbundle
├ info.json
├ <Swift SDK artifact>
│ ├ swift-sdk.json
│ ├ toolset.json
│ └ <Swift SDK files and directories>
├ <Swift SDK artifact>
│ ├ swift-sdk.json
│ ├ toolset.json
│ └ <Swift SDK files and directories>
├ <Swift SDK artifact>
┆ └┄
```

For example, a Swift SDK bundle allowing to cross-compile Swift 5.9 source code to recent versions of Ubuntu from
macOS would look like this:

```
swift-5.9_ubuntu.artifactbundle
├ info.json
├ toolset.json
├ ubuntu_jammy
│ ├ swift-sdk.json
│ ├ toolset.json
│ ├ <Swift SDK files and directories shared between triples>
│ ├ aarch64-unknown-linux-gnu
│ │ ├ toolset.json
│ │ └ <triple-specific Swift SDK files and directories>
│ └ x86_64-unknown-linux-gnu
│   ├ toolset.json
│   └ <triple-specific Swift SDK files and directories>
├ ubuntu_focal
│ ├ swift-sdk.json
│ └ x86_64-unknown-linux-gnu
│   ├ toolset.json
│   └ <triple-specific Swift SDK files and directories>
├ ubuntu_bionic
┆ └┄
```

Here each artifact directory is dedicated to a specific Swift SDK, while files specific to each triple are placed
in `aarch64-unknown-linux-gnu` and `x86_64-unknown-linux-gnu` subdirectories.

`info.json` bundle manifests at the root of artifact bundles should specify `"type": "swiftSDK"` for
corresponding artifacts. Artifact identifiers in this manifest file uniquely identify a Swift SDK, and
`supportedTriples` property in `info.json` should contain host triples that a given Swift SDK supports. The rest
of the properties of bundle manifests introduced in SE-0305 are preserved.

Here's how `info.json` file could look like for `swift-5.9_ubuntu.artifactbundle` introduced in the example
above:

```json5
{
  "artifacts" : {
    "swift-5.9_ubuntu22.04" : {
      "type" : "swiftSDK",
      "version" : "0.0.1",
      "variants" : [
        {
          "path" : "ubuntu_jammy",
          "supportedTriples" : [
            "arm64-apple-darwin",
            "x86_64-apple-darwin"
          ]
        }
      ]
    },
    "swift-5.9_ubuntu20.04" : {
      "type" : "swiftSDK",
      "version" : "0.0.1",
      "variants" : [
        {
          "path" : "ubuntu_focal",
          "supportedTriples" : [
            "arm64-apple-darwin",
            "x86_64-apple-darwin"
          ]
        }
      ]
    }
  },
  "schemaVersion" : "1.0"
}
```

### `toolset.json` Files

We find that properties dedicated to tools configuration are useful outside of the cross-compilation context. Due to
that, separate toolset configuration files are introduced:

```json5
{
  "schemaVersion": "1.0",
  "rootPath": "optional path to a root directory containing toolchain executables",
  // If `rootPath` is specified, all relative paths below will be resolved relative to `rootPath`.
  "swiftCompiler": {
    "path": "<optional path to the Swift compiler>",
    "extraCLIOptions": ["<optional array of additional flags passed to the Swift compiler>"]
  },
  "cCompiler": {
    "path": "<optional path to the C compiler>",
    "extraCLIOptions": ["<optional array of additional flags passed to the C compiler>"]
  },
  "cxxCompiler": {
    "path": "<optional path to the C++ compiler>",
    "extraCLIOptions": ["<optional array of additional flags passed to the C++ compiler>"]
  },
  "linker": {
    "path": "<optional path to the linker>",
    "extraCLIOptions": ["<optional array of additional flags passed to the linker>"]
  },
  "librarian": {
    "path": "<optional path to the librarian, such as `libtool`, `ar`, or `link`>",
    "extraCLIOptions": ["<optional array of additional flags passed to the librarian>"]
  },
  "debugger": {
    "path": "<optional path to the debugger>",
    "extraCLIOptions": ["<optional array of additional flags passed to the debugger>"]
  },
  "testRunner": {
    "path": "<optional path to the test runner, such as `xctest` on macOS>",
    "extraCLIOptions": ["<optional array of additional flags passed to the test runner>"]
  },
}
```

More types of tools may be enabled in toolset files in the future in addition to those listed above.

Users familiar with CMake can draw an analogy between toolset files and CMake toolchain files. Toolset files are
designed to supplant previous ad-hoc ways of specifying paths and flags in SwiftPM, such as `SWIFT_EXEC` and `CC`
environment variables, which were applied in use cases unrelated to cross-compilation. We propose that
users also should be able to pass `--toolset <path_to_toolset.json>` option to `swift build`, `swift test`, and
`swift run`.

We'd like to allow using multiple toolset files at once. With this users can "assemble" toolchains on the fly out of
tools that in certain scenarios may even come from different vendors. A toolset file can have an arbitrary name, and
each file should be passed with a separate `--toolset` option, i.e. `swift build --toolset t1.json --toolset t2.json`.

All of the properties related to names of the tools are optional, which allows merging configuration from multiple
toolset files. For example, consider `toolset1.json`:

```json5
{
  "schemaVersion": "1.0",
  "swiftCompiler": {
    "path": "/usr/bin/swiftc",
    "extraCLIOptions": ["-Xfrontend", "-enable-cxx-interop"]
  },
  "cCompiler": {
    "path": "/usr/bin/clang",
    "extraCLIOptions": ["-pedantic"]
  }
}
```

and `toolset2.json`:

```json5
{
  "schemaVersion": "1.0",
  "swiftCompiler": {
    "path": "/custom/swiftc"
  }
}
```

With multiple `--toolset` options, passing both of those files will merge them into a single configuration. Tools passed
in subsequent `--toolset` options will shadow tools from previous options with the same names. That is, 
`swift build --toolset toolset1.json --toolset toolset2.json` will build with `/custom/swiftc` and no extra flags, as
specified in `toolset2.json`, but `/usr/bin/clang -pedantic` from `toolset1.json` will still be used.

Tools not specified in any of the supplied toolset files will be looked up in existing implied search paths that are
used without toolsets, even when `rootPath` is present. We'd like toolsets to be explicit in this regard: if a
tool would like to participate in toolset path lookups, it must provide either a relative or an absolute path in a
toolset.

Tools that don't have `path` property but have `extraCLIOptions` present will append options from that property to a
tool with the same name specified in a preceding toolset file. If no other toolset files were provided, these options
will be appended to the default tool invocation. For example `pedanticCCompiler.json` that looks like this

```json5
{
  "schemaVersion": "1.0",
  "cCompiler": {
    "extraCLIOptions": ["-pedantic"]
  }
}
```

in `swift build --toolset pedanticCCompiler.json` will pass `-pedantic` to the C compiler located at a default path.

When cross-compiling, paths in `toolset.json` files supplied in Swift SDK bundles should be self-contained:
no absolute paths and no escaping symlinks are allowed. Users are still able to provide their own `toolset.json` files
outside of artifact bundles to specify additional developer tools for which no relative "non-escaping" path can be
provided within the bundle.

### `swift-sdk.json` Files

Note the presence of `swift-sdk.json` files in each `<Swift SDK artifact>` subdirectory. These files should contain
a JSON dictionary with an evolved version of the schema of [existing `destination.json` files that SwiftPM already
supports](https://github.com/apple/swift-package-manager/pull/1098) and `destination.json` files presented in the pitch
version of this proposal, hence `"schemaVersion": "4.0"`. We'll keep parsing `"version": 1`, `"version": 2`,
and `"version": "3.0"` for backward compatibility, but for consistency with `info.json` this field is renamed to
`"schemaVersion"`. Here's an informally defined schema for these files:

```json5
{
  "schemaVersion": "4.0",
  "targetTriples": {
    "<triple1>": {
      "sdkRootPath": "<a required path relative to `swift-sdk.json` containing SDK root>",
      // all of the properties listed below are optional:
      "swiftResourcesPath": "<a path relative to `swift-sdk.json` containing Swift resources for dynamic linking>",
      "swiftStaticResourcesPath": "<a path relative to `swift-sdk.json` containing Swift resources for static linking>",
      "includeSearchPaths": ["<array of paths relative to `swift-sdk.json` containing headers>"],
      "librarySearchPaths": ["<array of paths relative to `swift-sdk.json` containing libraries>"],
      "toolsetPaths": ["<array of paths relative to `swift-sdk.json` containing toolset files>"]
    },
    // a Swift SDK can support more than one target triple:
    "<triple2>": {
      "sdkRootPath": "<a required path relative to `swift-sdk.json` containing SDK root>",
      // all of the properties listed below are optional:
      "swiftResourcesPath": "<a path relative to `swift-sdk.json` containing Swift resources for dynamic linking>",
      "swiftStaticResourcesPath": "<a path relative to `swift-sdk.json` containing Swift resources for static linking>",
      "includeSearchPaths": ["<array of paths relative to `swift-sdk.json` containing headers>"],
      "librarySearchPaths": ["<array of paths relative to `swift-sdk.json` containing libraries>"],
      "toolsetPaths": ["<array of paths relative to `swift-sdk.json` containing toolset files>"]
    }
    // more triples can be supported by a single Swift SDK if needed, primarily for sharing files between them.
  }
}
```

We propose that all relative paths in `swift-sdk.json` files should be validated not to "escape" the Swift SDK
bundle for security reasons, in the same way that `toolset.json` files are validated when contained in Swift SDK
bundles. That is, `../` components, if present in paths, will not be allowed to reference files and
directories outside of a corresponding Swift SDK bundle. Symlinks will also be validated to prevent them from escaping
out of the bundle.
 
If `sdkRootPath` is specified and `swiftResourcesPath` is not, the latter is inferred to be
`"\(sdkRootPath)/usr/lib/swift"` when linking the Swift standard library dynamically, `"swiftStaticResourcesPath"` is
inferred to be `"\(sdkRootPath)/usr/lib/swift_static"` when linking it statically. Similarly, `includeSearchPaths` is
inferred as `["\(sdkRootPath)/usr/include"]`, `librarySearchPaths` as  `["\(sdkRootPath)/usr/lib"]`.

Here's `swift-sdk.json` file for the `ubuntu_jammy` artifact previously introduced as an example:

```json5
{
  "schemaVersion": "4.0",
  "targetTriples": {
    "aarch64-unknown-linux-gnu": {
      "sdkRootPath": "aarch64-unknown-linux-gnu/ubuntu-jammy.sdk",
      "toolsetPaths": ["aarch64-unknown-linux-gnu/toolset.json"]
    },
    "x86_64-unknown-linux-gnu": {
      "sdkRootPath": "x86_64-unknown-linux-gnu/ubuntu-jammy.sdk",
      "toolsetPaths": ["x86_64-unknown-linux-gnu/toolset.json"]
    }
  }
}
```

Since not all platforms can support self-contained Swift SDK bundles, users will be able to provide their own
additional paths on the filesystem outside of bundles after a Swift SDK is installed. The exact options for specifying
paths are proposed in a subsequent section for a newly introduced `swift sdk configure` command.

### Swift SDK Installation and Configuration

To manage Swift SDKs, we'd like to introduce a new `swift sdk` command with three subcommands:

- `swift sdk install <bundle URL or local filesystem path>`, which downloads a given bundle if needed and
  installs it in a location discoverable by SwiftPM. For Swift SDKs installed from remote URLs an additional
  `--checksum` option is required, through which users of a Swift SDK can specify a checksum provided by a publisher of
  the SDK. The latter can produce a checksum by running `swift package compute-checksum` command (introduced in
  [SE-0272](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0272-swiftpm-binary-dependencies.md)) with the
  Swift SDK bundle archive as an argument.
  
  If a Swift SDK with a given artifact ID has already been installed and its version is equal or higher to a version
  of a new Swift SDK, an error message will be printed. If the new version is higher, users should invoke the
  `install` subcommand with `--update` flag to allow updating an already installed Swift SDK artifact to a new
  version.
- `swift sdk list`, which prints a list of already installed Swift SDKs with their identifiers.
- `swift sdk configure <identifier> <target-triple>`, which allows users to provide additional search paths and toolsets to be
used subsequently when building with a given Swift SDK. Specifically, multiple `--swift-resources-path`,
`--include-search-path`, `--library-search-path`, and `--toolset` options with corresponding paths can be provided,
which then will be stored as configuration for this Swift SDK. 
`swift sdk configure <identifier> --show-configuration` will print currently set paths, while
`swift sdk configure <identifier> --reset` will reset all of those at once.
- `swift sdk remove <identifier>` will remove a given Swift SDK from the filesystem.

### Using a Swift SDK

After a Swift SDK is installed, users can refer to it via its identifier passed to the `--swift-sdk` option, e.g.

```
swift build --swift-sdk ubuntu_focal
```

We'd also like to make `--swift-sdk` option flexible enough to recognize target triples when there's only a single
Swift SDK installed for such triple:

```
swift build --swift-sdk x86_64-unknown-linux-gnu
```

When multiple Swift SDKs support the same triple, an error message will be printed listing these Swift SDKs and
asking the user to select a single one via its identifier instead.

### Swift SDK Bundle Generation

Swift SDKs can be generated quite differently, depending on host and target triple combinations and user's
needs. We intentionally don't specify in this proposal how exactly Swift SDK bundles should be generated.

Authors of this document intend to publish source code for a macOS → Linux Swift SDK generator, which community is
welcome to fork and reuse for their specific needs. As a configurable option, this generator will use Docker for setting
up the build environment locally before copying it to a Swift SDK tree. Relying on Docker in this generator makes it
easier to reuse and customize existing build environments. Important to clarify, that Docker is only used for bundle
generation, and users of Swift SDK bundles do not need to have Docker installed on their machine to use these bundles.

As an example, Swift SDK publishers looking to add a library to an Ubuntu 22.04 target environment would modify a
`Dockerfile` similar to this one in their Swift SDK generator source code:

```dockerfile
FROM swift:5.9-jammy

apt-get install -y \
  # PostgreSQL library provided as an example.
  libpq-dev
  # Add more libraries as arguments to `apt-get install`.
```

Then to generate a new Swift SDK, a generator executable delegates to Docker for downloading and installing
required tools and libraries, including the newly added ones. After a Docker image with Swift SDK environment is
ready, the generator copies files from the image to a corresponding `.artifactbundle` Swift SDK tree.

## Security

The proposed `--checksum` flag provides basic means of verifying Swift SDK bundle's validity. As a future direction,
we'd like to consider sandboxed and codesigned toolchains included in Swift SDKs running on macOS.

## Impact on Existing Packages

This is an additive change with no impact on existing packages.

## Prior Art

### Rust

In the Rust ecosystem, its toolchain and standard library built for a target triple are managed by [the `rustup`
tool](https://github.com/rust-lang/rustup). For example, artifacts required for cross-compilation to
`aarch64-linux-unknown-gnu` are installed with
[`rustup target add aarch64-linux-unknown-gnu`](https://rust-lang.github.io/rustup/cross-compilation.html). Then
building for this target with Rust’s package manager looks like `cargo build --target=aarch64-linux-unknown-gnu` .

Mainstream Rust tools don’t provide an easy way to create your own targets. You’re only limited to the list
of targets provided by Rust maintainers. This likely isn’t a big problem per se for Rust users, as Rust doesn’t provide
C/C++ interop on the same level as Swift. It means that Rust packages much more rarely than Swift expect certain
system-provided packages to be available in the same way that SwiftPM allows with `systemLibrary`.

Currently, Rust doesn’t supply all of the required tools when running `rustup target add`. It’s left to a user to
specify paths to a linker that’s suitable for their host/target triple combination manually in a config file. We
feel that this should be unnecessary, which is why Swift SDK bundles proposed for Swift can provide their own tools
via toolset configuration files.

### Go

Go’s standard library is famously self-contained and has no dependencies on C or C++ standard libraries. Because of this
there’s no need to install artifacts. Cross-compiling in Go works out of the box by passing `GOARCH` and `GOOS`
environment variables with chosen values, an example of this is `GOARCH=arm64 GOOS=linux go build` invocation.

This would be a great experience for Swift, but it isn’t easily achievable as long as Swift standard library depends on
C and C++ standard libraries. Any code interoperating with C and/or C++ would have to link with those libraries as well.
When compared to Go, our proposed solution allows both dynamic and, at least on Linux when Musl is supported, full
static linking. We’d like Swift to allow as much customization as needed for users to prepare their own Swift SDK
bundles.

## Alternatives Considered

### Extensions Other Than `.artifactbundle`

Some members of the community suggested that Swift SDK bundles should use a more specific filepath extension. Since
we're relying on the existing `.artifactbundle` format and extension, which is already used for binary targets, we think
a specialized extension only for Swift SDKs would introduce an inconsistency. On the other hand, we think that
specific extensions could make sense with a change applied at once. For example, we could consider `.binarytarget` and
`.swiftsdk` extensions for respective artifact types. But that would require a migration strategy for existing
`.artifactbundle`s containing binary targets.

### Building Applications in Docker Containers

Instead of coming up with a specialized bundle format for Swift SDKs, users of Swift on macOS building for Linux could
continue to use Docker. But, as discussed in the [Motivation](#motivation) section, building applications in Docker
doesn’t cover all of the possible use cases and complicates onboarding for new users. It also only supports Linux, while
we’re looking for a solution that can be generalized for all possible platforms.

### Alternative Bundle Formats

One alternative is to allow only a single host/target combination per bundle, but this may complicate
distribution of Swift SDK bundles in some scenarios. The existing `.artifactbundle` format is flexible enough to
support bundles with a single or multiple combinations.

Different formats of Swift SDK bundles can be considered, but we don't think those would be significantly different
from the proposed one. If they were different, this would complicate bundle distribution scenarios for users who want to
publish their own artifact bundles with executables, as defined in SE-0305.

### Triples nomenclature

Authors of the proposal considered alternative nomenclature to the established "build/host/target platform" naming convention,
but felt that preserving consistency with other ecosystems is more important. 

While "target" already has a different meaning within the build systems nomenclature, users are most likely to stumble upon
targets when working with SwiftPM package manifests. To avoid this ambiguity, as a future direction SwiftPM can consider renaming
`target` declarations used in `Package.swift` to a different unambiguous term.

### Making Swift SDK Bundles Fully Self-Contained

Some users expressed interest in self-contained Swift SDK bundles that ignore the value of `PATH` environment variable
and prevent launching any executables from outside of a bundle. So far in our practice we haven't seen any problems
caused by the use of executables from `PATH`. Quite the opposite, we think most Swift SDKs would want to reuse as many
tools from `PATH` as possible, which would allow making Swift SDK bundles much smaller. For example as of Swift 5.7,
on macOS `clang-13` binary takes ~360 MB, `clangd` ~150 MB, and `swift-frontend` ~420 MB. Keeping copies of these
binaries in every Swift SDK bundle seems quite redundant when existing binaries from `PATH` can be easily reused.
Additionally, we find that preventing tools from being launched from arbitrary paths can't be technically enforced
without sandboxing, and there's no cross-platform sandboxing solution available for SwiftPM. Until such sandboxing
solution is available, we'd like to keep the existing approach where setting `PATH` environment variable behaves in a
predictable way and is consistent with established CLI conventions.

## Future Directions

### Identifying Platforms with Dictionaries of Properties

Platform triples are not specific enough in certain cases. For example, `aarch64-unknown-linux` host triple can’t
prevent a user from installing a Swift SDK bundle on an unsupported Linux distribution. In the future we could
deprecate `supportedTriples` and `targetTriples` JSON properties in favor of dictionaries with keys and values that
describe aspects of platforms that are important for Swift SDKs. Such dictionaries could look like this:

```json5
"platform": {
  "kernel": "Linux",
  "libcFlavor": "Glibc",
  "libcMinVersion": "2.36",
  "cpuArchitecture": "aarch64"
  // more platform capabilities defined here...
}
```

A toolchain providing this information could allow users to refer to these properties in their code for conditional
compilation and potentially even runtime checks.

### SwiftPM Plugins for Remote Running, Testing, Deployment, and Debugging

After an application is built with a Swift SDK, there are other development workflow steps to be improved. We could
introduce new types of plugins invoked by `swift run` and `swift test` for purposes of remote running, debugging, and
testing. For Linux target triples, these plugins could delegate to Docker for running produced executables.

### `swift sdk select` Subcommand

While `swift sdk select` subcommand or a similar one make sense for selecting a Swift SDK instead of
passing `--swift-sdk` to `swift build` every time, users will expect `swift run` and `swift test` to also work for any
Swift SDK previously passed to `swift sdk select`. That’s out of scope for this proposal on its own and
depends on making plugins (from the previous subsection) or some other remote running and testing implementation to
fully work.

### SwiftPM and SourceKit-LSP Improvements

It is a known issue that SwiftPM can’t run multiple concurrent builds for different target triples. This may cause
issues when SourceKit-LSP is building a project for indexing purposes (for a host platform by default), while a user may
be trying to build for a target for testing, for example. One of these build processes will fail due to the
process locking the build database. A potential solution would be to maintain separate build databases per platform.

Another issue related to SourceKit-LSP is that [it always build and indexes source code for the host
platform](https://github.com/apple/sourcekit-lsp/issues/601). Ideally, we want it to maintain indices for multiple
platforms at the same time. Users should be able to select target triples and corresponding indices to enable
semantic syntax highlighting, auto-complete, and other features for areas of code that are conditionally compiled with
`#if` directives.

### Source-Based Swift SDKs

One interesting solution is distributing source code of a minimal base SDK, as explored by [Zig programming
language](https://andrewkelley.me/post/zig-cc-powerful-drop-in-replacement-gcc-clang.html). In this scenario, Swift SDK
binaries are produced on the fly when needed. We don't consider this option to be mutually exclusive with solutions
proposed in this document, and so it could be explored in the future for Swift as well. However, this requires reducing
the number of dependencies that Swift runtime and core libraries have.

### Swift SDK Bundles and Package Registries

Since `info.json` manifest files contained within bundles contain versions, it would make sense to host Swift SDK
bundles at package registries. Although, it remains to be seen whether it makes sense for an arbitrary SwiftPM package
to specify a Swift SDK bundle within its list of dependencies.
