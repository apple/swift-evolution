# Adding `Darwin` as a supported OS type for `#if os`

* Proposal: [SE-NNNN](NNNN-ifOSDarwin.md)
* Authors: [Serena A.](https://github.com/SerenaKit)
* Review Manager: TBD
* Status: **Implementation complete, awaiting review**
* Implementation: [apple/swift#59426](https://github.com/apple/swift/pull/59426)

## Introduction

This proposal aims to makes it possible to make checking for Darwin platforms easier to read & write in code, by introducing a `Darwin` alias which can be used on `#if os` to check for Darwin platforms.

[Swift-evolution discussion thread](https://forums.swift.org/t/if-os-darwin-a-shorthand-for-checking-for-darwin-platforms/58146)

## Motivation

Currently, to check for Darwin platforms, the below is usually done:
```swift
func foo() {
    #if os(macOS) || os(iOS) || os(watchOS) || os(tvOS)
    // Code for Darwin platforms
    #else
    // Code for non Darwin platforms
    #endif
}
```

Not only does this end up being hard to read, it also requires extensive project-wide changes every time a new Darwin platform is released. For API that is unlikely to ever be removed–such as Foundation, the Objective-C runtime, or the C standard library–there is little benefit to listing out the supported platforms explicitly.

## Proposed solution

The proposed solution is simple, adding a new supported platform for `#if os`, named `Darwin`, to cover current and future Darwin platforms, ie:
```swift
func foo() {
    #if os(Darwin)
    // Code for Darwin platforms
    #else
    // Code for non Darwin platforms
    #endif
}
```

In addition to being easier to read, this also ensures that the user doesn't have to rewrite their `#if os` statements when a new Darwin platform is released.

## Detailed design

The design of the implementation consists of a new platform type which can be used in `#if os` statements, being an alias for existing Darwin OSes, which checks for the following platforms:
- OSX
- iOS
- tvOS
- watchOS

## Source compatibility

N/A. This is a additive change.

## Effect on ABI stability

N/A. No affect on ABI Stability.

## Effect on API resilience

N/A. Not an API.

## Alternatives considered

The current implementation was chosen as it makes sense to check the platform type using `#if os`, making it clear what is being checked.

Current, existing, alternatives include:
- Checking for `#if _runtime(_Objc)`
- Checking for `#if canImport(ObjectiveC)`
- Checking for `#if canImport(Darwin)`

Which, although work, aren't meant to be used for checking for Darwin platforms. ie, using `#if canImport(Darwin)` for code which doesn't use symbols from the Darwin framework (such as Darwin-only symbols from Foundation) feels like a clunky alternative and may be confusing.

As mentioned in the beginning of this proposal, there is also the existing, widely used method of checking for the 4 supported Darwin OSes individually (`#if os(macOS) || os(iOS) || os(tvOS) || os(watchOS)`), however is harder to read than a simple one check of `#if os(Darwin)`, and is not future-proof in case any new Darwin OSes which support the same code are released.
