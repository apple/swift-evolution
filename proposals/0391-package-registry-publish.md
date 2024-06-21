# Package Registry Publish

* Proposal: [SE-0391](0391-package-registry-publish.md)
* Author: [Yim Lee](https://github.com/yim-lee)
* Review Manager: [Tom Doron](https://github.com/tomerd)
* Status: **Implemented (Swift 5.9)**
* Implementation:
  * [apple/swift-package-manager#6101](https://github.com/apple/swift-package-manager/pull/6101)
  * [apple/swift-package-manager#6146](https://github.com/apple/swift-package-manager/pull/6146)
  * [apple/swift-package-manager#6159](https://github.com/apple/swift-package-manager/pull/6159)
  * [apple/swift-package-manager#6169](https://github.com/apple/swift-package-manager/pull/6169)
  * [apple/swift-package-manager#6188](https://github.com/apple/swift-package-manager/pull/6188)
  * [apple/swift-package-manager#6189](https://github.com/apple/swift-package-manager/pull/6189)
  * [apple/swift-package-manager#6215](https://github.com/apple/swift-package-manager/pull/6215)
  * [apple/swift-package-manager#6217](https://github.com/apple/swift-package-manager/pull/6217)
  * [apple/swift-package-manager#6220](https://github.com/apple/swift-package-manager/pull/6220)
  * [apple/swift-package-manager#6229](https://github.com/apple/swift-package-manager/pull/6229)
  * [apple/swift-package-manager#6237](https://github.com/apple/swift-package-manager/pull/6237)
* Review: ([pitch](https://forums.swift.org/t/pitch-package-registry-publish/62828)), ([review](https://forums.swift.org/t/se-0391-package-registry-publish/63405)), ([acceptance](https://forums.swift.org/t/accepted-se-0391-swift-package-registry-authentication/64088))

## Introduction

A package registry makes packages available to consumers. Starting with Swift 5.7,
SwiftPM supports dependency resolution and package download using any registry that 
implements the [service specification](https://github.com/apple/swift-package-manager/blob/main/Documentation/PackageRegistry/Registry.md) proposed alongside with [SE-0292](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0292-package-registry-service.md).
SwiftPM does not yet provide any tooling for publishing packages, so package authors 
must manually prepare the contents (e.g., source archive) and interact 
with the registry on their own to publish a package release. This proposal 
aims to standardize package publishing such that SwiftPM can offer a complete and 
well-rounded experience for using package registries.

## Motivation

Publishing package release to a Swift package registry generally involves these steps:
  1. Gather package release metadata.
  1. Prepare package source archive by using the [`swift package archive-source` subcommand](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0292-package-registry-service.md#archive-source-subcommand).
  1. Sign the metadata and archive (if needed).
  1. [Authenticate](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0378-package-registry-auth.md) (if required by the registry).
  1. Send the archive and metadata (and their signatures if any) by calling the ["create a package release" API](https://github.com/apple/swift-package-manager/blob/main/Documentation/PackageRegistry/Registry.md#endpoint-6).
  1. Check registry server response to determine if publication has succeeded or failed (if the registry processes request synchronously), or is pending (if the registry processes request asynchronously).

SwiftPM can streamline the workflow by combining all of these steps into a single 
`publish` command.

## Proposed solution

We propose to introduce a new `swift package-registry publish` subcommand to SwiftPM 
as well as standardization on package release metadata and package signing to ensure a
consistent user experience for publishing packages.

## Detailed design

### Package release metadata

Typically a package release has metadata associated with it, such as URL of the source
code repository, license, etc. In general, metadata gets set when a package release is
being published, but a registry service may allow modifications of the metadata afterwards.

The current [registry service specification](https://github.com/apple/swift-package-manager/blob/main/Documentation/PackageRegistry/Registry.md) states that:
  - A client (e.g., package author, publishing tool) may provide metadata for a package release by including it in the ["create a package release" request](https://github.com/apple/swift-package-manager/blob/main/Documentation/PackageRegistry/Registry.md#462-package-release-metadata). The registry server will store the metadata and include it in the ["fetch information about a package release" response](https://github.com/apple/swift-package-manager/blob/main/Documentation/PackageRegistry/Registry.md#endpoint-2).
  - If a client does not include metadata, the registry server may populate it unless the client specifies otherwise (i.e., by sending an empty JSON object `{}` in the "create a package release" request).

It does not, however, define any requirements or server-client API contract on the 
metadata contents. We would like to change that by proposing the following:
  - Package release metadata will continue to be sent as a JSON object.
  - Package release metadata must adhere to the [schema](#package-release-metadata-standards).
  - Package release metadata will continue to be included in the "create a package release" request as a multipart section named `metadata` in the request body.
  - Registry server may allow and/or populate additional metadata by expanding the schema, but it must not alter any of the predefined properties. 
  - Registry server may make any properties in the schema and additional metadata it defines required. Registry server may fail the "create a package release" request if any required metadata is missing.
  - Client cannot change how registry server handles package release metadata. In other words, client will no longer be able to instruct registry server not to populate metadata by sending an empty JSON object `{}`.
  - Registry server will continue to include metadata in the "fetch information about a package release" response.
  
#### Package release metadata standards

Package release metadata submitted to a registry must be a JSON object of type 
[`PackageRelease`](#packagerelease-type), the schema of which is defined below.

<details>

<summary>Expand to view <a href="https://json-schema.org/specification.html">JSON schema</a></summary>  

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://github.com/apple/swift-package-manager/blob/main/Documentation/PackageRegistry/Registry.md",
  "title": "Package Release Metadata",
  "description": "Metadata of a package release.",
  "type": "object",
  "properties": {
    "author": {
      "type": "object",
      "properties": {
        "name": {
          "type": "string",      
          "description": "Name of the author."
        },  
        "email": {
          "type": "string",      
          "description": "Email address of the author."
        },              
        "description": {
          "type": "string",      
          "description": "A description of the author."
        },
        "organization": {
          "type": "object",
          "properties": {
            "name": {
              "type": "string",      
              "description": "Name of the organization."
            },  
            "email": {
              "type": "string",      
              "description": "Email address of the organization."
            },              
            "description": {
              "type": "string",      
              "description": "A description of the organization."
            },        
            "url": {
              "type": "string",      
              "description": "URL of the organization."
            },        
          },
          "required": ["name"]
        },                
        "url": {
          "type": "string",      
          "description": "URL of the author."
        },        
      },
      "required": ["name"]
    },
    "description": {
      "type": "string",      
      "description": "A description of the package release."
    },
    "licenseURL": {
      "type": "string",
      "description": "URL of the package release's license document."
    },
    "readmeURL": {
      "type": "string",      
      "description": "URL of the README specifically for the package release or broadly for the package."
    },
    "repositoryURLs": {
      "type": "array",
      "description": "Code repository URL(s) of the package release.",
      "items": {
        "type": "string",
        "description": "Code repository URL."
      }      
    }
  }
}
```

</details>

##### `PackageRelease` type

| Property          | Type                | Description                                      | Required |
| ----------------- | :-----------------: | ------------------------------------------------ | :------: |
| `author`          | [Author](#author-type) | Author of the package release. | |
| `description`     | String | A description of the package release. | |
| `licenseURL`      | String | URL of the package release's license document. | |
| `readmeURL`       | String | URL of the README specifically for the package release or broadly for the package. | |
| `repositoryURLs`  | Array | Code repository URL(s) of the package. It is recommended to include all URL variations (e.g., SSH, HTTPS) for the same repository. This can be an empty array if the package does not have source control representation.<br/>Setting this property is one way through which a registry can obtain repository URL to package identifier mappings for the ["lookup package identifiers registered for a URL" API](https://github.com/apple/swift-package-manager/blob/main/Documentation/PackageRegistry/Registry.md#45-lookup-package-identifiers-registered-for-a-url). A registry may choose other mechanism(s) for package authors to specify such mappings. | |

##### `Author` type

| Property          | Type                | Description                                      | Required |
| ----------------- | :-----------------: | ------------------------------------------------ | :------: |
| `name`            | String | Name of the author. | ✓ |
| `email`           | String | Email address of the author. | |
| `description`     | String | A description of the author. | |
| `organization`    | [Organization](#organization-type) | Organization that the author belongs to. | |
| `url`             | String | URL of the author. | |

##### `Organization` type

| Property          | Type                | Description                                      | Required |
| ----------------- | :-----------------: | ------------------------------------------------ | :------: |
| `name`            | String | Name of the organization. | ✓ |
| `email`           | String | Email address of the organization. | |
| `description`     | String | A description of the organization. | |
| `url`             | String | URL of the organization. | |

### Package signing

A registry may require packages to be signed. In order for SwiftPM to be able to
download and handle signed packages from a registry, we propose to standardize 
package signature format and establish server-client API contract on package
signing.

#### Package signature

Package signature format will be identified by the underlying standard/technology 
(e.g., Cryptographic Message Syntax (CMS), JSON Web Signature (JWS), etc.) and 
version number. In the initial release, all signatures will be in [CMS](https://www.rfc-editor.org/rfc/rfc5652.html).

| Signature format ID | Description                                               |
| ------------------- | --------------------------------------------------------- |
| `cms-1.0.0`         | Version 1.0.0 of package signature in CMS                 |

##### Package signature format `cms-1.0.0`

Package signature format `cms-1.0.0` uses CMS.

| CMS Attribute                  | Details                                                   |
| ------------------------------ | --------------------------------------------------------- |
| Content type                   | [Signed-Data](https://www.rfc-editor.org/rfc/rfc5652.html#section-5) |
| Encapsulated data              | The content being signed ([`EncapsulatedContentInfo.eContent`](https://www.rfc-editor.org/rfc/rfc5652.html#section-5.2)) is omitted since we are constructing an external signature. |
| Message digest algorithm       | SHA-256, computed on the package source archive.          |
| Signature algorithm            | ECDSA P-256                                               |
| Number of signatures           | 1                                                         |
| Certificate                    | Certificate that contains the signing key. It is up to the registry to define the certificate policy (e.g., trusted root(s)). |

The signature, represented in CMS, will be included as part
of the "create a package release" API request. 

A registry receiving such signed package will:
  - Check if the signature format (`cms-1.0.0`) is accepted.
  - Validate the signature is well-formed according to the signature format.
  - Validate the certificate chain meets registry policy.
  - Extract public key from the certificate and use it to verify the signature.

Then the registry will process the package and save it for client downloads if publishing is successful.

The registry must include signature information in the "fetch information about a package release" API 
response to indicate the package is signed and the signature format (`cms-1.0.0`).

After downloading a signed package SwiftPM will:
  - Check if the signature format (`cms-1.0.0`) is supported.
  - Validate the signature is well-formed according to the signature format.
  - Validate that the signed package complies with the locally-configured signing policy.
  - Extract public key from the certificate and use it to verify the signature.

#### Server-side requirements for package signing

A registry that requires package signing should provide documentations
on the signing requirements (e.g., any requirements for certificates 
used in signing).

A registry must also modify the ["create package release" API](#create-package-release-api) to allow
signature in the request, as well as the response for the ["fetch package release metadata"](#fetch-package-release-metadata-api)
and ["download package source archive"](#download-package-source-archive-api) API to include signature information.
    
#### SwiftPM's handling of registry packages

##### SwiftPM configuration

Users will be able to configure how SwiftPM handles packages downloaded from a 
registry. In the user-level `registries.json` file, which by default is located at 
`~/.swiftpm/configuration/registries.json`, we will introduce a new `security` key:

```json5
{
  "security": {
    "default": {
      "signing": {
        "onUnsigned": "prompt", // One of: "error", "prompt", "warn", "silentAllow"
        "onUntrustedCertificate": "prompt", // One of: "error", "prompt", "warn", "silentAllow"
        "trustedRootCertificatesPath": "~/.swiftpm/security/trusted-root-certs/",
        "includeDefaultTrustedRootCertificates": true,
        "validationChecks": {
          "certificateExpiration": "disabled", // One of: "enabled", "disabled"
          "certificateRevocation": "disabled"  // One of: "strict", "allowSoftFail", "disabled"
        }
      }
    },
    "registryOverrides": {
      // The example shows all configuration overridable at registry level
      "packages.example.com": {
        "signing": {
          "onUnsigned": "warn",
          "onUntrustedCertificate": "warn",
          "trustedRootCertificatesPath": <STRING>,
          "includeDefaultTrustedRootCertificates": <BOOL>,
          "validationChecks": {
            "certificateExpiration": "enabled",
            "certificateRevocation": "allowSoftFail"
          }
        }
      }
    },
    "scopeOverrides": {
      // The example shows all configuration overridable at scope level
      "mona": {
        "signing": {
          "trustedRootCertificatesPath": <STRING>,
          "includeDefaultTrustedRootCertificates": <BOOL>
        }
      }
    },
    "packageOverrides": {
      // The example shows all configuration overridable at package level
      "mona.LinkedList": {
        "signing": {
          "trustedRootCertificatesPath": <STRING>,
          "includeDefaultTrustedRootCertificates": <BOOL>
        }
      }
    }
  },
  ...
}
```

Security configuration for a package is computed using values from 
the following (in descending precedence):
1. `packageOverrides` (if any)
1. `scopeOverrides` (if any)
1. `registryOverrides` (if any)
1. `default`

The `default` JSON object contains all configurable security options 
and their default value when there is no override.

- `signing.onUnsigned`: Indicates how SwiftPM will handle an unsigned package.

  | Option        | Description                                               |
  | ------------- | --------------------------------------------------------- |
  | `error`       | SwiftPM will reject the package and fail the build. |
  | `prompt`      | SwiftPM will prompt user to see if the unsigned package should be allowed. <ul><li>If no, SwiftPM will reject the package and fail the build.</li><li>If yes and the package has never been downloaded, its checksum will be stored for [local TOFU](#local-tofu). Otherwise, if the package has been downloaded before, its checksum must match the previous value or else SwiftPM will reject the package and fail the build.</li></ul> SwiftPM will record user's response to prevent repetitive prompting. |
  | `warn`        | SwiftPM will not prompt user but will emit a warning before proceeding. |
  | `silentAllow` | SwiftPM will allow the unsigned package without prompting user or emitting warning. |

- `signing.onUntrustedCertificate`: Indicates how SwiftPM will handle a package signed with an [untrusted certificate](#trusted-vs-untrusted-certificate).

  | Option        | Description                                               |
  | ------------- | --------------------------------------------------------- |
  | `error`       | SwiftPM will reject the package and fail the build. |
  | `prompt`      | SwiftPM will prompt user to see if the package signed with an untrusted certificate should be allowed. <ul><li>If no, SwiftPM will reject the package and fail the build.</li><li>If yes, SwiftPM will proceed with the package as if it were an unsigned package.</li></ul> SwiftPM will record user's response to prevent repetitive prompting. |
  | `warn`        | SwiftPM will not prompt user but will emit a warning before proceeding. |
  | `silentAllow` | SwiftPM will allow the package signed with an untrusted certificate without prompting user or emitting warning. |

- `signing.trustedRootCertificatesPath`: Absolute path to the directory containing custom trusted roots. SwiftPM will include these roots in its [trust store](#trusted-vs-untrusted-certificate), and certificates used for package signing must chain to roots found in this store. This configuration allows override at the package, scope, and registry levels.
- `signing.includeDefaultTrustedRootCertificates`: Indicates if SwiftPM should include default trusted roots in its [trust store](#trusted-vs-untrusted-certificate). This configuration allows override at the package, scope, and registry levels.
- `signing.validationChecks`: Validation check settings for the package signature.

  | Validation               | Description                                               |
  | ------------------------ | --------------------------------------------------------------- |
  | `certificateExpiration`  | <ul><li>`enabled`: SwiftPM will check that the current timestamp when downloading falls within the signing certificate's validity period. If it doesn't, SwiftPM will reject the package and fail the build.</li><li>`disabled`: SwiftPM will not perform this check.</li></ul> |
  | `certificateRevocation`  | With the exception of `disabled`, SwiftPM will check revocation status of the signing certificate. SwiftPM will only support revocation check done through [OCSP](https://www.rfc-editor.org/rfc/rfc6960) in the first feature release.<ul><li>`strict`: Revocation check must complete successfully and the certificate must be in good status. SwiftPM will reject the package and fail the build if the revocation status is revoked or unknown (including revocation check not supported or failed).</li><li>`allowSoftFail`: SwiftPM will reject the package and fail the build iff the certificate has been revoked. SwiftPM will allow the certificate's revocation status to be unknown (including revocation check not supported or failed).</li><li>`disabled`: SwiftPM will not perform this check.</li></ul> |

##### Trusted vs. untrusted certificate

A certificate is **trusted** if it is chained to any roots in SwiftPM's 
trust store, which is a combination of:
  - SwiftPM's default trust store, if `signing.includeDefaultTrustedRootCertificates` is `true`.
  - Custom root(s) in the configured trusted roots directory at `signing.trustedRootCertificatesPath`.

Otherwise, a certificate is **untrusted** and handled according to the `signing.onUntrustedCertificate` setting.

Both `signing.includeDefaultTrustedRootCertificates` and `signing.trustedRootCertificatesPath`
support multiple levels of overrides. SwiftPM will choose the configuration value that has the 
highest specificity.


For example, when evaluating the value of `signing.includeDefaultTrustedRootCertificates` 
or `signing.trustedRootCertificatesPath` for package `mona.LinkedList`:
  1. SwiftPM will use the package override from `packageOverrides` (i.e., `packageOverrides["mona.LinkedList"]`), if any.
  1. Otherwise, SwiftPM will use the scope override from `scopeOverrides` (i.e., `scopeOverrides["mona"]`), if any.
  1. Next, depending on the registry the package is downloaded from, SwiftPM will look for and use the registry override in `registryOverrides`, if any.
  1. Finally, if no override is found, SwiftPM will use the value from `default`.

##### Local TOFU

When SwiftPM downloads a package release from registry via the 
["download source archive" API](https://github.com/apple/swift-package-manager/blob/main/Documentation/PackageRegistry/Registry.md#endpoint-4), it will:
  1. Search local fingerprints storage, which by default is located at `~/.swiftpm/security/fingerprints/`, to see if the package release has been downloaded before and its recorded checksum. The checksum of the downloaded source archive must match the previous value or else [trust on first use (TOFU)](https://en.wikipedia.org/wiki/Trust_on_first_use) check would fail.
  1. Fetch package release metadata from the registry to get:
    <ul>
    <li>Checksum for TOFU if the package release is downloaded for the first time.</li>
    <li>Signature information if the package release is signed.</li>
    </ul>
  1. Retrieve security settings from the user-level `registries.json`.
  1. Check if the package is allowed based on security settings.
  1. Validate the signature according to the signature format if package is signed.
  1. Some certificates allow SwiftPM to extract additional information that can drive additional security features. For packages signed with these certificates, SwiftPM will apply additional, publisher-level TOFU by extracting signing identity from the certificate and enforcing the same signing identity across all signed versions of a package. 

### New `package-registry publish` subcommand

The new `package-registry publish` subcommand will create a package
source archive, sign it if needed, and publish it to a registry.

```manpage
> swift package-registry publish --help
OVERVIEW: Publish a package release to registry

USAGE: package-registry publish <id> <version>

ARGUMENTS:
  <package-id>            The package identifier.
  <package-version>       The package release version being created.

OPTIONS:
  --url                   The registry URL.
  --scratch-directory     The path of the directory where working file(s) will be written.

  --metadata-path         The path to the package metadata JSON file if it's not 'package-metadata.json' in the package directory.

  --signing-identity      The label of the signing identity to be retrieved from the system's secrets store if supported.

  --private-key-path      The path to the certificate's PKCS#8 private key (DER-encoded).
  --cert-chain-paths      Path(s) to the signing certificate (DER-encoded) and optionally the rest of the certificate chain. The signing certificate must be listed first.

  --dry-run               Dry run only; prepare the archive and sign it but do not publish to the registry.
```

- `id`: The package identifier in the `<scope>.<name>` notation as defined in [SE-0292](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0292-package-registry-service.md#package-identity). It is the package author's responsibility to register the package identifier with the registry beforehand.
- `version`: The package release version in [SemVer 2.0](https://semver.org) notation.
- `url`: The URL of the registry to publish to. SwiftPM will try to determine the registry URL by searching for a scope-to-registry mapping or use the `[default]` URL in `registries.json`. The command will fail if this value is missing.
- `scratch-directory`: The path of the working directory. SwiftPM will write to the package directory by default.

The following may be required depending on registry support and/or requirements:
  - `metadata-path`: The path to the JSON file containing [package release metadata](#package-release-metadata). By default, SwiftPM will look for a file named `package-metadata.json` in the package directory if this is not specified. SwiftPM will include the content of the metadata file in the request body if present. If the package source archive is being signed, the metadata will be signed as well.
  - `signing-identity`: The label that identifies the signing identity to use for package signing in the system's secrets store if supported. 
  - `private-key-path`: Required for package signing unless `signing-identity` is specified, this is the path to the private key used for signing.
  - `cert-chain-paths`: Required for package signing unless `signing-identity` is specified, this is the signing certificate chain.
  
A signing identity encompasses a private key and a certificate. On 
systems where it is supported, SwiftPM can look for a signing identity 
using the query string given via the `--signing-identity` option. This
feature will be available on macOS through Keychain in the initial 
release, so a certificate and its private key can be located by the
certificate label alone.

Otherwise, both `--private-key-path` and `--cert-chain-paths` must be
provided to locate the signing key and certificate chain.

SwiftPM will sign the package source archive and package release metadata if `signing-identity` 
or both `private-key-path` and `cert-chain-paths` are set.
  
All signatures in the initial release will be in the [`cms-1.0.0`](#package-signature-format-cms-100) format.

Using these inputs, SwiftPM will:
  - Generate source archive for the package release.
  - Sign the source archive and metadata if the required parameters are provided.
  - Make HTTP request to the "create a package release" API.
  - Check server response for any errors.

Prerequisites:
- Run [`swift package-registry login`](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0378-package-registry-auth.md#new-login-subcommand) to authenticate registry user if needed. 
- The user has the necessary permissions to call the ["create a package release" API](https://github.com/apple/swift-package-manager/blob/main/Documentation/PackageRegistry/Registry.md#endpoint-6) for the package identifier.

### Changes to the registry service specification

#### Create package release API

A registry must update [this existing endpoint](https://github.com/apple/swift-package-manager/blob/main/Documentation/PackageRegistry/Registry.md#endpoint-6) to handle package release 
metadata as described in a [previous section](#package-release-metadata) of this document.
  
If the package being published is signed, the client must identify the signature format
in the `X-Swift-Package-Signature-Format` HTTP request header so that the
server can process the signature accordingly.

Signatures of the source-archive and metadata are sent as part of the request body 
(`source-archive-signature` and `metadata-signature`, respectively):

```
PUT /mona/LinkedList?version=1.1.1 HTTP/1.1
Host: packages.example.com
Accept: application/vnd.swift.registry.v1+json
Content-Type: multipart/form-data;boundary="boundary"
Content-Length: 336
Expect: 100-continue
X-Swift-Package-Signature-Format: cms-1.0.0

--boundary
Content-Disposition: form-data; name="source-archive"
Content-Type: application/zip
Content-Length: 32
Content-Transfer-Encoding: base64

gHUFBgAAAAAAAAAAAAAAAAAAAAAAAA==

--boundary
Content-Disposition: form-data; name="source-archive-signature"
Content-Type: application/octet-stream
Content-Length: 88
Content-Transfer-Encoding: base64

l1TdTeIuGdNsO1FQ0ptD64F5nSSOsQ5WzhM6/7KsHRuLHfTsggnyIWr0DxMcBj5F40zfplwntXAgS0ynlqvlFw==

--boundary
Content-Disposition: form-data; name="metadata"
Content-Type: application/json
Content-Transfer-Encoding: quoted-printable
Content-Length: 25

{ "repositoryURLs": [] }

--boundary
Content-Disposition: form-data; name="metadata-signature"
Content-Type: application/octet-stream
Content-Length: 88
Content-Transfer-Encoding: base64

M6TdTeIuGdNsO1FQ0ptD64F5nSSOsQ5WzhM6/7KsHRuLHfTsggnyIWr0DxMcBj5F40zfplwntXAgS0ynlqvlFw==
```

#### Fetch package release metadata API

A registry may update [this existing endpoint](https://github.com/apple/swift-package-manager/blob/main/Documentation/PackageRegistry/Registry.md#endpoint-2) for the [metadata changes](#package-release-metadata)
described in this document.

If the package release is signed, the registry must include a `signing` JSON 
object in the response:

```json5
{
  "id": "mona.LinkedList",
  "version": "1.1.1",
  "resources": [
    {
      "name": "source-archive",
      "type": "application/zip",
      "checksum": "a2ac54cf25fbc1ad0028f03f0aa4b96833b83bb05a14e510892bb27dea4dc812",
      "signing": {
        "signatureBase64Encoded": "l1TdTeIuGdNsO1FQ0ptD64F5nSSOsQ5WzhM6/7KsHRuLHfTsggnyIWr0DxMcBj5F40zfplwntXAgS0ynlqvlFw==",
        "signatureFormat": "cms-1.0.0"
      }      
    }
  ],
  "metadata": { ... }
}
```

#### Download package source archive API

If a registry supports signing, it must update [this existing endpoint](https://github.com/apple/swift-package-manager/blob/main/Documentation/PackageRegistry/Registry.md#endpoint-4) 
to include the `X-Swift-Package-Signature-Format` and `X-Swift-Package-Signature` headers in
the HTTP response for a signed package source archive.

```
HTTP/1.1 200 OK
Accept-Ranges: bytes
Cache-Control: public, immutable
Content-Type: application/zip
Content-Disposition: attachment; filename="LinkedList-1.1.1.zip"
Content-Length: 2048
Content-Version: 1
Digest: sha-256=oqxUzyX7wa0AKPA/CqS5aDO4O7BaFOUQiSuyfepNyBI=
Link: <https://mirror-japanwest.example.com/mona-LinkedList-1.1.1.zip>; rel=duplicate; geo=jp; pri=10; type="application/zip"
X-Swift-Package-Signature-Format: cms-1.0.0
X-Swift-Package-Signature: l1TdTeIuGdNsO1FQ0ptD64F5nSSOsQ5WzhM6/7KsHRuLHfTsggnyIWr0DxMcBj5F40zfplwntXAgS0ynlqvlFw==
```

## Security

This proposal introduces the framework for package signing, allowing package
authors the ability to provide additional authenticity guarantees by signing their
source archives before publishing them to the registry. Package users will be
able to control the kind(s) of packages they trust by specifying a local validation
policy. This can include a trust on first use approach, or by validating against a
pre-configured set of trusted roots.

While this proposal introduces the package signature format, it does not validate
that a package is published by a specific entity. Instead, it validates that a package
is published by an entity who can obtain a signing certificate that meets the
requirements defined by the registry, which could be anybody. As such, it does
not provide any protection against malware, and it would be wrong to assumed that
signed packages can be trusted unconditionally.

In this proposal, package signing is primarily intended to provide additional security
controls for package registries. By requiring packages be signed, and by the
registry limiting what keys or identities are allowed to publish packages, a registry
can provide additional security in the event the package author's registry credentials
are compromised.

Although this proposal introduces policy controls for package users, they are limited
in scope, and do not yet allow SwiftPM to validate that multiple package versions are
from the same entity--recording signing identities for [TOFU](#local-tofu) provides some 
protection against a compromised registry, but it is not for all packages and 
[more work needs to be done](#local-signing-identity-checks) before it can be so. As such, SwiftPM continues to trust 
the registry to provide authentic packages and accurate information about the 
signature status of the package.

### Privacy implications of certificate revocation check

Revocation checking via OCSP implicitly discloses to the certificate 
authority and anyone on the network the packages that a user may be
downloading. If this is a concern, revocation check can be disabled
in [SwiftPM configuration](#swiftpm-configuration).

## Impact on existing packages

Current packages won't be affected by changes in this proposal.

## Alternatives considered

### Signing package source archive vs. manifest

A package manifest is a reference list of files that are present in the 
source archive. We considered an approach where SwiftPM would produce 
such manifest, sign the manifest instead of the source archive,
then create a new archive containing the source archive, manifest, and
signature file. This way the archive and its signature can be distributed
by the registry as a single file. 

However, given the potential complications with extracting files from the
archive and verifying manifest contents, moreover there is no restriction
that would require single-file download (i.e., SwiftPM can download the 
source archive and signature separately), we have decided to take the approach
covered in previous sections of this proposal. 

### Use key in certificate as signing identity for local publisher-level TOFU

We considered using the key in a certificate as signing identity for 
[local publisher-level TOFU](#local-tofu) (i.e., different versions of a package must 
have the same signing identity). However, since key can change easily 
(e.g., lost key, key rotation, etc.), all users of the package must reset data 
used for local TOFU each time or else TOFU check would fail, which can introduce 
significant overhead and confusion.

## Future directions

### Support encrypted private keys

Private keys are encrypted typically. SwiftPM commands that have private key
as input, such as `package sign` and `package-registry publish`, should support
reading encrypted private key. This could mean modifying the command to prompt
user for the passphrase if needed, and adding a `--private-key-passphrase` 
option to the command for non-interactive/automation use-cases.

### Auto-populate package release metadata

Parts of the [package release metadata](#package-release-metadata-standards) can be populated by SwiftPM using
information found in the package directory. The auto-generated metadata can 
serve as a default or starting point which package authors may optionally edit, 
and ensure every package release to have metadata.

### Support additional certificate revocation checks

SwiftPM may support alternative mechanisms to check revocation besides OCSP.

### Local signing identity checks

In the current proposal, signing identity is left for the package registry to define, implement, and
enforce at publication time. However, this requires SwiftPM to rely on the package
registry to correctly implement these checks, and a compromise of the registry, or
SwiftPM's connection to the registry, would potentially allow for unauthorized packages
to be published. Performing additional checks in SwiftPM can mitigate this risk, but
requires defining a consistent identity that can be extracted and relied upon, and
determining how those identities are provisioned and authorized. 

A future Swift evolution proposal can provide specification of a certificate 
from which signing identity can be extracted, such that more certificates can 
be used for [local publisher-level TOFU](#local-tofu), which provides an extra layer of
trust on top of checksum TOFU done at the package release level.

### Timestamping and Countersignatures

In the proposed implementation, the signing certificate associated with
the package may expire, and this can prevent SwiftPM from validating
the information and revocation status of the certificate. Using
approaches such as [Time Stamping Authority](https://www.rfc-editor.org/rfc/rfc3161) or having the registry 
itself perform a [countersignature](https://www.rfc-editor.org/rfc/rfc5652#section-11.4), information about when a 
package was first published can be provided,
even after the signing certificate has expired. This can avoid the need
for package authors to re-sign packages when the signing certificate
expires.

### Transitive trust

SwiftPM's TOFU mitigation could be further improved by 
including checksum and signing identity in `Package.resolved` 
(or another similar file), which then gets included in the package content. 
Including such security metadata would allow distributing information about 
direct and transitive dependencies across the ecosystem much faster than a 
local-only TOFU without requiring a centralized database/service to vend 
this information.

```json5
{
  "pins": [
    {
      "identity": "mona.LinkedList",
      "kind": "registry",
      "location": "https://packages.example.com/mona/LinkedList",
      "state": {
        "checksum": "ed008d5af44c1d0ea0e3668033cae9b695235f18b1a99240b7cf0f3d9559a30d",
        "version": "0.12.0"
      },
      "signingBy": {
        "identityType": <STRING>,
        "name": <STRING>,
        ...
      }
    },
    {
      "identity": "Foo",
      "kind": "remoteSourceControl",
      "location": "https://github.com/something/Foo.git",
      "state": {
        "revision": "90a9574276f0fd17f02f58979423c3fd4d73b59e",
        "version": "1.0.2",
      }
    }    
  ],
  "version": 2
}
```
