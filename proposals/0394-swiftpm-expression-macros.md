# Package Manager Support for Custom Macros

* Proposal: [SE-0394](0394-swiftpm-expression-macros.md)
* Authors: [Boris Buegling](https://github.com/neonichu), [Doug Gregor](https://github.com/DougGregor)
* Review Manager: [Becca Royal-Gordon](https://github.com/beccadax)
* Status: **Implemented (Swift 5.9)**
* Implementation: **Available behind pre-release tools-version** ([apple/swift-package-manager#6185](https://github.com/apple/swift-package-manager/pull/6185), [apple/swift-package-manager#6200](https://github.com/apple/swift-package-manager/pull/6200))
* Review: ([pitch 1](https://forums.swift.org/t/pitch-package-manager-support-for-custom-macros/63482)) ([pitch 2](https://forums.swift.org/t/pitch-2-package-manager-support-for-custom-macros/63868)) ([review](https://forums.swift.org/t/se-0394-package-manager-support-for-custom-macros/64170)) ([acceptance](https://forums.swift.org/t/accepted-se-0394-package-manager-support-for-custom-macros/64589))

## Introduction

Macros provide a way to extend Swift by performing arbitrary syntactic transformations on input source code to produce new code. One example for this are expression macros which were previously proposed in [SE-0382](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0382-expression-macros.md). This proposal covers how custom macros are defined, built and distributed as part of a Swift package.

## Motivation

[SE-0382](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0382-expression-macros.md) and [A Possible Vision for Macros in Swift](https://gist.github.com/DougGregor/4f3ba5f4eadac474ae62eae836328b71) covered the motivation for macros themselves, defining them as part of a package will offer a straightforward way to reuse and distribute macros as source code.

## Proposed solution

Macros implemented in an external program can be declared as part of a package via a new macro target type, defined in
the `CompilerPluginSupport` library:

```swift
public extension Target {
    /// Creates a macro target.
    ///
    /// - Parameters:
    ///     - name: The name of the macro.
    ///     - dependencies: The macro's dependencies.
    ///     - path: The path of the macro, relative to the package root.
    ///     - exclude: The paths to source and resource files you want to exclude from the macro.
    ///     - sources: The source files in the macro.
    ///     - swiftSettings: The Swift settings for this macro.
    ///     - linkerSettings: The linker settings for this macro.
    ///     - plugins: The plugins used by this macro.
    static func macro(
        name: String,
        dependencies: [Dependency] = [],
        path: String? = nil,
        exclude: [String] = [],
        sources: [String]? = nil,
        swiftSettings: [SwiftSetting]? = nil,
        linkerSettings: [LinkerSetting]? = nil,
        plugins: [PluginUsage]? = nil
    ) -> Target { ... }
}
```

Similar to package plugins ([SE-0303 "Package Manager Extensible Build Tools"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0303-swiftpm-extensible-build-tools.md)), macro plugins are built as executables for the host (i.e, where the compiler is run). The compiler receives the paths to these executables from the build system and will run them on demand as part of the compilation process. Macro executables are automatically available for any target that transitively depends on them via the package manifest.

A minimal package containing the implementation, definition and client of a macro would look like this:

```swift
import PackageDescription
import CompilerPluginSupport

let package = Package(
    name: "MacroPackage",
    dependencies: [
        .package(url: "https://github.com/apple/swift-syntax", from: "509.0.0"),
    ],
    targets: [
        .macro(name: "MacroImpl",
               dependencies: [
                   .product(name: "SwiftSyntaxMacros", package: "swift-syntax"),
                   .product(name: "SwiftCompilerPlugin", package: "swift-syntax")
               ]),
        .target(name: "MacroDef", dependencies: ["MacroImpl"]),
        .executableTarget(name: "MacroClient", dependencies: ["MacroDef"]),
        .testTarget(name: "MacroTests", dependencies: ["MacroImpl"]),
    ]
)
```

Macro implementations will be executed in a sandbox [similar to package plugins](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0303-swiftpm-extensible-build-tools.md#security), preventing file system and network access. This is a practical way of encouraging macros to not depend on any state other than the specific macro expansion node they are given to expand and its child nodes (but not its parent nodes), and the information specifically provided by the macro expansion context. If in the future macros need access to other information, this will be accomplished by extending the macro expansion context, which also provides a mechanism for the compiler to track what information the macro actually queried.

Any code from macro implementations can be tested by declaring a dependency on the macro target from a test, this works similarly to the [testing of executable targets](https://github.com/apple/swift-package-manager/pull/3316).

## Detailed Design

SwiftPM builds each macro as an executable for the host platform, applying certain additional compiler flags. Macros are expected to depend on SwiftSyntax using a versioned dependency that corresponds to a particular major Swift release. Note that SwiftPM's dependency resolution is workspace-wide, so all macros (and potentially other clients) will end up consolidating on one particular version of SwiftSyntax. Each target that transitively depends on a macro will have access to it, concretely this happens by SwiftPM passing `-load-plugin-executable` to the compiler to specify which executable contains the implementation of a certain macro module (e.g. `-load-plugin-executable /path/to/package/.build/debug/MacroImpl#MacroImpl` where the argument after the hash symbol is a comma separated list of module names which can be referenced by the `module` parameter of external macro declarations). The macro definition refers to the module and concrete type via an `#externalMacro` declaration which allows any dependency of the defining target to have access to the concrete macro. If any target of a library product depends on a macro, clients of said library will also get access to any public macros. Macros can have dependencies like any other target, but product dependencies of macros need to be statically linked, so explicitly dynamic library products cannot be used by a macro target.

Concretely, the code for the macro package shown earlier would contain a macro implementation looking like this:

```swift
import SwiftSyntax
import SwiftCompilerPlugin
import SwiftSyntaxBuilder
import SwiftSyntaxMacros

@main
struct MyPlugin: CompilerPlugin {
  var providingMacros: [Macro.Type] = [FontLiteralMacro.self]
}

/// Implementation of the `#fontLiteral` macro, which is similar in spirit
/// to the built-in expressions `#colorLiteral`, `#imageLiteral`, etc., but in
/// a small macro.
public struct FontLiteralMacro: ExpressionMacro {
  public static func expansion(
    of macro: some FreestandingMacroExpansionSyntax,
    in context: some MacroExpansionContext
  ) -> ExprSyntax {
    let argList = replaceFirstLabel(
      of: macro.argumentList,
      with: "fontLiteralName"
    )
    let initSyntax: ExprSyntax = ".init(\(argList))"
    if let leadingTrivia = macro.leadingTrivia {
      return initSyntax.with(\.leadingTrivia, leadingTrivia)
    }
    return initSyntax
  }
}

/// Replace the label of the first element in the tuple with the given
/// new label.
private func replaceFirstLabel(
  of tuple: TupleExprElementListSyntax,
  with newLabel: String
) -> TupleExprElementListSyntax {
  guard let firstElement = tuple.first else {
    return tuple
  }

  return tuple.replacing(
    childAt: 0,
    with: firstElement.with(\.label, .identifier(newLabel))
  )
}
```

The macro definition would look like this:

```swift
public enum FontWeight {
  case thin
  case normal
  case medium
  case semiBold
  case bold
}

public protocol ExpressibleByFontLiteral {
  init(fontLiteralName: String, size: Int, weight: FontWeight)
}

/// Font literal similar to, e.g., #colorLiteral.
@freestanding(expression) public macro fontLiteral<T>(name: String, size: Int, weight: FontWeight) -> T = #externalMacro(module: "MacroImpl", type: "FontLiteralMacro")
  where T: ExpressibleByFontLiteral
```

And the client of the macro would look like this:

```swift
import MacroDef

struct Font: ExpressibleByFontLiteral {
  init(fontLiteralName: String, size: Int, weight: MacroDef.FontWeight) {
  }
}

let _: Font = #fontLiteral(name: "Comic Sans", size: 14, weight: .thin)
```

SwiftSyntax's versioning scheme is based on Swift major versions (e.g. 509.0.0 for Swift 5.9).

If a package depends on two macros using the `from` version dependency and minor versions of a macro use different versions of SwiftSyntax, users should automatically get a version that's compatible with all macros. For example consider the following where a package depends on both Macro 1 and Macro 2 using `from: "1.0.0"`

```
Macro 1           SwiftSyntax             Macro 2

1.0 --------------> 509.0.0 <-------------- 1.0
                    509.0.1 <-------------- 1.1
                    510.0.0 <-------------- 1.2
```

In this case, SwiftPM would choose version 1.0 for Macro 1, version 1.1 for Macro 2 and end up with version 509.0.1 for SwiftSyntax. We're going to monitor how the versioning story plays out in practice and may take further action in SwiftSyntax or SwiftPM's dependency resolution if the concrete need arises.


## Impact on existing packages

Since macro plugins are entirely additive, there's no impact on existing packages.

## Alternatives considered

### Package plugins

The original pitch of expression macros considered declaring macros by introducing a new capability to [package plugins](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0303-swiftpm-extensible-build-tools.md), but since the execution model is significantly different and the APIs used for macros are external to SwiftPM, this idea was discarded.

### `.macroTarget()`

We're (slowly) trying to move away from having the target suffix since it is implied by the context. This is already the case for plugin targets and eventually we'd like to have e.g. `.test()` as well. This would also make the target APIs be more in line with the product ones, where e.g. we don't use `.libraryProduct()`.

### Dependencies on macro targets

In [SE-0303](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0303-swiftpm-extensible-build-tools.md), we introduced the `plugins` parameter for build order dependencies on plugins, so it could have make sense to use a `macros` parameter for dependencies on macros. However, introducing bespoke API for each type of host-side content used during the build does not seem scalable. We also already have precedence of executables being part of `dependencies` even though that dependency is strictly for build ordering (with the exception of tests, which also applies to macros). Because of this, dependencies on macros are declared via the `dependencies` parameter, however it could be interesting to revisit a separation of build order and linked dependencies in the future.

## Future Directions

### Generalized support for additional manifest API

The macro target type is provided by a new library `CompilerPluginSupport` as a starting point for making package manifests themselves more extensible. Support for product and target type plugins should eventually be generalized to allow other types of externally defined specialized target types, such as, for example, a Windows application.
