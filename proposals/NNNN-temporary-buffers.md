# Temporary uninitialized buffers

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Jonathan Grynspan](https://github.com/grynspan)
* Review Manager: TBD
* Status: **Awaiting implementation**
* Implementation: [apple/swift#37666](https://github.com/apple/swift/pull/37666)

## Introduction

This proposal introduces new Standard Library functions for manipulating temporary buffers that are preferentially allocated on the stack instead of the heap.

Swift-evolution thread: [[Pitch] Temporary uninitialized buffers](https://forums.swift.org/t/pitch-temporary-uninitialized-buffers/48954)

## Motivation

Library-level code often needs to deal with C functions, and C functions have a wide variety of memory-management approaches. A common way to handle buffers of value types (i.e. structures) in C is to have a caller allocate a buffer of sufficient size and pass it to a callee to initialize; the caller is then responsible for deinitializing the buffer (if non-trivial) and then deallocating it. In C or Objective-C, it's easy enough to stack-allocate such a buffer, and the logic to switch to the heap for a larger allocation is pretty simple too. This sort of pattern is pervasive:

```c
size_t tacoCount = ...;
taco_fillings_t fillings = ...;

taco_t *tacos = NULL;
taco_t stackBuffer[SOME_LIMIT];
if (tacoCount < SOME_LIMIT) {
  tacos = stackBuffer;
} else {
  tacos = calloc(tacoCount, sizeof(taco_t));
}

taco_init(tacos, tacoCount, &fillings);

// do some work here
ssize_t tacosEatenCount = tacos_consume(tacos, tacoCount);
if (tacosEatenCount < 0) {
  int errorCode = errno;
  fprintf(stderr, "Error %i eating %zu tacos: %s", errorCode, tacoCount, strerror(errorCode));
  exit(EXIT_FAILURE);
}

// Tear everything down.
taco_destroy(tacos, tacoCount);
if (buffer != stackBuffer) {
  free(buffer);
}
```

In C++, we can make judicious use of `std::array` and `std::vector` to achieve the same purpose while generally preserving memory safety.

But there's not really any way to express this sort of transient buffer usage in Swift. All values must be initialized in Swift before they can be used, but C typically claims responsibility for initializing the values passed to it, so a Swift caller ends up initializing the values _twice_. The caller can call `UnsafeMutableBufferPointer<taco_t>.allocate(capacity:)` to get uninitialized memory, of course. `allocate(capacity:)` places buffers on the heap by default, and the optimizer can only stack-promote them after performing escape analysis.

Since Swift requires values be initialized before being used, and since escape analysis is [undecidable](https://duckduckgo.com/?q=escape+analysis+undecidable) for non-trivial programs, it's not possible to get efficient (i.e. stack-allocated and uninitialized) temporary storage in Swift.

It is therefore quite hard for developers to provide Swift overlays for lower-level C libraries that use these sorts of memory-management techniques. This means that an idiomatic Swift program using a C library's Swift overlay will perform less optimally than the same program would if written in C.

## Proposed solution

I propose adding a new transparent function to the Swift Standard Library that would allocate a buffer of a specified type and capacity, provide that buffer to a caller-supplied closure, and then deallocate the buffer before returning. The buffer would be passed to said closure in an uninitialized state and treated as uninitialized on closure return—that is, the closure would be responsible for initializing and deinitializing the elements in the buffer.

A typical use case might look like this:

```swift
// Eat a temporary taco buffer. A buffet, if you will.
try taco_t.consume(count: tacoCount, filledWith: ...)

// MARK: - Swift Overlay Implementation

extension taco_t {
  public static func consume(count: Int, filledWith fillings: taco_fillings_t) throws {
    withUnsafeUninitializedMutableBufferPointer(to: taco_t.self, capacity: count) { buffer in
      withUnsafePointer(to: fillings) { fillings in
        taco_init(buffer.baseAddress!, buffer.count, fillings)
      }
      defer {
        taco_destroy(buffer.baseAddress!, buffer.count)
      }
    
      let eatenCount = tacos_consume(buffer.baseAddress!, buffer.count)
      guard eatenCount >= 0 else {
        let errorCode = POSIXErrorCode(rawValue: errno) ?? .ENOTHUNGRY
        throw POSIXError(errorCode)
      }
    }
  }
}
```

The proposed function allows developers to effectively assert to the compiler that the buffer pointer used in the closure cannot escape the closure's context (even if calls are made to non-transparent functions that might otherwise defeat escape analysis.) Because the compiler then "knows" that the pointer does not escape, it can optimize much more aggressively.

A library developer going the extra mile would probably want to produce an interface that we would consider idiomatic in Swift. With the proposed function, the developer would be able to build such an interface without sacrificing C's performance. (The exact design of the hypothetical Swift "taco" interface is beyond the scope of this proposal.)

## Detailed design

A new free function would be introduced in the Standard Library:

```swift
/// Provides scoped access to a buffer pointer to memory of the specified type
/// and with the specified capacity.
///
/// - Parameters:
///   - type: The type of the buffer pointer being temporarily allocated.
///   - capacity: The capacity of the buffer pointer being temporarily
///     allocated.
///   - body: A closure to invoke and to which the allocated buffer pointer
///     should be passed.
///
///  - Returns: Whatever is returned by `body`.
///
///  - Throws: Whatever is thrown by `body`.
///
/// This function is useful for cheaply allocating storage for a sequence of
/// values for a brief duration. Storage may be allocated on the heap or on the
/// stack, depending on the required size and alignment.
///
/// When `body` is called, the contents of the buffer pointer passed to it are
/// in an unspecified, uninitialized state. `body` is responsible for
/// initializing the buffer pointer before it is used _and_ for deinitializing
/// it before returning. `body` does not need to deallocate the buffer pointer.
///
/// The implementation may allocate a larger buffer pointer than is strictly
/// necessary to contain `capacity` values of type `type`. The behavior of a
/// program that attempts to access any such additional storage is undefined.
///
/// The buffer pointer passed to `body` (as well as any pointers to elements in
/// the buffer) must not escape—it will be deallocated when `body` returns and
/// cannot be used afterward.
@_transparent
public func withUnsafeUninitializedMutableBufferPointer<T, R>(
  to type: T.Type,
  capacity: Int,
  _ body: (UnsafeMutableBufferPointer<T>) throws -> R
) rethrows -> R
```

We could optionally provide additional free functions for dealing with a raw buffer or a pointer to a single value. All of the proposed functions can be layered atop each other, so only one underlying implementation is ultimately needed:

```swift
/// Provides scoped access to a raw buffer pointer with the specified byte count
/// and alignment.
///
/// - Parameters:
///   - byteCount: The number of bytes to temporarily allocate. `byteCount` must
///     not be negative.
///   - alignment: The alignment of the new, temporary region of allocated
///     memory, in bytes.
///   - body: A closure to invoke and to which the allocated buffer pointer
///     should be passed.
///
///  - Returns: Whatever is returned by `body`.
///
///  - Throws: Whatever is thrown by `body`.
///
/// This function is useful for cheaply allocating raw storage for a brief
/// duration. Storage may be allocated on the heap or on the stack, depending on
/// the required size and alignment.
///
/// When `body` is called, the contents of the buffer pointer passed to it are
/// in an unspecified, uninitialized state. `body` is responsible for
/// initializing the buffer pointer before it is used _and_ for deinitializing
/// it before returning. `body` does not need to deallocate the buffer pointer.
///
/// The implementation may allocate a larger buffer pointer than is strictly
/// necessary to contain `byteCount` bytes. The behavior of a program that
/// attempts to access any such additional storage is undefined.
///
/// The buffer pointer passed to `body` (as well as any pointers to elements in
/// the buffer) must not escape—it will be deallocated when `body` returns and
/// cannot be used afterward.
@_transparent
public func withUnsafeUninitializedMutableRawBufferPointer<R>(
  byteCount: Int,
  alignment: Int,
  _ body: (UnsafeMutableRawBufferPointer) throws -> R
) rethrows -> R

/// Provides scoped access to a pointer to memory of the specified type.
///
/// - Parameters:
///   - type: The type of the pointer to allocate.
///   - body: A closure to invoke and to which the allocated pointer should be
///     passed.
///
///  - Returns: Whatever is returned by `body`.
///
///  - Throws: Whatever is thrown by `body`.
///
/// This function is useful for cheaply allocating storage for a single value
/// for a brief duration. Storage may be allocated on the heap or on the stack,
/// depending on the required size and alignment.
///
/// When `body` is called, the contents of the pointer passed to it are in an
/// unspecified, uninitialized state. `body` is responsible for initializing the
/// pointer before it is used _and_ for deinitializing it before returning.
/// `body` does not need to deallocate the pointer.
///
/// The pointer passed to `body` must not escape—it will be deallocated when
/// `body` returns and cannot be used afterward.
@_transparent
public func withUnsafeUninitializedMutablePointer<T, R>(
  to type: T.Type,
  _ body: (UnsafeMutablePointer<T>) throws -> R
) rethrows -> R
```

Note the functions are marked `@_transparent` to ensure they are emitted into the calling frame. This is consistent with the annotations on most other pointer manipulation functions.

### New builtin

The proposed functions will need to invoke a new builtin function equivalent to C's `alloca()`, which I have named `Builtin.stackAlloc()`. Its effective declaration is:

```swift
extension Builtin {
  func stackAlloc(_ byteCount: Builtin.Word, _ alignment: Builtin.Word) -> Builtin.RawPointer?
}
```

If the alignment and size are known at compile-time, the compiler can convert a call to `stackAlloc()` into a single LLVM `alloca` instruction. If either needs to be computed at runtime, a dynamic stack allocation can instead be emitted by the compiler.

### Location of allocated buffers

The proposed functions do _not_ guarantee that their buffers will be stack-allocated. This omission is intentional: guaranteed stack-allocation would make this feature equivalent to C99's variable-length arrays—a feature that is extremely easy to misuse and which is the cause of many [real-world security vulnerabilities](https://duckduckgo.com/?q=cve+variable-length+array). Instead, the proposed functions should stack-promote aggressively, but heap-allocate (just as `UnsafeMutableBufferPointer.allocate(capacity:)` does today) when passed overly large sizes.

This fallback heuristic is an implementation detail and may be architecture- or system-dependent. A common C approach is to say "anything larger than _n_ bytes uses `calloc()`. The Standard Library could refine this approach by checking information available to it at runtime, e.g. the current thread's available stack space. Because the Standard Library would own this heuristic, all adopters would benefit from it and, subject to a recompile, from any enhancements made in future Swift revisions.

## Source compatibility

This is new API, so there are no source compatibility considerations.

## Effect on ABI stability

This is new API, so there are no ABI stability considerations. The proposed functions should always be inlined into their calling frames, so they should be back-deployable to older Swift targets.

## Effect on API resilience

The addition of the proposed functions does not affect API resilience. If they were removed in a future release, it would be a source-breaking change but not an ABI-breaking change, because the proposed functions should always be inlined into their calling frames.

## Alternatives considered

In the pitch thread for this proposal, a number of alternatives were discussed:

### Doing nothing

* A number of developers both at Apple and in the broader Swift community have indicated that the performance costs cited above are measurably affecting the performance of their libraries and applications.
* The proposed functions would let developers build higher-order algorithms, structures, and interfaces that they cannot build properly today.

### Naming the functions something different

* One commenter suggested making the proposed functions static members of `UnsafeMutableBufferPointer` (etc.) instead of free functions. I don't feel strongly here, but the Standard Library has precedent for producing transient resources via free function, e.g. `withUnsafePointer(to:)` and `withUnsafeThrowingContinuation(_:)`. I am not immediately aware of counter-examples in the Standard Library.
* Several commenters proposed less verbose names: `withEphemeral(...)`, `withUnsafeLocalStorage(...)`, and `withUnsafeUninitializedBuffer(...)` were all suggested. I don't have strong opinions here and will defer to reviewers' wisdom here.

### Exposing some subset of the three proposed functions

* One commenter wanted to expose _only_ `withUnsafeUninitializedMutableRawBufferPointer(byteCount:alignment:_:)` in order to add friction and reduce the risk that someone would adopt the function without understanding its behaviour. Since most adopters would immediately need to call `bindMemory(to:)` to get a typed buffer, my suspicion is that developers would quickly learn to do so anyway.
* Another commenter did not want to expose `withUnsafeUninitializedMutablePointer(to:_)` on the premise that it is trivial to get an `UnsafeMutablePointer` out of an `UnsafeMutableBufferPointer` with a `count` of `1`. It is indeed easy to do so, however the two types have different sets of member functions and I'm not sure that the added friction _improves_ adopting code. On the other hand, if anyone needs a _single_ stack-allocated value, they can use `Optional` today to get one.

### Letting the caller specify a size limit for stack promotion

* It is unlikely that the caller will have sufficient additional information about the state of the program such that it can make better decisions about stack promotion than the compiler and/or Standard Library.

### Exposing this functionality as a type rather than as a scoped function

* It is likely the capacity of such a type would need to be known at compile time. Swift already has a mechanism for declaring types with a fixed-size sequence of values: homogeneous tuples. In fact, Michael Gottesman has [a pitch](https://forums.swift.org/t/pitch-improved-compiler-support-for-large-homogenous-tuples/49023) open at the time of this writing to add syntactic sugar to make homogeneous tuples look more like C arrays.
* As a type, values thereof would need to be initialized before being used. They would impose the same initialization overhead we want to avoid.
* A type, even a value type, suffers from the same stack-promotion limitations as `UnsafeMutableBufferPointer<T>` or `Array<T>`, namely that the optimizer must act conservatively and may still need to heap-allocate. Value types also get copied around quite a bit (although the Swift compiler is quite good at copy elision.)
* One commenter suggested making this hypothetical type _only_ stack-allocatable, but no such type exists today in Swift. It would be completely new to both the Swift language and the Swift compiler. It would not generalize well, because (to my knowledge) there are no other use cases for stack-only types.

### Telling adopters to use `ManagedBuffer<Void, T>`

* `ManagedBuffer` has the same general drawbacks as any other type (see above.)
* `ManagedBuffer` is a reference type, not a value type, so the compiler _defaults_ to heap-allocating it. Stack promotion is possible but is not the common case.
* `ManagedBuffer` is not a great interface in and of itself.
* To me, `ManagedBuffer` says "I want to allocate a refcounted object with an arbitrarily long tail-allocated buffer" thus avoiding two heap allocations when one will do. I can then use that object as I would use any other object. This sort of use case doesn't really align with the use cases for the proposed functions.

### Exposing an additional function to initialize a value by address without copying

* One commenter suggested:
    > A variation of the signature that gives you the initialized value you put in the memory as the return value, `makeValueAtUninitializedPointer<T>(_: (UnsafeMutablePointer<T>) -> Void) -> T`, which could be implemented with return value optimization to do the initialization in-place.

    The proposed functions can be used for this purpose:

    ```swift
    let value = withUnsafeUninitializedMutablePointer(to: T.self) { ptr in
      ...
      return ptr.move()
    }
    ```

    Subject to the optimizer eliminating the `move()`, which in the common case it should be able to do.

### Eliminating "unsafe" interfaces from Swift entirely

* Some commenters were concerned by the idea of adding more "unsafe" interfaces to the Standard Library. The proposed functions are "unsafe" by the usual Swift definition, but not moreso than existing functions such as `withUnsafePointer(to:_:)` or `Data.withUnsafeBytes(_:)`.
* As discussed previously, in order to provide a high-level "safe" interface, the language needs some amount of lower-level unsafety. `Data` cannot be implemented without `UnsafeRawPointer` (or equivalent,) nor `Array<T>` without `UnsafeMutableBufferPointer<T>`, nor `CheckedContinuation<T, E>` without `UnsafeContinuation<T, E>`.
* The need for unsafe interfaces is not limited to the Standard Library: developers working at every layer of the software stack can benefit from careful use of unsafe interfaces like the proposed functions.
* Creating a dialect of Swift that bans unsafe interfaces entirely is an interesting idea and is worth discussing in more detail, but it is beyond the scope of this proposal.

## Acknowledgments

Thank you to the Swift team for your help and patience as I learn how to write Swift proposals. And thank you to everyone who commented in the pitch thread—it was great to see your feedback and your ideas!
