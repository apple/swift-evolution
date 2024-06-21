# Deprecate String Index Encoded Offsets

* Proposal: [SE-0241](0241-string-index-explicit-encoding-offset.md)
* Author: [Michael Ilseman](https://github.com/milseman)
* Review Manager: [John McCall](https://github.com/rjmccall)
* Status: **Implemented (Swift 5.0)**
* Implementation: [apple/swift#22108](https://github.com/apple/swift/pull/22108)
* Review: ([review](https://forums.swift.org/t/se-0241-explicit-encoded-offsets-for-string-indices/19929)) ([acceptance](https://forums.swift.org/t/accepted-se-0241-explicit-encoded-offsets-for-string-indices/20540))

## Introduction

[SE-0180][] introduced a computed variable and initializer surrounding the concept of an `encodedOffset` for serialization purposes. Unfortunately, that approach is flawed for its intended purpose and is commonly misused in ways that Swift 5 is [more likely to expose](https://bugs.swift.org/browse/SR-9749). It is too late in the Swift 5.0 release to solve all existing problems, so we propose deprecating `encodedOffset` and introducing a targeted, semantics-preserving alternative.

## Motivation

String abstracts away details about the underlying encoding used in its storage. String.Index is opaque and represents a position within a String or Substring. This can make serializing a string alongside its indices difficult, and for that reason [SE-0180][] added a computed variable and initializer `encodedOffset` in Swift 4.0.

String was always meant to be capable of handling multiple backing encodings for its contents, and this is realized in Swift 5. [String now uses UTF-8](https://forums.swift.org/t/string-s-abi-and-utf-8/17676) for its preferred “fast” native encoding, but has a resilient fallback for strings of different encodings. Currently, we only use this fall-back for lazily-bridged Cocoa strings, which are commonly encoded as UTF-16, though it can be extended in the future thanks to resilience.

Unfortunately, [SE-0180][]’s approach of a single notion of `encodedOffset` is flawed. A string can be serialized with a choice of encodings, and the offset is therefore encoding-dependent and requires access to the contents of the string to calculate. A comment in [SE-0180][]’s example source mentioned that `encodedOffset` assumes UTF-16, which happened to be the only encoding used internally by String at the time (for offset purposes).

Furthermore, the majority of uses of `encodedOffset` in the wild are not following [SE-0180][]’s intended purpose and are sensitive to encoding changes. `encodedOffset` is frequently misused under the assumption that all Characters are comprised of a single code unit, which is error-prone and Swift 5 might surface the underlying bugs in more situations. It is also sometimes used for mapping Cocoa string indices, which happens to work in Swift 4 but might not in Swift 5, and Foundation already provides better alternatives.



## Proposed solution

We propose a targeted semantics-preserving off-ramp for uses of `encodedOffset`. Because Swift 5 may introduce a semantic difference in behavior, it is important to rush this fix into the 5.0 release so that developers can preserve existing semantics. Potential solutions to other problems, including the original intended purpose of `encodedOffset`, are highlighted in “Alternatives Considered”.


## Detailed design

```swift
extension String.Index {
  /// The UTF-16 code unit offset corresponding to this Index
  public func utf16Offset<S: StringProtocol>(in s: S) -> Int {
    return s.utf16.distance(from: s.utf16.startIndex, to: self)
  }
  /// Creates a new index at the specified UTF-16 code unit offset
  ///
  /// - Parameter offset: An offset in UTF-16 code units.
  public init<S: StringProtocol>(utf16Offset offset: Int, in s: S) {
    let (start, end) = (s.utf16.startIndex, s.utf16.endIndex)
    guard offset >= 0,
          let idx = s.utf16.index(start, offsetBy: offset, limitedBy: end)
    else {
      self = end.nextEncoded // internal method returning endIndex+1
      return
    }
    self = idx
  }
}

```

We try to match the original semantics as close as we reasonably can. If the user supplies an out-of-bounds offset to the initializer, we will form an invalid index. If the out-of-bounds offset is exactly equivalent to the count, the returned index will compare equal with `endIndex`, otherwise it will compare greater than `endIndex`.


## Source Compatibility

This deprecates existing API and provides a semantics-preserving alternative. Deprecation preserves source compatibility and strongly hints towards correct usage. But, other changes in Swift 5 introduce potential semantic drift in old code.

## Effect of ABI stability

This change is ABI-additive, but necessary due to other ABI changes in Swift 5.

## Effect on API resilience

Added APIs are all resilient and can be replaced with more efficient implementations that preserve correctness as String evolves.

## Alternatives Considered

### Do Nothing

If `encodedOffset` was only used for serialization, *and* such serialization/deserialization would record and preserve the original encoding, *and* we amend [SE-0180][]’s comment to avoid nailing it down to any given encoding, no change would be necessary. Unfortunately, there is no way to query or preserve internal encoding and there is considerable use and misuse in the wild, as mentioned in the “Uses in the Wild” disclosure section.

### Fix all the Bugs

This proposal originally introduced a set of API attempting to solve 3 problems:

1. [SE-0180][]’s `encodedOffset`, meant for serialization purposes, needs to be parameterized over the encoding in which the string will be serialized in
2. Existing uses of `encodedOffset` need a semantics-preserving off-ramp for Swift 5, which is expressed in terms of UTF-16 offsets
3. Existing misuses of `encodedOffset`, which assume all characters are a single UTF-16 code unit, need a semantics-fixing alternative


<details><summary>Details: String’s views and encodings</summary>

String has 3 views which correspond to the most popular Unicode encodings: UTF-8, UTF-16, and UTF-32 (via the Unicode scalar values). String’s default view is of Characters.

```swift
let myString = "abc\r\nいろは"
Array(myString.utf8) // UTF-8 encoded
Array(myString.utf16) // UTF-16 encoded
Array(myString.unicodeScalars.lazy.map { $0.value }) // UTF-32 encoded
Array(myString); Array(myString.indices) // Not an encoding, but provides offset-based access to `Characters`
```
</details>

#### Uses in the Wild
<details>

GitHub code search yields [nearly 1500 uses](https://github.com/search?l=Swift&q=encodedOffset&type=Code) , and nearly-none of them are for [SE-0180][]’s intended purpose. Below I present the 3 most common uses.

```swift
// Common code for these examples
let myString: String = ...
let start: String.Index = ...
let end: String.Index = ...
let utf16OffsetRange: Range<Int> = ...
let nsRange: NSRange = ...
```


#### Offset-based `Character` indexing

The most common misuse of `encodedOffset` assumes that all Characters in a String are comprised of a single code unit. This is wrong and a source of surprising bugs, even for exclusively ASCII content: `"\r\n".count == 1`.

```swift
let (i, j): (Int, Int) = ... // Something computed in terms of myString.count

// Problematic code
myString[String.Index(encodedOffset: i]..<String.Index(encodedOffset: j)]

// Semantic preserving alternative from this proposal
myString[String.Index(offset: i, within: myString)..<String.Index(offset: j, within: myString)]

// Even better alternative
let myIndices = Array(myString.indices)
let (i, j): (Int, Int) = ... // Something computed in terms of myIndices.count
myString[myIndices[i]..<myIndices[j]]
```


#### Range Mapping

Many of the uses in the wild are trying to map between `Range<String.Index>` and `NSRange`. Foundation already provides convenient initializers for this purpose already, and using them is the preferred approach:

```swift
// Problematic code
let myNSRange = NSRange(location: start.encodedOffset, length: end.encodedOffset - start.encodedOffset)
let myStrRange = String.Index(encodedOffset: nsRange.lowerBound)..<String.Index(encodedOffset: nsRange.upperBound)

// Better alternative
let myNSRange = NSRange(start..<end, in: myString)
let myStrRange = Range(nsRange, in: myString)
```


#### Naked Ints

Some uses in the wild, through no fault of their own, have an Int which represents a position in UTF-16 encoded contents and need to convert that to a `String.Index`.


```swift
// Problematic code
let strLower = String.Index(encodedOffset: utf16OffsetRange.lowerBound)
let strUpper = String.Index(encodedOffset: utf16OffsetRange.upperBound)
let subStr = myString[strLower..<strUpper]

// Semantic preserving alternative from this proposal
let strLower = String.Index(offset: utf16OffsetRange.lowerBound, within: str.utf16)
let strUpper = String.Index(offset: utf16OffsetRange.upperBound, within: str.utf16)
let subStr = myString[strLower..<strUpper]
```

</details>

#### Potential Solution

<details><summary>Original Proposed Solution</summary>

Here is a (slightly revised) version of the original proposal:

```swift
  /// The UTF-16 code unit offset corresponding to this Index
  public func offset<S: StringProtocol>(in utf16: S.UTF16View) -> Int { ... }

  /// The UTF-8 code unit offset corresponding to this Index
  public func offset<S: StringProtocol>(in utf8: S.UTF8View) -> Int { ... }

  /// The Unicode scalar offset corresponding to this Index
  public func offset<S: StringProtocol>(in scalars: S.UnicodeScalarView) -> Int { ... }

  /// The Character offset corresponding to this Index
  public func offset<S: StringProtocol>(in str: S) -> Int { ... }

  /// Creates a new index at the specified UTF-16 code unit offset
  ///
  /// - Parameter offset: An offset in UTF-16 code units.
  public init<S: StringProtocol>(offset: Int, in utf16: S.UTF16View) { ... }

  /// Creates a new index at the specified UTF-8 code unit offset
  ///
  /// - Parameter offset: An offset in UTF-8 code units.
  public init<S: StringProtocol>(offset: Int, in utf8: S.UTF8View) { ... }

  /// Creates a new index at the specified Unicode scalar offset
  ///
  /// - Parameter offset: An offset in terms of Unicode.Scalars
  public init<S: StringProtocol>(offset: Int, in scalars: S.UnicodeScalarView) { ... }

  /// Creates a new index at the specified Character offset
  ///
  /// - Parameter offset: An offset in terms of Characters
  public init<S: StringProtocol>(offset: Int, in str: S) { ... }
}
```


This gives developers:

1. The ability to choose a specific encoding for serialization, the original intended purpose.
2. The ability to fix any code that assumed fixed-encoding-width Characters by choosing the most-natural variant that just takes a String.
3. The ability to migrate their uses for Cocoa index mapping by choosing UTF-16.

However, it’s not clear this is the best approach for Swift and more design work is needed:

* Overloading only on view type makes it easy to accidentally omit a view and end up with character offsets. E.g. `String.Index(offset: myUTF16Offset, in: myUTF16String)` instead of `String.Index(offset: myUTF16Offset, in: myUTF16String.utf16)`.
* Producing new indices is usually done by the collection itself rather than parameterizing an index initializer.  This should be handled with something more ergonomic such as offset-based indexing in a future release.
* In real code in the wild, almost all created indices are immediately used to subscript the string or one of its views. This should be handled with something more ergonomic such as [offset-based subscripting](https://forums.swift.org/t/shorthand-for-offsetting-startindex-and-endindex/9397) in a future release.

</details>

#### Conclusion

It is too late in the Swift 5.0 release to design and add all of these API. Instead, we’re proposing an urgent, targeted fix for the second problem.

[SE-0180]: <https://github.com/swiftlang/swift-evolution/blob/master/proposals/0180-string-index-overhaul.md>
