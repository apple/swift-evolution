# Package Manager Manifest API Redesign

* Proposal: [SE-0158](0158-package-manager-manifest-api-redesign.md)
* Author: [Ankit Aggarwal](https://github.com/aciidb0mb3r)
* Review Manager: [Rick Ballard](https://github.com/rballard)
* Status: **Implemented (Swift 4.0)**
* Bug: [SR-3949](https://bugs.swift.org/browse/SR-3949)
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-se-0158-package-manager-manifest-api-redesign/5468)

## Introduction

This is a proposal for redesigning the `Package.swift` manifest APIs provided
by Swift Package Manager.  
This proposal only redesigns the existing public APIs and does not add any
new functionality; any API to be added for new functionality will happen in
separate proposals.

## Motivation

The `Package.swift` manifest APIs were designed prior to the [API Design
Guidelines](https://swift.org/documentation/api-design-guidelines/), and their
design was not reviewed by the evolution process. Additionally, there are
several small areas which can be cleaned up to make the overall API more
"Swifty".

We would like to redesign these APIs as necessary to provide clean,
conventions-compliant APIs that we can rely on in the future. Because we
anticipate that the user community for the Swift Package Manager will grow
considerably in Swift 4, we would like to make these changes now, before
more packages are created using the old API.

## Proposed solution

Note: Access modifier is omitted from the diffs and examples for brevity. The
access modifier is `public` for all APIs unless specified.

* Remove `successor()` and `predecessor()` from `Version`.

    These methods neither have well defined semantics nor are used a lot
    (internally or publicly). For e.g., the current implementation of
    `successor()` always just increases the patch version.


    <details>
      <summary>View diff</summary>
      <p>

    ```diff
    struct Version {
    -    func successor() -> Version

    -    func predecessor() -> Version
    }
    ```
    </p></details>

* Convert `Version`'s `buildMetadataIdentifier` property to an array.

    According to SemVer 2.0, build metadata is a series of dot separated
    identifiers. Currently this is represented as an optional string property
    in the `Version` struct. We propose to change this property to an array
    (similar to `prereleaseIdentifiers` property). To maintain backwards
    compatiblility in PackageDescription 3 API, we will keep the optional
    string as a computed property based on the new array property. We will also
    keep the version initializer that takes the `buildMetadataIdentifier` string.

* Make all properties of `Package` and `Target` mutable.

    Currently, `Package` has three immutable and four mutable properties, and
    `Target` has one immutable and one mutable property. We propose to make all
    properties mutable to allow complex customization on the package object
    after initial declaration.

    <details>
      <summary>View diff and example</summary>
      <p>

    Diff:
    ```diff
    final class Target {
    -    let name: String
    +    var name: String
    }

    final class Package {
    -    let name: String
    +    var name: String

    -    let pkgConfig: String?
    +    var pkgConfig: String?

    -    let providers: [SystemPackageProvider]?
    +    var providers: [SystemPackageProvider]?
    }
    ```

    Example:
    ```swift
    let package = Package(
        name: "FooPackage",
        targets: [
            Target(name: "Foo", dependencies: ["Bar"]),
        ]
    )

    #if os(Linux)
    package.targets[0].dependencies = ["BarLinux"]
    #endif
    ```
    </p></details>

* Change `Target.Dependency` enum cases to lowerCamelCase.

    According to API design guidelines, everything other than types should be in lowerCamelCase.

    <details>
      <summary>View diff and example</summary>
      <p>

     Diff:
    ```diff
    enum Dependency {
    -    case Target(name: String)
    +    case target(name: String)

    -    case Product(name: String, package: String?)
    +    case product(name: String, package: String?)

    -    case ByName(name: String)
    +    case byName(name: String)
    }
    ```

    Example:
    ```diff
    let package = Package(
        name: "FooPackage",
        targets: [
            Target(
                name: "Foo", 
                dependencies: [
    -                .Target(name: "Bar"),
    +                .target(name: "Bar"),

    -                .Product(name: "SwiftyJSON", package: "SwiftyJSON"),
    +                .product(name: "SwiftyJSON", package: "SwiftyJSON"),
                ]
            ),
        ]
    )
    ```
    </p></details>

* Add default parameter to the enum case `Target.Dependency.product`.

    The associated value `package` in the (enum) case `product`, is an optional
    `String`. It should have the default value `nil` so clients don't need to
    write it if they prefer using explicit enum cases but don't want to specify
    the package name i.e. it should be possible to write `.product(name:
    "Foo")` instead of `.product(name: "Foo", package: nil)`.

    If
    [SE-0155](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0155-normalize-enum-case-representation.md)
    is accepted, we can directly add a default value. Otherwise, we will use a
    static factory method to provide default value for `package`.

* Rename all enums cases to have a suffix `Item` and favor static methods.

    Since static methods are more extensible than enum cases right now, we
    should discourage use of direct use of enum initializers and provide a
    static method for each case. It is not possible to have overloads on enum
    cases, so as a convention we propose to rename all enum cases to have a
    suffix "Item" for future extensibility.

* Change `SystemPackageProvider` enum cases to lowerCamelCase and their payloads to array.

    According to API design guidelines, everything other than types should be
    in lowerCamelCase.

    This enum allows SwiftPM System Packages to emit hints in case of build
    failures due to absence of a system package. Currently, only one system
    package per system packager can be specified. We propose to allow
    specifying multiple system packages by changing the payload to be an array.

    <details>
      <summary>View diff and example</summary>
      <p>

     Diff:
    ```diff
    enum SystemPackageProvider {
    -    case Brew(String)
    +    case brew([String])

    -    case Apt(String)
    +    case apt([String])
    }
    ```

    Example:

    ```diff
    let package = Package(
        name: "Copenssl",
        pkgConfig: "openssl",
        providers: [
    -        .Brew("openssl"),
    +        .brew(["openssl"]),

    -        .Apt("openssl-dev"),
    +        .apt(["openssl", "libssl-dev"]),
        ]
    )
    ```
    </p></details>


* Remove implicit target dependency rule for test targets.

    There is an implicit test target dependency rule: a test target "FooTests"
    implicitly depends on a target "Foo", if "Foo" exists and "FooTests" doesn't
    explicitly declare any dependency. We propose to remove this rule because:

    1. It is a non obvious "magic" rule that has to be learned.
    2. It is not possible for "FooTests" to remove dependency on "Foo" while
       having no other (target) dependency.
    3. It makes real dependencies less discoverable.
    4. It may cause issues when we get support for mechanically editing target
       dependencies.

* Use factory methods for creating objects.

    We propose to always use factory methods to create objects except for the
    main `Package` object. This gives a lot of flexibility and extensibility to
    the APIs because Swift's type system can infer the top level type in a
    context and allow using the shorthand dot syntax.

    Concretely, we will make these changes:

    * Add a factory method `target` to `Target` class and change the current
      initializer to private.

        <details>
          <summary>View example and diff</summary>
          <p>

        Example:

        ```diff
        let package = Package(
            name: "Foo",
            target: [
        -        Target(name: "Foo", dependencies: ["Utility"]),
        +        .target(name: "Foo", dependencies: ["Utility"]),
            ]
        )
        ```
        </p></details>

    * Introduce a `Product` class with two subclasses: `Executable` and
      `Library`.  These subclasses will be nested inside `Product` class
      instead of being a top level declaration in the module. Nesting will give
      us a namespace for products and it is easy to find all the supported
      products when the product types grows to a large number. We will add two
      factory methods to `Product` class: `library` and `executable` to create
      respective products.

        ```swift
        /// Represents a product.
        class Product {
        
            /// The name of the product.
            let name: String

            private init(name: String) {
                self.name = name
            }
        
            /// Represents an executable product.
            final class Executable: Product {

                /// The names of the targets in this product.
                let targets: [String]

                private init(name: String, targets: [String])
            }
        
            /// Represents a library product.
            final class Library: Product {
                /// The type of library product.
                enum LibraryType: String {
                    case `static`
                    case `dynamic`
                }

                /// The names of the targets in this product.
                let targets: [String]
        
                /// The type of the library.
                ///
                /// If the type is unspecified, package manager will automatically choose a type.
                let type: LibraryType?
        
                private init(name: String, type: LibraryType? = nil, targets: [String])
            }

            /// Create a library product.
            static func library(name: String, type: LibraryType? = nil, targets: [String]) -> Library

            /// Create an executable product.
            static func executable(name: String, targets: [String]) -> Library
        }
        ```

        <details>
          <summary>View example</summary>
          <p>

        Example:

        ```swift
        let package = Package(
            name: "Foo",
            target: [
                .target(name: "Foo", dependencies: ["Utility"]),
                .target(name: "tool", dependencies: ["Foo"]),
            ],
            products: [
                .executable(name: "tool", targets: ["tool"]), 
                .library(name: "Foo", targets: ["Foo"]), 
                .library(name: "FooDy", type: .dynamic, targets: ["Foo"]), 
            ]
        )
        ```
        </p></details>


* Special syntax for version initializers.

    A simplified summary of what is commonly supported in other package managers:

    | Package Manager | x-ranges      | tilde (`~` or `~>`)     | caret (`^`)   |
    |-----------------|---------------|-------------------------|---------------|
    | npm             | Supported     | Allows patch-level changes if a minor version is specified on the comparator. Allows minor-level changes if not.  | patch and minor updates |
    | Cargo           | Supported     | Same as above           | Same as above |
    | CocoaPods       | Not supported | Same as above           | Not supported |
    | Carthage        | Not supported | patch and minor updates | Not supported |

    Some general notes:

    Every package manager we looked at supports the tilde `~` operator in some form, and it's generally
    recommended as "the right thing", because package maintainers often fail to increment their major package
    version when they should, incrementing their minor version instead. See e.g. how Google created a
    [6-minute instructional video](https://www.youtube.com/watch?v=x4ARXyovvPc) about this operator for CocoaPods.
    This is a form of version overconstraint; your package should be compatible with everything with the same
    major version, but people don't trust that enough to rely on it. But version overconstraint is harmful,
    because it leads to "dependency hell" (unresolvable dependencies due to conflicting requirements for a package
    in the dependency graph).
    
    We'd like to encourage a better standard of behavior in the Swift Package Manager. In the future, we'd like
    to add tooling to let the package manager automatically help you use Semantic Versioning correctly,
    so that your clients can trust your major version. If we can get package maintainers to use SemVer correctly,
    through automatic enforcement in the future or community norms for now, then caret `^` becomes
    the best operator to use most of the time. That is, you should be able to specify a minimum version,
    and you should be willing to let your package use anything after that up to the next major version. This
    means you'll get safe updates automatically, and you'll avoid overconstraining and introducing dependency hell.
    
    Caret `^` and tilde `~` syntax is somewhat standard, but is syntactically non-obvious; we'd prefer a syntax
    that doesn't require reading a manual for novices to understand, even if that means we break with the
    syntactic convention established by the other package managers which support caret `^` and tilde `~`.
    We'd like to make it possible to follow the tilde `~` use case (with different syntax), but caret `^`
    should be the most convenient, to encourage its use.

    What we propose:

    * We will introduce a factory method which takes a lower bound version and
      forms a range that goes upto the next major version (i.e. caret).

      ```swift
      // 1.0.0 ..< 2.0.0
      .package(url: "/SwiftyJSON", from: "1.0.0"),

      // 1.2.0 ..< 2.0.0
      .package(url: "/SwiftyJSON", from: "1.2.0"),

      // 1.5.8 ..< 2.0.0
      .package(url: "/SwiftyJSON", from: "1.5.8"),
      ```

    * We will introduce a factory method which takes `Requirement`, to
      conveniently specify common ranges.

      `Requirement` is an enum defined as follows:

      ```swift
      enum Requirement {
          /// The requirement is specified by an exact version.
          case exact(Version)

          /// The requirement is specified by a version range.
          case range(Range<Version>)

          /// The requirement is specified by a source control revision.
          case revision(String)

          /// The requirement is specified by a source control branch.
          case branch(String)

          /// Creates a specified for a range starting at the given lower bound
          /// and going upto next major version.
          static func upToNextMajor(from version: Version) -> Requirement

          /// Creates a specified for a range starting at the given lower bound
          /// and going upto next minor version.
          static func upToNextMinor(from version: Version) -> Requirement
      }
      ```

      Examples:

      ```swift
      // 1.5.8 ..< 2.0.0
      .package(url: "/SwiftyJSON", .upToNextMajor(from: "1.5.8")),

      // 1.5.8 ..< 1.6.0
      .package(url: "/SwiftyJSON", .upToNextMinor(from: "1.5.8")),

      // 1.5.8
      .package(url: "/SwiftyJSON", .exact("1.5.8")),
      ```

    * This will also give us ability to add more complex features in future:

      Examples:
      > Note that we're not actually proposing these as part of this proposal.

      ```swift
      .package(url: "/SwiftyJSON", .upToNextMajor(from: "1.5.8").excluding("1.6.4")),

      .package(url: "/SwiftyJSON", .exact("1.5.8", "1.6.3")),
      ```

    * We will introduce a factory method which takes `Range<Version>`, to specify
      arbitrary open range.

      ```swift
      // Constraint to an arbitrary open range.
      .package(url: "/SwiftyJSON", "1.2.3"..<"1.2.6"),
      ```

    * We will introduce a factory method which takes `ClosedRange<Version>`, to specify
      arbitrary closed range.

      ```swift
      // Constraint to an arbitrary closed range.
      .package(url: "/SwiftyJSON", "1.2.3"..."1.2.8"),
      ```
    * As a slight modification to the
      [branch proposal](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0150-package-manager-branch-support.md),
      we will add cases for specifying a branch or revision, rather than
      adding factory methods for them:

      ```swift
      .package(url: "/SwiftyJSON", .branch("develop")),
      .package(url: "/SwiftyJSON", .revision("e74b07278b926c9ec6f9643455ea00d1ce04a021"))
      ```

    * We will remove all of the current factory methods:

      ```swift
      // Constraint to a major version.
      .Package(url: "/SwiftyJSON", majorVersion: 1),

      // Constraint to a major and minor version.
      .Package(url: "/SwiftyJSON", majorVersion: 1, minor: 2),

      // Constraint to an exact version.
      .Package(url: "/SwiftyJSON", "1.2.3"),

      // Constraint to an arbitrary range.
      .Package(url: "/SwiftyJSON", versions: "1.2.3"..<"1.2.6"),

      // Constraint to an arbitrary closed range.
      .Package(url: "/SwiftyJSON", versions: "1.2.3"..."1.2.8"),
      ```

* Adjust order of parameters on `Package` class:

    We propose to reorder the parameters of `Package` class to: `name`,
    `pkgConfig`, `products`, `dependencies`, `targets`, `swiftLanguageVersions`.

    The rationale behind this reorder is that the most interesting parts of a
    package are its product and dependencies, so they should be at the top.
    Targets are usually important during development of the package.  Placing
    them at the end keeps it easier for the developer to jump to end of the
    file to access them. Note that the `swiftLanguageVersions` property will likely
    be removed once we support Build Settings, but that will be discussed in a separate proposal.


    <details>
      <summary>View example</summary>
      <p>

    Example:

    ```swift
    let package = Package(
        name: "Paper",
        products: [
            .executable(name: "tool", targets: ["tool"]),
            .library(name: "Paper", type: .static, targets: ["Paper"]),
            .library(name: "PaperDy", type: .dynamic, targets: ["Paper"]),
        ],
        dependencies: [
            .package(url: "http://github.com/SwiftyJSON/SwiftyJSON", from: "1.2.3"),
            .package(url: "../CHTTPParser", .upToNextMinor(from: "2.2.0")),
            .package(url: "http://some/other/lib", .exact("1.2.3")),
        ]
        targets: [
            .target(
                name: "tool",
                dependencies: [
                    "Paper",
                    "SwiftyJSON"
                ]),
            .target(
                name: "Paper",
                dependencies: [
                    "Basic",
                    .target(name: "Utility"),
                    .product(name: "CHTTPParser"),
                ])
        ]
    )
    ```
    </p></details>

* Eliminate exclude in future (via custom layouts feature).

    We expect to remove the `exclude` property after we get support for custom
    layouts. The exact details will be in the proposal of that feature.

## Example manifests

* A regular manifest.

```swift
let package = Package(
    name: "Paper",
    products: [
        .executable(name: "tool", targets: ["tool"]),
        .library(name: "Paper", targets: ["Paper"]),
        .library(name: "PaperStatic", type: .static, targets: ["Paper"]),
        .library(name: "PaperDynamic", type: .dynamic, targets: ["Paper"]),
    ],
    dependencies: [
        .package(url: "http://github.com/SwiftyJSON/SwiftyJSON", from: "1.2.3"),
        .package(url: "../CHTTPParser", .upToNextMinor(from: "2.2.0")),
        .package(url: "http://some/other/lib", .exact("1.2.3")),
    ]
    targets: [
        .target(
            name: "tool",
            dependencies: [
                "Paper",
                "SwiftyJSON"
            ]),
        .target(
            name: "Paper",
            dependencies: [
                "Basic",
                .target(name: "Utility"),
                .product(name: "CHTTPParser"),
            ])
    ]
)
```

* A system package manifest.

```swift
let package = Package(
    name: "Copenssl",
    pkgConfig: "openssl",
    providers: [
        .brew(["openssl"]),
        .apt(["openssl", "libssl-dev"]),
    ]
)
```

## Impact on existing code

The above changes will be implemented only in the new Package Description v4
library. The v4 runtime library will release with Swift 4 and packages will be
able to opt-in into it as described by
[SE-0152](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0152-package-manager-tools-version.md).

There will be no automatic migration feature for updating the manifests from v3
to v4. To indicate the replacements of old APIs, we will annotate them using
the `@unavailable` attribute where possible. Unfortunately, this will not cover
all the changes for e.g. rename of the target dependency enum cases.

All new packages created with `swift package init` command in Swift 4 tools
will by default to use the v4 manifest. It will be possible to switch to v3
manifest version by changing the tools version using `swift package
tools-version --set 3.1`.  However, the manifest will needed to be adjusted to
use the older APIs manually.

Unless declared in the manifest, existing packages automatically default
to the Swift 3 minimum tools version; since the Swift 4 tools will also include
the v3 manifest API, they will build as expected.

A package which needs to support both Swift 3 and Swift 4 tools will need to
stay on the v3 manifest API and support the Swift 3 language version for its
sources, using the API described in the proposal
[SE-0151](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0151-package-manager-swift-language-compatibility-version.md).

An existing package which wants to use the new v4 manifest APIs will need to bump its
minimum tools version to 4.0 or later using the command `$ swift package tools-version
--set-current`, and then modify the manifest file with the changes described in
this proposal.

## Alternatives considered

* Add variadic overloads.

    Adding variadic overload allows omitting parenthesis which leads to less
    cognitive load on eyes, especially when there is only one value which needs
    to be specified. For e.g.:

        Target(name: "Foo", dependencies: "Bar")

    might looked better than:

        Target(name: "Foo", dependencies: ["Bar"])

    However, plurals words like `dependencies` and `targets` imply a collection
    which implies brackets. It also makes the grammar wrong. Therefore, we
    reject this option.
    
* Version exclusion.
    
    It is not uncommon to have a specific package version break something, and
    it is undesirable to "fix" this by adjusting the range to exclude it
    because this overly constrains the graph and can prevent picking up the
    version with the fix.

    This is desirable but it should be proposed separately.

* Inline package declaration.

    We should probably support declaring a package dependency anywhere we
    support spelling a package name. It is very common to only have one target
    require a dependency, and annoying to have to specify the name twice.

    This is desirable but it should be proposed separately.

* Introduce an "identity rule" to determine if an API should use an initializer
  or a factory method:

    Under this rule, an entity having an identity, will use a type initializer
    and everything else will use factory methods. `Package`, `Target` and
    `Product` are identities. However, a product referenced in a target
    dependency is not an identity.

    We rejected this because it may become a source of confusion for users.
    Another downside is that the product initializers will have to used with
    the dot notation (e.g.: `.Executable(name: "tool", targets: ["tool"])`)
    which is a little awkward because we expect factory methods and enum cases
    to use the dot syntax. This can be solved by moving these products outside
    of `Product` class but we think having a namespace for product provides a
    lot of value.

* Upgrade `SystemPackageProvider` enum to a struct.

    We thought about upgrading `SystemPackageProvider` to a struct when we had
    the "identity" rule but since we're dropping that, there is no need for
    this change.

    ```swift
    public struct SystemPackageProvider {
        enum PackageManager {
            case apt
            case brew
        }

        /// The system package manager.
        let packageManager: PackageManager

        /// The array of system packages.
        let packages: [String]

        init(_ packageManager: PackageManager, packages: [String])
    }
    ```
