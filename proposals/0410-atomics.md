# Low-Level Atomic Operations ⚛︎

* Proposal: [SE-0410](0410-atomics.md)
* Author: [Karoy Lorentey](https://github.com/lorentey), [Alejandro Alonso](https://github.com/Azoy)
* Review Manager: [Joe Groff](https://github.com/jckarter)
* Bug: [SR-9144](https://github.com/apple/swift/issues/51640)
* Implementation: [apple/swift#68857](https://github.com/apple/swift/pull/68857)
* Version: 2023-12-04
* Status: **Implemented (Swift 6.0)**
* Previous Revision: [1](https://github.com/swiftlang/swift-evolution/blob/d35d6566fe2297f4782bdfac4d5253e0ca96b353/proposals/0410-atomics.md)
* Decision Notes: [pitch](https://forums.swift.org/t/atomics/67350), [first review](https://forums.swift.org/t/se-0410-atomics/68007), [first return for revision](https://forums.swift.org/t/returned-for-revision-se-0410-atomics/68522), [second review](https://forums.swift.org/t/second-review-se-0410-atomics/68810), [acceptance](https://forums.swift.org/t/accepted-with-modifications-se-0410-atomics/69244)

## Introduction

This proposal adds a limited set of low-level atomic operations to the Standard Library, including native spellings for C++-style memory orderings. Our goal is to enable intrepid library authors and developers writing system level code to start building synchronization constructs directly in Swift.

Previous Swift-evolution thread: [Low-Level Atomic Operations](https://forums.swift.org/t/low-level-atomic-operations/34683)

New Swift-evolution thread: [Atomics](https://forums.swift.org/t/atomics/67350)

## Revision History

- 2020-04-13: Initial proposal version.
- 2020-06-05: Second revision.
  - Removed all new APIs; the proposal is now focused solely on C interoperability.
- 2023-09-18: Third revision.
  - Introduced new APIs to the standard library.
- 2023-12-04: Fourth revision.
  - Response to language steering group [review decision notes](https://forums.swift.org/t/returned-for-revision-se-0410-atomics/68522).
  - New APIs are now in a `Synchronization` module instead of the default `Swift` module.
  - Declaring a `var` of `Atomic` type is now an error.

## Table of Contents

  * [Motivation](#motivation)
  * [Proposed Solution](#proposed-solution)
    * [The Synchronization Module](#the-synchronization-module)
    * [Atomic Memory Orderings](#atomic-memory-orderings)
    * [The Atomic Protocol Hierarchy](#the-atomic-protocol-hierarchy)
      * [Optional Atomics](#optional-atomics)
      * [Custom Atomic Types](#custom-atomic-types)
    * [Atomic Storage Types](#atomic-storage-types)
    * [WordPair](#wordpair)
    * [The Atomic type](#the-atomic-type)
    * [Basic Atomic Operations](#basic-atomic-operations)
    * [Specialized Integer Operations](#specialized-integer-operations)
    * [Specialized Boolean Operations](#specialized-boolean-operations)
    * [Atomic Lazy References](#atomic-lazy-references)
    * [Restricting Ordering Arguments to Compile\-Time Constants](#restricting-ordering-arguments-to-compile-time-constants)
  * [Interaction with Existing Language Features](#interaction-with-existing-language-features)
    * [Interaction with Swift Concurrency](#interaction-with-swift-concurrency)
  * [Detailed Design](#detailed-design)
    * [Atomic Memory Orderings](#atomic-memory-orderings-1)
    * [Atomic Protocols](#atomic-protocols)
      * [AtomicRepresentable](#atomicrepresentable)
      * [AtomicOptionalRepresentable](#atomicoptionalrepresentable)
    * [WordPair](#wordpair-1)
    * [Atomic Types](#atomic-types)
      * [Atomic&lt;Value&gt;](#atomicvalue)
      * [AtomicLazyReference&lt;Instance&gt;](#atomiclazyreferenceinstance)
  * [Source Compatibility](#source-compatibility)
  * [Effect on ABI Stability](#effect-on-abi-stability)
  * [Effect on API Resilience](#effect-on-api-resilience)
  * [Potential Future Directions](#potential-future-directions)
    * [Atomic Strong References and The Problem of Memory Reclamation](#atomic-strong-references-and-the-problem-of-memory-reclamation)
    * [Additional Low\-Level Atomic Features](#additional-low-level-atomic-features)
  * [Alternatives Considered](#alternatives-considered)
    * [Default Orderings](#default-orderings)
    * [A Truly Universal Generic Atomic Type](#a-truly-universal-generic-atomic-type)
    * [Providing a value Property](#providing-a-value-property)
    * [Alternative Designs for Memory Orderings](#alternative-designs-for-memory-orderings)
      * [Encode Orderings in Method Names](#encode-orderings-in-method-names)
      * [Orderings As Generic Type Parameters](#orderings-as-generic-type-parameters)
      * [Ordering Views](#ordering-views)
    * [Directly bring over `swift-atomics`'s API](#directly-bring-over-swift-atomicss-api)
  * [References](#references)

## Motivation

In Swift today, application developers use Swift's recently accepted concurrency features including async/await, structured concurrency with Task and TaskGroup, AsyncSequence/AsyncStream, etc. as well as dispatch queues and Foundation's NSLocking protocol to synchronize access to mutable state across concurrent threads of execution.

However, for Swift to be successful as a systems programming language, it needs to also provide low-level primitives that can be used to implement such synchronization constructs (and many more!) directly within Swift. Such low-level synchronization primitives allow developers more flexible ways to synchronize access to specific properties or storage allowing them to opt their types into Swift conconcurrency by declaring their types `@unchecked Sendable`. Of course these low-level primitives also allow library authors to build more high level synchronization structures that are both easier and safer to use that developers can also utilize to synchronize memory access.

One such low-level primitive is the concept of an atomic value, which (in the form we propose here) has two equally important roles:

- First, atomics introduce a limited set of types whose values provide well-defined semantics for certain kinds of concurrent access. This includes explicit support for concurrent mutations -- a concept that Swift never supported before.

- Second, atomic operations come with explicit memory ordering arguments, which provide guarantees on how/when the effects of earlier or later memory accesses become visible to other threads. Such guarantees are crucial for building higher-level synchronization abstractions.

These new primitives are intended for people who wish to implement synchronization constructs or concurrent data structures in pure Swift code. Note that this is a hazardous area that is full of pitfalls. While a well-designed atomics facility can help simplify building such tools, the goal here is merely to make it *possible* to build them, not necessarily to make it *easy* to do so. We expect that the higher-level synchronization tools that can be built on top of these atomic primitives will provide a nicer abstraction layer.

We want to limit this proposal to constructs that satisfy the following requirements:

1. All atomic operations need to be explicit in Swift source, and it must be possible to easily distinguish them from regular non-atomic operations on the underlying values.

2. The atomic type we provide must come with a lock-free implementation on every platform that implements them. (Platforms that are unable to provide lock-free implementations must not provide the affected constructs at all.)

3. Every atomic operation must compile down to the corresponding CPU instruction (when one is available), with minimal overhead. (Ideally even if the code is compiled without optimizations.) Wait-freedom isn't a requirement -- if no direct instruction is available for an operation, then it must still be implemented, e.g. by mapping it to a compare-exchange loop.

Following the acceptance of [Clarify the Swift memory consistency model (SE-0282)](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0282-atomics.md), the [swift-atomics package](https://github.com/apple/swift-atomics) was shortly created to experiment and design what a standard atomic API would look like. This proposal is relying heavily on some of the ideas that package has spent years developing and designing.

## Proposed Solution

We propose to introduce new low-level atomic APIs to the standard library via a new module. These atomic APIs will serve as the foundation for building higher-level concurrent code directly in Swift.

As a quick taste, this is how atomics will work:

```swift
import Synchronization
import Dispatch

let counter = Atomic<Int>(0)

DispatchQueue.concurrentPerform(iterations: 10) { _ in
  for _ in 0 ..< 1_000_000 {
    counter.wrappingAdd(1, ordering: .relaxed)
  }
}

print(counter.load(ordering: .relaxed))
```

### The Synchronization Module

While most Swift programs won't directly use the new atomic primitives, we still consider the new constructs to be an integral part of the core Standard Library.

That said, it seems highly undesirable to add low-level atomics to the default namespace of every Swift program, so we propose to place the atomic constructs in a new Standard Library module called `Synchronization`. Code that needs to use low-level atomics will need to explicitly import the new module:

```swift
import Synchronization
```

We expect that most Swift projects will use atomic operations only indirectly, through higher-level synchronization constructs. Therefore, importing the `Synchronization` module will be a relatively rare occurrance, mostly limited to projects that implement such tools.

### Atomic Memory Orderings

The atomic constructs later in this proposal implement concurrent read/write access by mapping to atomic instructions in the underlying architecture. All accesses of a particular atomic value get serialized into some global sequential timeline, no matter what thread executed them.

However, this alone does not give us a way to synchronize accesses to regular variables, or between atomic accesses to different memory locations. To support such synchronization, each atomic operation can be configured to also act as a synchronization point for other variable accesses within the same thread, preventing previous accesses from getting executed after the atomic operation, and/or vice versa. Atomic operations on another thread can then synchronize with the same point, establishing a strict (although partial) timeline between accesses performed by both threads. This way, we can reason about the possible ordering of operations across threads, even if we know nothing about how those operations are implemented. (This is how locks or dispatch queues can be used to serialize the execution of arbitrary blocks containing regular accesses to shared variables.) For more details, see \[[C++17], [N2153], [Boehm 2008]].

In order to enable atomic synchronization within Swift, we must first introduce memory orderings that will give us control of the timeline of these operations across threads. Luckily, with the acceptance of [Clarify the Swift memory consistency model (SE-0282)](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0282-atomics.md), Swift already adopts the C/C++ concurrency memory model. In this model, concurrent access to shared state remains undefined behavior unless all such access is forced into a conflict-free timeline through explicit synchronization operations.

This proposal introduces five distinct memory orderings, organized into three logical groups, from loosest to strictest:

* `.relaxed`
* `.acquiring`, `.releasing`, `.acquiringAndReleasing`
* `.sequentiallyConsistent`

These align with select members of the standard `std::memory_order` enumeration in C++, and are intended to carry the same semantic meaning:

|             C++             |            Swift            |
|             :---:           |            :---:            |
| `std::memory_order_relaxed` |         `.relaxed`          |
| `std::memory_order_consume` | *not yet adopted [[P0735]]* |
| `std::memory_order_acquire` |        `.acquiring`         |
| `std::memory_order_release` |        `.releasing`         |
| `std::memory_order_acq_rel` |  `.acquiringAndReleasing`   |
| `std::memory_order_seq_cst` |  `.sequentiallyConsistent`  |

Atomic orderings are grouped into three frozen structs based on the kind of operation to which they are attached, as listed below. By modeling these as separate types, we can ensure that unsupported operation/ordering combinations (such as an atomic "releasing load") will lead to clear compile-time errors:

```swift
/// Specifies the memory ordering semantics of an atomic load operation.
public struct AtomicLoadOrdering {
  public static var relaxed: Self { get }
  public static var acquiring: Self { get }
  public static var sequentiallyConsistent: Self { get }
}

/// Specifies the memory ordering semantics of an atomic store operation.
public struct AtomicStoreOrdering {
  public static var relaxed: Self { get }
  public static var releasing: Self { get }
  public static var sequentiallyConsistent: Self { get }
}

/// Specifies the memory ordering semantics of an atomic read-modify-write
/// operation.
public struct AtomicUpdateOrdering {
  public static var relaxed: Self { get }
  public static var acquiring: Self { get }
  public static var releasing: Self { get }
  public static var acquiringAndReleasing: Self { get }
  public static var sequentiallyConsistent: Self { get }
}
```

These structs behave like non-frozen enums with a known (non-public) raw representation. This allows us to define additional memory orderings in the future (if and when they become necessary, specifically `std::memory_order_consume`) while making use of the known representation to optimize existing cases. (These cannot be frozen enums because that would prevent us from adding more orderings, but regular resilient enums can't freeze their representation, and the layout indirection interferes with guaranteed optimizations, especially in -Onone.)

Every atomic operation introduced later in this proposal requires an ordering argument. We consider these ordering arguments to be an essential part of these low-level atomic APIs, and we require an explicit `ordering` argument on all atomic operations. The intention here is to force developers to carefully think about what ordering they need to use, each time they use one of these primitives. (Perhaps more importantly, this also makes it obvious to readers of the code what ordering is used -- making it far less likely that an unintended default `.sequentiallyConsistent` ordering slips through code review.) 

Projects that prefer to default to sequentially consistent ordering are welcome to add non-public `Atomic` extensions that implement that. However, we expect that providing an implicit default ordering would be highly undesirable in most production uses of atomics.

We also provide a top-level function called `atomicMemoryFence` that allows issuing a memory ordering constraint without directly associating it with a particular atomic operation. This corresponds to `std::atomic_thread_fence` in C++ [[C++17]].

```swift
/// Establishes a memory ordering without associating it with a
/// particular atomic operation.
///
/// - A relaxed fence has no effect.
/// - An acquiring fence ties to any preceding atomic operation that
///   reads a value, and synchronizes with any releasing operation whose
///   value was read.
/// - A releasing fence ties to any subsequent atomic operation that
///   modifies a value, and synchronizes with any acquiring operation
///   that reads the result.
/// - An acquiring and releasing fence is a combination of an
///   acquiring and a releasing fence.
/// - A sequentially consistent fence behaves like an acquiring and
///   releasing fence, and ensures that the fence itself is part of
///   the single, total ordering for all sequentially consistent
///   operations.
///
/// This operation corresponds to `std::atomic_thread_fence` in C++.
///
/// Be aware that Thread Sanitizer does not support fences and may report
/// false-positive races for data protected by a fence.
public func atomicMemoryFence(ordering: AtomicUpdateOrdering)
```

Fences are slightly more powerful (but even more difficult to use) than orderings tied to specific atomic operations [[N2153]]; we expect their use will be limited to the most performance-sensitive synchronization constructs.

### The Atomic Protocol Hierarchy

The notion of an atomic type is captured by the `AtomicRepresentable` protocol.

```swift
/// A type that supports atomic operations through a separate atomic storage
/// representation.
public protocol AtomicRepresentable {
  associatedtype AtomicRepresentation

  static func encodeAtomicRepresentation(
    _ value: consuming Self
  ) -> AtomicRepresentation

  static func decodeAtomicRepresentation(
    _ representation: consuming AtomicRepresentation
  ) -> Self
}
```

The requirements in `AtomicRepresentable` set up a bidirectional mapping between values of the atomic type and an associated storage representation that implements the actual primitive atomic operations.

`AtomicRepresentation` is intentionally left unconstrained because as you'll see later in the proposal, atomic operations are only available when `AtomicRepresentation` is one of the core atomic storage types found here: [Atomic Storage Types](#atomic-storage-types).

The full set of standard types implementing `AtomicRepresentable` is listed below:

```swift
extension Int: AtomicRepresentable {...}
extension Int64: AtomicRepresentable {...}
extension Int32: AtomicRepresentable {...}
extension Int16: AtomicRepresentable {...}
extension Int8: AtomicRepresentable {...}
extension UInt: AtomicRepresentable {...}
extension UInt64: AtomicRepresentable {...}
extension UInt32: AtomicRepresentable {...}
extension UInt16: AtomicRepresentable {...}
extension UInt8: AtomicRepresentable {...}

extension Bool: AtomicRepresentable {...}

extension Float16: AtomicRepresentable {...}
extension Float: AtomicRepresentable {...}
extension Double: AtomicRepresentable {...}

/// New type in the standard library discussed
/// shortly after this.
extension WordPair: AtomicRepresentable {...}

extension Duration: AtomicRepresentable {...}

extension Never: AtomicRepresentable {...}

extension UnsafeRawPointer: AtomicRepresentable {...}
extension UnsafeMutableRawPointer: AtomicRepresentable {...}
extension UnsafePointer: AtomicRepresentable {...}
extension UnsafeMutablePointer: AtomicRepresentable {...}
extension Unmanaged: AtomicRepresentable {...}
extension OpaquePointer: AtomicRepresentable {...}
extension ObjectIdentifier: AtomicRepresentable {...}

extension UnsafeBufferPointer: AtomicRepresentable {...}
extension UnsafeMutableBufferPointer: AtomicRepresentable {...}
extension UnsafeRawBufferPointer: AtomicRepresentable {...}
extension UnsafeMutableRawBufferPointer: AtomicRepresentable {...}

extension Optional: AtomicRepresentable where Wrapped: AtomicOptionalRepresentable {...}
```

* On 32 bit platforms that do not support double-word atomics, the following conformances are not available:
  * `UInt64`
  * `Int64`
  * `Double`
  * `UnsafeBufferPointer`
  * `UnsafeMutableBufferPointer`
  * `UnsafeRawBufferPointer`
  * `UnsafeMutableRawBufferPointer`
* On 64 bit plaforms that do not support double-word atomics, the following conformances are not available:
  * `Duration`
  * `UnsafeBufferPointer`
  * `UnsafeMutableBufferPointer`
  * `UnsafeRawBufferPointer`
  * `UnsafeMutableRawBufferPointer`

This proposal does not conform `Duration` to `AtomicRepresentable` on any currently supported 32 bit platform. (Not even those where quad-word atomics are technically available, like arm64_32.) 

#### Optional Atomics

The standard atomic pointer types and unmanaged references also support atomic operations on their optional-wrapped form. To spell out this optional wrapped, we introduce a new protocol:

```swift
public protocol AtomicOptionalRepresentable: AtomicRepresentable {
  associatedtype AtomicOptionalRepresentation

  static func encodeAtomicOptionalRepresentation(
    _ value: consuming Self?
  ) -> AtomicOptionalRepresentation

  static func decodeAtomicOptionalRepresentation(
    _ representation: consuming AtomicOptionalRepresentation
  ) -> Self?
}
```

Similar to `AtomicRepresentable`, `AtomicOptionalRepresentable`'s requirements create a bidirectional mapping between an optional value of `Self` to some atomic optional storage representation and vice versa.

`Optional` implements `AtomicRepresentable` through a conditional conformance to this new `AtomicOptionalRepresentable` protocol.

```swift
extension Optional: AtomicRepresentable where Wrapped: AtomicOptionalRepresentable {
  ...
}
```

This proposal enables optional-atomics support for the following types:

```swift
extension UnsafeRawPointer: AtomicOptionalRepresentable {}
extension UnsafeMutableRawPointer: AtomicOptionalRepresentable {}
extension UnsafePointer: AtomicOptionalRepresentable {}
extension UnsafeMutablePointer: AtomicOptionalRepresentable {}
extension Unmanaged: AtomicOptionalRepresentable {}
extension OpaquePointer: AtomicOptionalRepresentable {}
extension ObjectIdentifier: AtomicOptionalRepresentable {}
```

Atomic optional pointers and references are helpful when building lock-free data structures. (Although this initial set of reference types considerably limits the scope of what can be built; for more details, see the discussion on the [ABA problem](#wordpair) and [memory reclamation](#atomic-strong-references-and-the-problem-of-memory-reclamation).)

For example, consider the lock-free, single-consumer stack implementation below. (It supports an arbitrary number of concurrently pushing threads, but it only allows a single pop at a time.)

```swift
class LockFreeSingleConsumerStack<Element> {
  struct Node {
    let value: Element
    var next: UnsafeMutablePointer<Node>?
  }
  typealias NodePtr = UnsafeMutablePointer<Node>

  private let _last = Atomic<NodePtr?>(nil)
  private let _consumerCount = Atomic<Int>(0)

  deinit {
    // Discard remaining nodes
    while let _ = pop() {}
  }

  // Push the given element to the top of the stack.
  // It is okay to concurrently call this in an arbitrary number of threads.
  func push(_ value: Element) {
    let new = NodePtr.allocate(capacity: 1)
    new.initialize(to: Node(value: value, next: nil))

    var done = false
    var current = _last.load(ordering: .relaxed)
    while !done {
      new.pointee.next = current
      (done, current) = _last.compareExchange(
        expected: current,
        desired: new,
        ordering: .releasing
      )
    }
  }

  // Pop and return the topmost element from the stack.
  // This method does not support multiple overlapping concurrent calls.
  func pop() -> Element? {
    precondition(
      _consumerCount.wrappingAdd(1, ordering: .acquiring).oldValue == 0,
      "Multiple consumers detected")
    defer { _consumerCount.wrappingSubtract(1, ordering: .releasing) }
    var done = false
    var current = _last.load(ordering: .acquiring)
    while let c = current {
      (done, current) = _last.compareExchange(
        expected: c,
        desired: c.pointee.next,
        ordering: .acquiring
      )

      if done {
        let result = c.move()
        c.deallocate()
        return result.value
      }
    }
    return nil
  }
}
```

#### Custom Atomic Types

To enable a limited set of user-defined atomic types, `AtomicRepresentable` also provides a full set of default implementations for `RawRepresentable` types whose raw value is itself atomic:

```swift
extension RawRepresentable where Self: AtomicRepresentable, RawValue: AtomicRepresentable {
  ...
}
```

The default implementations work by forwarding all atomic operations to the raw value's implementation, converting to/from as needed.

This enables code outside of the Standard Library to add new `AtomicRepresentable` conformances without manually implementing any of the requirements. This is especially handy for trivial raw-representable enumerations, such as in simple atomic state machines:

```swift
enum MyState: Int, AtomicRepresentable {
  case starting
  case running
  case stopped
}

let currentState = Atomic<MyState>(.starting)
...
if currentState.compareExchange(
  expected: .starting, 
  desired: .running, 
  ordering: .sequentiallyConsistent
).exchanged {
  ...
}
...
currentState.store(.stopped, ordering: .sequentiallyConsistent)
```

We also support the `AtomicOptionalRepresentable` defaults for `RawRepresentable` as well:

```swift
extension RawRepresentable where Self: AtomicOptionalRepresentable, RawValue: AtomicOptionalRepresentable {
  ...
}
```

For example, we can use this to add atomic operations over optionals of types whose raw value is a pointer:

```swift
struct MyPointer: RawRepresentable, AtomicOptionalRepresentable {
	var rawValue: UnsafeRawPointer
  
  init(rawValue: UnsafeRawPointer) {
    self.rawValue = rawValue
  }
}

let myAtomicPointer = Atomic<MyPointer?>(nil)
...
if myAtomicPointer.compareExchange(
	expected: nil,
  desired: MyPointer(rawValue: somePointer),
  ordering: .relaxed
).exchanged {
  ...
}
...
myAtomicPointer.store(nil, ordering: .releasing)
```

(This gets you an `AtomicRepresentable` conformance for free as well because `AtomicOptionalRepresentable` refines `AtomicRepresentable`. So  this also allows non-optional use with `Atomic<MyPointer>`.)

### Atomic Storage Types

Fundamental to working with atomics is knowing that CPUs can only do atomic operations on integers. While we could theoretically do atomic operations with our current list of standard library integer types (`Int8`, `Int16`, ...), some platforms don't ensure that these types have the same alignment as their size. For example, `Int64` and `UInt64` have 4 byte alignment on i386. Atomic operations must occur on correctly aligned types. To ensure this, we need to introduce helper types that all atomic operations will be trafficked through. These types will serve as the `AtomicRepresentation` for all of the standard integer types:

```swift
extension Int8: AtomicRepresentable {
  public typealias AtomicRepresentation = ...
}

...

extension UInt64: AtomicRepresentable {
  public typealias AtomicRepresentation = ...
}

...
```

The actual underlying type is an implementation detail of the standard library. While we generally don't prefer to propose such API, the underlying types themselves are quite useless and only useful for the primitive integers. One can still access the underlying type by using the public name, `Int8.AtomicRepresentation`, for example. An example conformance to `AtomicRepresentable` may look something like the following:

```swift
struct MyCoolInt {
  var x: Int
}

extension MyCoolInt: AtomicRepresentable {
  typealias AtomicRepresentation = Int.AtomicRepresentation
  
  static func encodeAtomicRepresentation(
    _ value: consuming MyCoolInt
  ) -> AtomicRepresentation {
    Int.encodeAtomicRepresentation(value.x)
  }
  
  static func decodeAtomicRepresentation(
    _ representation: consuming AtomicRepresentation
  ) -> MyCoolInt {
    MyCoolInt(
      x:Int.decodeAtomicRepresentation(representation)
    )
  }
}
```

This works by going through `Int`'s `AtomicRepresentable` conformance and converting our `MyCoolInt` -> `Int` -> `Int.AtomicRepresentation` .

### `WordPair`

In their current single-word form, atomic pointer and reference types are susceptible to a class of race condition called the *ABA problem*. A freshly allocated object often happens to be placed at the same memory location as a recently deallocated one. Therefore, two successive `load`s of a simple atomic pointer may return the exact same value, even though the pointer may have received an arbitrary number of updates between the two loads, and the pointee may have been completely replaced. This can be a subtle, but deadly source of race conditions in naive implementations of many concurrent data structures.

While the single-word atomic primitives introduced in this document are already useful for some applications, it would be helpful to also provide a set of additional atomic operations that operate on two consecutive `Int`-sized values in the same transaction. All currently supported architectures provide direct hardware support for such double-word atomic operations.

We propose a new separate type that provides an abstraction over the layout of what a double-word is for a platform.

```swift
public struct WordPair {
  public var first: UInt { get }
  public var second: UInt { get }

  public init(first: UInt, second: UInt)
}

// Not a real compilation conditional
#if hasDoubleWideAtomics
extension WordPair: AtomicRepresentable {
// Not a real compilation conditional
#if 64 bit
  public typealias AtomicRepresentaton = ... 128 bit 16 aligned storage
#elseif 32 bit
  public typealias AtomicRepresentation = ... 64 bit 8 aligned storage
#else
#error("Not a supported platform")
#endif

  ...
}
#endif
```

For example, the second word can be used to augment atomic values with a version counter (sometimes called a "stamp" or a "tag"), which can help resolve the ABA problem by allowing code to reliably verify if a value remained unchanged between two successive loads.

Note that not all CPUs support double-word atomic operations and so if Swift starts supporting such processors, this type's conformance to `AtomicRepresentable` may not always be available. Platforms that cannot support double-word atomics must not make `WordPair`'s `AtomicRepresentable` conformance available for use.

(If this becomes a real concern, a future proposal could introduce something like a `#if hasDoubleWordAtomics` compile-time condition to let code adapt to more limited environments. However, this is deferred until Swift actually starts supporting such platforms.)

### The Atomic type

So far, we've introduced memory orderings, giving us control of memory access around atomic operations; the atomic protocol hierarchy, which give us the initial list of standard types that can be as atomic values; and the `WordPair` type, providing an abstraction over a platform's double-word type. However, we haven't yet introduced a way to actually _use_ atomics. Here we introduce the single Atomic type that exposes atomic operations for us:

```swift
/// An atomic value.
public struct Atomic<Value: AtomicRepresentable>: ~Copyable {
  public init(_ initialValue: consuming Value)
}
```

A value of `Atomic<Value>` shares the same layout as `Value.AtomicRepresentation`.

Now that we know how to create an atomic value, it's time to introduce some actual atomic operations.

### Basic Atomic Operations

`Atomic` provides seven basic atomic operations when `Value.AtomicRepresentation` is one of the fundamental atomic storage types on the standard integer types:

```swift
extension Atomic where Value.AtomicRepresentation == {U}IntNN.AtomicRepresentation {
  /// Atomically loads and returns the current value, applying the specified
  /// memory ordering.
  ///
  /// - Parameter ordering: The memory ordering to apply on this operation.
  /// - Returns: The current value.
  public borrowing func load(ordering: AtomicLoadOrdering) -> Value
  
  /// Atomically sets the current value to `desired`, applying the specified
  /// memory ordering.
  ///
  /// - Parameter desired: The desired new value.
  /// - Parameter ordering: The memory ordering to apply on this operation.
  public borrowing func store(
    _ desired: consuming Value,
    ordering: AtomicStoreOrdering
  )

  /// Atomically sets the current value to `desired` and returns the original
  /// value, applying the specified memory ordering.
  ///
  /// - Parameter desired: The desired new value.
  /// - Parameter ordering: The memory ordering to apply on this operation.
  /// - Returns: The original value.
  public borrowing func exchange(
    _ desired: consuming Value, 
    ordering: AtomicUpdateOrdering
  ) -> Value

  /// Perform an atomic compare and exchange operation on the current value,
  /// applying the specified memory ordering.
  ///
  /// This operation performs the following algorithm as a single atomic
  /// transaction:
  ///
  /// ```
  /// atomic(self) { currentValue in
  ///   let original = currentValue
  ///   guard original == expected else { return (false, original) }
  ///   currentValue = desired
  ///   return (true, original)
  /// }
  /// ```
  ///
  /// This method implements a "strong" compare and exchange operation
  /// that does not permit spurious failures.
  ///
  /// - Parameter expected: The expected current value.
  /// - Parameter desired: The desired new value.
  /// - Parameter ordering: The memory ordering to apply on this operation.
  /// - Returns: A tuple `(exchanged, original)`, where `exchanged` is true if
  ///   the exchange was successful, and `original` is the original value.
  public borrowing func compareExchange(
    expected: consuming Value,
    desired: consuming Value,
    ordering: AtomicUpdateOrdering
  ) -> (exchanged: Bool, original: Value)

  /// Perform an atomic compare and exchange operation on the current value,
  /// applying the specified success/failure memory orderings.
  ///
  /// This operation performs the following algorithm as a single atomic
  /// transaction:
  ///
  /// ```
  /// atomic(self) { currentValue in
  ///   let original = currentValue
  ///   guard original == expected else { return (false, original) }
  ///   currentValue = desired
  ///   return (true, original)
  /// }
  /// ```
  ///
  /// The `successOrdering` argument specifies the memory ordering to use when
  /// the operation manages to update the current value, while `failureOrdering`
  /// will be used when the operation leaves the value intact.
  ///
  /// This method implements a "strong" compare and exchange operation
  /// that does not permit spurious failures.
  ///
  /// - Parameter expected: The expected current value.
  /// - Parameter desired: The desired new value.
  /// - Parameter successOrdering: The memory ordering to apply if this
  ///    operation performs the exchange.
  /// - Parameter failureOrdering: The memory ordering to apply on this
  ///    operation if it does not perform the exchange.
  /// - Returns: A tuple `(exchanged, original)`, where `exchanged` is true if
  ///   the exchange was successful, and `original` is the original value.
  public borrowing func compareExchange(
    expected: consuming Value,
    desired: consuming Value,
    successOrdering: AtomicUpdateOrdering,
    failureOrdering: AtomicLoadOrdering
  ) -> (exchanged: Bool, original: Value)

  /// Perform an atomic weak compare and exchange operation on the current
  /// value, applying the memory ordering. This compare-exchange variant is
  /// allowed to spuriously fail; it is designed to be called in a loop until
  /// it indicates a successful exchange has happened.
  ///
  /// This operation performs the following algorithm as a single atomic
  /// transaction:
  ///
  /// ```
  /// atomic(self) { currentValue in
  ///   let original = currentValue
  ///   guard original == expected else { return (false, original) }
  ///   currentValue = desired
  ///   return (true, original)
  /// }
  /// ```
  ///
  /// (In this weak form, transient conditions may cause the `original ==
  /// expected` check to sometimes return false when the two values are in fact
  /// the same.)
  ///
  /// - Parameter expected: The expected current value.
  /// - Parameter desired: The desired new value.
  /// - Parameter ordering: The memory ordering to apply on this operation.
  /// - Returns: A tuple `(exchanged, original)`, where `exchanged` is true if
  ///   the exchange was successful, and `original` is the original value.
  public borrowing func weakCompareExchange(
    expected: consuming Value,
    desired: consuming Value,
    ordering: AtomicUpdateOrdering
  ) -> (exchanged: Bool, original: Value)

  /// Perform an atomic weak compare and exchange operation on the current
  /// value, applying the specified success/failure memory orderings. This
  /// compare-exchange variant is allowed to spuriously fail; it is designed to
  /// be called in a loop until it indicates a successful exchange has happened.
  ///
  /// This operation performs the following algorithm as a single atomic
  /// transaction:
  ///
  /// ```
  /// atomic(self) { currentValue in
  ///   let original = currentValue
  ///   guard original == expected else { return (false, original) }
  ///   currentValue = desired
  ///   return (true, original)
  /// }
  /// ```
  ///
  /// (In this weak form, transient conditions may cause the `original ==
  /// expected` check to sometimes return false when the two values are in fact
  /// the same.)
  ///
  /// The `ordering` argument specifies the memory ordering to use when the
  /// operation manages to update the current value, while `failureOrdering`
  /// will be used when the operation leaves the value intact.
  ///
  /// - Parameter expected: The expected current value.
  /// - Parameter desired: The desired new value.
  /// - Parameter successOrdering: The memory ordering to apply if this
  ///    operation performs the exchange.
  /// - Parameter failureOrdering: The memory ordering to apply on this
  ///    operation does not perform the exchange.
  /// - Returns: A tuple `(exchanged, original)`, where `exchanged` is true if
  ///   the exchange was successful, and `original` is the original value.
  public borrowing func weakCompareExchange(
    expected: consuming Value,
    desired: consuming Value,
    successOrdering: AtomicUpdateOrdering,
    failureOrdering: AtomicLoadOrdering
  ) -> (exchanged: Bool, original: Value)
}
```

Because these are only available when `Value.AtomicRepresentation == {U}IntNN.AtomicRepresentation`, some atomic specializations may not support atomic operations at all.

The first three operations are relatively simple:

- `load` returns the current value.
- `store` updates it.
- `exchange` is a combination of `load` and `store`; it updates the
  current value and returns the previous one as a single atomic
  operation.

The three `compareExchange` variants are somewhat more complicated: they implement a version of `exchange` that only performs the update if the original value is the same as a supplied expected value. To be specific, they execute the following algorithm as a single atomic transaction:

```swift
  guard currentValue == expected else { 
    return (exchanged: false, original: currentValue) 
  }
  currentValue = desired
  return (exchanged: true, original: expected)
```

All four variants implement the same algorithm. The single ordering variants use the same memory ordering whether or not the exchange succeeds, while the others allow callers to specify two distinct memory orderings for the success and failure cases. The two orderings are independent from each other -- all combinations of update/load orderings are supported [[P0418]]. (Of course, the implementation may need to "round up" to the nearest ordering combination that is supported by the underlying code generation layer and the targeted CPU architecture.)

The `weakCompareExchange` form may sometimes return false even when the original and expected values are equal. (Such failures may happen when some transient condition prevents the underlying operation from succeeding -- such as an incoming interrupt during a load-link/store-conditional instruction sequence.) This variant is designed to be called in a loop that only exits when the exchange is successful; in such loops, using `weakCompareExchange` may lead to a performance improvement by eliminating a nested loop in the regular, "strong", `compareExchange` variants.

The compare-exchange primitive is special: it is a universal operation that can be used to implement all other atomic operations, and more. For example, here is how we could use `compareExchange` to implement a wrapping add operation over `Atomic<Int>` values:

```swift
extension Atomic where Value == Int {
  func wrappingAdd(
    _ operand: Int,
    ordering: AtomicUpdateOrdering
  ) {
    var done = false
    var current = load(ordering: .relaxed)
    while !done {
      (done, current) = compareExchange(
        expected: current,
        desired: current &+ operand,
        ordering: ordering
      )
    }
  }
}
```

### Specialized Integer Operations

Most CPU architectures provide dedicated atomic instructions for certain integer operations, and these are generally more efficient than implementations using `compareExchange`. Therefore, it makes sense to expose a set of dedicated methods for common integer operations so that these will always get compiled into the most efficient implementation available.

| Method Name | Returns | Implements |
| --- | --- | --- |
| `wrappingAdd(_: Value, ordering: AtomicUpdateOrdering)` | `(oldValue: Value, newValue: Value)` | `a &+= b`  |
| `wrappingSubtract(_: Value, ordering: AtomicUpdateOrdering)` | `(oldValue: Value, newValue: Value)` | `a &-= b`  |
| `add(_: Value, ordering: AtomicUpdateOrdering)` | `(oldValue: Value, newValue: Value)` | `a += b` (checks for overflow) |
| `subtract(_: Value, ordering: AtomicUpdateOrdering)` | `(oldValue: Value, newValue: Value)` | `a -= b` (checks for overflow) |
| `bitwiseAnd(_: Value, ordering: AtomicUpdateOrdering)` | `(oldValue: Value, newValue: Value)` | `a &= b` |
| `bitwiseOr(_: Value, ordering: AtomicUpdateOrdering)` | `(oldValue: Value, newValue: Value)` | `a \|= b` |
| `bitwiseXor(_: Value, ordering: AtomicUpdateOrdering)` | `(oldValue: Value, newValue: Value)` | `a ^= b` |
| `min(_: Value, ordering: AtomicUpdateOrdering)` | `(oldValue: Value, newValue: Value)` | `a = Swift.min(a, b)` |
| `max(_: Value, ordering: AtomicUpdateOrdering)` | `(oldValue: Value, newValue: Value)` | `a = Swift.max(a, b)` |

All operations are also marked as `@discardableResult` in the case where one doesn't care about the old value or new value. The `add` and `subtract` operations explicitly check for overflow and will trap at runtime if one occurs, except in `-Ounchecked` builds. 

While we require all atomic operations to be free of locks, we don't require wait-freedom. Therefore, on architectures that don't provide direct hardware support for some or all of these operations, we still require them to be implemented using `compareExchange` loops like the one for `wrappingAdd` above.

`Atomic<Value>` exposes these operations when `Value` is one of the standard fixed-width integer types.

```swift
extension Atomic where Value == Int {...}
extension Atomic where Value == UInt8 {...}
...

let counter = Atomic<Int>(0)
counter.wrappingAdd(42, ordering: .relaxed)

let oldMax = counter.max(82, ordering: .relaxed).oldValue
```

### Specialized Boolean Operations

Similar to the specialized integer operations, we can provide similar ones for booleans:

| Method Name                  | Returns               | Implements   |
| ---------------------------- | --------------------- | ------------ |
| `logicalAnd(_: Bool, ordering: AtomicUpdateOrdering)` | `(oldValue: Bool, newValue: Bool)` | `a = a && b` |
| `logicalOr(_: Bool, ordering: AtomicUpdateOrdering)` | `(oldValue: Bool, newValue: Bool)` | `a = a \|\| b` |
| `logicalXor(_: Bool, ordering: AtomicUpdateOrdering)` | `(oldValue: Bool, newValue: Bool)` | `a = a != b` |

Like the integer operations, all of these boolean operations are marked as `@discardableResult`.

`Atomic<Value>` exposes these operations when `Value` is `Bool`.

```swift
extension Atomic where Value == Bool {...}

let tracker = Atomic<Bool>(false)
let newOr = tracker.logicalOr(true, ordering: .relaxed).newValue
```

### Atomic Lazy References

The operations provided by `Atomic<Unmanaged<T>>` only operate on the unmanaged reference itself. They don't allow us to directly access the referenced object -- we need to manually invoke the methods `Unmanaged` provides for this purpose (usually, `takeUnretainedValue`).

Note that loading the atomic unmanaged reference and converting it to a strong reference are two distinct operations that won't execute as a single atomic transaction. This can easily lead to race conditions when a thread releases an object while another is busy loading it:

```swift
// BROKEN CODE. DO NOT EMULATE IN PRODUCTION.
let myAtomicRef = Atomic<Unmanaged<Foo>>(...)

// Thread A: Load the unmanaged value and then convert it to a regular
//           strong reference.
let ref = myAtomicRef.load(ordering: .acquiring).takeUnretainedValue()
...

// Thread B: Store a new reference in the atomic unmanaged value and 
//           release the previous reference.
let new = Unmanaged.passRetained(...)
let old = myAtomicRef.exchange(new, ordering: .acquiringAndReleasing)
old.release() // RACE CONDITION
```

If thread B happens to release the same object that thread A is in the process of loading, then thread A's `takeUnretainedValue` may attempt to retain a deallocated object.

Such problems make `Atomic<Unmanaged<T>>` exceedingly difficult to use in all but the simplest situations. The section on [*Atomic Strong References*](#atomic-strong-references-and-the-problem-of-memory-reclamation) below describes some new constructs we may introduce in future proposals to assist with this issue.

For now, we provide the standalone type `AtomicLazyReference`; this is an example of a useful construct that could be built on top of `Atomic<Unmanaged<Instance>>` operations. (Of the atomic constructs introduced in this proposal, only `AtomicLazyReference` represents a regular strong reference to a class instance -- the other pointer/reference types leave memory management entirely up to the user.)

An `AtomicLazyReference` holds an optional reference that is initially set to `nil`. The value can be set exactly once, but it can be read an arbitrary number of times. Attempts to change the value after the first `storeIfNil` call are ignored, and return the current value instead.

```swift
/// A lazily initializable atomic strong reference.
///
/// These values can be set (initialized) exactly once, but read many
/// times.
public struct AtomicLazyReference<Instance: AnyObject>: ~Copyable {
  /// The value logically stored in an atomic lazy reference value.
  public typealias Value = Instance?

  /// Initializes a new managed atomic lazy reference with a nil value.
  public init()
}

extension AtomicLazyReference {
  /// Atomically initializes this reference if its current value is nil, then
  /// returns the initialized value. If this reference is already initialized,
  /// then `storeIfNil(_:)` discards its supplied argument and returns
  /// the current value without updating it.
  ///
  /// The following example demonstrates how this can be used to implement a
  /// thread-safe lazily initialized reference:
  ///
  /// ```
  /// class Image {
  ///   var _histogram: AtomicLazyReference<Histogram> = .init()
  ///
  ///   // This is safe to call concurrently from multiple threads.
  ///   var atomicLazyHistogram: Histogram {
  ///     if let histogram = _histogram.load() { return histogram }
  ///     // Note that code here may run concurrently on
  ///     // multiple threads, but only one of them will get to
  ///     // succeed setting the reference.
  ///     let histogram = ...
  ///     return _histogram.storeIfNil(histogram)
  /// }
  /// ```
  ///
  /// This operation uses acquiring-and-releasing memory ordering.
  public borrowing func storeIfNil(
    _ desired: consuming Instance
  ) -> Instance

  /// Atomically loads and returns the current value of this reference.
  ///
  /// The load operation is performed with the memory ordering
  /// `AtomicLoadOrdering.acquiring`.
  public borrowing func load() -> Instance?
}
```

This is the only atomic type in this proposal that doesn't provide the usual `load`/`store`/`exchange`/`compareExchange` operations.

This construct allows library authors to implement a thread-safe lazy initialization pattern:

```swift
let _foo: AtomicLazyReference<Foo> = ...

// This is safe to call concurrently from multiple threads.
nonisolated var atomicLazyFoo: Foo {
  if let foo = _foo.load() { return foo }
  // Note: the code here may run concurrently on multiple threads.
  // All but one of the resulting values will be discarded.
  let foo = Foo()
  return _foo.storeIfNil(foo)
}
```

The Standard Library has been internally using such a pattern to implement deferred bridging for `Array`, `Dictionary` and `Set`.

Note that unlike the rest of the atomic types, `load` and `storeIfNil(_:)` do not expose `ordering` parameters. (Internally, they map to acquiring/releasing operations to guarantee correct synchronization.)

### Restricting Ordering Arguments to Compile-Time Constants

Modeling orderings as regular function parameters allows us to specify them using syntax that's familiar to all Swift programmers. Unfortunately, it means that in the implementation of atomic operations we're forced to switch over the ordering argument:

```swift
extension Atomic where Value.AtomicRepresentation == {U}IntNN.AtomicRepresentation {
  public borrowing func compareExchange(
    expected: consuming Value,
    desired: consuming Value,
    ordering: AtomicUpdateOrdering
  ) -> (exchanged: Bool, original: Int) {
    // Note: This is a simplified version of the actual implementation
    let won: Bool
    let oldValue: Value

    switch ordering {
    case .relaxed:
      (oldValue, won) = Builtin.cmpxchg_monotonic_monotonic_IntNN(
        address, expected, desired
      )

    case .acquiring:
      (oldValue, won) = Builtin.cmpxchg_acquire_acquire_IntNN(
        address, expected, desired
      )

    case .releasing:
      (oldValue, won) = Builtin.cmpxchg_release_monotonic_IntNN(
        address, expected, desired
      )

    case .acquiringAndReleasing:
      (oldValue, won) = Builtin.cmpxchg_acqrel_acquire_IntNN(
        address, expected, desired
      )

    case .sequentiallyConsistent:
      (oldValue, won) = Builtin.cmpxchg_seqcst_seqcst_IntNN(
        address, expected, desired
      )

    default:
      fatalError("Unknown atomic memory ordering")
    }

    return (exchanged: won, original: oldValue)
  }
}
```

Given our requirement that primitive atomics must always compile down to the actual atomic instructions with minimal additional overhead, we must guarantee that these switch statements always get optimized away into the single case we need; they must never actually be evaluated at runtime.

Luckily, configuring these special functions to always get force-inlined into all callers guarantees that constant folding will get rid of the switch statement *as long as the supplied ordering is a compile-time constant*. Unfortunately, it's all too easy to accidentally violate this latter requirement, with dire consequences to the expected performance of the atomic operation.

Consider the following well-meaning attempt at using `compareExchange` to define an atomic integer addition operation that traps on overflow rather than allowing the result to wrap around:

```swift
extension Atomic where Value == Int {
  // Non-inlinable
  public func add(_ operand: Int, ordering: AtomicUpdateOrdering) {
    var done = false
    var current = load(ordering: .relaxed)

    while !done {
      (done, current) = compareExchange(
        expected: current,
        desired: current + operand, // Traps on overflow
        ordering: ordering
      )
    }
  }
}

// Elsewhere:
counter.add(1, ordering: .relaxed)
```

If for whatever reason the Swift compiler isn't able (or willing) to inline the `add` call, then the value of `ordering` won't be known at compile time to the body of the function, so even though `compareExchange` will still get inlined, its switch statement won't be eliminated. This leads to a potentially significant performance regression that could interfere with the scalability of the operation.

The big issue here is that if `add` is in another module, then callers of this function have no visibility inside this function's body. If callers can't see this function's implementation, then the switch statement will be executed at runtime regardless of the compiler optimization mode. However, another issue is that the ordering argument may still be dynamic in which case the compiler still can't eliminate the switch statement even though the caller may be able to see the entire implementation.

To prevent the last issue, the memory ordering arguments of all atomic operations must be compile-time constants. Any attempt to pass a dynamic ordering value (such as in the `compareExchange` call above) will result in a compile-time error.

An ordering expression will be considered constant-evaluable if it's either (1) a direct call to one of the `Atomic*Ordering` factory methods (`.relaxed`, `.acquiring`, etc.), or (2) it is a direct reference to a variable that is in turn constrained to be constant-evaluable.

## Interaction with Existing Language Features

Please refer to the [Clarify the Swift memory consistency model (SE-0282)](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0282-atomics.md#interaction-with-non-instantaneous-accesses) proposal which goes over how atomics interact with the Law of Exclusivity, Non-Instantaneous Accesses, and Implicit Pointer Conversions.

An additional note with regards to the Law of Exclusivity, atomic values should never be declared with a `var` binding, always prefer a `let` one. Consider the following:

```swift
class Counter {
  var value: Atomic<Int>
}
```

By declaring this variable as a `var`, we opt into Swift's dynamic exclusivity checking for this property, so all non-exclusive accesses incur a runtime check to see if there is an active exclusive (e.g. mutating) access. This inherently means that atomic operations through such a variable will incur undesirable runtime overhead -- they are no longer purely atomic. (Even if the check never actually triggers a trap.)

To prevent users from accidentally falling into this trap, `Atomic` (and `AtomicLazyReference`) will not support `var` bindings. It is a compile-time error to have a `var` that has an explicit or inferred type of `Atomic`.

```swift
// error: variable of type 'Atomic<Int>' must be declared with a 'let'
var myAtomic = Atomic<Int>(123)
```

By making this a compiler error, we can safely assume that atomic accesses will never incur an unexpected dynamic exclusivity check. It is forbidden to create mutable variables of type `struct Atomic`.

Similarly, it is an error to declare a computed property that returns an `Atomic`, as its getter would need to create and return a new instance each time it is accessed. Instead, you can return the actual value that would be the initial value for the atomic:

```swift
var computedInt: Int {
  123
}

let myAtomic = Atomic<Int>(computedInt)
```

Alternatively, you can choose to convert the property to a function. This makes it clear that a new instance is being returned every time the function is called:

```swift
func makeAnAtomic() -> Atomic<Int> {
  Atomic<Int>(123)
}

let myAtomic = makeAnAtomic()
```



In the same vein, these types must never be passed as `inout` parameters as that declares that the callee has exclusive access to the atomic, which would make the access no longer atomic. Attemping to create an `inout` binding for an atomic variable is also a compile-time error. Parameters that are used to pass `Atomic` values must either be `borrowing` or `consuming`. (Passing a variable as `consuming` is also an exclusive access, but it's destroying the original variable, so we no longer need to care for its atomicity.)

```swift
// error: parameter of type 'Atomic<Int>' must be declared as either 'borrowing' or 'consuming'
func passAtomic(_: inout Atomic<Int>)
```

Mutating methods on atomic types are also forbidden, as they introduce `inout` bindings on `self`. For example, trying to extend `Atomic` with our own `mutating` method results in a compile-time error:

```swift
extension Atomic {
  // error: type `Atomic` cannot have mutating function 'greet()'
  mutating func greet() {
    print("Hello! From: Atomic")
  }
}
```

These conditions for `Atomic` and `AtomicLazyReference` are important to prevent users from accidentally introducing dynamic exclusivity for these low-level performance sensitive concurrency primitives.

### Interaction with Swift Concurrency

The `Atomic` type is `Sendable` where `Value: Sendable`. One can pass a value of an `Atomic` to an actor or any other async context by using a borrow reference.

```swift
actor Updater {
  ...

  func update(_ counter: borrowing Atomic<Int>) {
    ...
  }

  func doOtherWork() {}
}

func version1() async {
  let counter = Atomic<Int>(0)

  //                   |--- There are no suspension points in this function, so
  //                   |    this atomic value will be allocated on whatever
  //                   |    thread's stack that decides to run this async function.
  //                   |
  //                   v
  let updatedCount = counter.load(ordering: .relaxed)
}

func version2() async {
  let counter = Atomic<Int>(0)

  let updater = Updater()
  await updater.update(counter) //  <--------- Potential suspension point that
                                //             uses the atomic value directly.
                                //             The atomic value will get
                                //             promoted to the async stack frame
                                //             meaning it will be available until
                                //             this async function has ended.

  //                   |----- Atomic value used after suspension point.
  //                   |      Because of the fact that we're passing it to a
  //                   |      suspension point, it's already been allocated on
  //                   |      the async stack frame, and accessing it later will
  //                   |      access that same resource, preserving atomicity.
  //                   |
  //                   v
  let updatedCount = counter.load(ordering: .relaxed)
}

func version3() async {
  let counter = Atomic<Int>(0)

  let updater = Updater()
  await updater.doOtherWork() //  <--------- Potential suspension point that
                              //             doesn't use the atomic value directly.


  //                   |----- Atomic value used after suspension point, so it is
  //                   |      promoted to the async stack frame which makes this
  //                   |      value's lifetime persist even after the await.
  //                   |      The compiler could in theory also reorder the
  //                   |      atomic value's initialization after the suspension
  //                   |      because it isn't used before nor during meaning it
  //                   |      could be allocated on whatever thread's stack frame.
  //                   |
  //                   v
  let updatedCount = counter.load(ordering: .relaxed)
}
```

Variables of type `struct Atomic` are always located at a single, stable memory location, no matter its nature (be that a stored property in a class type or a noncopyable struct, an associated value in a noncopyable enum, a local variable that got promoted to the heap through a closure capture, or any other kind of variable.)

Considering these factors, we can safely declare that `struct Atomic` is `Sendable` whenever its value is `Sendable`. By analogue reasoning, `struct AtomicLazyReference` is declared `Sendable` whenever its instance is `Sendable`.

## Detailed Design

In the interest of keeping this document (relatively) short, the following API synopsis does not include API documentation, inlinable method bodies, or `@usableFromInline` declarations, and omits most attributes (`@available`, `@inlinable`, etc.).

To allow atomic operations to compile down to their corresponding CPU instructions, most entry points listed here will be defined `@inlinable`.

For the full API definition, please refer to the [implementation](https://github.com/apple/swift/pull/68857).

### Atomic Memory Orderings

```swift
public struct AtomicLoadOrdering: Equatable, Hashable, CustomStringConvertible {
  public static var relaxed: Self { get }
  public static var acquiring: Self { get }
  public static var sequentiallyConsistent: Self { get }

  public static func ==(left: Self, right: Self) -> Bool
  public func hash(into hasher: inout Hasher)
  public var description: String { get }
}

public struct AtomicStoreOrdering: Equatable, Hashable, CustomStringConvertible {
  public static var relaxed: Self { get }
  public static var releasing: Self { get }
  public static var sequentiallyConsistent: Self { get }

  public static func ==(left: Self, right: Self) -> Bool
  public func hash(into hasher: inout Hasher)
  public var description: String { get }
}

public struct AtomicUpdateOrdering: Equatable, Hashable, CustomStringConvertible {
  public static var relaxed: Self { get }
  public static var acquiring: Self { get }
  public static var releasing: Self { get }
  public static var acquiringAndReleasing: Self { get }
  public static var sequentiallyConsistent: Self { get }

  public static func ==(left: Self, right: Self) -> Bool
  public func hash(into hasher: inout Hasher)
  public var description: String { get }
}

public func atomicMemoryFence(ordering: AtomicUpdateOrdering)
```

### Atomic Protocols

#### `AtomicRepresentable`

```swift
public protocol AtomicRepresentable {
  associatedtype AtomicRepresentation

  static func encodeAtomicRepresentation(
    _ value: consuming Self
  ) -> AtomicRepresentation

  static func decodeAtomicRepresentation(
    _ representation: consuming AtomicRepresentation
  ) -> Self
}
```

The requirements set up a bidirectional mapping between values of the atomic type and an associated storage representation that supplies the actual primitive atomic operations.

Conforming types:

```swift
extension Int: AtomicRepresentable {...}
extension Int64: AtomicRepresentable {...}
extension Int32: AtomicRepresentable {...}
extension Int16: AtomicRepresentable {...}
extension Int8: AtomicRepresentable {...}
extension UInt: AtomicRepresentable {...}
extension UInt64: AtomicRepresentable {...}
extension UInt32: AtomicRepresentable {...}
extension UInt16: AtomicRepresentable {...}
extension UInt8: AtomicRepresentable {...}

extension Bool: AtomicRepresentable {...}

extension Float16: AtomicRepresentable {...}
extension Float: AtomicRepresentable {...}
extension Double: AtomicRepresentable {...}

extension WordPair: AtomicRepresentable {...}
extension Duration: AtomicRepresentable {...}

extension Never: AtomicRepresentable {...}

extension UnsafeRawPointer: AtomicRepresentable {...}
extension UnsafeMutableRawPointer: AtomicRepresentable {...}
extension UnsafePointer: AtomicRepresentable {...}
extension UnsafeMutablePointer: AtomicRepresentable {...}
extension Unmanaged: AtomicRepresentable {...}
extension OpaquePointer: AtomicRepresentable {...}
extension ObjectIdentifier: AtomicRepresentable {...}

extension UnsafeBufferPointer: AtomicRepresentable {...}
extension UnsafeMutableBufferPointer: AtomicRepresentable {...}
extension UnsafeRawBufferPointer: AtomicRepresentable {...}
extension UnsafeMutableRawBufferPointer: AtomicRepresentable {...}

extension Optional: AtomicRepresentable where Wrapped: AtomicOptionalRepresentable {...}
```

To support custom "atomic-representable" types, `AtomicRepresentable` also comes with default implementations for all its requirements for `RawRepresentable` types whose `RawValue` is also atomic:

```swift
extension RawRepresentable where Self: AtomicRepresentable, RawValue: AtomicRepresentable {
  // Implementations for all requirements.
}
```

The default implementations work by converting values to their `rawValue` form, and forwarding all atomic operations to it.

#### `AtomicOptionalRepresentable`

```swift
public protocol AtomicOptionalRepresentable: AtomicRepresentable {
  associatedtype AtomicOptionalRepresentation

  static func encodeAtomicOptionalRepresentation(
    _ value: consuming Self?
  ) -> AtomicOptionalRepresentation

  static func decodeAtomicOptionalRepresentation(
    _ representation: consuming AtomicOptionalRepresentation
  ) -> Self?
}
```

Atomic `Optional` operations are currently enabled for the following `Wrapped` types:

```swift
extension UnsafeRawPointer: AtomicOptionalRepresentable {}
extension UnsafeMutableRawPointer: AtomicOptionalRepresentable {}
extension UnsafePointer: AtomicOptionalRepresentable {}
extension UnsafeMutablePointer: AtomicOptionalRepresentable {}
extension Unmanaged: AtomicOptionalRepresentable {}
extension OpaquePointer: AtomicOptionalRepresentable {}
extension ObjectIdentifier: AtomicOptionalRepresentable {}
```

### `WordPair`

```swift
public struct WordPair {
  public var first: UInt { get }
  public var second: UInt { get }

  public init(first: UInt, second: UInt)
}

extension WordPair: AtomicRepresentable {...}
extension WordPair: Equatable {...}
extension WordPair: Hashable {...}

// NOTE: WordPair is semantically a (UInt, UInt). Tuple comparability
// works based of lexicographical ordering, so WordPair will do
// the same. It will compare 'first' first, and 'second' second.
extension WordPair: Comparable {...}

extension WordPair: CustomStringConvertible {...}
extension WordPair: CustomDebugStringConvertible {...}
extension WordPair: Sendable {}
```

### Atomic Types

#### `Atomic<Value>`

```swift
public struct Atomic<Value: AtomicRepresentable>: ~Copyable {
  public init(_ initialValue: consuming Value)
}

extension Atomic where Value.AtomicRepresentation == {U}IntNN.AtomicRepresentation {
  // Atomic operations:

  public borrowing func load(
    ordering: AtomicLoadOrdering
  ) -> Value

  public borrowing func store(
    _ desired: consuming Value,
    ordering: AtomicStoreOrdering
  )

  public borrowing func exchange(
    _ desired: consuming Value,
    ordering: AtomicUpdateOrdering
  ) -> Value

  public borrowing func compareExchange(
    expected: consuming Value,
    desired: consuming Value,
    ordering: AtomicUpdateOrdering
  ) -> (exchanged: Bool, original: Value)

  public borrowing func compareExchange(
    expected: consuming Value,
    desired: consuming Value,
    successOrdering: AtomicUpdateOrdering,
    failureOrdering: AtomicLoadOrdering
  ) -> (exchanged: Bool, original: Value)

  public borrowing func weakCompareExchange(
    expected: consuming Value,
    desired: consuming Value,
    ordering: AtomicUpdateOrdering
  ) -> (exchanged: Bool, original: Value)

  public borrowing func weakCompareExchange(
    expected: consuming Value,
    desired: consuming Value,
    successOrdering: AtomicUpdateOrdering,
    failureOrdering: AtomicLoadOrdering
  ) -> (exchanged: Bool, original: Value)
}

extension Atomic: @unchecked Sendable where Value: Sendable {}
```

`Atomic` also provides a handful of integer operations for the standard fixed-width integer types. This is implemented via same type requirements:

```swift
extension Atomic where Value == Int {
  @discardableResult
  public borrowing func wrappingAdd(
    _ operand: Value,
    ordering: AtomicUpdateOrdering
  ) -> (oldValue: Value, newValue: Value)

  @discardableResult
  public borrowing func wrappingSubtract(
    _ operand: Value,
    ordering: AtomicUpdateOrdering
  ) -> (oldValue: Value, newValue: Value)

  @discardableResult
  public borrowing func add(
    _ operand: Value,
    ordering: AtomicUpdateOrdering
  ) -> (oldValue: Value, newValue: Value)
  
  @discardableResult
  public borrowing func subtract(
    _ operand: Value,
    ordering: AtomicUpdateOrdering
  ) -> (oldValue: Value, newValue: Value)
  
  @discardableResult
  public borrowing func bitwiseAnd(
    _ operand: Value,
    ordering: AtomicUpdateOrdering
  ) -> (oldValue: Value, newValue: Value)

  @discardableResult
  public borrowing func bitwiseOr(
    _ operand: Value,
    ordering: AtomicUpdateOrdering
  ) -> (oldValue: Value, newValue: Value)

  @discardableResult
  public borrowing func bitwiseXor(
    _ operand: Value,
    ordering: AtomicUpdateOrdering
  ) -> (oldValue: Value, newValue: Value)

  @discardableResult
  public borrowing func min(
    _ operand: Value,
    ordering: AtomicUpdateOrdering
  ) -> (oldValue: Value, newValue: Value)

  @discardableResult
  public borrowing func max(
    _ operand: Value,
    ordering: AtomicUpdateOrdering
  ) -> (oldValue: Value, newValue: Value)
}

extension Atomic where Value == Int8 {...}
...
```

as well as providing convenience functions for boolean operations:

```swift
extension Atomic where Value == Bool {
  @discardableResult
  public borrowing func logicalAnd(
    _ operand: Value,
    ordering: AtomicUpdateOrdering
  ) -> (oldValue: Value, newValue: Value)

  @discardableResult
  public borrowing func logicalOr(
    _ operand: Value,
    ordering: AtomicUpdateOrdering
  ) -> (oldValue: Value, newValue: Value)

  @discardableResult
  public borrowing func logicalXor(
    _ operand: Value,
    ordering: AtomicUpdateOrdering
  ) -> (oldValue: Value, newValue: Value)
}
```

#### `AtomicLazyReference<Instance>`

```swift
public struct AtomicLazyReference<Instance: AnyObject>: ~Copyable {
  public typealias Value = Instance?

  public init(_ initialValue: consuming Instance)

  // Atomic operations:

  public borrowing func storeIfNil(
    _ desired: consuming Instance
  ) -> Instance

  public borrowing func load() -> Instance?
}

extension AtomicLazyReference: @unchecked Sendable where Instance: Sendable {}
```

## Source Compatibility

This is a purely additive change with no source compatibility impact.

## Effect on ABI Stability

This proposal introduces new entry points to the Standard Library ABI in a standalone `Synchronization` module, but otherwise it has no effect on ABI stability.

On ABI-stable platforms, the struct types and protocols introduced here will become part of the stdlib's ABI with availability matching the first OS releases that include them.

Most of the atomic methods introduced in this document will be force-inlined directly into client code at every call site. As such, there is no reason to bake them into the stdlib's ABI -- the stdlib binary will not export symbols for them.

## Effect on API Resilience

This is an additive change; it has no effect on the API of existing code.

For the new constructs introduced here, the proposed design allows us to make the following changes in future versions of the Swift Standard Library:

- Addition of new atomic types (and higher-level constructs built around them). (These new types would not directly back-deploy to OS versions that predate their introduction.)

- Addition of new memory orderings. Because all atomic operations compile directly into user code, new memory orderings that we decide to introduce later could potentially back-deploy to any OS release that includes this proposal.

- Addition of new atomic operations on the types introduced here. These would be also be back deployable.

- Introducing a default memory ordering for atomic operations (either by adding a default value to `ordering`, or by adding new overloads that lack that parameter). This too would be a back-deployable change.

- Change the memory ordering model as long as the changes preserve source compatibility.

(We don't necessarily plan to actually perform any of these changes; we merely leave the door open to doing them.)

## Potential Future Directions

### Atomic Strong References and The Problem of Memory Reclamation

Perhaps counter-intuitively, implementing a high-performance, *lock-free* atomic version of regular everyday strong references is not a trivial task. This proposal doesn't attempt to provide such a construct beyond the limited use-case of `AtomicLazyReference`.

Under the hood, Swift's strong references have always been using atomic operations to implement reference counting. This allows references to be read (but not mutated) from multiple, concurrent threads of execution, while also ensuring that each object still gets deallocated as soon as its last outstanding reference disappears. However, atomic reference counts on their own do not allow threads to safely share a single *mutable* reference without additional synchronization.

The difficulty is in the implementation of the atomic load operation, which boils down to two separate sub-operations, both of which need to be part of the *same atomic transaction*:

1. Load the value of the reference.
2. Increment the reference count of the corresponding object.

If an intervening store operation were allowed to release the reference between steps 1 and 2, then the loaded reference could already be deallocated by the time `load` tries to increment its refcount.

Without an efficient way to implement these two steps as a single atomic transaction, the implementation of `store` needs to delay releasing the overwritten value until it can guarantee that every outstanding load operation is completed. Exactly how to implement this is the problem of *memory reclamation* in concurrent data structures.

There are a variety of approaches to tackle this problem, but the one we think would be the best fit is the implementation of [`AtomicReference`][https://swiftpackageindex.com/apple/swift-atomics/1.2.0/documentation/atomics/atomicreference] in the [swift-atomics package](https://github.com/apple/swift-atomics).

### Additional Low-Level Atomic Features

To enable use cases that require even more fine-grained control over atomic operations, it may be useful to introduce additional low-level atomics features:

* support for additional kinds of atomic values (such as floating-point atomics [[P0020]]),
* new memory orderings, such as a consuming load ordering [[P0750]] or tearable atomics [[P0690]],
* "volatile" atomics that prevent certain compiler optimizations
* and more

We defer these for future proposals.

## Alternatives Considered

### Default Orderings

We considered defaulting all atomic operations to sequentially consistent ordering. While we concede that doing so would make atomics slightly more approachable, implicit ordering values tend to interfere with highly performance-sensitive use cases of atomics (which is *most* use cases of atomics). Sequential consistency tends to be relatively rarely used in these contexts, and implicitly defaulting to it would allow accidental use to easily slip through code review.

Users who wish for default orderings are welcome to define their own overloads for atomic operations:

```swift
extension Atomic where Value.AtomicRepresentation == UInt8.AtomicRepresentation {
  func load() -> Value { 
    load(ordering: .sequentiallyConsistent)
  }

  func store(_ desired: consuming Value) { 
    store(desired, ordering: .sequentiallyConsistent) 
  }

  func exchange(_ desired: consuming Value) -> Value {
    exchange(desired, ordering: .sequentiallyConsistent)
  }
  
  func compareExchange(
    expected: consuming Value,
    desired: consuming Value
  ) -> (exchanged: Bool, original: Value) {
    compareExchange(
      expected: expected, 
      desired: desired, 
      ordering: .sequentiallyConsistent
    )
  }

  func weakCompareExchange(
    expected: consuming Value,
    desired: consuming Value
  ) -> (exchanged: Bool, original: Value) {
    weakCompareExchange(
      expected: expected, 
      desired: desired, 
      successOrdering: .sequentiallyConsistent,
      failureOrdering: .sequentiallyConsistent
    )
  }
}

...

extension Atomic where Value == Int {
  func wrappingAdd(_ operand: Value) {
    wrappingAdd(operand, ordering: .sequentiallyConsistent)
  }

  etc.
}

...
```

### A Truly Universal Generic Atomic Type

While future proposals may add a variety of other atomic types, we do not expect to ever provide a truly universal generic `Atomic<T>` construct. The `Atomic<Value>` type is designed to provide high-performance lock-free primitives, and these are heavily constrained by the atomic instruction sets of the CPU architectures Swift targets.

A universal `Atomic<T>` type that can hold *any* value is unlikely to be implementable without locks, so it is outside the scope of this proposal. We may eventually consider adding such a construct in a future concurrency proposal:

```swift
struct Serialized<Value>: ~Copyable {
  private let _lock = UnfairLock()
  private var _value: Value
  
  func withLock<T>(_ body: (inout Value) throws -> T) rethrows -> T {
    _lock.lock()
    defer { _lock.unlock() }

    return try body(&_value)
  }
}
```

### Providing a `value` Property

Our atomic constructs are unusual because even though semantically they behave like containers holding a value, they do not provide direct access to it. Instead of exposing a getter and a setter on a handy `value` property, they expose cumbersome `load` and `store` methods. There are two reasons for this curious inconsistency:

First, there is the obvious issue that property getter/setters have no room for an ordering parameter.

Second, there is a deep underlying problem with the property syntax: it encourages silent race conditions. For example, consider the code below:

```swift
let counter = Atomic<Int>(0)
...
counter.value += 1
```

Even though this increment looks like it may be a single atomic operation, it gets executed as two separate atomic transactions:

```swift
var temp = counter.value // atomic load
temp += 1
counter.value = temp     // atomic store
```

If some other thread happens to update the value after the atomic load, then that update gets overwritten by the subsequent store, resulting in data loss.

To prevent this gotcha, none of the proposed atomic types provide a property for accessing their value, and we don't foresee adding such a property in the future, either.

(Note that this problem cannot be mitigated by implementing [modify accessors]. Lock-free updates cannot be implemented without the ability to retry the update multiple times, and modify accessors can only yield once.)

[modify accessors]: https://forums.swift.org/t/modify-accessors/31872

### Alternative Designs for Memory Orderings

Modeling memory orderings with enumeration(-like) values fits well into the Standard Library's existing API design practice, but `ordering` arguments aren't without problems. Most importantly, the quality of code generation depends greatly on the compiler's ability to constant-fold switch statements over these ordering values into a single instruction. This can be fragile -- especially in unoptimized builds. We think [constraining these arguments to compile-time constants](#restricting-ordering-arguments-to-compile-time-constants) strikes a good balance between readability and performance, but it's instructive to look at some of the approaches we considered before settling on this choice.

#### Encode Orderings in Method Names

One obvious idea is to put the ordering values directly in the method name for every atomic operation. This would be easy to implement but it leads to practically unusable API names. Consider the two-ordering compare/exchange variant below:

```swift
flag.sequentiallyConsistentButAcquiringAndReleasingOnFailureCompareExchange(
  expected: 0,
  desired: 1
)
```

We could find shorter names for the orderings (`Serialized`, `Barrier` etc.), but ultimately the problem is that this approach tries to cram too much information into the method name, and the resulting multitude of similar-but-not-exactly-the-same methods become an ill-structured mess.

#### Orderings As Generic Type Parameters

A second idea is model the orderings as generic type parameters on the atomic types themselves.

```swift
struct Atomic<Value: AtomicProtocol, Ordering: AtomicMemoryOrdering> {
  ...
}

let counter = Atomic<Int, Relaxed>(0)
counter.wrappingAdd(1)
```

This simplifies the typical case where all operations on a certain atomic value use the same "level" of ordering (relaxed, acquire/release, or sequentially consistent). However, there are considerable drawbacks:

* This design puts the ordering specification far away from the actual operations -- obfuscating their meaning. 
* It makes it a lot more difficult to use custom orderings for specific operations (like the speculative relaxed load in the `wrappingAdd` example in the section on [Atomic Operations](#atomic-operations) above).
* We wouldn't be able to provide a default value for a generic type parameter. 
* Finally, there is also the risk of unspecialized generics interfering with runtime performance.

#### Ordering Views

The most promising alternative idea to represent memory orderings was to model them like `String`'s encoding views:

```swift
let counter = Atomic<Int>(0)

counter.relaxed.wrappingAdd(1)

let current = counter.acquiring.load()
```

There are some things that we really like about this "ordering view" approach:

- It eliminates the need to ever switch over orderings, preventing any and all constant folding issues.
- It makes it obvious that memory orderings are supposed to be compile-time parameters.
- The syntax is arguably more attractive.

However, we ultimately decided against going down this route, for the following reasons:

  - **Composability.** Such ordering views are unwieldy for the variant of `compareExchange` that takes separate success/failure orderings. Ordering views don't nest very well at all:

    ```swift
    counter.acquiringAndReleasing.butAcquiringOnFailure.compareExchange(...)
    ```

  - **API surface area and complexity.** Ordering views act like a multiplier for API entry points. In our prototype implementation, introducing ordering views increased the API surface area of atomics by 3×: we went from 6 public structs with 53 public methods to 27 structs with 175 methods. While clever use of protocols and generics could reduce this factor, the increased complexity seems undesirable. (E.g., generic ordering views would reintroduce potential performance problems in the form of unspecialized generics.)

    API surface area is not necessarily the most important statistic, but public methods do have some cost. (In e.g. the size of the stdlib module, API documentation etc.)

  - **Unintuitive syntax.** While the syntax is indeed superficially attractive, it feels backward to put the memory ordering *before* the actual operation. While memory orderings are important, I suspect most people would consider them secondary to the operations themselves.

  - **Limited Reuse.** Implementing ordering views takes a rather large amount of (error-prone) boilerplate-heavy code that is not directly reusable. Every new atomic type would need to implement a new set of ordering views, tailor-fit to its own use-case.

#### Memory Orderings as Overloads

Another promising alternative was the idea to model each ordering as a separate type and have overloads for the various atomic operations.

```swift
struct AtomicMemoryOrdering {
  struct Relaxed {
    static var relaxed: Self { get }
  }
  
  struct Acquiring {
    static var acquiring: Self { get }
  }
  
  ...
}

extension Atomic where Value.AtomicRepresentation == {U}IntNN.AtomicRepresentation {
  func load(ordering: AtomicMemoryOrdering.Relaxed) -> Value {...}
  func load(ordering: AtomicMemoryOrdering.Acquiring) -> Value {...}
  ...
}
```

This approach shares a lot of the same benefits of views, but the biggest reason for this alternative was the fact that the switch statement problem we described earlier just doesn't exist anymore. There is no switch statement! The overload always gets resolved to a single atomic operation + ordering + storage meaning there's no question about what to compile the operation down to. However, this is just another type of flavor of views in that the API surface explodes especially with double ordering operations. 

There are 5 storage types and we define the primitive atomic operations on extensions of all of these. For the constant expression case for single ordering operations that's `5 (storage) * 1 (ordering) = 5` number of overloads and `5 (storage) * 1 (update ordering) * 1 (load ordering) = 5` for the double ordering case. The overload solution is now dependent on the number of orderings supported for a specific operation. So for single ordering loads it's `5 (storage) * 3 (orderings) = 15` different load orderings and for the double ordering compare and exchange it's `5 (storage) * 5 (update orderings) * 3 (load orderings) = 75` overloads.

|                               | Overloads                                                    | Constant Expressions                                         |
| ----------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Overload Resolution           | Very bad                                                     | Not so bad                                                   |
| API Documentation             | Very bad (but can be fixed!)                                 | Not so bad (but can be fixed!)                               |
| Custom Atomic Operations      | Requires users to define multiple overloads for their operations. | Allows users to define a single entrypoint that takes a constant ordering and passes that to the primitive atomic operations. |
| Back Deployable New Orderings | Almost impossible unless we defined the ordering types in C because types in Swift must come with availability. | Can easily be done because the orderings are static property getters that we can back deploy. |

The same argument for views creating a very vast API surface can be said about the overloads which helped us determine that the constant expression approach is still superior.

### Directly bring over `swift-atomics`'s API

The `swift-atomics` package has many years of experience using their APIs to interface with atomic values and it would be beneficial to simply bring over the same API. However, once we designed the general purpose `Atomic<Value>` type, we noticied a few deficiencies with the atomic protocol hierarchy that made using this type awkward for users. We've redesigned these protocols to make using the atomic type easier to use and easier to reason about.

While there are some API differences between this proposal and the package, most of the atomic operations are the same and should feel very familiar to those who have used the package before. We don't plan on drastically renaming any core atomic operation because we believe `swift-atomics` already got those names correct.

### A different name for `WordPair`

Previous revisions of this proposal named this type `DoubleWord`. This is a good name and is in fact the name used in the `swift-atomics` package. We felt the prefix `Double*` could cause confusion with the pre-existing type in the standard library `Double`. The name `WordPair` has a couple of advantages:

1. Code completion. Because this name starts with an less common letter in the English alphabet, the likelyhood of seeing this type at the top level in code completion is very unlikely and not generally a type used for newer programmers of Swift.
2. Directly conveys the semantic meaning of the type. This type is not semantically equivalent to something like `{U}Int128` (on 64 bit platforms). While although its layout shares the same size, the meaning we want to drive home with this type is quite simply that it's a pair of `UInt` words. If and when the standard library proposes a `{U}Int128` type, that will add a conformance to `AtomicRepresentable` on 64 bit platforms who support double-words as well. That itself wouldn't deprecate uses of `WordPair` however, because it's much easier to grab both words independently with `WordPair` as well as being a portable name for such semantics on both 32 bit and 64 bit platforms.

### A different name for the `Synchronization` module

In its [notes returning the initial version of this proposal for revision](https://forums.swift.org/t/returned-for-revision-se-0410-atomics/68522), the Swift Language Steering Group suggested the strawman name `Atomics` as a for this module. I think this name is far too restrictive because it prevents other similar low-level concurrency primitives or somewhat related features like volatile loads/stores from sharing a module. It would also be extremely source breaking for folks that upgrade their Swift SDK to a version that may include this proposed new module while depending on the existing [swift-atomics](https://github.com/apple/swift-atomics) whose module is also named `Atomics`.

We shouldn't be afraid of conflicting module names causing spurious source breaks when introducing a new module to the standard libraries; however, in this case, the direct precursor is prominently using this name, and reusing the same module name would cause significant breakage. We expect this package will need to remain in active use for a number of years, as it will be able to provide reimplementations of the constructs proposed here without the ABI availability constraints that come with Standard Library additions.

## References

[Clarify the Swift memory consistency model (SE-0282)]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0282-atomics.md
**\[Clarify the Swift memory consistency model (SE-0282)]** Karoy Lorenty. "Clarify the Swift memory consistency model."*Swift Evolution Proposal*, 2020. https://github.com/swiftlang/swift-evolution/blob/main/proposals/0282-atomics.md

[C++17]: https://isocpp.org/std/the-standard
**\[C++17]** ISO/IEC. *ISO International Standard ISO/IEC 14882:2017(E) – Programming Language C++.* 2017.
  https://isocpp.org/std/the-standard

[Boehm 2008]: https://doi.org/10.1145/1375581.1375591
**\[Boehm 2008]** Hans-J. Boehm, Sarita V. Adve. "Foundations of the C++ Concurrency Memory Model." In *PLDI '08: Proc. of the 29th ACM SIGPLAN Conf. on Programming Language Design and Implementation*, pages 68–78, June 2008.
  https://doi.org/10.1145/1375581.1375591

[N2153]: http://wg21.link/N2153
**\[N2153]** Raúl Silvera, Michael Wong, Paul McKenney, Bob Blainey. *A simple and efficient memory model for weakly-ordered architectures.* WG21/N2153, January 12, 2007. http://wg21.link/N2153

[P0020]: http://wg21.link/P0020
**\[P0020]** H. Carter Edwards, Hans Boehm, Olivier Giroux, JF Bastien, James Reus. *Floating Point Atomic.* WG21/P0020r6, November 10, 2017. http://wg21.link/P0020

[P0418]: http://wg21.link/P0418
**\[P0418]** JF Bastien, Hans-J. Boehm. *Fail or succeed: there is no atomic lattice.* WG21/P0417r2, November 9, 2016. http://wg21.link/P0418

[P0690]: http://wg21.link/P0690
**\[P0690]** JF Bastien, Billy Robert O'Neal III, Andrew Hunter. *Tearable Atomics.* WG21/P0690, February 10, 2018. http://wg21.link/P0690

[P0735]: http://wg21.link/P0735
**\[P0735]**: Will Deacon, Jade Alglave. *Interaction of `memory_order_consume` with release sequences.* WG21/P0735r1, June 17, 2019. http://wg21.link/P0735

[P0750]: http://wg21.link/P0750
**\[P0750]** JF Bastien, Paul E. McKinney. *Consume*. WG21/P0750, February 11, 2018. http://wg21.link/P0750 

⚛︎︎

<!-- Local Variables: -->
<!-- mode: markdown -->
<!-- fill-column: 10000 -->
<!-- eval: (setq-local whitespace-style '(face tabs newline empty)) -->
<!-- eval: (whitespace-mode 1) -->
<!-- eval: (visual-line-mode 1) -->
<!-- End: -->
