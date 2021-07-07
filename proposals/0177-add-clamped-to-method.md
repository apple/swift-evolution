# Add `clamped(to:)` methods to `Comparable` and `Strideable`

* Proposal: [SE-0177](0177-add-clamped-to-method.md)
* Author: [Nicholas Maccharoli](https://github.com/Nirma)
* Review Manager: TBD
* Status: **Returned for revision**
* Status: **Awaiting review**
* Implementation: [apple/swift#32268](https://github.com/apple/swift/pull/32268)
* Previous Revision: [1](https://github.com/apple/swift-evolution/blob/8d9f51f060b8c772f7adabee486112a97569d2b3/proposals/0177-add-clamped-to-method.md)

## Introduction

This proposal aims to add functionality to the standard library for clamping a value to a provided range.
The proposed function would allow the user to specify a range to clamp a value to where if the value fell within the range, the value would be returned as is, if the value being clamped exceeded the upper or lower bound then the upper or lower bound would be returned respectively.

Swift-evolution thread: [Add a `clamp` function to Algorithm.swift](https://forums.swift.org/t/add-a-clamp-function-to-algorithm-swift/5405)

## Motivation

There have been quite a few times in my professional and personal programming life where I reached for a function to limit a value to a given range and was disappointed that it was not part of the standard library.

There already exists an extension to `CountableRange` in the standard library  implementing `clamped(to:)` that will limit the calling range to that of the provided range, so having the same functionality but just for types that conform to the `Comparable` protocol would be conceptually consistent.

Having functionality like `clamped(to:)` added to `Comparable` as a protocol extension would benefit users of the Swift language trying to express and ensure that a value is kept within bounds, perhaps one example of this coming in handy would be to limit the result of some calculation between two acceptable numerical limits, say the bounds of a normalized coordinate system.

## Proposed solution

The proposed solution is to add a general purpose `clamped(to:)` method to the Swift Standard Library as an extension to `Comparable` handling `ClosedRange` (`A...B`), `PartialRangeFrom` (`A...`) and `PartialRangeThrough` (`...B`).

The function would return a value within the bounds of the provided range, if the value `clamped(to:)` is being called on falls within the provided range then the original value would be returned.
If the value outside the bounds of the provided range then the respective lower or upper bound of the range would be returned.

Given a `clamped(to:)` function it could be called in the following ways, yielding the results in the adjacent comments:

```swift

42.clamped(to: 0...50) // 42
42.clamped(to: 200...) // 200
42.clamped(to: ...20) // 20
```

## Detailed design

### Overview

The implementation of `clamped(to:)` that is being proposed is composed of a protocol extension on `Comparable` accepting ranges of the types `ClosedRange`, `PartialRangeFrom` and `PartialRangeThrough`.

```swift
extension Comparable {
    func clamped(to range: ClosedRange<Self>) -> Self {
        max(range.lowerBound, min(self, range.upperBound))
    }

    func clamped(to range: PartialRangeFrom<Self>) -> Self {
        max(range.lowerBound, self)
    }

    func clamped(to range: PartialRangeThrough<Self>) -> Self {
        min(self, range.upperBound)
    }
}
```

### Behaviour of clamped(to:)
#### Value being clamped is less than lowerBound
If the value being clamped is less than the `lowerBound` the `lowerBound` will be returned.

```swift
100.clamped(to: 500...1000) // returns 500
```

#### Value being clamped is greater than upperBound
If the value being clamped is greater than the the upperBound then the `upperBound` will be returned.

```swift
9.clamped(to: 1...5) // returns 5
```

#### Value being clamped is within range
If the value being clamped is within the range the value is returned as is.

```swift
9.clamped(to: 1...10) // returns 9
```

#### Case where value is already within range
If the value being clamped already falls within the provided range then `self` will be returned.
To demonstrate that this is the case lets look at the `min` and `max` free functions that are used in the implementation of `clamped(to:)` .

in the case of `clamped(to range: ClosedRange<Self>) -> Self`

```swift
extension Comparable {
    func clamped(to range: ClosedRange<Self>) -> Self {
        max(range.lowerBound, min(self, range.upperBound))
    }
```

we can see that `max` is being called with the lowerBound passed in as the leftmost parameter followed by `min(self, range.upperBound)`.

Looking at the definition of `max` we can see as documented `max(a, b)` will return `b` if both values passed in are equal.
Here `min(self, range.upperBound)` is being passed in as the last parameter to max.
Looking at the definition of `min` will show that `min(a, b)` where `a == b` is `true` will return `a` unmodified, here we are passing in `self` in that position bringing us to the conclusion that `a.clamped(to: b...c)` will return self unmodified when `a` is equal to the upper bound, equal to the lower bound or within range.


## Source compatibility

This feature is purely additive; it has no effect on source compatibility.

## Effect on ABI stability

This feature is purely additive; it has no effect on ABI stability.

## Effect on API resilience

The proposed function would become part of the API but purely additive.

## Alternatives considered

### Clamping Exceptional Values
- Calling `preconditionFailure()` when receiving exceptional values like `.nan`
- return range.lowerBound or range.upperBound
