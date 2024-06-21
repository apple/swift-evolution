# Conform Never to Identifiable

* Proposal: [SE-0319](0319-never-identifiable.md)
* Author: [Kyle Macomber](https://github.com/kylemacomber)
* Review Manager: [Tom Doron](https://github.com/tomerd)
* Status: **Implemented (Swift 5.5)**
* Implementation: [apple/swift#38103](https://github.com/apple/swift/pull/38103)
* Review: [Forum discussion](https://forums.swift.org/t/se-0319-never-as-identifiable/50246)
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-se-0319-never-as-identifiable/50473)

## Introduction

This proposal conforms `Never` to `Identifiable` to make it usable as a "bottom type" for generic constraints that require `Identifiable`.

## Motivation and Proposed Solution

With the acceptance of [SE-0215](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0215-conform-never-to-hashable-and-equatable.md), `Never` was deemed as being a “blessed bottom type”, but that it wouldn’t implicitly conform to all protocols—instead explicit conformance would be added where valuable.

The conformance of `Never` to `Equatable` and `Hashable` in SE-0215 was motivated by examples like using `Never` as a generic constraint in types like `Result` and in enumerations. These same use cases motivate the conformance of `Never` to `Identifiable`, which is pervasive in commonly used frameworks like SwiftUI.

For example, the new `TableRowContent` protocol in SwiftUI follows a "recursive type pattern" and has the need for a primitive bottom type with an `Identifiable` assocated type:

```swift
extension Never: TableRowContent {
  public typealias TableRowBody /* conforms to TableRowContent */ = Never
  public typealias TableRowValue /* conforms to Identifiable */ = Never
}
```

## Detailed design

```swift
@available(macOS 12.0, iOS 15.0, watchOS 8.0, tvOS 15.0, *)
extension Never: Identifiable {
  public var id: Never {
    switch self {}
  }
}
```

## Source compatibility

If another module has already conformed `Never` to `Identifiable`, the compiler will emit a warning:

```
MyFile.swift: warning: conformance of 'Never' to protocol 'Identifiable' was already stated in the type's module 'Swift'
extension Never: Identifiable { 
                 ^
MyFile.swift: note: property 'id' will not be used to satisfy the conformance to 'Identifiable'
    var id: Never {
        ^
```

As the warning notes, the new conformance will be used to satisfy the protocol requirement. This difference shouldn't present an observable difference given that an instance of `Never` cannot be constructed.

## Effect on ABI stability

This change is additive.

## Effect on API resilience

As this change adds new ABI, it cannot be removed in the future without breaking the ABI.

## Alternatives considered

#### Add additional "missing" conformances to `Never` (e.g. `CaseIterable`) and other common types

A more thorough audit of "missing" conformances is called for. With this proposal we chose the narrowest possible scope in order to prioritize the addition of important functionality in a timely manner.
