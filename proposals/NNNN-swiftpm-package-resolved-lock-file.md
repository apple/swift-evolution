# Treating `Package.resolved` as a lock file

* Proposal: [SE-NNNN](https://github.com/apple/swift-evolution/pull/2212/files?short_path=14ad3ef#diff-14ad3ef424e6ca9cf3077fd1a3c0e898a34a2b14523c1358edbbf6fa1a5def16)
* Authors: [Franz Busch](https://github.com/FranzBusch)
* Status: **Pitch**
* Implementation: Awaiting implementation

## Introduction

The Swift package manager is an essential part of the ecosystem and provides
functionality to download your dependencies, version them and build your
packages. Furthermore, the Swift package manager provides a locking mechanism
for dependencies to avoid accidental updates. Locking of dependencies has become
more relevant in the past years due to many different supply chain attacks in
multiple language ecosystems. 

## Motivation

Since, the early days of the Swift package manager it had a couple of essential
commands to manage the dependencies of a package and building the package. These
commands are `swift package resolve`, `swift package update` and `swift build`. Over
the years, those commands gained more and more options to customize their
behaviour; however, there doesn't exist a clear definition when each of those
commands resolves or updates dependencies and how this interacts with the
presence or absence of a `Package.resolved` file. Furthermore, a lot of the
commands don't treat the `Package.resolved` file as a lock file and often update
its contents.

This proposal aims to define a clear behavior for the Swift package manager when
it resolves and updates dependencies.

## Proposed solution

### Deprecated: swift package resolve

We propose to deprecate this command since it is confusing in behavior. It
currently does both a resolution and download. Additionally, other commands may
trigger a resolution as well. We propose to replace it with swift package fetch
below. 

### swift package fetch

Fetches the dependencies of a package.

#### Description

If a `Package.resolved` file is present, this command will fetch all dependencies
and make them locally available. Making them locally available means cloning the
git repositories or downloading the artefact. Subsequent commands will be able
to run offline unless the `Package.resolved` file changes. If the versions in the
`Package.resolved` cannot be fetched this command will fail with an error. If the
contents of the `Package.resolved` are not up-to-date with the contents of the
`Package.swift` this command will prompt the user to ask to update if run from an
interactive terminal otherwise it will fail with an error.

> Note: Determining if the `Package.resolved` is out of data is done by storing a
hash of the `Package.swift` in the `Package.resolved` and comparing it when
executing the fetch command. Implemented in swift-package-manager/pull/6698

If there is no `Package.resolved` file then this command will generate a new file
by resolving the dependency graph. Afterwards, it will fetch all dependencies.
Resolving the dependency graph might require to fetch the dependencies in the
case of git dependencies. Importantly, this will not resolve to the latest
versions since it might use the local or system wide cache.


#### Options

##### **--verbose/—very-verbose**
Produces verbose/very verbose output.

##### **--quiet**
Produces quiet output that does only produce warning or higher level
output.

##### **--package-path**
The path to the `Package.swift`. By default, SwiftPM searches for
the `Package.swift` file in the current directory.

##### **--require-resolved-file**
Requires the `Package.resolved` file to be present. If the
lock resolved file is missing, SwiftPM will exit with an error. This is useful
for CI systems to make sure only packages with `Package.resolved` files are being
built.

##### **--offline**
Prevents SwiftPM to make any network requests. This is useful when
the dependencies have already been fetched into a system level cache and
validating that fetch is not making any network requests to grab dependencies
that are not available in the cache.

##### **--automatically-resolve-when-outdated**
Tells SwiftPM to resolve the dependency
graph again when the `Package.resolved` file is out-of-date with the `Package.swift`
file. This will skip the prompt that normally happens. Important: This should be
avoided when building applications on CI systems since it might result in
different dependencies being included than the ones specified in the
`Package.resolved`.

##### **--ensure-mirrored**
Requires that every dependency has a configured mirror.

### swift package update

Updates the dependencies of a package.

#### Description

If a `Package.resolved` is present, this command will try to update it to the
latest package versions.

If there is no `Package.resolved` file, then this command will generate a new one
with the latest versions.

Updating the dependencies includes updating all of the git based dependencies
and pulling the latest state from the repositories. Afterwards, SwiftPM will
resolve the dependency graph again and tries to update to the highest version
possible.

#### Options

##### **--verbose**
Produces verbose output and may be specified twice for very verbose
output.

##### **--quiet**
Produces quiet output that does only produce warning or higher level
output.

##### **--package-path**
The path to the `Package.swift`. By default, SwiftPM searches for
the `Package.swift` file in the current directory.

##### **--offline**
Prevents SwiftPM to make any network requests. 

##### **--ensure-mirrored**
Requires that every dependency has a configured mirror.

##### **--package**
Only this package will be updated. Transitives will only be updated if
they must be updated. This flag can be specified multiple times.

##### **--dry-run**
Displays to which version the dependencies would be updated but
doesn’t generate a new `Package.resolved`.

### swift build/test/run changes
When a `Package.resolved` file is present then the build, test and run commands
should check whether the `Package.resolved` file is out-of-date with the
`Package.swift` . If the contents of the `Package.resolved` are not up-to-date with
the contents of the `Package.swift` this command will prompt the user to ask to
update if run from an interactive terminal otherwise it will fail with an error.
This follows how the proposed swift package fetch is working; hence, we also
propose to add a --automatically-resolve-when-outdated to build/test/run.

If no `Package.resolved` is present then these commands should first trigger swift
package fetch before running the actual command.

The following options will be removed from the respective commands:

* `--force-resolved-versions`, `--disable-automatic-resolution`,
  `--only-use-versions-from-resolved-file` : This is the new default
* `--skip-update` : This currently only skips the remote update similar to the
  newly proposed `--offline` mode

## Compatibility

Changing the behaviour of CLI commands is in general a breaking change for any
tool that uses those commands; however, the new behaviour are gated by upgrading
the Swift version. Tooling that was using the existing commands would need to be
updated to the new semantics. Overall, we believe this is a one-time trade worth
doing to increase the security.

## Migration

Scripts and workflows have to migrated when adopting the new Swift version that
includes the updated commands. The biggest change is the removal of swift
package resolve . Any script that was currently using swift package resolve
needs to evaluate why it was using it and either migrate to `swift package fetch`
or `swift package update`.

The `build`/`run`/`test` commands are staying mostly the same and we expect very few
people have to adapt their scripts. The biggest difference is the check that the
`Package.resolved` file is up-to-date which might require users to add a
`--automatically-resolve-when-outdated` flag.

## Future directions

### Target/Host specific dependency resolution

In the future, we can extend the the `fetch` command to be able to fetch
dependencies for specific target/host triples. This however, is something that
requires changes to the `Package.swift` manifest and is outside the scope of this
proposal. 

### Vendor command

Vendoring dependencies locally is a common request and is closely related to
dependency resolution. However, this should be tackled in a separate proposal
since it also has to take into account how vendors dependencies are built.

### Minimum version resolution

Packages are declaring their dependencies using minimum version in SemVer format;
however, during resolution the latest possible version that still satisfies all
constraints is chosen. In practice, this leads to most developers using the
latest version of packages. Hence, almost nobody is building packages with
minimum versions of their dependencies. This often leads to the fact that the
minimum versions that library packages declare are not working. In the future,
we could provide a command to force resolution to pick the lower boundary. This
would allow library authors to test if their packages work with their lowest
declared dependency versions. 

### `--precise` and `--aggressive` options for swift package update

In the future, we can add two more options to `swift package update` to fine tune
to which version a package is updated and how transitive dependencies are
updated. However, adding those options can be done additively once we changed
the semantics.

`--aggressive` Used together with `--package` and forces all transitive dependencies
to be updated as well. This applies to all packages specified via `--package`
flags.

`--precise` Used together with `--package` and allows to specify a specific version
to update the package to. This applies to all packages specified via `--package`
flags.

## Alternatives considered

### Conditionally enable new behaviour on swift-tools-version

We could make the new behaviour only apply when a certain swift-tools-version
has been set; however, this would bring enormous complexity to SwiftPM since we
allow to pass a package-path to the various commands and we would need to do a
two step argument parsing depending on what tools version is in the
`Package.swift`.
