# Pitch: Formalize ‘language mode’ terminology

* Proposal: [SE-NNNN](nnnn-formalize-language-mode-terminology.md)
* Author: [James Dempsey](https://github.com/dempseyatgithub)
* Review Manager: TBD
* Status: **Awaiting review**
* Implementation: [apple/swift-package-manager#7620](https://github.com/apple/swift-package-manager/pull/7620)
* Review: ([first pitch](https://forums.swift.org/t/pitch-formalize-swift-language-mode-naming-in-tools-and-api/71733)), ([second pitch](https://forums.swift.org/t/pitch-2-formalize-language-mode-naming-in-tools-and-api/72136))

## Introduction
The term "Swift version” can refer to either the toolchain/compiler version or the language mode. This ambiguity is a consistent source of confusion. This proposal formalizes the term _language mode_ in tool options and APIs.

## Proposed Solution
The proposed solution is to use the term _language mode_ for the appropriate Swift compiler option and Swift Package Manager APIs. Use of "Swift version" to refer to language mode will be deprecated or obsoleted as needed.

### Terminology
The term _language mode_ has been consistently used to describe this compiler feature since it was introduced with Swift 4.0 and is an established term of art in the Swift community.

The **Alternatives Considered** section contains a more detailed discussion of the term's history and usage.

### Swift compiler option
Introduce a `-language-mode` option that has the same behavior as the existing `-swift-version` option, while de-emphasizing the `-swift-version` option in help and documentation.

#### Naming note
The proposed compiler option uses the term 'language mode' instead of 'Swift language mode' because the context of the usage strongly implies a Swift language mode. The intent is that the `languageMode()` compiler condition described in **Future directions** would also use that naming convention for the same reason.

### Swift Package Manager
Introduce four Swift Package Manager API changes limited to manifests \>= 6.0:

#### 1. A new Package init method that uses the language mode terminology

```swift
@available(_PackageDescription, introduced: 6)
Package(
	name: String,
	defaultLocalization: [LanguageTag]? = nil.
	platforms: [SupportedPlatform]? = nil,
	products: [Product] = [],
	dependencies: [Package.Dependency] = [],
	targets: [Target] = [],
	swiftLanguageModes: [SwiftLanguageMode]? = nil,
	cLanguageStandard: CLanguageStandard? = nil,
	cxxLanguageStandard: CXXLanguageStandard? = nil
)
```

Add a new `init` method to `Package` with the following changes from the current `init` method:

- The paramater `swiftLanguageVersions` is renamed to `swiftLanguageModes`
- The parameter type is now an optional array of `SwiftLanguageMode` values instead of `SwiftVersion` values

The existing init method will be marked as obsolete and renamed allowing the compiler to provide a fix-it.

#### 2. Rename `swiftLanguageVersions` property to `swiftLanguageModes`
Rename the public `Package` property `swiftLanguageVersions` to `swiftLanguageModes`. Add a `swiftLanguageVersions` computed property that accesses `swiftLanguageModes` for backwards compatibility. 


#### 3. Rename `SwiftVersion` enum to `SwiftLanguageMode`
Rename the `SwiftVersion` enum to `SwiftLanguageMode`. Add `SwiftVersion` as a type alias for backwards compatibility.


#### 4. Update API accepted in [SE-0435: Swift Language Version Per Target](https://github.com/apple/swift-evolution/blob/main/proposals/0435-swiftpm-per-target-swift-language-version-setting.md):

```swift
public struct SwiftSetting {
  // ... other settings
  
  @available(_PackageDescription, introduced: 6.0)
  public static func swiftLanguageMode(
	  _ mode: SwiftLanguageMode,
	  _ condition: BuildSettingCondition? = nil
 )
```

If both proposals are implemented in the same release, the accepted SE-0435 API would be added with the proposed naming change.

#### Naming note

In Swift PM manifests, multiple languages are supported. For clarity, there is existing precedent for parameter and enum type names to have a language name prefix.

For example the Package `init` method currently includes:

```swift
	...
	swiftLanguageVersions: [SwiftVersion]? = nil,
	cLanguageStandard: CLanguageStandard? = nil,
	cxxLanguageStandard: CXXLanguageStandard? = nil
	...
```

For clarity and to follow the existing precedent, the proposed Swift PM APIs will be appropriately capitalized versions of "Swift language mode".

## Detailed design

### New swift compiler option
A new `-language-mode` option will be added with the same behavior as the existing `-swift-version` option.

The `-swift-version` option will continue to work as it currently does, preserving backwards compatibility.

The `-language-mode` option will be presented in the compiler help.

The `-swift-version` option will likely be supressed from the top-level help of the compiler. More investigation is needed on the details of this.

### Swift Package Manager
Proposed Swift Package Manager API changes are limited to manifests \>= 6.0:

### New Package init method and obsoleted init method
A new `init` method will be added to `Package` that renames the `swiftLanguageVersions` parameter to `swiftLanguageModes` with the type of the parameter being an optional array of `SwiftLanguageMode` values instead of `SwiftVersion` values:

```swift
@available(_PackageDescription, introduced: 6)
Package(
	name: String,
	defaultLocalization: [LanguageTag]? = nil.
	platforms: [SupportedPlatform]? = nil,
	products: [Product] = [],
	dependencies: [Package.Dependency] = [],
	targets: [Target] = [],
	swiftLanguageModes: [SwiftLanguageMode]? = nil,
	cLanguageStandard: CLanguageStandard? = nil,
	cxxLanguageStandard: CXXLanguageStandard? = nil
)
```


The existing init method will be marked as obsoleted and renamed, allowing the compiler to provide a fix-it:

```
@available(_PackageDescription, introduced: 5.3, obsoleted: 6, renamed:
"init(name:defaultLocalization:platforms:pkgConfig:providers:products:
dependencies:targets:swiftLanguageModes:cLanguageStandard:
cxxLanguageStandard:)")
	public init(
		name: String,
		...
		swiftLanguageVersions: [SwiftVersion]? = nil,
		cLanguageStandard: CLanguageStandard? = nil,
		cxxLanguageStandard: CXXLanguageStandard? = nil
	) {
```

#### Obsoleting existing init method
The existing method must be obsoleted because the two methods are ambiguous when the default value for `swiftLanguageVersions` / `swiftLanguageModes` is used:

```
Package (  // Error: Ambiguous use of 'init'
  name: "MyPackage",
  products: ...,
  targets: ...
)
```

This follows the same approach used by all past revisions of the Package `init` method.

See the **Source compatibiity** section for more details about this change.

### Rename `swiftLanguageVersions` property to `swiftLanguageModes`
Rename the `Package` public property `swiftLanguageVersions` to `swiftLanguageModes`. Introduce a computed property named `swiftLanguageModes` that accesses the renamed stored property for backwards compatibility.

The computed property will be annotated as obsoleted in Swift 6, renamed to `swiftLanguageModes`.

For packages with swift tools version less than 6.0, accessing the `swiftLanguageModes` property will continue to work.  
For 6.0 and later, that access will be an error with a fix-it to use the new property name.

```swift
	@available(_PackageDescription, obsoleted: 6, renamed: "swiftLanguageModes")
	public var swiftLanguageVersions: [SwiftVersion]? {
		get { swiftLanguageModes }
		set { swiftLanguageModes = newValue }
	}
```

See the **Source compatibiity** section for more details about this change.

### Rename `SwiftVersion` enum to `SwiftLanguageMode`
Rename the existing `SwiftVersion` enum to `SwiftLanguageMode` with `SwiftVersion` added back as a type alias for backwards compatibility.

This change will not affect serializaiton of PackageDescription types. Serialization is handled by converting PackageDescription types into separate, corresponding Codable types. The existing serialization types will remain as-is.


### Rename API added in SE-0435
The API in the newly-accepted [SE-0435](https://github.com/apple/swift-evolution/blob/main/proposals/0435-swiftpm-per-target-swift-language-version-setting.md) will change to use the _language mode_ terminology:

```swift
public struct SwiftSetting {
  // ... other settings
  
  @available(_PackageDescription, introduced: 6.0)
  public static func swiftLanguageMode(
	  _ mode: SwiftLanguageMode,
	  _ condition: BuildSettingCondition? = nil
 )
```

The name of the function is `swiftLanguageMode()` instead of `languageMode()` to keep naming consistent with the `swiftLanguageModes` parameter of the Package init method. The parameter label `mode` is used to follow the precedent set by `interoperabilityMode()` in `SwiftSetting`.

## Source compatibility
The new Package `init` method and obsoleting of the existing `init` method will cause source breakage for package manifests that specify the existing `swiftLanguageVersions` parameter when updating to swift tools version 6.0

A search of manifest files in public repositories suggests that about 10% of manifest files will encounter this breakage.

Because the obsoleted `init` method is annotated as `renamed` the compiler will automatically provide a fix-it to update to the new `init` method.

Renaming the public `swiftLanguageVersions` property of `Package` preserves backwards compatibility by introducing a computed property with that name. Because this proposal already contains a necessary breaking change as detailed above, the computed property will also be marked as obsoleted in 6.0 and annotated as `renamed` to provide a fix-it.

Searching manifest files in public repositories suggests that accessing the `swiftLanguageVersions` property directly is not common. Making both breaking changes at once results in applying at most two fix-its to a manifest file instead of one.

## ABI compatibility
This proposal has no effect on ABI stability.

## Future directions
This proposal originally included the proposed addition of a `languageMode()` compilation condition to further standardize on the terminology and allow the compiler to check for valid language mode values.

That functionality has been removed from this proposal with the intent to pitch it seperately. Doing so keeps this proposal focused on the tools, including the source breaking API changes. The future direction is purely additive and would focus on the language change. 

## Alternatives considered

### Alternate terminology

In the pitch phase, a number of terms were suggested as alternatives for _language mode_. Some concerns were also expressed that the term _language mode_ may be too broad and cause future ambiguity.

The intent of this proposal is to formalize established terminology in tool options and APIs.

The term _language mode_ is a long-established term of art in the Swift community to describe this functionality in the compiler.

This includes the [blog post](https://www.swift.org/blog/swift-4.0-released/) annoucing the functionality as part of the release of Swift 4 in 2017 (emphasis added):

> With Swift 4, you may not need to modify your code to use the new version of the compiler. The compiler supports two _language modes_…

> The _language mode_ is specified to the compiler by the -swift-version flag, which is automatically handled by the Swift Package Manager and Xcode.
>
> One advantage of these _language modes_ is that you can start using the new Swift 4 compiler and migrate fully to Swift 4 at your own pace, taking advantage of new Swift 4 features, one module at a time.

Usage also includes posts in the last year from LSG members about Swift 6 language mode:

- [Design Priorities for the Swift 6 Language Mode](https://forums.swift.org/t/design-priorities-for-the-swift-6-language-mode/62408/27)
- [Progress toward the Swift 6 language mode](https://forums.swift.org/t/progress-toward-the-swift-6-language-mode/68315)

Finally, searching for "language modes" and "language mode" in the Swift forums found that at least 90% of the posts use the term in this way. Many of the remaining posts use the term in the context of Clang.

#### Alternatives mentioned

Alternate terms raised as possibilities were:
  - _Edition_: a term used by Rust for a similar concept    
  - _Standard_: similar to C or C++ standards  
	Language standards tend to be associated with a written specification, which Swift does not currently have.  
	Using the term _standard_ would preclude using the term in the future to describe a formal standard.


#### Potential overload of _language mode_
Some reviewers raised concern that Embedded Swift could be considered a language mode and lead to future ambiguity.

On consideration, this concern is mitigated in two ways:

1. As noted above, the term _language mode_ is a well-established term of art in the Swift community.

2. The term _Embedded Swift_ already provides an unambiguous, concise name that can be discussed without requiring a reference to modes.  
   
   This is demonstrated by the following hypothetical FAQ based on the Embedded Swift vision document:
>   _What is Embedded Swift?_  
>   Embedded Swift is a subset of Swift suitable for restricted environments such as embedded and low-level environments.
> 
>   _How do you enable Embedded Swift?_  
>   Pass the `-embedded` compiler flag to compile Embedded Swift.

Considering these alternatives, it seems likely that introducing a new term to replace the long-established term _language mode_ and potentially giving the existing term a new meaning would lead to more ambiguity than keeping and formalizing the existing meaning of _language mode_.