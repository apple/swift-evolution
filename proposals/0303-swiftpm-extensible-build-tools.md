# Package Manager Extensible Build Tools

* Proposal: [SE-0303](0303-swiftpm-extensible-build-tools.md)
* Authors: [Anders Bertelrud](https://github.com/abertelrud), [Konrad 'ktoso' Malawski](https://github.com/ktoso), [Tom Doron](https://github.com/tomerd)
* Review Manager: [Tom Doron](https://github.com/tomerd)
* Status: **Implemented (5.6)**
* Amendment status: **Accepted**
* Revision Pitch and Discussion: [Pitch: Amend SE-0303 Plugin API to Use `@main` for Plugin Entry Point](https://forums.swift.org/t/pitch-amend-se-0303-plugin-api-to-use-main-for-plugin-entry-point/51250)
* Original Pitch and Discussion: [Pitch: SwiftPM Extensible Build Tools](https://forums.swift.org/t/pitch-swiftpm-extensible-build-tools/44715)
* Original Reviews: 
  * [First review](https://forums.swift.org/t/se-0303-package-manager-extensible-build-tools/)
  * [Second review](https://forums.swift.org/t/se-0303-2nd-review-package-manager-extensible-build-tools/)
* Amendment (Amend SE-0303 Plugin API to Use `@main` for Plugin Entry Point)
  * [Pitch and Discussion](https://forums.swift.org/t/pitch-amend-se-0303-plugin-api-to-use-main-for-plugin-entry-point/51250)
  * [Review](https://forums.swift.org/t/amendment-se-0303-package-manager-extensible-build-tools/)
  * [Implementation](https://github.com/apple/swift-package-manager/pull/3712)
* Previous Revisions:
  * [First revision](https://github.com/swiftlang/swift-evolution/blob/878e496eb799fa407ad704d89fb401952fe8fd02/proposals/0303-swiftpm-extensible-build-tools.md) 
  * [Second revision](https://github.com/swiftlang/swift-evolution/blob/38731efc140a53553aff923a6616a1dee28c973a/proposals/0303-swiftpm-extensible-build-tools.md)
  * [Third revision](https://github.com/swiftlang/swift-evolution/blob/7c3de3eaed8e160feca1d39a35d2f8ba7b2add0d/proposals/0303-swiftpm-extensible-build-tools.md)


## Introduction

This is a proposal for extensible build tools support in Swift Package Manager. The initial set of functionality is intentionally basic, and focuses on a general way of extending the build command graph through plugins.

The approach is to:

- provide a scalable way for packages to define plugins that can implement various capabilities
- define a narrowly scoped initial build tool capability that allows plugins to create new build commands

The set of available plugin capabilities can then be extended in future SwiftPM versions. The goal of this proposal is to provide short-term support for common tasks such as source code generation, with a design that can scale to more complex tasks in the future.

This proposal depends on improvements to the existing `binaryTarget` type in SwiftPM — those details are the subject of the separate proposal [SE-0305](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0305-swiftpm-binary-target-improvements.md).

## Motivation

SwiftPM doesn’t currently provide any means of performing custom actions during a build. This includes source generation as well as custom processing for special types of resources.

This is quite restrictive, and affects even packages with relatively simple customization needs. Examples include invoking source generators such as [SwiftProtobuf](https://github.com/apple/swift-protobuf) or [SwiftGen](https://github.com/SwiftGen/SwiftGen), or running a custom command to modify a built artifact after it is produced.

Providing even basic support for extensibility is expected to allow more codebases to be built using the Swift Package Manager, and to automate many steps that package authors currently have to do by hand (such as generate sources manually and commit them to their package repositories).

## Proposed Solution

This proposal introduces *package plugins* that implement various *capabilities*, and it defines a single initial *build tool* capability.

Package plugins are Swift scripts that use API provided by a new `PackagePlugin` library to implement custom actions for SwiftPM to perform. Plugins are defined using package targets of a new type called `plugin`.

Package plugins are in some ways similar to package manifests: both are Swift scripts that are evaluated in sandboxed environments and that use specialized APIs for a limited and specific purpose. In the case of a package manifest, that purpose is to define those characteristics of a package that cannot be inferred from the contents of the file system; in the case of a package plugin, the purpose is to extend SwiftPM with new functionality.

Note that a plugin itself does *not* perform the actual work of a custom build tool or other action — that is done by a command invocation or other configured action that the plugin constructs and returns to SwiftPM. The plugin can be thought of as a procedural way of configuring commands, and it is typically quite small.

Many different kinds of plugin capabilities are envisioned for future proposals. This proposal only defines an initial capability that allows plugins to create commands that should run at build time.

Different kinds of plugins will be invoked in different ways, as appropriate for their capability. In the case of the build tool capability, plugins are applied on-demand by having targets opt into using those plugins.

To support that, this proposal adds a new `plugins` parameter to the declarations of `target`, `executableTarget`, and `testTarget` types that allow these kinds of targets to use one or more build tool plugins. The `binaryTarget` and `systemLibrary` target types do not support plugins in this initial proposal, since they are pseudo-targets and are not actually built.

A build tool plugin is invoked after package resolution and validation, and is given access to an input context that provides information about the target to which the plugin is applied. The plugin also has read-only access to the source directory of the target, and is also allowed to write to specially designated areas of the build output directory.

This initial proposal does not directly provide a way for the client target to pass configuration parameters to a plugin. However, because plugins have read-only access to the package directory, they can read custom configuration files as needed. While this means that configuration of the plugin resides outside of the package manifest, it does allow each plugin to use a configuration format suitable for its own needs. This pattern is commonly used in practice already, in the form of JSON or YAML configuration files for source generators, etc. Future proposals are expected to let package plugins define options that can be controlled from the client package's manifest.

A `plugin` target should declare dependencies on the targets that provide the executables needed during the build. The binary target type has been extended to let it vend pre-built executables for build tools that are not built using SwiftPM. This is the subject of the separate evolution proposal [SE-0305](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0305-swiftpm-binary-target-improvements.md).

A plugin script itself cannot initially use custom libraries built by other SwiftPM targets. Initially, the only APIs available are `PackagePlugin` and the Swift standard libraries. This is somewhat limiting, but note that the plugin itself is only expected to contain a minimal amount of logic to construct a command that will then invoke a tool during the actual build. The tool that is invoked during the build — the one for which the plugin generates a command line — is either a regular `executableTarget` (which can depend on as many library targets as it wants to) or a `binaryTarget` that provides prebuilt binary artifacts.

It is a future goal to allow package plugin scripts themselves to depend on custom libraries, but that will require larger changes to how SwiftPM — and any IDEs that use libSwiftPM — create their build plans and run their builds. In particular, it will require those build systems to be able to build any libraries that are needed by the plugin script before invoking it, prior to the actual build of the Swift package. SwiftPM's native build system does not currently support such tiered builds, and neither do the build systems of some of the IDEs that use libSwiftPM.

In order to let other packages use a `plugin` target, it must made visible to other packages through a `plugin` product type (just as for other kinds of targets). If and when a future version of SwiftPM unifies the concepts of products and targets — which would be desirable for other reasons — then this distinction between plugin targets and plugin products will become unnecessary.

A package plugin target can be used by other targets in the same package without declaring a corresponding product in the manifest. This can be useful for highly specialized build tool plugins that are defined and used in the same package.

As with the `PackageDescription` API for package manifests, the `PackagePlugin` API availability annotations will be tied to the Swift Tools Version of the package that contains the `plugin` target. This will allow the API to evolve over time without affecting older plugins.

## Detailed Design

To allow plugins to be declared, the following API will be added to `PackageDescription`:

```swift
extension Target {
    /// Defines a new package plugin target with a given name, declaring it as
    /// providing a capability of extending SwiftPM in a particular way.
    ///
    /// The capability determines the way in which the plugin extends SwiftPM,
    /// which determines the context that is available to the plugin and the
    /// kinds of commands it can create. The plugin capability also determines
    /// how the plugin is activated.
    ///
    /// In the initial version of this proposal, only a single plugin capability
    /// is defined: build tool. The intent is to define additional capabilities
    /// in the future.
    /// 
    /// Another possible capability that could be added in the future could be
    /// a way to augment the testing support in SwiftPM. This could take the
    /// form of allowing additional commands to run after the build and test
    /// have completed, with a well-defined way to access build results and
    /// test results. Another possible capability could be specific support
    /// for code linters that could emit structured diagnostics with fix-its,
    /// or for code formatters that can modify the source code as a separate
    /// action outside the build.
    ///
    /// The package plugin itself is implemented using a Swift script that is
    /// invoked for each target that uses it. The script is invoked after the
    /// package graph has been resolved, but before the build system creates its
    /// dependency graph. It is also invoked after changes to the target or the
    /// build parameters.
    ///
    /// Note that the role of the package plugin is only to define the commands
    /// that will run before the build and during the build. It does not itself
    /// run those commands. The commands are defined in an IDE-neutral way, and
    /// are run as appropriate by the build system that builds the package. The
    /// plugin itself is only a procedural way of generating commands and their
    /// input and output dependencies.
    ///
    /// The package plugin may specify the executable targets or binary targets
    /// that provide the build tools that will be used by the generated commands
    /// during the build. In the initial implementation, prebuild commands can
    /// only depend on binary targets. Regular build commands can depend on exe-
    /// cutables as well as binary targets. This is due to limitations in how
    /// SwiftPM's build system constructs its build plan. It is a goal to remove
    /// this restriction in a future release.
    ///
    /// The `path`, `exclude`, and `sources` parameters are the same as for any
    /// other target, and allow flexibility in where the package author can put
    /// the plugin scripts inside the package directory. The default subdirectory
    /// for plugin targets is in a subdirectory called "Plugins", but this can
    /// be customized using the `path` parameter.
    public static func plugin(
        name: String,
        capability: PluginCapability,
        dependencies: [Dependency] = [],
        path: String? = nil,
        exclude: [String] = [],
        sources: [String]? = nil
    ) -> Target
}

extension Product {
    /// Defines a product that vends a package plugin target for use by clients
    /// of the package. It is not necessary to define a product for a plugin that
    /// is only used within the same package as it is defined. All the targets
    /// listed must be plugin targets in the same package as the product. They
    /// will be applied to any client targets of the product in the same order
    /// as they are listed.
    public static func plugin(
        name: String,
        targets: [String]
    ) -> Product
}

final class PluginCapability {
    /// Plugins that define a `buildTool` capability define commands to run at vari-
    /// ous points during the build.
    public static func buildTool(        
        /// Currently the plugin is invoked for every target that is specified as
        /// using it. Future SwiftPM versions could refine this so that plugins
        /// could, for example, provide input filename filters that further control
        /// when they are invoked.
    ) -> PluginCapability
        
    // The idea is to add additional capabilities in the future, each with its own
    // semantics. A plugin can implement one or more of the capabilities, and it
    // will be invoked within a context relevant for that capability.
}
```

To allow plugins to be applied to targets, the following API will be added to `PackageDescription`:

```swift
extension Target {
    .target(
        . . .
        plugins: [PluginUsage] = []
    ),
    .executableTarget(
        . . .
        plugins: [PluginUsage] = []
    ),
    .testTarget(
        . . .
        plugins: [PluginUsage] = []
    )
}

final class PluginUsage {
    // Specifies the use of a package plugin with a given target or product name.
    // In the case of a plugin target in the same package, no package parameter is
    // provided; in the case of a plugin product in a different package, the name
    // of the package that provides it needs to be specified. This is analogous to
    // product dependencies and target dependencies.
    public static func plugin(
        _ name: String,
        package: String? = nil
    ) -> PluginUsage
}
```

The plugins that a target uses are applied to it in the order in which they are listed — this allows one plugin to act on output files produced by another plugin.

#### Plugin API

The API of the new `PackagePlugin` library lets the plugin construct build commands based on information about the target to which the plugin is applied. The context includes the target and module names, the set of source files in the target (including those generated by previously applied plugins), information about the target's dependency closure, and other inputs from the package.

The initial proposed `PackagePlugin` API is the following:

```swift
///
/// PackagePlugin API
///
/// Like package manifests, package plugins are Swift scripts that use API
/// from a special library provided by SwiftPM. In the case of plugins, this
/// library is `PackagePlugin`. Plugins run in a sandbox, and have read-only
/// access to the package directory.
///
/// The input to a package plugin is provided by SwiftPM when it is invoked,
/// and can be accessed through the `context` parameter. The plugin defines
/// commands to run during the build by constructing and returning them from
/// the main plugin function.
///
/// The `Plugin` protocol defines the functionality common to all types of
/// plugins, and there is a specific protocol for each of the defined plugin
/// capabilities (only the `BuildToolPlugin` is presently defined, but the
/// intent to is allow additional capabilities in the future).
///
/// Each plugin defines a type that conforms to the protocol corresponding
/// to the capability it provides, and annotates that type with `@main`.
/// It then implements the corresponding methods, which will be called to
/// perform the functionality of the plugin.

/// Defines functionality common to all plugins.
protocol Plugin {
    /// Instantiates the plugin. This happens once per invocation of the
    /// plugin; there is no facility for keeping in-memory state from one
    /// invocation to the next. Most plugins do not need to implement the
    /// initializer.
    ///
    /// If a future version of SwiftPM allows the usage of a plugin to
    /// also provide configuration parameters for that plugin, then a new
    /// initializer that accepts that configuration could be added here.
    init()
}

/// Defines functionality for all plugins having a `buildTool` capability.
protocol BuildToolPlugin: Plugin {
    /// Invoked by SwiftPM to create build commands for a particular target.
    /// The context parameter contains information about the package and its
    /// dependencies, as well as other environmental inputs.
    ///
    /// This function should create and return build commands or prebuild
    /// commands, configured based on the information in the context.
    func createBuildCommands(
        context: TargetBuildContext
    ) async throws -> [Command]
}

/// Provides information about the target being built, as well as contextual
/// information such as the paths of the directories to which commands should
/// be configured to write their outputs. This information should be used as
/// part of generating the commands to be run during the build.
protocol TargetBuildContext {
    /// The name of the target being built, as specified in the manifest.
    var targetName: String { get }
    
    /// The module name of the target. This is currently derived from the name,
    /// but could be customizable in the package manifest in a future SwiftPM
    /// version.
    var moduleName: String { get }
    
    /// The path of the target source directory.
    var targetDirectory: Path { get }
    
    /// That path of the package that contains the target.
    var packageDirectory: Path { get }
    
    /// Information about the input files specified in the target being built,
    /// including the sources, resources, and other files. This sequence also
    /// includes any source files generated by other plugins that are listed
    /// earlier than this plugin in the `plugins` parameter of the target
    /// being built.
    var inputFiles: FileList { get }
    
    /// Information about all targets in the dependency closure of the target
    /// to which the plugin is being applied. This list is in topologically
    /// sorted order, with immediate dependencies appearing earlier and more
    /// distant dependencies later in the list. This is mainly intended for
    /// generating lists of search path arguments, etc.
    var dependencies: [DependencyTargetInfo] { get }
    
    /// Provides information about a target that appears in the dependency
    /// closure of the target to which the plugin is being applied.
    protocol DependencyTargetInfo {
        
        /// The name of the target.
        var targetName: String { get }
        
        /// The module name of the target. This is currently derived from the
        /// name, but could be customizable in the package manifest in a future
        /// SwiftPM version.
        var moduleName: String { get }
        
        /// Path of the target source directory.
        var targetDirectory: Path { get }
        
        /// Path of the public headers directory, if any (Clang targets only).
        var publicHeadersDirectory: Path? { get }
    }

    /// The path of a writable directory into which the plugin or the build
    /// commands it constructs can write anything it wants. This could include
    /// any generated source files that should be processed further, and it
    /// could include any caches used by the build tool or the plugin itself.
    /// The plugin is in complete control of what is written under this di-
    /// rectory, and the contents are preserved between builds.
    ///
    /// A plugin would usually create a separate subdirectory of this directory
    /// for each command it creates, and the command would be configured to
    /// write its outputs to that directory. The plugin may also create other
    /// directories for cache files and other file system content that either
    /// it or the command will need.
    var pluginWorkDirectory: Path { get }
      
    /// The path of the directory into which built products associated with
    /// the target are written.
    var builtProductsDirectory: Path { get }
    
    /// Looks up and returns the path of a named command line executable tool.
    /// The executable must be provided by an executable target or a binary
    /// target on which the package plugin target depends. This function throws
    /// an error if the tool cannot be found. The lookup is case sensitive.
    func tool(named name: String) throws -> ToolInfo
    
    /// Information about a particular tool that is available to a plugin.
    protocol ToolInfo {
        /// Name of the tool, suitable for display purposes.
        var name: String { get }
        
        /// Path of the built or provided tool in the file system.
        var path: Path { get }
    }
}

/// A command to run during the build, including executable, command lines,
/// environment variables, initial working directory, etc. All paths should
/// be based on the ones passed to the plugin in the target build context.
enum Command {
    
    /// Creates a command to run during the build. The executable should be a
    /// tool returned by `TargetBuildContext.tool(named:)`, and any paths in
    /// the arguments list as well as in the input and output lists should be
    /// based on the paths provided in the target build context structure.
    ///
    /// The build command will run whenever its outputs are missing or if its
    /// inputs have changed since the command was last run. In other words,
    /// it is incorporated into the build command graph.
    ///
    /// This is the preferred kind of command to create when the outputs that
    /// will be generated are known ahead of time.
    static func buildCommand(
        /// An arbitrary string to show in build logs and other status areas.
        displayName: String,
        /// The executable to be invoked; should be a tool looked up using
        /// `tool(named:)`, which may reference either a binary-provided tool
        /// or a source-built tool.
        executable: Path,
        /// Arguments to be passed to the tool. Any paths should be based on
        /// the paths provided in the target build context.
        arguments: [String],
        /// Optional initial working directory of the command.
        workingDirectory: Path? = nil,
        /// Any custom settings for the environment of the subprocess.
        environment: [String: String] = [:],
        /// Input files to the build command. Any changes to the input files
        /// cause the command to be rerun.
        inputFiles: [Path] = [],
        /// Output files that should be processed further, according to the
        /// rules defined by the build system.
        outputFiles: [Path] = []
    ) -> Command

    /// Creates a command to run before the build. The executable should be a
    /// tool returned by `TargetBuildContext.tool(named:)`, and any paths in
    /// the arguments list and the output files directory should be based on
    /// the paths provided in the target build context structure.
    ///
    /// The build command will run before the build starts, and is allowed to
    /// create an arbitrary set of output files based on the contents of the
    /// inputs.
    ///
    /// Because prebuild commands are run on every build, they are can have
    /// significant performance impact and should only be used when there is
    /// no way to know the names of the outputs before the command is run.
    ///
    /// The `outputFilesDirectory` parameter is the path of a directory into
    /// which the command will write its output files. Any files that are in
    /// that directory after the prebuild command finishes will be interpreted
    /// according to same build rules as for sources.
    static func prebuildCommand(
        /// An arbitrary string to show in build logs and other status areas.
        displayName: String,
        /// The executable to be invoked; should be a tool looked up using
        /// `tool(named:)`, which may reference either a binary-provided tool
        /// or a source-built tool.
        executable: Path,
        /// Arguments to be passed to the tool. Any paths should be based on
        /// the paths provided in the target build context.
        arguments: [String],
        /// Optional initial working directory of the command.
        workingDirectory: Path? = nil,
        /// Any custom settings for the environment of the subprocess.
        environment: [String: String] = [:],
        /// A directory into which the command can write output files that
        /// should be processed further.
        outputFilesDirectory: Path
    ) -> Command
}

/// Emits errors, warnings, and remarks to be shown as a result of running
/// the plugin. If any errors are emitted, the plugin is considered to have
/// have failed, which will be reported to users during the build.
struct Diagnostics {
    /// Emits an error that is shown in SwiftPM or an IDE using it. If one
    /// or more errors are emitted by a plugin, it is considered to have
    /// failed to run.
    static func error(_ message: String, file: Path? = #file, line: Int? = #line)

    /// Emits a warning that is shown in SwiftPM or an IDE using it.
    static func warning(_ message: String, file: Path? = #file, line: Int? = #line)

    /// Emits a remark that may be shown in SwiftPM or an IDE using it.
    static func remark(_ message: String, file: Path? = #file, line: Int? = #line)

    /// Emits a diagnostic with the specified severity and descriptive message.
    static func emit(_ severity: Serverity, _ message: String, file: Path? = #file, line: Int? = #line)
    
    /// The seriousness with which the diagnostic is treated. An error causes
    /// SwiftPM to consider the plugin to have failed to run.
    enum Severity {
        case error, warning, remark
    }
}

/// Provides information about a list of files. The order is not defined
/// but is guaranteed to be stable. This allows the implementation to be
/// more efficient than a static file list.
protocol FileList: Sequence {
    func makeIterator() -> FileListIterator
}
struct FileListIterator: IteratorProtocol {
    mutating func next() -> FileInfo?
}

/// Provides information about a single file in a FileList.
protocol FileInfo {
    /// The absolute path in the local file system.
    var path: Path { get }

    /// The role of the file in SwiftPM.
    var type: FileType { get }
}

/// Provides information about the type of a file. Any future cases will
/// use availability annotations to make sure existing plugins still work
/// until they increase their required tools version.
enum FileType {
    /// A source file.
    case source
    /// A resource file (either processed or copied).
    case resource
    /// A file not covered by any other rule.
    case unknown
}

/// A simple representation of a path in the file system. This is aligned
/// with SwiftSystem.FilePath to minimize any changes if that is adopted
/// in the future.
protocol Path: ExpressibleByStringLiteral, CustomStringConvertible {
    /// A string representation of the path.
    public var string: String { get }
    
    /// The last path component (including any extension).
    public var lastComponent: String { get }
    /// The last path component (without any extension).
    public var stem: String { get }
    /// The filename extension, if any (without any leading dot).
    public var `extension`: String? { get }
    
    /// The path except for the last path component.
    public func removingLastComponent() -> Path
    /// The result of appending one or more path components.
    public func appending(_ other: [String]) -> Path
    /// The result of appending one or more path components.
    public func appending(_ other: String, ...) -> Path

  }
```

#### How SwiftPM Applies Plugins to Targets

During package graph resolution, all package dependencies and binary artifacts are fetched as usual. As part of resolving the package graph, SwiftPM then checks the syntax of any plugin scripts and reports any errors as diagnostics.

After package resolution but before building, SwiftPM applies build tool plugins to any target that has a `plugins` parameter in its declaration in the package manifest.

An error is reported if any of the plugins specified by a target cannot be found in the dependency closure of the target, or if they don't support the `buildTool` capability. The plugins are applied to the target in the order in which they are listed in `plugins`.

Applying a plugin to a target is done by executing the Swift script that implements it, passing inputs that describe the target to which the plugin is being applied, and interpreting its outputs as instructions for SwiftPM. The plugin is run — either by interpreting it or by compiling it to an executable and running it — in a sandbox that prevents network access and that restricts file system access.

SwiftPM passes input to the plugin in serialized form, which is decoded and made available through the `context` parameter to the plugin function.

Output from the plugin, in the form of created commands and diagnostics, is passed back to SwiftPM in a serialized form.

Each build tool plugin is invoked once for each target to which it is applied, with that target’s context as its input. Plugins that are defined but never used are not invoked at all. The command definitions emitted by a package plugin are used to set up build commands to run before or during a build.

If the plugin script throws an error or emits one or more diagnostics with an error severity, the invocation of the plugin is considered to have failed, and an error is reported to the user. Any console output emitted by the plugin script is shown as debug output.

It is important to emphasize the distinction between a package plugin and the build commands it defines:

- The *plugin* is invoked after the structure of the target that uses it is known, but before any build commands are run. It may also be invoked again if the target structure or other input conditions change. A build tool plugin is not invoked again when the *contents* of source files change.

- The *commands* that are returned by a plugin are invoked either before or during the build, in accordance with their defined input and output dependencies.

When SwiftPM applies a plugin to a particular target, it passes the path of a directory that is unique to that combination of target and plugin. The plugin and the commands it creates have write access to this work directory, and should use it for any intermediates or caches needed by the plugin or by the commands it creates. It should also write any derived sources inside this directory (or in subdirectories of it).

The plugin should specify any generated source files that should be fed back into the build system by providing their paths when creating the command. Any other files written to the output directory are considered a private implementation detail of the plugin. The contents are preserved between builds but are removed along with any other intermediates when the build directory is removed.

When SwiftPM invokes a plugin, it also passes information about any build tools on which the plugin has declared dependencies in the manifest. These build tools may be either executable targets in the package graph or may be binary targets containing prebuilt executables.

The `binaryTarget` type in SwiftPM has been improved to allow binaries to contain command line tools and required support files (such as the system `.proto` files in the case of `protoc`, etc). Binary targets will need to support different executable binaries based on platform and architecture. This is the topic of the separate evolution proposal [SE-0305](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0305-swiftpm-binary-target-improvements.md), which extends binary targets to support executables in addition to XCFrameworks.

#### Plugin Capabilities

Although this proposal defines a general approach to plugins, it specifies only one initial capability: `buildTool`. This capability is used for plugins that generate various kinds of build commands to run during a build.

Future proposals may extend the APIs to support new kinds of build commands for the `buildTool` capability, but the intent is also that entirely new kinds of capabilities can be defined in the future. New capabilities may focus on specific areas of functionality, such as source code formatters or linters, new types of unit tests, or actions that can be invoked on-demand to perform particular tasks.

Significantly different types of plugin functionality are likely to use different capabilities. They would also be coupled with a new specialization of the `Plugin` protocol in the `PackagePlugin` API to define the methods that should be implemented for that capability.

In the package manifest, the capability is expressed as a function invocation rather than an enum in order to make it easier to add arguments with default values in the future.

#### Types of Build Commands

The initial `PackagePlugin` API allows a plugin to define two different kinds of build commands: build commands and prebuild commands. They both involve invoking a build tool with a particular command line and environment, but the specific semantics of each make them suitable for different purposes.

Whenever the command is used for generating other files and when the names of the output files can be determined before the command is run, a regular build command should be used. This lets the command be incorporated into the build system and the outputs be processed according to the build system's rules.

Prebuild commands should be used when the tool being invoked can produce outputs whose names are determined by the _contents_ (not names) of the input files, or when there are other reasons why the names of the outputs cannot be known before actually running the command.

##### Build Commands

Commands of type `.buildCommand` that are retured by the plugin are incorporated into the build system's dependency graph, so that they run as needed during the build, based on their declared inputs and outputs. This requires that the paths of any outputs can be known before the command is run. This is usually done by forming the names of the outputs based on some combination of the output directory and the name of the input file.

Examples of plugins that can use regular build commands include compiler-like translators such as Protobuf and other tools that take a fixed set of inputs and produce a fixed set of outputs. (note that one nuance with Protobuf in particular is that it is actually up to the source generator invoked by `protoc` to determine the output paths — however, the relevant source generators for Swift and C do produce output files with predictable names).

Other examples include translators that "compile" data files in JSON or other editable formats to a suitable binary runtime respresentation.

Regular build commands with defined outputs are preferable whenever possible, because such commands don't have to run unless their outputs are missing or their inputs have changed since the last time they ran.

##### Prebuild Commands

Commands of type `.prebuildCommand` run before the start of every build. When creating prebuild commands, the plugin needs to specify a directory into which the command will write its output files. This is how the prebuild command communicates its outputs to the build system.

Before invoking a prebuild command, the build system will create the associated output directory if needed (but it will not remove any directory contents that already exist). After invoking the command, SwiftPM will use the contents of that directory as inputs to the construction of other build commands. The prebuild command should add or remove files so that the directory contents match the source files that should be processed by the build system. If the set of files in the directory has changed since the last time the prebuild command was run, the build system planning will be updated so that the changed file set is incorporated into the build.

Every plugin invocation is passed the path of a directory in `TargetBuildContext.pluginWorkDirectory`. A plugin would usually create a separate subdirectory of this directory for each prebuild command it creates, and the command would be configured to write its output files into that directory. The plugin may also create other subdirectories for cache files and other file system content that either it or the command needs. If these additional files need to be available to the command, the plugin would construct the command line to include their paths, for the command to use.

Examples of plugins that need to use prebuild commands include SwiftGen and other tools that need to see all the input files, and whose set of output files is determined by the *contents* (not just the paths) of the input files. Such a plugin usually generates just one command and configures it with an output directory into which all generated sources will be written.

Because they run on every build, prebuild commands should use caching to do as little work as possible (ideally none) when there are no changes to their inputs. This should include preserving the timestamps of generated source files whose contents haven't changed, since most build systems (including SwiftPM's own build system) use timestamps as a shorthand for detecting whether files have changed.

#### Using Command to Postprocess Built Artifacts

This proposal focuses on tools that run during the build, as part of constructing artifacts. A future proposal could define new plugin capabilities for more general actions that should take place at the very end of a build, after tests have completed.

Nonetheless it is sometimes useful to define commands that run at the end of the build in order to post-process the built artifacts. This may include modifying the contents of property lists or other data files in custom ways, or it might include modifying various properties of linked executables, etc.

Regular build commands can be used for this purpose if they are configured with artifacts inside the built-products directory as their inputs.

#### Plugin Errors

Any errors emitted by the build command will be handled by the build system in the same way as for other build commands. SwiftPM will show the output in its console output, and IDEs that use libSwiftPM will show it in the same way as it does for the other build commands.

Diagnostics from the plugin script itself can be reported using `Diagnostic()` APIs. Also, any error thrown from the main plugin function will be emitted as an error. In either case, if there are any errors, the plugin script will be considered to have failed, and SwiftPM and any IDEs that use libSwiftPM will emit them as it does with other configuration errors. In an IDE this would be similar to errors encountered during the rule matching of source file types, for example.

The script can use `print()` statements to emit debug output, which will be shown in SwiftPM's console and as detailed output text in the build logs of IDEs that use libSwiftPM.

## Example 1: SwiftGen

This example is a package that uses SwiftGen to generate source code for accessing resources. The package plugin can be defined in the same package as the one that provides the source generation tool (SwiftGen, in this case), so that client packages access it just by adding a package dependency on the SwiftGen package.

The `swiftgen` command may generate output files with any name, and they cannot be known without either running the command or separately parsing the configuration file. In this initial proposal for build plugins, this means that the SwiftGen plugin must construct a prebuild command in order for the source files it generates to be processed during the build.

#### Client Package

This is the structure of an example client package of SwiftGen:

```
MyPackage
 ├ Package.swift
 ├ swiftgen.yml
 └ Sources
    └ MyLibrary
       ├ Assets.xcassets
       └ SourceFile.swift
```


SwiftGen supports using a config file named `swiftgen.yml` and this example implementation of the plugin assumes a convention that it is located in the package directory. A different plugin implementation might instead use a per-target convention, or might use a combination of the two.

The package manifest has a dependency on the SwiftGen package, which vends a plugin product that the client package can use for any of its targets by adding a `plugins` parameter to the package manifest:

```swift
// swift-tools-version: 999.0
import PackageDescription

let package = Package(
    name: "MyPackage",
    dependencies: [
        .package(url: "https://github.com/SwiftGen/SwiftGen", from: "6.4.0")
    ]
    targets: [
        .executable(
            name: "MyLibrary",
            plugins: [
                .plugin(name: "SwiftGenPlugin", package: "SwiftGen")
            ]
        )
    ]
)
```

By specifying that the `MyLibrary` target uses the plugin, `SwiftGenPlugin` will be invoked for the target.

The order in which plugins are listed in `plugin` determines the order in which they will be applied compared with other plugins in the list that provide the same capability. This can be used to control which plugins will see the outputs of which other plugins. In this case, only one plugin is used.

#### Plugin Package

Using the facilities in this proposal, the SwiftGen package authors could implement a package plugin that creates a command to run `swiftgen` before the build.

This is the hypothetical `SwiftGenPlugin` target referenced in the client package:

```
SwiftGen
 ├ Package.swift
 ├ Plugins
 │  └ SwiftGenPlugin
 │     └ plugin.swift
 └ Sources
    └ SwiftGen
       └ ...
```

In this case, `plugin.swift` is the Swift script that implements the package plugin target. The plugin is treated as a Swift executable, and can consist of one or more Swift source files, once of which contains a type attributed with `@main`.

The package manifest would have a `plugin` target in addition to the existing target that provides the `swiftgen` command line tool itself:

```swift
// swift-tools-version: 5.6
import PackageDescription

let package = Package(
    name: "SwiftGen",
    targets: [
        /// Package plugin that tells SwiftPM how to run `swiftgen` based on
        /// the configuration file. Client targets use this plugin by listing
        /// it in their `plugins` parameter.
        .plugin(
            name: "SwiftGenPlugin",
            capability: .buildTool(),
            dependencies: ["SwiftGen"]
        ),
        
        /// Binary target that provides the built SwiftGen executables.
        .binaryTarget(
            name: "SwiftGen",
            url: "https://url/to/the/built/swiftgen-executables.zip",
            checksum: "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"
        ),
    ]
)
```

The package plugin script might look like this:

```swift
import PackagePlugin

@main struct SwiftGenPlugin: BuildToolPlugin {
    /// This plugin's implementation returns a single `prebuild` command to run `swiftgen`.
    func createBuildCommands(context: TargetBuildContext) throws -> [Command] {
        // This example configures `swiftgen` to take inputs from a `swiftgen.yml` file
        let swiftGenConfigFile = context.packageDirectory.appending("swiftgen.yml")
        
        // This example configures the command to write to a "GeneratedSources" directory.
        let genSourcesDir = context.pluginWorkDirectory.appending("GeneratedSources")

        // Return a command to run `swiftgen` as a prebuild command. It will be run before
        // every build and generates source files into an output directory provided by the
        // build context. This example sets some environment variables that `swiftgen.yml`
        // bases its output paths on.
        return [.prebuildCommand(
            displayName: "Running SwiftGen",
            executable: try context.tool(named: "swiftgen").path,
            arguments: [
                "config", "run",
                "--config", "\(swiftGenConfigFile)"
            ],
            environment: [
                "PROJECT_DIR": "\(context.packageDirectory)",
                "TARGET_NAME": "\(context.targetName)",
                "DERIVED_SOURCES_DIR": "\(genSourcesDir)",
            ],
            outputFilesDirectory: genSourcesDir)]
    }
}
```

An alternate use of `swiftgen` could instead invoke it once for each input file, passing it output files whose names are derived from the names of the input files. This might, however, make per-file configuration somewhat more difficult.

There is a trade-off here between implementing a prebuild command or a regular build command. Future improvements to SwiftPM's build system — and to those of any IDEs that support Swift packages — could let it support commands whose outputs aren't known until it is run. That would allow the use of regular build commands to generate output files whose names aren't know until the command is known.

Possibly, the `swiftgen` tool itself could also be modified to provide a simplified way to invoke it, to take advantage of SwiftPM's new ability to dynamically provide the names of the input files in the target.

## Example 2: SwiftProtobuf

This example is a package that uses SwiftProtobuf to generate source files from `.proto` files. In addition to the package plugin product, the package provides the runtime library that the generated Swift code uses.

Since `protoc` isn’t built using SwiftPM, it also has a binary target with a reference to a `zip` archive containing the executable.

#### Client Package

This is the structure of an example package that uses Protobuf:

```
MyPackage
 ├ Package.swift
 └ Sources
    └ MyExe
       ├ messages.proto
       └ main.swift
```

The `messages.proto` source file needs to be processed using the `protoc` compiler to generate Swift source files that are then compiled. In addition, the `protoc` compiler needs to be passed the path of a source generator plug-in built by the SwiftProtobuf package.

The package manifest has a dependency on the `SwiftProtobuf` package, and references the hypothetical new `SwiftProtobuf` plugin defined in it:

```swift
// swift-tools-version: 5.6
import PackageDescription

let package = Package(
    name: "MyPackage",
    dependencies: [
        .package(url: "https://github.com/apple/swift-protobuf", from: "1.15.0")
    ]
    targets: [
        .executable(
            name: "MyExe",
            dependencies: [
                .product(name: "SwiftProtobufRuntimeLib", package: "swift-protobuf")
            ],
            plugins: [
                .plugin(name: "SwiftProtobuf", package: "swift-protobuf")
            ]
        )
    ]
)
```

As with the previous example, listing the plugin in the `plugins` parameter applies that plugin to the `MyExe` target. As with product dependencies, the package name needs to be provided when the plugin is defined in a different package.

In this initial version of the proposal, the client target must also specify dependencies on any runtime libraries that will be needed, as this example shows with the hypothetical `SwiftProtobufRuntimeLib` library. A future improvement could extend the `PackagePlugin` API to let the plugin define additional dependencies that targets using the plugin would automatically get.

This version of the initial proposal does not define a way to pass options to the plugin through the manifest. Because the plugin has read-only access to the package directory, it can define its own conventions for a configuration file in the package or target directory. A future improvement to the proposal should allow a way for the plugin to provide custom types that the client package manifest could use to specify options to the plugin. This is described in more detail under _Future Directions_, below.

#### Plugin Package

The structure of the hypothetical `SwiftProtobuf` target that provides the plugin is:

```
SwiftProtobuf
 ├ Package.swift
 ├ Plugins
 │  └ SwiftProtobuf
 │     └ plugin.swift     
 └ Sources
    ├ SwiftProtobufRuntimeLib
    │  └ ...
    └ protoc-gen-swift
       └ ...
```

The package manifest is:

```swift
// swift-tools-version: 5.6
import PackageDescription

let package = Package(
    name: "SwiftProtobuf",
    targets: [
        /// Package plugin that tells SwiftPM how to deal with `.proto` files.
        .plugin(
            name: "SwiftProtobuf",
            capability: .buildTool(),
            dependencies: ["protoc", "protoc-gen-swift"]
        ),
                
        /// Binary target that provides the prebuilt `protoc` executable.
        .binaryTarget(
            name: "protoc",
            url: "https://url/to/the/built/protoc-executables.zip",
            checksum: "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"
        ),
        
        /// Swift target that builds the plug-in executable that will be passed to
        /// the `protoc` compiler.
        .executableTarget(
            name: "protoc-gen-swift"
        ),

        /// Runtime library that clients will need to link against by specifying
        /// it as a dependency.
        .libraryTarget(
            name: "SwiftProtobufRuntimeLib"
        ),
    ]
)
```

The `protoc-gen-swift` target is a regular executable target that implements a source generation plug-in to the Protobuf compiler. This is particular to the Protobuf compiler, which in addition to the `protoc` executable uses separate source code generator tools specific to the emitted languages. The implementation of the hypothetical `SwiftProtobuf` plugin generates the commands to invoke `protoc` passing it the `protoc-gen-swift` plug-in.

The package plugin script might look like:

```swift
import PackagePlugin
import Foundation

@main struct MyPlugin: BuildToolPlugin {
    /// This plugin's implementation returns multiple build commands, each of which
    /// calls `protoc`.
    func createBuildCommands(context: TargetBuildContext) throws -> [Command] {
        // In this case we generate an invocation of `protoc` for each input file,
        // passing it the path of the `protoc-gen-swift` generator tool.
        let protocTool = try context.tool(named: "protoc")
        let protocGenSwiftTool = try context.tool(named: "protoc-gen-swift")
        
        // Construct the search paths for the .proto files, which can include any
        // of the targets in the dependency closure. Here we assume that the public
        // ones are in a `protos` directory, but this can be made arbitrarily complex.
        var protoSearchPaths = context.dependencies.map { target in
            target.targetDirectory.appending("protos")
        }
        
        // This example configures the commands to write to a "GeneratedSources"
        // directory.
        let genSourcesDir = context.pluginWorkDirectory.appending("GeneratedSources")
        
        // This example uses a different directory for other files generated by
        // the plugin.
        let otherFilesDir = context.pluginWorkDirectory.appending("OtherFiles")
        
        // Add the search path to the system proto files. This sample implementation
        // assumes that they are located relative to the `protoc` compiler provided
        // by the binary target, but real implementation could be more sophisticated.
        protoSearchPaths.append(protocTool.path.removingLastComponent().appending("system-protos"))
        
        // Create a module mappings file. This is something that the Swift source
        // generator `protoc` plug-in we are using requires. The details are not
        // important for this proposal, except that it needs to be able to be con-
        // structed from the information in the context given to the plugin, and
        // to be written out to the intermediates directory.
        let moduleMappingsFile = otherFilesDir.appending("module-mappings")
        let outputString = ". . . module mappings file . . ."
        let outputData = outputString.data(using: .utf8)
        FileManager.default.createFile(atPath: moduleMappingsFile.string, contents: outputData)
        
        // Iterate over the .proto input files, creating a command for each.
        let inputFiles = context.inputFiles.filter { $0.path.extension == "proto" }
        return inputFiles.map { inputFile in            
            // The name of the output file is based on the name of the input file,
            // in a way that's determined by the protoc source generator plug-in
            // we're using.
            let outputName = inputFile.path.stem + ".swift"
            let outputPath = genSourcesDir.appending(outputName)
            
            // Specifying the input and output paths lets the build system know
            // when to invoke the command.
            let inputFiles = [inputFile.path]
            let outputFiles = [outputPath]

            // Construct the command arguments.
            var commandArgs = [
                "--plugin=protoc-gen-swift=\(protocGenSwiftTool.path)",
                "--swift_out=\(genSourcesDir)",
                "--swift_opt=ProtoPathModuleMappings=\(moduleMappingsFile)"
            ]
            commandArgs.append(contentsOf: protoSearchPaths.flatMap { ["-I", "\($0)"] })
            commandArgs.append("\(inputFile.path)")

            // Append a command containing the information we generated.
            return .buildCommand(
                displayName: "Generating \(outputName) from \(inputFile.path.stem)",
                executable: protocTool.path,
                arguments: commandArgs,
                inputFiles: inputFiles,
                outputFiles: outputFiles)
        }
    }
}
```

In this case, the script iterates over the input files that have a `.proto` suffix, creating a command to invoke the Protobuf compiler for each one.

## Example 3: Source Analyzer

A third important use case is source generators that analyze Swift files and generate some additional sources based on the Swift definitions found in them. Tool authors may use a package such as [Sourcery](https://github.com/krzysztofzablocki/Sourcery), or parse the sources using [SwiftSyntax](https://github.com/apple/swift-syntax) to generate some boilerplate code, in order to avoid having to maintain it manually.

One could imagine a source generation tool called `GenSwifty` generating some additional “sugar” for existing definitions in a swift target. It would be configured like this by end users:

```swift
// swift-tools-version: 5.6
import PackageDescription

let package = Package(
    name: "MyPackage",
    dependencies: [
        .package(url: "https://github.com/example/gen-swifty", from: "1.0.0")
    ]
    targets: [
        .executable(
            name: "MyExe",
            plugins: [
                .plugin(name: "GenSwifty", package: "gen-swifty")
            ]
        )
    ]
)
```

The package manifest of the `gen-swifty` package would be as follows:

```swift
// swift-tools-version: 5.6
import PackageDescription

let package = Package(
    name: "gen-swifty",
    targets: [
        .plugin(
            name: "GenSwifty",
            capability: .buildTool(),
            dependencies: ["GenSwiftyTool"]
        ),
        .executable(
            name: "GenSwiftyTool",
            ...  // this is the target that builds the tool
                 // The tool can depend on swift-syntax or other tools
        ),
    ]
)
```

The implementation of the package plugin would be similar to the previous examples, but with a somewhat differently formed command line.

## Example 4: Custom Source Generator

This example uses a custom source generator implemented in the same package as the target that uses it.

```
MyPackage
 ├ Package.swift
 ├ Plugins
 │  └ MySourceGenPlugin
 │     └ plugin.swift     
 └ Sources
    ├ MyExe
    │   │ file.dat
    │   └ main.swift
    └ MySourceGenTool
        └ main.swift
```

The manifest is:

```swift
// swift-tools-version: 5.6
import PackageDescription

let package = Package(
    name: "MyPackage",
    targets: [
        .executable(
            name: "MyExe",
            plugins: [
                .plugin("MySourceGenPlugin")
            ]
        ),
        .plugin(
            name: "MySourceGenPlugin",
            capability: .buildTool(),
            dependencies: ["MySourceGenTool"]
        ),
        .executableTarget(
            name: "MySourceGenTool"
        ),
    ]
)
```

In this case, the `.plugin()` expression in the `plugin` parameter refers to a plugin target defined in the same package, so it does not need a `package` parameter and no product needs to be defined.

The implementation of the package plugin script would be similar to the previous examples, but with a somewhat differently formed command line to transform the `.dat` file into something else.

## Impact on Existing Packages

The new API and functionality will only be available to packages that specify a tools version equal to or later than the SwiftPM version in which this functionality is implemented, so there will be no impact on existing packages.

## Security

Package plugins will be run in a sandbox that prevents network access and restricts writing to the file system to specific intermediate directories. This is the same sandbox that manifest parsing uses, except that it allows writing to a limited set of output directories.

In addition, SwiftPM and IDEs that use libSwiftPM should run each command generated by an plugin in a sandbox. This sandbox should prevent all network access, and should only allow writing to the directories specified as outputs by the plugin.

There is inherent risk in running build tools provided by other packages. This can possibly be mitigated by requiring the root package to list the approved package plugins, no matter where they are in the graph. This requires further consideration, and would be a subject of a future proposal.

## Future Directions

This proposal is intentionally fairly basic and leaves many improvements for the future. Among them are:

- the ability for a build tool to have a separate dependency graph so that it can use different versions of the same packages as those used by the client of the build tool
- the ability for plugins to have access to the full build graph at a detailed level
- the ability for prebuild actions to depend on tools built by SwiftPM
- the ability for package plugin scripts to use libraries provided by other targets
- the ability for build commands to emit output files that feed back into the rule system to generate new work during the build (this requires support in SwiftPM's native build system as well as in the build systems of some IDEs that use libSwiftPM)
- the ability to provide per-target type-safe options to a use of an plugin (details below)
- the ability to define commands that run after building and testing is complete
- specific support for testing plugin scripts (they can currently be tested using test fixtures that exercise the plugins, but it would be useful to be able to invoke them directly, for example through a `swift package` command that invokes the plugin with specific inputs)


### Type-Safe Options

We are aware that many plugins will want to allow specific per-target configuration which would be best done in the package manifest of the package that uses the plugin.

We are purposefully leaving options out of this initial proposal, and plan to revisit and add these in a future proposal.

In theory, options could just be done as a dictionary of string key-value pairs, like this:

```swift
// NOT proposed
.plugin(name: "Foo", options: ["Visibility": "Public"]) // not type-safe!
```

However, we believe that this results in a suboptimal user experience. It is hard to know what the available keys are, and what values are accepted. Is only `"Public"` correct in this example, or would `"public"` work too?

We would therefore prefer to exclude plugin options from this proposal and explore a type-safe approach to options in a future proposal.

We would especially like to allow plugins to define some form of `struct MyOptions: PluginOptions` type, where `PluginOptions` is also `Codable`. SwiftPM could then take care of serializing these options and providing them to the plugin.

This would allow something like this:

```swift
.plugin(name: "Foo", options: FooOptions(visibility: .public)) // type-safe!
```

Such a design is difficult to implement well, because it would require the plugin to add a type that is accessible to the package package manifest. It would also mean that the package manifest couldn't be parsed at all until the plugin module that provides the types has been compiled.

Designing such type safe options is out of scope for this initial proposal, as it involves many complexities with respect to how the types would be made available from the plugin definition to the package manifest that needs them.

It is an area we are interested in exploring and improving in the near future, so rather than lock ourselves into supporting untyped dictionaries of strings, we suggest to introduce target specific, type safe plugin options in a future Swift evolution proposal.

### Separate Dependency Graphs for Build Tools

This proposal uses the existing facilities that SwiftPM provides for declaring dependencies on packages that provide plugins and tools. This keeps the scope of the proposal bounded enough to allow it to be implemented in the near term, but it does mean that it does not allow there to be any conflicts between the set of package dependencies that the build tool needs compared with those that the client package needs.

For example, a situation in which a build tool needs [Yams](https://github.com/jpsim/Yams) v3.x while a package client needs v4.x would not be supported by this proposal, even though it would pose no real problem at runtime. It would be very desirable to support this, but that would require significant work on SwiftPM to allow it to keep multiple independent package graphs. It should be noted that if a package vends both a build tool and a runtime library that clients using the build tool must link against, then even this apparently simple case would get more complicated. In this case the runtime library would have to use Yams v4.x in order to be usable by the client package, even if the tool itself used Yams v3.x.

It should also be noted that this is no different from a package that defines an executable and a library and would like to use a particular version of Yams without requiring the client of the library to use a compatible version. This is not supported in SwiftPM today either, and it is in fact another manifestation of the same problem.

Solving this mixture of build-time and run-time package dependencies is possible but not without significant work in SwiftPM (and IDEs that use libSwiftPM). We think that this proposal could be extended to support declaring such dependencies in the future, and that even with this restriction, this proposal provides useful functionality for packages.

### A More Sophisticated `Path` Type

The current `Path` type provided in the `PackagePlugin` API is intentionally kept minimal, and should be sufficient for the needs of most plugins. A future version of this proposal would extend this type, possibly aligning it with `SwiftSystem`'s `FilePath` type. The initial API has purposefully been kept the same as `FilePath` to make such a transition easier.

An alternate direction would be to replace it with a more domain-specific representation that could also keep track of the path root in relation to the directories that matter to packages and build systems, avoiding the need to form absolute paths.

Since the API that is available to the plugin script is based on the tools version of the package that contains it, existing plugin scripts are expected to be able to run without modification even if the API does change.

### Contextual Information About the Target Platform

The current `TargetBuildContext` type in the `PackagePlugin` API provides minimal information about the structure and configuration of the target, but it does not in this initial proposal provide any information about the target platform for which the package is being built.

This would be needed in order to implement more advanced tools such as code generators or linkers.

### Specific Support for Code Linters and Formatters

A future proposal should add specific support for code linters. In particular, there should be a way for build tools to convey fixits and other mechanical editing instructions to SwiftPM or to an IDE that uses libSwiftPM.

One approach would be to use the existing Clang diagnostics file format, possibly together with a library making it easy to generate such files, and to extend the `PackagePlugin` API to allow the plugin to configure commands to emit this information in a way that SwiftPM and IDEs can apply it. Such a capability could also be useful for build tools such as source translators, if they want to be able to apply fixits to their input files.

Code formatters (which typically modify the source code directly) should probably be supported using a new plugin capability that allows some specific action that a package author can take to run the formatter on their code, since it seems a bit subtle to allow source code to be modified as a side effect of a regular build action.

### Postbuild Commands

This proposal allows regular build commands to be configured to act on artifacts after they are built, but there is a greater need for a more general kind of postbuild command that can run after building and testing are complete for all the targets in the build graph.

A future proposal should extend plugins to support commands that can be run after both building and testing is complete, possibly using a new plugin capability and additional `PackagePlugin` APIs.  Such commands would be able to report on the results of the build and testing, to post notifications, etc.

Many of the most useful cases need information about the results of the build and the testing. This includes detailed information about what was built, how it was built, and what the results were. This could include structured diagnostics, code coverage information, etc.

A future proposal should define a structured format through which SwiftPM communicates this kind of information to postbuild commands, and should ideally provide a library with API that allows such a tool to load and query this structure.

## Alternatives Considered

A simpler approach would be to allow a package to contain shell scripts that are unconditionally invoked during the build. In order to support even moderately complex uses, however, there would still need to be some way for the script to get information about the target being built and its dependencies, and to know where to write output files.

This information would need to be passed to the script through some means, such as command line flags or environment variables, or through file system conventions. This seems a more subtle and less clear approach than providing a Swift API to access this information. If shell scripts are needed for some use cases, it would be fairly straightforward for a package author to write a custom plugin to invoke that shell script and pass it inputs from the build context using either command line arguments or environment variables, as it sees fit.

Even with an approach based on shell scripts, there would still need to be some way of defining the names of the output files ahead of time in order for SwiftPM to hook them into its build graph (needing to know the names of outputs ahead of time is a limitation of the native build system that SwiftPM currently uses, as well as of some of the IDEs that are based on libSwiftPM — these build systems currently have no way to apply build rules to the output of commands that are run in the same build).

Defining these inputs and outputs using an API seems clearer than some approach based on naming conventions or requiring output files to be listed in the client package’s manifest.

Another approach would be to delay providing extensible build tools until the build systems of SwiftPM and of IDEs using it can be reworked to support arbitrary discovery of new work. This would, however, delay these improvements for the various kinds of build tools that *can* take advantage of the support proposed here.

We believe that the approach taken here — defining a common approach for specifying plugins that declare the capabilities they provide, and with a goal of defining more advanced capabilities over time — will provide some benefit now while still allowing more sophisticated behavior in the future.

## References

- https://github.com/aciidb0mb3r/swift-evolution/blob/extensible-tool/proposals/NNNN-package-manager-extensible-tools.md
- https://forums.swift.org/t/package-manager-extensible-build-tools/10900
