# Package Registry Service

* Proposal: [SE-0292](0292-package-registry-service.md)
* Authors: [Bryan Clark](https://github.com/clarkbw),
           [Whitney Imura](https://github.com/whitneyimura),
           [Mattt Zmuda](https://github.com/mattt)
* Review Manager: [Tom Doron](https://github.com/tomerd)
* Status: **Implemented (Swift 5.7)**
* Implementation: [apple/swift-package-manager#3023](https://github.com/apple/swift-package-manager/pull/3023)
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-with-modifications-se-0292-package-registry-service/49849)
* Review:
  [1](https://forums.swift.org/t/se-0292-package-registry-service/)
  [2](https://forums.swift.org/t/se-0292-2nd-review-package-registry-service/)
  [3](https://forums.swift.org/t/se-0292-3rd-review-package-registry-service/)
  [Amendment](https://forums.swift.org/t/amendment-se-0292-package-registry-service/)
* Previous Revision:
  [1](https://github.com/swiftlang/swift-evolution/blob/b48527526b5748a60b0b23846d5880e9cc2c4711/proposals/0292-package-registry-service.md)
  [2](https://github.com/swiftlang/swift-evolution/blob/53bd6d3813c40ebd07701727c8cfb6fedd751e2a/proposals/0292-package-registry-service.md)
  [3](https://github.com/swiftlang/swift-evolution/blob/971d1f43bce718a45227432782a312cc5de99870/proposals/0292-package-registry-service.md)

## Introduction

Swift Package Manager downloads dependencies using Git.
Our proposal defines a standard web service interface
that it can also use to download dependencies from a package registry.

Swift-evolution thread:
[Swift Package Registry Service](https://forums.swift.org/t/swift-package-registry-service/37219)

## Motivation

A package dependency is currently specified by a URL to its source repository.
When Swift Package Manager builds a project for the first time,
it clones the Git repository for each dependency
and attempts to resolve the version requirements from the available tags.

Although Git is a capable version-control system,
it's not well-suited to this kind of workflow for the following reasons:

* **Reproducibility**:
  A version tag in the Git repository for a dependency
  can be reassigned to another commit at any time.
  This can cause a project to produce different build results
  depending on when it was built.
* **Availability**:
  The Git repository for a dependency can be moved or deleted,
  which can cause subsequent builds to fail.
* **Efficiency**:
  Cloning the Git repository for a dependency
  downloads all versions of a package when only one is used at a time.
* **Speed**:
  Cloning a Git repository for a dependency can be slow
  if it has a large history.
  Also, cloning a Git repository is expensive for both the server and client,
  and may be significantly slower than downloading the same content
  using HTTP through a [content delivery network (CDN)][CDN].

Many language ecosystems have a *package registry*, including
[RubyGems] for Ruby,
[PyPI] for Python,
[npm] for JavaScript, and
[crates.io] for Rust.
In fact,
many Swift developers build apps today using
[CocoaPods] and its index of libraries.

A package registry for Swift Package Manager
could offer faster and more reliable dependency resolution
than downloading dependencies using Git.
It could also support other useful functionality,
including package search, security audits, and local offline caches.

## Proposed solution

This proposal defines a standard interface for package registry services
and describes how Swift Package Manager integrates with them
to download dependencies.

A user may [configure](#registry-configuration-subcommands)
a package registry for their project
by specifying a URL to a [conforming web service](#package-registry-service-1).
When a registry is configured,
Swift Package Manager resolves external dependencies
in the project's package manifest (`Package.swift`) file
that are [declared](#new-packagedescription-api)
with a [scoped package identifier](#package-identity) in the form
`scope.package-name`.
These package identifiers resolve potential
[name collisions](#package-name-collision-resolution)
across build targets.

For each external dependency declared in the package manifest,
Swift Package Manager first sends a
`GET` request to `/{scope}/{name}`
to fetch a list of available releases
from the configured registry.
If a release is found that satisfies the declared version requirement
(for example, `.upToNextMinor(from: "1.1.0")`),
Swift Package Manager sends a
`GET` request to `/{scope}/{name}/{version}/Package.swift`
to fetch the manifest for that release.
This process continues with the package manifests of each dependency,
each of their respective dependencies,
and so on.
Once the dependency graph is [resolved](#dependency-graph-resolution),
Swift Package Manager downloads the
[source archive](#archive-source-subcommand) for each dependency
by sending a `GET` request to `/{scope}/{name}/{version}.zip`.

## Detailed design

### Package registry service

A package registry service implements the following REST API endpoints
for listing releases for a package,
fetching information about a release,
and downloading the source archive for a release:

| Method | Path                                                      | Description                                     |
| ------ | --------------------------------------------------------- | ----------------------------------------------- |
| `GET`  | `/{scope}/{name}`                                         | List package releases                           |
| `GET`  | `/{scope}/{name}/{version}`                               | Fetch metadata for a package release            |
| `GET`  | `/{scope}/{name}/{version}/Package.swift{?swift-version}` | Fetch manifest for a package release            |
| `GET`  | `/{scope}/{name}/{version}.zip`                           | Download source archive for a package release   |
| `GET`  | `/identifiers{?url}`                                      | Lookup package identifiers registered for a URL |

A formal specification for the package registry interface is provided
[alongside this proposal](https://github.com/apple/swift-package-manager/blob/main/Documentation/PackageRegistry/Registry.md).
In addition,
an OpenAPI (v3) document
and a reference implementation written in Swift
are provided for the convenience of developers interested
in building their own package registry.

### Changes to Swift Package Manager

#### Package identity

Currently, the identity of a package is computed from
the last path component of its effective URL
(which can be changed with dependency mirroring).
However, this approach can lead to a conflation of
distinct packages with similar names
and the duplication of the same package under different names.

We propose using a scoped identifier
in the form `scope.package-name`
to identify package dependencies.

A *scope* provides a namespace for related packages within a package registry.
A package scope consists of
alphanumeric characters and hyphens.
Hyphens may not occur at the beginning or end,
nor consecutively within a scope.
The maximum length of a package scope is 39 characters.
A valid package scope matches the following regular expression pattern:

```regexp
\A[a-zA-Z\d](?:[a-zA-Z\d]|-(?=[a-zA-Z\d])){0,38}\z
```

A package's *name* uniquely identifies a package in a scope.
A package name consists of alphanumeric characters, underscores, and hyphens.
Hyphens and underscores may not occur at the beginning or end,
nor consecutively within a name.
The maximum length of a package name is 100 characters.
A valid package scope matches the following regular expression pattern:

```regexp
\A[a-zA-Z0-9](?:[a-zA-Z0-9]|[-_](?=[a-zA-Z0-9])){0,99}\z
```

Package scopes and names are compared using locale-independent case folding.

#### New `PackageDescription` API

The `Package.Dependency` type adds the following static method:

```swift
extension Package.Dependency {
    /// Adds a dependency on a package with the specified identifier
    /// that uses the provided version requirement.
    public static func package(
        id: String,
        _ requirement: Package.Dependency.VersionBasedRequirement
    ) -> Package.Dependency
}
```

These methods may be called in the `dependencies` field of a package manifest
to declare one or more dependencies by their respective package identifier.

```swift
dependencies: [
   .package(id: "mona.LinkedList", .upToNextMinor(from: "1.1.0")),
   .package(id: "mona.RegEx", .exact("2.0.0"))
]
```

A package dependency declared with an identifier using this method
may only specify a version-based requirement.
`Package.Dependency.VersionBasedRequirement` is a new type
that provides the same interface as `Package.Dependency.Requirement`
for version-based requirements,
but excluding branch-based and commit-based requirements.

#### Package name collision resolution

Consider a dependency graph that includes both
a package declared with the identifier `mona.LinkedList` and
an equivalent package declared with the URL `https://github.com/mona/LinkedList`.

When Swift Package Manager fetches a list of releases for the identified package
(`GET /mona/LinkedList`),
the response includes a `Link` header field
with URLs to that project's source repository
that are known to the registry.

```http
Link: <https://github.com/mona/LinkedList>; rel="canonical",
      <ssh://git@github.com:mona/LinkedList.git>; rel="alternate"
```

Swift Package Manager uses this information
to reconcile the URL-based dependency declaration with
the package identifier `mona.LinkedList`.
Link relation URLs may also be normalized to mitigate insignificant variations.
For example,
a package with an ["scp-style" URL][scp-url] like
`git@github.com:mona/LinkedList.git`
is determined to be equivalent to a URL with an explicit scheme like
`ssh:///git@github.com/mona/LinkedList`.
Swift Package Manager may additionally consult the registry
to associate a URL-based package declaration with a package identifier
by sending a `GET /identifiers{?url}` request with that package's URL.

A package identifier serves as the package name
in target-based dependency declarations —
that is, the `package` parameter in `.product(name:package)` method calls.

```diff
    targets: [
        .target(name: "MyLibrary",
                dependencies: [
                  .product(name: "LinkedList",
-                          package: "LinkedList")
+                          package: "mona.LinkedList")
                ]
    ]
```

Any path-based dependency declaration
or URL-based declaration without an associated package identifier
will continue to synthesize its identity from
the last path component of its location.

#### Dependency graph resolution

In its `PackageGraph` module, Swift Package Manager defines
the `PackageContainer` protocol as the top-level unit of package resolution.
Conforming types are responsible for
determining the available tags for a package
and its contents at a particular revision.
A `PackageContainerProvider` protocol adds a level of indirection
for resolving package containers.

There are currently two concrete implementations of `PackageContainer`:
`LocalPackageContainer` and `RepositoryPackageContainer`.
This proposal adds a new `RegistryPackageContainer` type
that adopts `PackageContainer`
and performs equivalent operations with HTTP requests to a registry service.
These client-server interactions are facilitated by
a new `RegistryManager` type.
When requesting resources from a registry,
Swift Package Manager will employ techniques like
exponential backoff, circuit breakers, and client-side validation
to safeguard against adverse network conditions and malicious server responses.

The following table lists the
tasks performed by Swift Package Manager during dependency resolution
alongside the Git operations used
and their corresponding package registry API calls.

| Task                                  | Git operation               | Registry request                              |
| ------------------------------------- | --------------------------- | --------------------------------------------- |
| Fetch the contents of a package       | `git clone && git checkout` | `GET /{scope}/{name}/{version}.zip`           |
| List the available tags for a package | `git tag`                   | `GET /{scope}/{name}`                         |
| Fetch a package manifest              | `git clone`                 | `GET /{scope}/{name}/{version}/Package.swift` |

Package registries support
[version-specific _manifest_ selection][version-specific-manifest-selection]
by providing a list of versioned manifest files for a package
(for example, `Package@swift-5.3.swift`)
in its response to `GET /{scope}/{name}/{version}/Package.swift`.
However, package registries don't support
[version-specific _tag_ selection][version-specific-tag-selection].

### Changes to `Package.resolved`

Swift package registry releases are archived as Zip files.

When an external package dependency is downloaded through a registry,
Swift Package Manager compares the integrity checksum provided by the server
against any existing checksum for that release in the `Package.resolved` file
as well as the integrity checksum reported by the `compute-checksum` subcommand:

```console
$ swift package compute-checksum LinkedList-1.2.0.zip
1feec3d8d144814e99e694cd1d785928878d8d6892c4e59d12569e179252c535
```

If no prior checksum exists,
it's saved to `Package.resolved`.

```json
{
    "object": {
        "pins": [
            {
                "package": "mona.LinkedList",
                "state": {
                    "checksum": "ed008d5af44c1d0ea0e3668033cae9b695235f18b1a99240b7cf0f3d9559a30d",
                    "version": "1.2.0"
                }
            }
        ]
    },
    "version": 1
}
```

Suppose the checksum reported by the server
is different from the existing checksum
(or the checksum of the downloaded artifact is different from either of them).
In that case,
a package's contents may have changed at some point.
Swift Package Manager will refuse to download dependencies
if there's a mismatch in integrity checksums.

```console
$ swift build
error: checksum of downloaded source archive of dependency 'mona.LinkedList' (c2b934fe66e55747d912f1cfd03150883c4f037370c40ca2ad4203805db79457) does not match checksum specified by the manifest (ed008d5af44c1d0ea0e3668033cae9b695235f18b1a99240b7cf0f3d9559a30d)
```

Once the correct checksum is determined,
the user can update `Package.resolved` with the correct value
and try again.

### Archive-source subcommand

An anecdotal look at other package managers suggests that
a checksum mismatch is more likely to be a
disagreement in how to create the archive and/or calculate the checksum
than, say, a forged or corrupted package.

This proposal adds a new `swift package archive-source` subcommand
to provide a standard way to create source archives for package releases.

```manpage
SYNOPSIS
	swift package archive-source [--output=<file>]

OPTIONS
	-o <file>, --output=<file>
		Write the archive to <file>.
		If unspecified, the package is written to `\(PackageName).zip`.
```

Run the `swift package archive-source` subcommand
in the root directory of a package
to generate a source archive for the current working tree.
For example:

```console
$ tree -a -L 1
LinkedList
├── .git
├── Package.swift
├── README.md
├── Sources
└── Tests

$ head -n 5 Package.swift
// swift-tools-version:5.3
import PackageDescription

let package = Package(
name: "LinkedList",

$ swift package archive-source
Created LinkedList.zip
```

By default,
generated archive's filename is
the name of the package with a `.zip` extension
(for example, "LinkedList.zip").
You can override this behavior with the `--output` option:

```console
$ git checkout 1.2.0
$ swift package archive-source --output="LinkedList-1.2.0.zip"
# Created LinkedList-1.2.0.zip
```

The `archive-source` subcommand has the equivalent behavior of
[`git-archive(1)`] using the `zip` format at its default compression level,
with entries prefixed by the basename of the generated archive's filename.
Therefore, the following command produces
equivalent output to the previous example:

```console
$ git archive --format zip \
              --prefix LinkedList-1.2.0
              --output LinkedList-1.2.0.zip \
              1.2.0
```

If desired, this behavior could be changed in future tool versions.

> **Note**:
> `git-archive` ignores files with the `export-ignore` Git attribute.
> By default, this ignores hidden files and directories,
> including`.git` and `.build`.

### Registry configuration subcommands

This proposal adds a new `swift package-registry` subcommand
for managing the registry used for all packages
and/or packages in a particular scope.

Custom registries can serve a variety of purposes:

- **Private dependencies**:
  Users may configure a custom registry for a particular scope
  to incorporate private packages with those fetched from a public registry.
- **Geographic colocation**:
  Developers working under adverse networking conditions can
  host a mirror of official package sources on a nearby network.
- **Policy enforcement**:
  A corporate network can enforce quality or licensing standards,
  so that only approved packages are available through a custom registry.
- **Auditing**:
  A custom registry may analyze or meter access to packages
  for the purposes of ranking popularity or charging licensing fees.

#### Setting a custom registry

```manpage
SYNOPSIS
	swift package-registry set <url> [options]
OPTIONS:
  --global    Apply settings to all projects for this user
  --scope     Associate the registry with a given scope
  --login     Specify a user name for the remote machine
  --password  Supply a password for the remote machine
```

Running the `package-registry set` subcommand
in the root directory of a package
creates or updates the `.swiftpm/configuration/registries.json` file
with a new top-level `registries` key
that's associated with an object containing the specified registry URLs.
The default, unscoped registry is associated with the key `[default]`.
Any scoped registries are keyed by their case-folded name.

For example,
a build server that doesn't allow external network connections
may configure a registry URL to resolve dependencies
using an internal registry service.

```console
$ swift package-registry set https://internal.example.com/
$ cat .swiftpm/configuration/registries.json
```

```json
{
  "registries": {
    "[default]": {
      "url": "https://internal.example.com"
    }
  },
  "version": 1
}

```

If no registry is configured,
Swift Package Manager commands like
`swift package resolve` and `swift package update`
fail with an error.

```console
$ swift package resolve
error: cannot resolve dependency 'mona.LinkedList' without a configured registry
```

#### Associating a registry with a scope

The user can associate a package scope with a custom registry
by passing the `--scope` option.

For example,
a user might resolve all packages with the package scope `example`
(such as `example.PriorityQueue`)
to a private registry.

```console
$ swift package-registry set https://internal.example.com/ --scope example
$ cat .swiftpm/configuration/registries.json
```

```json
{
  "registries": {
    "example": {
      "url": "https://internal.example.com"
    }
  },
  "version": 1
}

```

When a custom registry is associated with a package scope,
package dependencies with that scope are resolved through the provided URL.
A custom registry may be associated with one or more scopes,
but a scope may be associated with only a single registry at a time.
Scoped custom registries override any unscoped custom registry.

#### Unsetting a custom registry

This proposal also adds a new `swift package-registry unset` subcommand
to complement the `package-registry set` subcommand.

```manpage
SYNOPSIS
	swift package-registry unset [options]
OPTIONS:
  --global    Apply settings to all projects for this user
  --scope     Removes the registry's association to a given scope
```

Running the `package-registry unset` subcommand
in the root directory of a package
updates the `.swiftpm/configuration/registries.json` file
to remove the `default` entry in the top-level `registries` key, if present.
If a `--scope` option is passed,
only the entry for the specified scope is removed, if present.

#### Global registry configuration

The user can pass the `--global` option to the `set` or `unset` subcommands
to update the user-level configuration file located at
`~/.swiftpm/configuration/registries.json`.

Any default or scoped registries configured locally in a project directory
override any values configured globally for the user.
For example,
consider the following global and local registry configuration files:

```jsonc
// Global configuration (~/.swiftpm/configuration/registries.json)
{
  "registries": {
    "[default]": {
      "url": "https://global.example.com"
    },
    "foo": {
      "url": "https://global.example.com"
    },
  },
  "version": 1
}

// Local configuration (.swiftpm/configuration/registries.json)
{
  "registries": {
    "foo": {
      "url": "https://local.example.com"
    }
  },
  "version": 1
}

```

Running the `swift package resolve` command with these configuration files
resolves packages with the `foo` scope
using the registry located at "https://local.example.com",
and all other packages
using the registry located at "https://global.example.com".

In summary,
the behavior of `swift package resolve` and related commands
depends on the following factors,
in descending order of precedence:

* The package manifest in the current directory (`./Package.swift`)
* Any existing lock file (`./Package.resolved`)
* Any local configuration (`./.swiftpm/configuration/registries.json`)
* Any global configuration file (`~/.swiftpm/configuration/registries.json`)

#### Specifying credentials for a custom registry

Some servers may require a username and password.
The user can provide credentials when setting a custom registry
by passing the `--login` and `--password` options.

When credentials are provided,
the corresponding object in the `registries.json` file
includes a `login` key with the passed value.
If the project's `.netrc` file has an existing entry
for a given machine and login,
it's updated with the new password;
otherwise, a new entry is added.
If no `.netrc` file exists,
a new one is created and populated with the new entry.

```console
$ swift package-registry set https://internal.example.com/ \
    --login jappleseed --password alpine

$ cat .netrc
machine internal.example.com
login jappleseed
password alpine

$ cat .swiftpm/configuration/registries.json

{
  "registries": {
    "[default]": {
      "url": "https://internal.example.com"
      "login": "jappleseed"
    }
  },
  "version": 1
}
```

If the user passes the `--login` and `--password` options
to the `set` subcommand along with the `--global` option,
the user-level `.netrc` file is updated instead.
When Swift Package Manager connects to a custom registry,
it first consults the project's `.netrc` file, if one exists.
If no entry is found for the custom registry,
Swift Package Manager then consults the user-level `.netrc` file, if one exists.

If the provided credentials are missing or invalid,
Swift Package Manager commands like
`swift package resolve` and `swift package update`
fail with an error.

### Changes to config subcommand

#### Set-mirror option for package identifiers

A user can currently specify an alternate location for a package
by setting a [dependency mirror][SE-0219] for that package's URL.

```console
$ swift package config set-mirror \
    --original-url https:///github.com/mona/linkedlist \
    --mirror-url https:///github.com/octocorp/swiftlinkedlist
```

This proposal updates the `swift package config set-mirror` subcommand
to accept a `--package-identifier` option in place of an `--original-url`.
Running this subcommand with a `--package-identifier` option
creates or updates the `.swiftpm/configuration/mirrors.json` file,
modifying the array associated with the top-level `object` key
to add a new entry or update an existing entry
for the specified package identifier,
that assigns its alternate location.

```json
{
  "object": [
    {
      "mirror": "https://github.com/OctoCorp/SwiftLinkedList.git",
      "original": "mona.LinkedList"
    }
  ],
  "version": 1
}

```

When a mirror URL is set for a package identifier,
Swift Package Manager resolves any dependencies with that identifier
through Git using the provided URL.

## Security

Adding external dependencies to a project
increases the attack surface area of your software.
However, much of the associated risk can be mitigated,
and a package registry can offer stronger guarantees for safety and security
compared to downloading dependencies using Git.

Core security measures,
such as the use of HTTPS and integrity checksums,
are required by the registry service specification.
Additional decisions about security
are delegated to the registries themselves.
For example,
registries are encouraged to adopt a
scoped, revocable authorization framework like [OAuth 2.0][RFC 6749],
but this isn't a strict requirement.
Package maintainers and consumers should
consider a registry's security posture alongside its other features
when deciding where to host and fetch packages.

Our proposal's package identity scheme is designed to prevent or mitigate
vulnerabilities common to packaging systems and networked applications:

- Package scopes are restricted to a limited set of characters,
  preventing [homograph attacks].
  For example,
  "А" (U+0410 CYRILLIC CAPITAL LETTER A) is an invalid scope character
  and cannot be confused for "A" (U+0041 LATIN CAPITAL LETTER A).
- Package scopes disallow leading, trailing, or consecutive hyphens (`-`),
  and disallows underscores (`_`) entirely,
  which mitigates look-alike package scopes
  (for example, "llvm--swift" and "llvm_swift" are both invalid
  and cannot be confused for "llvm-swift").
- Package scopes disallow dots (`.`),
  which prevents potential confusion with domain variants of scopes
  (for example, "apple.com" is invalid
  and cannot be confused for "apple").
- Packages are registered within a scope,
  which mitigates [typosquatting].
  Package registries may further restrict the assignment of new scopes
  that are intentionally misleading
  (for example, "G00gle", which looks like "Google").
- Package names disallow punctuation and whitespace characters used in
  [cross-site scripting][xss] and
  [CRLF injection][http header injection] attacks.

To better understand the security implications of this proposal —
and Swift dependency management more broadly —
we employ the
<abbr title="Spoofing, Tampering, Repudiation, Information disclosure, Denial of Service, Escalation of privilege">
[STRIDE]
</abbr> mnemonic below:

### Spoofing

An attacker could interpose a proxy between the client and the package registry
to intercept credentials for that host
and use them to impersonate the user in subsequent requests.

The impact of this attack is potentially high,
depending on the scope and level of privilege associated with these credentials.
However, the use of secure connections over HTTPS
goes a long way to mitigate the overall risk.

Swift Package Manager could further mitigate this risk
by taking the following measures:

* Enforcing HTTPS for all URLs
* Resolving URLs using DNS over HTTPS (DoH)
* Requiring URLs with Internationalized Domain Names (IDNs)
  to be represented as Punycode

### Tampering

An attacker could interpose a proxy between the client and the package registry
to construct and send Zip files containing malicious code.

Although the impact of such an attack is potentially high,
the risk is largely mitigated by the use of cryptographic checksums
to verify the integrity of downloaded source archives.

```console
$ echo "$(swift package compute-checksum LinkedList-1.2.0.zip) *LinkedList-1.2.0.zip" | \
    shasum -a 256 -c -
LinkedList-1.2.0.zip: OK
```

Integrity checks alone can't guarantee
that a package isn't a forgery;
an attacker could compromise the website of the host
and provide a valid checksum for a malicious package.

`Package.resolved` provides a [Trust on first use (TOFU)][TOFU] security model
that can offer strong guarantees about the integrity of dependencies over time.
A registry can further improve on this model by implementing a
[transparent log],
[checksum database],
or another comparable, tamper-proof system
for authenticating package contents.

Distribution of packages through Zip files
introduces new potential attack vectors.
For example,
an attacker could maliciously tamper with a generated source archive
in an attempt to exploit
a known vulnerability like [Zip Slip],
or a common software weakness like susceptibility to a [Zip bomb].
Swift Package Manager should take care to
identify and protect against these kinds of attacks 
in its implementation of source archive decompression.

### Repudiation

A compromised host could serve a malicious package with a valid checksum
and be unable to deny its involvement in constructing the forgery.

This threat is unique and specific to binary and source artifacts;
Git repositories can have their histories audited,
and individual commits may be cryptographically signed by authors.
Unless you can establish a direct connection between
an artifact and a commit in a source tree,
there's no way to determine the provenance of that artifact.

Source archives generated by [`git-archive(1)`]
include the checksum of the `HEAD` commit as a comment.
If the history of a project is available
and the commit used to generate the source archive is signed with [GPG],
the cryptographic signature may be used to verify the authenticity.

```console
$ git rev-parse HEAD
b7c37c81f164e5dce0f64e3d75c79a48fb1fe00b3

$ swift package archive-source -o LinkedList-1.2.0.zip
Generated LinkedList-1.2.0.zip

$ zipnote LinkedList-1.2.0.zip | grep "@ (zip file comment below this line)" -A 1 | tail -n 1
b7c37c81f164e5dce0f64e3d75c79a48fb1fe00b3

$ git verify-commit b7c37c81f164e5dce0f64e3d75c79a48fb1fe00b3
gpg: Signature made Tue Dec 16 00:00:00 2020 PST
gpg:                using RSA key BFAA7114B920808AA4365C203C5C1CF
gpg: Good signature from "Mona Lisa Octocat <mona@noreply.github.com>" [ultimate]
```

Otherwise,
a checksum database and the use of digital signatures
can both provide similar non-repudiation guarantees.

### Information disclosure

A user may inadvertently expose credentials
by checking in their project's configuration files.
An attacker could scrape public code repositories for configuration files
and attempt to reuse credentials to impersonate the user.

The risk of leaking credentials can be mitigated by
storing them in a `.netrc` file located outside the project directory
(typically in the user's home directory).
However,
a user may run `swift package` subcommands with the `--netrc-file` option
to configure the location of their project's `.netrc` file.
To mitigate the risk of a user inadvertently
adding a local `.netrc` file to version control,
Swift Package Manager could add an entry to the `.gitignore` file template
for new projects created with `swift package init`.

Code hosting providers can also help minimize this risk
by [detecting secrets][secret scanning]
that are committed to public repositories.

Credentials may also be unintentionally disclosed
by Swift Package Manager or other tools in logging statements.
Care should be taken to redact usernames and passwords
when displaying feedback to the user.

### Denial of service

An attacker could scrape public code repositories
for `.swiftpm/configuration/registries.json` files
that declare one or more custom registries
and launch a denial-of-service attack
in an attempt to reduce the availability of those resources.

```json
{
  "registries": {
      "[default]": {
        "url": "https://private.example.com"
      }
  },
  "version": 1
}

```

The likelihood of this attack is generally low
but could be used in a targeted way
against resources known to be important or expensive to distribute.

This kind of attack can be mitigated on an individual basis
by adding `.swiftpm/configuration` to a project's `.gitignore` file.

### Escalation of privilege

Even authentic packages from trusted creators can contain malicious code.

Code analysis tools can help to some degree,
as can system permissions and other OS-level security features.
However, developers are ultimately responsible for the code they ship to users.

## Impact on existing packages

Current packages won't be affected by this change,
as they'll continue to download dependencies directly through Git.

## Alternatives considered

### Use of alternative naming schemes

Some package systems,
including [RubyGems], [PyPI], and [CocoaPods]
identify packages with bare names in a flat namespace
(for example, `rails`, `pandas`, or `Alamofire`).
Other systems,
including [Maven],
use [reverse domain name notation] to identify software components
(for example, `com.squareup.okhttp3`).

We considered these and other schemes for identifying packages,
but they were rejected in favor of the scoped package identity
described in this proposal.

### Use of `tar` or other archive formats

Swift Package Manager currently uses Zip archives for binary dependencies,
which is reason enough to use it again here.

Zip files are also a convenient format for package registries,
because they support the access of individual files within an archive.
This allows a registry to satisfy
the package manifest endpoint
(`GET /{scope}/{name}/{version}/Package.swift`)
without storing anything separately from the archive used for the
package archive endpoint
(`GET /{scope}/{name}/{version}.zip`).

We briefly considered `tar` as an archive format
but concluded that its behavior of preserving symbolic links and executable bits
served no useful purpose in the context of package management,
and instead raised concerns about portability and security.

### Inclusion of alternative source locations in package releases payload

To maintain compatibility with existing, URL-based dependency declarations
Swift Package Manager needs to reconcile source locations
with their respective identifiers.
For example,
the declarations
`.package(url: "https://github.com/mona/LinkedList", .exact("1.1.0"))` and
`.package(id: "mona.LinkedList", .exact("1.1.0"))`,
must be deemed equivalent
to resolve a dependency graph that contains both of them.

We considered including alternative source locations in the response body,
but rejected that in favor of using link relations.

[Web linking][RFC 8288] provides a standard way to
describe the relationships between resources.
Standard `canonical` and `alternative` [IANA link relations]
convey precise semantics for
the relationship between a package and its source repositories
that are broadly useful beyond any individual client.

### Addition of an `unarchive-source` subcommand

This proposal adds an `archive-source` subcommand
as a standard way for developers and registries
to create source archives for packages.
Having a canonical tool for creating source archives
avoids any confusion when attempting to verify the integrity of
Zip files sent from a registry
with the source code for that package.

We considered including a complementary `unarchive-source` subcommand
but ultimately decided against it,
the reason being that unarchiving a Zip archive
is unambiguous and well-supported on most platforms.

### Use of digital signatures

[SE-0272] includes a discussion about
the use of digital signatures for binary dependencies,
concluding that they were unsuitable
because of complexity around transitive dependencies.
However, it's unclear what specific objections were raised in this proposal.
We didn't see any inherent tension with the example provided,
and no further explanation was given.

Without understanding the context of this decision,
we decided it was best to abide by their determination
and instead consider adding this functionality in a future proposal.
For the reasons outlined in the preceding Security section,
we believe that digital signatures may offer additional guarantees
of authenticity and non-repudiation beyond what's possible with checksums alone.

## Future directions

Defining a standard interface for package registries
lays the groundwork for several useful features.

### Package publishing

A package registry is responsible for determining
which package releases are made available to a consumer.
This proposal sets no policies for how
package releases are published to a registry.
Nor does it specify how package scopes are registered or verified.

Many package managers —
including the ones mentioned above —
and artifact repository services, such as
[Docker Hub],
[JFrog Artifactory],
and [AWS CodeArtifact]
follow what we describe as a *"push"* model of publication:
When a package owner wants to releases a new version of their software,
they produce a build locally and push the resulting artifact to a server.
This model has the benefit of operational simplicity and flexibility.
For example,
maintainers have an opportunity to digitally sign artifacts
before uploading them to the server.

Alternatively,
a system might incorporate build automation techniques like
continuous integration (CI) and continuous delivery (CD)
into what we describe as a *"pull"* model:
When a package owner wants to release a new version of their software,
their sole responsibility is to notify the package registry;
the server does all the work of downloading the source code
and packaging it up for distribution.
This model can provide strong guarantees about
reproducibility, quality assurance, and software traceability.

We intend to work with industry stakeholders
to develop standards for publishing Swift packages
in an extension to the registry specification.

### Package removal

Removing a package from a registry
can break other packages that depend on it,
as demonstrated by the ["left-pad" incident][left-pad] in March 2016.
We believe package registries can and should
provide strong durability guarantees
to ensure the health of the ecosystem.

At the same time,
there are valid reasons why a package release may be removed:

* The package maintainer publishing a release by mistake
* A security researcher disclosing a vulnerability for a release
* The registry being compelled by law enforcement to remove a release

It's unclear whether and to what extent package deletion policies
should be informed by the registry specification itself.
For now,
a registry is free to exercise its own discretion
about how to respond to out-of-band removal requests.

We plan to consider these questions
as part of the future extension to the specification
described in the previous section.

### Package dependency URL normalization

As described in ["Package name collision resolution"](#package-name-collision-resolution)
Swift Package Manager cannot build a project
if two or more packages in the project
are located by URLs with the same (case-insensitive) last path component.
Swift Package Manager may improve support URL-based dependencies
by normalizing package URLs to mitigate insignificant variations.
For example,
a package with an ["scp-style" URL][scp-url] like
`git@github.com:mona/LinkedList.git`
may be determined to be equivalent to a package with an HTTPS scheme like
`https:///github.com/mona/LinkedList`.

### Local offline cache

Swift Package Manager could implement an [offline cache]
that would allow it to work without network access.
While this is technically possible today,
a package registry makes for a simpler and more secure implementation
than would otherwise be possible with Git repositories alone.

### Binary framework distribution

The registry specification could be amended to support the distribution of
[XCFramework] bundles or [artifact archives][SE-0305].

```http
GET /github.com/mona/LinkedList/1.1.1.xcframework HTTP/1.1
Host: packages.github.com
Accept: application/vnd.swift.registry.v1+xcframework
```

Swift Package Manager could then use XCFramework archives as
[binary dependencies][SE-0272]
or as part of a future binary artifact distribution mechanism.

```swift
let package = Package(
    name: "SomePackage",
    /* ... */
    targets: [
        .binaryTarget(
            name: "LinkedList",
            url: "https://packages.github.com/github.com/mona/LinkedList/1.1.1.xcframework",
            checksum: "ed04a550c2c7537f2a02ab44dd329f9e74f9f4d3e773eb883132e0aa51438b37"
        ),
    ]
)
```

### Updates to package editor commands

[Package editor commands][SE-0301]
could be extended to add dependencies using scoped identifiers
in addition to URLs.

```console
$ swift package add-dependency mona.LinkedList
# Installed LinkedList 1.2.0
```

```diff
+    .package(id: "mona.LinkedList", .exact("1.2.0"))
```

### Package manifest dependency migration

Swift Package Manager could add tooling
to help package maintainers adopt registry-supported identifiers
in their projects.

```console
$ swift package-registry migrate
```

```diff
-    .package(url: "https://github.com/mona/LinkedList", .exact("1.2.0"))
+    .package(id: "mona.LinkedList", .exact("1.2.0"))
```

### Security audits

The response for listing package releases could be updated to include
information about security advisories.

```jsonc
{
    "releases": { /* ... */ },
    "advisories": [{
        "cve": "CVE-20XX-12345",
        "cwe": "CWE-400",
        "package": "mona.LinkedList",
        "vulnerable_versions": "<=1.0.0",
        "patched_versions": ">1.0.0",
        "severity": "moderate",
        "recommendation": "Update to version 1.0.1 or later.",
        /* additional fields */
    }]
}
```

Swift Package Manager could communicate this information to users
when installing or updating dependencies
or as part of a new `swift package audit` subcommand.

```console
$ swift package audit
┌───────────────┬────────────────────────────────────────────────┐
│ High          │ Regular Expression Denial of Service           │
├───────────────┼────────────────────────────────────────────────┤
│ Package       │ mona.RegEx                                     │
├───────────────┼────────────────────────────────────────────────┤
│ Dependency of │ PatternMatcher                                 │
├───────────────┼────────────────────────────────────────────────┤
│ Path          │ SomePackage > PatternMatcher > RegEx           │
├───────────────┼────────────────────────────────────────────────┤
│ More info     │ https://example.com/advisories/526             │
└───────────────┴────────────────────────────────────────────────┘

Found 3 vulnerabilities (1 low, 1 moderate, 1 high) in 8 scanned packages.
  Run `swift package audit fix` to fix 3 of them.
```

### Package search

The package registry API could be extended to add a search endpoint
to allow users to search for packages by name, keywords, or other criteria.
This endpoint could be used by clients like Swift Package Manager.

```console
$ swift package search LinkedList
LinkedList (github.com/mona/LinkedList) - One thing links to another.

$ swift package search --author "Mona Lisa Octocat"
LinkedList (github.com/mona/LinkedList) - One thing links to another.
RegEx (github.com/mona/RegEx) - Expressions on the reg.
```

[AWS CodeArtifact]: https://aws.amazon.com/codeartifact/
[BCP 13]: https://tools.ietf.org/html/rfc6838 "Media Type Specifications and Registration Procedures"
[CDN]: https://en.wikipedia.org/wiki/Content_delivery_network "Content delivery network"
[checksum database]: https://sum.golang.org "Go Module Mirror, Index, and Checksum Database"
[CocoaPods]: https://cocoapods.org "A dependency manager for Swift and Objective-C Cocoa projects"
[crates.io]: https://crates.io "crates.io: The Rust community’s crate registry"
[Docker Hub]: https://hub.docker.com
[GPG]: https://gnupg.org
[homograph attacks]: https://en.wikipedia.org/wiki/IDN_homograph_attack
[http header injection]: https://en.wikipedia.org/wiki/HTTP_header_injection
[IANA link relations]: https://www.iana.org/assignments/link-relations/link-relations.xhtml "IANA Link Relation Types"
[ICANN]: https://www.icann.org
[JFrog Artifactory]: https://jfrog.com/artifactory/
[JSON-LD]: https://w3c.github.io/json-ld-syntax/ "JSON-LD 1.1: A JSON-based Serialization for Linked Data"
[left-pad]: https://qz.com/646467/how-one-programmer-broke-the-internet-by-deleting-a-tiny-piece-of-code/ "How one programmer broke the internet by deleting a tiny piece of code"
[Maven]: https://maven.apache.org
[npm]: https://www.npmjs.com "The npm Registry"
[offline cache]: https://yarnpkg.com/features/offline-cache "Offline Cache | Yarn - Package Manager"
[PyPI]: https://pypi.org "PyPI: The Python Package Index"
[reverse domain name notation]: https://en.wikipedia.org/wiki/Reverse_domain_name_notation
[RFC 2119]: https://tools.ietf.org/html/rfc2119 "Key words for use in RFCs to Indicate Requirement Levels"
[RFC 3230]: https://tools.ietf.org/html/rfc5843 "Instance Digests in HTTP"
[RFC 3492]: https://tools.ietf.org/html/rfc3492 "Punycode: A Bootstring encoding of Unicode for Internationalized Domain Names in Applications (IDNA)"
[RFC 3986]: https://tools.ietf.org/html/rfc3986 "Uniform Resource Identifier (URI): Generic Syntax"
[RFC 3987]: https://tools.ietf.org/html/rfc3987 "Internationalized Resource Identifiers (IRIs)"
[RFC 5234]: https://tools.ietf.org/html/rfc5234 "Augmented BNF for Syntax Specifications: ABNF"
[RFC 5843]: https://tools.ietf.org/html/rfc5843 "Additional Hash Algorithms for HTTP Instance Digests"
[RFC 6249]: https://tools.ietf.org/html/rfc6249 "Metalink/HTTP: Mirrors and Hashes"
[RFC 6570]: https://tools.ietf.org/html/rfc6570 "URI Template"
[RFC 6749]: https://tools.ietf.org/html/rfc6749 "The OAuth 2.0 Authorization Framework"
[RFC 7230]: https://tools.ietf.org/html/rfc7230 "Hypertext Transfer Protocol (HTTP/1.1): Message Syntax and Routing"
[RFC 7231]: https://tools.ietf.org/html/rfc7231 "Hypertext Transfer Protocol (HTTP/1.1): Semantics and Content"
[RFC 7233]: https://tools.ietf.org/html/rfc7233 "Hypertext Transfer Protocol (HTTP/1.1): Range Requests"
[RFC 7234]: https://tools.ietf.org/html/rfc7234 "Hypertext Transfer Protocol (HTTP/1.1): Caching"
[RFC 7807]: https://tools.ietf.org/html/rfc7807 "Problem Details for HTTP APIs"
[RFC 8288]: https://tools.ietf.org/html/rfc8288 "Web Linking"
[RFC 8446]: https://tools.ietf.org/html/rfc8446 "The Transport Layer Security (TLS) Protocol Version 1.3"
[RubyGems]: https://rubygems.org "RubyGems: The Ruby community’s gem hosting service"
[Schema.org]: https://schema.org/
[scp-url]: https://git-scm.com/book/en/v2/Git-on-the-Server-The-Protocols#_the_ssh_protocol
[SE-0219]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0219-package-manager-dependency-mirroring.md "Package Manager Dependency Mirroring"
[SE-0272]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0272-swiftpm-binary-dependencies.md "Package Manager Binary Dependencies"
[SE-0301]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0301-package-editing-commands.md "Package Editor Commands"
[SE-0305]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0305-swiftpm-binary-target-improvements.md "Package Manager Binary Target Improvements"
[secret scanning]: https://docs.github.com/en/github/administering-a-repository/about-secret-scanning
[SemVer]: https://semver.org/ "Semantic Versioning"
[SoftwareSourceCode]: https://schema.org/SoftwareSourceCode
[STRIDE]: https://en.wikipedia.org/wiki/STRIDE_(security) "STRIDE (security)"
[thundering herd effect]: https://en.wikipedia.org/wiki/Thundering_herd_problem "Thundering herd problem"
[TOFU]: https://en.wikipedia.org/wiki/Trust_on_first_use "Trust on First Use"
[transparent log]: https://research.swtch.com/tlog
[typosquatting]: https://en.wikipedia.org/wiki/Typosquatting
[UTI]: https://en.wikipedia.org/wiki/Uniform_Type_Identifier
[version-specific-manifest-selection]: https://github.com/apple/swift-package-manager/blob/main/Documentation/Usage.md#version-specific-manifest-selection "Swift Package Manager - Version-specific Manifest Selection"
[version-specific-tag-selection]: https://github.com/apple/swift-package-manager/blob/main/Documentation/Usage.md#version-specific-tag-selection "Swift Package Manager - Version-specific Tag Selection"
[XCFramework]: https://developer.apple.com/videos/play/wwdc2019/416/ "WWDC 2019 Session 416: Binary Frameworks in Swift"
[xss]: https://en.wikipedia.org/wiki/Cross-site_scripting
[Zip bomb]: https://en.wikipedia.org/wiki/Zip_bomb "Zip bomb"
[Zip Slip]: https://snyk.io/research/zip-slip-vulnerability "Zip Slip Vulnerability"
