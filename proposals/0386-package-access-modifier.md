# New access modifier: `package`

* Proposal: [SE-0386](0386-package-access-modifier.md)
* Authors: [Ellie Shin](https://github.com/elsh), [Alexis Laferriere](https://github.com/xymus)
* Review Manager: [John McCall](https://github.com/rjmccall)
* Status: **Implemented (Swift 5.9)**
* Implementation: [apple/swift#61546](https://github.com/apple/swift/pull/62700), [apple/swift#62704](https://github.com/apple/swift/pull/62704), [apple/swift#62652](https://github.com/apple/swift/pull/62652), [apple/swift#62652](https://github.com/apple/swift/pull/62652)
* Review: ([pitch](https://forums.swift.org/t/new-access-modifier-package/61459)) ([first review](https://forums.swift.org/t/se-0386-package-access-modifier/62808)) ([second review](https://forums.swift.org/t/second-review-se-0386-package-access-modifier/64086)) ([acceptance](https://forums.swift.org/t/accepted-se-0386-package-access-modifier/64904))
* Previous Revision: [1](https://github.com/swiftlang/swift-evolution/blob/28fd2fb9b7258117f912cec5e5f7eb178520fbf2/proposals/NNNN-package-access-modifier.md), [2](https://github.com/swiftlang/swift-evolution/blob/32e51946296f67be79a58a8c23eb9d7460a06232/proposals/0386-package-access-modifier.md), [3](https://github.com/swiftlang/swift-evolution/blob/4a3a11b18037526cf8d83a9d10b22b94890727e8/proposals/0386-package-access-modifier.md)

## Introduction

This proposal introduces `package` as a new access modifier.  Currently, to access a symbol in another module, that symbol needs to be declared `public`.  However, a symbol being `public` allows it to be accessed from any module at all, both within a package and from outside of a package, which is sometimes undesirable.  We need a new access modifier to enable more control over the visibility scope of such symbols.  

## Motivation

At the most basic level, every Swift program is just a collection of declarations: functions, types, variables, and so on.  In principle, every level of organization above this is arbitrary; all of those declarations could be piled into a single file, compiled, and run.  In reality, Swift programs are organized into separate files, directories, libraries, and so on.  At each level, this organization reflects programmer judgment about relationships, both in the code and in how it is developed.

As a language, Swift recognizes some of these levels.  Modules are the smallest unit of library structure, with an independent interface and non-cyclic dependencies, and it makes sense for Swift to recognize that in both namespacing and access control.  Files are the smallest grouping beneath that and are often used to collect tightly-related declarations, so they also make sense to respect in access control.

Packages, as expressed by the Swift Package Manager, are a unit of code distribution.  Some packages contain just a single module, but it's frequently useful to split a package's code into multiple modules.  For example, when a module contains some `internal` helper APIs, those APIs can be split out into a utility module and maybe reused by other modules or packages.

However, because Swift does not recognize organizations of code above the module level, it is not possible to create APIs like this that are purely internal to the package.  To be usable from other modules within the package, the API must be public, but this means it can also be used outside of the package.  This allows clients to form unwanted source dependencies on the API.  It also means the built module has to export the API, which has negative implications for code size and performance.

For example, here’s a scenario where a client has access to a utility API from a package it depends on.  The client `App` could be an executable or an Xcode project.  It depends on a package called `gamePkg`, which contains two modules, `Game` and `Engine`.  


Module `Engine` in `gamePkg`:
```swift
public struct MainEngine {
    public init() { ...  }
    // Intended to be public
    public var stats: String { ...  }
    // A helper function made public only to be accessed by Game
    public func run() { ...  }
}
```

Module `Game` in `gamePkg`:
```swift
import Engine

public func play() {
    MainEngine().run() // Can access `run` as intended since it's within the same package
}
```

Client `App` in `appPkg`:
```swift
import Game
import Engine

let engine = MainEngine()
engine.run() // Can access `run` from App even if it's not an intended behavior
Game.play()
print(engine.stats) // Can access `stats` as intended
```

In the above scenario, `App` can import `Engine` (a utility module in `gamePkg`) and access its helper API directly, even though the API is not intended to be used outside of its package.

Allowing this kind of unintended public access to package APIs is especially bad because packages are a unit of code distribution. Swift wants to encourage programs to be divided into modules with well-defined interfaces, so it enforces the boundaries between modules with access control. Despite being divided this way, it's not uncommon for closely-related modules to be written by closely-related (or even the same) people. Access control between such modules still serves a purpose — it promotes the separation of concerns — but if a module's interface needs to be fixed, that's usually easy to coordinate, maybe even as simple as a single commit. However, packages allow code to be shared much more broadly than a single small organization. The boundaries between packages often represent significant differences between programmers, making coordination around API changes much more difficult. For example, the developers of an open source package generally don't know most of their clients, and the standard recommendation is for such packages to only ever remove existing APIs in major-version releases. It's therefore particularly important to allow programmers to enforce these boundaries between packages.

## Proposed solution

Our goal is to introduce a mechanism to Swift to recognize a package as a unit in the aspect of access control.  We propose to do so by introducing a new access modifier called `package`.  The `package` access modifier allows symbols to be accessed from outside of their defining module, but only from other modules in the same package.  This helps to set clear boundaries between packages.  

## Detailed design

### `package` Keyword

`package` is introduced as an access modifier.  It cannot be combined with other access modifiers.
`package` is a contextual keyword, so existing declarations named `package` will continue to work.  This follows the precedent of `open`, which was also added as a contextual keyword.  For example, the following is allowed:

```swift
package var package: String {...}
```

### Declaration Site

The `package` keyword is added at the declaration site.  Using the scenario above, the helper API `run` can be declared with the new access modifier like so:

Module `Engine`:
```swift
public struct MainEngine {
    public init() { ...  }
    public var stats: String { ...  }
    package func run() { ...  }
}
```

The `package` access modifier can be used anywhere that the existing access modifiers can be used, e.g. `class`, `struct`, `enum`, `func`, `var`, `protocol`, etc.

Swift requires that the declarations used in certain places (such as the signature of a function) be at least as accessible as the containing declaration. For the purposes of this rule, `package` is less accessible than `open` and `public` and more accessible than `internal`, `fileprivate`, and `private`. For example, a `public` function cannot use a `package` type in its parameters or return type, and a `package` function cannot use an `internal` type in its parameters or return type. Similarly, an `@inlinable` `public` function cannot use a `package` declaration in its implementation, and an `@inlinable` `package` function cannot use an `internal` declaration in its implementation.

### Use Site

The `Game` module can access the helper API `run` since it is in the same package as `Engine`.

Module `Game`:
```swift
import Engine

public func play() {
    MainEngine().run() // Can access `run` as it is a package symbol in the same package
}
```

However, if a client outside of the package tries to access the helper API, it will not be allowed.

Client `App`:
```swift
import Game
import Engine

let engine = MainEngine()
engine.run() // Error: cannot find `run` in scope
```

### Package Names

Swift as a language leaves it up to the build system to define the boundaries of a package.  The compiler considers two modules to belong to the same package if they were built with the same package name, which is just a Unicode string.  The package name is not exposed in the source language, so its exact contents are not significant as long as it is unique to a "package".

A new flag `-package-name` is passed down to a commandline invocation, as follows.

```sh
swiftc -module-name Engine -package-name gamePkg ...
swiftc -module-name Game -package-name gamePkg ...
swiftc -module-name App -package-name appPkg ...
```

When building the `Engine` module, the package name `gamePkg` is recorded in the built interface to the module.  When building `Game`, its package name `gamePkg` is compared with the package name recorded in `Engine`'s built interface; since they match, `Game` is allowed to access `Engine`'s `package` declarations.  When building `App`, its package name `appPkg` is different from `gamePkg`, so it is not allowed to access `package` symbols in either `Engine` or `Game`, which is what we want.

If `-package-name` is not given, the `package` access modifier is disallowed.  Swift code that does not use `package` access will continue to build without needing to pass in `-package-name`.  Modules built without a package name are never considered to be in the same package as any other module.

The build system should make a best effort to ensure that package names are unique.  The Swift Package Manager already has a concept of a package identity string for every package.  This string is verified to be unique, and it already works as a package name, so SwiftPM will pass it down automatically.  Other build systems such as Bazel may need to introduce a new build setting for a package name.  Since it needs to be unique, a reverse-DNS name may be used to avoid clashing.

If a target needs to be excluded from the package boundary, that can be done with a new `packageAccess` setting in the manifest, like so: 

```swift
  .target(name: "Game", dependencies: ["Engine"], packageAccess: false)
```

The `packageAccess` setting is set to `true` by default, and the target is built with `-package-name PACKAGE_ID` where `PACKAGE_ID` is the manifest's package identifier.  If `packageAccess` is set to `false`, `-package-name` is not passed when building the target, thus the target has no access to any package symbols; it essentially acts as if it's a client outside of the package. This would be useful for an example app or a black-box test target in the package.

### Package Symbols Distribution

When the Swift frontend builds a `.swiftmodule` file directly from source, the file will include the package name and all of the `package` declarations in the module.  When the Swift frontend builds a `.swiftinterface` file from source, the file will include the package name, but it will put `package` declarations in a secondary `.package.swiftinterface` file.  When the Swift frontend builds a `.swiftmodule` file from a `.swiftinterface` file that includes a package name, but it does not have the corresponding `.package.swiftinterface` file, it will record this in the `.swiftmodule`, and it will prevent this file from being used to build other modules in the same package.

### Package Symbols and `@inlinable`

`package` types can be made `@inlinable`.  Just as with `@inlinable public`, not all symbols are usable within the body of `@inlinable package`: they must be `open`, `public`, or `@usableFromInline`. The `@usableFromInline` attribute can be applied to `package` besides `internal` declarations. These attributed symbols are allowed in the bodies of `@inlinable public` or `@inlinable package` declarations (that are defined anywhere in the same package).  Just as with `internal` symbols, the `package` declarations with `@usableFromInline` or `@inlinable` are stored in the public `.swiftinterface` for a module. 

Here's an example.

```swift
func internalFuncA() {}
@usableFromInline func internalFuncB() {}

package func packageFuncA() {}
@usableFromInline package func packageFuncB() {}

public func publicFunc() {}

@inlinable package func pkgUse() {
    internalFuncA() // Error
    internalFuncB() // OK
    packageFuncA() // Error
    packageFuncB() // OK
    publicFunc() // OK
}

@inlinable public func publicUse() {
    internalFuncA() // Error
    internalFuncB() // OK
    packageFuncA() // Error
    packageFuncB() // OK
    publicFunc() // OK
}
```
 
### Subclassing and Overrides

Access control in Swift usually doesn't distinguish between different kinds of use.  If a program has access to a type, for example, that gives the programmer a broad set of privileges: the type name can be used in most places, values of the type can be borrowed, copied, and destroyed, members of the type can be accessed (up to the limits of their own access control), and so on.  This is because access control is a tool for enforcing encapsulation and allowing the future evolution of code.  Broad privileges are granted because restricting them more precisely usually doesn't serve that goal.

However, there are two exceptions.  The first is that Swift allows `var` and `subscript` to restrict mutating accesses more tightly than read-only accesses; this is done by writing a separate access modifier for the setter, e.g. `private(set)`.  The second is that Swift allows classes and class members to restrict subclassing and overriding more tightly than normal references; this is done by writing `public` instead of `open`.  Allowing these privileges to be separately restricted serves the goals of promoting encapsulation and evolution.

Because setter access levels are controlled by writing a separate modifier from the primary access, the syntax naturally extends to allow `package(set)`.  However, subclassing and overriding are controlled by choosing a specific keyword (`public` or `open`) as the primary access modifier, so the syntax does not extend to `package` the same way.  This proposal has to decide what `package` by itself means for classes and class members.  It also has to decide whether to support the options not covered by `package` alone or to leave them as a possible future direction.

Here is a matrix showing where symbols with each current access level can be used or overridden:

<table>
<thead>
<tr>
<th></th>
<th>Accessible Anywhere</th>
<th>Accessible in Module</th>
</tr>
</thead>
<tbody>
<tr>
<th>Subclassable Anywhere</th>
<td align="center">open</td>
<td align="center">(illegal)</td>
</tr>
<tr>
<th>Subclassable in Module</th>
<td align="center">public</td>
<td align="center">internal</td>
</tr>
<tr>
<th>Subclassable Nowhere</th>
<td align="center">public final</td>
<td align="center">internal final</td>
</tr>
</tbody>
</table>

With `package` as a new access modifier, the matrix is modified like so:

<table>
<thead>
<tr>
<th></th>
<th>Accessible Anywhere</th>
<th>Accessible in Package</th>
<th>Accessible in Module</th>
</tr>
</thead>
<tbody>
<tr>
<th>Subclassable Anywhere</th>
<td align="center">open</td>
<td align="center">(illegal)</td>
<td align="center">(illegal)</td>
</tr>
<tr>
<th>Subclassable in Package</th>
<td align="center">?(a)</td>
<td align="center">?(b)</td>
<td align="center">(illegal)</td>
</tr>
<tr>
<th>Subclassable in Module</th>
<td align="center">public</td>
<td align="center">package</td>
<td align="center">internal</td>
</tr>
<tr>
<th>Subclassable Nowhere</th>
<td align="center">public final</td>
<td align="center">package final</td>
<td align="center">internal final</td>
</tr>
</tbody>
</table>


This proposal takes the position that `package` alone should not allow subclassing or overriding outside of the defining module.  This is consistent with the behavior of `public` and makes `package` fit into a simple continuum of ever-expanding privileges.  It also allows the normal optimization model of `public` classes and methods to still be applied to `package` classes and methods, implicitly making them `final` when they aren't subclassed or overridden, without requiring a new "whole package optimization" build mode.

However, this choice leaves no way to spell the two combinations marked in the table above with `?`.  These are more complicated to design and implement and are discussed in Future Directions.

## Future Directions

### Subclassing and Overrides

The entities marked with `?(a)` and `?(b)` from the matrix above both require accessing and subclassing cross-modules in a package (`open` within a package). The only difference is that (b) hides the symbol from outside of the package and (a) makes it visible outside. Use cases involving (a) should be rare but its underlying flow should be the same as (b) except its symbol visibility. 

Potential solutions include introducing new keywords for specific access combinations (e.g. `packageopen`), allowing `open` to be access-qualified (e.g. `open(package)`), and allowing access modifiers to be qualified with specific purposes (e.g. `package(override)`).

### Package-Private Modules

Sometimes entire modules are meant to be private to the package that provides them.  Allowing this to be expressed directly would allow these utility modules to be completely hidden outside of the package, avoiding unwanted dependencies on the existence of the module.  It would also allow the build system to automatically namespace the module within the package, reducing the need for [explicit module aliases](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0339-module-aliasing-for-disambiguation.md) when utility modules of different packages share a name (such as `Utility`) or when multiple versions of a package need to be built into the same program.

### Grouping Within A Package

The basic language design of this proposal can work for any group of related modules, but the application of that design in SPM allows only a single such group per SPM package.  Developers with complex SPM packages sometimes find that they have multiple architectural "layers" within a single package and may wish to make `package` apply only within a layer.  Logically, it makes some sense to put each layer in its own package.  Pragmatically, because different SPM packages must currently live in separate repositories and be independently versioned, splitting a package that way introduces a huge amount of extra complexity to the development process, and it is not something that should be done casually.

There are several reasonable ways that SPM could evolve to support multiple layers within a single package repository.  One would be to allow targets to be grouped within a manifest, such as by adding a `group` parameter to `.target`.  An earlier version of this proposal suggested this and even designed the `packageAccess:` exclusion feature around it.  However, this would tend to lead to large, complex manifests that mingled the details of all the layers together.  A very different approach would be to allow the creation of sub-packages within a repository, each with its own manifest.  SPM would treat these sub-packages as logically separate units that happen to share a single repository and version.  Because they would be described in independent manifests, they would feel like different packages, and it would make sense for `package` access to be scoped within them.

### Optimizations

* A package can be treated as a resilience domain, even with library evolution enabled which makes modules resilient.  The Swift frontend will assume that modules defined in the same package will always be rebuilt together and do not require a resilient ABI boundary between them. This removes the need for performance and code size overhead introduced by ABI-resilient code generation, and it also eliminates language requirements such as `@unknown default` for a non-`frozen enum`.

* By default, `package` symbols are exported in the final libraries/executables.  It would be useful to introduce a build setting that allows users to hide package symbols for statically linked libraries; this would help with code size and build time optimizations.

## Source Compatibility

The new `package` access modifier is a contextual keyword.  Existing symbols that are named `package` should not require renaming, and existing source code should continue to work.

## Effect on ABI stability

Boundaries between separately-built modules within a package are still potentially ABI boundaries.  The ABI for package symbols is not different from the ABI for public symbols, although it might be considered in the future to add an option to not export package symbols that can be resolved within an image.  

## Alternatives considered

### `@_spi`

One workaround for the scenario in the Motivation would be to use the `@_spi(groupName)` attribute, which allows part of the API of a module to be hidden unless it is imported in a special way that explicitly requests access to it.  This is an unsatisfying alternative to package-level access control because it is designed around a very different situation.  An SPI is a "hole" in the normal public interface, one meant for the use of a specific client.  That client is typically outside of the module's normal code-distribution boundary, but the module authors still have a cooperative working relationship.  This relationship is reflected in the design of `@_spi` in multiple ways:

* First, access to the SPI is granted to a specific client by name.  This is a clear and unavoidable communication of intent about who is meant to be using the SPI.  Other clients can still pose as this client and use the SPI, but that would be a clear breach of trust with predictable consequences.

* Second, clients must explicitly request the SPI by name.  This means that clients must opt in to using the SPI in every file, which works to limit its accidental over-use even by the intended client.  It also means that SPI use is obvious in the code, which code reviewers can see and raise questions about, and which SPI authors can easily find with a code search.

The level of care implied by these properties is appropriate for a carefully-targeted hole in an API that must cross a code-distribution boundary and will therefore require equal amounts of care to ever modify or close.  That rarely applies to two modules within the same package, where a package-level interface can ideally be changed with just a quick edit to a few different parts of a repository.  The `@_spi` attribute is intentionally designed to not be as lightweight as a package-local change should be.

* `@_spi` would also not be easy to optimize.  By design, clients of an SPI can be anywhere, making it effectively part of the public ABI of a module.  To avoid exporting an SPI, the build system would have to know about that specific SPI group and promise the compiler that it was only used in the current built image.  Recognizing that all of the modules in a package are being linked into the same image and can be optimized together is comparatively easy for a build system and so is a much more feasible future direction.

### `@_implementationOnly`

Another workaround for the scenario in the Motivation is to use the `@_implementationOnly` attribute on the import of a module.  This attribute causes the module to be imported only for the use of the current module; clients of the current module don't implicitly transitively import the target module, and the symbols of the target module are restricted from appearing in the `public` API of the current module.  This would prevent clients from accidentally using APIs from the target module.  However, this is a very incomplete workaround for the lack of package-level access control.  For one, it doesn't actually prevent access to the module, which can still be explicitly imported and used.  For another, it only works on an entire module at a time, so a module cannot restrict some of its APIs to the package while making others available publicly.  Taming transitive import would be a good future direction for Swift, but it does not solve the problems of package-level APIs.

### Other Workarounds

There are a few other workarounds to the absence of package-level access control, such as using `@testable` or the `-disable-access-control` flag.  These are hacky subversions of Swift's language design, and they severely undermine the use of module boundaries for encapsulation.  `-disable-access-control` is also an unstable and unsupported feature that can introduce build failures by causing symbol name collisions.

### Introduce Submodules

Instead of adding a new package access level above modules, we could allow modules to contain other modules as components.  This is an idea often called "submodules".  Packages would then define an "umbrella" module that contains the package's modules as components.  However, there are several weaknesses in this approach:

* It doesn't actually solve the problem by itself.  Submodule APIs would still need to be able to declare whether they're usable outside of the umbrella or not, and that would require an access modifier.  It might be written in a more general way, like `internal(MyPackage)`, but that generality would also make it more verbose.

* Submodule structure would be part of the source language, so it would naturally be source- and ABI-affecting.  For example, programmers could use the parent module's name to qualify identifiers, and symbols exported by a submodule would include the parent module's name.  This means that splitting a module into submodules or adding an umbrella parent module would be much more impactful than desired; ideally, those changes would be purely internal and not change a module's public interface.  It also means that these changes would end up permanently encoding package structure.

* The "umbrella" submodule structure doesn't work for all packages.  Some packages include multiple "top-level" modules which share common dependencies.  Forcing these to share a common umbrella in order to use package-private dependencies is not desirable.

* In a few cases, the ABI and source impact above would be desirable.  For example, many packages contain internal Utility modules; if these were declared as submodules, they would naturally be namespaced to the containing package, eliminating spurious collisions.  However, such modules are generally not meant to be usable outside of the package at all.  It is a reasonable future direction to allow whole modules to be made package-private, which would also make it reasonable to automatically namespace them.

### `@usableFromPackageInline`

An earlier version of this proposal included a new attribute `@usableFromPackageInline`, which would have allowed an `internal` declaration to be used in the body of an `@inlinable package` declaration, but not in an `@inlinable public` declaration. Under the logic of this proposal, there is no good reason to make a declaration `@usableFromPackageInline internal` instead of simply `package`: the uses of the latter will be restricted to the package and therefore by assumption can still be easily found and reviewed. Furthermore, it is a goal of the Swift project to not require extensive `@inlinable` annotations just to enable basic optimizations between modules: there should be little reason in the long run to have an `@inlinable package` declaration at all. Therefore this attribute has been removed from the proposal.

## Acknowledgments

Doug Gregor, Becca Royal-Gordon, Allan Shortlidge, Artem Chikin, and Xi Ge provided helpful feedback and analysis as well as code reviews on the implementation.
