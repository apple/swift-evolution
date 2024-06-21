# Introduce `compactMapValues` to Dictionary

* Proposal: [SE-0218](0218-introduce-compact-map-values.md)
* Author: [Daiki Matsudate](https://github.com/d-date)
* Review Manager: [Ben Cohen](https://github.com/airspeedswift)
* Status: **Implemented (Swift 5.0)**
* Implementation: [apple/swift#15017](https://github.com/apple/swift/pull/15017)
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-se-0218-introduce-compactmapvalues-to-dictionary/14448)

## Introduction

This proposal adds a combined filter/map operation to `Dictionary`, as a companion to the `mapValues` and filter methods introduced by [SE-0165](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0165-dict.md). The new compactMapValues operation corresponds to compactMap on Sequence.

- Swift forums pitch: [Add compactMapValues to Dictionary](https://forums.swift.org/t/add-compactmapvalues-to-dictionary/8741)

## Motivation

Swift 4 introduced two new `Dictionary` operations: the new method `mapValues` and a new version of `filter`. They correspond to the `Sequence` methods `map` and `filter`, respectively, but they operate on `Dictionary` values and return dictionaries rather than arrays.

However, [SE-0165](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0165-dict.md) left a gap in the API: it did not introduce a `Dictionary`-specific version of `compactMap`. We sometimes need to transform and filter values of a `Dictionary` at the same time, and `Dictionary` does not currently provide an operation that directly supports this.

For example, consider the task of filtering out `nil` values from a `Dictionary` of optionals:

```swift
let d: [String: String?] = ["a": "1", "b": nil, "c": "3"]
let r1 = d.filter { $0.value != nil }.mapValues { $0! }
let r2 = d.reduce(into: [String: String]()) { (result, item) in result[item.key] = item.value }
// r1 == r2 == ["a": "1", "c": "3"]
```

Or try running a failable conversion on dictionary values:

```swift
let d: [String: String] = ["a": "1", "b": "2", "c": "three"]
let r1 = d.mapValues(Int.init).filter { $0.value != nil }.mapValues { $0! }
let r2 = d.reduce(into: [String: Int]()) { (result, item) in result[item.key] = Int(item.value) }
// r == ["a": 1, "b": 2]
```

While `mapValues` and `filter` can be combined to solve this tasks, the solution needs multiple passes on the input dictionary, which is not particularly efficient. `reduce(into:)` provides a more efficient solution, but it is rather tricky to get right, and it obscures the intended meaning of the code with implementation details.

It seems worth adding an extra extension method to `Dictionary` for this operation; its obvious name is `compactMapValues(_:)`, combining the precedents set by `compactMap` and `mapValues`.

```swift
let r3 = d.compactMapValues(Int.init)
```

## Proposed solution

Add the following to `Dictionary`:

```swift
let d: [String: String?] = ["a": "1", "b": nil, "c": "3"]
let r4 = d.compactMapValues({$0})
// r4 == ["a": "1", "c": "3"]
```

Or,

```swift
let d: [String: String] = ["a": "1", "b": "2", "c": "three"]
let r5 = d.compactMapValues(Int.init)
// r5 == ["a": 1, "b": 2]
```

## Detailed design

Add the following to `Dictionary`:

```swift
extension Dictionary {
    public func compactMapValues<T>(_ transform: (Value) throws -> T?) rethrows -> [Key: T] {
        return try self.reduce(into: [Key: T](), { (result, x) in
            if let value = try transform(x.value) {
                result[x.key] = value
            }
        })
    }
}
```

## Source compatibility

This change is purely additive so has no source compatibility consequences.

## Effect on ABI stability

This change is purely additive so has no ABI stability consequences.

## Effect on API resilience

This change is purely additive so has no API resilience consequences.

## Alternatives considered

We can simply omit this method from the standard library -- however, we already have `mapValues` and `filter`, and it seems reasonable to fill the API hole left between them with a standard extension.
