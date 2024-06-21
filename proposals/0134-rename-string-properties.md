# Rename two UTF8-related properties on String

* Proposal: [SE-0134](0134-rename-string-properties.md)
* Authors: [Xiaodi Wu](https://github.com/xwu), [Erica Sadun](https://github.com/erica)
* Review Manager: [Chris Lattner](https://github.com/lattner)
* Status: **Implemented (Swift 3.0)**
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-with-revision-se-0134-rename-two-utf8-related-properties-on-string/3576)
* Implementation: [apple/swift#3816](https://github.com/apple/swift/pull/3816)
* Previous Revision: [1](https://github.com/swiftlang/swift-evolution/blob/aea8b836d21051076663c5692ec1d09bb3222527/proposals/0134-rename-string-properties.md)

## Introduction

This proposal removes `nulTerminatedUTF8` and renames `nulTerminatedUTF8CString` to enhance clarity and reduce mismatch between user expectations and the Swift programming language.

Swift-evolution thread: [Discussion thread](https://forums.swift.org/t/draft-fix-a-typo-in-two-string-methods/3524)

## Motivation

Both `nulTerminatedUTF8` and `nulTerminatedUTF8CString` are poorly named for the following reasons:
 
* **Inappropriate abbreviation of a term of art**: The ASCII names for \0 are "null character" or "null terminator". Both properties instead use the ASCII abbreviation "NUL" in place of the English word "null". A Google search returned approximately 20,000 results for "NUL-terminated string" and approximately 200,000 results for "null-terminated string".
* **Impaired recognition**: "NUL" is less recognizable than "null". Further, "NUL" suffers from reduced recognition when written in lowercase: `nulTerminated`.
* **Hindered source completion**: When using source editor completion, users who type "null" will not find a property named `nulTerminatedUTF8` or `nulTerminatedUTF8CString`.
* **Redundant terminology**: C strings are terminated by the null character. Using both "C string" and "null-terminated" is redundant and, to some, could unintentionally raise questions as to whether some C strings might not be null-terminated.

This proposal removes `nulTerminatedUTF8` and eliminates the redundancy in `nulTerminatedUTF8CString`.

## Detailed design
This proposal introduces the following changes to the Swift standard library:

* Rename `nulTerminatedUTF8CString` to `utf8CString`.
* Remove `nulTerminatedUTF8`.

#### `utf8CString`

This property renaming follows the precedent of the related Foundation method `cString(using: .utf8)` and lowercases its leading `utf8`.

#### `nulTerminatedUTF8`

This property is a null-terminated contiguous array of a string's UTF8 representation. The core team has indicated that clients would be better served by using the `utf8CString` property and has concluded that `nulTerminatedUTF8` should be removed outright.

## Impact on existing code

Fix-its will be needed to help transition existing code.

## Alternatives considered

The alternative is not to rename the stated properties.
