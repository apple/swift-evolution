# Library Evolution for Stable ABIs

* Proposal: [SE-0260](0260-library-evolution.md)
* Authors: [Jordan Rose](https://github.com/jrose-apple), [Ben Cohen](https://github.com/airspeedswift)
* Review Manager: [John McCall](https://github.com/rjmccall)
* Status: **Implemented (Swift 5.1)**
* Implementation: Implemented in Swift 5 for standard library use. PR with renamed attributes [here](https://github.com/apple/swift/pull/24185).
* Pre-review discussion: [Forum thread](https://forums.swift.org/t/pitch-library-evolution-for-stable-abis/)
* Review: ([review](https://forums.swift.org/t/se-0260-library-evolution-for-stable-abis/24260)) ([acceptance](https://forums.swift.org/t/accepted-se-0260-library-evolution-for-stable-abis/24845))

## Introduction

One of Swift's goals is to be a good language for libraries with binary compatibility concerns, such as those shipped as part of Apple's OSs. This includes giving library authors the flexibility to add to their public interface, and to change implementation details, without breaking binary compatibility. At the same time, it's important that library authors be able to opt out of this flexibility in favor of performance.

This proposal introduces:

- a "library evolution" build mode for libraries that are declaring ABI stability, which preserves the ability to make certain changes to types without breaking the library's ABI; and
- an attribute for such libraries to opt out of this flexibility on a per-type basis, allowing certain compile-time optimizations.

The mechanisms for this are already in place, and were used to stabilize the ABI of the standard library. This proposal makes them features for use by any 3rd-party library that wishes to declare itself ABI stable.

**This build mode will have no impact on libraries built and distributed with an app**. Such libraries will receive the same optimization that they have in previous versions of Swift.

_Note: this proposal will use the word "field" to mean "stored instance property"._

## Motivation

As of Swift 5, libraries are able to declare a stable ABI, allowing a  library binary to be replaced with a newer version without requiring client programs to be recompiled. 

What consitutes the ABI of a library differs from language to language. In C and C++, the public ABI for a library includes information that ideally would be kept purely as an implementation detail. For example, the _size_ of a struct is fixed as part of the ABI, and is known to the library user at compile time. This prevents adding new fields to that type in later releases once the ABI is declared stable. If direct access to fields is allowed, the _layout_ of the struct can also become part of the ABI, so fields cannot then be reordered.

This often leads to manual workarounds. A common technique is to have the struct hold only a pointer to an "impl" type, which holds the actual stored properties. All access to these properties is made via function calls, which can be updated to handle changes to the layout of the "impl" type. This has some obvious downsides:

- it introduces indirection and function call overhead when accessing fields;
- it means the fields of the struct must be stored on the heap even when they could otherwise be stored on the stack, with that heap allocation needing to be managed via reference counting or some other mechanism;
- it means that the fields of the struct data are not stored in contiguous memory when placed in an array; and 
- the implementation of all this must be provided by the library author.

A similar challenge occurs with Swift enums. As discussed in [SE-0192][], introducing new cases to an enum can break source compatibility. There are also ABI consequences to this: adding new enum cases sometimes means increasing the storage needed for the enum, which can affect the size and layout of an enum in the same way adding fields to a struct can.

The goal of this proposal is to reduce the burden on library developers by building into the compiler an automatic mechanism to reserve the flexibility to alter the internal representation of structs and enums, without manual workarounds. This mechanism can be implemented in such a way that optimizations such as stack allocation or contiguous inline storage of structs can still happen, while leaving the size of the type to be determined at runtime. 

  [SE-0192]: https://github.com/swiftlang/swift-evolution/blob/master/proposals/0192-non-exhaustive-enums.md

## Proposed solution

The compiler will gain a new mode, called "library evolution mode". This mode will be off by default. A library compiled with this mode enabled is said to be _ABI-stable_.

When a library is compiled with library evolution mode enabled, it is not an ABI-breaking change to modify the fields in a struct (to add, remove, or reorder them), and to add new enum cases (including with associated values). This implies that clients must manipulate fields and enum cases indirectly, via non-inlinable function calls. Information such as the size of the type, and the layout of its fields, becomes something that can only be determined at runtime. Types that reserve this flexibility are referred to as "resilient types".

A new `@frozen` attribute will be introduced to allow library authors to opt out of this flexibility on a per-type basis. This attribute promises that stored instance properties within the struct will not be *added,* *removed,* or *reordered*, and that an enum will never *add,* *remove,* or *reorder* its cases (note removing, and sometimes reordering, cases can already be source breaking, not just ABI breaking). The compiler will use this for optimization purposes when compiling clients of the type. The precise set of allowed changes is defined below.

This attribute has does not affect a compiled binary when library evolution mode is off. In that case, the compiler acts as if *every* struct or enum in the library being compiled is "frozen". However, restrictions such as requiring stored properties on frozen structs to be ABI-public will still be enforced to avoid confusion when moving between modes.

Binaries compiled without this mode should not be declared as ABI-stable. While they should be stable even without library evolution mode turned on, doing this will not be a supported configuration.

## Detailed design

### Library evolution mode

A new command-line argument, `-enable-library-evolution`, will enable this new mode.

Turning on library evolution mode will have the following effects on source code:

- The default behavior for enums will change to be non-frozen. This is the only change visible to _users_ of the library, who will need to use the `@unknown default:` technique described in SE-0192.
- To ensure that all fields are initialized, inlinable `init`s of the type must delegate to a non-inlinable `init`.

### `@frozen` on `struct` types

When a library author is certain that there will never be a need to add fields to a struct in future, they may mark that type as `@frozen`. 

This will allow the compiler to optimize away at compile time some calls that would otherwise need to be made at runtime (for example, it might access fields directly without the indirection).

When compiling with binary stability mode on, a struct can be marked `@frozen` as long as it meets all of the following conditions:

- The struct is *ABI-public* (see [SE-0193][]), i.e. `public` or marked `@usableFromInline`.
- Every class, enum, struct, protocol, or typealias mentioned in the types of the struct's fields is ABI-public.
- No fields have observing accessors (`willSet` or `didSet`).
- If a field has an initial value, the expression computing the initial value does not reference any types or functions that are not ABI-public.

```swift
@frozen
public struct Point {
  public var x: Double
  public var y: Double

  public init(_ x: Double, _ y: Double) {
    self.x = x
    self.y = y
  }
}
```

This affects what changes to the struct's fields affect the ABI of the containing library:

| Change | Normal struct | `@frozen` struct
|---|:---:|:---:
| Adding fields | Allowed | **Affects ABI**
| Reordering fields | Allowed | **Affects ABI**
| Removing ABI-public fields | Affects ABI | Affects ABI
| Removing non-ABI-public fields | Allowed | **Affects ABI**
| Changing the type of an ABI-public field | Affects ABI | Affects ABI
| Changing the type of a non-ABI-public field | Allowed | **Affects ABI**
| Changing a stored instance property to computed | Allowed | **Affects ABI**
| Changing a computed instance property to stored | Allowed | **Affects ABI**
| Changing the access of a non-ABI-public field | Allowed | Allowed
| Marking an `internal` field as `@usableFromInline` | Allowed | Allowed
| Changing an `internal` ABI-public field to be `public` | Allowed | Allowed

> Note: This proposal is implemented already and in use by the standard library, albeit under different names. The command-line flag is `-enable-library-evolution`; the attribute is `@_fixed_layout` for structs, and `@_frozen` for enums.

  [SE-0193]: https://github.com/swiftlang/swift-evolution/blob/master/proposals/0193-cross-module-inlining-and-specialization.md

#### Guarantees

Marking a struct `@frozen` only guarantees that its stored instance properties won't change. This allows the compiler to perform certain optimizations, like ignoring properties that are never accessed, or eliminating redundant loads from the same instance property. However, it does not provide a handful of other guarantees that a C struct might:

- **It is not guaranteed to be "trivial"** ([in the C++ sense][trivial]). A frozen struct containing a class reference or closure still requires reference-counting when copied and when it goes out of scope.

- **It does not necessarily have a known size or alignment.** A generic frozen struct's layout might depend on the generic argument provided at run time.

- **Even concrete instantiations may not have a known size or alignment.** A frozen struct with a field that's a *non-*frozen, has a size that may not be known until run time.

- **It is not guaranteed to use the same layout as a C struct with a similar "shape".** If such a struct is necessary, it should be defined in a C header and imported into Swift.

- **The fields are not guaranteed to be laid out in declaration order.** The compiler may choose to reorder fields, for example to minimize padding while satisfying alignment requirements.

That said, the compiler is allowed to use its knowledge of the struct's contents and layout to derive any of these properties. For instance, the compiler can statically prove that copying `Point` is "trivial", because each of its members has a statically-known type that is "trivial". However, depending on this at the language level is not supported, with two exceptions:

1. The run-time memory layout of a struct with a single field is always identical to the layout of the instance property on its own, whether the struct is declared `@frozen` or not. This has been true since Swift 1. (This does not extend to the calling convention, however. If the struct is not frozen, it will be passed indirectly even if its single field is frozen and thus _can_ be passed directly)

2. The representation of `nil` for any type that is "nullable" in C / Objective-C is the same as the representation of `nil` or `NULL` in those languages. This includes class references, class-bound protocol references, class type references, unsafe-pointers, `@convention(c)` functions, `@convention(block)` functions, OpaquePointer, Selector, and NSZone. This has been true since Swift 3 ([SE-0055][]).

This proposal does not change either of these guarantees.

  [trivial]: https://docs.microsoft.com/en-us/cpp/cpp/trivial-standard-layout-and-pod-types
  [SE-0055]: https://github.com/swiftlang/swift-evolution/blob/master/proposals/0055-optional-unsafe-pointers.md

### `@frozen` on `enum` types

Marking an enum as `@frozen` will similarly allow the compiler to optimize away runtime calls.

In addition, marking an enum as frozen restores the ability of a library user to exhaustively switch over that enum without an `@unknown default:`, because it guarantees no further cases will be added.

Once frozen, all changes made to an enum's cases affect its ABI.

## Naming

`@frozen` was used here, as originally used in [SE-0192][], for both fixed-layout structs and enums. 

- While they share a fixing of their layout going forward, declaring an enum as frozen has additional meaning to _users_ of the library, whereas a frozen struct is an implementation detail of the library.

- It would be reasonable to use `@frozen` to describe an enum without associated values that promises not to *add* any cases with associated values. This would allow for similar optimization as `@frozen` on a struct, while still not declaring that the enum is frozen.

- Since this feature is only relevant for libraries with binary compatibility concerns, it may be more useful to tie its syntax to `@available` from the start, as with [SE-0193][].

But that said, `@frozen` still has the right *connotations* to describe a type whose stored instance properties or cases can no longer be changed. So the final candidates for names are:

- `@frozen` for both, to match the term used in [SE-0192][]
- `@frozen` for enums but `@fixedContents`, or something else, specific to structs
- `@available(*, frozen)`, to leave space for e.g. "frozen as of dishwasherOS 5"

Other names are discussed in "Alternatives considered"

## Comparison with other languages

### C

C structs have a very simple layout algorithm, which guarantees that fields are laid out in source order with adjustments to satisfy alignment restrictions. C has no formal notion of non-public fields; if used in an ABI-stable interface, the total size and alignment of a struct and its fields must not change. On the other hand, because the layout algorithm is so well-known and all fields are "trivial", C struct authors can often get away with adding one or more "reserved" fields to the end of their struct, and then renaming and possibly splitting those fields later to get the effect of adding new fields. A fixed-contents struct in Swift can get the same effect with a private field that backs public computed properties.

When a C library author wants to reserve the right to *arbitrarily* change the layout of a struct, they will usually not provide a definition for the struct at all. Instead, they will forward-declare the type and vend pointers to instances of the struct, usually allocated on the heap. Clients have to use "getter" and "setter" functions to access the fields of the struct. Non-fixed-contents Swift structs work in much the same way, except that the compiler can be a little more efficient about it.

Finally, C has an interesting notion of a *partially* fixed-contents struct. If a developer knows that two C structs start with the same fields and have the same alignment, they can reinterpret a pointer to one struct as a pointer to the other, as long as they only access those shared fields. This can be used to get fast access to some fields while going through indirect "getter" functions for others. Swift does not have any equivalent feature at this time.


### C++

C++ shares many of the same features as C, except that C++ does not guarantee that all fields are laid out in order unless they all have the same access control. C++ access control does not otherwise affect the ABI of a type, at least not with regards to layout; if a private field is removed from a struct (or class), the layout changes.

C++ structs may also contain non-trivial fields. Rather than client code being able to copy them element-by-element, C++ generates a *copy constructor* that must be called whenever a struct value is copied (including when it is passed to a function). Copy constructors may also be written explicitly, in which case they may run arbitrary code; this affects what optimizations a C++ compiler can do. However, if the compiler can *see* that the copy constructor has not been customized, it may be able to optimize copies similarly to Swift.

(This is overly simplified, but sufficient for this comparison.)


### Rust

Rust likewise has a custom layout algorithm for the fields of its structs. While Rust has not put too much effort into having a stable ABI (other than interoperation with C), their current design behaves a lot like C's in practice: layout is known at compile-time, and changing a struct's fields will change the ABI.

(there is an interesting post a couple of years ago about [optimizing the layout of Rust structs, tuples, and enums][rust].)

Like C++ (and Swift), the fields of a Rust struct may be non-trivial to copy; unlike C++ (or Swift), Rust simply does not provide an implicit copy operation if that is the case. In fact, in order to preserve source compatibility, the author of the struct must *promise* that the type will *always* be trivial to copy by implementing a particular trait (protocol), if they want to allow clients to copy the struct through simple assignment.

Rust also allows explicitly opting into C's layout algorithm. We could add that to Swift too if we wanted; for now we continue encouraging people to define such structs in C instead.

[rust]: http://camlorn.net/posts/April%202017/rust-struct-field-reordering.html


### VM/JIT/Interpreted languages

Languages where part of the compilation happens at run time are largely immune to these issues. In theory, the layout of a "struct" can be changed at any time, and any client code will automatically be updated to deal with that, because the runtime system will know either to handle the struct indirectly, or to emit fresh JITed code that can dynamically access the struct directly.

### Objective-C

Objective-C uses C structs for its equivalent of fixed-contents structs, but *non-*fixed-contents structs are instead represented using classes, often immutable classes. (An immutable class sidesteps any questions about value semantics as long as you don't test its identity, e.g. with `===`.) The presence of these classes in Apple's frameworks demonstrates the need for non-fixed-contents structs in Swift.

Old versions of Objective-C (as in, older than iOS) had the same restrictions for the instance variables of a class as C does for the fields of a struct: any change would result in a change in layout, which would break any code that accessed instance variables directly. This was largely acceptable for classes that didn't need to be subclassed, but those that did had to keep their set of instance variables fixed. (This led to the same sort of "reserved" tricks used by C structs.) This restriction was lifted at the cost of some first-use set-up time with Apple's "non-fragile Objective-C runtime", introduced with iOS and with the 64-bit version of Mac OS X; it is now also supported in the open-source GNUstep runtime.


### Other languages

Many other languages allow defining structs, but most of them either don't bother to define a stable ABI or don't allow modifying the fields of a struct once it becomes part of an ABI. Haskell and Ocaml fall into this bucket.


## Source compatibility

This change does not impact source compatibility. It has always been a source-compatible change to modify a struct's non-public stored instance properties, and to add a new public stored instance property, as long as the struct's existing API does not change (including any initializers).


## Effect on ABI stability

Currently, the layout of a public struct is known at compile time in both the defining library and in its clients. For a library concerned about binary compatibility, the layout of a non-fixed-contents struct must not be exposed to clients, since the library may choose to add new stored instance properties that do not fit in that layout in its next release, or even remove existing properties as long as they are not public.

These considerations should not affect libraries shipped with their clients, including SwiftPM packages. These libraries should always have library evolution mode turned off, indicating that the compiler is free to optimize based on the layout of a type (because the library won't change).


## Effect on Library Evolution

Both structs and enums can gain new protocol conformances or methods, even when `@frozen`. Binary compatibility only affects additions of fields or cases.

The set of binary-compatible changes to a struct's stored instance properties is described above. There are no binary-compatible changes to an enum's cases.

Taking an existing struct or enum and marking it `@frozen` is something we'd like to support without breaking binary compatibility, but there is no design for that yet.

Removing `@frozen` from a type is not allowed; this would break any existing clients that rely on the existing layout.

### Breaking the contract

Because the compiler uses the set of fields in a fixed-contents struct to determine its in-memory representation and calling convention, adding a new field or removing `@frozen` from a type in a library will result in "undefined behavior" from any client apps that have not been recompiled. This means a loss of memory-safety and type-safety on par with a misuse of "unsafe" types, which would most likely lead to crashes but could lead to code unexpectedly being executed or skipped. In short, things would be very bad.

Some ideas for how to prevent library authors from breaking the rules accidentally are discussed in "Compatibility checking" under "Future directions".


## Future directions

### Compatibility checking

Of course, the compiler can't stop a library author from modifying the fields of a fixed-contents struct, even though that will break binary compatibility. We already have two ideas on how we could catch mistakes of this nature:

- A checker that can compare APIs across library versions, using swiftmodule files or similar.

- Encoding the layout of a type in a symbol name. Clients could link against this symbol so that they'd fail to launch if it changes, but even without that an automated system could check the list of exported symbols to make sure nothing was removed.


### Allowing observing accessors

The limitation on observing accessors for stored instance properties is an artificial one; clients could certainly access the field directly for "read" accesses but go through a setter method for "write" accesses. However, the expected use for `@frozen` is for structs that need to be as efficient as C structs, and that includes being able to do direct stores into public instance properties. The effect of observing accessors can be emulated in a fixed-contents struct with a private stored property and a public computed property.

If we wanted to add this later, the way we would probably implement it is by saying that accessors for stored instance properties of `@frozen` structs must be marked `@inlinable`, with all of the implications that has for being able to make changes in the future.


## Alternatives considered

### Annotation syntax

#### Naming

`@frozen` comes from [SE-0192][], where it was originally used as a term for for enums that will not gain new cases.

Other than `@frozen`, most names have revolved around the notion of "fixed" somehow: `@fixedContents`, `@fixedLayout`, or even just `@fixed`. We ultimately chose `@frozen` as there is a strong similarity between this feature for both enums and structs.

`final` was suggested at one point, but the meaning is a little far from that of `final` on classes.

#### Modifier or attribute?

This proposal suggests a new *attribute*, `@frozen`; it could also be a modifier `frozen`, implemented as a context-sensitive keyword. Because this annotation only affects the type's *implementation* rather than anything about its use, an attribute seemed more appropriate.

### Command-line flag naming

As mentioned above, the current spelling of the flag`-enable-library-evolution`.

The term "resilience" has been an umbrella name for the Swift project's efforts to support evolving an API without breaking binary compatibility, so for some time the flag was flag `-enable-resilience`. But it isn't well-known outside of the Swift world. It's better to have this flag be more self-explanatory.

Alternate bikeshed colors:

- `-enable-abi-stability`
- `-enable-stable-abi`
- just `-stable-abi`

In practice, the precise name of this flag won't be very important, because (1) the number of people outside of Apple making libraries with binary stability concerns won't be very high, and (2) most people will be building those libraries through Xcode or SwiftPM, which will have its own name for this mode. But naming the *feature* is still important.

### "No reordering" vs. "No renaming"

As written, this proposal disallows *reordering* field of a fixed-contents struct. This matches the restrictions of C structs, but is different from everything else in Swift, which freely allows reordering. Why?

Let's step back and remember our goal: a stable, unchanging ABI for a fixed-contents struct. That means that whatever changes are made to the struct, the in-memory representation and the rest of the ABI must be consistent across library versions. We can think of this as a function that takes fields and produces a layout:

```swift
struct Field {
  var name: String
  var type: ValueType
}
typealias Offset = Int
func layoutForFixedContentsStruct(fields: [Field]) -> [Offset: Field]
```

To make things simple, let's assume all the fields in the struct have the same type, like this PolarPoint:

```swift
@frozen public struct PolarPoint {
  // Yes, these have different access levels.
  // You probably wouldn't do that in real life,
  // but it's needed for this contrived example.
  public var radius: Double
  private var angle: Double

  public init(x: Double, y: Double) {
    self.radius = sqrt(x * x, y * y)
    self.angle = atan2(y, x)
  }

  public var x: Double {
    return radius * cos(angle)
  }
  public var y: Double {
    return radius * sin(angle)
  }
}
```

We have two choices for how to lay out the PolarPoint struct:

- `[0: radius, 8: angle]`
- `[0: angle, 8: radius]`

*It's important that everyone using the struct agrees on which layout we're using.* Otherwise, an angle could get misinterpreted as a radius when someone outside the module tries to access the `radius` property.

So how do we decide? We have three choices:

1. Sort the fields by declaration order.
2. Sort the fields by name.
3. Do something clever with access levels.

Which translates to three possible restrictions:

1. Fields of a fixed-contents struct may not be reordered, even non-public ones.
2. Fields of a fixed-contents struct may not be renamed, even non-public ones.
3. Copying the struct requires calling a function because we can't see the private fields.

(3) is a non-starter, since it defeats the performance motivation for `@frozen`. That leaves (1) and (2), and we decided it would be *really weird* if renaming the private property `angle` to `theta` changed the layout of the struct. So the restriction on reordering fields was considered the least bad solution. It helps that it matches the behavior of C, Rust, and other languages.


### Fixed-contents by default

No major consideration was given towards making structs fixed-contents by default, with an annotation to make them "changeable". While this is the effective behavior of other languages, the library design philosophy in Swift is that changes that don't break source compatibility shouldn't break binary compatibility whenever possible. Furthermore, while this may be the behavior of other languages, it's also been a longstanding *complaint* in other languages like C. We want to make structs in ABI-stable libraries as easy to evolve over time as classes are in Objective-C, while still providing an opt-out for structs that have high performance requirements.


### Binary stability mode by default

Rather than introduce a new compiler flag, Swift could just treat *all* library code as if it defined ABI-stable libraries. However, this could result in a noticeable performance hit when it isn't really warranted; if a library does *not* have binary-compatibility concerns (say, if it is distributed with the app that uses it), there is no benefit to handling structs and enums indirectly. In practice, it's likely we'd start seeing people put `@frozen` in their source packages. Keeping this as a flag means that the notion of a "frozen" type can be something that most users and even most library authors don't have to think about.

(Note that we'd love to have this be true for inlinable functions too, but that's a lot trickier on the implementation side, which is why `@inlinable` was considered worth adding in [SE-0193][] despite the additional complexity.)
