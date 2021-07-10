# Circular Buffer

* Proposal: [SE-CircularBuffer](NNNN-CircularBuffer.md)
* Authors: [Maksim Kita](https://github.com/kitaisreal)
* Review Manager: TBD
* Status: **Awaiting implementation**
* Implementation: [apple/swift#30242](https://github.com/apple/swift/pull/30242), [apple/swift-evolution-staging#1](https://github.com/apple/swift-evolution-staging/pull/1)
* Bugs: [SR-8190](https://bugs.swift.org/browse/SR-8190), [SR-6868](https://bugs.swift.org/browse/SR-6868), [SR-11956](https://bugs.swift.org/browse/SR-11956)

## Introduction

Introduce the `CircularBuffer` collection type conforming to the `RandomAccessCollection`, 
`RangeReplaceableCollection` and `MutableCollection` protocols. With random element 
access and support for constant back and front element insertion and deletion.

https://en.wikipedia.org/wiki/Circular_buffer

## Motivation

Swift currently does not have collection with both element random access
and constant O(1) elements back and front insertion and deletion. A good
usage examples for such collection are queue, deque, fixed length buffers. 
Such abstractions cannot effectively be build on Array because of O(n) first element deletion.

## Proposed solution

This proposal adds a `CircularBuffer` type to standard library.

## Detailed design

`CircularBuffer` is generic collection with random element access and
constant O(1) elements back and front insertion and deletion.

```swift
public struct CircularBuffer<Element> : RandomAccessCollection, RangeReplaceableCollection {

    public init(capacity: Int)

    /// A boolean value indicating that CircularBuffer is full.
    public var isFull: Bool { get }
    
    /// A total number of elements that CircularBuffer can contain.
    public var capacity: Int { get }

    /// Resizes CircularBuffer capacity.
    public mutating func resize(newCapacity: Int)

    /// Pushes element to the back
    public mutating func pushBack(_ newElement: Element)

    /// Pushes elements to the back
    public mutating func pushBack<S>(contentsOf newElements: S) where Element == S.Element, S : Sequence

    /// Pushes element to the front
    public mutating func pushFront(_ newElement: Element)

    /// Pushed elements to the front
    public mutating func pushFront<S>(contentsOf newElements: S) where Element == S.Element, S : Sequence

    /// Removes and returns element from the back
    public mutating func popBack() -> Element

    /// Removes and returns element from the front
    public mutating func popFront() -> Element
}
```

`CircularBuffer` conforms to `RandomAccessCollection`, `RangeReplaceableCollection`, `CustomStringConvertible`, `CustomDebugStringConvertible`,
`ExpressibleByArrayLiteral` and also to `Equatable` and `Hashable` when its `Element` type conforms to it.

#### Capacity

Capacity is essential part of RingBuffer, not just for holding elements but also for 
implementing behaviour of rewriting old data when buffer is full. These methods:
```swift
public mutating func pushBack(_ newElement: Element)

public mutating func pushBack<S>(contentsOf newElements: S) where Element == S.Element, S : Sequence

public mutating func pushFront(_ newElement: Element)

public mutating func pushFront<S>(contentsOf newElements: S) where Element == S.Element, S : Sequence
```
does not increase capacity automatically. When CircularBuffer is full, new data will be written 
to the beginning of the buffer so old data will be overwritten.

```swift
var circularBuffer = CircularBuffer<Int>(capacity: 2)
circularBuffer.pushBack(1)
circularBuffer.pushBack(2)
circularBuffer.pushBack(3)
print(circularBuffer)
// Prints "[2, 3]"
```

Circular Buffer capacity can be increased with manual call 
to resize(newCapacity:).

Also `CircularBuffer` support `RangeReplaceableCollection` so client can 
increase capacity using methods that increase element count of 
generic `RangeReplaceableCollection` collection `append`, `append(contentsOf:)`,
`replaceSubrange(subrange:, with:)`. `CircularBuffer` will grow exponentially
using grow factor 1.5.

Example of usage
```swift
var circularBuffer = CircularBuffer<Int>(capacity: 2)
circularBuffer.pushBack(1)
circularBuffer.pushFront(2)
print(circularBuffer)
// Prints "[2, 1]"
// Now buffer isFull so next writes will overwrite data at beggining

circularBuffer.pushFront(4)
print(circularBuffer)
// Prints "[4, 2]"

circularBuffer.pushBack(3)
print(circularBuffer)
// Prints "[2, 3]"

print(circularBuffer.popFront())
// Prints "2"

print(circularBuffer)
// Prints "[3]"

circularBuffer.popBack()
print(circularBuffer)
// Prints "[]"
```

## Source compatibility

N/A

## Effect on ABI stability

This proposal only makes additive changes to the existing ABI.

## Effect on API resilience

All the proposed additions are versioned.

## Alternatives considered

#### Naming

There can be a lot of possible names for this kind of structures like circular buffer, 
circular queue, cyclic buffer or ring buffer. The name `CircularBuffer` is one that is used
 in most of the sources so it was picked.
