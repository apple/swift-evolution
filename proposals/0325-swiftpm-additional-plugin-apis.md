# Additional Package Plugin APIs

* Proposal: [SE-0325](0325-swiftpm-additional-plugin-apis.md)
* Authors: [Anders Bertelrud](https://github.com/abertelrud)
* Review Manager: [Tom Doron](https://github.com/tomerd)
* Status: **Implemented (Swift 5.6)**
* Implementation: [apple/swift-package-manager#3758](https://github.com/apple/swift-package-manager/pull/3758)
* Pitch: [Forum discussion](https://forums.swift.org/t/pitch-additional-api-available-to-swiftpm-plugins/)
* Review: [Forum discussion](https://forums.swift.org/t/se-0325-additional-package-plugin-apis/)

## Introduction

[SE-0303](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0303-swiftpm-extensible-build-tools.md) introduced the ability to define *build tool plugins* in SwiftPM, allowing custom tools to be invoked while building a package. In support of this, SE-0303 introduced a minimal initial API through which plugins can access information about the target for which they are invoked.

This proposal extends the plugin API to provide more context, including a richer representation of the package graph. This is in preparation for supporting new kinds of plugins in the future.

## Motivation

The build tool plugin support introduced in SE-0303 is focused on code generation during a build of a package, for such purposes as generating Swift source files from `.proto` files or other inputs. The initial API provided to plugins was oriented toward that task, and was purposefully kept minimal in order to keep the scope of the proposal bounded.

New kinds of plugins that are being discussed will require a richer context. In particular, providing a distilled form of the whole package graph would allow for a wide variety of new kinds of plugins.

## Proposed Solution

This proposal extends the plugin API that was introduced in SE-0303 by defining a generic `PluginContext` structure that supersedes `TargetBuildContext`. This new structure provides a distilled form of the resolved package graph as seen by SwiftPM, with information about all the products and targets therein.

This is the same structure that SwiftPM’s built-in subsystems currently use, and the intent is that, over time, at least some of those subsystems can be reimplemented as plugins. This information is also expected to be useful to various kinds of other plugins, provided by external packages.

In addition to the new information, this proposal adds API for traversing the package graph, such as being able to access topologically sorted lists of target dependencies. This will make it more convenient for build tool plugins that, for example, need to generate command line arguments that include a search paths for each dependency target.

## Detailed Design

This proposal defines a new `PluginContext` structure that contains:

* a reference to the `Package` at the root of the subgraph to which the plugin is being applied
* the contextual information that was previously part of `TargetBuildContext`

This structure factors out all the information related to the package graph, such as the package and target names and directories, leaving the context with just the top-level contextual information.

The `BuildToolPlugin` protocol entry point defined by SE-0303 is superseded by a new entry point that takes the new `PluginContext` type and a reference to the `Target` for which build commands should be generate. The previous API remains so that existing plugins continue to work.

### Plugin API

The new `PluginContext` structure in the `PackagePlugin` API is defined like this:

```swift
/// Provides information about the package for which the plugin is invoked,
/// as well as contextual information based on the plugin's stated intent
/// and requirements.
public struct PluginContext {
    /// Information about the package to which the plugin is being applied,
    /// and any other package reachable from it.
    public let package: Package

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
    public let pluginWorkDirectory: Path

    /// Looks up and returns the path of a named command line executable tool.
    /// The executable must be provided by an executable target or a binary
    /// target on which the package plugin target depends. This function throws
    /// an error if the tool cannot be found. The lookup is case sensitive.
    public func tool(named name: String) throws -> Tool
    
    /// Information about a particular tool that is available to a plugin.
    public struct Tool {
        /// Name of the tool (suitable for display purposes).
        public let name: String

        /// Full path of the built or provided tool in the file system.
        public let path: Path
    }
}
```

The `package` property is a reference to the package to which the plugin is being applied. Through it, the script that implements the plugin can reach the entire subgraph of resolved packages on which it either directly or indirectly depends. Note that this might only constitute part of the package graph, if the plugin is being applied to a package other than the root package of the whole graph SwiftPM sees.

The function and structure definition that relates to looking up tools with a particular name are unchanged from the original SE-0303 proposal.

This has the effect of factoring out all information related to the package and target, and it puts them into its own directed acyclic graph consisting of the following types:

```swift
/// Represents a single package in the graph (either the root or a dependency).
public protocol Package {
    /// Opaque package identifier, unique among the packages in the graph.
    var id: ID { get }
    typealias ID = String
    
    /// The name of the package (for display purposes only).
    var displayName: String { get }

    /// The absolute path of the package directory in the local file system,
    /// regardless of the original provenance of the package.
    var directory: Path { get }
  
    /// The origin of the package (root, local, repository, registry, etc).
    var origin: PackageOrigin { get }

    /// The tools version specified by the resolved version of the package.
    /// Behavior is often gated on the tools version, to make sure older
    /// packages continue to work as intended.
    var toolsVersion: ToolsVersion { get }
  
    /// Any dependencies on other packages, in the same order as they are
    /// specified in the package manifest.
    var dependencies: [PackageDependency] { get }

    /// Any regular products defined in this package (except plugin products),
    /// in the same order as they are specified in the package manifest.
    var products: [Product] { get }

    /// Any regular targets defined in this package (except plugin targets),
    /// in the same order as they are specified in the package manifest.
    var targets: [Target] { get }
}

/// Represents the origin of a package in the graph.
public enum PackageOrigin {
    /// A root package (unversioned).
    case root

    /// A local package, referenced by path (unversioned).
    case local(path: String)

    /// A package from a Git repository, with a URL and with a textual
    /// description of the resolved version or branch name (for display
    /// purposes only), along with the corresponding SCM revision. The
    /// revision is the Git commit hash and may be useful for plugins
    /// that generates source code that includes version information.
    case repository(url: String, displayVersion: String, scmRevision: String)

    /// A package from a registry, with an identity and with a textual
    /// description of the resolved version or branch name (for display
    /// purposes only).
    case registry(identity: String, displayVersion: String)
}

/// Represents a version of SwiftPM on whose semantics a package relies.
public struct ToolsVersion: CustomStringConvertible, Comparable {
    /// The major version.
    public let major: Int

    /// The minor version.
    public let minor: Int

    /// The patch version.
    public let patch: Int
}

/// Represents a resolved dependency of a package on another package. Other
/// information in addition to the resolved package is likely to be added
/// to this struct in the future.
public struct PackageDependency {
    /// A description of the dependency as declared in the package (intended
    /// for display purposes only).
    public let description: String
    
    /// The package to which the dependency was resolved.
    public let package: Package
}

/// Represents a single product defined in a package.
public protocol Product {
    /// Opaque product identifier, unique among the products in the graph.
    var id: ID { get }
    typealias ID = String

    /// The name of the product, as defined in the package manifest. This name
    /// is unique among the products in the package in which it is defined.
    var name: String { get }
    
    /// The targets that directly comprise the product, in the order in which
    /// they are declared in the package manifest. The product will contain the
    /// transitive closure of the these targets and their depdendencies.
    var targets: [Target] { get }
}

/// Represents an executable product defined in a package.
public struct ExecutableProduct: Product {
    /// The target that contains the main entry point of the executable. Every
    /// executable product has exactly one main executable target. This target
    /// will always be one of the targets that is also included in the product's
    /// `targets` list.
    public let mainTarget: Target
}

/// Represents a library product defined in a package.
public struct LibraryProduct: Product {
    /// Whether the library is static, dynamic, or automatically determined.
    public let kind: Kind

    /// Represents a kind of library product.
    public enum Kind {
        /// A static library, whose code is copied into its clients.
        case `static`

        /// Dynamic library, whose code is referenced by its clients.
        case `dynamic`

        /// The kind of library produced is unspecified and will be determined
        /// by the build system based on how the library is used.
        case automatic
    }
}

/// Represents a single target defined in a package.
public protocol Target {
    /// Opaque target identifier, unique among the targets in the graph.
    var id: ID { get }
    typealias ID = String

    /// The name of the target, as defined in the package manifest. This name
    /// is unique among the targets in the package in which it is defined.
    var name: String { get }
    
    /// The absolute path of the target directory in the local file system.
    var directory: Path { get }
    
    /// Any other targets on which this target depends, in the same order as
    /// they are specified in the package manifest. Conditional dependencies
    /// that do not apply have already been filtered out.
    var dependencies: [Dependency] { get }
}

/// Represents a dependency of a target on a product or on another target.
public enum TargetDependency {
    /// A dependency on a target in the same package.
    case target(Target)

    /// A dependency on a product in another package.
    case product(Product)
}

/// Represents a target consisting of a source code module, containing either
/// Swift or source files in one of the C-based languages.
public protocol SourceModuleTarget: Target {
    /// The name of the module produced by the target (derived from the target
    /// name, though future SwiftPM versions may allow this to be customized).
    public let moduleName: String

    /// The source files that are associated with this target (any files that
    /// have been excluded in the manifest have already been filtered out).
    public let sourceFiles: FileList

    /// Any custom linked libraries required by the module, as specified in
    /// the package manifest.
    public let linkedLibraries: [String]

    /// Any custom linked frameworks required by the module, as specified in
    /// the package manifest.
    public let linkedFrameworks: [String]
}

/// Represents a target consisting of a source code module compiled using Swift.
public struct SwiftSourceModuleTarget: SourceModuleTarget {
    /// Any custom compilation conditions specified for the Swift target in
    /// the package manifest.
    public let compilationConditions: [String]
}

/// Represents a target consisting of a source code module compiled using Clang.
public struct ClangSourceModuleTarget: SourceModuleTarget {
    /// Any preprocessor definitions specified for the Clang target.
    public let preprocessorDefinitions: [String]
    
    /// Any custom header search paths specified for the Clang target.
    public let headerSearchPaths: [Path]

    /// The directory containing public C headers, if applicable. This will
    /// only be set for targets that have a directory of a public headers.
    public let publicHeadersDirectory: Path?
}

/// Represents a target describing an artifact (e.g. a library or executable)
/// that is distributed as a binary.
public struct BinaryArtifactTarget: Target {
    /// The kind of binary artifact.
    public let kind: Kind
    
    /// The original source of the binary artifact.
    public let origin: Origin
    
    /// The location of the binary artifact in the local file system.
    public let artifact: Path

    /// Represents a kind of binary artifact.
    public enum Kind {
        /// Represents a .xcframework directory containing frameworks for
        /// one or more platforms.
        case xcframework
      
        /// Represents a .artifactsarchive directory containing SwiftPM
        /// multiplatform artifacts.
        case artifactsArchive
    }
	
    // Represents the original location of a binary artifact.
    public enum Origin: Equatable {
        /// Represents an artifact that was available locally.
        case local

        /// Represents an artifact that was downloaded from a remote URL.
        case remote(url: String)
    }
}

/// Represents a target describing a system library that is expected to be
/// present on the host system.
public struct SystemLibraryTarget: Target {
    /// The name of the `pkg-config` file, if any, describing the library.
    public let pkgConfig: String?

    /// Flags from `pkg-config` to pass to Clang (and to SwiftC via `-Xcc`).
    public let compilerFlags: [String]
  
    /// Flags from `pkg-config` to pass to the platform linker.
    public let linkerFlags: [String]
}

/// Provides information about a list of files. The order is not defined
/// but is guaranteed to be stable. This allows the implementation to be
/// more efficient than a static file list.
public protocol FileList: Sequence {
    func makeIterator() -> FileListIterator
}
public struct FileListIterator: IteratorProtocol {
    mutating public func next() -> File?
}

/// Provides information about a single file in a FileList.
public struct File {
    /// The path of the file.
    public let path: Path
    
    /// File type, as determined by SwiftPM.
    public let type: FileType
}

/// Provides information about the type of a file. Any future cases will
/// use availability annotations to make sure existing plugins still work
/// until they increase their required tools version.
public enum FileType {
    /// A source file.
    case source

    /// A header file.
    case header

    /// A resource file (either processed or copied).
    case resource

    /// A file not covered by any other rule.
    case unknown
}
```

Note that the `Target` and `Product` types are defined using protocols, with structing implementing the specific types of targets and products. Although they define unique identifiers, they do not in this proposal conform to `Identifiable`, since that would introduce `Self` requirements on the protocol that makes it difficult to have heterogeneous collections of targets and products, as the SwiftPM package model has.  This should be alleviated via [SE-0309](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0309-unlock-existential-types-for-all-protocols.md). The `ID` type alias and `id` property should make it easy to conform these protocols to `Identifiable` in the future without affecting existing plugins.

The `BuildToolPlugin` is extended with the following new entry point that takes the new, more general context and a direct reference to the target for which build commands should be created:

```swift
/// Defines functionality for all plugins having a `buildTool` capability.
public protocol BuildToolPlugin: Plugin {
    /// Invoked by SwiftPM to create build commands for a particular target.
    /// The context parameter contains information about the package and its
    /// dependencies, as well as other environmental inputs.
    ///
    /// This function should create and return build commands or prebuild
    /// commands, configured based on the information in the context. Note
    /// that it does not directly run those commands.
    func createBuildCommands(
        context: PluginContext,
        target: Target
    ) throws -> [Command]
}
```

The previous entry point remains, and a default implementation of the new one calls through to the old one. Its implementation is this, which also provides an example of using the new API and how it maps to the old one:

```swift
extension BuildToolPlugin {
    /// Default implementation that invokes the old callback with an old-style
    /// context, for compatibility.
    public func createBuildCommands(
        context: PluginContext,
        target: Target
    ) throws -> [Command] {
        return try self.createBuildCommands(context: TargetBuildContext(
            targetName: target.name,
            moduleName: (target as? SourceModuleTarget)?.moduleName ?? target.name,
            targetDirectory: target.directory,
            packageDirectory: context.package.directory,
            inputFiles: (target as? SourceModuleTarget)?.sourceFiles ?? .init([]),
            dependencies: target.recursiveTargetDependencies.map { .init(
                targetName: $0.name,
                moduleName: ($0 as? SourceModuleTarget)?.moduleName ?? $0.name,
                targetDirectory: $0.directory,
                publicHeadersDirectory: ($0 as? SourceModuleTarget)?.publicHeadersDirectory)
            },
            pluginWorkDirectory: context.pluginWorkDirectory,
            toolNamesToPaths: context.toolNamesToPaths))
    }
}
```

## Additional APIs

This proposal also adds the first of what is expected to be a toolbox of APIs to cover common things that plugins want to do:

```swift
extension Target {
    /// The transitive closure of all the targets on which the reciver depends,
    /// ordered such that every dependency appears before any other target that
    /// depends on it (i.e. in "topological sort order").
    public var recursiveTargetDependencies: [Target]
}

extension SourceModuleTarget {
    /// A possibly empty list of source files in the target that have the given
    /// filename suffix.
    public func sourceFiles(withSuffix: String) -> FileList
}
```

Future proposals might add other useful APIs.

## Example 1:  SwiftGen

```swift
import PackagePlugin

@main
struct SwiftGenPlugin: BuildToolPlugin {
    /// This plugin's implementation returns a single `prebuild` command to run `swiftgen`.
    func createBuildCommands(context: PluginContext, target: Target) throws -> [Command] {
        // This example configures `swiftgen` to take inputs from a `swiftgen.yml` file
        let swiftGenConfigFile = context.package.directory.appending("swiftgen.yml")
        
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
                "TARGET_NAME": "\(target.name)",
                "DERIVED_SOURCES_DIR": "\(genSourcesDir)",
            ],
            outputFilesDirectory: genSourcesDir)]
    }
}
```

## Example 2:  SwiftProtobuf

```swift
import PackagePlugin
import Foundation

@main
struct MyPlugin: BuildToolPlugin {
    /// This plugin's implementation returns multiple build commands, each of which
    /// calls `protoc`.
    func createBuildCommands(context: PluginContext, target: Target) throws -> [Command] {
        // In this case we generate an invocation of `protoc` for each input file,
        // passing it the path of the `protoc-gen-swift` generator tool.
        let protocTool = try context.tool(named: "protoc")
        let protocGenSwiftTool = try context.tool(named: "protoc-gen-swift")
        
        // Construct the search paths for the .proto files, which can include any
        // of the targets in the dependency closure. Here we assume that the public
        // ones are in a `protos` directory, but this can be made arbitrarily complex.
        var protoSearchPaths = target.recursiveTargetDependencies.map {
            $0.directory.appending("protos")
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
        let inputFiles = target.sourceFiles(withSuffix: ".proto")
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

## Security Considerations

As specified in SE-0303, plugins are invoked in a sandbox that prevents network access and file system write operations other than to a small set of predetermined locations. This proposal only extends the information that is available to plugins so that it contains information that is already defined in a package graph — it doesn’t grant any new abilities to the plugin.
