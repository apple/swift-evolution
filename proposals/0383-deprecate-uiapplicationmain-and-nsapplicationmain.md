# Deprecate @UIApplicationMain and @NSApplicationMain

* Proposal: [SE-0383](0383-deprecate-uiapplicationmain-and-nsapplicationmain.md)
* Authors: [Robert Widmann](https://github.com/codafi)
* Review Manager: [John McCall](https://github.com/rjmccall)
* Status: **Implemented (Swift 5.10)**
* Upcoming Feature Flag: `DeprecateApplicationMain`
* Implementation: [PR 62151](https://github.com/apple/swift/pull/62151)
* Review: ([pitch](https://forums.swift.org/t/deprecate-uiapplicationmain-and-nsapplicationmain/61493)) ([review](https://forums.swift.org/t/se-0383-deprecate-uiapplicationmain-and-nsapplicationmain/62375)) ([acceptance](https://forums.swift.org/t/accepted-se-0383-deprecate-uiapplicationmain-and-nsapplicationmain/62645))

## Introduction

`@UIApplicationMain` and `@NSApplicationMain` used to be the standard way for
iOS and macOS apps respectively to declare a synthesized platform-specific
entrypoint for an app. These functions have since been obsoleted by
[SE-0281](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0281-main-attribute.md)'s
introduction of the `@main` attribute, and they now represent a confusing bit of
duplication in the language. This proposal seeks to deprecate these alternative
entrypoint attributes in favor of `@main` in pre-Swift 6, and it makes their use
in Swift 6 a hard error.

## Motivation

UIKit and AppKit have fully embraced the `@main` attribute and have made
adoption by applications as simple as conforming to the `UIApplicationDelegate`
and `NSApplicationDelegate` protocols. This now means that an author of an
application is presented with two different, but ultimately needless, choices
for an entrypoint:

* use one of the hard coded framework-specific attributes `@UIApplicationMain` or `@NSApplicationMain`, or
* use the more general `@main` attribute.

At runtime, the behavior of the `@main` attribute on classes that conform to
one of the application delegate protocols above is identical to the corresponding
framework-specific attribute. Having two functionally identical ways to express the
concept of an app-specific entrypoint is clutter at best and confusing at worst.
This proposal seeks to complete the migration work implied by
[SE-0281](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0281-main-attribute.md)
by having the compiler push Swift authors towards the more general, unified
solution.

## Proposed solution

Using either `@UIApplicationMain` and `@NSApplicationMain` in a pre-Swift 6
language mode will unconditionally warn and offer to replace these attributes
with the appropriate conformances. In Swift 6 language mode (and later), using
these attributes will result in a hard error.

## Detailed design

> Because `@UIApplicationMain` and `@NSApplicationMain` are used in identical
> ways, this portion of the document will only discuss `@UIApplicationMain`.
> The design for `@NSApplicationMain` follows the exact same pattern.

Framework-specific attributes were added to the language to automate the
boilerplate involved in declaring a standard application entrypoint. In UIKit
code, the entrypoint always ends with a call to `UIApplicationMain`. The last
parameter of this call is the name of a subclass of `UIApplicationDelegate`.
UIKit will search for and instantiate this delegate class so it can issue 
application lifecycle callbacks.  Swift, therefore, requires this attribute to 
appear on a class that conforms to the `UIApplicationDelegate` protocol so it 
can provide the name of that class to UIKit.

But a conformance to `UIApplicationDelegate` comes with more than just lifecycle
callbacks. A default implementation of a `main` entrypoint is [provided for
free](https://developer.apple.com/documentation/uikit/uiapplicationdelegate/3656306-main)
to a conforming type, but the `@UIApplicationMain` attribute suppresses it. This
fact is key to the migration path for existing users of the framework-specific
attribute.

Under this proposal, when the compiler sees a use of `@UIApplicationMain`, it
will emit a diagnostic including a suggestion to replace the attribute with
`@main`.  In Swift 6 and later language modes, this diagnostic will be an error;
otherwise it will be a warning.

```swift 
@UIApplicationMain // warning: '@UIApplicationMain' is deprecated in Swift 5 
                   // fixit: Change `@UIApplicationMain` to `@main` 
final class MyApplication: UIResponder, UIApplicationDelegate {
  /**/
}
```

Once the fixit has been applied, the result will be

```swift
@main
final class MyApplication: UIResponder, UIApplicationDelegate {
  /**/
}
```

This simple migration causes the compiler to select the `main` entrypoint
inherited by the conformance to `UIApplicationDelegate`. No further source
changes are required.

## Source compatibility

Current Swift libraries will continue to build because they compile under
pre-Swift 6 language modes. Under such language modes this proposal adds only an
unconditional warning when framework-specific entrypoints are used, and provides
diagnostics to avoid the warning by automatically migrating user code.

In Swift 6 and later modes, this proposal is intentionally source-breaking as the
compiler will issue an unconditional error upon encountering a framework-specific
attribute. This source break will occur primarily in older application code, as 
most libraries and packages do not use framework-specific attributes to define a main
entrypoint. Newer code, including templates for applications provided by Xcode 14
and later, already use the `@main` attribute. 

## Effect on ABI stability

This proposal has no impact on ABI.

## Effect on API resilience

None.
