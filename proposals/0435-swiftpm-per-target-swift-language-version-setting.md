# Swift Language Version Per Target

* Proposal: [SE-0435](0435-swiftpm-per-target-swift-language-version-setting.md)
* Authors: [Pavel Yaskevich](https://github.com/xedin)
* Review Manager: [Becca Royal-Gordon](https://github.com/beccadax)
* Status: **Implemented (Swift 6.0)**
* Review: ([pitch](https://forums.swift.org/t/pitch-swiftpm-swift-language-version-per-target/71067)) ([review](https://forums.swift.org/t/se-0435-swift-language-version-per-target/71546)) ([acceptance](https://forums.swift.org/t/accepted-se-0435-swift-language-version-per-target/71846))

## Introduction

The current Swift Package Manager manifest API for specifying Swift language version(s) applies to an entire package which is limiting when adopting new language versions that have implications for source compatibility.

Swift-evolution thread: [Pitch: [SwiftPM] Swift Language Version Per Target](https://forums.swift.org/t/pitch-swiftpm-swift-language-version-per-target/71067)

## Motivation

Adopting new language versions at the target granularity allows for gradual migration to prevent possible disruptions. Swift 6, for example, turns on strict concurrency by default, which can have major implications for the project in the form of new errors that were previously downgraded to warnings. SwiftPM should allow to specify a language version per target so that package authors can incrementally transition their project to the newer version.

## Proposed solution

Add a new Swift target setting API, similar to `enable{Upcoming, Experimental}Feature`, to specify a Swift language version that should be used to build the target, if such version is not specified, fallback to the current language version determination logic.

## Detailed design

Add a new `swiftLanguageVersion` API to `SwiftSetting` limited to manifests >= 6.0:

```swift
public struct SwiftSetting {
  // ... other settings
  
  @available(_PackageDescription, introduced: 6.0)
  public static func swiftLanguageVersion(
      _ version: SwiftVersion,
      _ condition: BuildSettingCondition? = nil
  ) -> SwiftSetting {
     ...
  }
}
```

## Security

New language version setting has no implications on security, safety or privacy.

## Impact on existing packages

Since this is a new API, all existing packages will use the default behavior - version specified at the package level when set or determined based on the tools version.

## Alternatives considered

- Add a new setting for 'known-safe' flags, of which `-swift-version` could be the first. This seems less user-friendly and error prone than re-using `SwiftLanguageVersion`, which has known language versions as its cases (with a plaintext escape hatch when necessary).

- Add new initializer parameter to `Target` API and all of the convenience static functions, this is less flexible because it would require a default value.

