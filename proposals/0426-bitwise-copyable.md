# BitwiseCopyable

* Proposal: [SE-0426](0426-bitwise-copyable.md)
* Authors: [Kavon Farvardin](https://github.com/kavon), [Guillaume Lessard](https://github.com/glessard), [Nate Chandler](https://github.com/nate-chandler), [Tim Kientzle](https://github.com/tbkka)
* Review Manager: [Tony Allevato](https://github.com/allevato)
* Implementation: in main branch of compiler (https://github.com/apple/swift/pull/73235)
* Status: **Implemented (Swift 6.0)**
* Review: ([Pitch](https://forums.swift.org/t/pitch-bitwisecopyable-marker-protocol/69943)) ([First review](https://forums.swift.org/t/se-0426-bitwisecopyable/70479)) ([Returned for revision](https://forums.swift.org/t/returned-for-revision-se-0426-bitwisecopyable/70892)) ([Second review](https://forums.swift.org/t/se-0426-second-review-bitwisecopyable/71316)) ([Acceptance](https://forums.swift.org/t/accepted-se-0426-bitwisecopyable/71600))

<!-- *During the review process, add the following fields as needed:*

* Implementation: [apple/swift#NNNNN](https://github.com/apple/swift/pull/NNNNN) or [swiftlang/swift-evolution-staging#NNNNN](https://github.com/swiftlang/swift-evolution-staging/pull/NNNNN)
* Decision Notes: [Rationale](https://forums.swift.org/), [Additional Commentary](https://forums.swift.org/)
* Bugs: [SR-NNNN](https://bugs.swift.org/browse/SR-NNNN), [SR-MMMM](https://bugs.swift.org/browse/SR-MMMM)
* Previous Revision: [1](https://github.com/swiftlang/swift-evolution/blob/...commit-ID.../proposals/NNNN-filename.md)
* Previous Proposal: [SE-XXXX](XXXX-filename.md) -->

## Introduction

We propose a new, [limited](#limitations) protocol `BitwiseCopyable` that _can_ be conformed to by types that are "bitwise-copyable"[^1]--that is, that can be moved or copied with direct calls to `memcpy` and which require no special destroy operation.
When compiling generic code with such constraints, the compiler can emit these efficient operations directly, only requiring minimal overhead to look up the size of the value at runtime.
Alternatively, developers can use this constraint to selectively provide high-performance variations of specific operations, such as bulk copying of a container.

[^1]: The term "trivial" is used in [SE-138](0138-unsaferawbufferpointer.md) and [SE-0370](0370-pointer-family-initialization-improvements.md) to refer to types with this property. The discussion below will explain why certain generic or exported types that are bitwise-copyable will not in fact be `BitwiseCopyable`.

## Motivation

Swift can compile generic code into an unspecialized form in which the compiled function receives a value and type information about that value.
Basic operations are implemented by the compiler as calls to a table of "value witness functions."

This approach is flexible, but can represent significant overhead.
For example, using this approach to copy a buffer with a large number of `Int` values requires a function call for each value.

Constraining the types in generic functions to `BitwiseCopyable` allows the compiler (and in some cases, the developer) to instead use highly efficient direct memory operations in such cases.

The standard library already contains many examples of functions that could benefit from such a concept, and more are being proposed:

The `UnsafeMutablePointer.initialize(to:count:)` function introduced in [SE-0370](0370-pointer-family-initialization-improvements.md) could use a bulk memory copy whenever it statically knew that its argument was `BitwiseCopyable`.

The proposal for [`StorageView`](nnnn-safe-shared-contiguous-storage.md) includes the ability to copy items to or from potentially-unaligned storage, which requires that it be safe to use bulk memory operations:
```swift
public func loadUnaligned<T: BitwiseCopyable>(
  fromByteOffset: Int = 0, as: T.Type
) -> T

public func loadUnaligned<T: BitwiseCopyable>(
  from index: Index, as: T.Type
) -> T
```

And this proposal includes the addition of three overloads of existing standard library functions.

## Proposed solution

We add a new protocol `BitwiseCopyable` to the standard library:
```swift
@_marker public protocol BitwiseCopyable {}
```

That a type conforms to the protocol [implies](#transient-and-permanent) that the type is bitwise-copyable; the reverse is _not_ true.

Many basic types in the standard library will conformed to this protocol.

Developer's own types may be conformed to the protocol, as well.
The compiler will check any such conformance and emit a diagnostic if the type contains elements that are not `BitwiseCopyable`.

Furthermore, when building a module, the compiler will infer conformance to `BitwiseCopyable` for any non-exported struct or enum defined within the module whose stored members are all `BitwiseCopyable`,
except those for which conformance is explicitly [suppressed](#suppression).

Developers cannot conform types defined in other modules to the protocol.

## Detailed design

Our design first conforms a number of core types to `BitwiseCopyable`, and then extends that to aggregate types.

### Standard library changes

Many types and a few key protocols are constrained to `BitwiseCopyable`.
A few highlights:

* Integer types
* Floating point types
* SIMD types
* Pointer types
* `Unmanaged`
* `Optional`

For an exhaustive list, see the [appendix](#all-stdlib-conformers).

### Additional BitwiseCopyable types

In addition to the standard library types marked above, the compiler will recognize several other types as `BitwiseCopyable`:

* Tuples of `BitwiseCopyable` elements.

* `unowned(unsafe)` references.
  Such references can be copied without reference counting operations.

* `@convention(c)` and `@convention(thin)` function types do not carry a reference-counted capture context, unlike other Swift function types, and are therefore `BitwiseCopyable`.

### Explicit conformance to `BitwiseCopyable`

Enum and struct types can be explicitly declared to conform to `BitwiseCopyable`.
When a type is declared to conform, the compiler will check that its elements are all `BitwiseCopyable` and emit an error otherwise.

For example, the following struct can conform to `BitwiseCopayble`
```swift
public struct Coordinate : BitwiseCopyable {
  var x: Int
  var y: Int
}
```
because `Int` is `BitwiseCopyable`.

Similarly, the following enum can conform to `BitwiseCopyable`
```swift
public enum PositionUpdate : BitwiseCopyable {
  case begin(Coordinate)
  case move(x_change: Int, y_change: Int)
  case end
}
```
because both `Coordinate` and `(x_change: Int, y_change: Int)` are `BitwiseCopyable`.

The same applies to generic types.  For example, the following struct can conform to `BitwiseCopyable`
```swift
struct BittyBox<Value : BitwiseCopyable> : BitwiseCopyable {
  var first: Value
}
```
because its field `first` is a of type `Value` which is `BitwiseCopyable`.

Generic types may be `BitwiseCopyable` only some of the time.
For example,
```swift
struct RegularBox<Value> {
  var first: Value
}
```
cannot conform unconditionally because `Value` needn't conform to `BitwiseCopyable`.
In this case, a conditional conformance may be written:

```swift
extension Box : BitwiseCopyable where Value : BitwiseCopyable {}
```

### Automatic inference for aggregates

As a convenience, unconditional conformances will be inferred for structs and enums[^2] much of the time.
When the module containing the type is built, if all of the type's fields are `BitwiseCopyable`, the compiler will generate a conformance for it to `BitwiseCopyable`.

For generic types, a conformance will only be inferred if its fields unconditionally conform to `BitwiseCopyable`.
In the `RegularBox` example above, a conditional conformance will not be inferred.
If such a conformance is desired, the developer must explicitly write the conditional conformance.

[^2]: This includes raw-value enums.  While such enums do include a conformance to `RawRepresentable` where `RawValue` could be a non-conforming type (`String`), the instances of the enums themselves are `BitwiseCopyable`.

### Inference for imported types

The same inference will be done on imported C and C++ types.

For an imported C or C++ enum, the compiler will always generate a conformance to to `BitwiseCopyable`.

For an imported C struct, if all its fields are `BitwiseCopyable`, the compiler will generate a conformance to `BitwiseCopyable`.
The same is true for an imported C++ struct or class, unless the type is non-trivial[^3].

For an imported C or C++ struct, if any of its fields cannot be represented in Swift, the compiler will not generate a conformance.
This can be overridden, however, by annotating the type `__attribute__((__swift_attr__("BitwiseCopyable")))`.

[^3]: A C++ type is considered non-trivial (for the purpose of calls, as defined by the Itanium ABI) if any of the following is non-default: its constructor; its copy-constructor; its destructor.

### Inference for exported types

This does not apply to exported (`public`, `package`, or `@usableFromInline`) types.
In the case of a library built with library evolution, while all the type's fields may be `BitwiseCopyable` at the moment, the compiler can't predict that they will always be.
If this is the developer's intent, they can explicitly conform the type.
To avoid having semantics that vary based on library evolution, the same applies to all exported (`public`, `package`, or `@usableFromInline`) types.

For `@frozen` types, however, `BitwiseCopyable` conformance will be inferred.
That's allowed, even in the case of a library built with library evolution, because the compiler can see that the type's fields are all `BitwiseCopyable` and knows that they will remain that way.

For example, the compiler will infer a conformance of the following struct
```swift
@frozen
public struct Coordinate3 {
  public var x: Int
  public var y: Int
}
```
to `BitwiseCopyable`.

### Suppressing inferred conformance<a name="suppression"></a>

To suppress the inference of `BitwiseCopyable`, `~BitwiseCopyable` can be added to the type's inheritance list.

```swift
struct Coordinate4 : ~BitwiseCopyable {...}
```

Suppression must be declared on the type declaration itself, not on an extension.

### Transient and permanent notions<a name="transient-and-permanent"></a>

The Swift runtime already describes[^4] whether a type is bitwise-copyable.
It is surfaced, among other places, in the standard library function `_isPOD`[^5].

[^4]: The `IsNonPOD` value witness flag is set for every type that is _not_ bitwise-copyable.

[^5]: "POD" here is an acronym for "plain old data" which is yet another name for the notion of bitwise-copyable or trivial.

If a type conforms to `BitwiseCopyable`, then `_isPOD` must be true for the type.
The converse is not true, however.

As a type evolves, it may [both gain _and_ lose bitwise-copyability](#fluctuating-bitwise-copyability).
A type may only _gain_ a conformance to `BitwiseCopyable`, however;
it cannot _lose_ its conformance without breaking source and ABI.

The two notions are related, but distinct:
That a type `_isPOD` is a statement that the type is currently bitwise-copyable.
That a type conforms to `BitwiseCopyable` is a promise that the type is now and will remain bitwise-copyable as the library evolves.
In other words returning true from `_isPOD` is a transient property, and conformance to `BitwiseCopyable` is a permanent one.

For this reason, conformance to `BitwiseCopyable` is not inherent.
Its declaration on a public type provides a guarantee that the compiler cannot infer.

### Limitations of BitwiseCopyable<a name="limitations"></a>

Being declared with `@_marker`, `BitwiseCopyable` is a limited protocol.
Its limited nature allows the protocol's runtime behavior to be defined later, as needed.

1. `BitwiseCopyable` cannot be extended.
This limitation is similar to that on `Sendable` and `Any`:
it prevents polluting the namespace of conforming types, especially types whose conformance is inferred.

2. Because conformance to `BitwiseCopyable` is distinct from being bitwise-copyable,
the runtime cannot use the `IsNonPOD` bit as a proxy for conformance (although actual [conformance could be ignored](#casting-by-duck-typing)).
A separate mechanism would be necessary.
Until such a mechanism is added, `is`, `as?` and usage as a generic constraint to enable conditional conformance to another protocol is not possible.

### Standard library API improvements

The standard library includes a load method on both `UnsafeRawPointer` and `UnsafeMutableRawPointer`

```swift
@inlinable
@_alwaysEmitIntoClient
public func loadUnaligned<T>(
  fromByteOffset offset: Int = 0,
  as type: T.Type
) -> T
```

and a corresponding write method on `UnsafeMutableRawPointer`

```swift
@inlinable
@_alwaysEmitIntoClient
public func storeBytes<T>(
  of value: T, toByteOffset offset: Int = 0, as type: T.Type
)
```

that must be called with a trivial `T`.

We propose adding overloads of these methods to constrain the value to `BitwiseCopyable`:

```swift
// on both UnsafeRawPointer and UnsafeMutableRawPointer
@inlinable
@_alwaysEmitIntoClient
public func loadUnaligned<T : BitwiseCopyable>(
  fromByteOffset offset: Int = 0,
  as type: T.Type
) -> T

// on UnsafeMutableRawPointer
@inlinable
@_alwaysEmitIntoClient
public func storeBytes<T : BitwiseCopyable>(
  of value: T, toByteOffset offset: Int = 0, as type: T.Type
)
```

This allows for optimal code generation because `memcpy` instead of value witnesses can be used.

The existing methods that use a runtime assert instead of a type constraint will still be available (see [alternatives considered](#deprecation)).

## Effect on ABI stability

The addition of the `BitwiseCopyable` constraint to either a type or a protocol in a library will not cause an ABI break for users.

## Source compatibility

This addition of a new protocol will not impact existing source code that does not use it.

Removing the `BitwiseCopyable` conformance from a type is source-breaking.
As a result, future versions of Swift may conform additional existing types to `BitwiseCopyable`, but will not remove it from any type already conforming to `BitwiseCopyable`.

## Effect on API resilience

Adding a `BitwiseCopyable` constraint on a generic type will not cause an ABI break.
As with any protocol, the additional constraint can cause a source break for users.

## Future Directions

### Automatic derivation of conditional conformances

The wrapper type mentioned above
```swift
struct RegularBox<Value> {
  var first: Value
}
```
cannot conform to `BitwiseCopyable` unconditionally.
It can, however, so long as `Value` is `BitwiseCopyable`.

With this proposal, such a conditional conformance can be added manually:

```swift
extension Box : BitwiseCopyable where Value : BitwiseCopyable {}
```

In the future we may in some cases be able to derive it automatically.

### Dynamic casting

Being a [limited](#limitations) protocol, `BitwiseCopyable` does not currently have any runtime representation.
While a type's [transient](#transient-and-permanent) bitwise-copyability has a preexisting runtime representation, that is different from the type conforming to `BitwiseCopyable`.

Being a low-level, performance-enabling feature, it is not clear that dynamic casting should be allowed at all.
If it were to be allowed at some point, a few different approaches can already be foreseen:

#### Explicitly record a type's conformance

The standard way to support dynamic casting would be to represent a type's conformance to the protocol and query the type at runtime.

This approach has the virtue that dynamic casting behaves as usual.
A type could only be cast to `BitwiseCopyable` if it actually conformed to the protocol.
For example, casting a type which suppressed a conformance to `BitwiseCopyable` would fail.

If this approach were taken, such casting could be back-deployed as far as the oldest OS in which this runtime representation was added.
Further back deployment would be possible by adding conformance records to back deployed binaries.

#### Duck typing for BitwiseCopyable<a name="casting-by-duck-typing"></a>

An alternative would be to dynamically treat any type that's bitwise-copyable as if it conformed to `BitwiseCopyable`.

This is quite different from typical Swift casting behavior.
Rather than relying on a permanent characteristic of the type, it would rely on a [transient](#transient-and-permanent) one.
This would be visible to the programmer in several ways:
- different overloads would be selected for a value of concrete type from those selected for a value dynamically cast to `BitwiseCopyable`
- dynamic casts to `BitwiseCopyable` could fail, then succeed, then fail again in successive OS versions

On the other hand, these behavioral differences may be desireable.

Considering that this approach would just ignore the existence of conformances to `BitwiseCopyable`,
it would be reasonable to ignore the existence of a suppressed conformance as well.

This approach also has the virtue of being completely back-deployable[^6].
[^6]: All runtimes have had the `IsNonPOD` bit.

### BitwiseMovable

Most Swift types have the property that their representation can be relocated in memory with direct memory operations.
This could be represented with a `BitwiseMovable` protocol that would be handled similarly to `BitwiseCopyable`.

### BitwiseCopyable as a composition

Some discussion in the pitch thread discussed how `BitwiseCopyable` could be defined as the composition of several protocols.
For example,
```swift
typealias BitwiseCopyable = Bitwise & Copyable & DefaultDeinit
```
Such a definition remains possible after this proposal.

Because `BitwiseCopyable` is annotated `@_marker`, its ABI is rather limited.
Specifically, it only affects name mangling.
If, in a subsequent proposal, the protocol were redefined as a composition, symbols into which `BitwiseCopyable` was mangled could still be mangled in the same way, ensuring ABI compatibility.

## Alternatives considered

### Alternate Spellings

**Trivial** is widely used within the compiler and Swift evolution discussions to refer to the property of bitwise copyability. `BitwiseCopyable`, on the other hand, is more self-documenting.

### Deprecation of unconstrained functions dependent on `isPOD`<a name="deprecation"></a>

The standard library has a few pre-existing functions that receive a generic bitwise-copyable value as a parameter. These functions work with types for which the `_isPOD()` function returns true, even though they do not have a `BitwiseCopyable` conformance. If we were to deprecate these unconstrained versions, we would add unresolvable warnings to some of the codebases that use them. For example, they might use types that could be conditionally `BitwiseCopyable`, but come from a module whose types have not been conformed to `BitwiseCopyable` by their author. Furthermore, as explained [above](#transient-and-permanent), it is not necessarily the case that a transiently bitwise-copyable type can be permanently annotated as `BitwiseCopyable`.

At present, the unconstrained versions check that `_isPOD()` returns true in debug mode only. We may in the future consider changing them to check at all times, since in general their use in critical sections will have been updated to use the `BitwiseCopyable`-constrained overloads.

## Acknowledgments

This proposal has benefitted from discussions with John McCall, Joe Groff, Andrew Trick, Michael Gottesman, and Arnold Schwaigofer.

## Appendix: Standard library conformers<a name="all-stdlib-conformers"></a>

The following protocols in the standard library will gain the `BitwiseCopyable` constraint:

- `_Pointer`
- `SIMDStorage`, `SIMDScalar`, `SIMD`


The following types in the standard library will gain the `BitwiseCopyable` constraint:

- `Optional<T>` when `T` is `BitwiseCopyable`
- The fixed-precision integer types:
  - `Bool`
  - `Int8`, `Int16`, `Int32`, `Int64`, `Int`
  - `UInt8`, `UInt16`, `UInt32`, `UInt64`, `UInt`
  - `StaticBigInt`
  - `UInt8.Words`, `UInt16.Words`, `UInt32.Words`, `UInt64.Words`, `UInt.Words`
  - `Int8.Words`, `Int16.Words`, `Int32.Words`, `Int64.Words`, `Int.Words`
- The fixed-precision floating-point types:
  - `Float`, `Double`, `Float16`, `Float80`
  - `FloatingPointSign`, `FloatingPointClassification`
- The family of `SIMDx<Scalar>` types
- The family of unmanaged pointer types:
  - `OpaquePointer`
  - `UnsafeRawPointer`, `UnsafeMutableRawPointer`
  - `UnsafePointer`, `UnsafeMutablePointer`, `AutoreleasingUnsafeMutablePointer`
  - `UnsafeBufferPointer`, `UnsafeMutableBufferPointer`
  - `UnsafeRawBufferPointer`, `UnsafeMutableRawBufferPointer`
  - `Unmanaged`
  - `CVaListPointer`
- Some types related to collections
  - `EmptyCollection`
  - `UnsafeBufferPointer.Iterator`, `UnsafeRawBufferPointer.Iterator`, `EmptyCollection.Iterator`
  - `String.Index`, `CollectionDifference.Index`
- Some types related to unicode
  - `Unicode.ASCII`, `Unicode.UTF8`, `Unicode.UTF16`, `Unicode.UTF32`, `Unicode.Scalar`
  - `Unicode.ASCII.Parser`, `Unicode.UTF8.ForwardParser`, `Unicode.UTF8.ReverseParser`, `Unicode.UTF16.ForwardParser`, `Unicode.UTF16.ReverseParser`, `Unicode.UTF32.Parser`
  - `Unicode.Scalar.UTF8View`, `Unicode.Scalar.UTF16View`
  - `UnicodeDecodingResult`
- Some fieldless types
  - `Never`, `SystemRandomNumberGenerator`
- `StaticString`
- `Hasher`
- `ObjectIdentifier`
- `Duration`
- Atomic changes
  - `AtomicRepresentable.AtomicRepresentation`
  - `AtomicOptionalRepresentable.AtomicOptionalRepresentation`

## Appendix: Fluctuating bitwise-copyability<a name="fluctuating-bitwise-copyability"></a>

Let's say the following type is defined in a framework built with library evolution.

```swift
public struct Dish {...}
```

In the first version of the framework, the type only contains bitwise-copyable fields:

```swift
/// NoodleKit v1.0

public struct Dish {
  public let substrate: Noodle
  public let isTopped: Bool
}
```

So in version `1.0`, the type is bitwise-copyable.

In the next version of the framework, to expose more information to its clients, the stored `Bool` is replaced with a stored `Array`:

```swift
/// NoodleKit v1.1

public struct Dish {
  public let substrate: Noodle
  public let toppings: [Topping]
  public var isTopped: Bool { toppings.count > 0 }
}
```

As a result, in version `1.1`, the type is _not_ bitwise-copyable.

In a subsequent version, as an optimization, the stored `Array` is replaced with an `OptionSet`

```swift
/// NoodleKit v2.0

public struct Dish {
  public let substrate: Noodle
  private let toppingOptions: Topping
  public let toppings: [Topping] { ... }
  public var isTopped: Bool { toppings.count > 0 }
}
```

In release `2.0` the type is once again bitwise-copyable.
