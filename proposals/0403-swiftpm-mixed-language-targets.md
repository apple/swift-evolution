# Package Manager Mixed Language Target Support

* Proposal: [SE-0403](0403-swiftpm-mixed-language-targets.md)
* Authors: [Nick Cooke](https://github.com/ncooke3)
* Review Manager: [Saleem Abdulrasool](https://github.com/compnerd)
* Status: **Returned for Revision**
* Implementation: [apple/swift-package-manager#5919](https://github.com/apple/swift-package-manager/pull/5919)
* Review: ([pitch](https://forums.swift.org/t/61564)), ([review](https://forums.swift.org/t/66202)), ([returned for revision](https://forums.swift.org/t/66975))

## Introduction

This is a proposal for adding package manager support for targets containing
both Swift and [C based language sources][SE-0038] (henceforth, referred to as
mixed language sources). Currently, a target’s source can be either Swift or a
C based language ([SE-0038]), but not both.

Swift-evolution thread: [Discussion thread topic for that
proposal](https://forums.swift.org/)

## Motivation

This proposal enables Swift Package Manager support for multi-language targets.

Packages may need to contain mixed language sources for both legacy or
technical reasons. For developers building or maintaining packages with mixed
languages (e.g. Swift and Objective-C), there are two workarounds for doing so
with Swift Package Manager, but they have drawbacks that degrade the developer
experience, and sometimes are not even an option:
- Distribute binary frameworks via binary targets. Drawbacks include that the
  package will be less portable as it can only support platforms that the
  binaries support, binary dependencies are only available on Apple platforms,
  customers cannot view or easily debug the source in their project workspace,
  and tooling is required to generate the binaries for release.
- Separate a target’s implementation into sub-targets based on language type,
  adding dependencies where necessary. For example, a target `Foo` may have
  Swift-only sources that can call into an underlying target `FooObjc` that
  contains Clang-only sources. Drawbacks include needing to depend on the
  public API surfaces between the targets, increasing the complexity of the
  package’s manifest and organization for both maintainers and clients, and
  preventing package developers from incrementally migrating internal
  implementation from one language to another (e.g. Objective-C to Swift) since
  there is still a separation across targets based on language.

Package manager support for mixed language targets addresses both of the above
drawbacks by enabling developers to mix sources of supported languages within a
single target without complicating their package’s structure or developer
experience.

## Proposed solution

Package authors can create a mixed target by mixing language sources in their
target's source directory. When mixing some languages, like C++, authors have
the option of opting in to advanced interoperability features by configuring
the target with an interoperability mode [`SwiftSetting.InteroperabilityMode`].

When building a mixed language target, the package manager will build the
public API into a single module for use by clients.

At a high level, the build process is split into two parts based on
the language of the sources. The Swift sources are built by the Swift compiler
and the C/Objective-C/C++ sources are built by the Clang compiler.

1. The Swift compiler is made aware of the Clang part of the package when
   building the Swift sources into a `swiftmodule`.
1. The Clang part of the package is built with knowledge of the
   interoperability Swift header. The contents of this header will vary
   depending on if/what language-specific interoperability mode is configured
   on the target. The interoperability header is modularized as part of the
   mixed target's public interface.


The [following example][mixed-package] defines a package containing mixed
language sources.

```
MixedPackage
├── Package.swift
├── Sources
│   └── MixedPackage
│       ├── Jedi.swift          ⎤-- Swift sources
│       ├── Lightsaber.swift    ⎦
│       ├── Sith.m              ⎤-- Implementations & internal headers
│       ├── SithRegistry.h      ⎟
│       ├── SithRegistry.m      ⎟
│       ├── droid_debug.c       ⎦
│       ├── hello_there.txt     ]-- Resources
│       └── include             ⎤-- Public headers
│           ├── MixedPackage.h  ⎟
│           ├── Sith.h          ⎟
│           └── droid_debug.h   ⎦
└── Tests
    └── MixedPackageTests
        ├── JediTests.swift          ]-- Swift tests
        ├── SithTests.m              ]-- Objective-C tests
        ├── ObjcTestConstants.h      ⎤-- Mixed language test utils
        ├── ObjcTestConstants.m      ⎟
        └── SwiftTestConstants.swift ⎦
```

The proposed solution would enable the above targets to do the following:
1. Export their public API, if any, from across the mixed language sources.
1. Use C/Objective-C/C++ compatible Swift API from target’s Swift sources
   within the target’s C/Objective-C/C++ sources.
1. Use Swift compatible C/Objective-C/C++ API from target’s C/Objective-C/C++
   sources within the target’s Swift sources.
1. Access target resources from Swift and Objective-C contexts.

### Limitations

Initial support for targets containing mixed language sources will have the
following limitations:
1. The target must be either a library or test target. Support for other types
   of targets is deferred until the use cases become clear.
1. If the target contains a custom module map, it cannot contain a submodule of
   the form `$(ModuleName).Swift`. This is because the package manager will
   synthesize an _extended_ module map that includes a submodule that
   modularizes the generated Swift interop header.

### Importing a mixed target

Mixed targets can be imported into a client target in several ways. The
following examples will reference `MixedPackage`, a package containing mixed
language target(s).

#### Importing within a **Swift** context

The public API of a mixed target, `MixedPackage`, can be imported into a
**Swift** file via an `import` statement:

```swift
// MyClientTarget.swift

import MixedPackage
```

Testing targets can import the mixed target via
`@testable import MixedPackage`. As expected, this will expose internal Swift
types within the module. It will not expose any non-public C language types.

#### Importing within an **C/Objective-C/C++** context

How a mixed target, `MixedPackage`, is imported into an **C/Objective-C/C++**
file will vary depending on the language it is being imported in.

When Clang modules are supported, clients can import the module. Textual
imports are also an option.


For this example, consider `MixedPackage` being organized as such:

```
MixedPackage
├── Package.swift
└── Sources
     ├── NewCar.swift
     └── include                  ]-- Public headers directory
        ├── OldCar.h
        └── MixedPackage-Swift.h  ]-- This header is generated
                                      during the build.
```


Like Clang targets, `MixedPackage`'s public headers directory (`include` in the
above example) is added a header search path to client targets. The following
example demonstrates all the possible public headers that can be imported from
`MixedPackage`.

```objc
// MyClientTarget.m

// If module imports are supported, the public API (including API in the
// generated Swift header) can be imported via a module import.
@import MixedPackage;
// Imports types defined in `OldCar.h`.
#import "OldCar.h"
// Imports Objective-C compatible Swift types defined in `MixedPackage`.
#import "MixedPackage-Swift.h"
```

## Plugin Support

Package manager plugins should be able to process mixed language source
targets. The following type will be added to the `PackagePlugin` module
to represent a mixed language target in a plugin's context.

This API was created by joining together the properties of the existing
`SwiftSourceModuleTarget` and `ClangSourceModuleTarget` types
([source][Swift-Clang-SourceModuleTarget]).

```swift
/// Represents a target consisting of a source code module compiled using both the Clang and Swift compiler.
public struct MixedSourceModuleTarget: SourceModuleTarget {
    /// Unique identifier for the target.
    public let id: ID

    /// The name of the target, as defined in the package manifest. This name
    /// is unique among the targets of the package in which it is defined.
    public let name: String

    /// The kind of module, describing whether it contains unit tests, contains
    /// the main entry point of an executable, or neither.
    public let kind: ModuleKind

    /// The absolute path of the target directory in the local file system.
    public let directory: Path

    /// Any other targets on which this target depends, in the same order as
    /// they are specified in the package manifest. Conditional dependencies
    /// that do not apply have already been filtered out.
    public let dependencies: [TargetDependency]

    /// The name of the module produced by the target (derived from the target
    /// name, though future SwiftPM versions may allow this to be customized).
    public let moduleName: String

    /// The source files that are associated with this target (any files that
    /// have been excluded in the manifest have already been filtered out).
    public let sourceFiles: FileList

    /// Any custom compilation conditions specified for the target's Swift sources.
    public let swiftCompilationConditions: [String]

    /// Any preprocessor definitions specified for the target's Clang sources.
    public let clangPreprocessorDefinitions: [String]

    /// Any custom header search paths specified for the Clang target.
    public let headerSearchPaths: [String]

    /// The directory containing public C headers, if applicable. This will
    /// only be set for targets that have a directory of a public headers.
    public let publicHeadersDirectory: Path?

    /// Any custom linked libraries required by the module, as specified in the
    /// package manifest.
    public let linkedLibraries: [String]

    /// Any custom linked frameworks required by the module, as specified in the
    /// package manifest.
    public let linkedFrameworks: [String]
}
```


## Detailed design

### Modeling a mixed language target

Up until this proposal, when a package was loading, each target was represented
programmatically as either a [`SwiftTarget`] or [`ClangTarget`]. Which of these
types to use was informed by the sources found in the target. For targets with
mixed language sources, an error was thrown and surfaced to the client. During
the build process, each of those types mapped to another type
([`SwiftTargetBuildDescription`] or [`ClangTargetBuildDescription`]) that
described how the target should be built.

This proposal adds two new types, `MixedTarget` and `MixedTargetDescription`,
that represent targets with mixed language sources during the package loading
and building phases, respectively.

While an implementation detail, it’s worth noting that in this approach, a
`MixedTarget` is a wrapper type around an underlying `SwiftTarget` and
`ClangTarget`. Initializing a `MixedTarget` will internally initialize a
`SwiftTarget` from the given Swift sources and a `ClangTarget` from the given
Clang sources. This extends to the `MixedTargetDescription` type in that it
wraps a `SwiftTargetDescription` and `ClangTargetDescription`.

Using this approach allows for greater code-reuse, and reduces the chance of
introducing a regression from changing existing sub-target types like
`SwiftTarget` and `ClangTarget`.

The role of the `MixedTargetBuildDescription` is to generate auxiliary
artifacts needed for the build and pass specific build flags to the underlying
`SwiftTargetBuildDescription` and `ClangTargetBuildDescription`.

The following diagram shows the relationship between the various types.
```mermaid
flowchart LR
    A>Swift sources] --> B[SwiftTarget] --> C[SwiftTargetBuildDescription]
    D>Clang sources] --> E[ClangTarget] --> F[ClangTargetBuildDescription]

    subgraph MixedTarget
      SwiftTarget
      ClangTarget
    end

    subgraph MixedTargetBuildDescription
      SwiftTargetBuildDescription
      ClangTargetBuildDescription
    end

    G>Mixed sources] --> MixedTarget --> MixedTargetBuildDescription
```

### Building a mixed language target







The Swift part of the target is built before the Clang part. This is because
the C language sources may require resolving a textual import of the generated
interop header, and that header is emitted alongside the Swift module when the
Swift part of the target is built. This relationship is enforced in that the
generated interop header is listed as an input to the compilation commands for
the target’s C language sources. This is specified in the llbuild manifest
(`debug.yaml` in the package's `.build` directory).

##### Additional Swift build flags
The following flags are additionally used when compiling the Swift sub-target:
1. `-import-underlying-module` This flag triggers a partial build of the
   underlying C language sources when building the Swift module. This critical
   flag enables the Swift sources to use C language types defined in the Clang
   part of the target.
1. `-I /path/to/modulemap_dir` The above `-import-underlying-module` flag
   will look for a module map in the given header search path. The module
   map used here cannot modularize the generated interop header as will be
   created from building the Swift sub-target and therefore does not exist
   yet. If a custom module map is provided, the public headers directory
   will be used as that is where the custom module map is enforced to be
   located. It's also enforced that this module map does not expose an
   interop header. If a custom module map is _not_ provided, the package
   manager will pass the target's build directory as that is where a module
   map will be synthesized. This module map will be _un-extended_, in that
   it does not modularize the generated interop header.
1. _If a custom module is NOT provided,_ the package manager will synthesize
   two module maps. One is _extended_ in that it modualrizes the generated
   interop header. The other is _un-extended_ in that it does not modularize
   the generated interop header. A VFS Overlay file is created to swap the
   extended one (named `module.modulemap`) for the unextended one
   (`unextended-module.modulemap`) for the build.
1. `-Xcc -I -Xcc $(TARGET_SRC_PATH)` Adding the target's [path] allows for
   importing headers using paths relative to the root of the target. Because
   passing `-import-underlying-module` triggers a partial build of the Clang
   sources, this is needed for resolving possible header imports.
1. `-Xcc -I -Xcc $(TARGET_PUBLIC_HDRS)` Adding the target's public header's
   path allows for importing headers using paths relative to the public
   header's directory. Because passing `-import-underlying-module` triggers
   a partial build of the Clang sources, this is needed for resolving
   possible header imports.

##### Additional Clang build flags
The following flags are additionally used when compiling the Clang sub-target:
1. `-I $(target’s path)` Adding the target's [path] allows for importing
   headers using paths relative to the root of the target.
1. `-I /path/to/generated_swift_header_dir/` The generated Swift header may be
   needed when compiling the Clang sources.

#### Performing the build

To actually build a package, the package manager creates a llbuild manifest and
passes it to the llbuild system. Adding support for mixed targets involved
modifying [LLBuildManifestBuilder.swift] to convert a
`MixedTargetBuildDescription` into llbuild build nodes.
`MixedTargetBuildDescription` intentionally wraps and configures an underlying
`SwiftTargetBuildDescription` and `ClangTargetBuildDescription`. This means
that creating a llbuild build node for a mixed target is really just creating
build nodes for the its `SwiftTargetBuildDescription` and
`ClangTargetBuildDescription`, respectively.

#### Build artifacts for client targets







##### Module Maps

The client-facing module map’s purpose is to define the public API of the mixed
language module. It has two parts, a primary module declaration and a secondary
submodule declaration. The former of which exposes the public C language
headers and the latter of which exposes the generated interop header.

There are two cases when creating the client-facing module map:
- If a custom module map exists in the target, its contents are copied and
  extended to modularize the generated interop header. These contents are
  written to the build directory as `extended-custom-module.modulemap`.
  Since the public header directory and build directory are passed as import
  paths to the build invocations, a different name is needed for this module
  map as the `-import-underlying-module` should only be able to find one
  `module.modulemap` file from the given import paths.
- Else, the module map’s contents will be generated via the same
  generation rules established in [SE-0038] with an added step to generate the
  `.Swift` submodule. This file is called `module.modulemap` and lives in the
  build directory.

Clients will use an _extended_ module map that includes the modularized interop
header. Building the target will use _unextended_ module map.

> Note: It’s possible that the Clang part of the module exports no public API.
> This could be the case for a target whose public API surface is written in
> Swift but whose implementation is written in Objective-C. In this case, the
> primary module declaration will expose no headers.

Below is an example of a module map for a target that has an umbrella
header in its public headers directory (`include`).

```
// extended-custom-module.modulemap

// This declaration is either copied from the custom module map or generated
// via the rules from SE-0038.
module MixedTarget {
    umbrella header "/Users/crusty/Developer/MixedTarget/Sources/MixedTarget/include/MixedTarget.h"
    export *
}
// This is added on by the package manager as part of this proposal.
module MixedTarget.Swift {
    header "MixedTarget-Swift.h"
    requires objc
}
```

##### all-product-headers.yaml

An `all-product-headers.yaml` VFS overlay file will adjust the public headers
directory to expose the interop header as a relative path, and, if a custom
module map exists, swap it out for the extended one that modualrizes the interop
header.

In either case, it will be passed alongside the module map as a compilation
argument to clients:
```
-fmodule-map-file=/Users/crusty/Developer/MixedTarget/Sources/MixedTarget/include/module.modulemap
-ivfsoverlay /Users/crusty/Developer/MixedTarget/.build/.../MixedTarget.build/Product/all-product-headers.yaml
```

### Additional changes to the package manager

It is the goal for mixed language targets to work on all platforms supported
by the package manager. One obstacle to that is that the package manager,
at the time of this proposal, does not invoke the build system with the
flag needed to emit the interoperability header
([code][should-emit-header]). This limitation is outdated and will be
removed as part of this proposal.

See the related discussion [thread][swift-emit-header-fr] from the initial
formal review.

### Related change to the Swift compiler

When the Swift compiler creates the generated interop header (via
`-emit-objc-header`), any Objective-C symbol referenced in the Swift API that
cannot be forward declared (e.g. superclass, protocol, etc.) will attempt to
be imported via an umbrella header. Since the compiler evaluates
the target as a framework (as opposed to an app), the compiler assumes an
umbrella header exists in a subdirectory (named after the module) within
the public headers directory:
        
    #import <$(ModuleName)/$(ModuleName).h>
        
The compiler assumes that the above path can be resolved relative to the public
header directory. Instead of forcing package authors to structure their
packages around that constraint, the Swift compiler's interop header generation
logic will be ammended to do the following in such cases where the target
does not have the public headers directory structure of an xcframework:

- If an umbrella header that is modularized by the Clang module exists, the
  interop header emit a reference directly to that umbrella header instead.
- Else, the interop header will import all textual includes from the Clang
  module map.

See the related discussion [thread][swift-compiler-thread-fr] from the initial
formal review.

### Mixed language Test Targets

To complement library targets with mixed languages, mixed test targets are
also supported as part of this proposal.

Using the [example package][mixed-package] from before, consider the following
layout of the package's `Tests` directory.

```
MixedPackage
├── ...
└── Tests
    └── MixedPackageTests
        ├── JediTests.swift     ]-- Swift tests
        ├── SithTests.m         ]-- Objective-C tests
        ├── ObjcTestConstants.h ⎤-- Mixed language test utils
        ├── ObjcTestConstants.m ⎟
        └── TestConstants.swift ⎦
```

The types defined in `ObjcTestConstants.h` are visible in `SithTests.m` (via
importing the header).

The Objective-C compatible types defined in `TestConstants.swift` are visible
in `JediTests.swift` (via importing the header).

This design should give package authors flexibility in designing test suites
for their mixed targets.


### Failure cases

There are several failure cases that may surface to end users:
- Attempting to build a mixed target using a tools version that does not
  include this proposal’s implementation.
  ```
  target at '\(path)' contains mixed language source files; feature not supported
  ```
- Attempting to build a mixed target that is neither a library target
  or test target.
  ```
  Target with mixed sources at '\(path)' is a \(type) target; targets
  with mixed language sources are only supported for library and test
  targets.
  ```
- Attempting to build a mixed target containing a custom module map
  that contains a `$(MixedTargetName).Swift` submodule.
  ```
  The target's module map may not contain a Swift submodule for the
  module \(target name).
  ```

## Security

This has no impact on security, safety, or privacy.

## Impact on existing packages

This proposal will not affect the behavior of existing packages. In the
proposed solution, the code path to build a mixed language package is separate
from the existing code paths to build packages with Swift sources and C
Language sources, respectively.

Additionally, this feature will be gated on a tools minor version update, so
mixed language targets building on older toolchains that do not support this
feature will continue to [throw an error][mixed-target-error].

## Future Directions

- Enable package authors to expose non-public headers to their mixed
  target's Swift implemention.
- Extend mixed language target support to currently unsupported types of
  targets (e.g. executables).
- Extend this solution so that all targets are mixed language targets by
  default. This could simplify the implemention as language-specific types
  like `ClangTarget`, `SwiftTarget`, and `MixedTarget` could be consolidated
  into a single type. This approach was avoided in the initial implementation
  of this feature to reduce the risk of introducing a regression.

<!-- Links -->

[SE-0038]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0038-swiftpm-c-language-targets.md

[mixed-package]: https://github.com/ncooke3/MixedPackage

[`SwiftTarget`]: https://github.com/apple/swift-package-manager/blob/ce099264a187759c2f587393bd209d317a0352b4/Sources/PackageModel/Target.swift#L313

[`ClangTarget`]: https://github.com/apple/swift-package-manager/blob/ce099264a187759c2f587393bd209d317a0352b4/Sources/PackageModel/Target.swift#L470

[`SwiftTargetBuildDescription`]: https://github.com/apple/swift-package-manager/blob/main/Sources/Build/BuildPlan.swift#L549

[`ClangTargetBuildDescription`]: https://github.com/apple/swift-package-manager/blob/ce099264a187759c2f587393bd209d317a0352b4/Sources/Build/BuildPlan.swift#L232

[path]: https://developer.apple.com/documentation/packagedescription/target/path

[LLBuildManifestBuilder.swift]: https://github.com/apple/swift-package-manager/blob/14d05ccaa13b768449cd405fff81d630a520e04a/Sources/Build/LLBuildManifestBuilder.swift

[mixed-target-error]: https://github.com/apple/swift-package-manager/blob/ce099264a187759c2f587393bd209d317a0352b4/Sources/PackageLoading/TargetSourcesBuilder.swift#L183-L189

[`SwiftSetting.InteroperabilityMode`]: https://developer.apple.com/documentation/packagedescription/swiftsetting/interoperabilitymode

[swift-compiler-thread-fr]: https://forums.swift.org/t/se-0403-package-manager-mixed-language-target-support/66202/32

[should-emit-header]: https://github.com/apple/swift-package-manager/blob/6478e2724b8bf77856ff358cba5f59a4a62978bf/Sources/Build/BuildDescription/SwiftTargetBuildDescription.swift#L732-L735

[swift-emit-header-fr]: https://forums.swift.org/t/se-0403-package-manager-mixed-language-target-support/66202/31

[Swift-Clang-SourceModuleTarget]: https://github.com/apple/swift-package-manager/blob/8e512308530f808e9ef0cd149f4f632339c65bc4/Sources/PackagePlugin/PackageModel.swift#L231-L319
