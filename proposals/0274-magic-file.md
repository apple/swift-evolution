# Concise magic file names

* Proposal: [SE-0274](0274-magic-file.md)
* Authors: [Becca Royal-Gordon](https://github.com/beccadax), [Dave DeLong](https://github.com/davedelong)
* Review Manager: [Ben Cohen](https://github.com/airspeedswift/)
* Status: **Implemented (Swift 5.8)**
* Upcoming Feature Flag: `ConciseMagicFile` (Enabled in Swift 6 language mode)
* Decision Notes: [Review #1](https://forums.swift.org/t/se-0274-concise-magic-file-names/32373/50), [Review #2](https://forums.swift.org/t/re-review-se-0274-concise-magic-file-names/33171/11), [Additional Commentary](https://forums.swift.org/t/revisiting-the-source-compatibility-impact-of-se-0274-concise-magic-file-names/37720)
* Next Proposal: [SE-0285](0285-ease-pound-file-transition.md)

## Introduction

Today, `#file` evaluates to a string literal containing the full path to the current source file. We propose to instead have it evaluate to a human-readable string containing the filename and module name, while preserving the existing behavior in a new `#filePath` expression.

Swift-evolution thread: [Concise Magic File Names](https://forums.swift.org/t/concise-magic-file-names/31297), [We need `#fileName`](https://forums.swift.org/t/we-need-filename/19781)

## Changes since the first review

* We now specify that the `#file` string will have the format `<module-name>/<file-name>` (with a second form for future expansion) and discuss how to parse it. The previous revision left this format as a compiler implementation detail that tools should always treat as opaque.

* We now discuss the behavior of `#sourceLocation`'s `file` parameter and provide for a warning when it creates conflicts. The previous revision did not discuss these topics.

* We now mention the need for tooling to map `#file` strings back to paths.

* We now provide for a warning when a wrapper around a `#filePath`-defaulting function passes it `#file` instead, or vice versa. The previous revision did not discuss this.

* We have added several suggestions from the first review to the "alternatives considered" section to explain why we aren't proposing them.

## Motivation

In Swift today, the magic identifier `#file` evaluates to a string literal containing the full path (that is, the path passed to `swiftc`) to the current file. It's a nice way to trace the location of logic occurring in a Swift process, but its use of a full path has a lot of drawbacks:

* It can inadvertently reveal private or sensitive information. The full path to a source file may contain a developer's username, hints about the configuration of a build farm, proprietary versions or identifiers, or the Sailor Scout you named an external disk after. Developers probably don't know that this information is embedded in their binaries and may not want it to be there. And most uses of `#file` are in default arguments, which makes this information capture invisible at the use site. The information leaks here are quite serious; if other languages hadn't already normalized this, I doubt we would think that `#file` was an acceptable design.

* It bloats binaries produced by the Swift compiler. In testing with the Swift benchmark suite, a shorter `#file` string reduced code size by up to 5%. The large code also impacts runtime performance—in the same tests, a couple dozen benchmarks ran noticeably faster, with several taking 22% less time. While we didn't benchmark app launch times, it's likely that they are also adversely affected by lengthy `#file` strings.

* It introduces artificial differences between binaries built on different machines. For instance, the same code built in two different environments might produce different binaries with different hashes. This makes life difficult for anyone trying to cache build artifacts, find the differences between two binaries, or otherwise hoping to benefit from reproducible builds.

Meanwhile, the benefits you might expect from using a full path aren't really there:

* The paths are not portable between machines, so you can't use it to automatically match an error to a line of source code unless your machine has the same directory layout as the machine that built the binary.

* The paths are not guaranteed to be absolute—XCBuild and SwiftPM happen to use absolute paths, but other build systems like Bazel don't—so you can't be sure that you can resolve paths to a file on disk.
  
* The paths are not even guaranteed to be unique within a process because the same file could be included in several different modules, or different projects could be located at the same paths on different machines or at different times.

For `#file`'s most important use case, it seems like the current string computed by `#file` is suboptimal and a better default would improve Swift in several ways.

## Proposed solution

We propose changing the string that `#file` evaluates to—instead of evaluating to the full path, it will now have the format `<module-name>/<file-name>`. For those applications which still need a full path, we will provide a new magic identifier, `#filePath`. Both of these features will otherwise behave the same as the old `#file`, including capturing the call site location when used in default arguments. The standard library's assertion and error functions will continue to use `#file`.

With this proposal, a file at `/Users/becca/Desktop/0274-magic-file.swift` in a module named `MagicFile` with this content:

```swift
print(#file)
print(#filePath)
fatalError("Something bad happened!")
```

Would produce this output:

```text
MagicFile/0274-magic-file.swift
/Users/becca/Desktop/0274-magic-file.swift
Fatal error: Something bad happened!: file MagicFile/0274-magic-file.swift, line 3
```

The Swift compiler does not currently allow two files with the same name (regardless of path) to be compiled into the same module.[1] This means that `<module-name>/<file-name>` does a better job of uniquely identifying a file than the full path without taking up nearly as much space. It is also easy for a human to read and map back to a source file; tools can automatically map these strings back to paths if they know which files are in each module, which compiler tooling could help them to determine.

> [1] This limitation ensures that identically-named `private` and `fileprivate` declarations in different files will have unique mangled names. A future version of the Swift compiler could lift this rule.

## Detailed design

### Specification of the `#file` string format

The formal grammar of the string generated by `#file` is:

```text
file-string → module-name "/"  file-name
file-string → module-name "/" disambiguator "/" file-name   // Reserved for future use

module-name → [same as Swift identifier]
disambiguator → [all characters except U+0000 NULL]
file-name → [all characters except U+002F SOLIDUS and U+0000 NULL]
```

`"/"` means `"/"` in this grammar, even on systems with a different path separator.

Note that *disambiguator* may contain `"/"` characters. To parse *file-string* correctly, developers should split the string on `"/"` characters and use the first element for the module name, or the last element for the file name, without assuming that there are exactly two elements. Alternatively, they may use the index of the first `"/"` as the end of the module name and the index after the last `"/"` as the start of the file name.

### Computation of *file-string* components

The components of *file-string* are currently computed from the module name and `#filePath` at the source location. Specifically:

* *module-name* is the name of the module being compiled.

* *file-name* is the substring of the `#filePath` string falling after the last path separator. "Path separator" here is either `"/"` or the host's path separator character (e.g. `"\"` on Windows).

* *disambiguator* is currently always omitted.

Future compilers may also use the other potential `#filePath` strings in the module as an input to this function and use them to compute a *disambiguator* field to distinguish between `#filePath`s that would otherwise produce the same *file-string*. We do not currently specify how they will do this; we simply constrain future compilers to fit this information into the *disambiguator* field.

### Interaction with `#sourceLocation`

The `file` parameter of a `#sourceLocation` directive will specify the content of `#filePath`. Since *file-string* is calculated from the `#filePath`, this means that the last component of the `file` parameter will be used as the *file-name*. We do not provide a way to change the *module-name* or to directly specify the *file-string*.

In current compilers, a `#sourceLocation` directive may produce `#file` strings which collide with the `#file` strings produced by other `#sourceLocation`-introduced paths or physical files. The compiler will warn about this. Future compilers may use the *disambiguator* field to differentiate them instead.

### Tooling

SourceKit will provide facilities for mapping `#file` strings back to their matching `#filePath`s.

### Default argument mismatch diagnostics

To help users who are wrapping a `#file` or `#filePath`-defaulting function avoid using the wrong default argument in their wrappers, the compiler will emit a warning when a parameter which captures one of the magic identifiers is passed to a default argument which captures a different one. This warning will also apply to existing magic identifiers, like `#file` vs. `#function` or `#line` vs. `#column`.

We do not specify the exact diagnostics in this proposal, but as an illustration, the mismatch in this code:

```swift
func fn1(file: String = #filePath) { ... }
func fn2(file: String = #file) {
    fn1(file: file)
}
```

Might be diagnosed like:

```
sample.swift:3: warning: parameter 'file' with default argument '#file' passed to parameter 'file', whose default argument is '#filePath'
    fn1(file: file)
              ^~~~
sample.swift:2: note: did you mean for parameter 'file' to default to '#filePath'?
func fn2(file: String = #file) {
                        ^~~~~
                        #filePath
sample.swift:3: note: add parentheses to silence this warning
    fn1(file: file)
              ^   ^
              (   )
```

### Implementation status

A prototype of this feature is already in master; it can be enabled by passing `-Xfrontend -enable-experimental-concise-pound-file` to the Swift compiler. This prototype does not include some of the revisions for the second review:

* The specified `#file` string format (the prototype uses `file-name (module-name)` instead).
* The warning for colliding `#sourceLocation`s.
* Proof-of-concept for tooling support, in the form of a table printed into comments in `-emit-sil` output.

These are implemented in the unmerged [apple/swift#29412](https://github.com/apple/swift/pull/29412).

## Source compatibility

All existing source code will continue to compile with this change, and `#file`'s documentation never specified precisely what its contents were; in one of the pitch threads, [Ben Cohen](https://forums.swift.org/t/concise-magic-file-names/31297/19) said that this is sufficient to satisfy Swift's source compatibility requirements. However, the proposal *will* cause behavior to change in existing code, and in some cases it will change in ways that cause existing code to behave incorrectly when run. Code that is adversely affected by this change can access the previous behavior by using `#filePath` instead of `#file`.

The change to the behavior of `#file` was deferred to the next major language version (Swift 6). However, it can be enabled with the [upcoming feature flag](0362-piecemeal-future-features.md) `ConciseMagicFile`.

## Effect on ABI stability

None. `#file` is a compile-time feature; existing binaries will continue to work as they did before.

## Effect on API resilience

As with any addition to Swift's syntax, older compilers won't be able to use module interface files that have `#filePath` in default arguments or inlinable code. Otherwise, none.

## Alternatives considered

### Deprecate `#file` and introduce two new syntaxes

Rather than changing the meaning of `#file`, we could keep its existing behavior, deprecate it, and provide two alternatives:

* `#filePath` would continue to use the full path.
* `#fileName` would use the new concise string suggested by this proposal.

This is a more conservative approach that would avoid breaking any existing uses. We choose not to propose it for three reasons:

1. The name `#fileName` is misleading because it sounds like the string only contains the file name, but it also contains the module name. `#file` is more vague, so we're more comfortable saying that it's "a string that identifies the file".
  
2. This alternative will force developers to update every use of `#file` to one or the other option. We feel this is burdensome and unnecessary given how much more frequently the `#fileName` behavior would be appropriate.

3. This alternative gives users no guidance on which feature developers ought to use. We feel that giving `#file` a shorter name gives them a soft push towards using it when they can, while resorting to `#filePath` only when necessary.

4. Since all uses of `#file`—not just ones that require a full path—would change, all uses of `#file` in module interfaces—not just ones that require a full path—would become stumbling blocks for backwards compatibility. This includes uses in the swiftinterface files for the standard library and XCTest.

However, if the core team feels that changing `#file`'s behavior will cause unacceptable behavior changes, this ready-made alternative would accomplish most of the goals of this proposal.

### Support more than two `#file` variants

We considered introducing additional `#file`-like features to generate other strings, selecting between them either with a compiler flag or with different magic identifiers. The full set of behaviors we considered included:

1. Path as written in the compiler invocation
2. Guaranteed-absolute path
3. Path relative to the Xcode `SOURCE_DIR` value, or some equivalent
4. Last component of the path (file name only)
5. File name plus module name
6. Empty string (sensible as a compiler flag)

We ultimately decided that supporting only 1 (as `#filePath`) and 5 (as `#file`) would adequately cover the use cases for `#file`. Five different syntaxes would devote a lot of language surface area to a small niche, and controlling the behavior with a compiler flag would create six language dialects that might break some code. Some of these behaviors would also require introducing new concepts into the compiler or would cause trouble for distributed build systems.

### Make `#filePath` always absolute

While we're looking at this area of the language, we could change `#filePath` to always generate an absolute path. This would make `#filePath` more stable and useful, but it would cause problems for distributed builds unless it respected `-debug-prefix-map` or something similar. It would also mean that there'd be no simple way to get the *exact* same behavior as Swift previously provided, which would make it more difficult to adapt code to this change.

Ultimately, we think changing to an absolute path is severable from this proposal and that, if we want to do this, we should consider it separately.

### Provide separate `#moduleName` and `#fileName` magic identifiers

Rather than combining the module and file names into a single string, we could provide separate ways to retrieve each of them. There are several reasons we chose not to do this:

* `#fileName` is so ambiguous between modules that good uses for it by itself are few and far between. You really need either the module name or the path to tell which file by that name you're talking about. The full path can be so verbose that it's better to truncate to the filename and deal with the ambiguity, but module names are short enough that module name + filename doesn't really have this problem.

* Existing clients, like `fatalError(_:file:line:)`, have only one parameter for filename information. Adding a second would break ABI, and there would not be a way to concatenate the caller's `#moduleName` and `#fileName` into a single string in a default argument. (Magic identifiers only give you the caller's location if they are the only thing in the default argument; something like `file: String = "\(#moduleName)/\(#fileName)"` would capture the callee's location instead.)

* `#file` provides a standard format for this information so that different tools are all on the same page.

While we don't see many practical uses for `#fileName`, we *do* think there are reasonable uses for `#moduleName`. However, this information can now be easily parsed out of the `#file` string. If we want to also provide a separate `#moduleName`, we can consider that in a separate proposal.

### Other alternatives

We considered leaving the *file-string* unspecified rather than specifying it with an unused *disambiguator* field for future expansion. Feedback from the first review convinced us that clients will want to interpret this string, which requires a specified format.

We considered renaming the `file` parameter of `#sourceLocation` to `filePath`. This would make its meaning clearer, but it seems like unnecessary churn, and `#sourceLocation` is begging for a redesign to address its other shortcomings anyway.

We considered adding a new magic identifier like `#context` which represents several pieces of contextual information simultaneously. This would not give us most of the privacy and code size improvements we seek because, when passed across optimization barriers like module boundaries, we would have to conservatively generate all of the information the callee might want to retrieve from `#context`.

We considered making `#file`'s behavior change conditional on enabling a new language version mode. While we're not opposed to that in principle, we don't think the breakage from this change will be severe enough to justify delaying this proposal's implementation until the next release with a language version mode.

We considered introducing a new alternative to `#file` (e.g. `#fileName`) while preserving the existing meaning of `#file`. However, a great deal of code already uses `#file` and would in practice probably never be converted to `#fileName`. Most of this code would benefit from the new behavior, so we think it would be better to automatically adopt it. (Note that clang supports a `__FILE_NAME__` alternative, but most code still uses `__FILE__` anyway.)

We considered switching between the old and new `#file` behavior with a compiler flag. However, this creates a language dialect, and compiler flags are not a natural interface for users.

Finally, we could change the behavior of `#file` without offering an escape hatch. However, we think that the existing behavior is useful in rare circumstances and should not be totally removed.
