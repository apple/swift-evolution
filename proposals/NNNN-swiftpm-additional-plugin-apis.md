# Extended Package Plugin APIs

* Proposal: [SE-NNNN](NNNN-swiftpm-additional-plugin-apis.md)
* Authors: [Anders Bertelrud](https://github.com/abertelrud)
* Review Manager: TBD
* Status: **Draft**
* Implementation: [apple/swift-package-manager#3758](https://github.com/apple/swift-package-manager/pull/3758)
* Review:  TBD

## Introduction

SE-0303 introduced the ability to define *build tool plugins* in SwiftPM, allowing custom tools to be invoked while building a package. In support of this, SE-0303 introduced a minimal initial API through which plugins can access information about the target for which they are invoked.

This proposal extends the plugin API to provide more context, including a richer representation of the package graph.  This is in preparation for supporting new kinds of plugins in the future.

## Motivation

The build tool plugin support introduced in SE-0303 is focused on code generation during a build of a package, for such purposes as generating Swift source files from `.proto` files or other inputs. The initial API provided to plugins was oriented toward that task, and was purposefully kept minimal in order to keep the scope of the proposal bounded.

New kinds of plugins that are being discussed will require a richer context. In particular, providing a distilled form of the whole package graph would allow for a wide variety of new kinds of plugins.

## Proposed Solution

This proposal extends the plugin API that was introduced in SE-0303 by defining a generic `PluginContext` structure that supersedes `TargetBuildContext`. This new structure provides a distilled form of the resolved package graph as seen by SwiftPM, with information about all the products and targets therein.

This is the same structure that SwiftPM’s built-in subsystems currently use, and the intent is that, over time, at least some of those subsystems can be reimplemented as plugins. This information is also expected to be useful to all kinds of plugins.

In addition to the new information, this proposal adds API for traversing the package graph, such as being able to access topologically sorted lists of target dependencies. This will make it more convenient for build tool plugins that need to, for example, generate command line arguments that include a search paths for each dependency target.

## Detailed Design

This proposal defines a new `PluginContext` structure that contains:

* a reference to the `Package` object at the root of the subgraph to which the plugin is being applied
* the contextual information that was previously part of `TargetBuildContext`

This structure factors out all the information related to the package graph, such as the package and target names and directories, leaving the context with just the top-level contextual information.

The `BuildToolPlugin` protocol entry point defined by SE-0303 is superseded with a new entry point that takes the new `PluginContext` type and a reference to the `Target` for which build commands should be generated, but the previous API remains so that existing plugins continue to function.

### Plugin API

The new `PluginContext` structure in the `PackagePlugin` API is defined as:

```swift
/// Provides information about the package for which the plugin is invoked,
/// as well as contextual information based on the plugin's stated intent
/// and requirements.
public struct PluginContext {
    /// Information about the package to which the plugin is being applied.
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

    /// The path of the directory into which built products associated with
    /// the target are written.
    public let builtProductsDirectory: Path

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

The `package` property is a reference to the top-level package to which the plugin is being applied. Through it, the entire subgraph of resolved packages reachable from it can be accessed.

The properties related to looking up tools with a particular name are unchanged from the original SE-0303 proposal.

This has the effect of factoring out all information related to the package and target, and it puts them into its own directed acyclic graph consisting of the following types:

```swift
/// Represents a single package in the graph (either the root or a dependency).
public class Package {
    /// The name of the package (for display purposes only).
    public let name: String

    /// The absolute path of the package directory in the local file system.
    public let directory: Path

    /// Any dependencies on other packages, in the same order as they are
    /// specified in the package manifest.
    public let dependencies: [Dependency]

    /// Represents a resolved dependency of a package on another package. This is a
    /// separate entity in order to make it easier for future versions of the API to
    /// add information about the dependency itself.
    public struct Dependency {
        /// The package to which the dependency was resolved.
        public let package: Package
    }

    /// Any regular products defined in this package (except plugin products),
    /// in the same order as they are specified in the package manifest.
    public let products: [Product]

    /// Any regular targets defined in this package (except plugin targets),
    /// in the same order as they are specified in the package manifest.
    public let targets: [Target]
}

/// Represents a single product defined in a package. Specializations represent
/// different types of products.
public class Product {
    /// The name of the product, as defined in the package manifest. It's unique
    /// among the products of the package in which it is defined.
    public let name: String
    
    /// The targets that directly comprise the product, in the order in which
    /// they are declared in the package manifest. The product will contain the
    /// transitive closure of the these targets and their depdendencies.
    public let targets: [Target]
}

/// Represents an executable product defined in a package.
public class ExecutableProduct: Product {
    /// The target that contains the main entry point of the executable. Every
    /// executable product has exactly one main executable target. This target
    /// will always be one of the targets in the product's `targets` array.
    public let mainTarget: Target
}

/// Represents a library product defined in a package.
public class LibraryProduct: Product {
    /// Whether the library is static, dynamic, or automatically determined.
    public let type: LibraryType
}

/// A type of library product.
public enum LibraryType {
    /// Static library.
    case `static`

    /// Dynamic library.
    case `dynamic`

    /// The type of library is unspecified and will be determined at build time.
    case `automatic`
}

/// Represents a single target defined in a package. Specializations represent
/// different types of targets.
public class Target {
    /// The name of the target, as defined in the package manifest. It's unique
    /// among the targets of the package.
    public var name: String
    
    /// The absolute path of the target directory in the local file system.
    public var directory: Path
    
    /// Any other targets on which this target depends, in the same order as
    /// they are specified in the package manifest. Conditional dependencies
    /// that do not apply have already been filtered out.
    public var dependencies: [Dependency]

    /// Represents a dependency of a target on a product or on another target.
    public enum Dependency {
        /// A dependency on a target in the same package.
        case target(Target)

        /// A dependency on a product in another package.
        case product(Product)
    }
}

/// Represents a target consisting of a source code module, containing either
/// Swift or source files in one of the C-based languages.
public class SourceModuleTarget: Target {
    /// The name of the module produced by the target (derived from the target
    /// name, though future SwiftPM versions may allow this to be customized).
    public var moduleName: String

    /// The directory containing public C headers, if applicable. This will
    /// only be set for targets that have Clang sources, and only if there is
    /// a public headers directory.
    public var publicHeadersDirectory: Path?

    /// The source files that are associated with this target (any files that
    /// have been excluded in the manifest have already been filtered out).
    public var sourceFiles: FileList
}

/// Represents a target describing a library that is distributed as a binary.
public class BinaryLibraryTarget: Target {
    /// The library in the local file system.
    public var libraryPath: Path
}

/// Represents a target describing a system library that is expected to be
/// present on the host system.
public class SystemLibraryTarget: Target {
    /// The directory containing public C headers, if applicable.
    public let publicHeadersDirectory: Path?
  
    /// The library in the local file system.
    public let libraryPath: Path
}

/// Provides information about a list of files. The order is not defined
/// but is guaranteed to be stable. This allows the implementation to be
/// more efficient than a static file list.
public struct FileList: Sequence {
    public struct Iterator: IteratorProtocol {
        mutating public func next() -> File?
    }
    public func makeIterator() -> Iterator
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
            builtProductsDirectory: context.builtProductsDirectory,
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
```

Future proposals might add other useful APIs, such as ones to filters dependencies or input files in other ways. For example, it’s common to want to operate on only a small subset of input files, such as those with a particular filename suffix.

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
        let inputFiles = target.sourceFiles.filter { $0.path.extension == "proto" }
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

## Open Questions

This is a list of the currently open questions that will need to be resolved before this proposal is put up for review.

* Should the whole package graph be available to every kind of plugin, or just the package to which it is being applied? If the whole graph is available, then does the plugin entry point need a separate parameter to specify the package to which the plugin is being applied?

* For how long should the `TargetBuildContext` compatibility structure be supported? Presumably it can eb removed in the next tools version.

* Should this proposal include better API for filtering on file types? This is a very common thing for plugins to want to do.