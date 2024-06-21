# Relax diagnostics for pointer arguments to C functions

* Proposal: [SE-0324](0324-c-lang-pointer-arg-conversion.md)
* Authors: [Andrew Trick](https://github.com/atrick), [Pavel Yaskevich](https://github.com/xedin)
* Review Manager: [Saleem Abdulrasool](https://github.com/compnerd)
* Status: **Implemented (Swift 5.6)**
* Implementation: [apple/swift#37956](https://github.com/apple/swift/pull/37956)
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-se-0324-relax-diagnostics-for-pointer-arguments-to-c-functions/52599)
* Bugs: [SR-10246](https://bugs.swift.org/browse/SR-10246)

## Introduction

C has special rules for pointer aliasing, for example allowing `char *` to alias other pointer types, and allowing pointers to signed and unsigned types to alias. The usability of some C APIs relies on the ability to easily cast pointers within the boundaries of those rules. Swift generally disallows typed pointer conversion. See [SE-0107 UnsafeRawPointer API](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0107-unsaferawpointer.md). Teaching the Swift compiler to allow pointer conversion within the rules of C when invoking functions imported from C headers will dramatically improve interoperability with no negative impact on type safety.

Swift-evolution thread: [Pitch: Implicit Pointer Conversion for C Interoperability](https://forums.swift.org/t/pitch-implicit-pointer-conversion-for-c-interoperability/51129)

## Motivation

Swift exposes untyped, contiguous byte sequences using `UnsafeRawPointer`. This completely bypasses thorny strict aliasing rules when encoding and decoding byte streams. However, Swift programmers often need to call into low-level C functions to help implement the encoding. Those C functions commonly expect a `char *` pointer rather than a `void *` pointer to the contiguous bytes. Swift does not allow raw pointers to be passed as typed pointers because it can easily introduce undefined behavior.

Calling a C function from Swift that takes a byte sequence as a typed pointer currently requires confusing, ugly, and likely incorrect workarounds. Swift programmers typically reach for `UnsafeRawPointer`'s "memory binding" APIs. Either `bindMemory(to:capacity:)` or `assumingMemoryBound(to:)`. We regularly see reports from programmers who were blocked while attempting a seemingly trivial task and needed to reach out to Swift experts to understand how to call a simple C API.

Memory binding APIs were never intended for regular Swift programming. Any use of them outside of low-level libraries is a usability bug. Furthermore, observing how the memory binding APIs are commonly used to workaround compiler errors reveals that they are often used incorrectly. And sometimes there is no correct alternative short of copying memory. Swift's model for typed memory was designed to be completely verifiable with a runtime sanitizer. When such a sanitizer is deployed, many of these workarounds will again raise an issue.

Consider using Foundation's `OutputStream.write` API. The programmer's initial attempt will look like this:

    func write(messageData: Data, output: OutputStream) -> Int {
      return messageData.withUnsafeBytes { rawBuffer in
        guard let rawPointer = rawBuffer.baseAddress else { return 0 }
        return output.write(rawPointer, maxLength: rawBuffer.count)
      }
    }
    
The compiler issues an unhelpful error:

    error: cannot convert value of type 'UnsafeRawPointer' to expected argument type 'UnsafePointer<UInt8>'
    
There's no way to make the diagnostic helpful because there's no way to make this conversion generally safe. A determined programmer will eventually figure out how to defeat the compiler's type check by arbitrarily picking either `bindMemory` or `assumingMemoryRebound`, both of which require global understanding of how `messageData`'s memory is used to be correct. Now the code may look like this, or worse:
     
    func write(messageData: Data, output: OutputStream) -> Int {
      return messageData.withUnsafeBytes { rawBuffer in
        guard let rawPointer = rawBuffer.baseAddress else { return 0 }
        let bufferPointer = rawPointer.assumingMemoryBound(to: UInt8.self)
        return output.write(bufferPointer, maxLength: rawBuffer.count)
      }
    }

This problem crops up regularly in compression and cryptographic APIs. You can see a couple examples from CommonCrypto in the forums: [CryptoKit: SHA256 much much slower than CryptoSwift](https://forums.swift.org/t/cryptokit-sha256-much-much-slower-than-cryptoswift/27983/12), and [withUnsafeBytes Data API confusion](https://forums.swift.org/t/withunsafebytes-data-api-confusion/22142/10)

As a generalization of this problem, consider a toy example:

*encrypt.h*

    #include <stddef.h>
     
    struct DigestWrapper {
      unsigned char digest[20];
    };
     
    int computeDigest(unsigned char *output,
                      const unsigned char *input,
                      size_t length);

It should be possible to call `computeDigest` from Swift as follows:

*encrypt.swift*

    func makeDigest(data: Data, wrapper: inout DigestWrapper) -> Int32 {
        data.withUnsafeBytes { inBytes in
            withUnsafeMutableBytes(of: &wrapper.digest) { outBytes in
                computeDigest(outBytes.baseAddress, inBytes.baseAddress,
                              inBytes.count)
            }
        }
    }

Without implicit conversion we need to write something like this instead:

    func makeDigest(data: Data, wrapper: inout DigestWrapper) -> Int32 {
        data.withUnsafeBytes { inBytes in
            withUnsafeMutableBytes(of: &wrapper.digest) { outBytes in
                let inPointer =
                    inBytes.baseAddress?.assumingMemoryBound(to: UInt8.self)
                let outPointer =
                    outBytes.baseAddress?.assumingMemoryBound(to: UInt8.self)
                return computeDigest(outPointer, inPointer, inBytes.count)
            }
        }
    }

In some cases, a typed Swift pointer, rather than a raw pointer must be converted to `char *`. It is always safe to construct a raw pointer from a typed pointer, so the same implicit conversion to C arguments should work for both `UnsafePointer<T>` and `UnsafeRawPointer`. A common use case involves a sequence of characters stored a buffer of any type other than CChar that needs to be passed to a C helper that takes `char *`. The character data may reside in an imported tuple (of any element type) or in a Swift array of UInt8 serving as a byte buffer.

The implicit conversion issue isn't limited to `char *`. It also comes up when APIs expect signed/unsigned pointer conversion. This has been a problem in practice for Swift programmers calling the mach kernel's task_info API. Wherever the compiler C language's special aliasing rules apply, they should all apply consistently.

The problematic cases that are documented in bug reports and forum posts are just a very small sampling of the issues that we've been made aware of both from direct communication with programmers and by searching Swift code bases for suspicious uses of "bind memory" APIs.

## Proposed solution

For imported C functions, allow implicit pointer conversion between pointer types that are allowed to alias according to C language rules:

1. A raw or typed unsafe pointer, `Unsafe[Mutable]RawPointer` or
   `Unsafe[Mutable]Pointer<T1>`, will be convertible to a typed
   pointer, `Unsafe[Mutable]Pointer<T2>`, whenever `T2` is
   `[U]Int8`. This allows conversion to any pointer type declared in C
   as `[signed|unsigned] char *`.

2. A typed unsafe pointer, `Unsafe[Mutable]Pointer<T1>`, will be
   convertible to `Unsafe[Mutable]Pointer<T2>` whenever `T1` and `T2`
   are integers that differ only in their signedness.

The conversions automatically apply to any function imported by the compiler frontend that handles the C family of languages. As a consequence, a Swift programmer's initial attempt to call a C, Objective-C, or C++ function will just work in most cases. See the above Motivation section for examples.

This solution does not affect type safety because the C compiler must already assume pointers of either type may alias.

Note that implicit conversion to a `const` pointer type was implemented when unsafe pointers were introduced. The new conversions extend the existing design. In fact, this extension was anticipated when raw pointers were introduced, but the implementation was deferred until developers had experience using raw pointers.

This solution does not cover C APIs that take function pointers. However, that case is much less common. For function pointer based APIs, its more appropriate to provide a Swift shim around the C API to encapsulate both the workaround for converting the pointer type and the function pointer handling in general.

## Detailed design

Implementation of this feature is based on the constraint restriction mechanism also used for other implicit conversions such as pointer/optional conversions. It introduces a new `PointerToCPointer` restriction kind which is only applied in argument positions when call is referencing an C/ObjC imported declaration and argument is either `Unsafe[Mutable]RawPointer` or `Unsafe[Mutable]Pointer<T>` and parameter is a pointer type or an optional (however deep) type wrapping a pointer.

To support new conversion in interaction with optional types e.g.  `UnsafeRawPointer` -> `UnsafePointer<UInt8>?` new restriction won't be recorded until there are other restrictions left to try (e.g. value-to-optional or optional-to-optional conversions), doing so makes sure that optional promotion or unwrap happens before new implicit conversion is considered.

Note that only conversions between typed signed and unsigned integral
pointers are commutative, conversions from raw pointers are more
restrictive:

|Actual Swift Argument|Parameter Imported from C|Is Commutative|
---|---|---
|`UnsafeRawPointer`|`UnsafePointer<[U]Int8>`|No|
|`UnsafeMutableRawPointer`|`Unsafe[Mutable]Pointer<[U]Int8>`|No|
|`UnsafePointer<T>`|`UnsafePointer<[U]Int8>`|No|
|`UnsafeMutablePointer<T>`|`Unsafe[Mutable]Pointer<[U]Int8>`|No|
|`UnsafePointer<Int8>`|`UnsafePointer<UInt8>`|Yes|
|`UnsafePointer<Int16>`|`UnsafePointer<UInt16>`|Yes|
|`UnsafePointer<Int32>`|`UnsafePointer<UInt32>`|Yes|
|`UnsafePointer<Int64>`|`UnsafePointer<UInt64>`|Yes|
|`UnsafeMutablePointer<Int8>`|`Unsafe[Mutable]Pointer<UInt8>`|Yes|
|`UnsafeMutablePointer<Int16>`|`Unsafe[Mutable]Pointer<UInt16>`|Yes|
|`UnsafeMutablePointer<Int32>`|`Unsafe[Mutable]Pointer<UInt32>`|Yes|
|`UnsafeMutablePointer<Int64>`|`Unsafe[Mutable]Pointer<UInt64>`|Yes|


## Source compatibility

No effect.

In general, adding implicit conversions is not source compatible. But this proposal only adds implicit conversions for function argument types that would already cause an override conflict had they both been part of an overridden function declared in C. Since the new implicit conversions are only applied to functions imported from C, this change cannot introduce any new override conflicts.

## Effect on ABI stability

Not applicable. Pointer conversion is entirely handled on the caller side.

## Effect on API resilience

Not applicable. Pointer conversion is entirely handled on the caller side.

## Alternatives considered

*Use C shims to make C APIs more raw-pointer-friendly.* In SwiftNIO, the pointer conversion problem was prevalent enough that it made sense to introduce a replacement C APIs taking `void *` instead of `char *`. For example: https://github.com/apple/swift-nio/blob/nio-1.14/Sources/CNIOHTTPParser/include/CNIOHTTPParser.h#L22-L27 This is not an obvious workaround, and it it impractical for most developers to introduce shims in their project for C APIs.

*Rely on C APIs to be replaced or wrapped with Swift shims.* The rate at which programmers run into this interoperability problem is speeding up, not slowing down. Swift continues to be adopted in situations that require interoperability. There are a large number of bespoke C APIs that won't be replaced by Swift APIs in the foreseeable future. If the existing C API is wrapped with a Swift shim, then that only hides the incorrect memory binding workaround rather than fixing it.

*Add more implicit conversions to Swift.* This would introduce C's legacy pointer aliasing rules into the Swift language. Swift's model for type pointer aliasing should remain simple and robust. Special case aliasing rules that happen to work for common cases are deeply misleading. They introduce complexity in the language definition, implementation, and tooling. These special cases are unnecessary and undesirable for well-designed Swift APIs. Implicit type punning introduces more opportunities for bugs. Special aliasing rules would also penalize performance of pure Swift code. Finally, this would not be a source-compatible change.

*Introduce `UnsafeRawPointer.withMemoryRebound(to:capacity:)`.* This is a generally useful, although somewhat unsafe API. We also plan to introduce this API, but it isn't a sufficient fix for C interoperability. It only provides yet another ugly and confusing workaround alternative.

## Acknowledgments

Thank you to all the patient Swift programmers who have struggled with C interoperability and shared their experience with the Swift team.

Thanks to @eskimo, @lukasa, @jrose, @karl, and @itaiferber for helping those programmers use unsafe pointers while waiting for the language and libraries to be improved.
