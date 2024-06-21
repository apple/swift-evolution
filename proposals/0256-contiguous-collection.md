# Introduce `{Mutable}ContiguousCollection` protocol

* Proposal: [SE-0256](0256-contiguous-collection.md)
* Authors: [Ben Cohen](https://github.com/airspeedswift)
* Review Manager: [Ted Kremenek](https://github.com/tkremenek)
* Status: **Rejected**
* Previous Proposal: [SE-0237](0237-contiguous-collection.md)
* Implementation: [apple/swift#23616](https://github.com/apple/swift/pull/23616)
* Decision Notes: [Rationale](https://forums.swift.org/t/se-0256-introduce-mutable-contiguouscollection-protocol/22569/8)

## Introduction

This proposal introduces two new protocols: `ContiguousCollection`, which
refines `Collection`, and `MutableContiguousCollection`, which refines
`MutableCollection`. Both provide guaranteed access to an underlying
unsafe buffer.

## Motivation

[SE-0237](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0237-contiguous-collection.md) 
introduced two new methods, `withContiguous{Mutable}StorageIfAvailable`, to
allow generic algorithms to test for and use an underlying unsafe buffer when
it is available. This has significant performance benefit for certain
algorithms, such as `sort`, which can achieve big speedups when they
have access to the unsafe buffer, but can still operate without that fast path
if needed.

There is another class of operation that _only_ wants to be available when
there is a fast path. A good example would be a Swift-friendly wrapper for
the `vDSP` suite of algorithms. 

For example, you might want to write a convenient wrapper for `vDSP_vadd`.

```swift 
// note this is **not** a proposal about vDSP wrappers, this is just a
// simplified example :)
func dspAdd<A: Collection, B: Collection>(
  _ a: A, _ b: B, _ result: inout [Float]
) where A.Element == Float, B.Element == Float {
  let n = a.count
  // try accessing contiguous underlying buffers:
  let wasContiguous: ()?? =
    a.withContiguousStorageIfAvailable { abuf in
      b.withContiguousStorageIfAvailable { bbuf in
        vDSP_vadd(abuf.baseAddress!, 1, bbuf.baseAddress!, 1, &result, 1, UInt(n))
      }
  }
  // if they weren't contiguous, create two arrays try again
  if wasContiguous == nil || wasContiguous! == nil {
    dspAdd(Array(a), Array(b), &result)
  }
}
```

This follows a similar pattern to `sort`: provide a fast path when available,
but fall back to a slower path when it isn't.

But in the case of functions like `vDSP_vsaddi` this is very much the wrong
thing to do. These functions often operate on a thin (but very material)
performance edge over their open-coded equivalent, and allocating and
initializing two arrays purely to be able to call it would probably vastly
outweigh the speed benefits gained by using the function instead of a regular
loop. This encourages misuse by the caller, who might not realize they are
getting worse performance than if they reorganized their code.

Trapping on non-contiguous inputs would flag the problem more clearly to the
user, who could realize immediately on first use, that types like `Range`
should not be used with this API. But ideally we would use Swift's type system
to enforce this instead, guiding the user to a better solution.

## Proposed solution

Introduce two new protocols which guarantee access to a contiguous underlying
buffer.

```swift
/// A collection that supports access to its underlying contiguous storage.
public protocol ContiguousCollection: Collection
where SubSequence: ContiguousCollection {
  /// Calls a closure with a pointer to the array's contiguous storage.
  func withUnsafeBufferPointer<R>(
    _ body: (UnsafeBufferPointer<Element>) throws -> R
  ) rethrows -> R
}

/// A collection that supports mutable access to its underlying contiguous
/// storage.
public protocol MutableContiguousCollection: ContiguousCollection, MutableCollection
where SubSequence: MutableContiguousCollection {
  /// Calls the given closure with a pointer to the array's mutable contiguous
  /// storage.
  mutating func withUnsafeMutableBufferPointer<R>(
    _ body: (inout UnsafeMutableBufferPointer<Element>) throws -> R
  ) rethrows -> R
}
```

Conformances will be added for the following types:

- `Array`, `ArraySlice` and `ContiguousArray` will conform to `MutableContiguousCollection`
- `UnsafeBufferPointer` will conform to `ContiguousCollection`
- `UnsafeMutableBufferPointer` will conform to `MutableContiguousCollection`
- `Slice` will conditionally conform:
  - to `ContiguousCollection where Base: ContiguousCollection`
  - to `MutableContiguousCollection where Base: MutableContiguousCollection`

Conforming to to `ContiguousCollection` should also provide types with a default
implementation of `Collection.withContiguousStorageIfAvailable`, via an extension 
that calls `withUnsafeBufferPointer`. Same for `MutableContiguousCollection` and
`Collection.withMutableContiguousStorageIfAvailable`.

## Detailed design

The introduction of these protocols allows an appropriate constraint that would
prevent a user passing a `Range` or `Repeating` collection into our `dspAdd`
function. It also allows an easy path to a generic result buffer instead of a
concrete array; this is important as often these functions are used in a tiled
mode where you would want to repeatedly pass in an array slice. As a nice
side-benefit, it also cleans up the function implementation:

```swift
func dspAdd<A: ContiguousCollection, B: ContiguousCollection, R: MutableContiguousCollection>(
  _ a: A, _ b: B, _ result: inout R
) where A.Element == Float, B.Element == Float, R.Element == Float {
  let n = a.count
  a.withUnsafeBufferPointer { abuf in
    b.withUnsafeBufferPointer { bbuf in
      result.withUnsafeMutableBufferPointer { rbuf in
        vDSP_vadd(abuf.baseAddress!, 1, bbuf.baseAddress!, 1, rbuf.baseAddress!, 1, UInt(n))
      }
    }
  }
}
```

## Source compatibility

These are additive changes and do not affect source compatibility.

## Effect on ABI stability

These are additive changes of new protocols and so can be introduced in an
ABI-stable way. On platforms that have declared ABI stability, they will need
to have availability annotations.

## Effect on API resilience

N/A

## Alternatives considered

This is a re-pitch of these protocols. They originally appeared in SE-0237, but
were deferred pending further use cases. This proposal is motivated by such
cases.

As mentioned in the motivation, there are some alternatives available in the
absence of this protocol:

- Libraries can add their own version of the protocol, and retroactively
  conform standard library types to it. This is a workable solution, but has some downsides:
  
  - Non-standard types will not conform. Callers will need to conform these types to the
    library's protocol themselves manually. A standard library protocol is more likely to
    be adopted by other types.

  - If this pattern proves common, it will lead to multiple libraries declaring
    the same protocol.

- Libraries can use the `IfAvailable` variant, and document that using types
  without contiguous storage is inefficient. This leaves enforcing the correct
  usage to the user. This is not always possible in a generic context, where
  the calling function does not know exactly what concrete type they are using.

- Libraries can use the `IfAvailable` variant, and trap when not available.
  This could alert callers to inefficient usage on first use. For example, if
  you ever pass in a `Range`, your call will always fail, detectable by any
  testing. Some types, however, may respond to the `IfAvailable` call in some
  cases but not others. For example, a ring buffer might often not have wrapped
  around, so can provide a single contiguous buffer sometimes, but not always.
  Trapping then would lead to unpredictable crashes.

### `Array` and lazy bridging

The conformance of `Array` to these protocols presents a particular concern.

`Array` at its core is a contiguously stored block of memory, and so naturally
lends itself to conformance to `ContiguousCollection`. However, this has one
specific carved-out exception on Darwin platforms for the purposes of
Objective-C interop. When an `NSArray` of classes is bridged from Objective-C
code, it remains as an `NSArray`, and the `Array` forwards element accesses to
that bridged `NSArray`.

This is very different from `NSArray` itself, which abstracts away the storage
to a much greater degree, giving it flexibility to present an "array-like"
interface to multiple different backings.

Here's a run-down of when Array will be contiguously stored:

- Arrays created in Swift will **always** be contiguously stored;

- Arrays of structs and enums will **always** be contiguously stored;

- Arrays on platforms without an Objective-C runtime (i.e. non-Darwin
  platforms) are **always** contiguously stored;

- The only time an array **won't** be contiguously stored is if it is of
  classes and has been bridged from an `NSArray`. Even then, in many cases, the
  NSArray will be contiguously stored and could present a pointer at no or
  amortized cost.

These caveats should be documented clearly on both the protocol and on `Array`.
Note that in use cases such as the `vDSP` family of functions, this is not a
concern as the element types involved are structs. The documented caveat
approach has precedent in several places already in Swift: the conformance of
floating-point types to `Comparable`, the complexity of operations like `first`
and `last` on some lazy random-access types, and of the existing implementation
of `withUnsafeBufferPointer` on `Array` itself.

Note that these caveats only apply to the un-mutable variant. The
first thing an array does when you call a mutating method is ensure that it's
uniquely referenced and contiguous, so even lazily bridged arrays will become
contiguous at that point. This copying occurs naturally in other cases, such 
as multiply-referenced CoW buffers.

