# Safe Access to Contiguous Storage

* Proposal: [SE-NNNN](nnnn-safe-shared-contiguous-storage.md)
* Authors: [Guillaume Lessard](https://github.com/glessard), [Andrew Trick](https://github.com/atrick)
* Review Manager: TBD
* Status: **Awaiting implementation**
* Roadmap: [BufferView Roadmap](https://forums.swift.org/t/66211)
* Bug: rdar://48132971, rdar://96837923
* Implementation: (pending)
* Upcoming Feature Flag: (pending)
* Review: ([pitch](https://forums.swift.org/t/69888))

## Introduction

We introduce `StorageView<T>`, an abstraction for container-agnostic access to contiguous memory. It will expand the expressivity of performant Swift code without giving up on the memory safety properties we rely on: temporal safety, spatial safety, definite initialization and type safety.

In the C family of programming languages, memory can be shared with any function by using a pointer and (ideally) a length. This allows contiguous memory to be shared with a function that doesn't know the layout of a struct being used by the caller. A heap-allocated array, contiguously-stored named fields or even a single stack-allocated instance can all be accessed through a C pointer. We aim to create a similar idiom in Swift, with no compromise to memory safety.

This proposal is related to two other features being proposed along with it: [non-escapable type constraint]() (`~Escapable`) and [compile-time lifetime dependency annotations](https://github.com/tbkka/swift-evolution/blob/tbkka-lifetime-dependency/proposals/NNNN-lifetime-dependency.md). This proposal also supersedes [SE-0256](https://github.com/apple/swift-evolution/blob/main/proposals/0256-contiguous-collection.md). The overall feature of ownership and lifetime constraints has previously been discussed in the [BufferView roadmap](https://forums.swift.org/t/66211) forum thread. Additionally, we refer to proposals for [`BitwiseCopyable`](https://forums.swift.org/t/69943) and [Non-copyable Generics](https://forums.swift.org/t/68180).

## Motivation

Consider for example a program using multiple libraries, including [base64](https://datatracker.ietf.org/doc/html/rfc4648) decoding. The program would obtain encoded data from one or more of its dependencies, which could supply it in the form of `[UInt8]`, `Foundation.Data` or even `String`, among others. None of these types is necessarily more correct than another, but the base64 decoding library must pick an input format. It could declare its input parameter type to be `some Sequence<UInt8>`, but such a generic function significantly limits performance. This may force the library author to either declare its entry point as inlinable, or to implement an internal fast path using `withContiguousStorageIfAvailable()` and use an unsafe type. The ideal interface would have a combination of the properties of both `some Sequence<UInt8>` and `UnsafeBufferPointer<UInt8>`.

## Proposed solution

`StorageView` will allow sharing the contiguous internal representation of a type, by providing access to a borrowed view of a span of contiguous memory. A view does not copy the underlying data: it instead relies on a guarantee that the original container cannot be modified or destroyed during the lifetime of the view. `StorageView`'s lifetime is statically enforced as a lifetime dependency to a binding of the type vending it, preventing its escape from the scope where it is valid for use. This guarantee preserves temporal safety. `StorageView` also performs bounds-checking on every access to preserve spatial safety. Additionally `StorageView` always represents initialized memory, preserving the definite initialization guarantee.

By relying on borrowing, `StorageView` can provide simultaneous access to a non-copyable container, and can help avoid unwanted copies of copyable containers. Note that `StorageView` is not a replacement for a copyable container with owned storage; see the future directions for more details ([Resizable, contiguously-stored, untyped collection in the standard library](#Bytes))

A type can indicate that it can provide a `StorageView` by conforming to the `ContiguousStorage` protocol. For example, for the hypothetical base64 decoding library mentioned above, a possible API could be:

```swift
extension HypotheticalBase64Decoder {
  public func decode(bytes: some ContiguousStorage<UInt8>) -> [UInt8]
}
```

## Detailed design

`StorageView<Element>` is a simple representation of a span of initialized memory.

```swift
public struct StorageView<Element: ~Copyable & ~Escapable>
: ~Escapable, Copyable {
  internal var _start: StorageViewIndex<Element>
  internal var _count: Int
}
```

It provides a collection-like interface to the elements stored in that span of memory:

```swift
extension StorageView {
	public struct Index: Copyable, Escapable, Strideable { /* .... */ }
  public struct Iterator: Copyable, ~Escapable {
    // Should conform to a `BorrowingIterator` protocol
    // that will be defined at a later date.
  }
  
  public typealias SubSequence: Self

  public var startIndex: Index { _read }
  public var endIndex: Index { _read }
  public var count: Int { get }

  public func makeIterator() -> copy(self) StorageViewIterator<Element>

  public var isEmpty: Bool { get }

  // index-based subscripts
  subscript(_ position: Index) -> copy(self) Element { _read }
  subscript(_ bounds: Range<Index>) -> copy(self) StorageView<Element> { _read }

  // integer-offset subscripts
  subscript(offset: Int) -> copy(self) Element { _read }
  subscript(offsets: Range<Int>) -> copy(self) StorageView<Element> { _read }
}

extension StorageView.Iterator where Element: Escapable, Copyable {
  // Cannot conform to `IteratorProtocol` because `Self: ~Escapable`
  public mutating func next() -> Element?
}
```

Note that `StorageView` does _not_ conform to `Collection`. This is because `Collection`, as originally conceived and enshrined in existing source code, assumes pervasive copyability and escapability for itself as well as its elements. In particular a subsequence of a `Collection` is semantically a separate value from the instance it was derived from. In the case of `StorageView`, the slice _must_ have the same lifetime as the view from which it originates. Another proposal will consider collection-like protocols to accommodate different combinations of `~Copyable` and `~Escapable` for the collection and its elements.

A type can declare that it can provide access to contiguous storage by conforming to the `ContiguousStorage` protocol:

```swift
public protocol ContiguousStorage<Element>: ~Copyable, ~Escapable {
  associatedtype Element: ~Copyable & ~Escapable

  var storage: borrow(self) StorageView<Element> { _read }
}
```

The key safety feature is that a `StorageView` cannot escape to a scope where the value it borrowed no longer exists.

An API that wishes to read from contiguous storage can declare a parameter type of `some ContiguousStorage`. The implementation will internally consist of a brief generic section, followed by business logic implemented in terms of a concrete `StorageView`. Frameworks that support library evolution (resilient frameworks) have an additional concern. Resilient frameworks have an ABI boundary that may differ from the API proper. Resilient frameworks may wish to adopt a pattern such as the following:

```swift
extension MyResilientType {
  // public API
  @inlinable public func essentialFunction(_ a: some ContiguousStorage<some Any>) -> Int {
    self.essentialFunction(a.storage)
  }

  // ABI boundary
  public func essentialFunction(_ a: StorageView<some Any>) -> Int { ... }
}
```

Here, the public function obtains the `StorageView` from the type that vends it in inlinable code, then calls a concrete, opaque function defined in terms of `StorageView`. Inlining the generic shim in the client is often a critical optimization. The need for such a pattern and related improvements are discussed in the future directions below (see [Syntactic Sugar for Automatic Conversions](#Conversions).)



#### Extensions to Standard Library and Foundation types

```swift
extension Array: ContiguousStorage<Self.Element> {
  var storageView: borrow(self) StorageView<Element> { _read }
}
extension ArraySlice: ContiguousStorage<Self.Element> {
  var storageView: borrow(self) StorageView<Element> { _read }
}
extension ContiguousArray: ContiguousStorage<Self.Element> {
  var storageView: borrow(self) StorageView<Element> { _read }
}

extension Foundation.Data: ContiguousStorage<UInt8> {
  var storageView: borrow(self) StorageView<UInt8> { _read }
}

extension String.UTF8View: ContiguousStorage<Unicode.UTF8.CodeUnit> {
  // note: this could borrow a temporary copy of the original `String`'s storage object
  var storageView: borrow(self) StorageView<Unicode.UTF8.CodeUnit> { _read }
}
extension Substring.UTF8View: ContiguousStorage<Unicode.UTF8.CodeUnit> {
  // note: this could borrow a temporary copy of the original `Substring`'s storage object
  var storageView: borrow(self) StorageView<Unicode.UTF8.CodeUnit> { _read }
}
extension Character.UTF8View: ContiguousStorage<Unicode.UTF8.CodeUnit> {
  // note: this could borrow a temporary copy of the original `Character`'s storage object
  var storageView: borrow(self) StorageView<Unicode.UTF8.CodeUnit> { _read }
}

extension SIMD: ContiguousStorage<Self.Scalar> {
  var storageView: borrow(self) StorageView<Self.Scalar> { _read }
}
extension KeyValuePairs: ContiguousStorage<(Self.Key, Self.Value)> {
  var storageView: borrow(self) StorageView<(Self.Key, Self.Value)> { _read }
}
extension CollectionOfOne: ContiguousStorage<Element> {
  var storageView: borrow(self) StorageView<Element> { _read }
}

extension Slice: ContiguousStorage where Base: ContiguousStorage {
  var storageView: borrow(self) StorageView<Base.Element> { _read }
}

extension UnsafeBufferPointer: ContiguousStorage<Self.Element> {
  // note: this applies additional preconditions to `self` for the duration of the borrow
  var storageView: borrow(self) StorageView<Element> { _read }
}
extension UnsafeMutableBufferPointer: ContiguousStorage<Self.Element> {
  // note: this applies additional preconditions to `self` for the duration of the borrow
  var storageView: borrow(self) StorageView<Element> { _read }
}
extension UnsafeRawBufferPointer: ContiguousStorage<UInt8> {
  // note: this applies additional preconditions to `self` for the duration of the borrow
  var storageView: borrow(self) StorageView<UInt8> { _read }
}
extension UnsafeMutableRawBufferPointer: ContiguousStorage<UInt8> {
  // note: this applies additional preconditions to `self` for the duration of the borrow
  var storageView: borrow(self) StorageView<UInt8> { _read }
}
```

#### Using `StorageView` with C functions or other unsafe code:

`StorageView` has an unsafe hatch for use with unsafe code.

```swift
extension StorageView {
  func withUnsafeBufferPointer<Result>(
    _ body: (_ buffer: UnsafeBufferPointer<Element>) -> Result
  ) -> Result
}

extension StorageView where Element: BitwiseCopyable {
  func withUnsafeBytes<Result>(
    _ body: (_ buffer: UnsafeRawBufferPointer) -> Result
  ) -> Result
}
```

#### Complete `StorageView` API:

```swift
public struct StorageView<Element: ~Copyable & ~Escapable>
: Copyable, ~Escapable {
  internal var _start: StorageViewIndex<Element>
  internal var _count: Int
}
```

##### Creating a `StorageView`:

The initialization of a `StorageView` instance is an unsafe operation. When it is initialized correctly, subsequent uses of the borrowed instance are safe. Typically these initializers will be used internally to a container's implementation of functions or computed properties that return a borrowed `StorageView`.

```swift
extension StorageView {

  /// Unsafely create a `StorageView` over a span of initialized memory.
  ///
  /// The memory must be owned by the instance `owner`, meaning that
  /// as long as `owner` is alive, then the memory will remain valid.
  ///
  /// - Parameters:
  ///   - unsafeBufferPointer: a buffer to initialized elements.
  ///   - owner: a binding whose lifetime must exceed that of
  ///            the returned `StorageView`.
  public init<Owner>(
    unsafeBufferPointer: UnsafeBufferPointer<Element>, owner: borrowing Owner
  ) -> borrow(owner) Self

  /// Unsafely create a `StorageView` over a span of initialized memory.
  ///
  /// The memory representing `count` instances starting at
  /// `unsafePointer` must be owned by the instance `owner`, meaning that
  /// as long as `owner` is alive, then the memory will remain valid.
  ///
  /// - Parameters:
  ///   - unsafePointer: a pointer to the first initialized element.
  ///   - count: the number of initialized elements in the view.
  ///   - owner: a binding whose lifetime must exceed that of
  ///            the returned `StorageView`.
  public init<Owner>(
    unsafePointer: UnsafePointer<Element>, count: Int, owner: borrowing Owner
  ) -> borrow(owner) Self
}

extension StorageView where Element: BitwiseCopyable {

  /// Unsafely create a `StorageView` over a span of initialized memory.
  ///
  /// The memory in `unsafeBytes` must be owned by the instance
  /// `owner`, meaning that as long as `owner` is alive, then the
  /// memory will remain valid.
  ///
  /// `unsafeBytes` must be correctly aligned for accessing
  /// an element of type `Element`, and must contain a number of bytes
  /// that is an exact multiple of `Element`'s stride.
  ///
  /// - Parameters:
  ///   - unsafeBytes: a buffer to initialized elements.
  ///   - type: the type to use when interpreting the bytes in memory.
  ///   - owner: a binding whose lifetime must exceed that of
  ///            the returned `StorageView`.
  public init<Owner>(
    unsafeBytes: UnsafeRawBufferPointer, as type: Element.Type, owner: borrowing Owner
  ) -> borrow(owner) Self
  
  /// Unsafely create a `StorageView` over a span of initialized memory.
  ///
  /// The memory representing `count` instances starting at
  /// `unsafeRawPointer` must be owned by the instance `owner`,
  /// meaning that as long as `owner` is alive, then the memory
  /// will remain valid.
  ///
  /// `unsafeRawPointer` must be correctly aligned for accessing
  /// an element of type `Element`.
  ///
  /// - Parameters:
  ///   - unsafeRawPointer: a pointer to the first initialized element.
  ///   - type: the type to use when interpreting the bytes in memory.
  ///   - count: the number of initialized elements in the view.
  ///   - owner: a binding whose lifetime must exceed that of
  ///            the returned `StorageView`.
  public init<Owner>(
    unsafeRawPointer: UnsafeRawPointer, as type: Element.Type, count: Int, owner: borrowing Owner
  ) -> borrow(owner) Self
}
```

#####  `Collection`-like API:

The following typealiases, properties, functions and subscripts have direct counterparts in the `Collection` protocol hierarchy. Their semantics shall be as described where they counterpart is declared (in `Sequence`, `Collection`, `BidirectionalCollection` or `RandomAccessCollection`). The only difference with their counterpart should be a lifetime dependency annotation, allowing them to return borrowed nonescapable values or borrowed noncopyable values.

```swift
extension StorageView {
  public typealias Index = StorageViewIndex<Element>
  public typealias SubSequence = Self

  public var startIndex: Index { _read }
  public var endIndex: Index { _read }
  public var count: Int { get }

  public func makeIterator() -> copy(self) StorageViewIterator<Element>

  // indexing operations
  public func index(after i: Index) -> Index
  public func index(before i: Index) -> Index
  public func index(_ i: Index, offsetBy distance: Int) -> Index
  public func index(
    _ i: Index, offsetBy distance: Int, limitedBy limit: Index
  ) -> Index?

  public func formIndex(after i: inout Index)
  public func formIndex(before i: inout Index)
  public func formIndex(_ i: inout Index, offsetBy distance: Int)
	public func formIndex(
    _ i: inout Index, offsetBy distance: Int, limitedBy limit: Index
  ) -> Bool

  public func distance(from start: Index, to end: Index) -> Int

  // subscripts
  public subscript(
    _ position: Index
  ) -> copy(self) Element { _read }
  public subscript(
    _ bounds: Range<Index>
  ) -> copy(self) StorageView<Element> { _read }
  public subscript(
    _ bounds: some RangeExpression<Index>
  ) -> copy(self) StorageView<Element> { _read }
  public subscript(
    x: UnboundedRange
  ) -> copy StorageView<Element> { _read }
  
  // utility properties
  public var first: copy(self) Element? { _read }
  public var last: copy(self) Element? { _read }

  // one-sided slicing operations
  public func prefix(upTo: Index) -> copy(self) StorageView<Element>
  public func prefix(through: Index) -> copy(self) StorageView<Element>
  public func prefix(_ maxLength: Int) -> copy(self) StorageView<Element>
  public func dropLast(_ k: Int = 1) -> copy(self) StorageView<Element>
  public func suffix(from: Index) -> copy(self) StorageView<Element>
  public func suffix(_ maxLength: Int) -> copy(self) StorageView<Element>
  public func dropFirst(_ k: Int = 1) -> copy(self) StorageView<Element>
}
```

##### Additions not in the `Collection` family API:

```swift
extension StorageView {
  /// Traps if `position` is not a valid index for this `StorageView`
  public boundsCheckPrecondition(_ position: Index)

  /// Traps if `bounds` is not a valid range of indices for this `StorageView`
  public boundsCheckPrecondition(_ bounds: Range<Index>)

  // Integer-offset subscripts

  /// Accesses the element at the specified offset in the `StorageView`.
  ///
  /// - Parameter offset: The offset of the element to access. `offset`
  ///     must be greater or equal to zero, and less than the `count` property.
  ///
  /// - Complexity: O(1)
  public subscript(offset: Int) -> copy(self) Element { _read }

  /// Accesses the contiguous subrange of elements at the specified
  /// range of offsets in this `StorageView`.
  ///
  /// - Parameter offsets: A range of offsets. The bounds of the range
  ///     must be greater or equal to zero, and less than the `count` property.
  ///
  /// - Complexity: O(1)
  public subscript(offsets: Range<Int>) -> copy(self) StorageView<Element> { _read }

  /// Accesses the contiguous subrange of elements at the specified
  /// range of offsets in this `StorageView`.
  ///
  /// - Parameter offsets: A range of offsets. The bounds of the range
  ///     must be greater or equal to zero, and less than the `count` property.
  ///
  /// - Complexity: O(1)
  public subscript(
    offsets: some RangeExpression<Int>
  ) -> copy(self) StorageView<Element> { _read }

  // Unchecked subscripts

  /// Accesses the element at the specified `position`.
  ///
  /// This subscript does not validate `position`; this is an unsafe operation.
  ///
  /// - Parameter position: The position of the element to access. `position`
  ///     must be a valid index that is not equal to the `endIndex` property.
  ///
  /// - Complexity: O(1)
  public subscript(unchecked position: Index) -> copy(self) Element { _read }

  /// Accesses a contiguous subrange of the elements represented by this `StorageView`
  ///
  /// This subscript does not validate `bounds`; this is an unsafe operation.
  ///
  /// - Parameter bounds: A range of the collection's indices. The bounds of
  ///     the range must be valid indices of the collection.
  ///
  /// - Complexity: O(1)
  public subscript(
    uncheckedBounds bounds: Range<Index>
  ) -> copy(self) StorageView<Element> { _read }

  /// Accesses the contiguous subrange of the elements represented by this `StorageView`,
  /// specified by a range expression.
  ///
  /// This subscript does not validate `bounds`; this is an unsafe operation.
  ///
  /// - Parameter bounds: A range of the collection's indices. The bounds of
  ///     the range must be valid indices of the collection.
  ///
  /// - Complexity: O(1)
  public subscript(
    uncheckedBounds bounds: some RangeExpression<Index>
  ) -> copy(self) StorageView<Element>

  // Unchecked integer-offset subscripts

  /// Accesses the element at the specified offset in the `StorageView`.
  ///
  /// This subscript does not validate `offset`; this is an unsafe operation.
  ///
  /// - Parameter offset: The offset of the element to access. `offset`
  ///     must be greater or equal to zero, and less than the `count` property.
  ///
  /// - Complexity: O(1)
  public subscript(uncheckedOffset offset: Int) -> copy(self) Element { _read }

  /// Accesses the contiguous subrange of elements at the specified
  /// range of offsets in this `StorageView`.
  ///
  /// This subscript does not validate `offsets`; this is an unsafe operation.
  ///
  /// - Parameter offsets: A range of offsets. The bounds of the range
  ///     must be greater or equal to zero, and less than the `count` property.
  ///
  /// - Complexity: O(1)
  public subscript(
    uncheckedOffsets offsets: Range<Int>
  ) -> copy(self) StorageView<Element> { _read }
}
```

`StorageView` gains additional functions when its `Element` is `BitwiseCopyable`:

```swift
extension StorageView where Element: BitwiseCopyable {
  // We may not need to require T: BitwiseCopyable for the aligned load operations

  /// Returns a new instance of the given type, constructed from the raw memory
  /// at the specified byte offset.
  ///
  /// The memory at `offset` bytes from the start of this `StorageView`
  /// must be properly aligned for accessing `T` and initialized to `T`
  /// or another type that is layout compatible with `T`.
  ///
  /// - Parameters:
  ///   - offset: The offset from the start of this `StorageView`, in bytes.
  ///       `offset` must be nonnegative. The default is zero.
  ///   - type: The type of the instance to create.
  /// - Returns: A new instance of type `T`, read from the raw bytes at
  ///   `offset`. The returned instance is memory-managed and unassociated
  ///   with the value in the memory referenced by this `StorageView`.
  public func load<T: BitwiseCopyable>(
    fromByteOffset: Int = 0, as: T.Type
  ) -> T

  /// Returns a new instance of the given type, constructed from the raw memory
  /// at the specified index.
  ///
  /// The memory starting at `index` must be properly aligned for accessing `T`
  /// and initialized to `T` or another type that is layout compatible with `T`.
  ///
  /// - Parameters:
  ///   - index: The index into this `StorageView`
  ///   - type: The type of the instance to create.
  /// - Returns: A new instance of type `T`, read from the raw bytes starting at
  ///   `index`. The returned instance is memory-managed and isn't associated
  ///   with the value in the memory referenced by this `StorageView`.
  public func load<T: BitwiseCopyable>(
    from index: Index, as: T.Type
  ) -> T

  /// Returns a new instance of the given type, constructed from the raw memory
  /// at the specified byte offset.
  ///
  /// The memory at `offset` bytes from the start of this `StorageView`
  /// must be laid out identically to the in-memory representation of `T`.
  ///
  /// - Parameters:
  ///   - offset: The offset from the start of this `StorageView`, in bytes.
  ///       `offset` must be nonnegative. The default is zero.
  ///   - type: The type of the instance to create.
  /// - Returns: A new instance of type `T`, read from the raw bytes at
  ///   `offset`. The returned instance isn't associated
  ///   with the value in the memory referenced by this `StorageView`.
  public func loadUnaligned<T: BitwiseCopyable>(
    fromByteOffset: Int = 0, as: T.Type
  ) -> T

  /// Returns a new instance of the given type, constructed from the raw memory
  /// at the specified index.
  ///
  /// The memory starting at `index` must be laid out identically
  /// to the in-memory representation of `T`.
  ///
  /// - Parameters:
  ///   - index: The index into this `StorageView`
  ///   - type: The type of the instance to create.
  /// - Returns: A new instance of type `T`, read from the raw bytes starting at
  ///   `index`. The returned instance isn't associated
  ///   with the value in the memory referenced by this `StorageView`.
  public func loadUnaligned<T: BitwiseCopyable>(
    from index: Index, as: T.Type
  ) -> T
  
  /// View the memory span represented by this view as a different type
  ///
  /// The memory must be laid out identically to the in-memory representation of `T`.
  ///
  /// - Parameters:
  ///   - type: The type you wish to view the memory as
  /// - Returns: A new `StorageView` over elements of type `T`
  public func view<T: BitwiseCopyable>(as: T.Type) -> borrow(self) StorageView<T>
}
```

##### Interoperability with unsafe code:

We provide two functions for interoperability with C or other legacy pointer-taking functions.

```swift
extension StorageView {
  /// Calls a closure with a pointer to the viewed contiguous storage.
  ///
  /// The buffer pointer passed as an argument to `body` is valid only
  /// during the execution of `withUnsafeBufferPointer(_:)`.
  /// Do not store or return the pointer for later use.
  ///
  /// - Parameter body: A closure with an `UnsafeBufferPointer` parameter
  ///   that points to the viewed contiguous storage. If `body` has
  ///   a return value, that value is also used as the return value
  ///   for the `withUnsafeBufferPointer(_:)` method. The closure's
  ///   parameter is valid only for the duration of its execution.
  /// - Returns: The return value of the `body` closure parameter.
  func withUnsafeBufferPointer<Result>(
    _ body: (_ buffer: UnsafeBufferPointer<Element>) -> Result
  ) -> Result
}

extension StorageView where Element: BitwiseCopyable {
  /// Calls the given closure with a pointer to the underlying bytes of
  /// the viewed contiguous storage.
  ///
  /// The buffer pointer passed as an argument to `body` is valid only
  /// during the execution of `withUnsafeBytes(_:)`.
  /// Do not store or return the pointer for later use.
  ///
  /// - Parameter body: A closure with an `UnsafeRawBufferPointer`
  ///   parameter that points to the viewed contiguous storage.
  ///   If `body` has a return value, that value is also
  ///   used as the return value for the `withUnsafeBytes(_:)` method.
  ///   The closure's parameter is valid only for the duration of
  ///   its execution.
  /// - Returns: The return value of the `body` closure parameter.
  func withUnsafeBytes<Result>(
    _ body: (_ buffer: UnsafeRawBufferPointer) -> Result
  ) -> Result
}
```


## Source compatibility

This proposal is additive and source-compatible with existing code.

## ABI compatibility

This proposal is additive and ABI-compatible with existing code.

## Implications on adoption

The additions described in this proposal require a new version of the standard library and runtime.

## Alternatives considered

##### Make `StorageView` a noncopyable type
Making `StorageView` non-copyable was in the early vision of this type. However, we found that would make `StorageView` a poor match to model borrowing semantics. This realization led to the initial design for non-escapable declarations.

##### A protocol in addition to `ContiguousStorage` for unsafe buffers
This document proposes adding the `ContiguousStorage` protocol to the standard library's `Unsafe{Mutable,Raw}BufferPointer` types. On the surface this seems like whitewashing the unsafety of these types. The lifetime constraint only applies to the binding used to obtain a `StorageView`, and the initialization precondition can only be enforced by documentation. Nothing will prevent unsafe code from deinitializing a portion of the storage while a `StorageView` is alive. There is no safe bridge from `UnsafeBufferPointer` to `ContiguousStorage`. We considered having the unsafe buffer types conforming to a different version of `ContiguousStorage`, which would vend a `StorageView` through a closure-taking API. Unfortunately such a closure would be perfectly capable of capturing the `UnsafeBufferPointer` binding and be as unsafe as can be. For this reason, the `UnsafeBufferPointer` family will conform to `ContiguousStorage`, with safety being enforced in documentation.

##### Use a non-escapable index type
Eventually we want a similar usage pattern for a `MutableStorageView` as we are proposing for `StorageView`. If the index of a `MutableStorageView` were to borrow the view, then it becomes impossible to implement a mutating subscript without also requiring an index to be consumed. This seems untenable.

##### Naming
The ideas in this proposal previously used the name `BufferView`. While the use of the word "buffer" would be consistent with the `UnsafeBufferPointer` type, it is nevertheless not a great name, since "buffer" is usually used in reference to transient storage. On the other hand we already have a nomenclature using the term "Storage" in the `withContiguousStorageIfAvailable()` function, and the term "View" in the API of `String`. A possible alternative name is `StorageSpan`, which mark it as a relative of C++'s `std::span`.

## Future directions

##### Defining `BorrowingIterator` with support in `for` loops
This proposal defines a `StorageViewIterator` that is borrowed and non-escapable. This is not compatible with `for` loops as currently defined. A `BorrowingIterator` protocol for non-escapable and non-copyable containers must be defined, providing a `for` loop syntax where the element is borrowed through each iteration. Ultimately we should arrive at a way to iterate through borrowed elements from a borrowed view:

```swift
borrowing view: StorageView<Element> = ...
for borrowing element in view {
  doSomething(element)
}
```

In the meantime, it is possible to loop through a `StorageView`'s elements by direct indexing:

```swift
func doSomething(_ e: borrowing Element) { ... }
let view: StorageView<Element> = ...
// either:
var i = view.startIndex
while i < view.endIndex {
  doSomething(view[i])
  view.index(after: &i)
}
// ...or:
for o in 0..<view.count {
  doSomething(view[offset: o])
}
```

##### Collection-like protocols for non-copyable and non-escapable types
Non-copyable and non-escapable containers would benefit from a `Collection`-like protocol family to represent a set basic, common operations. This may be `Collection` if we find a way to make it work; it may be something else.

##### Sharing piecewise-contiguous memory
Some types store their internal representation in a piecewise-contiguous manner, such as trees and ropes. Some operations naturally return information in a piecewise-contiguous manner, such as network operations. These could supply results by iterating through a list of contiguous chunks of memory.

##### Delegating mutations of memory with `MutableStorageView<T>`
Some data structures can delegate mutations of their owned memory. In the standard library we have `withMutableBufferPointer()`, for example. A `MutableStorageView<T>` should provide a better, safer alternative.

##### Delegating initialization of memory with `OutputBuffer<T>`
Some data structures can delegate initialization of their initial memory representation, and in some cases the initialization of additional memory. In the standard library we have `Array.init(unsafeUninitializedCapacity:initializingWith:)` and `String.init(unsafeUninitializedCapacity:initializingUTF8With:)`. A safer abstraction for initialization would make such initializers less dangerous, and would allow for a greater variety of them.

##### <a name="Bytes"></a>Resizable, contiguously-stored, untyped collection in the standard library

The example in the [motivation](#motivation) section mentions the `Foundation.Data` type. There has been some discussion of either replacing `Data` or moving it to the standard library. This document proposes neither of those. A major issue is that in the "traditional" form of `Foundation.Data`, namely `NSData` from Objective-C, it was easier to control accidental copies because the semantics of the language did not lead to implicit copying. 

Even if `StorageView` were to replace all uses of a constant `Data` in API, something like `Data` would still be needed, just as `Array<T>` will: resizing mutations (e.g. `RangeReplaceableCollection` conformance.) We may still want to add an untyped-element equivalent of `Array` at a later time.

##### <a name="Conversions"></a>Syntactic Sugar for Automatic Conversions
In the context of a resilient library, a generic entry point in terms of `some ContiguousStorage` may add unwanted overhead. As detailed above, an entry point in an evolution-enabled library requires an inlinable  generic public entry point which forwards to a publicly-accessible function defined in terms of `StorageView`. If `StorageView` does become a widely-used type to interface between libraries, we could simplify these conversions with a bit of compiler help.

We could provide an automatic way to use a `ContiguousStorage`-conforming type with a function that takes a `StorageView` of the appropriate element type:

```swift
func myStrnlen(_ b: StorageView<UInt8>) -> Int {
  guard let i = b.firstIndex(of: 0) else { return b.count }
  return b.distance(from: b.startIndex, to: e)
}
let data = Data((0..<9).reversed()) // Data conforms to ContiguousStorage
let array = Array(data) // Array<UInt8> also conforms to ContiguousStorage
myStrnlen(data)  // 8
myStrnlen(array) // 8
```

This would probably consist of a new type of custom conversion in the language. A type author would provide a way to convert from their type to an owned `StorageView`, and the compiler would insert that conversion where needed. This would enhance readability and reduce boilerplate.

##### Interopability with C++'s `std::span` and with llvm's `-fbounds-safety`
The [`std::span`](https://en.cppreference.com/w/cpp/container/span) class template from the C++ standard library is a similar representation of a contiguous range of memory. LLVM may soon have a [bounds-checking mode](https://discourse.llvm.org/t/70854) for C. These are an opportunity for better, safer interoperation with a type such as `StorageView`.

## Acknowledgments

Joe Groff, John McCall, Tim Kientzle, Michael Ilseman, Karoy Lorentey contributed to this proposal with their clarifying questions and discussions.
