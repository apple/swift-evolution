# Renaming `String.init<T>(_: T)`

* Proposal: [SE-0089](0089-rename-string-reflection-init.md)
* Authors: [Austin Zheng](https://github.com/austinzheng), [Becca Royal-Gordon](https://github.com/beccadax)
* Review Manager: [Chris Lattner](https://github.com/lattner)
* Status: **Implemented (Swift 3.0)**
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-se-0089-renaming-string-init-t-t/3097)
* Bug: [SR-1881](https://bugs.swift.org/browse/SR-1881)
* Previous Revisions: [1](https://github.com/swiftlang/swift-evolution/blob/40aecf3647c19ae37730e39aa9e54b67fcc2be86/proposals/0089-rename-string-reflection-init.md)

## Introduction

Swift's `String` type ships with a large number of initializers that take one unlabeled argument. One of these initializers, defined as `init<T>(_: T)`, is used to create a string containing the textual representation of an object. It is very easy to write code which accidentally invokes this initializer, when one of the other synonymous initializers was desired. Such code will compile without warnings and can be very difficult to detect.

Discussion threads: [pre-proposal](https://forums.swift.org/t/string-initializers-and-developer-ergonomics/2507), [review thread](https://forums.swift.org/t/review-se-0089-renaming-string-init-t-t/2663), [post-review thread](https://forums.swift.org/t/returned-for-revision-se-0089-renaming-string-init-t-t/2782)

## Motivation

`String` ships with a number of initializers which take a single unlabeled argument. These include non-failable initializers which create a `String` out of a `Character`, `NSString`, `CharacterView`, or `UnicodeScalarView`, initializers which build a string out of a number, and failable initializers which take a `UTF8View` or a `UTF16View`.

There are at least two possible situations in which a user may write incorrect code which nevertheless compiles successfully:

* The user means to call one of the non-failable initializers besides the `init<T>(_: T)` initializer, but passes in an argument of incorrect type.
* The user means to call one of the failable initializers, but accidentally assigns the created object to a value of non-nullable type.

In both cases the compiler silently infers the use of the `init<T>(_: T)` initializer in lieu of the desired initializer. This may result in code which inadvertently utilizes the expensive reflection machinery and/or produces an unintentionally lossy representation of the value.

## Proposed solution

A proposed solution to this problem follows:

* The current reflection-based `String.init<T>(_: T)` initializer will be renamed to `String.init<T>(describing: T)`. This initializer will rarely be invoked directly by user code.

* A new protocol will be introduced: `LosslessStringConvertible`, which refines/narrows `CustomStringConvertible`. This protocol will be defined as follows:

	```swift
	protocol LosslessStringConvertible : CustomStringConvertible {
		/// Instantiate an instance of the conforming type from a string representation.
		init?(_ description: String)
	}
	```

	Values of types that conform to `LosslessStringConvertible` are capable of being represented in a lossless, unambiguous manner as a string. For example, the integer value `1050` can be represented in its entirety as the string `"1050"`. The `description` property for such a type must be a value-preserving representation of the original value. As such, it should be possible to attempt to create an instance of a `LosslessStringConvertible` conforming type from a string representation.

	A possible alternate name for this protocol is `ValuePreservingStringLiteral`. The core team may wish to choose this name instead, or another name that better describes the protocol's contract.

* A new `String` initializer will be introduced: `init<T: LosslessStringConvertible>(_ v: T) { self = v.description }`. This allows the `String(x)` syntax to continue to be used on all values of types that can be converted to a string in a value-preserving way.

* As a performance optimization, the implementation of the string literal interpolation syntax will be changed to prefer the unlabeled initializer when interpolating a type that is `LosslessStringConvertible` or that otherwise has an unlabeled `String` initializer, but use the `String.init<T>(describing: T)` initializer if not.

### Standard library types to conform

The following standard library types and protocols should be changed to conform to `LosslessStringConvertible`.

#### Protocols

* `FloatingPoint`: "FP types should be able to conform. There are algorithms that are guaranteed to turn IEEE floating point values into a decimal representation in a reversible way. I don’t think we care about NaN payloads, but an encoding could be created for them as well." (Chris Lattner)
* `Integer`

#### Types

* `Bool`: either "true" or "false", since these are their canonical representations.
* `Character`
* `UnicodeScalar`
* `String`
* `String.UTF8View`
* `String.UTF16View`
* `String.CharacterView`
* `String.UnicodeScalarView`
* `StaticString`

## Future directions

### Additional conformances to `LosslessStringLiteral`

Once [conditional conformance of generic types to protocols](https://github.com/apple/swift/blob/master/docs/GenericsManifesto.md#conditional-conformances-) is implemented, the additional protocols and types below are candidates for conformance to `LosslessStringLiteral`:

#### Protocols

* `RangeReplaceableCollection where Iterator.Element == Character`
* `RangeReplaceableCollection where Iterator.Element == UnicodeScalar`
* `SetAlgebra where Iterator.Element == Character`
* `SetAlgebra where Iterator.Element == UnicodeScalar`

#### Types

* `ClosedRange where Bound : LosslessStringConvertible`
* `CountableClosedRange where Bound : LosslessStringConvertible`
* `CountableRange where Bound : LosslessStringConvertible`
* `Range where Bound : LosslessStringConvertible`

## Impact on existing code

This API change may impact existing code.

Code which intends to invoke `init<T>(_: T)` will need to be modified so that the proper initializer is called. In addition, it is possible that this change may uncover instances of the erroneous behavior described previously.

## Alternatives considered

One alternative solution might be to make `LosslessStringConvertible` a separate protocol altogether from `CustomStringConvertible` and `CustomDebugStringConvertible`. Arguments for and against that approach can be found in this [earlier version of this proposal](https://github.com/austinzheng/swift-evolution/blob/27ba68c2fbb8978aac6634c02d8a572f4f5123eb/proposals/0089-rename-string-reflection-init.md).

