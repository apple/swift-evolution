# Import Objective-C `id` as Swift `Any` type

* Proposal: [SE-0116](0116-id-as-any.md)
* Author: [Joe Groff](https://github.com/jckarter)
* Review Manager: [Chris Lattner](https://github.com/lattner)
* Status: **Implemented (Swift 3.0)**
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-se-0116-import-objective-c-id-as-swift-any-type/3476)
* Previous Revision: [1](https://github.com/swiftlang/swift-evolution/blob/b9a0ab5f7db4d3806c7941a07acedc5f0fe36e55/proposals/0116-id-as-any.md)

## Introduction

Objective-C interfaces that use `id` and untyped collections should be imported
into Swift as taking the `Any` type instead of `AnyObject`.

Swift-evolution thread: [Importing Objective-C `id` as Swift `Any`](https://forums.swift.org/t/pitch-importing-objective-c-id-as-swift-any/3236)

## Motivation

Objective-C's `id` type is currently imported into Swift as `AnyObject`. This
is the intuitive thing to do, but creates a growing tension between idiomatic
Objective-C and Swift code. One of Swift's defining features is its value
types, such as `String`, `Array`, and `Dictionary`, which allow for efficient
mutation without the hazards of accidental state sharing prevalent with mutable
classes. To interoperate with Objective-C, we transparently **bridge** value
types to matching idiomatic Cocoa classes when we know the static types of
method parameters and returns. However, this doesn't help with polymorphic
Objective-C interfaces, which to this day are frequently defined in terms of
`id`. These interfaces come into Swift as `AnyObject`, which doesn't naturally
work with value types. To keep the Swift experience using value types with
Cocoa feeling idiomatic, we've papered over this impedance mismatch via various
language mechanisms:

- Bridgeable types *implicitly convert* to their bridged object type. This
  makes it convenient to use bridgeable types with polymorphic Objective-C
  interfaces, for example, to build a heterogeneous property list as an
  `[AnyObject]` of bridged objects (which in turn bridges to an `NSArray`).
- Given a dynamically-typed object of static type `AnyObject`, the value can be
  dynamically cast back to a Swift value type using `is`, `as?`, and `as!`.

While often convenient, these features are inconsistent with the rest of the
language and have in practice been a common source of problems and surprising
behavior. We have popular proposals in flight to remove the special
cases:

- [SE-0072](0072-eliminate-implicit-bridging-conversions.md)
  (accepted) removes the implicit conversion, requiring one to explicitly write
  `x as NSString` or `x as AnyObject` to use a bridgeable value as an object.
- [SE-0083](0083-remove-bridging-from-dynamic-casts.md)
  (deferred for later consideration) removes the dynamic casting behavior and
  overloading of `as` coercion, requiring one to use normal constructors to
  convert between value types and object types.

Meanwhile, Foundation has extensively adopted value types in Swift 3, making
this a bigger problem in scope than a handful of standard library types. Swift
and Foundation are also being ported to non-Apple platforms that don't ship an
Objective-C runtime, and we want to provide a consistent interface to
Foundation between Darwin and other platforms. This means that, even
independent of Objective-C, Foundation is still forced to express abstractions
in terms of `AnyObject`. Our current status quo
pits the goal of providing a more consistent and predictable standalone
language against the goal of providing a portable set of core libraries--if we
chip away at the implicit bridging behavior to make the language more
predictable, the parts of the standard library and Foundation that are designed
to take most advantage of Swift's features become harder and less attractive to
use, and the less idiomatic `NS` container classes need to be interacted with
more frequently.

The fundamental tension here is that, whereas ObjC's polymorphism is centered
on objects, Swift opens up polymorphism to all types. Rather than treat
bridging as something only a set of preordained types can partake in, we can
say that **all** Swift types can bridge to an Objective-C object. By doing
this, we can import Objective-C APIs in terms of Swift's `Any`,
making them interoperate seamlessly with Swift value types without special-case
language behavior.  If we achieve this, we can move nearly all of the bridging
glue "below the fold" into the compiler implementation, allowing users to
work with value types and have them just work with Cocoa APIs without relying
on special language rules.

## Proposed solution

- We change the behavior of Objective-C APIs imported into Swift so that the
  `id` type is imported as `Any` in bridgeable positions. At compile time and
  runtime, the compiler introduces a **universal bridging conversion** operation
  when a Swift value or object is passed into Objective-C as an `id` parameter.
- When `id` values are brought into Swift as `Any`, we use the runtime's
  existing **ambivalent dynamic casting** support to handle bridging back to
  either class references or Swift value types.
- Untyped Cocoa collections come in as collections of `Any`.  `NSArray` imports
  as `[Any]`, `NSDictionary` as `[AnyHashable: Any]`, and `NSSet` as
  `Set<AnyHashable>` (using an `AnyHashable` type erasing container to be
  designed [in a follow-up proposal](#anyhashabletype)).

## Detailed design

### Universal bridging conversion into Objective-C `id`

To describe what bridging an `Any` to `id` means, we need to establish a
*universal bridging conversion* from any Swift type to an Objective-C object.
There are several cases to consider:

* **Classes** are the easiest case—they exist in both Objective-C and Swift
  and play many of the same roles. A Swift class reference can be brought into
  Objective-C as is.
* **Bridged value types** with established bridging behavior, such as `String`,
  `Array`, `Dictionary`, `Set`, etc., should continue to bridge to instances
  of their corresponding idiomatic Cocoa classes, using the existing internal
  `_ObjectiveCBridgeable` protocol. The set of bridged types can be extended
  in the Swift implementation (and hopefully, eventually, by third parties too)
  by adding conformances to that protocol. This proposal does not address
  adding or removing any new bridging behavior, though that would be a
  natural [follow-up proposal](#bridgingmoretypestoidiomaticobjects).
* **Unbridged value types** without an obvious Objective-C analog can still be
  boxed in an instance of an immutable class. The name and functionality of
  this class doesn't need to exposed in the language model, beyond being
  minimally `id`-compatible to round-trip through Objective-C code, and being
  dynamically castable back into the original Swift type from Swift code when
  an `Any` value contains a reference to a box.

### Dynamic casting from `Any`

The runtime currently has the ability to dynamically apply bridging conversions.
If an `Any` or other existential contains a value of bridgeable type, dynamic
casts will succeed for either the dynamic type or its bridged counterpart:

```swift
var x: Any = "foo" as String
x as? String   // => String "foo"
x as? NSString // => NSString "foo"

x = "bar" as NSString
x as? String   // => String "bar"
x as? NSString // => NSString "bar"
```

This *ambivalent dynamic casting* behavior is exactly what we need to interface
with Objective-C APIs that return `id`s back into Swift as `Any`, since it is
impossible to know locally whether the object is intended to be consumed in
Swift as a bridged value or as a class instance.

### Bridging Objective-C Collections

If we take the class constraint away from singular `id` values, it also makes
sense to do so for collections, for instance, bridging an untyped `NSArray`
from Objective-C to a Swift `[Any]`. This also implies that we would need to
lift the current class restriction on covariant Array conversions—`[T]` would
need to be supported as a subtype of `[Any]`.

`Dictionary` and `Set` require their keys to be `Hashable` at minimum, so we
would need a way to represent a heterogeneous `Hashable` type to bridge an
untyped `NSDictionary` or `NSSet`. The `Hashable`
protocol type cannot itself be used due to limitations in Swift 3; namely, 
`Hashable` refines the `Equatable` protocol, which demands `Self` constraints
of its `==` requirement, and beyond that, we do not support protocol types
conforming to their own protocols in general. As a stopgap, we will likely need
an [`AnyHashable` type-erased container](#anyhashabletype) in the standard library.

## Impact on existing code

For most code, the combination of this proposal with
[SE-0072](0072-eliminate-implicit-bridging-conversions.md)
should have the net effect of most Swift 2 style code working as it does today,
allowing value types to be passed into untyped Objective-C APIs without
requiring explicit bridging or unbridging operations. There will definitely
be edge cases that may behave slightly differently, since the `AnyObject`
constraint may nudge overload resolution or implicit conversion in
a different direction from what they would take absent that constraint.

## `AnyHashable` type

We need a type-erased container to represent a heterogeneous hashable type
that is itself `Hashable`, for use as the upper-bound type of heterogeneous
`Dictionary`s and `Set`s. The user model for this type would ideally align
with our long-term goal of supporting `Hashable` existentials directly, so
the type deserves some short-term compiler support to help us get there.
This type deserves its own proposal and design discussion.

## Future Directions

Once we've established a universal bridging mechanism for all Swift types,
this enables further closing of the expressivity gap with value types and the
Objective-C bridge:

### Importing Objective-C generics as unconstrained

We could lift the `AnyObject` constraint on imported ObjC generic type
parameters, allowing ObjC generics to work with Swift value types.

### Letting Value Types Conform to Objective-C Protocols

If we can bridge arbitrary Swift values to Objective-C objects, then we could
conceivably implement `@objc` protocol conformance for Swift value types as
well, by setting up the bridged Objective-C class to conform to the protocol
and respond to the necessary messages in the Objective-C runtime. This would
allow Foundation to vend protocols that work with its value types without
compromising portability between Darwin and corelibs platforms. If we wanted to
make this work, it would inform some tradeoffs in the potential implementation:

- We would probably need to produce a unique boxing Objective-C class for every
  type that conformed to an Objective-C protocol, where we might otherwise be
  able to share one class (or `NSValue` for C types).
- For value types with custom bridging, like `String`/`NSString`, does an
  `@objc` conformance automatically apply to the bridged class, if not at
  compile time, at least at runtime?
- Many Objective-C protocols are *intended* to be class-constrained,
  particularly delegate protocols, which are idiomatically weak-referenced from
  the delegatee class. If `@objc` no longer implies a class constraint in
  Swift, it wouldn't be possible for a property of `@objc` protocol type to be
  `weak`, unless we underwent an annotation or heuristic effort to distinguish
  Objective-C protocols that are supposed to be class-constrained.

### Deciding the fate of `AnyObject` lookup

We currently bestow the `AnyObject` existential type with the special ability
to look up any `@objc` method dynamically, in order to ensure `id`-based ObjC
APIs remain fluent when used in Swift. This is another special, unprincipled,
nonportable feature that relies on the Objective-C runtime. If we change  `id`
to bridge to `Any`, it definitely no longer makes sense to apply to
`AnyObject`. A couple of possibilities to consider:

- We could transfer the existing `AnyObject` behavior verbatim to `Any`.
- We could attempt to eliminate the behavior as a language feature. An
  approximation of AnyObject's magic behavior can be made using operators and
  unapplied method references, in a way that also works for Swift types:

    ```swift
    /// Dynamically dispatch a method on Any.
    func => <T, V>(myself: Any, method: (T) -> V) -> V? {
      if let myself = myself as? T {
        return method(myself)
      }
      return nil
    }
    ```
    
    though that's not quite the right thing for `id` lookup, since you want a
    `respondsToSelector` rather than `isKindOfClass` check.
- We could narrow the scope of the behavior. Jordan has suggested allowing
  only property and subscript lookup off of `AnyObject` or `Any`, as a way
  of allowing easy navigation of property lists, one of the most common
  sources of `id` in Foundation.
- If we're confident that the SDK will be sufficiently Swiftified that `id`s
  become relatively rare, maybe we could get away without a replacement at all.

### Hiding the `NSObjectProtocol` in Swift

Aside from `AnyObject`, another way unnecessary `@objc`-isms intrude themselves
into Swift code is through `NSObjectProtocol` requirements. In practice, nearly
every class in Swift on an Apple platform conforms to this protocol--native
Swift classes inherit from a common Objective-C `SwiftObject` base class
internal to the Swift runtime that implements the `NSObjectProtocol` methods,
and almost all Cocoa classes inherit either `NSObject` or `NSProxy`. We can
also make the box class used to bridge Swift values provide `NSObjectProtocol`
functionality. Eliminating `NSObjectProtocol` as a formal requirement in Swift
will allow native Swift classes, and often value types too, to interoperate
more smoothly with Cocoa code with less explicit `@objc` interop glue.

### Bridging more types to idiomatic objects

Removing the `AnyObject` constraint and special typing rules makes it more
important for the `Any`-to-`id` to do the right thing for as many types as
possible. Some obvious candidates include:

- Extending `NSNumber` bridging to cover not only `Int` and `Double`, but all
  `[U]IntNN` and `FloatNN` numeric types, as well as the `Decimal` struct from
  Foundation.
- Bridging Foundation and CoreGraphics structs like `CGRect` and `NSRange` to
  `NSValue`, the idiomatic box class for those types.
- When an `Optional` is passed as a non-nullable `id`, we might consider
  bridging the optional's `nil` value to `NSNull`. This would allow containers
  of optional such as `[Foo?]` to bridge idiomatically to `NSArray`s of
  `Foo` and `NSNull` elements.

### Simplifying pure Swift dynamic casting behavior

[SE-0083](0083-remove-bridging-from-dynamic-casts.md)
sought to remove the *ambivalent dynamic casting* behavior and
overloading of `as` coercion from Swift. This proposal *relies* on ambivalent
dynamic casting to make sense of incoming `id` values returned from Objective-C
into Swift. We could conceivably still *limit* ambivalent dynamic casting only
to `Any` existentials with Objective-C provenance, so that "pure" Swift code has
simpler, more predictable dynamic casting behavior where interop is not
involved. We don't have time to evaluate this in the remaining time for Swift 3,
but since the ambivalent casting behavior must remain in the runtime for
Objective-C interop and will at best be conditionalized, we can potentially
evaluate this later as a dialect change; in Swift 3, all existentials
effectively have the "ambivalent" flag set, but in a future version of Swift,
we could start turning it off for some values.

--------------------------------------------------------------------------------

## Revision history

### version 2

Reduced the scope of the proposal further based on design discussion,
implementation, and scheduling concerns:

- Subset out conditional ambivalent dynamic casting from the proposal. We don't
  have time in Swift 3 to implement or evaluate this.
- Move `NSObjectProtocol` and `NSValue`/`NSNumber` bridging to future
  directions. These can be done additively.
