# Hashable Enhancements

* Proposal: [SE-0206](0206-hashable-enhancements.md)
* Authors: [Karoy Lorentey](https://github.com/lorentey), [Vincent Esche](https://github.com/regexident)
* Review Manager: [Joe Groff](https://github.com/jckarter)
* Status: **Implemented (Swift 4.2)**
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-se-0206-hashable-enhancements/11675/115)
* Implementation:<br> 
    [apple/swift#14913](https://github.com/apple/swift/pull/14913) (standard library, underscored),<br>
    [apple/swift#16009](https://github.com/apple/swift/pull/16009) (`Hasher` interface),<br>
    [apple/swift#16073](https://github.com/apple/swift/pull/16073) (automatic synthesis, de-underscoring)<br>
* Previous Revision: [1](https://github.com/swiftlang/swift-evolution/blob/f5a020ec79cdb64fc8700af91b1a1ece2d2fb141/proposals/0206-hashable-enhancements.md)

<!--
*During the review process, add the following fields as needed:*

* Decision Notes: [Rationale](https://forums.swift.org/), [Additional Commentary](https://forums.swift.org/)
* Bugs: [SR-NNNN](https://bugs.swift.org/browse/SR-NNNN), [SR-MMMM](https://bugs.swift.org/browse/SR-MMMM)
* Previous Proposal: [SE-XXXX](XXXX-filename.md)
-->

## Table of contents

* [Introduction](#intro)
* [Motivation](#why)
    - [The status quo](#status-quo)
    - [Universal hash functions](#universal-hashing)
* [Proposed solution](#proposed-solution)
    - [The `Hasher` struct](#hasher)
    - [The `hash(into:)` requirement](#hash-into)
* [Detailed design](#detailed-design)
    - [`Hasher`](#hasher-details)
    - [`Hashable`](#hashable-details)
* [Source compatibility](#source-compatibility)
* [Effect on ABI stability](#abi)
* [Effect on API resilience](#resilience)
* [Alternatives considered](#alternatives)
    - [Leaving `Hashable` as is](#leave-hashable-alone)
    - [Defining a new protocol](#new-protocol)
    - [Making `hash(into:)` generic over a `Hasher` protocol](#generic-hasher)
    - [Change `hash(into:)` to take a closure instead of a new type](#closure-hasher)

## <a name="intro">Introduction</a>

This proposal introduces a new `Hasher` type representing the standard
library's universal hash function, and it extends the `Hashable`
protocol with a new `hash(into:)` requirement that expresses hashing
in terms of `Hasher`. This new requirement is intended to replace the
old `hashValue` property, which is deprecated.

Switching to `hash(into:)` moves the choice of a hash function out of
`Hashable` implementations, and into the standard library. This makes
it considerably easier to manually conform types to `Hashable` -- the
task of writing a custom implementation reduces to identifying the
parts of a type that should contribute to the hash value.

Standardizing on a single, high-quality hash function greatly improves
the reliability of `Set` and `Dictionary`. The hash function can be
specially selected and tuned to work well with the hash tables used by
these collections, preventing hash collision patterns that would break
the expected performance of common operations.

`Hasher` is a resilient struct, enabling future versions of the
standard library to improve the hash function without the need to
change (or even recompile) existing code that implements `Hashable`.
Not baking any particular hash function into `Hashable` types is
especially important in case a weakness is found in the current
algorithm.

Swift-evolution thread: [Combining Hashes](https://forums.swift.org/t/combining-hashes/9082)

[SE-0185]: 0185-synthesize-equatable-hashable.md
[SE-0143]: 0143-conditional-conformances.md

## <a name="why">Motivation</a>

The Swift Standard Library includes two general-purpose hashing
collections, `Set` and `Dictionary`. These collections are built
around hash tables, whose performance is critically dependent on the
expected distribution of the elements stored in them, along with the
quality of the hash function that is used to derive bucket indices for
individual elements.

With a good hash function, simple lookups, insertions and removals
take constant time on average. However, when the hash function isn't
carefully chosen to suit the data, the expected time of such
operations can become proportional to the number of elements stored in
the table. If the table is large enough, such a regression can easily
lead to unacceptable performance. When they're overwhelmed with hash
collisions, applications and network services may stop processing new
events for prolonged periods of time; this can easily be enough to
make the app unresponsive or to bring down the service.

### <a name="status-quo">The status quo</a>

Since Swift version 1.0, `Hashable` has had a single requirement on
top of `Equatable`: the `hashValue` property. `hashValue` looks
deceptively simple, but implementing it is unreasonably hard: Not only
do we need to decide which components of our type should be involved
in hashing, but we also have to come up with a way to somehow distill
these components down into a single integer value.  The API is
essentially asking us to implement a new hash function, from scratch,
every single time we conform to `Hashable`.

Given adequate documentation, it is reasonable to expect that an
experienced programmer implementing a custom type would be able to
identify what parts need to be hashed. On the other hand, implementing
a good hash function requires careful consideration and specialist
knowledge. It is unreasonable to expect Swift programmers to invest
time and effort to get this right for every `Hashable` type out
there.

For example, consider the code below, extracted directly from the
documentation of Swift 4.1's `Hashable`. Is this a good implementation
of `hashValue`?

```swift
struct GridPoint {
  var x: Int
  var y: Int
}

extension GridPoint: Hashable {
  var hashValue: Int {
    return x.hashValue ^ y.hashValue &* 16777619
  }

  static func == (lhs: GridPoint, rhs: GridPoint) -> Bool {
    return lhs.x == rhs.x && lhs.y == rhs.y
  }
}
```

The answer is that it depends; while the hash values it produces are
perfectly fine if `x` and `y` are expected to be small integers coming
from trusted sources, this hash function does have some undesirable
properties:

1.  The clever bit manipulations make it hard to understand what the
    code does, or how it works. For example, can you tell what makes
    16777619 a better choice for the multiplier than, say, 16777618?
    What are the precedence rules for `^` and `&*` again? What's with
    the ampersand, anyway?

    We just wanted to use `GridPoint` values as keys in a
    `Dictionary`, but first, we need to spend a couple of hours
    learning about bitwise operations, integer overflows and the
    exciting properties of coprime numbers.

    (For what it's worth, the magic constant used in the example above
    is the same as the one used for the 32-bit version of the [FNV-1a]
    hashing algorithm, which uses a similar (if a little more
    complicated) method to distill arbitrary byte sequences down into
    a single integer.)

2.  It is trivially easy to construct an arbitrarily large set of
    `GridPoint` values that aren't equal, but have the same hash
    value. If the values come from an untrusted source, they may
    sometimes be deliberately chosen to induce collisions.

3.  The hash function doesn't do a particularly great job at mixing up
    the input data; the hash values it produces tend to form long
    chains of sequential integer clusters. While these aren't as bad
    as hash collisions, some hash table operations can slow down
    drastically when such clusters are present. (In Swift 4.1, `Set`
    and `Dictionary` use open addressing with linear probing, and they
    have to do some clever postprocessing of hash values to get rid of
    such patterns.)

It seems desirable for the standard library to provide better guidance
for people implementing `hashValue`.

### <a name="universal-hashing">Universal hash functions</a>

With [SE-0185], Swift 4.1 introduced compiler support for automatic
synthesis of `Hashable` conformance for certain types. For example,
the `GridPoint` struct above can be made to conform to `Hashable`
without explicitly defining `hashValue` (or `==`):

```swift
struct GridPoint: Hashable {
  var x: Int
  var y: Int
  
  // hashValue and == are automatically synthesized by the compiler
}
```

[SE-0185] did not specify a hash function to be used for such
conformances, leaving it as an implementation detail of the compiler
and the standard library. Doing this well requires the use of a hash
function that works equally well on any number of components,
regardless of their expected distributions.

[SE-0185]: https://github.com/swiftlang/swift-evolution/blob/master/proposals/0185-synthesize-equatable-hashable.md

Luckily, this problem has occurred in other contexts before, and there
is an extensive list of hash functions that have been designed for
exactly such cases: [Foller-Noll-Vo][FNV-1a], [MurmurHash],
[CityHash], [SipHash], and [HighwayHash] are just a small selection of
these. The last two algorithms include some light cryptographic
elements so that they provide a level of protection against deliberate
hash collision attacks. This makes them a better choice for
general-purpose hashed collections like `Set` and `Dictionary`.

[FNV-1a]: http://www.isthe.com/chongo/tech/comp/fnv/index.html
[MurmurHash]: https://github.com/aappleby/smhasher
[CityHash]: https://github.com/google/cityhash
[SipHash]: https://131002.net/siphash/
[HighwayHash]: https://github.com/google/highwayhash

Since [SE-0185] required the standard library to implement a
high-quality universal hash function, it seems like a good idea to
expose it as public API, so that manual `Hashable` implementations can
take advantage of it, too.

Universal hash functions work by maintaining some internal state --
this can be as simple as a single 32/64-bit integer value (for
e.g. [FNV-1a]), but it is usually much wider than that. For example,
[SipHash] maintains a state of 256 bits, while [HighwayHash] uses 1024
bits. 

## <a name="proposed-solution">Proposed solution</a>

We solve `Hashable`'s implementation problems in two parts. First, we
make the standard library's hash function public. Second, we replace
`hashValue` with a requirement that is designed specifically to
eliminate the guesswork from manual `Hashable` implementations.

### <a name="hasher">The `Hasher` struct</a>

We propose to expose the standard library's standard hash function as
a new, public struct type, called `Hasher`. This new struct captures
the state of the hash function, and provides the following operations:

1. An initializer to create an empty state. To make hash values less
   predictable, the standard hash function uses a per-execution random
   seed, so that generated hash values will be different in each
   execution of a Swift program.

    ```swift
    public struct Hasher {
      public init()  // Uses a per-execution random seed value
    }
    ```

2. Operations to feed new bytes to the hash function, mixing them
   into its state:
   
    ```swift
    extension Hasher {
      public mutating func combine(bytes buffer: UnsafeRawBufferPointer)
      public mutating func combine<H: Hashable>(_ value: H)
    }
    ```
    
    `combine(bytes:)` is the most general operation, suitable for use
    when the bytes to be hashed are available as a single contiguous
    region in memory. 
    
    `combine(_:)` is a convenience operation that can take any
    `Hashable` value; we expect this will be more frequently
    useful. (We'll see how this is implemented in the next section.)

3. An operation to finalize the state, extracting the hash value from it.
   
    ```swift
    extension Hasher {
      public __consuming func finalize() -> Int
    }
    ```

   Finalizing the hasher state consumes it; it is illegal to call
   `combine` or `finalize` on a hasher you don't own, or on one that's
   already been finalized.

For example, here is how one may use `Hasher` as a standalone type:

```swift
var hasher = Hasher()        // Initialize state, usually by random seeding
hasher.combine(23)           // Mix in several integer's worth of bytes
hasher.combine(42)
print(hasher.finalize())     // Finalize the state and return the hash
```

Within the same execution of a Swift program, `Hasher`s are guaranteed
to return the same hash value in `finalize()`, as long as they are fed
the exact same sequence of bytes. (Note that the order of `combine`
operations matters; swapping the two integers above will produce a
completely different hash.)

However, `Hasher` may generate entirely different hash values in other
executions, *even if it is fed the exact same byte sequence*. This
randomization is a critical feature, as it makes it much harder for
potential attackers to predict hash values. `Hashable` has always been
documented to explicitly allow such nondeterminism:

> - Important: Hash values are not guaranteed to be equal across
>   different executions of your program. Do not save hash values to
>   use in a future execution.
>
> <cite>-- `Hashable` documentation</cite>

(Random seeding can be disabled by setting a special environment
variable; see [Effect on ABI stability](#abi) for details.)

The choice of which hash function `Hasher` implements is an
implementation detail of the standard library, and may change in any
new release. This includes the size and internal layout of `Hasher`
itself. (The current implementation uses SipHash-1-3 with 320 bits of
state.)

### <a name="hash-into">The `hash(into:)` requirement</a>

Introducing `Hasher` is a big improvement, but it's only half of the
story: `Hashable` itself needs to be updated to make better use of it.

We propose to change the `Hashable` protocol by adding a new
`hash(into:)` requirement:

```swift
public protocol Hashable: Equatable {
  var hashValue: Int { get }
  func hash(into hasher: inout Hasher)
}
```

At the same time, we deprecate custom implementations of the
`hashValue` property. (Please see the section on [Source
compatibility](#source-compatibility) on how we'll do this without
breaking code written for previous versions of Swift.) In some future
language version, we intend to convert `hashValue` to an extension
method.

To make it easier to express `hash(into:)` in terms of `Hashable`
components, `Hasher` provides a variant of `combine` that simply calls
`hash(into:)` on the supplied value:

```swift
extension Hasher {
  @inlinable
  public mutating func combine<H: Hashable>(_ value: H) {
    value.hash(into: &self)
  }
}
```

This is purely for convenience; `hasher.combine(foo)` is slightly
easier to type than `foo.hash(into: &hasher)`.


At first glance, it may not be obvious why we need to replace
`hashValue`. After all, `Hasher` can be used to take the guesswork out
of its implementation:

```swift
extension GridPoint: Hashable {
  var hashValue: Int { 
    var hasher = Hasher()
    hasher.combine(x)
    hasher.combine(y)
    return hasher.finalize()
  }
}
```

What's wrong with this? What makes `hash(into:)` so much better that's
worth the cost of a change to a basic protocol?

* **Better Discoverability** -- With `hashValue`, you need to know
  about `Hasher` to make use of it: the API does not direct you to do
  the right thing. Worse, you have to do extra busywork by manually
  initializing and finalizing a `Hasher` instance directly in your
  `hashValue` implementation. Compare the code above to the
  `hash(into:)` implementation below:
  
  ```swift
  extension GridPoint: Hashable {
    func hash(into hasher: inout Hasher) {
      hasher.combine(x)
      hasher.combine(y)
    }
  }
  ```
  
  This is nice and easy, with minimal boilerplate. `Hasher` is part of
  the function signature; people who need to implement `hash(into:)`
  are naturally guided to learn about it.
  
* **Guaranteed Dispersion Quality** -- Keeping the existing
  `hashValue` interface would mean that there was no way for `Set` and
  `Dictionary` to guarantee that `Hashable` types produce hash values
  with good enough dispersion. Therefore, these collections would need
  to keep postprocessing hash values. We'd like to eliminate
  postprocessing overhead for types that upgrade to `Hasher`.
  
* **Hasher Customizability** -- `hash(into:)` moves the initialization
  of `Hasher` out of `Hashable` types, and into hashing
  collections. This allows us to customize `Hasher` to the needs of
  each hashing data structure. For example, the stdlib could start
  using a different seed value for every new `Set` and `Dictionary`
  instance; this somewhat improves reliability by making hash values
  even less predictable, but (probably more importantly), it
  drastically improves the performance of some relatively common
  operations involving [copying data between `Set`/`Dictionary`
  instances][quadratic-copy].
  
* **Better Performance** -- Similarly, `hash(into:)` moves the
  finalization step out of `Hashable`. Finalization is a relatively
  expensive operation; for example, in SipHash-1-3, it costs three
  times as much as a single 64-bit `combine`. Repeating it for every
  single component of a composite type would make hashing unreasonably
  slow.
  
  For example, consider the `GridRectangle` type below:
  ```swift
  struct GridRectangle {
    let topLeft: GridPoint
    let bottomRight: GridPoint
  }
  ```
    
  With `hashValue`, its `Hashable` implementation would look like this:
  ```swift
  extension GridRectangle: Hashable {
    var hashValue: Int { // SLOW, DO NOT USE
      var hasher = Hasher()
      hasher.combine(bits: topLeft.hashValue) 
      hasher.combine(bits: bottomRight.hashValue)
      return hasher.finalize()
    }
  }
  ```

  Both of the `hashValue` invocations above create and finalize
  separate hashers. Assuming finalization takes three times as much
  time as a single combine call (and generously assuming that initialization
  is free) this takes 15 combines' worth of time:
  
  ```
   1   hasher.combine(topLeft.hashValue)
   1       hasher.combine(topLeft.x)     (in topLeft.hashValue)
   1       hasher.combine(topLeft.y)
   3       hasher.finalize()
   1   hasher.combine(bottomRight.hashValue)
   1       hasher.combine(bottomRight.x) (in bottomRight.hashValue)
   1       hasher.combine(bottomRight.y)
   3       hasher.finalize()
   3   hasher.finalize()
  ---
  15
  ```

  Switching to `hash(into:)` gets us the following code:

  ```swift
  extension GridRegion: Hashable {
    func hash(into hasher: inout Hasher) {
      hasher.combine(topLeft)
      hasher.combine(bottomRight)
    }
  }
  ```
  
  This reduces the cost of hashing to just four combines and a single
  finalization, which takes less than half the time of our original
  approach:
  
  ```
   1   hasher.combine(topLeft.x)      (in topLeft.hash(into:))
   1   hasher.combine(topLeft.y)
   1   hasher.combine(bottomRight.x)  (in bottomRight.hash(into:))
   1   hasher.combine(bottomRight.y)
   3   hasher.finalize()             (outside of GridRectangle.hash(into:))
  ---
   7
  ```
  
Switching to `hash(into:)` gets us more robust hash values faster, and
with cleaner, simpler code.


[quadratic-copy]: https://bugs.swift.org/browse/SR-3268


## <a name="detailed-design">Detailed design</a>

### <a name="hasher-details">`Hasher`</a>

Add the following type to the standard library:

```swift
/// Represents the universal hash function used by `Set` and `Dictionary`.
///
/// The hash function is a mapping from a 128-bit seed value and an 
/// arbitrary sequence of bytes to an integer hash value. The seed value
/// is specified during `Hasher` initialization, while the byte sequence
/// is fed to the hasher using a series of calls to mutating `combine`
/// methods. When all bytes have been fed to the hasher, the hash value 
/// can be retrieved by calling `finalize()`:
///
///     var hasher = Hasher()
///     hasher.combine(23)
///     hasher.combine("Hello")
///     let hashValue = hasher.finalize()
///
/// The underlying hash algorithm is designed to exhibit avalanche
/// effects: slight changes to the seed or the input byte sequence
/// will produce drastic changes in the generated hash value.
///
/// - Note: `Hasher` is usually randomly seeded, which means it will return
///   different values on every new execution of your program. The hash 
///   algorithm implemented by `Hasher` may itself change between 
///   any two versions of the standard library. Do not save or otherwise 
///   reuse hash values across executions of your program.
public struct Hasher {
  /// Initialize a new hasher using a per-execution seed value.
  /// The seed is set during process startup, usually from a high-quality 
  /// random source.
  public init()
  
  /// Feed `value` to this hasher, mixing its essential parts into
  /// the hasher state.
  @inlinable
  public mutating func combine<H: Hashable>(_ value: H) {
    value.hash(into: &self)
  }

  /// Feed the raw bytes in `buffer` into this hasher, mixing its bits into
  /// the hasher state.
  public mutating func combine(bytes buffer: UnsafeRawBufferPointer)
  
  /// Finalize the hasher state and return the hash value.
  ///
  /// Finalizing consumes the hasher: it is illegal to finalize a hasher you
  /// don't own, or to perform operations on a finalized hasher. (These may
  /// become compile-time errors in the future.)
  public __consuming func finalize() -> Int
}
```

## <a name="hashable-details">`Hashable`</a>

Change the `Hashable` protocol as follows.

```swift
/// A type that can be hashed into a `Hasher`.
///
/// You can use any type that conforms to the `Hashable` protocol in a set or as
/// a dictionary key. Many types in the standard library conform to `Hashable`:
/// Strings, integers, floating-point and Boolean values, and even sets are
/// hashable by default. Some other types, such as optionals, arrays and ranges
/// automatically become hashable when their type arguments implement the same.
///
/// Your own custom types can be hashable as well. When you define an
/// enumeration without associated values, it gains `Hashable` conformance
/// automatically, and you can add `Hashable` conformance to your other custom
/// types by implementing the `hash(into:)` function. For structs whose stored
/// properties are all `Hashable`, and for enum types that have all-`Hashable`
/// associated values, the compiler is able to provide an implementation of
/// `hash(into:)` automatically.
///
/// Hashing a value means feeding its essential components into a hash function,
/// represented by the `Hasher` type. Essential components are those that
/// contribute to the type's implementation of `Equatable`. Two instances that
/// are equal must feed the same values to `Hasher` in `hash(into:)`, in the
/// same order.
///
/// Conforming to the Hashable Protocol
/// ===================================
///
/// To use your own custom type in a set or as the key type of a dictionary,
/// add `Hashable` conformance to your type. The `Hashable` protocol inherits
/// from the `Equatable` protocol, so you must also satisfy that protocol's
/// requirements.
///
/// A custom type's `Hashable` and `Equatable` requirements are automatically
/// synthesized by the compiler when you declare `Hashable` conformance in the
/// type's original declaration and your type meets these criteria:
///
/// - For a `struct`, all its stored properties must conform to `Hashable`.
/// - For an `enum`, all its associated values must conform to `Hashable`. (An
///   `enum` without associated values has `Hashable` conformance even without
///   the declaration.)
///
/// To customize your type's `Hashable` conformance, to adopt `Hashable` in a
/// type that doesn't meet the criteria listed above, or to extend an existing
/// type to conform to `Hashable`, implement the `hash(into:)` function in your
/// custom type. To ensure that your type meets the semantic requirements of the
/// `Hashable` and `Equatable` protocols, it's a good idea to also customize
/// your type's `Equatable` conformance to match the `hash(into:)` definition.
///
/// As an example, consider a `GridPoint` type that describes a location in a
/// grid of buttons. Here's the initial declaration of the `GridPoint` type:
///
///     /// A point in an x-y coordinate system.
///     struct GridPoint {
///         var x: Int
///         var y: Int
///     }
///
/// You'd like to create a set of the grid points where a user has already
/// tapped. Because the `GridPoint` type is not hashable yet, it can't be used
/// as the `Element` type for a set. To add `Hashable` conformance, provide an
/// `==` operator function and a `hash(into:)` function.
///
///     extension GridPoint: Hashable {
///         func hash(into hasher: inout Hasher) {
///             hasher.combine(x)
///             hasher.combine(y)
///         }
///
///         static func == (lhs: GridPoint, rhs: GridPoint) -> Bool {
///             return lhs.x == rhs.x && lhs.y == rhs.y
///         }
///     }
///
/// The `hash(into:)` property in this example feeds the properties `x` and `y`
/// to the supplied hasher; these are the same properties compared by the
/// implementation of the `==` operator function.
///
/// Now that `GridPoint` conforms to the `Hashable` protocol, you can create a
/// set of previously tapped grid points.
///
///     var tappedPoints: Set = [GridPoint(x: 2, y: 3), GridPoint(x: 4, y: 1)]
///     let nextTap = GridPoint(x: 0, y: 1)
///     if tappedPoints.contains(nextTap) {
///         print("Already tapped at (\(nextTap.x), \(nextTap.y)).")
///     } else {
///         tappedPoints.insert(nextTap)
///         print("New tap detected at (\(nextTap.x), \(nextTap.y)).")
///     }
///     // Prints "New tap detected at (0, 1).")
public protocol Hashable: Equatable {
  /// The hash value.
  ///
  /// Hash values are not guaranteed to be equal across different executions of
  /// your program. Do not save hash values to use during a future execution.
  var hashValue: Int { get }
  
  /// Hash the essential components of this value into the hash function
  /// represented by `hasher`, by feeding them into it using its `combine`
  /// methods.
  ///
  /// Essential components are precisely those that are compared in the type's
  /// implementation of `Equatable`.
  func hash(into hasher: inout Hasher)
}
```

## <a name="source-compatibility">Source compatibility</a>

The introduction of the new `Hasher` type is a purely additive change.
However, adding the `hash(into:)` requirement is potentially source
breaking. To ensure we keep compatibility with code written for
previous versions of Swift, while also allowing new code to only
implement `hash(into:)`, we extend [SE-0185]'s automatic `Hashable`
synthesis to automatically derive either of these requirements when
the other has been manually implemented.

Code written for Swift 4.1 or earlier will continue to compile (in the
corresponding language mode) after this proposal is implemented. The
compiler will synthesize the missing `hash(into:)` requirement
automatically:

```
struct GridPoint41: Hashable { // Written for Swift 4.1 or below
  let x: Int
  let y: Int
  
  var hashValue: Int {
    return x.hashValue ^ y.hashValue &* 16777619
  }
  
  // Supplied automatically by the compiler:
  func hash(into hasher: inout Hasher) {
    hasher.combine(self.hashValue)
  }
}
```

The compiler will emit a deprecation warning when it needs to do this
in 4.2 mode. (However, note that code that merely uses `hashValue`
will continue to compile without warnings.)

Code written for Swift 4.2 or later should conform to `Hashable` by
implementing `hash(into:)`. The compiler will then complete the
conformance with a suitable `hashValue` definition:

```
struct GridPoint42: Hashable { // Written for Swift 4.2
  let x: Int
  let y: Int
  
  func hash(into hasher: inout Hasher) {
    hasher.combine(x)
    hasher.combine(y)
  }
  
  // Supplied automatically by the compiler:
  var hashValue: Int {
    var hasher = Hasher()
    self.hash(into: &hasher)
    return hasher.finalize()
  }
}
```

When upgrading to Swift 4.2, `Hashable` types written for earlier
versions of Swift should be migrated to implement `hash(into:)`
instead of `hashValue`. 

For types that satisfy [SE-0185]'s requirements for `Hashable`
synthesis, this can be as easy as removing the explicit `hashValue`
implementation. Note that Swift 4.2 implements conditional `Hashable`
conformances for many standard library types that didn't have it
before; this enables automatic `Hashable` synthesis for types that use
these as components.

For types that still need to manually implement `Hashable`, the
migrator can be updated to help with this process. For example, the
`GridPoint41.hashValue` implementation above can be mechanically
rewritten as follows:

```
struct GridPoint41: Hashable { // Migrated from Swift 4.1
  let x: Int
  let y: Int

  // Migrated from hashValue:
  func hash(into hasher: inout Hasher) {
    hash.combine(x.hashValue ^ y.hashValue &* 16777619)
  }
}
```

This will not provide the same hash quality as combining both members
one by one, but it may be useful as a starting point.


## <a name="abi">Effect on ABI stability</a>

`Hasher` and `hash(into:)` are additive changes that extend the ABI of
the standard library. `Hasher` is a fully resilient struct, with
opaque size/layout and mostly opaque members. (The only exception is
the generic function `combine(_:)`, which is provided as a syntactic
convenience.)

While this proposal deprecates the `hashValue` requirement, it doesn't
remove it. Types implementing `Hashable` will continue to provide an
implementation for it, although the implementation may be provided
automatically by compiler synthesis.

To implement nondeterminism, `Hasher` uses an internal seed value
initialized by the runtime during process startup. The seed is usually
produced by a random number generator, but this may be disabled by
defining the `SWIFT_DETERMINISTIC_HASHING` environment variable with a
value of `1` prior to starting a Swift process.

## <a name="resilience">Effect on API resilience</a>

Replacing `hashValue` with `hash(into:)` moves the responsibility of
choosing a suitable hash function out of `Hashable` implementations
and into the standard library, behind a resiliency boundary.

`Hasher` is explicitly designed so that future versions of the
standard library will be able to replace the hash function.
`Hashable` implementations compiled for previous versions will
automatically pick up the improved algorithm when linked with the new
release. This includes changing the size or internal layout of the
`Hasher` state itself.

(We foresee several reasons why we may want to replace the hash
function. For example, we may need to do so if a weakness is
discovered in the current function, to restore the reliability of
`Set` and `Dictionary`. We may also want to tweak the hash function to
adapt it to the special requirements of certain environments (such as
network services), or to generally improve hashing performance.)

## <a name="alternatives">Alternatives considered</a>

### <a name="leave-hashable-alone">Leaving `Hashable` as is</a>

One option that we considered is to expose `Hasher`, but to leave the
`Hashable` protocol as is. Individual `Hashable` types would be able
to choose whether or not to use it or to roll their own hash
functions.

We felt this was an unsatisfying approach; the rationale behind this
is explained in the section on [The `hash(into:)` requirement](#hash-into).

### <a name="new-protocol">Defining a new protocol</a>

There have been several attempts to fix `Hashable` by creating a new
protocol to replace it. For example, there's a prototype
implementation of a [`NewHashable` protocol][h1] in the Swift test
suite. The authors of this proposal have done their share of this,
too: Karoy has previously published an open-source [hashing
package providing an opt-in replacement for `Hashable`][h2], while
Vincent wrote [a detailed pitch for adding a `HashVisitable` protocol
to the standard library][h3a] -- these efforts were direct precursors
to this proposal.

In these approaches, the new protocol could either be a refinement of
`Hashable`, or it could be unrelated to it. Here is what a refinement
would look like:

[h1]: https://github.com/apple/swift/blob/swift-4.1-branch/validation-test/stdlib/HashingPrototype.swift
[h2]: https://github.com/attaswift/SipHash
[h3a]: https://blog.definiteloops.com/ha-r-sh-visitors-8c0c3686a46f
[h3b]: https://gist.github.com/regexident/1b8e84974da2243e5199e760508d2d25

```swift
protocol Hashable: Equatable {
  var hashValue: Int { get }
}

protocol Hashable2: Hashable {
  func hash(into hasher: inout Hasher)
}

extension Hashable2 {
  var hashValue: Int {
    var hasher = Hasher()
    hash(into: &hasher)
    return hasher.finalize()
  }
}
```

While this is a great approach for external hashing packages, we
believe it to be unsuitable for the standard library. Adding a new
protocol would add a significant new source of user confusion about
hashing, and it would needlessly expand the standard library's API
surface area.

The new protocol would need to have a new name, but `Hashable` already
has the perfect name for a protocol representing hashable things -- so
we'd need to choose an imperfect name for the "better" protocol.

While deprecating a protocol requirement is a significant change, we
believe it to be less harmful overall than leaving `Hashable`
unchanged, or trying to have two parallel protocols for the same thing.

Additionally, adding a second protocol would lead to complications
with `Hashable` synthesis. It's also unclear how `Set` and `Dictionary`
would be able to consistently use `Hasher` for their primary hashing
API. (These problems are not unsolvable, but they may involve adding
special one-off compiler support for the new protocol. For example, we
may want to automatically derive `Hashable2` conformance for all types
that implement `Hashable`.)

### <a name="generic-hasher">Making `hash(into:)` generic over a `Hasher` protocol</a>

It would be nice to allow Swift programmers to define their own hash
functions, and to plug them into any `Hashable` type:

```swift
protocol Hasher {
  func combine(bytes: UnsafeRawBufferPointer)
}
protocol Hashable {
  func hash<H: Hasher>(into hasher: inout H)
}
```

However, we believe this would add a degree of generality whose costs
are unjustifiable relative to their potential gain. We expect the
ability to create custom hashers would rarely be exercised. For
example, we do not foresee a need for adding support for custom
hashers in `Set` and `Dictionary`. On the other hand, there are
distinct advantages to standardizing on a single, high-quality hash
function:

* Adding a generic type parameter to `hash(into:)` would complicate
    the `Hashable` API.
* By supporting only a single `Hasher`, we can concentrate our efforts
    on making sure it runs fast. For example, we know that the
    standard hasher's opaque mutating functions won't ever perform any
    retain/release operations, or otherwise mutate any of the
    reference types we may encounter during hashing; describing this
    fact to the compiler enables optimizations that would not
    otherwise be possible.
* Generics in Swift aren't zero-cost abstractions. We may be tempted
    to think that we could gain some performance by plugging in a less
    sophisticated hash function. This is not necessarily the case --
    support for custom hashers comes with significant overhead that
    can easily overshadow the (slight, if any) algorithmic
    disadvantage of the standard `Hasher`.

Note that the proposed non-generic `Hasher` still has full support for
Bloom filters and other data structures that require multiple hash
functions. (To select a different hash function, we just need to
supply a new seed value.)

### <a name="closure-hasher">Change `hash(into:)` to take a closure instead of a new type</a>

A variant of the previous idea is to represent the hasher by a simple
closure taking an integer argument:

```swift
protocol Hashable {
  func hash(into hasher: (Int) -> Void)
}

extension GridPoint: Hashable {
  func hash(into hasher: (Int) -> Void) {
    hasher(x)
    hasher(y)
  }
}
```

While this is an attractively minimal API, it has problems with
granularity -- it doesn't allow adding anything less than an
`Int`'s worth of bits to the hash state.

Additionally, like generics, the performance of such a closure-based
interface would compare unfavorably to `Hasher`, since the compiler
wouldn't be able to guarantee anything about the potential
side-effects of the closure.
