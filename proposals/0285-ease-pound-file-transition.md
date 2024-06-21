# Ease the transition to concise magic file strings

* Proposal: [SE-0285](0285-ease-pound-file-transition.md)
* Author: [Becca Royal-Gordon](https://github.com/beccadax)
* Review Manager: [Tom Doron](https://github.com/tomerd)
* Implementation: [apple/swift#32700](https://github.com/apple/swift/pull/32700)
* Status: **Implemented (Swift 5.3)**
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-se-0285-ease-the-transition-to-concise-magic-file-strings/38516)

## Introduction

In [SE-0274][], the core team accepted a proposal to change the behavior of `#file`. This proposal modifies that plan to transition into new behavior more gradually, treating it as a source break requiring a new language version mode to fully adopt.  

Swift-evolution thread: [Revisiting the source compatibility impact of SE-0274: Concise magic file names](https://forums.swift.org/t/revisiting-the-source-compatibility-impact-of-se-0274-concise-magic-file-names/37720/)

  [SE-0274]: https://github.com/swiftlang/swift-evolution/blob/master/proposals/0274-magic-file.md

## Motivation

SE-0274 made the following changes:

1. Introduced a new `#filePath` magic identifier which, like `#file` in prior versions of Swift, evaluates to the source file path passed when the compiler was invoked.

2. Modified the `#file` magic identifier to instead evaluate to a string of the form `ModuleName/FileName.swift`. (See SE-0274 for discussions of the motivations for this.)
   
3. Added a warning when, essentially, a wrapper around a function with a `#filePath` default argument inadvertently captured `#file` instead, or vice versa.

After acceptance, changes 1 and 3 were enabled in the release/5.3 branch. Change 2 was gated by a staging flag and was expected to be enabled in the release after 5.3.

On paper, this seemed like a workable plan, giving developers the entire Swift 5.3 cycle to update code like this:

```swift
/// Prints the path to the file it was called from.
func printSourcePath(file: String = #file) { print(file) }
```

To this when necessary:

```swift
/// Prints the path to the file it was called from.
func printSourcePath(file: String = #filePath) { print(file) }
```

However, many source-distributed libraries like SwiftNIO need more than one version of backwards compatibility in use sites which will need to transition to `#filePath`. Their authors had believed they could achieve this using `#if compiler(>=5.3)` directives, but when they began to implement these workarounds, they found they were stymied by language limitations.

For instance, they believed they would be able to use a helper function to abstract the choice of `#file` or `#filePath`, not realizing that the magic call-site behavior of `#file` and `#filePath` in default arguments isn't transitive:

```swift
/// Prints the path to the file it was called from.
/// - Warning: Actually prints the path to the file `printSourcePath(file:)` was
///            defined in instead!
func printSourcePath(file: String = filePathOrFile()) { print(file) }

#if compiler(>=5.3)
    func filePathOrFile(_ value: String = #filePath) -> String { value }
#else 
    func filePathOrFile(_ value: String = #file) -> String { value }
#endif
```

They also believed they could use `#if` in default arguments, not realizing that `#if` directives can only be wrapped around whole statements or declarations:

```swift
/// Prints the path to the file it was called from.
/// - Warning: Actually fails to compile with a syntax error!
func printSourcePath(file: String = #if compiler(>=5.3)
                                        #filePath
                                    #else
                                        #file
                                    #endif) { print(file) }
```

That left them with with only one reasonable option—conditionally-selected wrapper functions—which they understandably considered too burdensome:

```swift
fileprivate func _printSourcePathImpl(file: String) { print(file) }

#if compiler(>=5.3)
/// Prints the path to the file it was called from.
func printSourcePath(file: String = #filePath) { _printSourcePathImpl(file: file) }
#else
/// Prints the path to the file it was called from.
func printSourcePath(file: String = #file) { _printSourcePathImpl(file: file) }
#endif
```

While we still want to reach the same endpoint—most code using a syntax with concise file strings while full paths are still available to the use sites that need them—it has become clear that this transition needs to be more gradual, with `#file` continuing to have its current meaning in Swift 5 projects indefinitely.

## Proposed solution

We propose to modify all three aspects of SE-0274's changes:

1. In addition to `#filePath`, we will introduce a new `#fileID` magic identifier, which generates the new concise file string in all language modes.

2. `#file` will continue to produce the same string as `#filePath` in the Swift 5 and earlier language modes. When code is compiled using future language modes (e.g. a hypothetical "Swift 6 mode"[1]), `#file` will generate the same string as `#fileID`, and `#fileID` will be deprecated.

3. In Swift 5 mode, for the purposes of the "wrong magic identifier literal" warning, `#file` will be treated as compatible with *both* `#filePath` and `#fileID`. In future language modes, it will be treated as compatible with only `#fileID`.

<details>

<summary>[1]</summary>

> "Swift 6" is purely illustrative. There is no guarantee that Swift 6 will be a source-breaking version, that there won't be a source-breaking version before Swift 6, or indeed that there will ever be a Swift 6 at all. Offer void where prohibited.

</details>

## Detailed design

### The `#fileID` magic identifier literal

In addition to the existing `#file` and the new `#filePath`, Swift will support `#fileID`. This new magic identifier will generate the same module-and-filename string SE-0274 specified for `#file`. It will do so immediately, without any way to reverse its behavior.

Standard library assertion functions like `assert`, `precondition`, and `fatalError` will switch from `#file` to `#fileID`. When a filename is included in a compiler-generated trap, such as a force unwrap, it will also emit a literal equivalent to using `#fileID`.

`#fileID` is intended to allow Swift 5.3 code to adopt the new, more compact literals before the behavior of `#file` changes. In language version modes where `#file` produces the same string as `#fileID`, `#fileID` will be deprecated.

### Transitioning the `#file` magic identifier

The `#file` identifier will continue to generate the same string as `#filePath` in Swift 4, 4.2, and 5 modes. In any future language version modes, it will instead generate the `#fileID` string. This means that the change in `#file`'s behavior—and therefore the requirement to change some uses of `#file` to `#filePath`—will be delayed until Swift next makes source-breaking changes, and even then, the current behavior will be preserved in Swift 5 mode.

If a client compiled in Swift 5 mode or earlier uses a library compiled with a later language mode, the library's `#file` default arguments will be treated as though they were `#fileID`; if a client compiled with a later language mode uses a library compiled in Swift 5 mode or earlier, the library's `#file` default arguments will be treated as though they were `#filePath`.

### Magic identifier default argument mismatch warnings

Swift will not warn when:

* A parameter with a default argument of `#file` is passed to one with a default argument of `#fileID`.
* A parameter with a default argument of `#fileID` is passed to one with a default argument of `#file`.

Additionally, Swift 5 mode and earlier will not warn when:

* A parameter with a default argument of `#file` is passed to one with a default argument of `#filePath`.
* A parameter with a default argument of `#filePath` is passed to one with a default argument of `#file`.

In Swift 5 mode, substituting `#file` for `#fileID` *will* actually result in different behavior, so ideally we would warn about this. However, doing that would cause new warnings in existing functions which wrap functions that adopt `#fileID`. In practice, most `#fileID` adopters will work fine when they're passed `#file`—they'll just generate unnecessarily large file strings. 

### Swift API Design Guidelines amendment

This guideline will be added to [the "Parameters" section][api-params] of the Swift API Design Guidelines:

> * **If your API will run in production, prefer `#fileID`** over alternatives.
>   `#fileID` saves space and protects developers’ privacy. Use `#filePath` in
>   APIs that are never run by end users (such as test helpers and scripts) if
>   the full path will simplify development workflows or be used for file I/O.
>   Use `#file` to preserve source compatibility with Swift 5.2 or earlier.

  [api-params]: https://swift.org/documentation/api-design-guidelines/#parameter-names

### Schedule

Ideally, these changes will be included in Swift 5.3, with the "future language mode" parts tied to a frontend flag for testing, rather than a language version mode.

## Source compatibility

Significantly improved compared to SE-0274. Code that uses `#filePath` or `#fileID` will not be compilable in Swift 5.2 and earlier, but such code can continue to use `#file` until a future language version mode permits breaking changes.

## Effect on ABI stability

See SE-0274.

## Effect on API resilience

See SE-0274.

## Alternatives considered

The SE-0274 design was considered and accepted, but did not hold up to real-world use.

In future language version modes, we could deprecate `#file` instead of `#fileID`, creating a situation where users must choose whether to update their uses of `#file` to either `#fileID` or `#filePath`. This would prompt future users to make a choice of behavior, but we don't want to force those users to make an explicit choice; we continue to believe that the concise file string (i.e. `#fileID` string) is the right choice for most uses.

Similarly, we could simply introduce `#fileID` and change the standard library to use it, without adding `#filePath` or changing `#file`'s behavior. But since we think the `#fileID` string is the right default, this would leave us with a design that encouraged the wrong defaults.

We could remove the `#fileID` syntax from this proposal, probably retaining it in underscored form as an implementation detail used to transition the standard library assertions to the new strings in Swift 5 mode. This would give us a cleaner "Swift 6 mode" without a deprecated `#fileID` hanging around, but we don't want to keep potential adopters outside the standard library from gaining the advantages of the new strings if they want them.

There are many colors other than `#fileID` that we could use to paint this bikeshed; we like this spelling because it's shorter than `#filePath` and suitably vague about the exact format of the string it produces.
