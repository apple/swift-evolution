# Constrain implicit raw pointer conversion to bitwise-copyable values

* Proposal: [SE-NNNN](nnnn-implicit-raw-bitwise-conversion.md)
* Authors: [Andrew Trick](https://github.com/atrick)
* Review Manager: TBD
* Status: **Awaiting review**
* Implementation: [apple/swift#63825](https://github.com/apple/swift/pull/63825)
* Review: [[Pitch] Constrain implicit raw pointer conversion to bitwise-copyable values](https://forums.swift.org/t/pitch-constrain-implicit-raw-pointer-conversion-to-bitwise-copyable-values/63314)

## Introduction

This proposal adds restrictions on implicit casts to raw pointers, allowing them only when the source value is "bitwise copyable". A type is bitwise copyable if copying a value of that type requires nothing more than copying each bit in its representation (i.e. `memcpy`). Bitwise copyable types do not require deinitialization. Notably, bitwise copyable types cannot contain object references. Raw pointers are primarily intended for use with bitwise-copyable types because it is extremely difficult and dangerous to work with raw bytes in the presence of object references.

The implicit raw pointer casts that we want to prohibit happen accidentally, without any source-level indication of unsafety. They nonetheless pose a serious safety and security concern. They are a source of programming mistakes leading to runtime crashes but have no discernable benefit.

## Motivation

Swift supports implicit inout conversion to unsafe pointers, meaning that a mutable variable can be passed `inout` to a function that receives an unsafe pointer to the value. According to the normal rules for pointer conversion, this naturally extends to raw pointers. Raw pointers allow the pointer-taking function to operate directly on the value's bytes.

```swift
func readBytes(_ pointer: UnsafeRawPointer) {
  assert(pointer.load(as: UInt16.self) == 0xaaaa)
}
    
var x: UInt16 = 0xaaaa
readBytes(&x)
```
     
Another higher-level form of implicit pointer conversion allows Swift Array and String values to be passed directly to pointer-taking functions. Here, the callee receives a pointer to the object's contiguously stored elements, rather than to the object itself:

```swift
let array = [UInt16(0xaaaa)]
readBytes(array)
```

These two features are a dangerous combination. The Array and String conversion feature sets up an expectation that collection data types can, in general, be implicitly converted to their elements. When a programmer attempts to make use of this feature for non-Array, non-String data types, they can easily fall into the more general `inout` conversion behavior instead. Rather than creating a pointer to the collection's elements, the compiler instead exposes the data type's internal representation without warning.

Here, the user likely wants to inspect the contents of a string, but instead they've leaked the internal representation:

```swift
func inspectString(string: inout String) {
  readBytes(&string) // reads the string's internal representation
}
```

This is a pernicious security issue because the code will happen to work during testing for small strings. Removing the '&' sigil changes the string conversion into an array-like conversion:

```swift
func inspectString(string: inout String) {
  readBytes(string) // reads the string's characters
}
```

In the next example, the author clearly expected Foundation.Data to have the same sort of implicit conversion support as Array:

```swift
func foo(data: inout Data) {
  readBytes(&data)
}
```

This compiles without warning, but it unintentionally exposes data object's internal storage representation rather than its elements.

Swift 5.7 generalized the type checker's handling of raw pointers to include C functions that take `char *`. This makes the dangerous combination easier to stumble into:

```swift
void read_char(char *input);
read_char(&string) // wrong
```

The problem of accidentally casting String and Foundation.Data to a raw pointer is a dramatic example of a much broader problem. A type that contains no class references is referred to as bitwise copyable, because copying its value is equivalent to calling `memcpy`--the copy operations does not carry any type-specific semantics. Casting bitwise-copyable types to raw pointers is reasonable, and has always been supported. Implicit conversion from a non-bitwise-copyable type to a raw pointer, however, is extremely dangerous, and almost always accidental:

```swift
var object: AnyObject = ...
readBytes(&object)
```

Balancing safety with C interoperability usability is often difficult, but in this case we can achieve much greater safety without sacrificing much overall usability. Given the recent improvements to pointer interoperability in Swift 5.7, and with a major language update on the horizon, this good time to correct this behavior.

## Proposed Solution

We propose introducing a new diagnostic that warns on implicit inout conversion of a non-bitwise-copyable value to a raw pointer.

    warning: forming an 'UnsafeRawPointer' to a variable of type 'T'; this is likely incorrect because 'T' may contain an object reference.

Based on user feedback from this warning, we intend to convert the diagnostic to an error in Swift 6.

Concrete types in which the compiler has visibility into the members of the type are considered bitwise-copyable if all their members are bitwise-copyable. This is exactly the same condition under which the `_Trivial` layout constrain holds and the `_isPOD()` runtime check returns true. This includes concrete instantiations of generic types, such as `SIMD4<Float>`.

Unbound generic types, on the other hand, are conservatively considered non-bitwise-copyable, so the following example will produce a warning:

```swift
func inoutGeneric<T>(t: inout T) {
  readBytes(&t)  // warning: forming an 'UnsafeRawPointer' to a variable of type 'T' ...
}
```

To mitigate source breakage, conversion of generic types that conform to `FixedWidthInteger` will be allowed.

```swift
func inoutGeneric<T: FixedWidthInteger>(t: inout T) {
  readBytes(&t)  // no warning
}
```

This will handle common generic cases that rely on inout to raw pointer conversion, such as `Unicode.Encoding.CodeUnit` and swift-nio's `ByteBuffer` API.
It is safe to assume that fixed-width integers are bitwise-copyable.

As discussion in Future Directions, we plan to provide a `BitwiseCopyable` layout constraint to generalize the solution for generic types.

## Detailed Design

Given Swift declarations:

```swift
func readBytes(_ pointer: UnsafeRawPointer) {...}
func writeBytes(_ pointer: UnsafeMutableRawPointer) {...}
```

The new diagnostic warns on the following implicit casts:

```swift
// Let T be a statically non-bitwise-copyable type...

var t: T = ... 
readBytes(&t)
writeBytes(&t)

let constArray: [T] = ...
readBytes(constArray)

var array: [T] = ...
readBytes(&array)
writeBytes(&array)

var string: String = ...
readBytes(&string)
writeBytes(&string)

var data: Data = ...
readBytes(&data)
writeBytes(&data)
```

The warning for general types takes the form:

    warning: forming an 'UnsafeRawPointer' to a variable of type 'T'; this is likely incorrect because 'T' may contain an object reference.

The warning for Arrays takes the form:

    warning: forming an 'UnsafeRawPointer' to a variable of type '[T]'; this is likely incorrect because 'T' may contain an object reference.

The warning for Strings takes the form:

    warning: forming an 'UnsafeRawPointer' to an inout variable of type String exposes the internal representation rather than the string contents.

Implicit casts from a FixedWidthInteger will continue to be supported without a warning:

```swift
var int: some FixedWidthInteger = ...
readBytes(&int)
writeBytes(&int)

let constIntArray: [some FixedWidthInteger] = ...
readBytes(constIntArray)

var intArray: [some FixedWidthInteger] = ...
readBytes(&intArray)
writeBytes(&intArray)
```

Implicit casts from bitwise copyable collection elements, will continue to be supported without a warning:

```swift
var byteArray: [UInt8] = [0]
readBytes(&byteArray)
writeBytes(&byteArray)

let string: String = sarg
readBytes(string)
readUInt8(string)
```

Given the C declarations:

```c
void read_char(const char *input);
void read_uchar(const unsigned char *input);
void write_char(char *input);
void write_uchar(unsigned char *input);
```

Per [SE-0324: Relax diagnostics for pointer arguments to C functions](https://github.com/apple/swift-evolution/blob/main/proposals/0324-c-lang-pointer-arg-conversion.md), all of the above implicit casts will have the same behavior after substituting `read_char` or `read_uchar` in place of `readBytes`  after substituting `write_char` or `write_uchar` in place of `writeBytes`.

Implementation: [PR #63825](https://github.com/apple/swift/pull/63825).

## Source compatibility

In Swift 5, the proposal introduces a warning without breaking source.

In Swift 6, the warning may become a source-breaking error. This is conditional on whether the community is receptive to using the workarounds that we provide for the warning.

The new diagnostic will only warn on code that views the raw bytes of a class reference or a generic type. In most cases, the source code that will be diagnosed as warning was never actually supported, and in some cases was already undefined behavior. In all cases, a simple workaround is available.

### Workarounds for common cases

Users can silence the warning using an explicit conversion, such as `withUnsafePointer`, `withUnsafeMutablePointer`, `withUnsafeBytes`, `withUnsafeMutableBytes`, or `Unmanaged.toOpaque()` as follows. Note that these are all extremely dangerous use cases that are not generally supported, but they will work in practice under specific conditions. Those conditions are out of scope for this proposal, but they are the same regardless of whether the code relies on implicit conversion or uses the explicit conversions below...

To pass the address of an internal stored property through an opaque
pointer (unsupported but not uncommon):

```swift
// C declaration
// void take_opaque_pointer(void *);

class TestStoredProperty {
  var property: AnyObject? = nil

  func testPropertyId() {
    withUnsafePointer(to: &property) {
      take_opaque_pointer($0)
    }
  }
}
```
    
Note that `property` must be passed `inout` by using the `&` sigil. Simply calling `withUnsafePointer(to: property)` would create a temporary copy of `property`.

To pass a class reference through an opaque pointer:

```swift
// C decl
// void take_opaque_pointer(void *);

let object: AnyObject = ...
withExtendedLifetime (object) {
  take_opaque_pointer(Unmanaged.passUnretained(object).toOpaque());
}
```

To expose the bitwise representation of class references:

```swift
func readBytes(_ pointer: UnsafeRawPointer) {...}

withUnsafePointer(to: object) {
  readBytes($0)
}
```

The diagnostic message does not mention specific workarounds, such as `withUnsafeBytes(of:)`, because, although helpful for migration, that would push developers toward writing invalid code in the future. For example, if a user incorrectly tries to convert a user-defined collection directly to a raw pointer, a diagnostic that suggests `withUnsafeBytes(of:)` would encourage rewriting the code as follows:

```swift
withUnsafeBytes(of: &collection) {
  readBytes($0.baseAddress!)
}
```

This actually promotes the behavior that we're trying to prevent! Quite often, the programmer instead needs to reach for a method on a collection type, such as Data.withUnsafeBytes().
    
### Associated object String keys

Associated objects are strongly discouraged in Swift and may be deprecated. Nonetheless, legacy code can't always be redesigned at the time of a language update. We offer some quick workarounds here.

Code that attempts to take the address of a String as an associated object key will now raise a type conversion warning:

```swift
import Foundation
 
class Container {
  static var key = "key"

  func getObject() -> Any? {
    // warning: forming 'UnsafeRawPointer' to an inout variable of type String
    // exposes the internal representation rather than the string contents.
    return objc_getAssociatedObject(self, &Container.key)
  }
}
```

This can be rewritten using the direct, low-level `withUnsafePointer` workaround:

```swift
class Container {
  static var key = "key"

  func getObject() -> Any? {
    withUnsafePointer(to: &Container.key) {
      return objc_getAssociatedObject(self, $0)
    }
  }
}
```

Note that `Container.key` must be passed `inout` by using the `&` sigil. Simply calling `withUnsafePointer(to: Container.key)` would create a temporary copy of the key.

If you don't need the key to be a String, then you can simply use any bitwise-copyable type instead to avoid the conversion warning:

    private struct Container {
        static var key: Void? // "key"
    }
     
    // no warning here
    objc_setAssociatedObject(self, &Container.key, value)

Alternatively, you can use the key's object identity rather the address of the property. But this only works with objects that require separate allocation at instantiation. Neither NSString, nor NSNumber can be safely used. NSObject is a safe bet:

```swift
class Container {
  static var key = NSObject()

  func getID(_ object: AnyObject) -> UnsafeRawPointer {
    return UnsafeRawPointer(Unmanaged.passUnretained(object).toOpaque())
  }
  func getObject() -> Any? {
    return objc_getAssociatedObject(self, getID(Container.key))
  }
}
```

The object identity workaround above can be packaged in a property wrapper to avoid redefining `getID` for every key:

```swift
@propertyWrapper
public class UniqueAddress {
  public var wrappedValue: UnsafeRawPointer {
    return UnsafeRawPointer(Unmanaged.passUnretained(self).toOpaque())
  }

  public init() { }
}

class Container {
  @UniqueAddress static var key

  func getObject() -> Any? {
    return objc_getAssociatedObject(self, Container.key)
  }
}
```

## Effects on ABI stability

None. Restricting casts has no effect on the ABI.

## Effects on API resilience

Strictly speaking, restricting casts has not effect on resilience. Nonetheless, a resilient source change can invalidate code that converts a type to a raw pointer. As soon as code makes an assumption about the physical object layout of a type, the code must be adjusted regardless of whether the layout change was considered resilient or not.

In particular, if a library author adds a new non-bitwise-copyable stored property to the end of a frozen struct that was previously bitwise-copyable, an implicit raw pointer cast in another module will now be an error. This is a desirable outcome because the presence of the new object reference may make the existing implicit cast memory unsafe. If the author of the client code can prove that the reference won't be accessed via the raw pointer, then they can workaround the new diagnostic just by adding an explicit cast.

## Alternatives considered

### Narrow, type-specific diagnostics

We could add an attribute to Data and other copy-on-write containers to more selectively suppress implicit conversion. This approach would miss an opportunity to significantly improve safety of the language. As soon as `BitwiseCopyable` is available, we will use it to qualify other APIs that manipulate raw bytes. `BitwiseCopyable` is the type that we want programmers to associate with raw memory features going forward.

### Forbid implicit inout conversion to raw pointers

We could forbid all implicit inout conversion to raw pointers, except for direct calls to C functions. We do have precedent for allowing certain conversions only for C interoperability:
[SE-0324: Relax diagnostics for pointer arguments to C functions](https://github.com/apple/swift-evolution/blob/main/proposals/0324-c-lang-pointer-arg-conversion.md).

The most common uses involving C functions would still work:

```c
char c_buffer[10]; // C header
read_char(&c_buffer) // called from Swift code
```

This would, however, break legitimate uses of inout-to-raw-pointer conversion in Swift, forcing the use of a closure-taking API. There are already published examples of Swift-to-C interoperability that rely on this feature, similar to the example from the introduction:

```swift
var x: UInt16 = 0xaaaa
readBytes(&x)
```

We would need to ask users to migrate all these cases to:

```swift
var x: UInt16 = 0xaaaa
withUnsafeBytes(of: x) {
  readBytes($0)
}
```
    
While this may seem consistent with Swift's policy of explicitly requiring an "unsafe" API in cases that may lead to undefined behavior, the downsides outweigh that benefit:

- it creates a migration barrier to Swift 6

- it makes it impossible to write Swift shims on top of C APIs that
  have the same usability. This was not the case in the aforementioned
  SE-0324, because those Swift shims simply need to use the correct
  raw pointer type.

More importantly, *this solution does not actually address the cases that pose the most danger*. Misuses of implicit conversions usually do involve a  pointer-taking C function. In the case of `inout` conversion, the C function may get a pointer to the data structures control fields rather then its element. And in the case of constant pointers, the C function sometimes returns or otherwise escapes the same pointer.

### Forbid implicit Array and String conversion to unsafe pointers

Implicit conversion is generally dangerous because it allows interior pointers to escape, exposing undefined behavior and use-after-free security bugs. The most commonly misused cases are the special cases that were added for convenience: Arrays, and Strings. We could disable these special cases. For example, this would now be en error:

```c
void read_char(char *input);
```

```swift
let string: String = ...
read_char(string)
```

This case is not as dangerous as the inout conversion to raw pointers case addressed by this proposal because it does not expose internal control data. This case is only problematic when the pointer-taking-function returns or otherwise escapes the transient pointer, potentially resulting in a use-after-free, which is common problem with C APIs in general.

Naturally, this would break C interoperability in some common cases, such as using an Array as a byte buffer or viewing a String's null-terminated UTF8 representation as raw bytes.

Regardless, this issue is unrelated to raw pointers. If additional Array or String conversion restrictions are worthwhile, they merit separate discussion and should be pursued in an independent proposal.

## Future directions

### Add a BitwiseCopyable layout constraint

Generic code may require a source compatibility workaround even for generic types that are always bitwise copyable at runtime. In the near future, we plan to provide a `BitwiseCopyable` layout constraint for this purpose.

The following conversions would now be valid:

```swift
func foo<T: BitwiseCopyable>(_: T.Type) {
  var t: T = ...
  readBytes(&t)

  let array: [T] = ...
  readBytes(t)

  var array: [T] = ...
  readBytes(&t)
}
```
    
Various standard API's already require bitwise copyability, but there is no way to express the requirement. Instead, we use dynamic `_isPOD()` assertions. For years, there has been strong consensus that this should be handled by a layout constraint. Having this generic type constraint would give users a way to work around the new conversion restrictions in generic code.

A proposal for this feature is in progress. We expect `BitwiseCopyable` to be available before converting this proposed pointer conversion diagnostic to an error. Until then, programmers need to use the `withUnsafePointer` or `withUnsafeBytes` workarounds.

### Forbid withUnsafeBytes(of:_:) for non-bitwise-copyable or CoW types

The following use cases should be illegal similar to their inout conversion counterparts. When programmers ask for the "bytes" of a collection, they almost certainly wanted to view the elements. Rather than leak the internal representation of the collection.

```swift    
var string: String = ...
withUnsafeBytes(of: &string) {...}

var array: [T] = ...
withUnsafeBytes(of: &array) {...}

var set: Set<T> = ...
withUnsafeBytes(of: &set) {...}

var data: Data = ...
withUnsafeBytes(of: &data) {...}
```

In the future, this can be done with an overload:

```swift
@available(swift, deprecated: 6, message: "either use withUnsafePointer(of:) to point to a value containing object references, or use a method to retrieve the contents of a container")
public func withUnsafeBytes<T: BitwiseCopyable, Result>(
  of value: inout T,
  _ body: (UnsafeRawBufferPointer) throws -> Result
) rethrows -> Result {...}
```

By adding a new CopyOnWriteValue marker protocol with conformances from String, Array, Set, and Data, this restriction could be narrowed to diagnose only the most confusing cases shown above without affecting all non-bitwise-copyable values.

```swift
@available(swift, deprecated: 6, message: "use a method to retrieve the contents of a container")
public func withUnsafeBytes<T: CopyOnWriteValue, Result>(
  of value: inout T,
  _ body: (UnsafeRawBufferPointer) throws -> Result
) rethrows -> Result {...}
```

This is also clear, complementary improvement, but it can be pursued separately as a standard library enhancement. This proposal is focussed on implicit conversions.

## Acknowledgments

Thanks to Robert Widmann and Mike Ash for advice on handling associated objects.

Thanks to Joe Groff for suggesting the property wrapper workaround.

Thanks to Becca Royal-Gordon, Guillaume Lessard, and Tim Kientzle for in-depth proposal review.
