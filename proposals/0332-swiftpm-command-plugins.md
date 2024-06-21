# Package Manager Command Plugins

* Proposal: [SE-0332](0332-swiftpm-command-plugins.md)
* Author: [Anders Bertelrud](https://github.com/abertelrud)
* Review Manager: [Tom Doron](https://github.com/tomerd)
* Status: **Implemented (Swift 5.6)**
* Implementation: [apple/swift-package-manager#3855](https://github.com/apple/swift-package-manager/pull/3855)
* Pitch: [Forum discussion](https://forums.swift.org/t/pitch-package-manager-command-plugins/)
* Review: [Forum discussion](https://forums.swift.org/t/se-0332-package-manager-command-plugins/)

## Introduction

[SE-0303](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0303-swiftpm-extensible-build-tools.md) introduced the ability to define *build tool plugins* in SwiftPM, allowing custom tools to be automatically invoked during a build. This proposal extends that plugin support to allow the definition of custom *command plugins* — plugins that users can invoke directly from the SwiftPM CLI, or from an IDE that supports Swift Packages, in order to perform custom actions on their packages.

## Motivation

The *build tool plugins* that were introduced in SE-0303 are focused on code generation during the build of a package, for such purposes as generating Swift source files from `.proto` files or from other inputs. In order to allow build tools to be incorporated into the build graph and to run automatically in a safe manner, there are restrictions on what such plugins can do. For example, build tool plugins are prevented from modifying any files inside a package directory.

It would be useful to support a different kind of plugin that users can invoke directly, and that can be allowed to have more flexibility than a build tool that's invoked automatically during the build. Such custom command plugins could be used for documentation generation, source code reformatting, unit test report generation, build artifact post-processing, and other uses that don't fit the definition of a typical build tool. Rather than extending the build system, such plugins could extend and improve the workflow for package authors and users, whether or not those workflows have anything to do with the build system.

One key tension in this proposal is between providing functionality that is rich enough to be useful for plugins, while still presenting that functionality in a way that's general enough to be implemented in both the SwiftPM CLI and in any IDE that supports packages. To that end, this proposal provides a minimal initial API, with the intention of adding more functionality in future proposals.

Separately to this proposal, it would also be useful to define custom actions that could run as a side effect of operations such as building and testing, and which would be called in response to various events that can happen during a build or test run — but that is not what this proposal is about, and that kind of plugin would be the subject of a future proposal. Rather, this proposal focuses on the direct invocation of a custom command by a user, independently of whether the plugin that implements the command then decides to ask SwiftPM to perform a build as part of its implementation.

## Proposed Solution

This proposal defines a new plugin capability called `command` that allows packages to augment the set of package-related commands availabile in the SwiftPM CLI and in IDEs that support packages. A command plugin specifies the semantic intent of the command — this might be one of the predefined intents such “documentation generation” or “source code formatting”, or it might be a custom intent with a specialized verb that can be passed to the `swift` `package` command. A command plugin can also specify any special permissions it needs (such as the permission to modify the files under the package directory).

The command's intent declaration provides a way of grouping command plugins by their functional categories, so that SwiftPM — or an IDE that supports SwiftPM packages — can show the commands that are available for a particular purpose. For example, this approach supports having different command plugins for generating documentation for a package, while still allowing those different commands to be grouped and discovered by intent.

As with build tool plugins, command plugins are made available to a package by declaring dependencies on the packages that provide the plugins. Unlike build tool plugins, which are applied on a target-by-target basis using a declaration in the package manifest, custom command plugins are not invoked automatically — instead they can be invoked directly by the user after the package graph has been resolved. This proposal adds options to the `swift` `package` CLI that allow users to invoke a plugin-provided command and to control the set of targets to which the command should apply. It is expected that IDEs that support SwiftPM packages should provide a way to invoke the command plugins thorugh their user interfaces.

Command plugins are implemented similarly to build tool plugins: each plugin is a Swift script that has access to API in the `PackagePlugin` module and that is invoked with parameters describing its inputs. The Swift script contains the logic to carry out the functionality of the plugin, usually by invoking other subprocesses, but potentially also by asking the package manager to perform certain actions such as building package products or running unit tests.

Unlike build tool plugins, which operate indirectly by defining build commands for SwiftPM to run at a later point in time (and only when needed), custom command plugins directly carry out the functionality of the plugin at the time they are invoked. This usually involves invoking tools that are in the toolchain or that are provided by dependencies, but it could also involve logic that is implemented completely inside the plugin itself (using Foundation APIs, for example). The plugin does not return until the command is complete.

Command plugins are provided with a read-only snapshot of the package, but they can also call into SwiftPM's build system to have it produce or update certain artifacts if it needs to. For example, a command that post-processes a release build can ask SwiftPM to build the release artifacts.

In this initial proposal there is a fairly modest set of build parameters that can be controlled by the plugin. The intent is to extend this over time, although it will only be possible to support functionality that is common enough to be available in the build systems of any IDE that supports SwiftPM command plugins (the "host" of the plugin).

As with all kinds of plugins, command plugins can emit diagnostics if it encounters any problems. Console output emitted by the command plugin is shown to users.

## Detailed Design

This proposal extends both the package manifest API and the package plugin API.

### Manifest API

This proposal defines a new `command` plugin capability in `PackageDescription`:

```swift
extension PluginCapability {
    /// Plugins that specify a `command` capability define commands that can be run
    /// using the SwiftPM CLI (`swift package <verb>`), or in an IDE that supports
    /// Swift Packages.
    public static func command(
        /// The semantic intent of the plugin (either one of the predefined intents,
        /// or a custom intent).
        intent: PluginCommandIntent,
        
        /// Any permissions needed by the command plugin. This affects what the
        /// sandbox in which the plugin is run allows. Some permissions may require
        /// approval by the user.
        permissions: [PluginPermission] = []
    ) -> PluginCapability
}
```

The plugin specifies the intent of the command as either one of a set of predefined intents or as a custom intent with an custom verb and help description.

In this proposal, the intent is expressed as an opaque struct with enum semantics in `PackageDescription`:

```swift
public struct PluginCommandIntent {
    /// The intent of the command is to generate documentation, either by parsing the
    /// package contents directly or by using the build system support for generating
    /// symbol graphs. Invoked by a `generate-documentation` verb to `swift package`.
    public static func documentationGeneration() -> PluginCommandIntent
    
    /// The intent of the command is to modify the source code in the package based
    /// on a set of rules. Invoked by a `format-source-code` verb to `swift package`.
    public static func sourceCodeFormatting() -> PluginCommandIntent
        
    /// An intent that doesn't fit into any of the other categories, with a custom
    /// verb through which it can be invoked.
    public static func custom(verb: String, description: String) -> PluginCommandIntent
}
```

Future proposals will almost certainly add to this set of possible intents, using availability annotations gated on the tools version to conditionally make new types of intent available.

If multiple command plugins in the dependency graph of a package specify the same intent, or specify a custom intent with the same verb, then the user will need to specify which plugin to invoke by qualifying the verb with the name of the plugin target followed by a `:` character, e.g. `MyPlugin:do-something`.  Because plugin names are target names, they are already known to be unique within the package graph, so the combination of plugin name and verb is known to be unique.

A command plugin can also specify the permissions it needs, which affect the ways in which the plugin can access external resources such as the file system or network. By default, command plugins have only read-only access to the file system (except for temporary-files locations) and cannot access the network.

A command plugin that wants to modify the package source code (as for example a source code formatter might want to) needs to request the `writeToPackageDirectory` permission. This modifies the sandbox in which the plugin is invoked to let it write inside the package directory in the file system, after notifying the user about what is going to happen and getting approval in a way that is appropriate for the IDE in question.

The permissions needed by the command are expressed as an opaque static struct with enum semantics in `PackageDescription`:

```swift
public struct PluginPermission {
    /// The command plugin wants permission to modify the files under the package
    /// directory. The `reason` string is shown to the user at the time of request
    /// for approval, explaining why the plugin is requesting this access.
    public static func writeToPackageDirectory(reason: String) -> PluginPermission
}
```

Future proposals will almost certainly add to this set of possible permissions, using availability annotations gated on the tools version to conditionally make new types of permission available.

In particular, it is likely that future proposals will want to provide a way for a plugin to ask for permission to access the network. In the interest of keeping this proposal bounded, we note that as a possible future need here, but do not initially allow any network access.

### Plugin API

This proposal extends the PackagePlugin API to:

* define a new kind of plugin entry point specific to command plugins
* allow the plugin to ask the Swift Package Manager to perform actions such as building or testing
* allow the plugin to ask the Swift Package Manager for specialized information such as Swift symbol graphs


#### Plugin Entry Point

This proposal extends `PluginAPI` with an entry point for command plugins:

```swift
/// Defines functionality for all plugins that have a `command` capability.
public protocol CommandPlugin: Plugin {
    /// Invoked by SwiftPM to perform the custom actions of the command.
    func performCommand(
        /// The context in which the plugin is invoked. This is the same for all
        /// kinds of plugins, and provides access to the package graph, to cache
        /// directories, etc.
        context: PluginContext,
        
        /// Any literal arguments passed after the verb in the command invocation.
        arguments: [String],
    ) async throws
    
    /// A proxy to the Swift Package Manager or IDE hosting the command plugin,
    /// through which the plugin can ask for specialized information or actions.
    var packageManager: PackageManager { get }
}
```

This defines a basic entry point for a command plugin, passing it information about the context in which the plugin is invoked (including information about the package graph) and the arguments passed by the user after the verb in the `swift` `package` invocation.

The `context` parameter provides access to the package to which the user applies the plugin, including any dependencies, and it also provides access to a working directory that the plugin can use for any purposes, as well as a way to look up command line tools with a given name. This is the same as the support that is available to all plugins via SE-0325.

An opaque reference to a proxy for the Package Manager services in SwiftPM or the host IDE is also made available to the plugin, for use in accessing derived information and for carrying out more specialized actions. This is described in more detail below.

Many command plugins will invoke tools using subprocesses in order to do the actual work. A plugin can use the Foundation module’s `Process` API to invoke executables, after using the PackagePlugin module's `PluginContext.tool(named:)` API to obtain the full path of the command line tool in the local file system.

Plugins can also use Foundation APIs for reading and writing files, encoding and decoding JSON, and other actions.

The arguments are a literal array of strings that the user specified when invoking the plugin. Plugins that operate on individual targets or products would typically support a `--target` or `--product` option that allows users to specify the names of targets or products to operate on in the package to which the plugin command is applied.

#### Accessing Package Manager Services

In addition to invoking invoking tool executables and using Foundation APIs, command plugins can use the `packageManager` property to obtain more specialized information and to invoke certain SwiftPM services. This is a proxy to SwiftPM or to the IDE that is hosting the plugin, and provides access to some of its functionality. The set of services provided in this API is expected to grow over time, and would ideally, over time, comprise most of the SwiftPM functionality available in its CLI.

```swift
/// Provides specialized information and services from the Swift Package Manager or
/// an IDE that supports Swift Packages. Different plugin hosts will implement this
/// functionality in whatever way is appropriate for them, but should preserve the
/// same semantics described here.
public struct PackageManager {
    //
    //  Building
    //
    
    /// Performs a build of all or a subset of products and targets in a package.
    ///
    /// Any errors encountered during the build are reported in the build result,
    /// as is the log of the build commands that were run. This method throws an
    /// error if the input parameters are invalid or in case the build cannot be
    /// started.
    ///
    /// The SwiftPM CLI or any IDE that supports packages may show the progress
    /// of the build as it happens.
    ///
    /// Future proposals should consider adding ways for the plugin to receive
    /// incremental progress during the build.
    public func build(
        _ subset: BuildSubset,
        parameters: BuildParameters
    ) async throws -> BuildResult
    
    /// Specifies a subset of products and targets of a package to build.
    public enum BuildSubset {
        /// Represents the subset consisting of all products and of either all
        /// targets or (if `includingTests` is false) just non-test targets.
        case all(includingTests: Bool)

        /// Represents the product with the specified name.
        case product(String)

        /// Represents the target with the specified name.
        case target(String)
    }
    
    /// Parameters and options to apply during the build.
    public struct BuildParameters {
        /// Whether to build for debug or release.
        public var configuration: BuildConfiguration = .debug
        
        /// Controls the amount of detail to include in the build log.
        public var logging: BuildLogVerbosity = .concise

        /// Additional flags to pass to all C compiler invocations.
        public var otherCFlags: [String] = []

        /// Additional flags to pass to all C++ compiler invocations.
        public var otherCxxFlags: [String] = []

        /// Additional flags to pass to all Swift compiler invocations.
        public var otherSwiftcFlags: [String] = []
        
        /// Additional flags to pass to all linker invocations.
        public var otherLinkerFlags: [String] = []

        /// Future proposals should add more controls over the build.
    }
    
    /// Represents an overall purpose of the build, which affects such things as
    /// optimization and generation of debug symbols.
    public enum BuildConfiguration {
        case debug, release
    }
    
    /// Represents the amount of detail in a build log (corresponding to the `-v`
    /// and `-vv` options to `swift build`).
    public enum BuildLogVerbosity {
        case concise, verbose, debug
    }
    
    /// Represents the results of running a build.
    public struct BuildResult {
        /// Whether the build succeeded or failed.
        public var succeeded: Bool
        
        /// Log output (in this proposal just a long text string; future proposals
        /// should consider returning structured build log information).
        public var logText: String
        
        /// The artifacts built from the products in the package. Intermediates
        /// such as object files produced from individual targets are not listed.
        public var builtArtifacts: [BuiltArtifact]
        
        /// Represents a single artifact produced during a build.
        public struct BuiltArtifact {
            /// Full path of the built artifact in the local file system.
            public var path: Path
            
            /// The kind of artifact that was built.
            public var kind: Kind
            
            /// Represents the kind of artifact that was built. The specific file
            /// formats may vary from platform to platform — for example, on macOS
            /// a dynamic library may in fact be built as a framework.
            public enum Kind {
                case executable, dynamicLibrary, staticLibrary
            }
        }
    }
    
    //
    //  Testing
    //
    
    /// Runs all or a specified subset of the unit tests of the package, after
    /// an incremental build if necessary (the same as `swift test` does).
    ///
    /// Any test failures are reported in the test result. This method throws an
    /// error if the input parameters are invalid or in case the test cannot be
    /// started.
    ///
    /// The SwiftPM CLI or any IDE that supports packages may show the progress
    /// of the tests as they happen.
    ///
    /// Future proposals should consider adding ways for the plugin to receive
    /// incremental progress during the tests.
    public func test(
        _ subset: TestSubset,
        parameters: TestParameters
    ) async throws -> TestResult
        
    /// Specifies what tests in a package to run.
    public enum TestSubset {
        /// Represents all tests in the package.
        case all

        /// Represents one or more tests filtered by regular expression, with the
        /// format <test-target>.<test-case> or <test-target>.<test-case>/<test>.
        /// This is the same as the `--filter` option of `swift test`.
        case filtered([String])
    }
    
    /// Parameters that control how the tests are run.
    public struct TestParameters {
        /// Whether to collect code coverage information while running the tests.
        public var enableCodeCoverage: Bool = false
        
        /// Future proposals should add more controls over running the tests.
    }
    
    /// Represents the result of running unit tests.
    public struct TestResult {
        /// Whether the test run succeeded or failed.
        public var succeeded: Bool
        
        /// Results for all the test targets that were run (filtered based on
        /// the input subset passed when running the test).
        public var testTargets: [TestTarget]
        
        /// Represents the results of running some or all of the tests in a
        /// single test target.
        public struct TestTarget {
            public var name: String
            public var testCases: [TestCase]
            
            /// Represents the results of running some or all of the tests in
            /// a single test case.
            public struct TestCase {
                public var name: String
                public var tests: [Test]

                /// Represents the results of running a single test.
                public struct Test {
                    public var name: String
                    public var result: Result
                    public var duration: Double
                    
                    /// Represents the result of running a single test.
                    public enum Result {
                        case succeeded, skipped, failed
                    }
                }
            }
        }

        /// Path of a generated `.profdata` file suitable for processing using
        /// `llvm-cov`, if `enableCodeCoverage` was set in the test parameters.
        public var codeCoverageDataFile: Path?
    }
    
    //
    //  Accessing Specialized Information
    //
        
    /// Return a directory containing symbol graph files for the given target
    /// and options. If the symbol graphs need to be created or updated first,
    /// they will be. SwiftPM or an IDE may generate these symbol graph files
    /// in any way it sees fit.
    public func getSymbolGraph(
        for target: Target,
        options: SymbolGraphOptions
    ) async throws -> SymbolGraphResult

    /// Represents options for symbol graph generation. These options are taken
    /// into account when determining whether generated information is already
    /// up-to-date.
    public struct SymbolGraphOptions {
        /// The symbol graph will include symbols at this access level and higher.
        public var minimumAccessLevel: AccessLevel = .public
      
        /// Represents a Swift access level.
        public enum AccessLevel {
            case `private`, `fileprivate`, `internal`, `public`, `open`
        }
        
        /// Whether to include synthesized members.
        public var includeSynthesized: Bool = false
        
        /// Whether to include symbols marked as SPI.
        public var includeSPI: Bool = false
    }

    /// Represents the result of symbol graph generation.
    public struct SymbolGraphResult {
        /// The directory that contains the symbol graph files for the target.
        public var directoryPath: Path
    }
}
```

### Permissions

Like other plugins, command plugins are run in a sandbox on platforms that support it. By default this sandbox does not allow the plugin to modify the file system (except in special temporary-files paths) and it blocks any network access.

Some commands, such as source code formatters, might need to modify the file system in order to be useful. Such plugins can specify the permissions they need, and this will:

* notify the user about the need for the additional permission and provide a way to approve or reject it
* if the user approves, cause the sandbox to be modified in an appropriate manner

The exact form of the notification and approval will depend on the CLI or IDE that runs the plugin. SwiftPM’s CLI is expected to ask the user for permission using a console prompt (if connected to TTY), and to provide options for approving or rejecting the request when not connected to a TTY.

Note that this approval needs to be obtained before running the plugin, which is why it is declared in the package manifest. There is currently no provision for a plugin to ask for more permissions while it runs.

An IDE might present user interface affordances  providing the notification and allowing the choice. In order to avoid having to request permission every time the plugin is invoked, some kind of caching of the response could be implemented.

SwiftPM or IDEs may also provide options to allow users to specify additional writable file system locations for the plugin, but that would not affect the API described in this proposal.

### Invoking Command Plugins

In the SwiftPM CLI, command plugins provided by a package or its direct dependencies are available as verbs that can be specified in a `swift` `package` invocation. For example, if the root package defines a command plugin with a `do-something` verb — or if it has a dependency on a package that defines such a plugin — a user can run it using the invocation:

```shell
❯ swift package do-something
```

This will invoke the plugin and only return when it completes. Since no other options were provided, this will pass all regular targets in the package to the plugin ("special" targets such as those that define plugins will be excluded).

Any parameters passed after the name of the plugin command are passed verbatim to the entry point of the plugin. For example, if a plugin accepts a `--target` option, a subset of the targets to operate on can be passed on the command line that invokes the plugin:

```shell
❯ swift package do-something --target Foo --target Bar --someOtherFlag
```

It is the responsibility of the plugin to interpret any command line arguments passed to it.

Arguments are currently passed to the plugin exactly as they are written after the command’s verb. A future proposal could allow the plugin to define parameters (using SwiftArgumentParser) that SwiftPM could interpret and that would integrate better with SwiftPM’s own command line arguments.

As mentioned in the *Permissions* section, command plugins are by default blocked from modifying the files inside the package directory on platforms that support sandboxing. If a command plugin that requires file system writability is invoked, `swift` `package` will ask for approval — this is done using console input if stdin is connected to a TTY, or if not, an error will be reported without invoking the plugin. An `--allow-writing-to-package-directory` option can be used to bypass the request to approve the file system access, which is useful in CI and other automation.

```shell
❯ swift package --allow-writing-to-package-directory do-something
```

Asking for permission from the user helps to prevent unexpected modification of the package by command plugins.

In IDEs that support Swift packages, command plugins could be provided through context menus or other user interface affordances that allow the commands to be invoked on a package or possibly on a selection of targets in a package. A plugin itself does not need to know, and should not make assumptions about, how or in what context it is being invoked.

If a future proposal introduces a way of declaring parameters in a manner similar to SwiftArgumentParser, then an IDE could possibly also show a more targeted user interface for those parameters, since the types and optionality will be known.

### Discovering Command Plugins

Any plugins defined by a package are included in the `swift` `package` `describe` output for that package.

Because the command plugins that are available to a package also include those that are defined as plugin products by any package dependencies, it is also useful to have a convenient way of listing all commands that are visible to a particular package.  This is provided by the `swift` `package` `plugin` `--list` option, which defaults to text output but also supports a `--json` option. A `--capability` option can be used to filter plugins to only those supporting a particular capability.

For example:

```shell
❯ swift package plugin --list --capability=buildTool
```

would produce textual output of any plugins with a build tool capacity available to the package, while:

```shell
❯ swift package plugin --list --capability=command --json
```

would produce JSON output of an plugins with a command capacity available to the package.

## Example 1:  Generating Documentation

Here's a brief example of a hypothetical command plugin that uses `docc` to generate documentation for one or more targets in a package. This example calls back to the plugin host (SwiftPM or an IDE) to generate symbol graphs.

The package manifest contains the `.plugin()` declaration:

```swift
// swift-tools-version: 5.6
import PackageDescription

let package = Package(
    name: "MyDocCPlugin",
    products: [
        // Declaring the plugin product vends the plugin to clients of the package.
        .plugin(
            name: "MyDocCPlugin",
            targets: ["MyDocCPlugin"]
        ),
    ],
    targets: [
        // This is the target that implements the command plugin.
        .plugin(
            name: "MyDocCPlugin",
            capability: .command(
                intent: .documentationGeneration()
            )
        )
    ]
)
```

The implementation of the package plugin itself:

```swift
import PackagePlugin
import Foundation

@main
struct MyDocCPlugin: CommandPlugin {
    func performCommand(
        context: PluginContext,
        arguments: [String]
    ) async throws {
        // We'll be creating commands that invoke `docc`, so start by locating it.
        let doccTool = try context.tool(named: "docc")

        // Construct the path of the directory in which to emit documentation.
        let outputDir = context.pluginWorkDirectory.appending("Outputs")

        // Iterate over the targets in the package.
        for target in context.package.targets {
            // Only consider those kinds of targets that can have source files.
            guard let target = target as? SourceModuleTarget else { continue }

            // Find the first DocC catalog in the target, if there is one (a more
            // robust example would handle the presence of multiple catalogs).
            let doccCatalog = target.sourceFiles.first { $0.path.extension == "docc" }

            // Ask SwiftPM to generate or update symbol graph files for the target.
            let symbolGraphInfo = try await packageManager.getSymbolGraph(for: target,
                options: .init(
                    minimumAccessLevel: .public,
                    includeSynthesized: false,
                    includeSPI: false))

            // Invoke `docc` with arguments and the optional catalog.
            let doccExec = URL(fileURLWithPath: doccTool.path.string)
            var doccArgs = ["convert"]
            if let doccCatalog = doccCatalog {
                doccArgs += ["\(doccCatalog.path)"]
            }
            doccArgs += [
                "--fallback-display-name", target.name,
                "--fallback-bundle-identifier", target.name,
                "--fallback-bundle-version", "0",
                "--additional-symbol-graph-dir", "\(symbolGraphInfo.directoryPath)",
                "--output-dir", "\(outputDir)",
            ]
            let process = try Process.run(doccExec, arguments: doccArgs)
            process.waitUntilExit()

            // Check whether the subprocess invocation was successful.
            if process.terminationReason == .exit && process.terminationStatus == 0 {
                print("Generated documentation at \(outputDir).")
            }
            else {
                let problem = "\(process.terminationReason):\(process.terminationStatus)"
                Diagnostics.error("docc invocation failed: \(problem)")
            }
        }
    }
}
```

In order to use this plugin from another package, a dependency would be used on the package that declares the plugin:

```swift
// swift-tools-version: 5.6
import PackageDescription

let package = Package(
    name: "MyLibrary",
    dependencies: [
        .package(url: "https://url/of/docc/plugin/package", from: "1.0.0"),
    ],
    targets: [
        .target(name: "MyLibrary")
    ]
)
```

Note, that, unlike with built tool plugins, there is no `plugins` clause for command plugins — this is because they are applied explicitly by user action and not implicitly when building targets.

Users can then invoke this command plugin using the `swift` `package` invocation:

```shell
❯ swift package generate-documentation
```

The plugin would usually print the path at which it generated the documentation.

## Example 2: Formatting Source Code

This example uses `swift-format` to reformat the code in a package, which requires the plugin to have `.writeToPackageDirectory` permission.

Note that this package depends on the executable provided by the *swift-format* package.

```swift
// swift-tools-version: 5.6
import PackageDescription

let package = Package(
    name: "MyFormatterPlugin",
    dependencies: [
        .package(url: "https://github.com/apple/swift-format.git", from: "0.50500.0"),
    ],
    targets: [
        .plugin(
            name: "MyFormatterPlugin",
            capability: .command(
                intent: .sourceCodeFormatting(),
                permissions: [
                    .writeToPackageDirectory(reason: "This command reformats source files")
                ]
            ),
            dependencies: [
                .product(name: "swift-format", package: "swift-format"),
            ]
        )
    ]
)
```

The implementation of the package plugin itself:

```swift
import PackagePlugin
import Foundation

@main
struct MyFormatterPlugin: CommandPlugin {
    func performCommand(
        context: PluginContext,
        arguments: [String]
    ) async throws {
        // We'll be invoking `swift-format`, so start by locating it.
        let swiftFormatTool = try context.tool(named: "swift-format")

        // By convention, use a configuration file in the package directory.
        let configFile = context.package.directory.appending(".swift-format.json")

        // Iterate over the targets in the package.
        for target in context.package.targets {
            // Skip any type of target that doesn't have source files.
            // Note: We could choose to instead emit a warning or error here.
            guard let target = target as? SourceModuleTarget else { continue }

            // Invoke `swift-format` on the target directory, passing a configuration
            // file from the package directory.
            let swiftFormatExec = URL(fileURLWithPath: swiftFormatTool.path.string)
            let swiftFormatArgs = [
                "--configuration", "\(configFile)",
                "--in-place",
                "--recursive",
                "\(target.directory)"
            ]
            let process = try Process.run(swiftFormatExec, arguments: swiftFormatArgs)
            process.waitUntilExit()

            // Check whether the subprocess invocation was successful.
            if process.terminationReason == .exit && process.terminationStatus == 0 {
                print("Formatted the source code in \(target.directory).")
            }
            else {
                let problem = "\(process.terminationReason):\(process.terminationStatus)"
                Diagnostics.error("swift-format invocation failed: \(problem)")
            }
        }
    }
}
```

Users can then invoke this command using the `swift` `package` invocation:

```shell
❯ swift package format-source-code
```

Since `--allow-writing-to-package-directory` is not passed, `swift` `package` will ask the user for permission if its stdin is attached to a TTY, or will fail with an error if not. If `--allow-writing-to-package-directory` were passed, it would just allow the plugin to run (with package directory writability allowed by the sandbox profile) without asking for permission.

## Example 3: Building Deployment Artifacts

The final example of a command plugin uses the `PackageManager` service provider’s build functionality to do a release build of a hypothetical product and then to create a distribution archive from it.

This example shows use of a local plugin target, so no package dependency is needed. This is mostly appropriate for custom commands that are unlikely to be useful outside the package.

```swift
// swift-tools-version: 5.6
import PackageDescription

let package = Package(
    name: "MyExecutable",
    products: [
        .executable(name: "MyExec", targets: ["MyExec"])
    ],
    targets: [
        // This is the hypothetical executable we want to distribute.
        .executableTarget(
            name: "MyExec"
        ),
        // This is the plugin that defines a custom command to distribute the executable.
        .plugin(
            name: "MyDistributionArchiveCreator",
            capability: .command(
                intent: .custom(
                    verb: "create-distribution-archive",
                    description: "Creates a .zip containing release builds of products"
                )
            )
        )
    ]
)
```

The implementation of the package plugin itself:

```swift
import PackagePlugin
import Foundation

@main
struct MyDistributionArchiveCreator: CommandPlugin {
    func performCommand(
        context: PluginContext,
        arguments: [String]
    ) async throws {
        // Check that we were given the name of a product as the first argument
        // and the name of an archive as the second.
        guard arguments.count == 2 else {
            throw Error("Expected two arguments: product name and archive name")
        }
        let productName = arguments[0]
        let archiveName = arguments[1]

        // Ask the plugin host (SwiftPM or an IDE) to build our product.
        let result = try await packageManager.build(
            .product(productName),
            parameters: .init(configuration: .release, logging: .concise)
        )
        
        // Check the result. Ideally this would report more details.
        guard result.succeeded else { throw Error("couldn't build product") }

        // Get the list of built executables from the build result.
        let builtExecutables = result.builtArtifacts.filter{ $0.kind == .executable }

        // Decide on the output path for the archive.
        let outputPath = context.pluginWorkDirectory.appending("\(archiveName).zip")

        // Use Foundation to run `zip`. The exact details of using the Foundation
        // API aren't relevant; the point is that the built artifacts can be used
        // by the script.
        let zipTool = try context.tool(named: "zip")
        let zipArgs = ["-j", outputPath.string] + builtExecutables.map{ $0.path.string }
        let zipToolURL = URL(fileURLWithPath: zipTool.path.string)
        let process = try Process.run(zipToolURL, arguments: zipArgs)
        process.waitUntilExit()

        // Check whether the subprocess invocation was successful.
        if process.terminationReason == .exit && process.terminationStatus == 0 {
            print("Created distribution archive at \(outputPath).")
        }
        else {
            let problem = "\(process.terminationReason):\(process.terminationStatus)"
            Diagnostics.error("zip invocation failed: \(problem)")
        }
    }
}
```

Users can then invoke this custom command using the `swift` `package` invocation:

```shell
❯ swift package create-distribution-archive MyExec MyDistributionArchive-1.0
```

This example does not need to ask for permission to write to the package directory since it only writes to the temporary directory provided by the context. A future proposal could allow the plugin to also get permission to write to output directories provided by the user.

## Security Considerations

On platforms where SwiftPM supports sandboxing, all plugins are sandboxed in a way that restricts their access to certain system resources. By default, plugins are prevented from writing to the file system (other than to temporary directories and cache directories), and are prevented from accessing the network.

Custom command plugins that need special permissions — such as writing to the package source directory — can specify a requirement for this permission in the declaration of the plugin. This may cause user interaction to approve the plugin’s request, and if granted, the sandbox is modified to allow this access.

The form that this request for approval will take depends on whether the plugin is invoked from the SwiftPM CLI or from an IDE that supports Swift Packages. The CLI may implement an option that needs to be passed at the time the plugin is invoked, while an IDE should ideally cache the response in some way that prevents the user from being prompted every time they invoke the plugin.

On platforms where SwiftPM does not support sandboxing, the user should be notified that invoking the command plugin will result in running code that might perform any action, and should be given the location of the Swift script that implements the plugin so it can be examined by the user.

## Alternatives Considered

### Package Manager services

Most of the alternatives that were considered for this proposal center around what kinds of services the Package Manager provides to plugins. This proposal chooses to expose to the plugin some of the most common actions, such as building and testing, that are also available in SwiftPM CLI commands such as `swift` `build` and `swift` `test`.

There was an intentional choice to keep the set of options provided as simple as possible in order to keep this proposal bounded. Future proposals should be able to extend this API to provide more options and additional functionality.

### Declaring prerequisites in the manifest

An alternative to having the plugin use the `PackageManager` APIs to call back to the host to get specialized information and to perform builds or run tests would be to declare these prerequisites in the manifest of the package that provides the plugin. Under such an approach, SwiftPM would first make sure that the prerequisites are satisfied before invoking the plugin at all.

The major problem with such an approach is that it’s difficult to express conditional dependencies in the manifest without adding greatly to the complexity of the manifest API, and any approach relying on such up-front prerequisites would necessarily be less flexible than letting the plugin perform package actions when and if it needs them. It would also be contrary to the goal of keeping the package manifest as clear and simple as possible. The implementation of the plugin seems like a much more appropriate place for any non-trivial logic regarding its prerequisites.

## Future Directions

### Better support for plugin options

In this initial proposal, the command plugin is passed all the command line options that the user provided after the command verb in the `swift` `package` invocation. It is then up to the plugin logic to interpret these options.

Since SwiftPM currently has only a single-layered package dependency graph, it isn't feasible in today's SwiftPM to allow a plugin to define its own dependencies on packages such as SwiftArgumentParser.

Once this is possible, a future direction might be to have a command plugin use SwiftArgumentParser to declare a supported set of input parameters. This could allow SwiftPM (or possibly an IDE) to present an interface for those plugin options — IDEs, in particular, could construct user interfaces for well-defined options (possibly in the manner of the archaic MPW `Commando` tool).

Another direction might be for the PackagePlugin API to define its own facility for a plugin to declare externally visible properties. This might include considerations particular to plugins, such as whether or not a particular path property is intended to be writable (requiring permission from the user before the plugin runs). As with SwiftArgumentParser, a natural approach would be to declare such properties on the type that implements the plugin, with their values having been set by the plugin host at the time the plugin is invoked.

### Additional access to Package Manager services

The API in the `PackageManager` type that this proposal defines is just a start. The idea is to, over time, offer plugins a variety of functionality and derivable information that they can request and then further process.

Currently, the only specialized information that a user command plugin can request from SwiftPM is the directory of symbol graph files for a particular target. The intent is to provide a menu of useful information that might or might not require computation in order to provide, and to allow the plugin to request this information from SwiftPM whenever it needs it.

Extending the `PackageManager` API does need to be done in a way that is possible to implement in various IDEs that support Swift packages but that use a different build system than SwiftPM's.

### Providing access to build and test progress and structured results

The initial proposed API for having plugins run builds and tests is fairly minimal. In particular, the build log is returned at the end of the build as a single text string, and the plugin has no way to cancel the build. Future proposals should extend this, ideally to the point at which `swift` `build` and `swift` `test` could themselves be implemented using the same API as for custom commands.

### Allowing a plugin to report progress

While a plugin can emit diagnostics using the `Diagnostics` type, there is currently no way for a plugin to report progress while it is running. This would be very useful for long-running plugins, and should be addressed in a future proposal.
