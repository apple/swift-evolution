# Automatic Mutable Pointer Conversion

* Proposal: [SE-NNNN](NNNN-immutable-mutables.md)
* Authors: [John Holdsworth](https://github.com/johnno1962)
* Review Manager: TBD
* Status: **Awaiting pitch**

*During the review process, add the following fields as needed:*

* Implementation: [apple/swift#37214](https://github.com/apple/swift/pull/37214)
* Decision Notes: [Rationale](https://forums.swift.org/), [Additional Commentary](https://forums.swift.org/)
* Bugs: [SR-14511](https://bugs.swift.org/browse/SR-14511)
* Previous Revision: [1](https://github.com/apple/swift-evolution/blob/...commit-ID.../proposals/NNNN-filename.md)
* Previous Proposal: [SE-XXXX](XXXX-filename.md)

## Introduction

This proposal adds automatic conversion from mutable unsafe pointers to their immutable counterparts, reducing unnecessary casting when feeding mutable memory to APIs assuming immutable access. This is most common in, but not exclusive to, C-sourced APIs.

Swift-evolution thread: [Pitch: Automatic Mutable Pointer Conversion](https://forums.swift.org/t/automatic-mutable-pointer-conversion/49304/)

## Motivation

In C, you may pass mutable pointers (specifically, `void *`, `Type *`) to calls expecting immutable pointers (`const void *`, `const Type *`). This access is safe and conventional as immutable access to a pointer's memory can be safely assumed when you have mutable access. The same reasoning holds true for Swift but no such implicit cast to immutable counterparts exists. Instead, you must explicitly cast mutable unsafe pointer types (`UnsafeMutableRawPointer` and `UnsafeMutablePointer<Type>`) to immutable versions (`UnsafeRawPointer` and `UnsafePointer<Type>`). This adds unneeded friction when passing mutable pointers to C functions or comparing pointers of differing mutability.

This friction is most commonly encountered when working with C-sourced APIS, where pointer mutability is not always consistent. Consider the following code, which segments lines in a String using the `strchr()` function, which accepts an immutable pointer as an input and returns a mutable pointer:

```Swift
print("""
    line one
    line two
    line three

    """.withCString {
    bytes in
    var bytes = bytes // immutable
    var out = [String]()
    while let nextNewline = // mutable
        strchr(bytes, Int32(UInt8(ascii: "\n"))) {
        out.append(String(data:
            Data(bytes: UnsafePointer<Int8>(bytes),
                 count: UnsafePointer<Int8>(
                    nextNewline)-bytes),
                          encoding: .utf8) ?? "")
        bytes = UnsafePointer<Int8>(nextNewline) + 1
    }
    return out
})
```

In the preceding example, an unfortunate choice on the part of C API mutability requires a cascade of Swift language conversions. While this example slightly pushes the issue — it would be better to convert the pointer once and earlier — the conversions shouldn't really be necessary at all. Safe interaction between the two languages should be fluid, with minimal overhead for what seems to be unnecessary type safety bookkeeping. Swift has a history of allowing language tuning to reduce exactly this kind of friction.

## Proposed solution

This proposal introduces a one-direction automatic conversion from mutable raw- and pointee-typed pointers to their immutable counterparts, allowing developers to supply a mutable pointer wherever an immutable pointer is expected as an argument. Consequently, it will also allow mixed mutability in pointer comparisons and pointer arithmetic via their overloaded operators.

## Detailed design

We have prepared a [small PR](https://github.com/apple/swift/pull/37214) on the Swift compiler. This patch adds a "fix-up" applied after type checking using the existing intrinsic pointer-to-pointer conversion. This patch should not slow down type checking, the most commonly cited reservation for conversions. In our initial tests with compiler benchmarks, we've found an 8% improvement in run-time performance when initializing `String` from `Data`.

## Source compatibility

This change is purely additive and will facilitate writing simpler code that would previously not compile. The change does not invalidate existing code, as tested by running the source compatibility suite.

## Effect on ABI stability

Not applicable, this is source level change.

## Effect on API resilience

Not applicable, this is source level change.

## Alternatives considered

Continuing to have to apply conversions in code.

## Acknowledgments

The Swift language.
