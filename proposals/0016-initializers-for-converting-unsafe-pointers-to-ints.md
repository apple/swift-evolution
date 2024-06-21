# Add initializers to Int and UInt to convert from UnsafePointer and UnsafeMutablePointer

* Proposal: [SE-0016](0016-initializers-for-converting-unsafe-pointers-to-ints.md)
* Author: [Michael Buckley](https://github.com/MichaelBuckley)
* Review Manager: [Chris Lattner](https://github.com/lattner)
* Status: **Implemented (Swift 3.0)**
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-se-0016-adding-initializers-to-int-and-uint-to-convert-from-unsafepointer-and-unsafemutablepointer/2005)
* Bug: [SR-1115](https://bugs.swift.org/browse/SR-1115)
* Previous Revision: [1](https://github.com/swiftlang/swift-evolution/blob/ae2d7c24fff7cbdff754d9a4339e4fb02df5c690/proposals/0016-initializers-for-converting-unsafe-pointers-to-ints.md)

## Introduction

Just as users can create `Unsafe[Mutable]Pointer`s from `Int`s and `UInt`s, they
should be able to create Ints and UInts from `Unsafe[Mutable]Pointer`s. This will
allow users to call C functions with `intptr_t` and `uintptr_t` parameters, and will
allow users to perform more advanced pointer arithmetic than is allowed by
`UnsafePointer`s.

[Swift Evolution Discussion](https://forums.swift.org/t/proposal-add-initializers-for-converting-unsafepointers-to-int-and-unit/331), [Review](https://forums.swift.org/t/review-se-0016-adding-initializers-to-int-and-uint-to-convert-from-unsafepointer-and-unsafemutablepointer/1899)

## Motivation

Swift currently lacks the ability to perform many complex operations on
pointers, such as checking pointer alignment, tagging pointers, or XORing
pointers (for working with XOR linked lists, for example). As a systems
programming language, Swift ought to be able to solve these problems natively
and concisely.

Additionally, since some C functions take `intptr_t` and `uintptr_t` parameters,
Swift currently has no ability to call these functions directly. Users must wrap
calls to these functions in C code.

## Proposed solution

Initializers will be added to `Int` and `UInt` to convert from `UnsafePointer`,
`UnsafeMutablePointer`, and `OpaquePointer`.

Currently, the only workaround which can solve these problems is to write any
code that requires pointer arithmetic in C. Writing this code in Swift will be
no safer than it is in C, as this is a fundamentally unsafe operation. However,
it will be cleaner in that users will not be forced to write C code.

## Detailed design

The initializers will be implemented using the built-in `ptrtoint_Word` function.

```swift
extension UInt {
  init<T>(bitPattern: UnsafePointer<T>) {
    self = UInt(Builtin.ptrtoint_Word(bitPattern._rawValue))
  }

  init<T>(bitPattern: UnsafeMutablePointer<T>) {
    self = UInt(Builtin.ptrtoint_Word(bitPattern._rawValue))
  }

  init(bitPattern: OpaquePointer) {
    self = UInt(Builtin.ptrtoint_Word(bitPattern._rawValue))
  }
}

extension Int {
  init<T>(bitPattern: UnsafePointer<T>) {
    self = Int(Builtin.ptrtoint_Word(bitPattern._rawValue))
  }

  init<T>(bitPattern: UnsafeMutablePointer<T>) {
    self = Int(Builtin.ptrtoint_Word(bitPattern._rawValue))
  }

  init(bitPattern: OpaquePointer) {
    self = Int(Builtin.ptrtoint_Word(bitPattern._rawValue))
  }
}
```

As an example, these initializers will allow the user to get the next address of
an XOR linked list in Swift.

```swift
struct XORLinkedList<T> {
  let address: UnsafePointer<T>

  ...

  func successor(_ predecessor: XORLinkedList<T>) -> XORLinkedList<T> {
    let next = UInt(bitPattern: address) ^ UInt(bitPattern: predecessor.address)
    return XorLinkedList(UnsafePointer<T>(bitPattern: next))
  }
}
```

## Impact on existing code

There is no impact on existing code.

## Alternatives considered

Three alternatives were considered.

The first alternative was to add an `intValue` function to `Unsafe[Mutable]Pointer`.
This alternative was rejected because it is preferred that type conversions be
implemented as initializers where possible.

The next alternative was to add functions to `Unsafe[Mutable]Pointer` which
covered the identified pointer arithmetic cases. This alternative was rejected
because it either would have required us to imagine every use-case of pointer
arithmetic and write functions for them, which is an impossible task, or it
would have required adding a full suite of arithmetic and bitwise operators to
`Unsafe[Mutable]Pointer`. Because some of these operations are defined only on
signed integers, and others on unsigned, it would have required splitting
`Unsafe[Mutable]Pointer` into signed and unsigned variants, which would have
complicated things for users who did not need to do pointer arithmetic.
Additionally, the implementations of these operations would have probably
converted the pointers to integers, perform a single operation, and then convert
them back. When chaining operations, this would create a lot of unnecessary
conversions.

The last alternative was to forgo these initializers and force users to write
all their complicated pointer code in C. This alternative was rejected because
it makes Swift less useful as a systems programming language.

## Changes from revision 1

- The proposal was amended post-acceptance to include `OpaquePointer`.
  Originally it only included `UnsafePointer` and `UnsafeMutablePointer`.
