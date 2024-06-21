# Package Registry Service - Publish Endpoint

* Proposal: [SE-0321](0321-package-registry-publish.md)
* Authors: [Whitney Imura](https://github.com/whitneyimura),
           [Mattt Zmuda](https://github.com/mattt)
* Review Manager: [Tom Doron](https://github.com/tomerd)
* Status: **Accepted (2021-09-01)**
* Implementation: [apple/swift-package-manager#3671](https://github.com/apple/swift-package-manager/pull/3671)
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-se-0321-package-registry-service-publish-endpoint/51660)

## Introduction

The [package registry service][SE-0292] defines endpoints for fetching packages.

This proposal extends the existing [package registry specification][Registry.md]
with endpoints for publishing a package release.

## Motivation

A package registry is responsible for determining
which package releases are made available to a consumer.

Currently, the availability of a package release
is determined by an out-of-band process.
For example,
a registry may consult an index of public Swift packages
and make releases available for each tag with a valid version number.

Having a standard endpoint for publishing a new release to a package registry
would empower maintainers to distribute their software
and promote interoperability across service providers.

## Proposed solution

We propose to add the following endpoint to the registry specification:

| Method  | Path                          | Description              |
| ------- | ----------------------------- | -------------------------|
| `PUT`   | `/{scope}/{name}/{version}`   | Create a package release |

The goal of this proposal is to provide enough definition to ensure
a secure, robust mechanism for distributing software
while allowing individual registries enough flexibility in their
governance and operation.
For instance,
support for this endpoint would be optional,
so package registries may elect not to allow packages to be published.
And because there's an expectation of durability —
that is, package releases aren't removed after they're published —
registries make the ultimate determination of what is made available.

## Detailed design

This proposal amends the registry specification with a new, optional endpoint
that a registry may implement to support the publication of packages
through the web service interface.
To understand what the feature does and how it works,
consider the following use case:

A maintainer of an open-source Swift package (`mona.LinkedList`)
creates a new release (version `1.1.1`),
and wants to submit it to a registry (`packages.example.com`) for distribution.

First, they run the `swift package archive-source` subcommand
to generate a Zip file (`LinkedList-1.1.1.zip`) of their package.

```console
$ swift package archive-source
```

Next, they upload their release to a package registry
by making the following request:

```console
$ curl -X PUT --netrc                                      \
       -H "Accept: application/vnd.swift.registry.v1+json" \
       -F source-archive="@LinkedList-1.1.1.zip"           \
       "https://registry.example.com/mona/LinkedList?version=1.1.1"
```

The registry can respond to this request synchronously or asynchronously.
This allows the server an opportunity to perform any necessary
analysis and processing to ensure software quality and update its data stores.

After receiving and processing this request,
the registry can make `mona.LinkedList` at version `1.1.1` available
by including it in the response to `GET /mona/LinkedList`.

```console
$ curl -X GET -H "Accept: application/vnd.swift.registry.v1+json" \
       "https://registry.example.com/mona/LinkedList"             \
    | jq ".[] | keys"
[
  "1.0.0",
  "1.1.0",
  "1.1.1",
]
```

The next time a developer with a package that depends on `mona.LinkedList`
resolves the dependencies of that package,
Swift Package Manager would see `1.1.1`,
and may attempt to update to this new version.
If the version is selected,
the client would download the source archive for this release
by sending the request `GET /mona/LinkedList/1.1.1.zip`.

## Security

Although this proposal has no direct impact on Swift Package Manager,
it's important to consider the security implications of
introducing a publishing endpoint to the Swift package ecosystem.
To do this,
we employ the
<abbr title="Spoofing, Tampering, Repudiation, Information disclosure, Denial of service, Escalation of privilege">
[STRIDE]
</abbr> mnemonic below:

### Spoofing

An attacker could attempt to impersonate a package maintainer
in order to publish a new release containing malicious code.

Because the likelihood and potential impact of such an attack is high,
registry service providers should take all necessary precautions.
The registry specification recommends the use of multi-factor authentication
for all requests to publish a package release.

Additional countermeasures like
rate-limiting suspicious requests and
analyzing uploaded source archives
can also help mitigate the risk of this kind of attack.

An attacker could also attempt to trick users into downloading malicious code
by publishing a package with an identifier similar to a legitimate one
(for example, `4pple.swift-nio`, which looks like `apple/swift-nio`).
A registry can mitigate typosquatting attacks like this
by comparing the similarity of a new submission to existing package names
with a string metric like [Damerau–Levenshtein distance].

### Tampering

An attacker could maliciously tamper with a generated source archive
in an attempt to exploit
a known vulnerability like [Zip Slip],
or a common software weakness like susceptibility to a [Zip bomb].

Registry services should take care to
identify and protect against these kinds of attacks
in its implementation of source archive decompression.

To further improve the security of package submissions,
a registry could restrict publishing to trusted clients,
for which a chain of custody can be established.
(This is effectively the "pull" model described above).

### Repudiation

A dispute could arise between a package maintainer and a registry
about the content or existence of a package release.

This proposal doesn't specifically provide a mechanism
for resolving such a dispute.
However, the design supports a variety of possible solutions.
For example,
a software bill of materials and the use of digital signatures
can both provide non-repudiation guarantees
about the provenance of package release artifacts.

### Information disclosure

A user could inadvertently expose credentials
when uploading a source archive for a package release.

This threat isn't substantially different from that of
leaking credentials in source code with version management software,
so similar strategies can be employed here.
For example,
registry services can help minimize this risk
by rejecting any submissions that [contain sensitive information][secret scanning].

### Denial of service

An attacker could upload large payloads
in an attempt to reduce the availability of a registry.

This kind of attack is typical for any web service
with an endpoint for uploading resources.
A registry can mitigate this threat using defensive coding practices like
performing authentication checks before processing request bodies,
limiting the maximum allowed size of a message payload, and
routing requests through a reverse proxy or load balancer.

### Escalation of privilege

It's desirable for a registry to have information about
the content of a release submitted for publishing,
such as the package's supported platforms, products, and dependencies.
Swift package manifest files are executable code
and must be evaluated by the Swift toolchain to determine this information.
An attacker could construct a malicious `Package.swift` file
containing system calls in an attempt to perform remote code execution.

Registry services should take care to evaluate package manifest files
in an unprivileged container to mitigate the risk of evaluating untrusted code.

## Impact on existing packages

This feature provides a mechanism for package maintainers and registries
to migrate existing packages from the current URL-based system
to the new registry scheme.

The specific strategy for rolling out this functionality
is something to be determined by each registry operator
in advance of this feature.

## Alternatives considered

### Endpoint for scope registration

This proposal sets no policies for how
package scopes are registered or verified.

### Endpoint for publishing with "pull" model

Many package managers and artifact repository services
follow what we describe as a *"push"* model of publication:
When a maintainer wants to releases a new version of their software,
they produce a build locally and push the resulting artifact to a server.

For example,
a developer can distribute their Ruby library
by building a `.gem` archive and pushing it to a server like [RubyGems.org].

```console
$ gem build octokit.gemspec
$ gem push octokit-4.20.0.gem
```

This model has the benefit of operational simplicity and flexibility.
For example,
maintainers have an opportunity to digitally sign artifacts
before uploading them to the server.

Alternatively,
a system might incorporate build automation techniques like
continuous integration (CI) and continuous delivery (CD)
into what we describe as a *"pull"* model:
When a maintainer wants to release a new version of their software,
their sole responsibility is to notify the registry;
the server does all the work of downloading the source code
and packaging it up for distribution.

For example,
in addition to supporting the "push" model,
[Docker Hub] can [automatically build][autobuilds] images from source code
push the built image to a repository.

This model can provide strong guarantees about
reproducibility, quality assurance, and software traceability.

Initial drafts for this proposal
included separate endpoints for publishing with the "pull" and "push" models,
with a preference for the former and its stronger guarantees of traceability.
However,
we determined that while these models provide
a useful framework for understanding software distribution models,
they are both accommodated by a single endpoint;
a "pull" is equivalent a "push"
where the client and server are a single entity.

## Future directions

### Swift Package Manager subcommand for publishing

Swift Package Manager could be updated to add
a new `swift package publish` subcommand,
that provides a more convenient interface for publishing packages to a registry.
For example,
it could automatically read the configuration in
`.swiftpm/config/registries.json`
to determine the correct registry endpoint,
or read the user's `.netrc` file to authenticate the request.

The command could also subsume `swift package archive-source`
and perform additional tasks before uploading,
such as generating a software bill of materials
or signing the source archive.

This feature wasn't included in the proposal
because it's unnecessary for the core publishing functionality.
We are also concerned that this command
could bloat to the command-line interface
and undermine the benefits of publishing within a CI/CD system.
However,
if the community finds this to be a useful feature,
we'd be happy to include it in an amendment to our proposal.

### Mechanism for syndicating publishing activity

A registry could syndicate new releases through
an [activity stream][activitystreams] or [RSS feed][rss].
This functionality could be used as an information source by package indexes
or to provide federation across different registries.

### Transparency logs

Similar to a syndication feed,
each new package release could be added to an append-only log
like [Trillian] or [sigstore].

[activitystreams]: https://www.w3.org/TR/activitystreams-core/ "Activity Streams 2.0"
[autobuilds]: https://docs.docker.com/docker-hub/builds/ "Docker Hub: Set up Automated Builds"
[Damerau–Levenshtein distance]: https://en.wikipedia.org/wiki/Damerau%E2%80%93Levenshtein_distance "Damerau–Levenshtein distance"
[Docker Hub]: https://hub.docker.com
[Registry.md]: https://github.com/apple/swift-package-manager/blob/main/Documentation/PackageRegistry/Registry.md "Swift Package Registry Service Specification"
[rss]: https://validator.w3.org/feed/docs/rss2.html "RSS 2.0 specification"
[RubyGems.org]: https://rubygems.org/
[SE-0292]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0292-package-registry-service.md "Package Registry Service"
[secret scanning]: https://docs.github.com/en/github/administering-a-repository/about-secret-scanning
[sigstore]: https://sigstore.dev/ "sigstore: A non-profit, public good software signing & transparency service"
[STRIDE]: https://en.wikipedia.org/wiki/STRIDE_(security) "STRIDE (security)"
[Trillian]: https://github.com/google/trillian "Trillian: A transparent, highly scalable and cryptographically verifiable data store."
[Zip bomb]: https://en.wikipedia.org/wiki/Zip_bomb "Zip bomb"
[Zip Slip]: https://snyk.io/research/zip-slip-vulnerability "Zip Slip Vulnerability"
