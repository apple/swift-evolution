# Demangle Function

* Proposal: [SE-0262](0262-demangle.md)
* Author: [Alejandro Alonso](https://github.com/Azoy), [Tony Arnold](https://github.com/tonyarnold)
* Review Manager: [Joe Groff](https://github.com/jckarter)
* Status: **Returned for revision**
* Implementation: [apple/swift#25314](https://github.com/apple/swift/pull/25314)
* Decision Notes: [Returned for revision](https://forums.swift.org/t/returned-for-revision-se-0262-demangle-function/28186)

## Introduction

Introduce a new standard library function, `demangle`, that takes a mangled Swift symbol such as `$sSS7cStringSSSPys4Int8VG_tcfC`, and --- if it can --- outputs the human readable Swift symbol, like `Swift.String.init(cString: Swift.UnsafePointer<Swift.Int8>) -> Swift.String`.

Swift-evolution thread: [Demangle Function](https://forums.swift.org/t/demangle-function/25416)

## Motivation

Currently, if a user is given an unreadable mangled symbol, they're most likely to use the `swift-demangle` tool to get the demangled version. However, this is awkward when you want to demangle a symbol in-process in Swift: one could create a new `Process` from Foundation and set it up to launch a new process within the current process to use `swift-demangle`, but the standard library can do this more easily, and without the intermediary steps. 

## Proposed solution

The standard library will add the following new enumeration and function:

```swift
/// Represents the demangler function output style.
public enum DemangledOutputStyle {
  /// Includes module names and implicit self types.
  case full
  /// Excludes module names and implicit self types.
  case simplified
}

/// Given a mangled Swift symbol, return the demangled symbol. Defaults to the simplified style used by LLDB, Instruments and similar tools. 
public func demangle(
  _ input: String, 
  outputStyle: DemangledOutputStyle = .simplified
) -> String?
```

Examples:

```swift
print(demangle("$s8Demangle3FooV")!) // Foo

print(demangle("$s8Demangle3FooV", outputStyle: .full)!) // Demangle.Foo
```

## Detailed design

If one were to pass a string that wasn't a valid Swift mangled symbol, like `abc123`, then the function will return `nil` to indicate failure.

This implementation relies on the Swift runtime function `swift_demangle` which accepts symbols that start with `_T`, `_T0`, `$S`, and `$s`.

The `outputStyle` parameter of the `demangle(…)` function accepts one of two potential cases:
- `full`: this is equivalent to the output of `swift-demangle`
- `simplified`: this is equivalent to the output of `swift-demangle --simplified`

## Source compatibility

These are completely new standard library functions, thus source compatibility is unaffected.

## Effect on ABI stability

These are completely new standard library functions, thus ABI compatibility is unaffected.

## Effect on API resilience

These are completely new standard library functions, thus API resilience is unaffected.

## Alternatives considered

Earlier versions of this proposal included additional functions that supported demangling in limited runtime contexts using unsafe buffer-based APIs:

```swift
public func demangle(
  _ mangledNameBuffer: UnsafeBufferPointer<Int8>,
  into buffer: UnsafeMutableBufferPointer<Int8>
) -> DemangleResult

public func demangle(
  _ input: String,
  into buffer: UnsafeMutableBufferPointer<Int8>
) -> DemangleResult
```

Unfortunately, the current demangler implementation is not suitable for such applications, because even if it were given a preallocated output buffer for the returned string, it still freely allocates in the course of parsing the mangling and forming the parse tree for it. Presenting an API that might seem safe for use in contexts that can't allocate would be misleading.

This alternative could be considered under “Future Directions” as well, if/when the underlying implementation is made suitable for this purpose. 

Discussion on the forums also raised the concern of polluting the global namespace, and the suggestion was made to create a new “Runtime” module to house this function (and potentially others). The Core Team thought that the proposed demangle function makes sense as a standalone, top-level function, however it would be a natural candidate for inclusion in such a module if it existed.

## Future Directions

The `swift_demangle` runtime function has an extra `flags` parameter, but currently it is not being used for anything. In the future if that function ever supports any flags, it would make sense to introduce new overloads or something similar to expose those flags to the standard library as well. E.g.

```swift
public func demangle(_ input: String, flags: DemangleFlags) -> String?

public func demangle(
  _ mangledNameBuffer: UnsafeBufferPointer<Int8>,
  into buffer: UnsafeMutableBufferPointer<Int8>,
  flags: DemangleFlags
) -> DemangleResult

public func demangle(
  _ input: String,
  into buffer: UnsafeMutableBufferPointer<Int8>,
  flags: DemangleFlags
) -> DemangleResult
```

where `DemangleFlags` could be an enum, `OptionSet`, `[DemangleFlag]`, etc.
