# Noncopyable Standard Library Primitives

* Proposal: [SE-0437](0437-noncopyable-stdlib-primitives.md)
* Authors: [Karoy Lorentey](https://github.com/lorentey)
* Review Manager: [John McCall](https://github.com/rjmccall)
* Status: **Accepted**
* Roadmap: [Improving Swift performance predictability: ARC improvements and ownership control][Roadmap]
* Implementation:
   - The type/function generalizations are (provisionally) already present on main and release/6.0.
   - The proposed API additions are implemented by PRs [#73807](https://github.com/apple/swift/pull/73807) (main) and [#73810](https://github.com/apple/swift/pull/73810) (release/6.0).
* Review: ([pitch](https://forums.swift.org/t/pitch-noncopyable-standard-library-primitives/71566)) ([review](https://forums.swift.org/t/se-0437-generalizing-standard-library-primitives-for-non-copyable-types/72020)) ([acceptance](https://forums.swift.org/t/accepted-se-0437-generalizing-standard-library-primitives-for-non-copyable-types/72275))

[Roadmap]: https://forums.swift.org/t/a-roadmap-for-improving-swift-performance-predictability-arc-improvements-and-ownership-control/54206

Related proposals:

- [SE-0377] `borrowing` and `consuming` parameter ownership modifiers
- [SE-0390] Noncopyable structs and enums
- [SE-0426] BitwiseCopyable
- [SE-0427] Noncopyable generics
- [SE-0429] Partial consumption of noncopyable values
- [SE-0432] Borrowing and consuming pattern matching for noncopyable types

[SE-0377]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0377-parameter-ownership-modifiers.md
[SE-0390]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0390-noncopyable-structs-and-enums.md
[SE-0426]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0426-bitwise-copyable.md
[SE-0427]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0427-noncopyable-generics.md
[SE-0429]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0429-partial-consumption.md
[SE-0432]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0432-noncopyable-switch.md

### Table of Contents

  * [Motivation](#motivation)
    * [Low\-level memory management](#low-level-memory-management)
    * [Generalized optional types](#generalized-optional-types)
  * [Proposed Solution](#proposed-solution)
    * [Generalizing function members](#generalizing-function-members)
    * [Generalizing higher\-order functions](#generalizing-higher-order-functions)
    * [Generalizing caller\-provided return types](#generalizing-caller-provided-return-types)
    * [(Lack of) protocol generalizations](#lack-of-protocol-generalizations)
    * [Unblocking basic construction work](#unblocking-basic-construction-work)
  * [Detailed Design](#detailed-design)
    * [protocol ExpressibleByNilLiteral](#protocol-expressiblebynilliteral)
    * [enum Optional](#enum-optional)
    * [enum Result](#enum-result)
    * [enum MemoryLayout](#enum-memorylayout)
    * [Unsafe Pointer Types](#unsafe-pointer-types)
      * [Member generalizations](#member-generalizations)
      * [Related enhancements in other types](#related-enhancements-in-other-types)
      * [Setting temporary pointers on arbitrary entities](#setting-temporary-pointers-on-arbitrary-entities)
    * [Unsafe Buffer Pointers](#unsafe-buffer-pointers)
      * [Member generalizations](#member-generalizations-1)
      * [Protocol conformances](#protocol-conformances)
      * [Extracting parts of a buffer pointer](#extracting-parts-of-a-buffer-pointer)
      * [Exceptions](#exceptions)
      * [Related enhancements in other types](#related-enhancements-in-other-types-1)
      * [Temporary buffer pointers over arbitrary entities](#temporary-buffer-pointers-over-arbitrary-entities)
    * [Temporary Allocation Facility](#temporary-allocation-facility)
    * [Managed Buffers](#managed-buffers)
    * [Lifetime Management](#lifetime-management)
    * [Swapping and exchanging items](#swapping-and-exchanging-items)
  * [Source compatibility](#source-compatibility)
  * [ABI compatibility](#abi-compatibility)
  * [Alternatives Considered](#alternatives-considered)
    * [Omitting UnsafeBufferPointer](#omitting-unsafebufferpointer)
    * [Alternatives to UnsafeBufferPointer\.extracting()](#alternatives-to-unsafebufferpointerextracting)
  * [Future Work](#future-work)
    * [Non\-escapable Optional and Result](#non-escapable-optional-and-result)
    * [Generalizing higher\-order functions](#generalizing-higher-order-functions-1)
    * [Generalizing Optional\.unsafelyUnwrapped](#generalizing-optionalunsafelyunwrapped)
    * [Generalized managed buffer headers](#generalized-managed-buffer-headers)
    * [Additional raw pointer operations](#additional-raw-pointer-operations)
    * [Protocol generalizations](#protocol-generalizations)
    * [Additional future work](#additional-future-work)
  * [Appendix: struct Hypoarray](#appendix-struct-hypoarray)

## Motivation

[SE-0427] allowed noncopyable types to participate in Swift generics, and introduced the protocol `Copyable` to the Standard Library. However, it stopped short of adapting the Standard Library to support using such constructs. 

The expectation that everything is copyable has been a crucial simplifying assumption throughout all previous API design work in Swift. It allowed and encouraged us to define and use interfaces without having to think too deeply about who is responsible for owning the entities we pass between functions; it let us define convenient container types with implicit copy-on-write value semantics; it has been a constant, familiar, friendly companion of every Swift programmer for almost a decade. Fully rethinking the Standard Library to facilitate working with noncopyable types is not going to happen overnight: it is going to take a series of proposals. This document takes the first step by focusing on an initial set of core changes that will enable building simple generic abstractions using noncopyable types.

To achieve this, we need to tweak some core parts of the Standard Library to start eliminating the assumption of copyability. The changes proposed here only affect a small subset of the Standard Library's API surface; much more work remains to be done. But these changes are intended to be enough to let us start using Swift's new ownership control features in earnest, so that we can use them to solve real problems, but also so that we can gain crucial experience that will inform subsequent Standard Library work.

This proposal concentrates on two particular areas: low-level memory management and generalized optional types. We propose to modify some preexisting generic constructs in the Standard Library to eliminate the assumption of copyability. Such a retroactive generalization is unlikely to be the right approach for every construct (especially not for copy-on-write container types), but it is the appropriate choice for these particular abstractions.

### Low-level memory management

First, we need to extend the existing low-level unsafe pointer operations to allow managing memory that holds noncopyable entities.

- We need to teach `MemoryLayout` how to provide basic information on the memory layout of noncopyable types.
- `UnsafePointer` and `UnsafeMutablePointer` need to support noncopyable pointees. The existing pointer operations need to support working with such instances. This includes heap allocations, pointer conversions and comparisons, operations that bind or rebind raw memory, that initialize/deinitialize memory, etc.
- Similarly, `UnsafeBufferPointer` and `UnsafeMutableBufferPointer` must learn to support noncopyable elements.
- We need the standard low-level memory management facilities to allow working with noncopyable types:
  - Scoped pointer-based access to arbitrary entities (`func withUnsafePointer(to:)`)
  - Unmanaged heap memory allocations (`UnsafeMutablePointer.allocate`, `UnsafeMutableBufferPointer.allocate`)
  - Managed tail-allocated storage allocations (`class ManagedBuffer`, `struct ManagedBufferPointer`).
  - Allocating a temporary buffer (`func withUnsafeTemporaryAllocation`)

Generalizing these constructs for noncopyable types does not fundamentally change their nature -- an `UnsafePointer` to a noncopyable pointee is still a regular, copyable pointer, conforming to much the same protocols as before, and providing many of the same operations. For example, given this simple noncopyable type `Foo`:

```swift
struct Foo: ~Copyable {
  var value: Int
  mutating func increment() { value += 1 }
}
```

We want to be able to dynamically allocate memory for instances of `Foo`, and use the familiar pointer operations we've already learned while working with copyable types:

```swift
let p = UnsafeMutablePointer<Foo>.allocate(capacity: 2)
let q = p + 1
p.initialize(to: Foo(value: 42))
q.initialize(to: Foo(value: 23))
p.pointee.increment()
print(p.pointee.value)        // Prints "43"
print(p[0].value, p[1].value) // Prints "43 23"
print(p < q)                  // Prints "true"
let foo = p.move()
q.deinitialize(count: 1)
p.deallocate()
```

Most of the core pointer operations were already (implicitly) defined in terms of ownership control, and so they readily translate into noncopyable use.

Of course, not all operations can be generalized: for example, `p.initialize(repeating: Foo(7), count: 2)` cannot possibly work, as repeating an item inherently requires making copies of it. That's not a problem though: we can continue to have such operations require a copyable pointee. 


### Generalized optional types

The second area that requires immediate attention is the `Optional` enumeration and its close sibling, `Result`. `Optional` is particularly frequently used in the definition of programming interfaces: it is the standard way a Swift function can take or return a potentially absent value. It is also deeply integrated into the language itself: features such as optional chaining, failable initializers and `try?` statements all rely on it, and we need these features to work even in noncopyable contexts.

It is therefore very much desirable for optionals to start supporting noncopyable payloads, so that Swift functions can continue to use these well-known types in their interface definitions.

Instances of `Optional` and `Result` directly contain the items they wrap. Therefore, an optional wrapping a noncopyable type will necessarily need to become noncopyable itself. This is a far more radical change than generalizing a pointer type: it means these enumerations must turn into conditionally copyable types.

```swift
enum Optional<Wrapped: ~Copyable>: ~Copyable {
  case none
  case some(Wrapped)
}

extension Optional: Copyable where Wrapped: Copyable {}
```

This is no small matter -- every existing use of `Optional` implicitly assumes its copyability, including all its protocol conformances. We need to lift this assumption without breaking source- and (on some platforms) binary compatibility with existing code that relies on it.

Furthermore, compatibility expectations also go in the reverse direction, as `Optional` has been an unavoidable part of Swift since its initial release. On ABI stable platforms, we therefore also expect that code freshly built with this newly generalized `Optional` type will continue to be able to run on older versions of the Swift Standard Library. At minimum, we expect that all code that uses copyable types would be directly back-deployable.

Allowing noncopyable use will necessarily involve defining new operations to help dealing with problems that are specific to noncopyable types. However, when doing that, we need to balance the need to help developers who embrace ownership control with the desire to avoid confusing folks when they continue relying on copyability. (These aren't necessarily different groups of people -- we expect developers will often find it useful to generally stay with copyable abstractions, only reaching for ownership control in specific parts of their code.) Retrofitting noncopyable support on existing types risks muddling up their semantics, hurting our desire for progressive disclosure and potentially overwhelming newcomers.

However, in the particular case of `Optional`, the benefits of making copyability conditional greatly outweigh these drawbacks. Indeed, we don't have much choice but to retrofit `Optional`: we need a common idiom for representing a potentially absent item, shared across all contexts throughout the language.

For example, introducing a separate version of `Optional` that's dedicated to noncopyable use would not be workable. This would prevent generic functions that want to support noncopyable type arguments from taking or returning "classic" optionals, so generic code would quickly standardize on using the new type, while existing interfaces would be stuck with the original -- causing universal confusion.

The case for `Result` is less pressing, as it isn't tied as deeply into the language as `Optional` is. However, `Result` serves a similar purpose as `Optional`: it "merely" expands the nil case to explain the absence with an error value, to implement manual error propagation. It therefore makes sense to propose `Result`'s generalization alongside `Optional`: it involves solving effectively the same problems, and applying the same solutions.

## Proposed Solution

In this proposal, we extend the following generic types in the Standard Library with support for noncopyable type arguments:

- `enum Optional<Wrapped: ~Copyable>`
- `enum Result<Success: ~Copyable, Failure: Error>`
- `struct MemoryLayout<T: ~Copyable>`
- `struct UnsafePointer<Pointee: ~Copyable>`
- `struct UnsafeMutablePointer<Pointee: ~Copyable>`
- `struct UnsafeBufferPointer<Element: ~Copyable>`
- `struct UnsafeMutableBufferPointer<Element: ~Copyable>`
- `class ManagedBuffer<Header, Element: ~Copyable>`
- `struct ManagedBufferPointer<Header, Element: ~Copyable>`

`Optional` and `Result` become conditionally copyable, inheriting their copyability from their type arguments. All other types above remain unconditionally copyable, independent of the copyability of their type argument.

We also update a single standard protocol to allow noncopyable conforming types:

- `protocol ExpressibleByNilLiteral: ~Copyable`

Additionally, we generalize the following top-level function families:

- `func swap(_:_:)`
- `func withExtendedLifetime(_:_:)`
- `func withUnsafeTemporaryAllocation(byteCount:alignment:_:)`
- `func withUnsafeTemporaryAllocation(of:capacity:_:)`
- `func withUnsafePointer(to:_:)`
- `func withUnsafeMutablePointer(to:_:)`
- `func withUnsafeBytes(of:_:)`
- `func withUnsafeMutableBytes(of:_:)`

We also generalize some low-level generic functions elsewhere in the stdlib that take or return the types above -- such as pointer conversion or rebinding operations. (See below for details.)

In several example snippets, we'll be using the following (nonexistent) type to illustrate the use of noncopyable types:

```swift
struct File: ~Copyable, Sendable {
  init(opening path: FilePath) throws {...}
  mutating func readByte() throws -> UInt8 {...}
  consuming func close() throws {...}
  deinit {...}
}
```

This is for illustrative purposes only -- we're not proposing to add an I/O facility to the stdlib in this document, and we do not expect a hypothetical future I/O feature would actually use this exact API surface.

The rest of this section presents the principles this proposal follows in generalizing the constructs above. For a detailed list of changes, please see the [Detailed Design](#detailed-design) below.

### Generalizing function members

In past Swift versions, the difference between an operation consuming and borrowing its input argument was merely a subtle implementation detail: it was relevant for some performance optimization work, but generally there was little reason to learn or care about it. With noncopyable inputs, the distinction between consuming vs borrowing an argument rises to upmost importance -- it is one of the first things we need to learn when we write code that needs to use noncopyable types, whether we want to design our own APIs or to understand and use APIs provided by others.

Retrofitting existing generic APIs for noncopyable use involves determining what ownership semantics to apply on their potentially noncopyable input arguments (including the special `self` argument). If the result of a function is noncopyable, then that inherently means the function is passing ownership of its output to its caller. This means that the function cannot keep hold of that value.

When we cannot assume copyability, we need to carefully distinguish between consuming and borrowing use: functions need to declare this choice up front, for every parameter that isn't guaranteed to be copyable.

For example, take the existing pointer operation that initializes the addressed location:

```swift
extension UnsafeMutablePointer {
  func initialize(to value: Pointee) { ... }
}
```

Semantically, this used to take a _copy_ of its input value, as that's how copyable values get passed to functions. (In this case, the implementation of the function is exposed to clients---it is marked `@inlinable`---so the copying can often be optimized away when the call happens to be the last use of the original instance. However, this depends on optimization heuristics; there is no strict guarantee that it will always happen.)

With a noncopyable pointee, this will no longer work! The calling convention no longer makes sense.

To support potentially non-copyable `Pointee` types, this function must now explicitly specify ownership semantics for its input argument. In this case, the operation wants to _take ownership_ of its input, because it needs to move it into its new place at the addressed location in memory. Therefore the function must explicitly consume its input argument:

```swift
extension UnsafeMutablePointer where Pointee: ~Copyable {
  func initialize(to value: consuming Pointee) { ... }
}
```

This new function remains source-compatible with the classic definition, except now it can work with noncopyable types. If a copyable value is passed as a `consuming` argument, Swift can pass an implicit copy of it as needed, based on whether or not the caller will need to continue using the original. Importantly, the explicit `consuming` keyword now _guarantees_ that the code will avoid making unnecessary copies when possible, even if `Pointee` happens to be copyable: we no longer rely on optimizer heuristics to avoid unnecessary copying overhead. 

By distinguishing between consuming and borrowing use, we gain more precise control over how our code behaves. The downside, of course, is that we pay for this control by having to think about it -- not only while defining these operations, but also while using them. To call this function with a noncopyable entity, we need to provide it with an item we own, and we need to be willing to let the function consume it. For example, we cannot call it on an instance that we're only borrowing from someone else.

Preserving source compatibility is great, but unfortunately entirely replacing the old entry point with this new definition would be an ABI breaking change: the new function follows a new calling convention and it is exposed under a different linkage name. Existing binaries will keep linking with the original entry point, and we need to ensure they continue working. To allow this, on ABI stable platforms we continue to expose the old definition as an obsoleted `@usableFromInline internal` function. To allow newly built code to run on previously shipped Standard Library versions, the replacement needs to be defined in a back-deployable manner, such as by using the [`@backDeploy` attribute][SE-0376]. 

[SE-0376]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0376-function-back-deployment.md

```swift
// Non-normative illustration of an implementation technique.
extension UnsafeMutablePointer where Pointee: ~Copyable {
  @backDeployed(before: ...)
  public func initialize(to value: consuming Pointee) { ... }
}
extension UnsafeMutablePointer /*where Pointee: Copyable*/ {
  @available(swift, obsoleted: 1)
  @usableFromInline
  internal func initialize(to value: Pointee) { ... }
}
```

This way, existing binaries can continue to link with the original entry point, while newly built code will smoothly transition to the new definition.)

The new function needs to be marked back-deployable, as it is replacing the original copyable version, and as such it needs to have matching availability. The function's implementation is directly embedded into binaries, so this means that in this particular case, newly introduced support for noncopyable use is also expected to "magically" work on older releases. (This will not necessarily extend to all noncopyable generalizations, as not every operation can retroactively learn to deal with noncopyable entities. In particular, older runtimes aren't expected to understand how to perform dynamic operations on noncopyable types (such as looking up metatype instances, performing conformance checks, dealing with existentials, downcasting or reflection); any operation that requires such features is not expected to deploy back without limits.)

Declaring a function's input parameters `consuming` or `borrowing` can generally be done without tweaking the implementation, as long as the function does not happen to implicitly copy such arguments. If the implementation does rely on implicit copying, then it needs to be corrected to avoid doing that.

### Generalizing higher-order functions

For most operations, introducing support for noncopyable use is simply a matter of deciding what ownership rules they should follow and then adding the corresponding annotations. Sometimes the choice between consuming/borrowing use isn't obvious though -- the function may make sense in both flavors.

Such is often the case with operations that take function arguments. For, example, take `Optional.map`:

```swift
extension Optional {
  func map<E: Error, U>(
    _ transform: (Wrapped) throws(E) -> U
  ) throws(E) -> U?
}
```

Is this function supposed to consume or borrow the optional? Looking through existing (copyable) use cases, the answer seems to be both! 

In many cases, `map` is used to _transform_ the wrapped value into some other type, logically consuming it in the process. In some others, `map` is used to _project_ the wrapped value into some other entity, for example by copying a component or some computed property of it.

The existing `map` name cannot be used to name both flavors, as consuming/borrowing annotations are not involved in overload resolution, so trying to do that would make the `map` name ambiguous. Of course, we could use the above distinction between consuming transformations and borrowing projections to replace `map` with two functions named `transform` and `project`. However, this terminology would be way too subtle, and it would not apply to similar cases elsewhere, such as `map`'s close sibling, `Optional.flatMap`.

To resolve the ambiguity, we'll probably need to introduce a naming convention, such as to use `consuming` and/or `borrowing` as naming prefixes (as in `consumingMap` or `borrowingFlatMap`), or to invent `consuming` or `borrowing` views and move these operations there (as in `consuming.map` or `borrowing.flatMap`). Some of these choices depend on language features (non-escapable types, stored borrows, read accessors, consuming getters) that do not exist yet. Accordingly, we defer introducing consuming/borrowing higher-order functions until we can gain enough practical experience to make an informed decision.

Functions like `Optional.map` will therefore keep requiring copyability for now. However, we can and should immediately generalize such functions in a different direction.

### Generalizing caller-provided return types

Many of the higher-order functions in the Standard Library are designed to return whatever value is returned by their function arguments. This includes the `Optional.map` function we saw above:

```swift
extension Optional {
  func map<E: Error, U>(
    _ transform: (Wrapped) throws(E) -> U
  ) throws(E) -> U?
}
```

The generic return type `U` is implicitly required to be copyable here, which prevents the transformation from returning a noncopyable type:

```swift
let name: FilePath? = ...
let file: File? = try name.map { try File(opening: $0) }
// error: noncopyable type 'File' cannot be substituted for copyable generic parameter 'U' in 'map'
```

Code that uses noncopyable types would hit this limitation surprisingly frequently, and working around it requires annoying and error-prone acrobatics, such as using an inout capture of an optional:

```swift
let name: FilePath? = ...
var file: File? = nil
try name.map { file = try File(opening: $0) } // OK
```

To avoid forcing developers to use such workarounds, we systematically generalize such closure-taking APIs to allow noncopyable result types:

```swift
extension Optional {
  func map<E: Error, U: ~Copyable>(
    _ transform: (Wrapped) throws(E) -> U
  ) throws(E) -> U?
}

let name: FilePath? = ...
let file: File? = try name.map { try File(opening: $0) } // OK
```

This is particularly important for interfaces like `ManagedBuffer.withUnsafeMutablePointers`, which are commonly used to access some container type's backing storage. A typical example is inside the implementation of a removal operation where the closure wants to return the item it removed from the container.

(This generalization of outputs can be safely shipped separate from the consuming/borrowing generalizations of the input side. Doing these generalizations in two separate phases is not expected to cause any issues: it does not prevent getting us to whatever final APIs we want, and it does not introduce any unique compatibility problems that wouldn't also occur if we did both generalizations at the same time.)

### (Lack of) protocol generalizations

It would be very much desirable to generalize some of the existing Standard Library protocols for noncopyable use. However, each protocol needs careful consideration that is best deferred to subsequent proposals; therefore, in this particular document, we limit ourselves to generalizing just a single protocol, `ExpressibleByNilLiteral`.

Generalizing `ExpressibleByNilLiteral` allows our newly noncopyable `Optional` to unconditionally conform to it, so that we can continue to use the `nil` keyword to refer to an empty optional instance, even if it happens to be wrapping a noncopyable type.

```swift
var document: File? = nil // OK
```

All other public protocols in the Standard Library continue to require their conforming types as well as all their associated types to be copyable for now. This includes (but isn't limited to) such basic protocols as `Equatable`, `CustomStringConvertible`, `ExpressibleByArrayLiteral`, `Codable`, and `Sequence`. Some of these are directly generalizable, but most will require considerable design work, which we defer to future proposals.

Therefore, all other conformances on `Optional` and `Result` will remain conditional on `Wrapped`'s copyability until future proposals. For example, while we'll be able to use the `== nil` form to check if an optional wraps no entity, the full `==` function is leaning on `Equatable` and thus it will only work for copyable types for now:

```swift
let file: File? = try File(opening: "noncopyable-stdlib-primitives.md")
print(c == nil) // OK, prints "false"
let d: File? = nil
print(c == d)   // error: operator function '==' requires that 'File' conform to 'Equatable'
```

On the other hand, unsafe pointer types remain unconditionally copyable, so some of their conformances can continue to remain unconditional. `UnsafePointer` and `UnsafeMutablePointer` can remain `Equatable`, `Comparable`, `Strideable` etc. even if their pointee happens to be noncopyable.

```swift
let p: UnsafePointer<File> = ...
var q: UnsafePointer<File> = ...
print(p == q) // OK
q += 1 // OK
```

Unfortunately, the same isn't true for `UnsafeBufferPointer`, whose conformances to the `Collection` protocol hierarchy do not translate at all -- all our current container protocols require a copyable `Element`.

Unsafe buffer pointers gain much of their core functionality from the `Collection` protocol: for instance, even the idea of accessing an item by subscripting with an integer index comes from that protocol. However, all buffer pointers need a way identify positions in themselves, and so we must nevertheless generalize the `Index` type and its associated index navigation methods and the crucial indexing subscript operation.

```swift
extension UnsafeBufferPointer where Element: ~Copyable {
  typealias Index = Int
  var startIndex: Int { get }
  var endIndex: Int { get }
  var isEmpty: Bool { get }
  var count: Int { get }
  func index(after i: Int) -> Int
  func index(before i: Int) -> Int
  ...
  subscript(i: Int) -> Element // (unstable accessor not shown)
}
```

Note that the generalized indexing subscript cannot provide a regular getter, as that would work by returning a copy of the item -- so the Standard Library currently has to resort to an unstable/unsafe language feature to provide direct borrowing access. (This isn't new, as we previously relied on this scheme to optimize performance; but its use now becomes unavoidable. Defining a stable language feature to implement such accessors is expected to be a topic of a future proposal.)

While we can generalize the basic container primitives, sadly we need to leave the actual sequence & collection conformances conditional on copyability. Some of the protocol requirements also need to stick with copyability:

```swift
extension UnsafeBufferPointer: Sequence /* where Element: Copyable */ {
  struct Iterator: IteratorProtocol {...}
  func makeIterator() -> Iterator {...}
}

extension UnsafeBufferPointer: RandomAccessCollection /* where Element: Copyable */ {
  typealias Indices = Range<Int>
  typealias SubSequence = Slice<Unsafe${Mutable}BufferPointer<Element>>
  var indices: Indices { get }
  subscript(bounds: Range<Int>) -> Slice<Self>
}
```

For-in loops currently require a `Sequence` conformance, which means it will not yet be possible to iterate over the contents of an unsafe buffer pointer of noncopyable elements using a direct for-in loop. For now, we will need to write manual loops such as this one:

```swift
let buffer: UnsafeBufferPointer<Atomic<Int>> = ...
for i in buffer.startIndex ..< buffer.endIndex {
  buffer[i].add(1, ordering: .sequentiallyConsistent)
}
```

This will have to bide us over until we invent new protocols for noncopyable containers. (Of course, that is expected to be the subject of subsequent work; however, we'll first need to introduce nonescapable types and use them to build some fundamental Standard Library constructs.)

Another item of particular note is the loss of slicing subscript for noncopyable buffer pointers. The original slicing subscript returns a `Slice`, which requires a `Base` that conforms to `Collection`. Therefore, we can only provide the slicing subscript if `Element` happens to be copyable. Slicing buffer pointers is a very common operation, quite crucial for their usability. To make up for this, we propose to add an operation that returns a new standalone buffer over the supplied range of elements:

```swift
extension UnsafeBufferPointer where Element: ~Copyable {
  func extracting(_ bounds: Range<Int>) -> Self
  func extracting(_ bounds: some RangeExpression<Int>) -> Self
}
```

The returned buffer does not share indices with the original; its indices start at zero. 

For buffer pointers with noncopyable elements, this operation will be the only (easy) way to split a buffer into small parts:

```swift
import Synchronization

// A bank of atomic integers
let bank = UnsafeMutableBufferPointer<Atomic<Int>>.allocate(capacity: 4)
for i in 0 ..< 4 {
  bank.initializeElement(at: i, to: Atomic(i))
}

let part = bank.extracting(2 ..< 4)
print(part[0].load(ordering: .sequentiallyConsistent)) // Prints "2"
print(part[1].load(ordering: .sequentiallyConsistent)) // Prints "3"

bank.deinitialize().deallocate()
```

For copyable elements, the `extracting` operation is not crucial, but it is still useful: it is effectively a shorthand for slicing the buffer and immediately passing the returned slice to the `UnsafeBufferPointer.init(rebasing:)` initializer. This too is a common idiom, so it makes sense to provide a universally available shorter spelling for it.

### Unblocking basic construction work

The changes proposed here are enough to start constructing noncopyable containers, such as this illustrative noncopyable array variant, built entirely around an unsafe buffer pointer:

```swift
struct Hypoarray<Element: ~Copyable>: ~Copyable {
  private var _storage: UnsafeMutableBufferPointer<Element>
  private var _count: Int

  init() {
    _storage = .init(start: nil, count: 0)
    _count = 0
  }
  
  init(_ element: consuming Element) {
    _storage = .allocate(capacity: 1)
    _storage.initializeElement(at: 0, to: element)
    _count = 1
  }
  
  deinit {
    _storage.extracting(0 ..< count).deinitialize()
    _storage.deallocate()
  }
}
```

See the appendix for a full(er) definition of this sample type, including some of the fundamental array operations.

Note that this type is presented here only to illustrate the use of the newly enhanced Standard Library; we do not propose to add such a type to the library as part of this proposal. On the other hand, building up a suite of basic noncopyable data structure implementations is naturally expected to be the subject of subsequent future work.

## Detailed Design

### `protocol ExpressibleByNilLiteral`

In this proposal, we limit ourselves to generalizing just one standard protocol, `ExpressibleByNilLiteral`.  We lift the requirement that conforming types must be copyable:

```swift
protocol ExpressibleByNilLiteral: ~Copyable {
  init(nilLiteral: ())
}
```

This lets us continue to support the use of `nil` with noncopyable `Optional` types.

(We do need to eventually generalize additional protocols, of course; as we mentioned above, such work is deferred to future proposals.)

### `enum Optional`

The `Optional` enum needs to be generalized to allow wrapping non-copyable types. This requires `Optional` to itself become conditionally copyable.

```swift
@frozen
enum Optional<Wrapped: ~Copyable>: ~Copyable {
  case none
  case some(Wrapped)
}

extension Optional: Copyable /* where Wrapped: Copyable */ {}
extension Optional: Sendable where Wrapped: ~Copyable & Sendable { }

extension Optional: ExpressibleByNilLiteral where Wrapped: ~Copyable {
  init(nilLiteral: ())
}

extension Optional where Wrapped: ~Copyable {
  init(_ some: consuming Wrapped)
}
```

`Optional`'s `map` and `flatMap` members can be generalized to relax the copyability requirement on their return type:

```swift
extension Optional {
  func map<E: Error, U: ~Copyable>(
    _ transform: (Wrapped) throws(E) -> U
  ) throws(E) -> U?

  func flatMap<E: Error, U: ~Copyable>(
    _ transform: (Wrapped) throws(E) -> U?
  ) throws(E) -> U?
}
```

However, these members cannot work on noncopyable optionals, as we have to distinguish between consuming and borrowing variants in that context. Choosing which of these two variants to generalize the existing names, and precisely what notation to use for the remaining variant is deferred to a future proposal.

The current `unsafelyUnwrapped` property cannot currently be generalized for noncopyable types, so it is also kept restricted to the copyable case.

[As foreshadowed in SE-0390](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0390-noncopyable-structs-and-enums.md#noncopyable-optional), it is commonly useful to use noncopyable optionals to allow partial consumption of stored properties in [contexts where that isn't normally allowed][SE-0429]. To support such use, we introduce a brand new mutating member `Optional.take()` that resets `self` to nil, returning its original value:

```swift
extension Optional where Wrapped: ~Copyable {
  mutating func take() -> Self {
    let result = consume self
    self = nil
    return result
  }
}
```

Having a named operation for this common need establishes it as a universal idiom.

The standard nil-coalescing `??` operator is updated to explicitly consume its first operand:

```
func ?? <T: ~Copyable>(
  optional: consuming T?,
  defaultValue: @autoclosure () throws -> T
) rethrows -> T 

func ?? <T: ~Copyable>(
  optional: consuming T?,
  defaultValue: @autoclosure () throws -> T?
) rethrows -> T?
```

This matches the behavior of the second argument, where ownership of the default value is passed to the `??` implementation.

In this initial phase, `Equatable` continues to require conforming types to be copyable, so optionals containing noncopyable types cannot yet be compared for equality. However, `Optional` also provides special support for `== nil` and `!= nil` comparisons whether or not its wrapped type is Equatable. We do generalize this support to allow noncopyable wrapped types:

```swift
extension Optional where Wrapped: ~Copyable {
  static func ==(
    lhs: borrowing Wrapped?,
    rhs: _OptionalNilComparisonType
  ) -> Bool
  static func !=(
    lhs: borrowing Wrapped?,
    rhs: _OptionalNilComparisonType
  ) -> Bool
  static func ==(
    lhs: _OptionalNilComparisonType,
    rhs: borrowing Wrapped?
  ) -> Bool
  static func !=(
    lhs: _OptionalNilComparisonType,
    rhs: borrowing Wrapped?
  ) -> Bool
}
```

We are also generalizing the standard `~=` operators to support pattern matching `nil` on noncopyable Optionals.

```swift
extension Optional where Wrapped: ~Copyable {
  static func ~=(
    lhs: _OptionalNilComparisonType,
    rhs: borrowing Wrapped?
  ) -> Bool
}
```

(The implementations above currently rely on an unstable `_OptionalNilComparisonType` type to represent a type-agnostic `nil` value. This type and this particular way of implementing `== nil` is an internal implementation detail of the Standard Library that remains subject to change in future versions. These signatures are listed to illustrate the changes we're making; they aren't intended to stabilize this particular implementation.)

### `enum Result`

The standard `Result` type similarly needs to be generalized to allow noncopyable `Success` values, itself becoming conditionally noncopyable.

```swift
@frozen
enum Result<Success: ~Copyable, Failure: Error>: ~Copyable {
  case success(Success)
  case failure(Failure)
}

extension Result: Copyable where Success: Copyable {}
extension Result: Sendable where Success: Sendable & ~Copyable {}
```

Like with `Optional`, some of `Result`'s existing members can be directly generalized not to require a copyable success type.

```swift
extension Result where Success: ~Copyable {
  init(catching body: () throws(Failure) -> Success)
  
  consuming func get() throws(Failure) -> Success

  consuming func mapError<NewFailure>(
    _ transform: (Failure) -> NewFailure
  ) -> Result<Success, NewFailure

  consuming func flatMapError<NewFailure>(
    _ transform: (Failure) -> Result<Success, NewFailure>
  ) -> Result<Success, NewFailure>
}  
```

The `mapError` members need to potentially return the `Success` value they originally stored in `self`, so they need to become consuming functions -- we cannot provide any borrowing variants.

Like we saw with `Optional`, unfortunately this does not apply to members that transform the success value. We can still generalize the type of the result, but not the input:

```swift
extension Result {
  func map<NewSuccess: ~Copyable>(
    _ transform: (Success) -> NewSuccess
  ) -> Result<NewSuccess, Failure>
  
  func flatMap<NewSuccess: ~Copyable>(
    _ transform: (Success) -> Result<NewSuccess, Failure>
  ) -> Result<NewSuccess, Failure>
}
```

We defer generalizing the "input side" into borrowing/consuming map/flatMap variants until a future proposal; until then, `map` and `flatMap` continue to require a copyable `Success`.

### `enum MemoryLayout`

We extend `MemoryLayout` to allow querying the layout properties of noncopyable types:

```swift
enum MemoryLayout<T: ~Copyable>: Copyable {}

extension MemoryLayout where T: ~Copyable {
  static var size: Int { get }
  static var stride: Int { get }
  static var alignment: Int { get }
  
  static func size(ofValue value: borrowing T) -> Int
  static func stride(ofValue value: borrowing T) -> Int
  static func alignment(ofValue value: borrowing T) -> Int 
}
```

Note that the current `offset(of:)` member continues to require `T` to be copyable, as key paths do not (currently) support noncopyable targets.

### Unsafe Pointer Types

We have two typed unsafe pointer types, `UnsafePointer` and `UnsafeMutablePointer`. To allow building noncopyable constructs, these types need to start supporting noncopyable pointee types.

```swift
struct UnsafePointer<Pointee: ~Copyable>: Copyable
struct UnsafeMutablePointer<Pointee: ~Copyable>: Copyable
```

Pointers to noncopyable types still need to work like pointers -- in particular, the pointers themselves must always remain copyable. Unlike with `Optional` and `Result`, pointers can therefore continue to unconditionally conform to the `Equatable`, `Hashable`, `Comparable`, `Strideable` and `CVarArg` protocols, as well as the new `AtomicRepresentable`, `AtomicOptionalRepresentable` protocols, regardless of the copyability of their pointee type.

```swift
extension Unsafe[Mutable]Pointer: Equatable where Pointee: ~Copyable {...}
extension Unsafe[Mutable]Pointer: Hashable where Pointee: ~Copyable {...}
extension Unsafe[Mutable]Pointer: Comparable where Pointee: ~Copyable {...}
extension Unsafe[Mutable]Pointer: Strideable where Pointee: ~Copyable {...}
extension Unsafe[Mutable]Pointer: CustomDebugStringConvertible where Pointee: ~Copyable {...}
extension Unsafe[Mutable]Pointer: CustomReflectable where Pointee: ~Copyable {...}

// module Synchronization:
extension Unsafe[Mutable]Pointer: AtomicRepresentable where Pointee: ~Copyable {...}
extension Unsafe[Mutable]Pointer: AtomicOptionalRepresentable where Pointee: ~Copyable {...}
```

#### Member generalizations

Most existing members of unsafe pointers adapt directly into the noncopyable world, with some notable exceptions that inherently require copyability:

- Some operations rely on duplicating or copying pointee values:
   - `func initialize(repeating: Pointee, count: Int)`
   - `func update(from source: UnsafePointer<Pointee>, count: Int)`
   - `func initialize(from source: UnsafePointer<Pointee>, count: Int)`
- Others depend on key paths that have not been generalized for noncopyable types yet:
   - `func pointer<Property>(to: KeyPath<Pointee, Property>) -> Unsafe[Mutable]Pointer<Property>?`
   - `func pointer<Property>(to: WritableKeyPath<Pointee, Property>) -> UnsafeMutablePointer<Property>?`

These members will continue to require that `Pointee` be copyable. 

All other standard pointer operations lift the copyability requirement:

- In Swift 5.x, the `pointee` property and the standard offsetting subscript have already been defined with special accessors that provide in-place borrowing or mutating access to instances addressed by the pointer. These translate directly for noncopyable use.

   ```swift
   extension Unsafe[Mutable]Pointer where Pointee: ~Copyable {
     var pointee: Pointee         // (unstable accessors not shown)
     subscript(i: Int) -> Pointee // (unstable accessors not shown)
   }
   ```

- Of special note is the `withMemoryRebound` function, which needs to generalized not just for noncopyable pointees, but also for potentially noncopyable target and result types:

   ```swift
   extension Unsafe[Mutable]Pointer where Pointee: ~Copyable {
     func withMemoryRebound<T: ~Copyable, E: Error, Result: ~Copyable>(
       to type: T.Type,
       capacity count: Int,
       _ body: (_ pointer: Unsafe[Mutable]Pointer<T>) throws(E) -> Result
     ) throws(E) -> Result
   }
   ```

- In previous Swift releases, the existing `UnsafeMutablePointer.initialize(to:)` member used to be defined to (effectively) borrow, rather than consume, its argument. This used to be a minor performance wrinkle, but with noncopyable pointees, it has now become a correctness problem. Therefore, in its newly generalized form, `initialize(to:)` now consumes its argument:

    ```swift
    extension UnsafeMutablePointer where Pointee: ~Copyable {
      func initialize(to value: consuming Pointee)
    }
    ```

    This change does not affect source compatibility with existing copyable call sites, and its ABI impact is mitigated by continuing to expose the original borrowing entry point as an obsolete `@usableFromInline` function.

- All other pointer members generalize in a straightforward way:

    ```swift
    extension Unsafe[Mutable]Pointer where Pointee: ~Copyable {
      init(_ other: Self)
      init?(_ other: Self?)
      init(_ from: OpaquePointer)
      init?(_ from: OpaquePointer?)
      init?(bitPattern: Int)
      init?(bitPattern: UInt)
  
      func deallocate()
    }

    extension UnsafeMutablePointer where Pointee: ~Copyable {
      init(mutating other: UnsafePointer<Pointee>)
      init?(mutating other: UnsafePointer<Pointee>?)
      init(_ other: UnsafeMutablePointer<Pointee>)
      init?(_ other: UnsafeMutablePointer<Pointee>?)
  
      static func allocate(capacity count: Int) -> UnsafeMutablePointer<Pointee>
  
      func move() -> Pointee

      func moveInitialize(from source: UnsafeMutablePointer, count: Int)
      func moveUpdate(from source: UnsafeMutablePointer, count: Int)
  
      func deinitialize(count: Int) -> UnsafeMutableRawPointer
    }
    ```

#### Related enhancements in other types

To keep the Standard Library's family of pointer types coherent, we also need to ensure that pointers to noncopyable types continue to interact well with other pointer types in the language, including `UnsafeRawPointer` and `OpaquePointer`:

- The Swift Standard Library provides heterogeneous pointer comparison operators (`==`, `!=`, `<`, `<=`, `>`, `>=`) that allow comparing any two pointer values, no matter their type. We generalize these to extend their support to comparing pointers with noncopyable pointees.

- Similarly, the `init(bitPattern:)` initializers on `Int` and `UInt` can work with any pointer type. These initializers must now also extend support to the newly generalized pointer types.

   (Note: We do not list interface updates for the last two enhancements, as they are currently implemented by generalizing the source-unstable `_Pointer` protocol, an implementation detail of the Standard Library.)

- We need to generalize all generic conversion operations on raw and opaque pointers:

   ```swift
   extension OpaquePointer {
     init<T: ~Copyable>(_ from: Unsafe[Mutable]Pointer<T>)
     init?<T: ~Copyable>(_ from: Unsafe[Mutable]Pointer<T>?)
   }

   extension Unsafe[Mutable]RawPointer {
     init<T: ~Copyable>(_ other: Unsafe[Mutable]Pointer<T>)
     init?<T: ~Copyable>(_ other: Unsafe[Mutable]Pointer<T>?)
   }
   ```

- Operations that bind and initialize raw memory to arbitrary types also need to relax their copyability requirements:

   ```swift
   extension Unsafe[Mutable]RawPointer {
     func bindMemory<T: ~Copyable>(
       to type: T.Type, capacity count: Int
     ) -> Unsafe[Mutable]Pointer<T>
  
     func withMemoryRebound<T: ~Copyable, E: Error, Result: ~Copyable>(
       to type: T.Type,
       capacity count: Int,
       _ body: (_ pointer: Unsafe[Mutable]Pointer<T>) throws(E) -> Result
     ) throws(E) -> Result
  
     func assumingMemoryBound<T: ~Copyable>(
       to: T.Type
     ) -> Unsafe[Mutable]Pointer<T>

     func moveInitializeMemory<T: ~Copyable>(
       as type: T.Type, from source: UnsafeMutablePointer<T>, count: Int
     ) -> UnsafeMutablePointer<T>
   }
   ```

- As well as raw pointer operations that deal with a generic type's memory layout:

   ```swift
   extension Unsafe[Mutable]RawPointer {
     func alignedUp<T: ~Copyable>(for type: T.Type) -> Self
     func alignedDown<T: ~Copyable>(for type: T.Type) -> Self 
   }
   ```
   
#### Setting temporary pointers on arbitrary entities

The standard `withUnsafe[Mutable]Pointer` top-level functions allow temporary pointer access to any inout value. These now need to be extended to support noncopyable types:

```swift
func withUnsafeMutablePointer<T: ~Copyable, E: Error, Result: ~Copyable>(
  to value: inout T,
  _ body: (UnsafeMutablePointer<T>) throws(E) -> Result
) throws(E) -> Result

func withUnsafePointer<T: ~Copyable, E: Error, Result: ~Copyable>(
  to value: inout T,
  _ body: (UnsafePointer<T>) throws(E) -> Result
) throws(E) -> Result
```

Beware that the pointer argument to `body` continues to be valid only during the execution of the function, even if `T` happens to be noncopyable. There is also no guarantee that the address will remain unchanged across repeated calls to `withUnsafe[Mutable]Pointer`.

This also emphatically applies to the third `withUnsafePointer` variant that provides a temporary pointer to a borrowed instance. This one also gets generalized:

```swift
func withUnsafePointer<T: ~Copyable, E: Error, Result: ~Copyable>(
  to value: borrowing T,
  _ body: (UnsafePointer<T>) throws(E) -> Result
) throws(E) -> Result
```

Borrows aren't exclusive, so it is possible to reentrantly call this function multiple times on the same noncopyable instance. When we do so, it may sometimes appear that the same (ostensibly noncopyable) entity is concurrently occupying multiple different locations in memory:

```swift
struct Ghost: ~Copyable {
  var value: Int
}

let ghost = Ghost(value: 42)
withUnsafePointer(to: ghost) { p1 in
  withUnsafePointer(to: ghost) { p2 in
    print(p1 == p2) // Can print false!
  }
}
```

Do not adjust your set -- this curiosity is inherent in the call-by-value calling convention that Swift normally uses for passing borrowed instances. (Semantically, there is still only a single extant copy, although it can sometimes be smeared over multiple locations.)

### Unsafe Buffer Pointers

Like pointers, typed buffer pointers need to start supporting noncopyable elements, without themselves becoming noncopyable.

```swift
struct UnsafeBufferPointer<Element: ~Copyable>: Copyable {}
struct UnsafeMutableBufferPointer<Element: ~Copyable>: Copyable {}
```

#### Member generalizations

Most existing buffer pointer operations directly translate to the noncopyable world:

- Initializers adapt with no changes:

    ```swift
    extension UnsafeBufferPointer where Element: ~Copyable {
      init(start: UnsafePointer<Element>?, count: Int)
      init(_ other: UnsafeMutableBufferPointer<Element>)
    }
    extension UnsafeMutableBufferPointer where Element: ~Copyable {
      init(start: UnsafeMutablePointer<Element>?, count: Int)
      init(mutating other: UnsafeBufferPointer<Element>)
    }
    ```
    
- So do the properties for accessing the components of a buffer pointer:

    ```swift
    extension Unsafe[Mutable]BufferPointer where Element: ~Copyable {
      var baseAddress: Unsafe[Mutable]Pointer<Element>? { get }
      var count: Int { get }
    }
    ```

- As well as mutable/immutable deallocation:

    ```swift
    extension Unsafe[Mutable]BufferPointer where Element: ~Copyable {
      func deallocate()
    }
    ```

- And most mutating operations:

    ```swift
    extension UnsafeMutableBufferPointer where Element: ~Copyable {
      static func allocate(capacity count: Int) -> UnsafeMutableBufferPointer<Element>

      func moveInitialize(fromContentsOf source: Self) -> Index

      func moveUpdate(fromContentsOf source: Self) -> Index

      func deinitialize() -> UnsafeMutableRawBufferPointer
      func deinitializeElement(at index: Index)
      func moveElement(from index: Index) -> Element
    }
    ```

Like we saw with `UnsafeMutablePointer`, some operations need to be adjusted:

- In Swift 5.10, `initializeElement(at:to:)` has an issue where it borrows, rather than consumes, its argument. We need to replace it with a source compatible variant that resolves this:

    ```swift
    extension UnsafeMutableBufferPointer where Element: ~Copyable {
      func initializeElement(at index: Index, to value: consuming Element)
    }
    ```

    To ensure compatibility with current binaries, we also keep providing the old function as an obsolete entry point, like we did for `UnsafeMutablePointer.initialize(to:)`.
   
- Memory rebinding operations again need to be generalized along multiple axes:

    ```swift
    extension Unsafe[Mutable]BufferPointer where Element: ~Copyable {
      public func withMemoryRebound<T: ~Copyable, E: Error, Result: ~Copyable>(
        to type: T.Type,
        _ body: (_ buffer: Unsafe[Mutable]BufferPointer<T>) throws(E) -> Result
      ) throws(E) -> Result
    }
    ```

#### Protocol conformances

The buffer pointer types also conform to the `Sequence`, `Collection`, `BidirectionalCollection`, `RandomAccessCollection` and `MutableCollection` protocols. We aren't generalizing these protocols in this proposal -- they continue to require copyable `Element` types. Therefore, buffer pointer conformances to these protocols must remain restricted to the pre-existing copyable cases. 

This also affects some related typealiases and nested types: the `UnsafeBufferPointer.Iterator` type and its `SubSequence` and `Indices` typealiases will only exist when `Element` is copyable. A buffer pointer of noncopyable elements is not a sequence, so as of this proposal it cannot be iterated over by a for-in loop. It also doesn't get any of the standard Sequence/Collection algorithms. (We expect to reintroduce these features in the future.)

However, we do propose to generalize most of the core collection operations, even without carrying the actual conformance:

```swift
extension Unsafe[Mutable]BufferPointer where Element: ~Copyable {
  typealias Index = Int
  var isEmpty: Bool { get }
  var startIndex: Int { get }
  var endIndex: Int { get }
  func index(after i: Int) -> Int
  func formIndex(after i: inout Int)
  func index(before i: Int) -> Int
  func formIndex(before i: inout Int)
  func index(_ i: Int, offsetBy n: Int) -> Int
  func index(_ i: Int, offsetBy n: Int, limitedBy limit: Int) -> Int?
  func distance(from start: Int, to end: Int) -> Int
}

extension UnsafeMutableBufferPointer where Element: ~Copyable {
  func swapAt(_ i: Int, _ j: Int)
}
```

In Swift 5.x, the indexing subscript was already defined with special accessors that support in-place mutating access. To support in-place borrowing access, we can adapt the unstable/unsafe accessors from the unsafe pointers types, to define a subscript with direct support for use with noncopyable elements:

```swift
extension Unsafe[Mutable]BufferPointer where Element: ~Copyable {
  subscript(i: Int) -> Element // (special accessors not shown)
}
```

#### Extracting parts of a buffer pointer

Unfortunately, the slicing subscript cannot be generalized, as its `Slice` return type requires a base container that conforms to `Collection`.

We therefore propose to add the following new member methods for extracting a standalone buffer that covers a range of indices:

```swift
extension UnsafeBufferPointer where Element: ~Copyable {
  func extracting(_ bounds: Range<Int>) -> Self
  func extracting(_ bounds: some RangeExpression<Int>) -> Self
  func extracting(_ bounds: UnboundedRange) -> Self
}

extension UnsafeMutableBufferPointer where Element: ~Copyable {
  func extracting(_ bounds: Range<Int>) -> Self
  func extracting(_ bounds: some RangeExpression<Int>) -> Self
  func extracting(_ bounds: UnboundedRange) -> Self
}
```

Unlike with slicing, the returned buffer does not share indices with the original -- the result is a regular buffer that has its own 0-based indices. This operation is effectively equivalent to slicing the buffer and then immediately rebasing the slice into a standalone buffer pointer: `buffer.extracting(i ..< j)` produces the same result as the expression `UnsafeBufferPointer(rebasing: buffer[i ..< j])` did in Swift 5.x.


#### Exceptions

There are also some buffer pointer operations that inherently cannot be generalized for noncopyable cases. These include:

- Operations that require copying elements:
    - `func initialize(repeating: Element)`
    - `func update(repeating: Element)`
- Operations that operate on sequences or collections of items:
    - `func initialize<S: Sequence<Element>>(from: S) -> (unwritten: S.Iterator, index: Index)`
    - `func initialize(fromContentsOf source: some Collection<Element>)`
    - `func update<S: Sequence<Element>>(from: S) -> (unwritten: S.Iterator, index: Index)`
    - `func update(fromContentsOf: some Collection<Element>) -> Index`
- Members that involve buffer pointer slices:
    - `init(rebasing slice: Slice<Unsafe[Mutable]BufferPointer<Element>>)`
    - `func moveInitialize(fromContentsOf source: Slice<Self>) -> Index`
    - `func moveUpdate(fromContentsOf source: Slice<Self>) -> Index`

These operations are not in any way deprecated; they just continue requiring `Element` to be copyable. (We do expect to introduce noncopyable alternatives for the sequence/collection operations in a subsequent proposal.)

#### Related enhancements in other types

To keep the Standard Library's family of pointer types coherent, we also need to generalize some conversion/rebinding/initialization operations on raw buffer pointers:

```swift
extension Unsafe[Mutable]RawBufferPointer {
  init<T: ~Copyable>(_ buffer: UnsafeMutableBufferPointer<T>)
  init<T: ~Copyable>(_ buffer: UnsafeBufferPointer<T>)
  
  func bindMemory<T: ~Copyable>(
    to type: T.Type
  ) -> Unsafe[Mutable]BufferPointer<T>
  
  func withMemoryRebound<T: ~Copyable, E: Error, Result: ~Copyable>(
    to type: T.Type,
    _ body: (_ buffer: Unsafe[Mutable]BufferPointer<T>) throws(E) -> Result
  ) throws(E) -> Result
  
  func assumingMemoryBound<T: ~Copyable>(
    to: T.Type
  ) -> Unsafe[Mutable]BufferPointer<T>
}

extension UnsafeMutableRawBufferPointer {
  func moveInitializeMemory<T: ~Copyable>(
    as type: T.Type,
    fromContentsOf source: UnsafeMutableBufferPointer<T>
  ) -> UnsafeMutableBufferPointer<T>
}
```


#### Temporary buffer pointers over arbitrary entities

We also need to similarly generalize the top-level `withUnsafe[Mutable]Pointer` functions that provide temporary buffer pointers over arbitrary values:

```swift
func withUnsafeMutableBytes<T: ~Copyable, E: Error, Result: ~Copyable>(
  of value: inout T,
  _ body: (UnsafeMutableRawBufferPointer) throws(E) -> Result
) throws(E) -> Result

func withUnsafeBytes<T: ~Copyable, E: Error, Result: ~Copyable>(
  of value: inout T,
  _ body: (UnsafeRawBufferPointer) throws(E) -> Result
) throws(E) -> Result

func withUnsafeBytes<T: ~Copyable, E: Error, Result: ~Copyable>(
  of value: borrowing T,
  _ body: (UnsafeRawBufferPointer) throws(E) -> Result
) throws(E) -> Result
```

All of these (and especially the borrowing variant) is subject to the same limitations as the original copyable variants: the pointers exposed are only valid for the duration of the function invocation, and multiple executions on the same instance may provide different locations for the same entity.


### Temporary Allocation Facility

The [Standard Library's facility for allocating temporary uninitialozed buffers][SE-0322] needs to be generalized to support allocating storage for noncopyable types, as well as returning a potentially noncopyable type:

[SE-0322]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0322-temporary-buffers.md

```swift
func withUnsafeTemporaryAllocation<E: Error, R: ~Copyable>(
  byteCount: Int,
  alignment: Int,
  _ body: (UnsafeMutableRawBufferPointer) throws(E) -> R
) throws(E) -> R

func withUnsafeTemporaryAllocation<T: ~Copyable, E: Error, R: ~Copyable>(
  of type: T.Type,
  capacity: Int,
  _ body: (UnsafeMutableBufferPointer<T>) throws(E) -> R
) throws(E) -> R

```

### Managed Buffers

Managed buffers provide a way for Swift container types to dynamically allocate storage for their contents in the form of a managed class reference.

We generalize managed buffer types to support noncopyable element types, including all of their existing member operations.

```swift
open class ManagedBuffer<Header, Element: ~Copyable> {
  final var header: Header
  init(_doNotCallMe: ())
}

@available(*, unavailable)
extension ManagedBuffer: Sendable where Element: ~Copyable {}

extension ManagedBuffer where Element: ~Copyable {
  // All existing members
}

struct ManagedBufferPointer<Header, Element: ~Copyable> {...}

extension ManagedBufferPointer where Element: ~Copyable {
  // All existing members
}
```

The core `withUnsafeMutablePointer` interfaces are further generalized to allow noncopyable return types:
 
```swift
extension ManagedBuffer where Element: ~Copyable {
  final func withUnsafeMutablePointerToHeader<E: Error, R: ~Copyable>(
    _ body: (UnsafeMutablePointer<Header>) throws(E) -> R
  ) throws(E) -> R
  
  final func withUnsafeMutablePointerToElements<E: Error, R: ~Copyable>(
    _ body: (UnsafeMutablePointer<Element>) throws(E) -> R
  ) throws(E) -> R
  
  final func withUnsafeMutablePointers<E: Error, R: ~Copyable>(
    _ body: (
      UnsafeMutablePointer<Header>, UnsafeMutablePointer<Element>
    ) throws(E) -> R
  ) throws(E) -> R
}
```

```swift
extension ManagedBufferPointer where Element: ~Copyable {
  func withUnsafeMutablePointerToHeader<E: Error, R: ~Copyable>(
    _ body: (UnsafeMutablePointer<Header>) throws(E) -> R
  ) throws(E) -> R
  
  func withUnsafeMutablePointerToElements<E: Error, R: ~Copyable>(
    _ body: (UnsafeMutablePointer<Element>) throws(E) -> R
  ) throws(E) -> R
  
  func withUnsafeMutablePointers<E: Error, R: ~Copyable>(
    _ body: (
      UnsafeMutablePointer<Header>, UnsafeMutablePointer<Element>
    ) throws(E) -> R
  ) throws(E) -> R
}
```

Notably, we preserve the requirement that the `Header` type must be copyable for now. It would be desirable to allow noncopyable `Header` types, but preserving compatibility with the stored property `ManagedBuffer.header` requires further work, so it is deferred. (We do not believe this to be a significant obstacle in practice.)

### Lifetime Management

The Standard Library offers the `withExtendedLifetime` family of functions to explicitly extend the lifetime of an entity to cover the entire duration of a closure. To support ownership control, we lift the copyability requirement on both the item whose lifetime is being extended, and for the return type of the function argument:

```swift
func withExtendedLifetime<T: ~Copyable, E: Error, Result: ~Copyable>(
  _ x: borrowing T,
  _ body: () throws(E) -> Result
) throws(E) -> Result
```

There exists a second variant of `withExtendedLifetime` whose function argument is passed the entity whose lifetime is being extended. This variant is less frequently used, but it still makes sense to generalize this to pass a borrowed instance:

```swift
func withExtendedLifetime<T: ~Copyable, E: Error, Result: ~Copyable>(
  _ x: borrowing T,
  _ body: (borrowing T) throws(E) -> Result
) throws(E) -> Result
```

### Swapping and exchanging items

We have a standalone `swap` function that swaps the values of two `inout` values. We propose to generalize this operation to lift its copyability requirement. This is a good opportunity to make use of the new ownership control features to greatly simplify its implementation:

```swift
func swap<T: ~Copyable>(_ a: inout T, _ b: inout T) {
  let tmp = consume a
  a = consume b
  b = consume tmp
}
```

We also propose to add a new variant of this same operation that takes a single `inout` value, setting it to a given value and returning the original:

```swift
public func exchange<T: ~Copyable>(
  _ value: inout T, 
  with newValue: consuming T
) -> T {
  var oldValue = consume value
  value = consume newValue
  return oldValue
}
```

This is a nonatomic analogue of the `exchange` operation on `struct Atomic`. This is a commonly invoked idiom, and having a standard operation for it will reduce the need to reinvent it from scratch with each use. (Thereby eliminating a potential source of errors, and improving readability.) By using `exchange`, we can avoid the need to manually introduce a second `inout` binding just to be able to invoke `swap`.

## Source compatibility

This proposal is heavily built on the assumption that removing the assumption of copyability on these constructs will not break existing code that relies on it. This is largely the case, although there are subtle cases where these generalizations break code that relies on shadowing standard declarations.

For instance, code that used to substitute their own definition of `Optional.map` (or any other newly generalized function) in place of the stdlib's official definition may find that their declaration is no longer considered to shadow the original:

```swift
extension Optional {
  func map<U>(
    _ transform: (Wrapped) throws -> U
  ) rethrows -> U? {
    print("Hello from map!")
    switch self {
    case .some(let y):
      return .some(try transform(y))
    case .none:
      return .none
    }
  }
}

let foo: Int? = 42
foo.map { $0 + 1 } // error: ambiguous use of 'map'
```

The new `map` uses typed throws and it allows noncopyable return types, rendering it different enough to make this substitution no longer shadow the original. This makes such generalizations technically source breaking; however the breakage is similar in nature and severity as a source break that can arise from new API additions that happen to clash with preexisting extensions of similar names defined outside the Standard Library. If such issues prove harmful in practice, we can subsequently amend Swift's shadowing rules to ignore differences in throwing and noncopyability.

## ABI compatibility

We limited the changes proposed so that we allow maintaining full backward compatibility with existing binaries.

Adding support for noncopyable type parameters generally changes linker-level mangled symbol names in emitted code, which would break ABI -- we avoid this either by continuing to ship the original function definitions as obsoleted `@usableFromInline internal` functions, or by overriding mangling to ignore `~Copyable` (using an unstable `@_preInverseGenerics` attribute).

We also provide a measure of forward compatibility -- newly built code that calls newly generalized functions will continue to remain compatible with previously shipped versions of the Standard Library. This naturally must apply to the preexisting copyable cases, but it also extends to noncopyable use: the newly generalized generic operations are generally expected to work on older Swift runtime environments. Of course, older runtimes do not understand noncopyable generics (or even noncopyable types in general), so features that rely on runtime dynamism will come with a stricter deployment limit. (The feature set we propose in this document is not expected to hit this.)

The `Optional` and `Result` types that shipped in previous versions of the Standard Library were naturally built with the assumption of copyability, but they tended to avoid making unnecessary copies, which means they are mostly expected to be also "magically" compatible with noncopyable use. (It is okay to break an assumption that was never actually relied on.) The places where we preserved mangling are the places where we think this applies -- we expect newly built code that invokes the old implementations will still run fine. (If we missed a case where an earlier implementation did rely on copying or runtime dynamism, we can correct it at any point by switching to the `@backDeplpoyed`/`@usableFromInline` implementation pattern.)

## Alternatives Considered

The primary alternative is to delay this work until it becomes possible to express more of the functionality that is deferred by this proposal. However, this would leave noncopyable types in a limbo state, where the language ships with rich functionality to support them, but the core Standard Library continues to treat them as second class entities.

The inability to apply unsafe pointer APIs to noncopyable types would be a particularly severe obstacle to practical adoption -- it is tricky to fully embrace ownership control if we have no way to dynamically allocate storage for noncopyable entities.

Avoiding the use of `Optional` is a similarly severe API design issue, with no elegant solutions. Forcing adopters of ownership control to define custom `Optional` types has proved impractical beyond simple throwaway prototypes; it's better to have a standard solution.

We do not consider the generalization of `Result` to be anywhere near as important as `Optional`, although it does provide a standard way to implement manual error propagation. However, as it is a close relative to `Optional`, it seems undesirable to defer its generalization.

### Omitting `UnsafeBufferPointer`

`UnsafeBufferPointer` conforms to `Collection`, and it relies on the standard `Slice` type for its `SubSequence` concept. Neither `Collection` nor `Slice` can be directly generalized for noncopyable elements, and so these conformances need to continue require copyable elements.

Given that buffer pointers are essentially useless without an idea of an index (which comes from `Collection`), we considered omitting them from this proposal, deferring their generalization until we have protocols for noncopyable container types. 

However, in practice, this would not be acceptable: the buffer pointer is Swift's native way to represent a region of direct memory, and we urgently need to enable dealing with memory regions that contain noncopyable instances. Leaving buffer pointers ungeneralized would strongly encourage Swift code to start passing around base pointers and counts as distinct items, which would be a significant step backwards -- we must avoid training Swift developers to do that. (We'd also lose the ability to generalize the `withUnsafeTemporaryAllocation` function, which is built on top of buffer pointers.) 

Therefore, this proposal generalizes buffer pointers, including the parts of `Collection` that we strongly believe will directly translate to noncopyable containers (the basic concept of an index, the index navigation members and the indexing subscript).

### Alternatives to `UnsafeBufferPointer.extracting()`

A different concern arises with buffer pointer slices. Regrettably, it seems we have to give up on the `buffer[i..<j]` notation, as the slicing subscript is unfortunately defined to return `Slice`, and that type is not readily generalizable.

We cannot change the slicing subscript to return a new type, as that would break existing code. Therefore, we're left with the option of introducing a separate operation, distinct from slicing, that targets the same use cases.

Luckily, we have close to a decade's worth Swift code using `UnsafeBufferPointer` to analyze, and a pattern readily emerges: very often, a buffer pointer gets sliced only to immediately rebase it back into a new buffer pointer value:

```swift
UnsafeMutableBufferPointer(rebasing: buffer[i ..< j])
```

This combined slicing-and-rebasing operation does directly translate to buffers with noncopyable elements, and so it is an obvious choice for a slicing substitute. We considered providing it as a new initializer:

```swift
extension Unsafe[Mutable]BufferPointer where Element: ~Copyable {
  init(rebasing range: some RangeExpression<Int>, in buffer: UnsafeBufferPointer<Element>)
}

// Usage:
UnsafeMutableBufferPointer(rebasing: i ..< j, in: buffer)
```

This easily fits into Swift API design conventions, but it doesn't feel like a good enough solution in practice. Specifically, it suffers from two distinct (but related) problems:

1. It remains just as verbose, inconvenient and non-intuitive as the original rebasing initializer; and we have considered that a significant problem even in the copyable case. 

   <small>

   (Indeed, a large part of [SE-0370][SE-370-Slice] was dedicated to reducing the need to directly invoke this initializer, by cleverly extending the `Slice` type with direct methods that [hide the `init(rebasing:)` call](https://github.com/apple/swift/blob/swift-5.10-RELEASE/stdlib/public/core/UnsafeBufferPointerSlice.swift#L699-L702
). This is very helpful, but in exchange for simplifying use sites, we've made it more difficult to define custom operations: each operation has to be defined on both the buffer pointer and the slice type, and the latter requires advanced generics trickery. Of course, none of this work helps the noncopyable case, as `Slice` does not translate there -- so we get back to where we started.)

[SE-370-Slice]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0370-pointer-family-initialization-improvements.md#slices-of-bufferpointer

   </small>

2. The new initializer would also apply to the copyable case, but it would serve no discernible purpose in that context, other than to increase confusion.

The solution we propose is to make the new operation a regular member function. This solves the first problem: `buffer.extracting(i..<j)` is not quite as elegant as `buffer[i..<j]`, but is far more readable at a glance than anything that involves an initializer call. And it also solves the second problem, as the new member function provides a shorthand notation for a very common operation, and that makes it useful even in copyable contexts where slicing continues to remain available.

Of course, we also considered simply omitting providing a substitute for slicing, deferring to tackle it (e.g., in hopes of figuring out some way to generalize `Slice` in the future). However, given its vast importance, this would be a wildly impractical choice. For example, the tiny `Hypoarray` illustration in the appendix is chock full of these operations: it contains six different places where it needs to slice and dice buffers -- in this particular example, `extracting` is in fact _the most frequently mentioned buffer pointer operation_. This underscores the need to not only provide this operation, but also to give it a proper name that reflects its importance.

## Future Work

### Non-escapable `Optional` and `Result`

Once it becomes possible to [define non-escapable types and to express lifetime dependencies](https://forums.swift.org/t/pitch-non-escapable-types-and-lifetime-dependency/69865), we will need to apply a second round of generalizations on `Optional` and `Result` to also allow non-escapable payloads. We expect this will be a far less complex step, as it'll mostly consist on sprinkling `~Escapable` on the right parts, and applying the correct lifetime dependency annotations on the interfaces we already have.

An optional holding a non-escapable entity will itself be non-escapable. That is to say, `Optional`'s escapability will be conditional on its payload, similar to how this proposal suggests to have it inherit its copyability from the same.

```swift
// Illustration; this is not real Swift code (yet?)
public enum Optional<Wrapped: ~Copyable & ~Escapable>: ~Copyable, ~Escapable {
  case none
  case some(Wrapped)
}

extension Optional: Copyable where Wrapped: ~Escapable {}
extension Optional: Escapable where Wrapped: ~Copyable {}
extension Optional: Sendable where Wrapped: ~Copyable & ~Escapable & Sendable {}

extension Optional where Wrapped: ~Copyable & ~Escapable {
  public init(_ some: consuming Wrapped) dependsOn(some) -> Self { 
    self = .some(some) 
  }
}
```

It is likely that we will want to generalize `MemoryLayout` as well. Allowing unsafe pointers to address non-escapable types is not nearly as straightforward, but it's possible we'll need to tackle that, too.

### Generalizing higher-order functions

This proposal does not allow `map` or `flatMap` to be called on noncopyable `Optional` or `Result` types yet, to avoid prematurely establishing a pattern before it becomes possible to express better solutions.

As detailed in the [Proposed Solution]((#generalizing-higher-order-functions)) section, this is mostly a naming/presentation problem: we need distinct notations for the `map` that consumes `self` vs. the variant that merely borrows it.

One straightforward idea is to simply use `consuming` and `borrowing` as naming prefixes:

```swift
extension Optional where Wrapped: ~Copyable {
  consuming func consumingMap<E: Error, U: ~Copyable>(
    _ transform: (consuming Wrapped) throws(E) -> U
  ) throws(E) -> U?

  borrowing func borrowingMap<E: Error, U: ~Copyable>(
    _ transform: (borrowing Wrapped) throws(E) -> U
  ) throws(E) -> U?
}
```

This is a somewhat verbose choice, but it makes the choice eminently clear at point of use:

```swift
struct Wrapper<T: ~Copyable>: ~Copyable {
  var value: T
}

let v: Wrapper<Int>?
print(v.borrowingMap { $0.value })

let w: Wrapper<FilePath>?
let file = try v.consumingMap { try File(opening: $0) }
```

The primary drawback of this simple solution is that developers working with classic (i.e. copyable) `Optional` values would now be faced with three separate APIs for what is (from their viewpoint) the same operation. Making a distinction between guaranteed-consuming and guaranteed-borrowing transformations is not entirely pointless even in the copyable case, but it is mostly a nitpicky performance detail that wouldn't otherwise merit any new API additions. However, the distinction is crucial for noncopyable use, and that may excuse the new variations even if they mean additional noise for the classic copyable cases.

A similar idea is to introduce `consuming` and `borrowing` views, and to move the ownership-aware operations into them, leaving us with the notations `v.borrowing.map { $0.value }` or `v.consuming.map { try File(opening: $0) }`. These are also eminently readable, and they would also be a good spiritual fit with the `.lazy` sequence view we already have. They also help with the noise issue, as the nitpicky variants with explicit ownership annotations would all get hidden away in views dedicated to ownership control.

The idea of a "consuming view" is a bit of a stretch, as it doesn't seem particularly useful outside of this context; but a "borrowing view" certainly would have merit on its own -- it would be a type that consists of a "borrow" of an instance of some other type, which would be an independently useful construct. (E.g., it would allow us to generalize `Slice` into a `BorrowingSlice` while keeping it generic over the base container.) 

Therefore, the best choice may be to introduce the idea of a `borrowing` view (returning a standard `Borrow<T>` (or `Ref<T>`) type), but to avoid introducing a `consuming` view, preferring to instead generalize the existing `map`/`flatMap`/`filter`/`reduce` etc functions in the consuming sense. So `v.map { ... }` would be (implicitly) consuming, while `v.borrowing.map { ... }` would be explicitly borrowing.

It isn't currently possible to implement generic borrowing views, as structs can only contain owned instances of another type, not borrowed ones. Therefore, we need to delay work on consuming/borrowing higher-order functions until it becomes possible to express such a thing. (We could implement the `consumingMap` and `borrowingMap` naming convention right now, but it seems likely that we'd regret that when it becomes possible to express the borrowing view concept.)

### Generalizing `Optional.unsafelyUnwrapped`

The `unsafelyUnwrapped` property of `Optional` implements an unsafe variant of the safe force-unwrap operation that is built into Swift (denoted `!`). (This property is _unsafe_ because it does not guarantee to check if the optional is empty before attempting to extract its wrapped value. Trying to access a value that isn't there is undefined behavior.)

This proposal keeps this property in its original form, so it will be only available if `Wrapped` is copyable.

Ideally, `unsafelyUnwrapped` would be generalized to follow the same adaptive behavior as the force-unwrap form, allowing both consuming and borrowing use.

To achieve this, Swift would need to implement the following three enhancements:

1. Provide a way to define a (coroutine based) borrowing accessor on a computed property
2. Provide a way to define an accessor on a computed property that consumes `self` (i.e., a consuming getter).
3. Allow these two accessors to coexist within the same property, with the language inferring which one to use based on usage context.

Generalizing `unsafelyUnwrapped` needs to be deferred either until these become possible or until we decide not to do them.

```swift
// Illustration; this is not real Swift
extension Optional where Wrapped: ~Copyable {
  var unsafelyUnwrapped: Wrapped {
    consuming get { ... }
    read { ... }
    modify { ... } // Let's throw this in the mix as well
  }
}
```

In the meantime, we considered adding a separate `unsafeUnwrap()` member to provide a separate solution for point 2 above:

```swift
extension Optional where Wrapped: ~Copyable {
  consuming func unsafeUnwrap() -> Wrapped
}
```

However, if we do end up getting these enhancements, then this new function would become an unnecessary addition. As this is a rather obscure/niche operation, it doesn't seem worth this trouble.

### Generalized managed buffer headers

This proposal lifts the copyability requirement on `ManagedBuffer`'s `Element` type, but it continues to require `Header` to be copyable. 

Of course, it would be desirable to lift this requirement, too. Unfortunately, `ManagedBuffer` exposes the public (stored) property `header`, and lifting the copyability requirement would break this property's (implicit) ABI for low-level access. Until we find a way to mitigate this problem, we cannot generalize stored properties to remove the assumption of copyability; therefore, we need to postpone generalizing `Header`.

Requiring a copyable `Header` does not appear to be a significant hurdle in most use cases, so it seems preferable to leave time to design a proper solution rather than attempting to ship a quick stopgap fix that may prove to be incomplete.

### Additional raw pointer operations

`Unsafe[Mutable]RawPointer` includes the `load(fromByteOffset:as:)` operation that directly returns a copy an instance of an arbitrary type at the indicated location. We kept this restricted to copyable types, and we refrained from providing noncopyable equivalents, such as the closure-based member below:

```swift
extension Unsafe[Mutable]RawPointer {
   func withValue<T: ~Copyable, E: Error, Result: ~Copyable>(
    atByteOffset offset: Int = 0,
    as type: T.Type,
    _ body: (borrowing T) throws(E) -> Result
  ) throws(E) -> Result
}
```

We also do not provide a mutating operation that consumes an instance at a particular offset:

```swift
extension UnsafeMutableRawPointer {
   func move<T: ~Copyable>(
     fromByteOffset offset: Int = 0,
     as type: T.Type
   ) -> T
}
```

We omitted these, as it is unclear if these would be the best ways to express these. For now, we instead recommend explicitly binding memory and using `Unsafe[Mutable]Pointer` operations.

### Protocol generalizations

[As noted above](#lack-of-protocol-generalizations), this proposal leaves most standard protocols as is, deferring their generalizations to subsequent future work. The single protocol we do generalize is `ExpressibleByNilLiteral` -- the `nil` syntax is so closely associated with the `Optional` type that it would not have been reasonable to omit it.

This of course is not tenable; we expect that many (or even most) of our standard protocols will need to eventually get generalized for noncopyable use.

For some protocols, this work is relatively straightforward. For example, we expect that generalizing `Equatable`, `Hashable` and `Comparable` would not be much of a technical challenge -- however, it will involve overhauling/refining `Equatable`'s semantic requirements, which I do not expect to be an easy process. (`Equatable` currently requires that "equality implies substitutability"; if the two equal instances happen to be noncopyable, such unqualified, absolute statements no longer seem tenable.) The `RawRepresentable` protocol is also in this category.

In other cases, the generalization fundamentally requires additional language enhancements. For example, we may want to consider allowing noncopyable `Error` types -- but that implies that we'll also want to throw and catch noncopyable errors, and that will require a bit more work than adding a `~Copyable` clause on the protocol. It makes sense to defer generalizing the protocol until we decide to do this; if/when we do, the generalizations of `Result` can and should be part of the associated discussion and proposal. Another example is `ExpressibleByArrayLiteral`, which is currently built around an initializer with a variadic parameter -- to generalize it, we need to either figure out how to generalize those, or we need to design some alternative interface.

In a third category of cases, the existing protocols make heavy use of copyability to (implicitly) unify concerns that need stay distinct when we introduce ownership control. Retroactively untangling these concerns is going to be difficult at best -- and sometimes it may in fact prove impractical. For instance, the current `Sequence` protocol is shaped like a consuming construct: `makeIterator` semantically consumes the sequence, and `Iterator.next()` passes ownership of the elements to its caller. However, the documentation of `Sequence` explicitly allows conforming types to implement multipass/nondestructive behavior, and it in fact it _requires_ `Collection` types to do precisely that. By definition, a consuming sequence cannot be multipass; such sequences are borrowing by nature. To support noncopyable elements, we'll need to introduce distinct abstractions for borrowing and consuming sequences. Generalizing the existing `Sequence` in either of these directions seems fraught with peril.

Each of these protocol generalizations will require effort that's _at least_ comparable in complexity to this proposal; so it makes sense to consider them separately, in a series of future proposals.

### Additional future work

Fully supporting ownership control and noncopyable types will require overhauling much of the existing Standard Library. 

This includes generalizing dynamic runtime operations -- a huge area that includes facilities such as isa checks, downcasts, existentials, reflection, key paths, etc. (For instance, updating `print()` to fully support printing noncopyable types is likely to require many of these dynamic features to work.)

On the way to generalizing the Standard Library's current sequence and collection abstractions, we'll also need to implement a variety of alternatives to the existing copy-on-write collection types, `Array`, `Set`, `Dictionary`, `String`, etc, providing clients direct control over (runtime and memory) performance: consider a fixed-capacity array type, or a stack-allocated dictionary construct.

Many of these depend on future language enhancements, and as such they will be developed alongside those.

## Appendix: `struct Hypoarray`

Hypoarray is a simple noncopyable generic struct that is a very thin, safe wrapper around a piece of directly allocated memory. It is presented here as an illustration of the pointer improvements introduced in this document.

This section is not normative: we are not proposing to add a `Hypoarray` type to the Standard Library. However, it illustrates the use of the proposed Standard Library extensions, and it does serve as a first prototype for a potential future addition.

This type operates on a lower level of abstraction than the standard `Array` type. "Hypo" is greek for "under", so "hypoarray" is an apt name for such a construct. (In fact, if we started anew, the existing `Array` would potentially be built on top of such a construct.) 

A hypoarray is like an `Array` without the implicit copy-on-write machinery: it is still dynamically allocated, and it can still implicitly resize itself as needed, but it replaces copy-on-write behavior with strict ownership control. Its storage is always uniquely held, so every mutation can be done in place, resulting in more predictable performance. (Although implicit reallocations can still result in unexpected spikes of latency! To get rid of those, we'd need to introduce an even lower-level array variant that has a fixed capacity. We'll leave that as an exercise to the reader for now.)

A hypoarray consists of a dynamically allocated storage buffer (of variable capacity) and an integer count that specifies how many initialized elements it contains. The elements of the array are all compacted at the beginning of storage, with any remaining slots serving as free capacity for future additions.

```swift
struct Hypoarray<Element: ~Copyable>: ~Copyable {
  private var _storage: UnsafeMutableBufferPointer<Element>
  private var _count: Int
```

The buffer's count is the current capacity of the hypoarray. We'll need to keep referring to it elsewhere, so it makes sense to introduce a name for it early on:

```swift
  var capacity: Int { _storage.count }
```

Initializing an empty array can be done by simply setting up an empty buffer, and setting the count to zero.

```swift
  init() {
    _storage = .init(start: nil, count: 0)
    _count = 0
  }
```

That wasn't very interesting, so to spruce things up, we can also provide a single-element initializer that needs to actually allocate and initialize some memory:

```swift
  init(_ element: consuming Element) {
    _storage = .allocate(capacity: 1)
    _storage.initializeElement(at: 0, to: element)
    _count = 1
  }
```

This nontrivial initializer takes ownership of the element it is given, so naturally it has to be declared to consume its argument.

(Of course, we will eventually also want to have an initializer that can take any sequence of elements; however, this needs the idea a sequence type that produces consumable items, and we do not yet have a protocol that could express that. The `Sequence` we currently have requires its `Element` to be copyable, and it inherently combines borrowing and consuming iteration into a single, convenient abstraction. Sadly it does not directly translate to noncopyable use.)

When the array is destroyed, we need to properly deinitialize its elements and deallocate its storage. To do this, we need to define a deinitializer:

```swift
  deinit {
    _storage.extracting(0 ..< count).deinitialize()
    _storage.deallocate()
  }
}
```

Note the use of the new `extracting` operation to get a buffer pointer that consists of just the slots that have been populated. We cannot call `_storage.deinitialize()` as it isn't necessarily fully initialized; and we also cannot use the classic slicing operation `_storage[..<count]`, as it would need to return a `Slice`, and that type doesn't support noncopyable elements.

Hypoarrays can be declared sendable when their element type is sendable. 

```swift
extension Hypoarray: @unchecked Sendable 
where Element: Sendable & ~Copyable {}
```

`Hypoarray` relies on unsafe pointer operations and dynamic memory allocation to implement its storage, so the compiler is not able to prove that it'll be correctly sendable. The `@unchecked` attribute acknowledges this and promises that the type is still following the rules of sendability.

We need to remember to suppress the copyability of `Element`. If we forgot that, then this conditional sendability would only apply if `Element` happened to be copyable. 

Of course, an array needs to provide access to its contents, and it also needs operations to add and remove elements. The task of inventing variants of the `Sequence` and `Collection` protocols that allow noncopyable conforming types and element types is deferred to a subsequent proposal, but we can safely expect that even generalized array types will be based on the concept of an integer index, and that the existing indexing operations in `Collection` will largely translate into the noncopyable universe:

```swift
extension Hypoarray where Element: ~Copyable {
  typealias Index = Int

  var isEmpty: Bool { _count == 0 }
  var count: Int { _count }

  var startIndex: Int { 0 }
  var endIndex: Int { _count }
  func index(after i: Int) -> Int { i + 1 }
  func index(before i: Int) -> Int { i - 1 }
  func distance(from start: Int, to end: Int) -> Int { end - start }
  // etc.
}
```

The most fundamental `Collection` operation is probably its indexing subscript for accessing a particular element. Obviously, we need hypoarray to provide this functionality, too. 

Unfortunately, subscripts (and computed properties) cannot currently return noncopyable results without transferring ownership of the result to the caller. 

```swift
// Illustration: an array of atomic integers
import Synchronization
let array = Hypoarray(Atomic(42))
print(array[0]) // This cannot work!
```

The subscript getter would need to move the item out of the array to give ownership to the caller, which we do not want. Getter accessors will need to be generalized into a coroutine-based read accessor that supports in-place borrowing access. (And setters need to be generalized to allow in-place mutating access.) [Introducing such accessors is still in progress][_modify], so for now, the best we can do is to provide closure-based access methods:

[_modify]: https://forums.swift.org/t/modify-accessors/31872

```swift
extension Hypoarray where Element: ~Copyable {
  func borrowElement<E: Error, R: ~Copyable> (
    at index: Int, 
    by body: (borrowing Element) throws(E) -> R
  ) throws(E) -> R {
    precondition(index >= 0 && index < _count)
    return try body(_storage[index])
  }

  mutating func updateElement<E: Error, R: ~Copyable> (
    at index: Int, 
    by body: (inout Element) throws(E) -> R
  ) throws(E) -> R {
    precondition(index >= 0 && index < _count)
    return try body(&_storage[index])
  }
}
```

These are quite clumsy, but they do work safely, and they provide in-place borrowing and mutating access to any element in a hypoarray, without having to change its ownership.

```swift
// Example usage:
var array = Hypoarray<Int>(42)
array.updateElement(at: 0) { $0 += 1 }
array.borrowElement(at: 0) { print($0) } // Prints "43"
```

[[Aside: A future language extension will hopefully allow us to replace these with the subscript we actually want to write, along the lines of this hypothetical example:

```swift
// This isn't real Swift yet:
extension Hypoarray where Element: ~Copyable {
  subscript(position: Int) -> Element {
    read {
      precondition(position >= 0 && position < _count)
      try yield _storage[position]
    }
    modify {
      precondition(position >= 0 && position < _count)
      try yield &_storage[position]
    }
  }
}
```

```swift
// Example usage:
var array = Hypoarray<Int>(42)
array[0] += 1
print(array[0]) // Prints "43"
```

Note that the proposed `UnsafeMutableBufferPointer` changes already include a subscript that allows in-place borrowing and mutating use. However, the solution used there is tied to low-level unsafe pointer semantics that would not directly translate to a higher-level type like `Hypoarray`.]]

It would be desirable to allow iteration over `Hypoarray` instances. Unfortunately, Swift's `for in` construct currently relies on `protocol Sequence`, and that protocol doesn't support noncopyable use. (Not only does it require copyable conforming types and copyable `Element`s, but its iterator is also defined to give ownership of returned elements to the caller; that is to say, it is shaped like a _consuming_ construct, not a _borrowing_ one.) Introducing a mechanism for borrowing iteration, and retooling `for in` loops to allow such use is future work. While that work is in progress, we can of course still manually iterate over the contents of a hypoarray by using its indices:

```swift
// Example usage:
var array: Hypoarray<Int> = ...
for i in array.startIndex ..< array.endIndex { // a.k.a. 0 ..< array.count
  array.borrowElement(at: i) { print($0) }
}
```

Not having noncopyable container protocols also means that `Hypoarray` cannot conform to any, so subsequently it will not get any of the standard generic container algorithms for free: there is no `firstIndex(of:)`, there is no `map`, no `filter`, no slicing, no `sort`, no `reverse`. Indeed, many of these standard algorithms expect to work on `Equatable` or `Comparable` items, and those protocols are also yet to be generalized.

Okay, so all we have is `borrowElement` and `updateElement`, for borrowing and mutating access. What about consuming access, though? 

Consuming an item of an array at a particular index would require either removing the item from the array, or destroying and discarding the rest of the array. Neither of these looks desirable as a primitive operation for accessing an element. However, we do expect arrays to provide a named operation for removing items, `remove(at:)`. This operation is easily implementable on `Hypoarray`:

```swift
extension Hypoarray where Element: ~Copyable {
  @discardableResult
  mutating func remove(at index: Int) -> Element {
    precondition(index >= 0 && index < count)
    let old = _storage.moveElement(from: index)
    let source = _storage.extracting(index + 1 ..< count)
    let target = _storage.extracting(index ..< count - 1)
    let i = target.moveInitialize(fromContentsOf: source)
    assert(i == target.endIndex)
    _count -= 1
    return old
  }
}
```

Note how this moves the removed element out of the array, so it can legitimately give ownership of it to the caller. Following our preexisting convention, the result of `remove(at:)` is marked discardable; if the caller decides to discard it, then the removed item immediately gets destroyed, as expected.

(Implementing the classic `removeSubrange(_: some RangeExpression<Int>)` operation is left as an exercise for the reader.)

Okay, so now we know how to create simple single-element hypoarrays, how to access their contents, and we are even able to remove elements from them. How do we add new elements, though?

Hypoarray is supposed to implement a dynamically resizing array, so insertions generally need to be able to expand storage. Let's tackle this sub-problem first, by implementing `reserveCapacity`:

```swift
extension Hypoarray where Element: ~Copyable {
  mutating func reserveCapacity(_ n: Int) {
    guard capacity < n else { return }
    let newStorage: UnsafeMutableBufferPointer<Element> = .allocate(capacity: n)
    let source = _storage.extracting(0 ..< count)
    let i = newStorage.moveInitialize(fromContentsOf: source)
    assert(i == count)
    _storage.deallocate()
    _storage = newStorage
  }
}
```

Note again the use of `extracting` to operate on parts of a buffer -- in this case, we use it to move initialized items between the two allocations.

We want insertions to have amortized O(1) complexity, so they need to be careful about the rate at which they grow the array's storage. In this simple illustration, we'll use a geometric growth factor of 2, so that each reallocation will at least double the capacity of the array:

```swift
extension Hypoarray where Element: ~Copyable {
  mutating func _ensureFreeCapacity(_ minimumCapacity: Int) {
    guard capacity < _count + minimumCapacity else { return }
    reserveCapacity(max(_count + minimumCapacity, 2 * capacity))
  }
}
```

With that done, we can finally implement insertions, starting with the `append` operation. Its implementation is fairly straightforward:

```swift
extension Hypoarray where Element: ~Copyable {
  mutating func append(_ item: consuming Element) {
    _ensureFreeCapacity(1)
    _storage.initializeElement(at: _count, to: item)
    _count += 1
  }
}
```

Inserting at a particular index is complicated by the need to make room for the new item, but it's not that tricky, either:

```swift
extension Hypoarray where Element: ~Copyable {
  mutating func insert(_ item: consuming Element, at index: Int) {
    precondition(index >= 0 && index <= count)
    _ensureFreeCapacity(1)
    if index < count {
      let source = _storage.extracting(index ..< count)
      let target = _storage.extracting(index + 1 ..< count + 1)
      target.moveInitialize(fromContentsOf: source)
    }
    _storage.initializeElement(at: index, to: item)
    _count += 1
  }
}
```

```swift
// Example usage:
var array = Hypoarray<Int>()
for i in 0 ..< 10 {
  array.insert(i, at: 0)
}
// array now consists of 9, 8, 7, 6, 5, 4, 3, 2, 1, 0
```

Without noncopyable container protocols, we cannot yet implement `append(contentsOf:)`, `insert(contentsOf:)`, `replaceSubrange` operations. But we can still provide classic `Sequence`/`Collection`-based operations in cases where `Element` happens to copyable:

```swift
extension Hypoarray {
  mutating func append(contentsOf items: some Sequence<Element>) {
    for item in items {
      append(item)
    }
  }
}
```

Note how this extension omits the suppression of element copyability -- it does not have a `where Element: ~Copyable` clause. This means that the extension only applies if `Element` is copyable.

These operations give us all primitive operations we expect an array type to provide. Of course, the `Hypoarray` we have now created is just the very first draft of a future dynamically sized noncopyable array type. There is plenty of work left: we need to add more operations; we need to implement noncopyable variants of more data structures; we need to define the general shape of a noncopyable container; we need to populate that shape with a family of standard generic algorithms. Implicit resizing is not always appropriate in memory-starved or low-latency applications, so for those use cases we also need to design data structure variants that work within some fixed storage capacity (or even a fixed count). We may want the backing store to be allocated dynamically, like we've seen, or we may want it to become part of the construct's representation ("inline storage"); perhaps we want to allocate storage on the stack, or statically reserve space for a global variable at compile time. We expect future work will tackle all these tasks, and plenty more.
