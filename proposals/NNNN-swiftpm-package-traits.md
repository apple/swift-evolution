# Package traits

* Proposal: [SE-NNNN](NNNN-swiftpm-package-traits.md)
* Authors: [Franz Busch](https://github.com/FranzBusch), [Max Desiatov](https://github.com/MaxDesiatov)
* Review Manager: TBD
* Status: **Experimental implementation available, gated on `@_spi(ExperimentalTraits)` in package manifests and `--experimental` prefix for CLI options**

- https://github.com/swiftlang/swift-package-manager/pull/7704
- https://github.com/swiftlang/swift-package-manager/pull/7703
- https://github.com/swiftlang/swift-package-manager/pull/7702
- https://github.com/swiftlang/swift-package-manager/pull/7701
- https://github.com/swiftlang/swift-package-manager/pull/7694
- https://github.com/swiftlang/swift-package-manager/pull/7689

## Introduction

Over the past years the package ecosystem has grown tremendously in both the
amount of packages and the functionality that individual packages offer.
Additionally, Swift is being used in more environments such as embedded systems
or Wasm. This proposal aims to give package authors a new tool to conditionalize
the features they offer and the dependencies that they use.

## Motivation

There are various motivating use-cases where package authors might want to
express configurable compilation or optional dependencies. This section is going
to list a few of those use-cases.

### Pluggable dependencies

Some packages want to make it configurable what underlying technology is used.
The [Swift OpenAPIGenerator](https://github.com/apple/swift-openapi-generator)
for example is capable of running on top of `URLSession`, `AsyncHTTPClient`,
`Hummingbird` or `Vapor`. To avoid bringing all of those potential dependencies
into every adopters binary, the project has created individual repositories for
each transport. This achieves the goal of making the dependencies optional;
however, it requires users to discover those adjacent repositories and add
additional dependencies to their project.

### Configurable behavior

Packages often want to cater to multiple ecosystems such as the iOS or the
server ecosystem. While most of the technologies are shared between ecosystems
there are often some platform specific behaviors/libraries that one might use.
For example, on Apple's platforms `OSLog` is the canonical logging system
whereas the server ecosystem is mostly using `swift-log`. However, there are
some users that prefer to use `swift-log` on Apple's platforms which means
libraries and applications cannot use platform compiler conditionals.

### Replacing environment variables in Package manifests

A lot of packages are using environment variables in their `Package.swift` to
configure their package. This has various reasons such as optional dependencies
or setting certain defines for local development. Using environment variables
inside `Package.swift` is not officially supported and with stricter sandboxing
rules might break in the future.

### Experimental APIs

Some packages want to introduce new functionality without yet committing to a
stable public API Currently, those modules and APIs are often underscored or
specifically annotated. While this approach works it comes with downsides such
as hiding the APIs in code completion.

## Proposed solution

This proposal introduces a new configuration for packages called _package
traits_. Package authors can define a set of traits in their `Package.swift`
that their package offers which provide a way to express conditional compilation
and optional dependencies. Furthermore, a set of default enabled traits can be
specified.

```swift
let package = Package(
    name: "Example",
    traits: [
        "Foo",
        .trait(
            name: "Bar",
            enabledTraits: [ // Other traits that are enabled when this trait is being enabled
                "Foo",
            ]
        )
        .trait(
            name: "FooBar",
            enabledTraits: [
                "Foo",
                "Bar",
            ]
        ),
        .default(enabledTraits: ["Foo"]), // Defines all the default enabled traits
    ],
    /// ...
)
```

When depending on a package the `default` trait is enabled. However, the enabled
traits can be customized by passing a set of enabled traits when declaring the
dependency. When specifying the enabled traits of the dependencies the
`.default` trait can be passed which will enable the default trait. The below
example enables the default trait and the additional `SomeTrait` of the package.

```swift
dependencies: [
    .package(
        url: "https://github.com/Org/SomePackage.git",
        from: "1.0.0",
        traits: [
            .default,
            "SomeTrait"
        ]
    ),
]
```

To disable all traits including the default trait an empty set can be passed.

```swift
dependencies: [
    .package(
        url: "https://github.com/Org/SomePackage.git",
        from: "1.0.0",
        traits: [] // All traits are disabled
    ),
]
```

Another common scenario is to enable a trait of a dependency only when a trait
of the package is enabled. The below example enables the `SomeOtherTrait` when
the `Foo` trait of this package is enabled.

```swift
dependencies: [
    .package(
        url: "https://github.com/Org/SomePackage.git",
        from: "1.0.0",
        traits:[
            .trait("SomeOtherTrait", condition: .when(traits: ["Foo"])),
        ]
    ),
]
```

Conditional dependencies are specified per target and extend the current
`condition` syntax which is used for specifying platform dependent dependencies.

```swift
targets: [
    .target(
        name: "SomeTarget",
        dependencies: [
            .product(
                name: "SomeProduct",
                package: "SomePackage",
                condition: .when(traits: ["Foo"])
            ),
        ]
    )
]
```

Lastly, code can be conditionally compiled by checking if a trait is enabled.
This can be used for both optional dependencies by surrounding the `import`
statements in a trait check and for regular code where you want to modify its
behaviour depending on the enabled traits.

```swift
#if Foo
import SomeDependency
#endif

func hello() {
    #if Foo
    Foo.hello()
    #else
    print("Hello")
    #endif
}
```

## Detailed design

This proposal extends the current `PackageDescription` APIs by introducing the
following new `Trait` type.

```swift
/// A struct representing a package's trait.
///
/// Traits can be used for expressing conditional compilation and optional dependencies.
public struct Trait: Hashable, ExpressibleByStringLiteral {
    /// Declares the default traits for this package.
    public static func `default`(enabledTraits: Set<String>) -> Self

    /// The trait's canonical name.
    ///
    /// This is used when enabling the trait or when referring to it from other modifiers in the manifest.
    ///
    /// The following rules are enforced on trait names:
    /// - The first character must be a [Unicode XID start character](https://unicode.org/reports/tr31/#Figure_Code_Point_Categories_for_Identifier_Parsing)
    /// (most letters), a digit, or `_`.
    /// - Subsequent characters must be a [Unicode XID continue character](https://unicode.org/reports/tr31/#Figure_Code_Point_Categories_for_Identifier_Parsing)
    /// (a digit, `_`, or most letters), `-`, or `+`.
    /// - `default` and `defaults` (in any letter casing combination) are not allowed as trait names to avoid confusion with default traits.
    public var name: String

    /// The trait's description.
    ///
    /// Use this to explain what functionality this trait enables.
    public var description: String?

    /// A set of other traits of this package that this trait enables.
    public var enabledTraits: Set<String>

    /// Initializes a new trait.
    ///
    /// - Parameters:
    ///   - name: The trait's canonical name.
    ///   - description: The trait's description.
    ///   - enabledTraits: A set of other traits of this package that this trait enables.
    public init(
        name: String,
        description: String? = nil,
        enabledTraits: Set<String> = []
    )

    /// Initializes a new trait.
    ///
    /// This trait is disabled by default and enables no other trait of this package.
    public init(stringLiteral value: StringLiteralType)

    /// Initializes a new trait.
    ///
    /// - Parameters:
    ///   - name: The trait's canonical name.
    ///   - description: The trait's description.
    ///   - enabledTraits: A set of other traits of this package that this trait enables.
    public static func trait(
        name: String,
        description: String? = nil,
        enabledTraits: Set<String> = []
    ) -> Trait
}
```

The `Package` class is extended to define a set of traits:

```swift
public final class Package {
    // ...

    /// The set of traits of this package.
    public var traits: Set<Trait>

    /// Initializes a Swift package with configuration options you provide.
    ///
    /// - Parameters:
    ///   - name: The name of the Swift package, or `nil` to use the package's Git URL to deduce the name.
    ///   - defaultLocalization: The default localization for resources.
    ///   - platforms: The list of supported platforms with a custom deployment target.
    ///   - pkgConfig: The name to use for C modules. If present, Swift Package Manager searches for a
    ///   `<name>.pc` file to get the additional flags required for a system target.
    ///   - providers: The package providers for a system target.
    ///   - products: The list of products that this package makes available for clients to use.
    ///   - traits: The set of traits of this package.
    ///   - dependencies: The list of package dependencies.
    ///   - targets: The list of targets that are part of this package.
    ///   - swiftLanguageVersions: The list of Swift versions with which this package is compatible.
    ///   - cLanguageStandard: The C language standard to use for all C targets in this package.
    ///   - cxxLanguageStandard: The C++ language standard to use for all C++ targets in this package.
    public init(
        name: String,
        defaultLocalization: LanguageTag? = nil,
        platforms: [SupportedPlatform]? = nil,
        pkgConfig: String? = nil,
        providers: [SystemPackageProvider]? = nil,
        products: [Product] = [],
        traits: Set<Trait> = [],
        dependencies: [Dependency] = [],
        targets: [Target] = [],
        swiftLanguageVersions: [SwiftVersion]? = nil,
        cLanguageStandard: CLanguageStandard? = nil,
        cxxLanguageStandard: CXXLanguageStandard? = nil
    )
}
```

Furthermore, a new `Package.Dependency.Trait` type is introduced that can be used
to configure the traits of a dependency.

```swift
extension Package.Dependency {
    /// A struct representing an enabled trait of a dependency.
    public struct Trait: Hashable, Sendable, ExpressibleByStringLiteral {
        /// Enables the default traits of a package.
        public static let default: Self

        /// A condition that limits the application of a dependencies trait.
        public struct Condition: Hashable, Sendable {
            /// Creates a package dependency trait condition.
            ///
            /// - Parameter traits: The set of traits that enable the dependencies trait. If any of the traits are enabled on this package
            /// the dependencies trait will be enabled.
            public static func when(
                traits: Set<String>
            ) -> Self?
        }

        /// The name of the enabled trait.
        public var name: String

        /// The condition under which the trait is enabled.
        public var condition: Condition?

        /// Initializes a new enabled trait.
        ///
        /// - Parameters:
        ///   - name: The name of the enabled trait.
        ///   - condition: The condition under which the trait is enabled.
        public init(
            name: String,
            condition: Condition? = nil
        )

        public init(stringLiteral value: StringLiteralType)

        /// Initializes a new enabled trait.
        ///
        /// - Parameters:
        ///   - name: The name of the enabled trait.
        ///   - condition: The condition under which the trait is enabled.
        public static func trait(
            name: String,
            condition: Condition? = nil
        ) -> Trait
    }
}
```

The dependency APIs are then extended with new variants that take a `Trait` parameter:

```swift
extension Package.Dependency {
    // MARK: Path

    public static func package(
        path: String,
        traits: Set<Package.Dependency.Trait>
    ) -> Package.Dependency

    public static func package(
        name: String,
        path: String,
        traits: Set<Package.Dependency.Trait>
    ) -> Package.Dependency

    // MARK: Source repository

    public static func package(
        url: String,
        from version: Version,
        traits: Set<Package.Dependency.Trait>
    ) -> Package.Dependency

    public static func package(
        url: String,
        branch: String,
        traits: Set<Package.Dependency.Trait>
    ) -> Package.Dependency

    public static func package(
        url: String,
        revision: String,
        traits: Set<Package.Dependency.Trait>
    ) -> Package.Dependency

    public static func package(
        url: String,
        _ range: Range<Version>,
        traits: Set<Package.Dependency.Trait>
    ) -> Package.Dependency

    public static func package(
        url: String,
        _ range: ClosedRange<Version>,
        traits: Set<Package.Dependency.Trait>
    ) -> Package.Dependency

    public static func package(
        url: String,
        exact version: Version,
        traits: Set<Package.Dependency.Trait>
    ) -> Package.Dependency

    // MARK: Registry 

    public static func package(
        id: String,
        from version: Version,
        traits: Set<Package.Dependency.Trait>
    ) -> Package.Dependency

    public static func package(
        id: String,
        exact version: Version,
        traits: Set<Package.Dependency.Trait>
    ) -> Package.Dependency

    public static func package(
        id: String,
        _ range: Range<Version>,
        traits: Set<Package.Dependency.Trait>
    ) -> Package.Dependency

    public static func package(
        id: String,
        _ range: ClosedRange<Version>,
        traits: Set<Package.Dependency.Trait>
    ) -> Package.Dependency
}
```

Lastly, traits can also be used to conditionalize `SwiftSettings`, `CSettings`,
`CXXSettings` and `LinkerSettings`. For this the `BuildSettingCondition` is extended.

```swift
/// Creates a build setting condition.
///
/// - Parameters:
///   - platforms: The applicable platforms for this build setting condition.
///   - configuration: The applicable build configuration for this build setting condition.
///   - traits: The applicable traits for this build setting condition.
public static func when(
    platforms: [Platform]? = nil,
    configuration: BuildConfiguration? = nil,
    traits: Set<String>? = nil
) -> BuildSettingCondition {
    precondition(!(platforms == nil && configuration == nil))
    return BuildSettingCondition(platforms: platforms, config: configuration, traits: nil)
}
```

### Trait unification

At this point, it is important to talk about the trait unification across the
entire dependency graph. After dependency resolution the union of enabled traits
per package is calculated. This is then used to determine both the enabled
optional dependencies and the enabled traits for the compile time checks. Since
the enabled traits of a dependency are specified on a per package level and not
from the root of the tree, any combination of enabled traits must be supported.
A consequence of this is that all traits **should** be _additive_. Enabling a trait
**should not** disable functionality i.e. remove API or lead to any other
**SemVer-incompatible** change.

#### Mutally exclusive traits

Some rare use-cases may want mutally exclusive traits which are incompatible to
be enabled at the same time. This should be avoided if possible because it
requires the whole dependency graph to coordinate on what trait to enable. In
the rare case where mutually exclusive traits are used consider adding a
compiler error to detect this during build time.

```swift
#if Trait1 && Trait2
#error("Trait1 and Trait2 are mutuall exclusive")
#endif
```

A few options to avoid mutually exclusive traits:
- Separate the code into multiple packages
- Choose one trait over the other when possible
- Use platform checks `#if os(Windows)` when possible

### Default traits

Default traits allow package authors to define a set of traits that they think
cater to the majority use-cases of the package. When choosing the initial
default traits or adding a new default trait it is important to consider that
removing a default trait is a **SemVer-incompatible** change since it can potentially
remove APIs.

### Trait specific command line options for `swift build/run`

When executing one of `swift build/run` options can be passed to control which
traits for the root package are enabled:

- `--traits` _TRAITS_: Enables the passed traits of the package. Multiple traits
  can be specified by providing a comma separated list e.g. `--traits
  Trait1,Trait2`.
- `--enable-all-traits`: Enables all traits of the package.
- `--disable-default-traits`: Disables all default traits of the package.

### Trait namespaces

Trait names are namespaced per package; hence, multiple packages can define the
same trait names. Moreover, it is an expected scenario that multiple packages
define the same trait name and conditionally enable the equivalent named trait
in their dependencies.

### Trait limitations

To prevent abuse, limit the complexity and make sure it integrates with the
compiler a few limitations are imposed.

#### Number of traits

[Other
ecosystems](https://blog.rust-lang.org/2023/10/26/broken-badges-and-23k-keywords.html)
have shown that a large number of traits can have significant impact on
registries and dependency managers. To avoid such a scenario an initial maximum
number of 300 defined traits per package is imposed. This can be revisited later
once traits have been used in the ecosystem extensively.

### Allowed characters for trait names

Since traits can show up both in the `Package.swift` and in source code when
checking if a trait is enabled, the allowed characters for a trait name are
restricted to [legal Swift
identifier](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/summaryofthegrammar/).
Hence, the following rules are enforced on trait names:

- The first character must be a [Unicode XID start
  character](https://unicode.org/reports/tr31/#Figure_Code_Point_Categories_for_Identifier_Parsing)
  (most letters), a digit, or `_`.
- Subsequent characters must be a [Unicode XID continue
  character](https://unicode.org/reports/tr31/#Figure_Code_Point_Categories_for_Identifier_Parsing)
  (a digit, `_`, or most letters), `-`, or `+`.
- `default` and `defaults` (in any letter casing combination) are not allowed as
  trait names to avoid confusion with default traits.

### swift package dump-package

The `swift package dump-package` command will include information of the trait configuration for the
package and the dependencies it uses in the JSON output.

## Impact on existing packages

There is no impact on existing packages. Any package can start adopting package
traits but in doing so **must not** move existing API behind new traits. Even if
the trait is a enabled by default any consumer might have already disabled all
default traits; hence, moving API behind a new default trait could potentially
break them.

### Impact on other build systems

The initial impact on other build systems should be minimal. Exisiting packages
must not move exisiting APIs behind a trait. For new APIs that are guarded by a
trait a build system must pass the correct `SWIFT_ACTIVE_COMPILATION_CONDITIONS`
when building the modules of the package. Other build systems might want to
consider to expose a way to model traits in their target description.

### Impact on documentation

Traits are used to conditionally compile code. When building documentation the
symbol graph extracter will only see the code that is actually compiled. Systems
that produce documentation for packages should default to building with all
traits enabled so that all API documentation is visible.

## Future directions

### Consider traits during dependency resolution

The implementation to this proposal only considers traits **after** the
dependency resolution when constructing the module graph. This is inline with
how platform specific dependencies are currently handled. In the future, both
platform specific dependencies and traits can be taken into consideration during
dependency resolution to avoid fetching an optional dependency that is not
enabled by a trait. Changing this **doesn't** require a Swift evolution proposal
since it is just an implementation detail of how dependency resolution currently
works.

### Integrated compiler trait checking

The current proposal passes enabled traits via custom defines to the compiler
and code can check it using regular define checks (`#if DEFINE`). In the future,
we can extend the compiler to make it aware of package traits to allows syntax
like `#if trait(FOO)` or implement an extensible configuration macro similar to
Rust's `cfg` macro.

### Enabled trait compile time checking

Since trait unification is done for every package in the graph during build time
the information which module enabled which trait of its dependencies is lost.
As a consequence it might be that a package accidentally uses an API from a
dependency which is guarded by a trait that another package in the graph has
enabled. Since the traits that any one package in the graph enables on its
dependencies are not considered part of the semantic version, it can happen that
disabling a trait could result in breaking a build. In the future, we could
integrate trait checking further into the compiler where it understands if an
API is only available if a certain trait is set.

> Cargo currently [treats this
similar](https://users.rust-lang.org/t/is-disabling-features-of-a-dependency-considered-a-breaking-change/94302/2)
and doesn't consider disabling a cargo feature a breaking change. 

### Different default traits depending on platform

A future evolution could allow to mark traits as default depending on the
platform that the package is build on. This would allow packages such as the
`swift-openapi-generator` to default the used transport depending on the
platform which makes it even easier to offer users the best out of box
experience. This is left as a future evolution since it intersects interestingly
with the future direction "Consider traits during dependency resolution". If
default traits depend on the target build platform then this must be an input to
the dependency resolution.

### Globally configured traits

One use-case where mutually exclusive traits are used is to configure the
behaviour of a single package globally. This means that only the final
executable is deciding what trait to enable. In a future proposal, we could
introduce a new setting for marking a trait as globally configured and check
that only executable targets are enabling such a trait.

### Tooling to test different trait combinations

Since a single package should support building with any combination of traits,
it would be helpful to offer package authors tooling to build and test all
combinations. A new option `--all-trait-combinations` could be added to
`swift test/build/run` make testing all combinations easy as possible.

### Surface traits in documentation

If the compiler gains knowledge about package traits in the future, we could
extract information if a public API is guarded by a trait and surface this in
the documentation.

## Alternatives considered

### Different naming

During the implementation and writing of the proposal different names for
_package traits_ have been considered such as:
- Package features
- Package optional features
- Package options
- Package parameters
- Package flags
- Package configuration

A lot of the other considered names have other meanings in the language already.
For example `feature` is already used in expressing compiler feature via
`enable[Upcoming|Experimental]Feature` and the `hasFeature` check.

_Package traits_ are also consistent with [the "traits" concept in the
`swift-testing`
library](https://github.com/apple/swift-testing/blob/25d0eed9b339de36365ff16deb9a3d9c64322f1c/Sources/Testing/Traits/Trait.swift#L22).

### Using SPI instead

During the investigation how to solve the optional dependency problem `@_spi`
was considered; however, the problem with `@_spi` is that the code is still
compiled and present in the final binary. Optional dependencies can't work like
this since the symbols potentially aren't present during compile time.

### Enum based traits

Traits could be expressed via an enum in the package description which would
make sure that they are statically typed. This proposal decided to use `String`
based trait names instead to align with the other definitions inside the package
description such as `targets` or `products`.

## Prior art

Other dependency managers have similar features to control optional dependencies
and conditional compilation.

- [Cargo](https://doc.rust-lang.org/cargo/) has [optional features](https://doc.rust-lang.org/cargo/reference/features.html) that allow conditional compilation and optional dependencies.
- [Maven](https://maven.apache.org/) has [optional dependencies](https://maven.apache.org/guides/introduction/introduction-to-optional-and-excludes-dependencies.html).
- [Gradle](https://gradle.org/) has [feature variants](https://docs.gradle.org/current/userguide/feature_variants.html) that allow conditional compilation and optional dependencies.
- [Go](https://golang.org/) has [build constraints](https://golang.org/pkg/go/build/#hdr-Build_Constraints) which can conditionally include a file.
- [pip](https://pypi.org/project/pip/) dependencies can have [optional dependencies and extras](https://setuptools.pypa.io/en/latest/userguide/dependency_management.html#optional-dependencies).
- [Hatch](https://hatch.pypa.io/latest/) offers [optional dependencies](https://hatch.pypa.io/latest/config/metadata/#optional) and [features](https://hatch.pypa.io/latest/config/dependency/#features).