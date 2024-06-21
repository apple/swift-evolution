# Package Manager Revised Dependency Resolution

* Proposal: [SE-0175](0175-package-manager-revised-dependency-resolution.md)
* Author: [Rick Ballard](https://github.com/rballard)
* Review Manager: [Ankit Aggarwal](https://github.com/aciidb0mb3r)
* Status: **Implemented (Swift 4.0)**
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-se-0175-package-manager-revised-dependency-resolution/5896)

## Introduction
This proposal makes the package manager's dependency resolution behavior clearer and more intuitive. It removes the pinning commands (`swift package pin` & `swift package unpin`), replaces the `swift package fetch` command with a new `swift package resolve` command with improved behavior, and replaces the optional `Package.pins` file with a `Package.resolved` file which is always created during dependency resolution.

## Motivation
When [SE-0145 Package Manager Version Pinning](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0145-package-manager-version-pinning.md) was proposed, it was observed that the proposal was overly complex. In particular, it introduced a configuration option allowing some packages to have autopinning on (the default), while others turned it off; this option affected the behavior of other commands (like `swift package update`, which has a `--repin` flag that does nothing for packages that use autopinning). This configuration option has proved to be unnecessarily confusing.

In the existing design, when autopinning is on (which is true by default) the `swift package pin` command can't be used to pin packages at specific revisions while allowing other packages to be updated. In particular, if you edit your package's version requirements in the `Package.swift` manifest, there is no way to resolve your package graph to conform to those new requirements without automatically repinning all packages to the latest allowable versions. Thus, specific, intentional pins can not be preserved without turning off autopinning.

The problems here stem from trying to use one mechanism (pinning) to solve two different use cases: wanting to record and share resolved dependency versions, vs wanting to keep a badly-behaved package at a specific version. We think the package manager could be simplified by splitting these two use cases out into different mechanisms ("resolved versions" vs "pinning"), instead of using an "autopinning" option which makes these two features mutually-exclusive and confusing.

Additionally, some dependency resolution behaviors were not well-specified and do not behave well. The package manager is lax about detecting changes to the versions specified in the `Package.swift` manifest or `Package.pins` pinfile, and fails to automatically update packages when needed, or to issue errors if the version requirements are unsatisfiable, until the user explicitly runs `swift package update`, or until a new user without an existing checkout attempts to build. We'd like to clarify and revise the rules around when and how the package manager performs dependency resolution.

## Proposed solution
The pinning feature will be removed. This removes the `swift package pin` and `swift package unpin` commands, the `--repin` flag to `swift package update`, and use of the `Package.pins` file.

In a future version of the package manager we may re-introduce pinning. If we do, pins will only be recorded in the `Package.pins` file when explicitly set with `swift package pin`, and any pinned dependencies will _not_ be updated by the `swift package update` command; instead, they would need to be unpinned to be updated. This would be a purely additive feature which packages could use in addition to the resolved versions feature when desired.

A new "resolved versions" feature will be added, which behaves very similarly to how pinning previously behaved when autopinning was on. The version of every resolved dependency will be recorded in a `Package.resolved` file in the top-level package, and when this file is present in the top-level package it will be used when performing dependency resolution, rather than the package manager finding the latest eligible version of each package. `swift package update` will update all dependencies to the latest eligible versions and update the `Package.resolved` file accordingly.

Resolved versions will always be recorded by the package manager. Some users may chose to add the `Package.resolved` file to their package's `.gitignore` file. When this file is checked in, it allows a team to coordinate on what versions of the dependencies they should use. If this file is gitignored, each user will separately choose when to get new versions based on when they run the `swift package update` command, and new users will start with the latest eligible version of each dependency. Either way, for a package which is a dependency of other packages (e.g. a library package), that package's `Package.resolved` file will not have any effect on its client packages.

The existing `swift package fetch` command will be deprecated, removed from the help message, and removed completely in a future release of the Package Manager. In its place, a new `swift package resolve` command will be added. The behavior of `resolve` will be to resolve dependencies, taking into account the current version restrictions in the `Package.swift` manifest and `Package.resolved` resolved versions file, and issuing an error if the graph cannot be resolved. For packages which have previously resolved versions recorded in the `Package.resolved` file, the `resolve` command will resolve to those versions as long as they are still eligible. If the resolved versions file changes (e.g. because a teammate pushed a new version of the file) the next `resolve` command will update packages to match that file. After a successful `resolve` command, the checked out versions of all dependencies and the versions recorded in the resolved versions file will match. In most cases the `resolve` command will perform no changes unless the `Package.swift` manifest or `Package.resolved` file have changed.

The following commands will implicitly invoke the `swift package resolve` functionality before running, and will cancel with an error if dependencies cannot be resolved:

* `swift build`
* `swift test`
* `swift package generate-xcodeproj`

The `swift package show-dependencies` command will also implicitly invoke `swift package resolve`, but it will show whatever information about the dependency graph is available even if the resolve fails.

The `swift package edit` command will implicitly invoke `swift package resolve`, but if the resolve fails yet did identify and fetch a package with the package name the command supplied, the command will allow that package to be edited anyway. This is useful if you wish to use the `edit` command to edit version requirements and fix an unresolvable dependency graph. `swift package unedit` will unedit the package and _then_ perform a `resolve`.

## Detailed design
The `resolve` command is allowed to automatically add new dependencies to the resolved versions file, and to remove dependencies which are no longer in the dependency graph. It can also automatically update the recorded versions of any package whose previously-resolved version is no longer allowed by the version requirements from the `Package.swift` manifests. When changed version requirements force a dependency to be automatically re-resolved, the latest eligible version will be chosen; any other dependencies affected by that change will prefer to remain at their previously-resolved versions as long as those versions are eligible, and will otherwise update likewise.

The `Package.resolved` resolved versions file will record the git revision used for each resolved dependency in addition to its version. In future versions of the package manager we may use this information to detect when a previously-resolved version of a package resolves to a new revision, and warn the user if this happens.

The `swift package resolve` command will not actually perform a `git fetch` on any dependencies unless it needs to in order to correctly resolve dependencies. As such, if all dependencies are already resolved correctly and allowed by the version constraints in the `Package.swift` manifest and `Package.resolved` resolved versions file, the `resolve` command will not need to do anything (e.g. a normal `swift build` won't hit the network or make unnecessary changes during its implicit `resolve`).

If a dependency is in edit mode, it is allowed to have a different version checked out than that recorded in the resolved versions file. The version recorded for an edited package will not change automatically. If a `swift package update` operation is performed while any packages are in edit mode, the versions of those edited packages will be removed from the resolved versions file, so that when those packages leave edit mode the next resolution will record a new version for them. Any packages in the dependency tree underneath an edited package will also have their resolved version removed by `swift package update`, as otherwise the resolved versions file might record versions that wouldn't have been chosen without whatever edited package modifications have been made.

## Alternatives considered

We considered repurposing the existing `fetch` command for this new behavior, instead of renaming the command to `resolve`. However, the name `fetch` is defined by `git` to mean getting the latest content for a repository over the network. Since this package manager command does not always actually fetch new content from the network, it is confusing to use the name `fetch`. In the future, we may offer additional control over when dependency resolution is allowed to perform network access, and we will likely use the word `fetch` in flag names that control that behavior.

We considered continuing to write out the `Package.pins` file for packages whose [Swift tools version](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0152-package-manager-tools-version.md) was less than 4.0, for maximal compatibility with the Swift 3.1 tools. However, as the old pinning behavior was a workflow feature and not a fundamental piece of package compatibility, we do not consider it necessary to support in the 4.0 tools.

We considered keeping the `pin` and `unpin` commands, with the new behavior as discussed briefly in this proposal. While we think we may wish to bring this feature back in the future, we do not consider it critical for this release; the workflow it supports (updating all packages except a handful which have been pinned) is not something most users will need, and there are workarounds (e.g. specify an explicit dependency in the `Package.swift` manifest).

We considered using an `install` verb instead of `resolve`, as many other package managers use `install` for a very similar purpose. However, almost all of those package managers are for non-compiled languages, where downloading the source to a dependency is functionally equivalent to "installing" it as a product ready for use. In contrast, Swift is a compiled language, and our dependencies must be built (e.g. into libraries) before they can be installed. As such, `install` would be a misnomer for this workflow. In the future we may wish to add an `install` verb which actually does install built products, similar to `make install`.

### Why we didn't use "Package.lock"

We considered using the `.lock` file extension for the new resolved versions file, to be consistent with many other package managers. We expect that the decision not to use this extension will be controversial, as following established precedent is valuable. However, we think that a "lockfile" is a very poor name for this concept, and that using that name would cause confusion when we re-introduce pins. Specifically:

- Calling this a "lock" implies a stronger lockdown of dependencies than is supported by the actual behavior. As a simple `update` command will reset the locks, and a change to the specified versions in `Package.swift` will override them, they're not really "locked" at all. This is misleading.
- When we re-introduce pinning, it would be very confusing to have both "locks" and "pins". Having "resolved versions" and "pins" is not so confusing.
- The term "lock" is already overloaded between POSIX file locks and locks in concurrent programming.


For comparison, here is a list of other package managers which implement similar behavior and their name for this file:

| Package Manager | Language | Resolved versions file name |
| --- | --- | --- |
| Yarn | JS | yarn.lock |
| Composer | PHP | composer.lock |
| Cargo | Rust | Cargo.lock |
| Bundler | Ruby | Gemfile.lock |
| CocoaPods | ObjC/Swift | Podfile.lock |
| Glide | Go | glide.lock |
| Pub | Dart | pubspec.lock |
| Mix | Elixir | mix.lock |
| rebar3 | Erlang | rebar.lock |
| Carton | Perl | carton.lock |
| Carthage | ObjC/Swift | Cartfile.resolved |
| Pip | Python | requirements.txt |
| NPM | JS | npm-shrinkwrap.json |
| Meteor | JS | versions |

Some arguments for using ".lock" instead of ".resolved" are:

- Users of other package managers will already be familiar with the terminology and behavior.
- For packages which support multiple package managers, it will be possible to put "\*.lock" into the gitignore file instead of needing a separate entry for "\*.resolved".

However, we do not feel that these arguments outweigh the problems with the term "lock". If providing feedback asking that we reconsider this decision, please be clear about why the above decision is incorrect, with new information not already considered.
