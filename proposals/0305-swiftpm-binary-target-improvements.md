# Package Manager Binary Target Improvements

* Proposal: [SE-0305](0305-swiftpm-binary-target-improvements.md)
* Authors: [Anders Bertelrud](https://github.com/abertelrud), [Tom Doron](https://github.com/tomerd)
* Review Manager: [Tom Doron](https://github.com/tomerd)
* Status: **Implemented (Swift 5.6)**
* Decision Notes: [Acceptance](https://forums.swift.org/t/accepted-se-0305-package-manager-binary-target-improvements/47742) 
* Previous Revision: [1](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0305-swiftpm-binary-target-improvements.md)
* Forum Discussion: [SE-0305: Package Manager Binary Target Improvements](https://forums.swift.org/t/se-0305-package-manager-binary-target-improvements/45589)
* Review: [1](https://forums.swift.org/t/se-0305-package-manager-binary-target-improvements/) [2](https://forums.swift.org/t/se-0305-2nd-review-package-manager-binary-target-improvements/)
* Implementation: Available in [recent `main` snapshots](https://swift.org/download/#snapshots)

## Introduction

This proposal extends SwiftPM binary targets to also support other kinds of prebuilt artifacts, such as command line tools. It does not in and of itself add support for non-Darwin binary libraries, although the proposed improvements could be a step towards such support.

## Motivation

The Swift Package Manager’s [`binaryTarget` type](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0272-swiftpm-binary-dependencies.md) lets packages vend libraries that either cannot be built in Swift Package Manager for technical reasons, or for which the source code cannot be published for legal or other reasons.

In the current version of SwiftPM, binary targets only support libraries in an Xcode-oriented format called *XCFramework*, and only for Apple platforms.

As part of [SE-0303 SwiftPM Extensible Build Tools](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0303-swiftpm-extensible-build-tools.md), SwiftPM will need a way to allow packages to vend prebuilt binaries containing command line tools that can be invoked during the build. This is because:

* many popular command line tools (such as `protoc`) do not build using SwiftPM, and
* tools that should run during “prebuild” (i.e. before the build starts) cannot themselves be built as part of the build

There are many other reasons to extend binary targets in SwiftPM, such as to support binary libraries for non-Apple platforms. While this proposal is specifically focused on allowing the vending of command line tools and related kinds of artifacts, it tries to do so with an eye toward making binary targets more flexible and cross-platform in the future.

## Proposed solution

This proposal extends binary targets to allow a new “artifact bundle” format in addition to the XCFramework format that is supported in the current version of SwiftPM. This proposal does not change how `binaryTarget`s are declared in package manifests, nor how they are used for XCFrameworks. Instead, this proposal builds on the existing support for binary targets.

In this proposal, *artifact bundles* are directory structures that can contain multiple *artifacts*, each having a unique identifier within the bundle. Each artifact consists of a set of variants that support various architectures and platforms. An artifact bundle also contains a manifest file that describes the artifacts and their variants.

Prior to this proposal, a binary target could reference either:

* a remote `.zip` file with an XCFramework directory at its top level, or
* less commonly, a plain XCFramework directory embedded inside the package directory

This proposal extends each of these cases to alternatively support an artifact bundle in place of the XCFramework (a single binary target cannot contain both an XCFramework and an artifact bundle).

In addition to allowing a URL reference to a `.zip` file containing a single artifact bundle, this proposal allows the remote URL to refer to an *artifact bundle index* file that in turn refers to multiple `.zip` files, each containing the artifact bundle for a variant or a set of variants of the same conceptual artifact. This optimization allows SwiftPM to download only the variants it needs.

The artifact index file maps a variant selector to the `.zip` file that contains the appropriate variant of the artifact bundle. The *Detailed design* section below provides more information about the variant selector and how the appropriate variant is chosen.

Once a `.zip` file has been selected using the index file, it is downloaded in the same way as in the case of a single `.zip` file (which is exactly the same as for all binary targets today). It is unarchived to a location in the local file system, and the path of the artifact bundle becomes available to the rest of SwiftPM.

Regardless of whether the artifact bundle is local or remote, and whether or not it uses an index file, the appropriate variants of the artifacts declared in an artifact bundle will be made available to any package plugins that have a target dependency on the `binaryTarget`. This is done through API in the `PackagePlugin` library as described in [SE-0303](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0303-swiftpm-extensible-build-tools.md).

## Detailed design

This section describes *artifact bundles*, *artifact bundle manifests*, and *artifact bundle indices* in detail.

### Artifact bundle

The proposal defines the structure and semantics of an *artifact bundle* to be a directory that has the filename suffix `.artifactbundle` and which has the following content:

* a set of one or more *artifacts*, each with an identifier string that is unique within the bundle
* within each artifact, a set of *variants*, each with an identifier string that is unique with the artifact
* a manifest file containing information about the individual artifacts and their variants

Artifact bundles may appear as the referent of the `path` parameter of a `binaryTarget`, or as the sole top-level entity in a `.zip` file referenced by either a `url` parameter in the package manifest or through an artifact bundle index file (as described below).

The structure of the artifact bundle is:

```
<name>.artifactbundle
 ├ info.json
 ├ <artifact>
 │  ├ <variant>
 │  │  ├ <executable>
 │  │  └ <other files>
 │  └ <variant>
 │     ├ <executable>
 │     └ <other files>
 ├ <artifact>
 │  └ <variant>
 │     ├ <executable>
 │     └ <other files>
 │ <artifact>
 ┆  └┄
```

The manifest is always at the top level and is named `info.json`. Its contents are described in the next subsection.

At the top level of the artifact bundle directory is a subdirectory for each artifact in the bundle. The names of the artifacts are arbitrary, but must be unique within the bundle. These are the names that are used in the plugin API when looking up a binary artifact. A plugin has access to the artifacts defined in the artifact bundles specified by all the binary targets on which it has declared a dependency.

Within each artifact directory are variant directories. As with the names of artifacts, these names are arbitrary but must be unique within the artifact directory.

All name lookup is case-sensitive. In order to be resilient on case-insensitive file systems, artifact bundles should avoid using pairs of names that differ only in case.

Each artifact variant directory is the root of a file system hierarchy that can be made available to a plugin. The variant that is made available depends on the target triple of the host toolchain at the time of use.

### Artifact bundle manifest

The artifact bundle manifest is a JSON file named `info.json` at the top level of the artifact bundle. It has the following contents:

```json
{
    "schemaVersion": "1.0",
    "artifacts": {
        "<identifier>": {
            "version": "<version number>",
            "type": "executable",
            "variants": [
                {
                    "path": "<relative-path-to-executable>",
                    "supportedTriples": [ "<triple1>", ... ]
                },
                ...
            ]
        },
        ...
    }
}
```

The top level of the artifact bundle manifest contains:

* `schemaVersion` — in this proposal `1.0`; this allows changes to the format in the future
* `artifacts` — a mapping of artifact identifiers to artifact dictionaries

Each artifact dictionary contains:

* `version` — an arbitrary version number for informational purposes (available to plugins)
* `type` — in this proposal always `executable`; this allows further support for other types of artifacts in the future
* `variants` — an array of variant dictionaries

Each variant dictionary contains:

* `path` — the subpath of the command line executable (relative to the bundle)
* `supportedTriples` — array of target triples supported by this variant

Note that although the `type` key is always `executable` in this proposal, this is expected to allow the support for artifact bundles to be extended in future proposals to support libraries, resource sets, and other types of binary artifacts.

This proposal uses [target triples](https://clang.llvm.org/docs/CrossCompilation.html#target-triple) as the variant selectors. A single variant may support more than one target triple, as in the case of universal binaries.

### How artifact bundles are processed

As with binary targets today, after downloading the `.zip` file and validating its integrity using the checksum specified in the manifest, SwiftPM unarchives the contents of the `.zip` into an intermediate location in the local file system. If the archive does not contain a `.xcframework`, SwiftPM will look for a `.artifactbundle` directory instead. It is an error for both an `.artifactbundle` and a `.xcframework` to be present in the same `.zip`.

If SwiftPM finds a `.artifactbundle` directory, it will try to load the `info.json` within it. If the `schemaVersion` is not recognized, then SwiftPM will emit an error and not process the artifact bundle further. Otherwise it will register the artifacts present in the artifact bundle, for later use by plugins.

When a plugin asks for a tool with a particular identifier, SwiftPM will consider the artifact bundles specified by any binary targets on which the plugin depends. The artifacts defined in these bundles will be made available to the plugin and will be translated to the paths at which the executables have been unarchived. The plugin can access any support files provided with the executable in the same way.

Any lookup of names within the artifact bundle is case sensitive.

### Artifact bundle index

To avoid downloading files that will not be needed, an artifact bundle can be split up into multiple bundles. Each of these bundles has the same format as any other artifact bundle, but contains only a subset of the variants.

An example could include having one bundle for Apple platforms, another for Windows, and others for different Linux variants.

An artifact bundle index is a JSON file with a `.artifactbundleindex` extension, and that has the following contents:

```json
{
    "schemaVersion": "1.0",
    "bundles": [
        {
            "fileName": "<name of .zip file containing bundle>",
            "checksum": "<checksum of .zip file>",
            "supportedTriples": [ "<triple1>", ... ]
        },
        ...
    ]
}
```

The top level of the artifact bundle index contains:

* `schemaVersion` — in this proposal `1.0`; this allows changes to the format in the future
* `bundles` — a list of `.zip` files that contain bundles containing subsets of variants of the same conceptual artifacts

Each bundle dictionary contains:

* `fileName` — the filename of the `.zip` archive containing the artifact bundle
* `checksum` — the checksum of the `.zip` archive, computed using `swift` `package` `checksum`
* `supportedTriples` — array of all the target triples supported by the variants in the artifact bundle

The individual `.zip` files are expected to be located next to the `.artifactbundleindex` file, and thus only their filenames are listed in the index.

The checksum for each `.zip` file in the index is computed in the same manner as for other binary `.zip` files, i.e. using `swift package compute-checksum`. The checksum in the binary target that references the `.artifactbundleindex` is the checksum of the `.artifactbundleindex` file itself. In this way, SwiftPM can validate the integrity of any of the `.zip` archives referenced by the index file.

## Example

Here is a hypothetical example of how the Protobuf compiler (`protoc`) could be vended as an artifact bundle:

```
protoc.artifactbundle
├── info.json
├── protoc-3.15.6-linux-gnu
│   ├── bin
│   │   └── protoc
│   └── include
│       └── etc.proto
├── protoc-3.15.6-macos
│   ├── bin
│   │   └── protoc
│   └── include
│       └── etc.proto
└── protoc-3.15.6-windows
    ├── bin
    │   └── protoc.exe
    └── include
        └── etc.proto
```

The contents of the `info.json` manifest would be:

```json
{
    "schemaVersion": "1.0",
    "artifacts": {
        "protoc": {
            "type": "executable",
            "version": "3.15.6",
            "variants": [
                {
                    "path": "protoc-3.15.6-linux-gnu/bin/protoc",
                    "supportedTriples": ["x86_64-unknown-linux-gnu"]
                },
                {
                    "path": "protoc-3.15.6-macos/bin/protoc",
                    "supportedTriples": ["x86_64-apple-macosx", "arm64-apple-macosx"]
                },
                {
                    "path": "protoc-3.15.6-windows/bin/protoc.exe",
                    "supportedTriples": ["x86_64-unknown-windows"]
                },
            ]
        }
    }
}
```

In this hypothetical case, the `macos` variant supports both `x86_64` and `arm64`.

## Security considerations

The same checksum facility that binary targets already use will ensure that any downloaded `.zip` file will have the intended contents, exactly as for XCFrameworks. There is only a small conceptual difference between running a command during the build vs linking it into the built debug binary for a package using an XCFramework. Either way, the remote code will be run on the local machine, which has inherent security implications if the source is untrusted.

## Impact on existing packages

Artifact bundles would only be available for packages that specify the SwiftPM tools version in which this proposal is implemented. There will be no impact on existing packages.

## Future directions

### Binary compatibility for linux

This initial proposal leaves unanswered some questions about binary compatibility, especially with regards to Linux. For Apple platforms, binary distribution is fairly straightforward, owing to of an ABI-compatible set of SDKs and a strict versioning scheme.

For Linux, this is much more of a problem. Linux is not a single platform, and it is difficult to the variants in a way that will allow binary compatibility without having to provide an excessive number of specialized binaries.

A future direction would be to adopt the concepts from [manylinux](https://www.python.org/dev/peps/pep-0513) and provide tooling to let package authors build Linux binaries in a way that makes them usable across many Linux installations. This would, among other things, require tools to statically link against any non-ABI-stable dependencies.

### Support for executable scripts

The proposed support for executables is mainly focused on compiled executables, where it makes sense to use a target triple to represent architectures and ABI requirements. For executables implemented as shell scripts (or other kinds of scripts), it might make sense to extend the notion of variant selectors to allow the cross-platform nature of scripts to be better represented. For example, it might be reasonable to allow a `*` for the architecture if there are no architecture contstraints.

### Libraries on non-Darwin platforms

Generalizing binary targets to support arbitrary artifacts moves SwiftPM closer to supporting binary libraries other than XCFrameworks. In order to make this usable, however, a future proposal would need to define exactly how to provide libraries on Windows and Linux, and this would encounter even more ABI compatibility issues than executables, since the workaround of statically linking troublesome dependencies would not be available to libraries that themselves need to be linked into the client.

### Arbitrary binary artifacts

This proposal focuses on executables, since that is the immediate need in order to support [SE-0303](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0303-swiftpm-extensible-build-tools.md). However, a future direction would be to allow distribution of libraries of resources such as 3D models, textures, fonts, or other large assets that a package may want to make available but not include in the package repository itself. The proposed `.artifactbundle` format is flexible enough to handle this, but there would need to be an API for plugins to access those artifacts, and possibly to vend them directly to client packages if no separate processing is necessary (for example in the form of resource bundles, which is a concept that SwiftPM already has).

## Alternatives considered

One alternative would be to not extend binary targets and to instead require any executables that are needed by plugins to be installed on the host before building the package. However, one of the goals of packages are to be self-describing, and for SwiftPM to be able to fetch any dependencies as needed. This should include binary executables.

## References

* https://github.com/swiftlang/swift-evolution/blob/main/proposals/0272-swiftpm-binary-dependencies.md
* https://github.com/swiftlang/swift-evolution/blob/main/proposals/0303-swiftpm-extensible-build-tools.md
* https://www.python.org/dev/peps/pep-0513

