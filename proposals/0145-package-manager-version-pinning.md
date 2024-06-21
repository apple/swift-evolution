# Package Manager Version Pinning

* Proposal: [SE-0145](0145-package-manager-version-pinning.md)
* Author: [Daniel Dunbar](https://github.com/ddunbar), [Ankit Aggarwal](https://github.com/aciidb0mb3r), [Graydon Hoare](https://github.com/graydon)
* Review Manager: [Anders Bertelrud](https://github.com/abertelrud)
* Status: **Implemented (Swift 3.1)**
* Decision Notes: [Rationale](https://forums.swift.org/t/swift-evolution-accepted-se-0145-package-manager-version-pinning-revised/4653)
* Previous Revision: [1](https://github.com/swiftlang/swift-evolution/blob/91725ee83fa34c81942a634dcdfa9d2441fbd853/proposals/0145-package-manager-version-pinning.md)
* Previous Discussion: [Email Thread](https://forums.swift.org/t/review-se-0145-package-manager-version-pinning/4405/15)

## Introduction

This is a proposal for adding package manager features to "pin" or "lock" package dependencies to particular versions.

## Motivation

As used in this proposal, version pinning refers to the practice of controlling
*exactly* which specific version of a dependency is selected by the dependency
resolution algorithm, *independent from* the semantic versioning
specification. Thus, it is a way of instructing the package manager to select a
particular version from among all of the versions of a package which could be
chosen while honoring the dependency constraints.

### Terminology

*We have chosen to use "pinning" to refer to this feature, over "lockfiles", since
the term "lock" is already overloaded between POSIX file locks and locks in
concurrent programming.*

### Use Cases

Our proposal is designed to satisfy several different use cases for such a behavior:

1. Standardizing team workflows

   When collaborating on a package, it can be valuable for team members (and
   continuous integration) to all know they are using the same exact version of
   dependencies, to avoid "works for me" situations.

   This can be particularly important for certain kinds of open source projects
   which are actively being cloned by new users, and which want to have some
   measure of control around exactly which available version of a dependency is
   selected.

2. Difficult to test packages or dependencies

   Complex packages which have dependencies which may be hard to test, or hard
   to analyze when they break, may choose to maintain careful control over what
   versions of their upstream dependencies they recommend -- even if
   conceptually they regularly update those recommendations following the true
   semantic version specification of the dependency.

3. Dependency locking w.r.t. deployment

   When stabilizing a release for deployment, or building a version of a package
   for deployment, it is important to be able to lock down the exact versions of
   dependencies in use, so that the resulting product can be exactly recreated
   later if necessary.

### Current Behavior

The package manager *NEVER* updates a locally cloned package from its current
version without explicit user action (`swift package update`). We anticipate
encouraging users to update to newer versions of packages when viable, but this
has not yet been proposed or implemented.

Whenever a package is operated on locally in a way that requires its
dependencies be present (typically a `swift build`, but it could also be `swift
package fetch` or any of several other commands), the package manager will fetch
a complete set of dependencies. However, when it does so, it attempts to get
versions of the missing dependencies compatible with the existing dependencies.

From a certain perspective, the package manager today is acting as if the local
clones were "pinned", however, there has heretofore been no way to share that
pinning information with other team members. This proposal is aimed at
addressing that.

## Proposed solution

We will introduce support for an **optional** new file `Package.pins` adjacent
to the `Package.swift` manifest, called the "pins file". We will also introduce
a number of new commands (see below) for maintaining the pins file.

This file will record the active version pin information for the package,
including data such as the package identifier, the pinned version, and explicit
information on the pinned version (e.g., the commit hash/SHA for the resolved
tag).

The exact file format is unspecified/implementation defined, however, in
practice it will be a JSON data file.

This file *may* be checked into SCM by the user, so that its effects apply to
all users of the package. However, it may also be maintained only locally (e.g.,
placed in the `.gitignore` file). We intend to leave it to package authors to
decide which use case is best for their project. We will **recommend** that it
not be checked in by library authors, at least for released versions, since pins
are not inherited and thus this information may be confusing. We may codify this
recommendation into a warning in a future package manager workflow which
provided assistance in publishing package versions.

In the presence of a top-level `Package.pins` file, the package manager will
respect the pinned dependencies recorded in the file whenever it needs to do
dependency resolution (e.g., on the initial checkout or when updating).

In the absence of a top-level `Package.pins` file, the package manager will
operate based purely on the requirements specified in the package manifest, but
will then automatically record the choices it makes into a `Package.pins` file
as part of the "automatic pinning" feature. The goal of this behavior is to
encourage reproducible behavior among package authors who share the pin file
(typically by checking it in).

We will also provide an explicit mechanism by which package authors can opt out
of the automatic pinning default for their package.

The pins file will *not* override manifest specified version requirements and it
will be a warning if there is a conflict between the pins and the manifest
specification.

The pins file will also not influence dependency resolution for dependent packages;
for example if application A depends on library B which in turn depends on library C,
then package resolution for application A will use the manifest of library B to learn
of the dependency on library C, but ignore any `Package.pins` file belonging to
library B when deciding which version of library C to use.

## Detailed Design

1. We will add a new command `pin` to `swift package` tool with following semantics:

	```
	$ swift package pin ( [--all] | [<package-name>] [<version>] ) [--message <message>]
	```

	The `package-name` refers to the name of the package as specified in its manifest.

	This command pins one or all dependencies. The command which pins a single version can optionally take a specific version to pin to, if unspecified (or with `--all`) the behavior is to pin to the current package version in use. Examples:  
	* `$ swift package pin --all` - pins all the dependencies.
	* `$ swift package pin Foo` - pins `Foo` at current resolved version.
	* `$ swift package pin Foo 1.2.3` - pins `Foo` at 1.2.3. The specified version should be valid and resolvable.

  The `--message` option is an optional argument to document the reason for pinning a dependency. This could be helpful for user to later remember why a dependency was pinned. Example:   
 
	`$ swift package pin Foo --message "The patch updates for Foo are really unstable and need screening."`

  NOTE: When we refer to dependencies in the context of pinning, we are
  referring to *all* dependencies of a package, i.e. the transitive closure of
  its immediate dependencies specified in the package manifest. One of the
  important ways in which pinning is useful is because it allows specifying a
  behavior for the closure of the dependencies outside of them being named in
  the manifest.
   
2. We will add two additional commands to `pin` as part of the automatic pinning
   workflow (see below):

	```
	$ swift package pin ( [--enable-autopin] | [--disable-autopin] )
	```

   These will enable or disable automatic pinning for the package (this state is
   recorded in the `Package.pins` file).

   These commands are verbose, but the expectation is that they are very
   infrequently run, just to establish the desired behavior for a particular
   project, and then the pin file (containing this state) is checked in to
   source control.

3. We will add a new command `unpin`:

	```
	$ swift package unpin ( [--all] | [<package-name>] )
	``` 
	This is the counterpart to the pin command, and unpins one or all packages.

   It is an error to attempt to `unpin` when automatic pinning is enabled.

4. We will fetch and resolve the dependencies when running the pin commands, in case we don't have the complete dependency graph yet.

5. We will extend the workflow for update to honor version pinning, that is, it will only update packages which are unpinned, and it will only update to versions which can satisfy the existing pins. The update command will, however, also take an optional argument `--repin`:

	```
	$ swift package update [--repin]
	```

	* The update command will warn if there are no unpinned packages which can be updated.

	* Otherwise, the behavior is to update all unpinned packages to the latest possible versions which can be resolved while respecting the existing pins.

	* The `[--repin]` argument can be used to lift the version pinning restrictions. In this case, the behavior is that all packages are updated, and packages which were previously pinned are then repinned to the latest resolved versions.

   When automatic pinning is enabled, `package update` would by default have absolutely no effect without `--repin`. Thus, we will make `package update` act as if `--repin` was specified whenever automatic pinning is enabled. This is a special case, but we believe it is most likely to match what the user expects, and avoids have a command syntax which has no useful behavior in the automatic pinning mode.

6. The update and checkout will both emit logs, notifying the user that pinning is in effect.

7. The `swift package show-dependencies` subcommand will be updated to indicate if a dependency is pinned.


### Automatic Pinning

The package manager will have automatic pinning enabled by default (this is
equivalent to `swift package pin --enable-autopin`), although package project
owners can choose to disable this if they wish to have more fine grained control
over their pinning behavior.

When automatic pinning is enabled, the package manager will automatic record all
package dependencies in the `Package.pins` file. If package authors do not check
this file into their source control, the behavior will typically be no different
than the existing package manager behavior (one exception is the `package
update` behavior described above).

If a package author does check the file into source control, the effect will be
that anyone developing directly on this package will end up sharing the same
dependency versions (and modifications will be committed as part of the SCM
history).

The automatic pinning behavior is an extension of the behaviors above, and works
as follows:

 * When enabled, the package manager will write all dependency versions into the
   pin file after any operation which changes the set of active working
   dependencies (for example, if a new dependency is added).

 * A package author can still change the individual pinned versions using the
   `package pin` commands, these will simply update the pinned state.

 * Some commands do not make sense when automatic pinning is enabled; for
   example, it is not possible to `unpin` and attempts to do so will produce an
   error.

Since package pin information is **not** inherited across dependencies, our
recommendation is that packages which are primarily intended to be consumed by
other developers either *disable* automatic pinning or put the `Package.pins`
file into `.gitignore`, so that users are not confused why they get different
versions of dependencies that are those being used by the library authors while
they develop.


## Future Directions

We have intentionally kept the pin file format an implementation detail in order
to allow for future expansion. For example, we would like to consider embedding
additional information on a known tag (like its SHA, when using Git) in the pins
file as a security feature, to prevent man-in-the-middle attacks on parts of the
package graph.

## Impact on existing code

There will be change in the behaviors of `swift build` and `swift package update` in presence of the pins file, as noted in the proposal, however the existing package will continue to build without any modifications.

## Alternative considered

### Minimal pin feature set

A prior version of this proposal did not pin by default. Since this proposal
includes this behavior, we could in theory eliminate the fine grained pinning
feature set we expose, like `package pin <name>` and `package unpin`.

However, we believe it is important for package authors to retain a large amount
of control over how their package is developed, and we wish the community to
aspire to following semantic versioning strictly. For that reason, we wanted to
support mechanisms so that package authors wishing to follow this model could
still pin individual dependencies.


### Pin by default

_This discussion is historical, from a prior version of a proposal which did not
include the automatic pinning behavior; which we altered the proposal for. We
have left it in the proposal for historical context._

Much discussion has revolved around a single policy-default question: whether
SwiftPM should generate a pins file as a matter of course any time it
builds. This is how some other package managers work, and it is viewed as a
conservative stance with respect to making repeatable builds more likely between
developers. Developers will see the pins file and will be likely to check it in
to their SCM system as a matter of convention. As a side effect, other
developers cloning and trying out the package will then end up using the same
dependencies the developer last published.

While pinning does reduce the risk of packages failing to build, this practice
discourages the community from relying on semver compatibility to completely
specify what packages are compatible. That in turn makes it more likely for
packages to fail to correctly follow the semver specification when publishing
versions. Unfortunately, when packages don't correctly follow semver then it
requires downstream clients to overspecify their dependency constraints since
they cannot rely on the package manager automatically picking the appropriate
version.

Overconstraint is much more of a risk in Swift than in other languages
using this style of package management. Specifically: Swift does not support
linking multiple versions of a dependency into the same artifact at the same
time. Therefore the risk of producing a "dependency hell" situation, in which
two packages individually build but _cannot be combined_ due to over-constrained
transitive dependencies, is significantly higher than in other languages.
Changing the compiler support in this area is not something which is currently
planned as a feature, so our expectation is that we will have this limitation
for a significant time.

For example, if package `Foo` depends on library `LibX` version 1.2,
and package `Bar` depends on `LibX` 1.3, and these are _specific_
version constraints that do not allow version-range variation,
then SwiftPM will _not_ allow building a product that depends on
both `Foo` and `Bar`: their requirements for `LibX` are incompatible.
Where other package managers will simultaneously link two versions
of `LibX` -- and hope that the differing simultaneous uses of
`LibX` do not cause other compile-time or run-time errors -- 
SwiftPM will simply fail to resolve the dependencies.

We therefore wish to encourage library authors to keep their
packages building and testing with as recent and as wide a range
of versions of their dependencies as possible, and guard more
vigorously than other systems against accidental overconstraint.
One way to encourage this behavior is to avoid emitting pins files
by default.

We also believe that if packages default to exposing their pin files as part of
their public package, there is a substantial risk that when developers encounter
build failures they will default to copying parts of those pinned versions into
their manifest, rather than working to resolve the semver specification issues
in the dependencies. If this behavior becomes common place, it may even become
standard practice to do this proactively, simply to avoid the potential of
breakage.

This practice is likely because it resolves the immediate issue (a build
failure) without need for external involvement, but if it becomes widespread
then it has a side-effect of causing significant overconstraint of packages
(since a published package may end up specifying only a single version it is
compatible with).

Finally, we are also compelled by several pragmatic implications of an approach
which optimizes for reliance on the semver specifications:

1. We do not yet have a robust dependency resolution algorithm we can rely
   on. The complexity of the algorithm is in some ways relative to the degree of
   conflicts we expect to be present in the package graph (for example, this may
   mean we need to investigate significantly more work in optimizing its
   performance, or in managing its diagnostics).

2. The Swift package manager and its ecosystem is evolving quickly, and we
   expect it will continue to do so for some time. As a consequence, we
   anticipate that packages will frequently be updated simply to take advantage
   of new features. Optimizing for an ecosystem where everyone can reliably live
   on the latest semver-compatible release of a package should help make that a
   smoother process.

If, in practice, the resulting ecosystem either contains too many packages that
fail to build, or if a majority of users emit pins files manually regardless of
default, this policy choice can be revisited.

We considered approaches to "pin by default" that used separate mechanisms when
publishing a package to help address the potential for overconstraint, but were
unable to find a solution we felt was workable.


### Naming Choice

This feature is called "locking" and the files are "lockfiles" in many other
package managers, and there has been considerable discussion around whether the
Swift package manager should follow that precedent.

In Swift, we have tried to choose the "right" answer for names in order to make
the resulting language consistent and beautiful.

We have found significant consensus that without considering the prededent, the
"lock" terminology is conceptually the *wrong* word for the operation being
performed here. We view pinning as a workflow-focused feature, versus the
specification in the manifest (which is the "requirement"). The meaning of pin
connotes this transient relationship between the pin action and the underlying
dependency.

In constrast, not only does lock have the wrong connotation, but it also is a
heavily overloaded word which can lead to confusion. For example, if the package
manager used POSIX file locking to prevent concurrent manipulation of packages
(a feature we intend to implement), and we also referred to the pinning files as
"lock files", then any diagnostics using the term "lock file" would be confusing
to a newcomer to the ecosystem familiar with the pinning mechanism but
unfamiliar with the concept of POSIX file locking.

We believe that there are many more potential future users of the Swift package
manager than there are current users familiar with the lock, and chose the "pin"
terminology to reflect what we thought was ultimately the best word for the
operation, in order to contribute to the best long term experience.
