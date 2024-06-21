# Regex-powered string processing algorithms

* Proposal: [SE-0357](0357-regex-string-processing-algorithms.md)
* Authors: [Tina Liu](https://github.com/itingliu), [Michael Ilseman](https://github.com/milseman), [Nate Cook](https://github.com/natecook1000), [Tim Vermeulen](https://github.com/timvermeulen)
* Review Manager: [Ben Cohen](https://github.com/airspeedswift)
* Status: **Implemented (Swift 5.7)**
* Implementation: [apple/swift-experimental-string-processing](https://github.com/apple/swift-experimental-string-processing/)
    * Available in nightly toolchain snapshots with `import _StringProcessing`
* Review: ([pitch](https://forums.swift.org/t/pitch-regex-powered-string-processing-algorithms/55969))
         ([review](https://forums.swift.org/t/se-0357-regex-string-processing-algorithms/57225))
     ([acceptance](https://forums.swift.org/t/accepted-with-modifications-se-0357-regex-string-processing-algorithms/58706))
* Previous Revision: [1](https://github.com/swiftlang/swift-evolution/blob/7741017763f528dfbdfa54c6d11f559918ab53e4/proposals/0357-regex-string-processing-algorithms.md)

## Introduction

The Swift standard library's string processing algorithms are underpowered compared to other popular programming and scripting languages. Some of these omissions can be found in `NSString`, but these fundamental algorithms should have a place in the standard library.

We propose:

1. New regex-powered algorithms over strings, bringing the standard library up to parity with scripting languages
2. Generic `Collection` equivalents of these algorithms in terms of subsequences
3. `protocol CustomConsumingRegexComponent`, which allows 3rd party libraries to provide their industrial-strength parsers as intermixable components of regexes

This proposal is part of a larger [regex-powered string processing initiative](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0350-regex-type-overview.md), the status of each proposal is tracked [here](https://github.com/apple/swift-experimental-string-processing/blob/main/Documentation/Evolution/ProposalOverview.md). Further discussion of regex specifics is out of scope of this proposal and better discussed in their relevant reviews.

## Motivation

A number of common string processing APIs are missing from the Swift standard library. While most of the desired functionalities can be accomplished through a series of API calls, every gap adds a burden to developers doing frequent or complex string processing. For example, here's one approach to find the number of occurrences of a substring ("banana") within a string:

```swift
let str = "A banana a day keeps the doctor away. I love bananas; banana are my favorite fruit."

var idx = str.startIndex
var ranges = [Range<String.Index>]()
while let r = str.range(of: "banana", options: [], range: idx..<str.endIndex) {
    if idx != str.endIndex {
        idx = str.index(after: r.lowerBound)
    }
    ranges.append(r)
}

print(ranges.count)
```

While in Python this is as simple as

```python
str = "A banana a day keeps the doctor away. I love bananas; banana are my favorite fruit."
print(str.count("banana"))
```

We propose adding string processing algorithms so common tasks as such can be achieved as straightforwardly.

<details>
<summary> Comparison of how Swift's APIs stack up with Python's. </summary>

Note: Only a subset of Python's string processing API are included in this table for the following reasons:

- Functions to query if all characters in the string are of a specified category, such as `isalnum()` and `isalpha()`, are omitted. These are achievable in Swift by passing in the corresponding character set to `allSatisfy(_:)`, so they're omitted in this table for simplicity.
- String formatting functions such as `center(length, character)` and `ljust(width, fillchar)` are also excluded here as this proposal focuses on matching and searching functionalities.

##### Search and replace

|Python |Swift  |
|---    |---    |
| `count(sub, start, end)` |  |
| `find(sub, start, end)`, `index(sub, start, end)` | `firstIndex(where:)` |
| `rfind(sub, start, end)`, `rindex(sub, start, end)` | `lastIndex(where:)` |
| `expandtabs(tabsize)`, `replace(old, new, count)` | `Foundation.replacingOccurrences(of:with:)` |
| `maketrans(x, y, z)` + `translate(table)` |

##### Prefix and suffix matching

|Python |Swift  |
|---    |---    |
| `startswith(prefix, start, end)` | `starts(with:)` or `hasPrefix(:)`|
| `endswith(suffix, start, end)` | `hasSuffix(:)` |
| `removeprefix(prefix)` | Test if string has prefix with `hasPrefix(:)`, then drop the prefix with `dropFirst(:)`|
| `removesuffix(suffix)` | Test if string has suffix with `hasSuffix(:)`, then drop the suffix with `dropLast(:)` |

##### Strip / trim

|Python |Swift  |
|---    |---    |
| `strip([chars])`| `Foundation.trimmingCharacters(in:)` |
| `lstrip([chars])` | `drop(while:)` |
| `rstrip([chars])` | Test character equality, then `dropLast()` iteratively |

##### Split

|Python |Swift  |
|---    |---    |
| `partition(sep)` | `Foundation.components(separatedBy:)` |
| `rpartition(sep)` |  |
| `split(sep, maxsplit)` | `split(separator:maxSplits:...)` |
| `splitlines(keepends)` | `split(separator:maxSplits:...)` |
| `rsplit(sep, maxsplit)` |  |

</details>



### Complex string processing

Even with the API additions, more complex string processing quickly becomes unwieldy. String processing in the modern world involves dealing with localization, standards-conforming validation, and other concerns for which a dedicated parser is required.

Consider parsing the date field `"Date: Wed, 16 Feb 2022 23:53:19 GMT"` in an HTTP header as a `Date` type. The naive approach is to search for a substring that looks like a date string (`16 Feb 2022`), and attempt to post-process it as a `Date` with a date parser:

```swift
let regex = Regex {
    Capture {
        OneOrMore(.digit)
        " "
        OneOrMore(.word)
        " "
        OneOrMore(.digit)
    }
}

let dateParser = Date.ParseStrategy(format: "\(day: .twoDigits) \(month: .abbreviated) \(year: .padded(4))"
if let dateMatch = header.firstMatch(of: regex)?.0 {
    let date = try? Date(dateMatch, strategy: dateParser)
}
```

This requires writing a simplistic pre-parser before invoking the real parser. The pre-parser will suffer from being out-of-sync and less featureful than what the real parser can do.

Or consider parsing a bank statement to record all the monetary values in the last column:

```swift
let statement = """
CREDIT    04/06/2020    Paypal transfer    $4.99
CREDIT    04/03/2020    Payroll            $69.73
DEBIT     04/02/2020    ACH transfer       ($38.25)
DEBIT     03/24/2020    IRX tax payment    ($52,249.98)
"""
```

Parsing a currency string such as `$3,020.85` with regex is also tricky, as it can contain localized and currency symbols in addition to accounting conventions. This is why Foundation provides industrial-strength parsers for localized strings.


## Proposed solution

### Complex string processing

We propose a `CustomConsumingRegexComponent` protocol which allows types from outside the standard library participate in regex builders and `RegexComponent` algorithms. This allows types, such as `Date.ParseStrategy` and `FloatingPointFormatStyle.Currency`, to be used directly within a regex:

```swift
let dateRegex = Regex {
    Capture(dateParser)
}

let date: Date = header.firstMatch(of: dateRegex).map(\.result.1)

let currencyRegex = Regex {
    Capture(.localizedCurrency(code: "USD").sign(strategy: .accounting))
}

let amount: [Decimal] = statement.matches(of: currencyRegex).map(\.result.1)
```

### String algorithm additions

We also propose the following regex-powered algorithms as well as their generic `Collection` equivalents. See the Detailed design section for a complete list of variation and overloads .

|Function | Description |
|---    |---    |
|`contains(_:) -> Bool` | Returns whether the collection contains the given sequence or `RegexComponent` |
|`starts(with:) -> Bool` | Returns whether the collection contains the same prefix as the specified `RegexComponent` |
|`trimPrefix(_:)`| Removes the prefix if it matches the given `RegexComponent` or collection |
|`firstRange(of:) -> Range?` | Finds the range of the first occurrence of a given sequence or `RegexComponent`|
|`ranges(of:) -> some Collection<Range>` | Finds the ranges of the all occurrences of a given sequence or `RegexComponent` within the collection |
|`replace(:with:subrange:maxReplacements)`| Replaces all occurrences of the sequence matching the given `RegexComponent` or sequence with a given collection |
|`split(by:)`| Returns the longest possible subsequences of the collection around elements equal to the given separator |
|`firstMatch(of:)`| Returns the first match of the specified `RegexComponent` within the collection |
|`wholeMatch(of:)`| Matches the specified `RegexComponent` in the collection as a whole |
|`prefixMatch(of:)`| Matches the specified `RegexComponent` against the collection at the beginning |
|`matches(of:)`| Returns a collection containing all matches of the specified `RegexComponent` |

## Detailed design

### `CustomConsumingRegexComponent`

`CustomConsumingRegexComponent` inherits from `RegexComponent` and satisfies its sole requirement. Conformers can be used with all of the string algorithms generic over `RegexComponent`.

```swift
/// A protocol allowing custom types to function as regex components by
/// providing the raw functionality backing `prefixMatch`.
public protocol CustomConsumingRegexComponent: RegexComponent {
    /// Process the input string within the specified bounds, beginning at the given index, and return
    /// the end position (upper bound) of the match and the produced output.
    /// - Parameters:
    ///   - input: The string in which the match is performed.
    ///   - index: An index of `input` at which to begin matching.
    ///   - bounds: The bounds in `input` in which the match is performed.
    /// - Returns: The upper bound where the match terminates and a matched instance, or `nil` if
    ///   there isn't a match.
    func consuming(
        _ input: String,
        startingAt index: String.Index,
        in bounds: Range<String.Index>
    ) throws -> (upperBound: String.Index, output: RegexOutput)?
}
```

<details>
<summary>Example for protocol conformance</summary>

We use Foundation `FloatingPointFormatStyle<Decimal>.Currency` as an example for protocol conformance. It would implement the `match` function with `Match` being a `Decimal`. It could also add a static function `.localizedCurrency(code:)` as a member of `RegexComponent`, so it can be referred as `.localizedCurrency(code:)` in the `Regex` result builder:

```swift
extension FloatingPointFormatStyle<Decimal>.Currency : CustomConsumingRegexComponent {
    public func consuming(
        _ input: String,
        startingAt index: String.Index,
        in bounds: Range<String.Index>
    ) -> (upperBound: String.Index, match: Decimal)?
}

extension RegexComponent where Self == FloatingPointFormatStyle<Decimal>.Currency {
    public static func localizedCurrency(code: Locale.Currency) -> Self
}
```

Matching and extracting a localized currency amount, such as `"$3,020.85"`, can be done directly within a regex:

```swift
let regex = Regex {
    Capture(.localizedCurrency(code: "USD"))
}
```

</details>


### String and Collection algorithm additions

#### Contains

We propose a `contains` variant over collections that tests for subsequence membership. The second algorithm allows for specialization using e.g. the [two way search algorithm](https://en.wikipedia.org/wiki/Two-way_string-matching_algorithm).

```swift
extension Collection where Element: Equatable {
    /// Returns a Boolean value indicating whether the collection contains the
    /// given sequence.
    /// - Parameter other: A sequence to search for within this collection.
    /// - Returns: `true` if the collection contains the specified sequence,
    /// otherwise `false`.
    public func contains<C: Collection>(_ other: C) -> Bool
        where S.Element == Element
}
extension BidirectionalCollection where Element: Comparable {
    /// Returns a Boolean value indicating whether the collection contains the
    /// given sequence.
    /// - Parameter other: A sequence to search for within this collection.
    /// - Returns: `true` if the collection contains the specified sequence,
    /// otherwise `false`.
    public func contains<C: Collection>(_ other: C) -> Bool
        where S.Element == Element
}
```

We propose a regex-taking variant over string types (those that produce a `Substring` upon slicing).

```swift
extension Collection where SubSequence == Substring {
    /// Returns a Boolean value indicating whether the collection contains the
    /// given regex.
    /// - Parameter regex: A regex to search for within this collection.
    /// - Returns: `true` if the regex was found in the collection, otherwise
    /// `false`.
    public func contains(_ regex: some RegexComponent) -> Bool
}

// In RegexBuilder module
extension Collection where SubSequence == Substring {
    /// Returns a Boolean value indicating whether this collection contains a
    /// match for the regex, where the regex is created by the given closure.
    ///
    /// - Parameter content: A closure that returns a regex to search for within
    ///   this collection.
    /// - Returns: `true` if the regex returned by `content` matched anywhere in
    ///   this collection, otherwise `false`.
    public func contains(
        @RegexComponentBuilder _ content: () -> some RegexComponent
    ) -> Bool
}
```

#### Starts with

We propose a regex-taking `starts(with:)` variant for string types:

```swift
extension Collection where SubSequence == Substring {
    /// Returns a Boolean value indicating whether the initial elements of the
    /// sequence are the same as the elements in the specified regex.
    /// - Parameter regex: A regex to compare to this sequence.
    /// - Returns: `true` if the initial elements of the sequence matches the
    /// beginning of `regex`; otherwise, `false`.
    public func starts(with regex: some RegexComponent) -> Bool
}

// In RegexBuilder module
extension Collection where SubSequence == Substring {
    /// Returns a Boolean value indicating whether the initial elements of this
    /// collection are a match for the regex created by the given closure.
    ///
    /// - Parameter content: A closure that returns a regex to match at
    ///   the beginning of this collection.
    /// - Returns: `true` if the initial elements of this collection match
    ///   regex returned by `content`; otherwise, `false`.
    public func starts(
        @RegexComponentBuilder with content: () -> some RegexComponent
    ) -> Bool
}
```

#### Trim prefix

We propose generic `trimmingPrefix` and `trimPrefix` methods for collections that trim elements matching a predicate or a possible prefix sequence.

```swift
extension Collection {
    /// Returns a new collection of the same type by removing initial elements
    /// that satisfy the given predicate from the start.
    /// - Parameter predicate: A closure that takes an element of the sequence
    /// as its argument and returns a Boolean value indicating whether the
    /// element should be removed from the collection.
    /// - Returns: A collection containing the elements of the collection that are
    ///  not removed by `predicate`.
    public func trimmingPrefix(while predicate: (Element) throws -> Bool) rethrows -> SubSequence
}

extension Collection where SubSequence == Self {
    /// Removes the initial elements that satisfy the given predicate from the
    /// start of the sequence.
    /// - Parameter predicate: A closure that takes an element of the sequence
    /// as its argument and returns a Boolean value indicating whether the
    /// element should be removed from the collection.
    public mutating func trimPrefix(while predicate: (Element) throws -> Bool) rethrows
}

extension RangeReplaceableCollection {
    /// Removes the initial elements that satisfy the given predicate from the
    /// start of the sequence.
    /// - Parameter predicate: A closure that takes an element of the sequence
    /// as its argument and returns a Boolean value indicating whether the
    /// element should be removed from the collection.
    public mutating func trimPrefix(while predicate: (Element) throws -> Bool) rethrows
}

extension Collection where Element: Equatable {
    /// Returns a new collection of the same type by removing `prefix` from the
    /// start.
    /// - Parameter prefix: The collection to remove from this collection.
    /// - Returns: A collection containing the elements that does not match
    /// `prefix` from the start.
    public func trimmingPrefix<Prefix: Sequence>(_ prefix: Prefix) -> SubSequence
        where Prefix.Element == Element
}

extension Collection where SubSequence == Self, Element: Equatable {
    /// Removes the initial elements that matches `prefix` from the start.
    /// - Parameter prefix: The collection to remove from this collection.
    public mutating func trimPrefix<Prefix: Sequence>(_ prefix: Prefix)
        where Prefix.Element == Element
}

extension RangeReplaceableCollection where Element: Equatable {
    /// Removes the initial elements that matches `prefix` from the start.
    /// - Parameter prefix: The collection to remove from this collection.
    public mutating func trimPrefix<Prefix: Sequence>(_ prefix: Prefix)
        where Prefix.Element == Element
}
```

We propose regex-taking variants for string types:

```swift
extension Collection where SubSequence == Substring {
    /// Returns a new subsequence by removing the initial elements that matches
    /// the given regex.
    /// - Parameter regex: The regex to remove from this collection.
    /// - Returns: A new subsequence containing the elements of the collection
    /// that does not match `prefix` from the start.
    public func trimmingPrefix(_ regex: some RegexComponent) -> SubSequence
}

// In RegexBuilder module
extension Collection where SubSequence == Substring {
    /// Returns a subsequence of this collection by removing the elements
    /// matching the regex from the start, where the regex is created by
    /// the given closure.
    ///
    /// - Parameter content: A closure that returns the regex to search for at
    ///   the start of this collection.
    /// - Returns: A collection containing the elements after those that match
    ///   the regex returned by `content`. If the regex does not match at
    ///   the start of the collection, the entire contents of this collection
    ///   are returned.
    public func trimmingPrefix(
        @RegexComponentBuilder _ content: () -> some RegexComponent
    ) -> SubSequence
}

extension RangeReplaceableCollection where SubSequence == Substring {
    /// Removes the initial elements that matches the given regex.
    /// - Parameter regex: The regex to remove from this collection.
    public mutating func trimPrefix(_ regex: some RegexComponent)
}

// In RegexBuilder module
extension RangeReplaceableCollection where SubSequence == Substring {
    /// Removes the initial elements matching the regex from the start of
    /// this collection, if the initial elements match, using the given closure
    /// to create the regex.
    ///
    /// - Parameter content: A closure that returns the regex to search for
    ///   at the start of this collection.
    public mutating func trimPrefix(
        @RegexComponentBuilder _ content: () -> some RegexComponent
    )
}
```

#### First range

We propose a generic collection algorithm for finding the first range of a given subsequence:

```swift
extension Collection where Element: Equatable {
    /// Finds and returns the range of the first occurrence of a given sequence
    /// within the collection.
    /// - Parameter sequence: The sequence to search for.
    /// - Returns: A range in the collection of the first occurrence of `sequence`.
    /// Returns nil if `sequence` is not found.
    public func firstRange<C: Collection>(of other: C) -> Range<Index>?
        where C.Element == Element
}

extension BidirectionalCollection where Element: Comparable {
    /// Finds and returns the range of the first occurrence of a given sequence
    /// within the collection.
    /// - Parameter other: The sequence to search for.
    /// - Returns: A range in the collection of the first occurrence of `sequence`.
    /// Returns `nil` if `sequence` is not found.
    public func firstRange<C: BidirectionalCollection>(of other: C) -> Range<Index>?
        where C.Element == Element
}
```

We propose a regex-taking variant for string types.

```swift
extension Collection where SubSequence == Substring {
    /// Finds and returns the range of the first occurrence of a given regex
    /// within the collection.
    /// - Parameter regex: The regex to search for.
    /// - Returns: A range in the collection of the first occurrence of `regex`.
    /// Returns `nil` if `regex` is not found.
    public func firstRange(of regex: some RegexComponent) -> Range<Index>?
}

// In RegexBuilder module
extension Collection where SubSequence == Substring {
    /// Returns the range of the first match for the regex within this collection,
    /// where the regex is created by the given closure.
    ///
    /// - Parameter content: A closure that returns a regex to search for.
    /// - Returns: A range in the collection of the first occurrence of the first
    ///   match of if the regex returned by `content`. Returns `nil` if no match
    ///   for the regex is found.
    public func firstRange(
        @RegexComponentBuilder of content: () -> some RegexComponent
    ) -> Range<Index>?
}
```

#### Ranges

We propose a generic collection algorithm for iterating over all (non-overlapping) ranges of a given subsequence.

```swift
extension Collection where Element: Equatable {
    /// Finds and returns the ranges of the all occurrences of a given sequence
    /// within the collection.
    /// - Parameter other: The sequence to search for.
    /// - Returns: A collection of ranges of all occurrences of `other`. Returns
    ///  an empty collection if `other` is not found.
    public func ranges<C: Collection>(of other: C) -> some Collection<Range<Index>>
        where C.Element == Element
}

extension BidirectionalCollection where Element: Comparable {
    /// Finds and returns the ranges of the all occurrences of a given sequence
    /// within the collection.
    /// - Parameter other: The sequence to search for.
    /// - Returns: A collection of ranges of all occurrences of `other`. Returns
    ///  an empty collection if `other` is not found.
    public func ranges<C: Collection>(of other: C) -> some Collection<Range<Index>>
        where C.Element == Element
}
```

And of course regex-taking versions for string types:

```swift
extension Collection where SubSequence == Substring {
    /// Finds and returns the ranges of the all occurrences of a given sequence
    /// within the collection.
    /// - Parameter regex: The regex to search for.
    /// - Returns: A collection or ranges in the receiver of all occurrences of
    /// `regex`. Returns an empty collection if `regex` is not found.
    public func ranges(of regex: some RegexComponent) -> some Collection<Range<Index>>
}

// In RegexBuilder module
extension Collection where SubSequence == Substring {
    /// Returns the ranges of the all non-overlapping matches for the regex
    /// within this collection, where the regex is created by the given closure.
    ///
    /// - Parameter content: A closure that returns a regex to search for.
    /// - Returns: A collection of ranges of all matches for the regex returned by
    ///   `content`. Returns an empty collection if no match for the regex
    ///   is found.
    public func ranges(
        @RegexComponentBuilder of content: () -> some RegexComponent
    ) -> some Collection<Range<Index>>
}
```

#### Match

We propose algorithms for extracting a `Match` instance from a given regex from the start, anywhere in the middle, or over the entire `self`.

```swift
extension Collection where SubSequence == Substring {
    /// Returns the first match of the specified regex within the collection.
    /// - Parameter regex: The regex to search for.
    /// - Returns: The first match of `regex` in the collection, or `nil` if
    /// there isn't a match.
    public func firstMatch<R: RegexComponent>(of regex: R) -> Regex<R.RegexOutput>.Match?

    /// Match a regex in its entirety.
    /// - Parameter regex: The regex to match against.
    /// - Returns: The match if there is one, or `nil` if none.
    public func wholeMatch<R: RegexComponent>(of regex: R) -> Regex<R.RegexOutput>.Match?

    /// Match part of the regex, starting at the beginning.
    /// - Parameter regex: The regex to match against.
    /// - Returns: The match if there is one, or `nil` if none.
    public func prefixMatch<R: RegexComponent>(of regex: R) -> Regex<R.RegexOutput>.Match?
}

// In RegexBuilder module
extension Collection where SubSequence == Substring {
    /// Returns the first match for the regex within this collection, where
    /// the regex is created by the given closure.
    ///
    /// - Parameter content: A closure that returns the regex to search for.
    /// - Returns: The first match for the regex created by `content` in this
    ///   collection, or `nil` if no match is found.
    public func firstMatch<R: RegexComponent>(
        @RegexComponentBuilder of content: () -> R
    ) -> Regex<R.RegexOutput>.Match?

    /// Matches a regex in its entirety, where the regex is created by
    /// the given closure.
    ///
    /// - Parameter content: A closure that returns a regex to match against.
    /// - Returns: The match if there is one, or `nil` if none.
    public func wholeMatch<R: RegexComponent>(
        @RegexComponentBuilder of content: () -> R
    ) -> Regex<R.RegexOutput>.Match?

    /// Matches part of the regex, starting at the beginning, where the regex
    /// is created by the given closure.
    ///
    /// - Parameter content: A closure that returns a regex to match against.
    /// - Returns: The match if there is one, or `nil` if none.
    public func prefixMatch<R: RegexComponent>(
        @RegexComponentBuilder of content: () -> R
    ) -> Regex<R.RegexOutput>.Match?
}
```

#### Matches

We propose an algorithm for iterating over all (non-overlapping) matches of a given regex:

```swift
extension Collection where SubSequence == Substring {
    /// Returns a collection containing all matches of the specified regex.
    /// - Parameter regex: The regex to search for.
    /// - Returns: A collection of matches of `regex`.
    public func matches<R: RegexComponent>(of regex: R) -> some Collection<Regex<R.RegexOuput>.Match>
}

// In RegexBuilder module
extension Collection where SubSequence == Substring {
    /// Returns a collection containing all non-overlapping matches of
    /// the regex, created by the given closure.
    ///
    /// - Parameter content: A closure that returns the regex to search for.
    /// - Returns: A collection of matches for the regex returned by `content`.
    ///   If no matches are found, the returned collection is empty.
    public func matches<R: RegexComponent>(
        @RegexComponentBuilder of content: () -> R
    ) -> some Collection<Regex<R.RegexOutput>.Match>
}
```

#### Replace

We propose generic collection algorithms that will replace all occurences of a given subsequence:

```swift
extension RangeReplaceableCollection where Element: Equatable {
    /// Returns a new collection in which all occurrences of a target sequence
    /// are replaced by another collection.
    /// - Parameters:
    ///   - other: The sequence to replace.
    ///   - replacement: The new elements to add to the collection.
    ///   - subrange: The range in the collection in which to search for `other`.
    ///   - maxReplacements: A number specifying how many occurrences of `other`
    ///   to replace. Default is `Int.max`.
    /// - Returns: A new collection in which all occurrences of `other` in
    /// `subrange` of the collection are replaced by `replacement`.
    public func replacing<C: Collection, Replacement: Collection>(
        _ other: C,
        with replacement: Replacement,
        subrange: Range<Index>,
        maxReplacements: Int = .max
    ) -> Self where C.Element == Element, Replacement.Element == Element

    /// Returns a new collection in which all occurrences of a target sequence
    /// are replaced by another collection.
    /// - Parameters:
    ///   - other: The sequence to replace.
    ///   - replacement: The new elements to add to the collection.
    ///   - maxReplacements: A number specifying how many occurrences of `other`
    ///   to replace. Default is `Int.max`.
    /// - Returns: A new collection in which all occurrences of `other` in
    /// `subrange` of the collection are replaced by `replacement`.
    public func replacing<C: Collection, Replacement: Collection>(
        _ other: C,
        with replacement: Replacement,
        maxReplacements: Int = .max
    ) -> Self where C.Element == Element, Replacement.Element == Element

    /// Replaces all occurrences of a target sequence with a given collection
    /// - Parameters:
    ///   - other: The sequence to replace.
    ///   - replacement: The new elements to add to the collection.
    ///   - maxReplacements: A number specifying how many occurrences of `other`
    ///   to replace. Default is `Int.max`.
    public mutating func replace<C: Collection, Replacement: Collection>(
        _ other: C,
        with replacement: Replacement,
        maxReplacements: Int = .max
    ) where C.Element == Element, Replacement.Element == Element
}
extension RangeReplaceableCollection where Self: BidirectionalCollection, Element: Comparable {
    /// Returns a new collection in which all occurrences of a target sequence
    /// are replaced by another collection.
    /// - Parameters:
    ///   - other: The sequence to replace.
    ///   - replacement: The new elements to add to the collection.
    ///   - subrange: The range in the collection in which to search for `other`.
    ///   - maxReplacements: A number specifying how many occurrences of `other`
    ///   to replace. Default is `Int.max`.
    /// - Returns: A new collection in which all occurrences of `other` in
    /// `subrange` of the collection are replaced by `replacement`.
    public func replacing<C: Collection, Replacement: Collection>(
        _ other: C,
        with replacement: Replacement,
        subrange: Range<Index>,
        maxReplacements: Int = .max
    ) -> Self where C.Element == Element, Replacement.Element == Element

    /// Returns a new collection in which all occurrences of a target sequence
    /// are replaced by another collection.
    /// - Parameters:
    ///   - other: The sequence to replace.
    ///   - replacement: The new elements to add to the collection.
    ///   - maxReplacements: A number specifying how many occurrences of `other`
    ///   to replace. Default is `Int.max`.
    /// - Returns: A new collection in which all occurrences of `other` in
    /// `subrange` of the collection are replaced by `replacement`.
    public func replacing<C: Collection, Replacement: Collection>(
        _ other: C,
        with replacement: Replacement,
        maxReplacements: Int = .max
    ) -> Self where C.Element == Element, Replacement.Element == Element

    /// Replaces all occurrences of a target sequence with a given collection
    /// - Parameters:
    ///   - other: The sequence to replace.
    ///   - replacement: The new elements to add to the collection.
    ///   - maxReplacements: A number specifying how many occurrences of `other`
    ///   to replace. Default is `Int.max`.
    public mutating func replace<C: Collection, Replacement: Collection>(
        _ other: C,
        with replacement: Replacement,
        maxReplacements: Int = .max
    ) where C.Element == Element, Replacement.Element == Element
}
```

We propose regex-taking variants for string types as well as variants that take a closure which will generate the replacement portion from a regex match (e.g. by reading captures).

```swift
extension RangeReplaceableCollection where SubSequence == Substring {
    /// Returns a new collection in which all occurrences of a sequence matching
    /// the given regex are replaced by another collection.
    /// - Parameters:
    ///   - regex: A regex describing the sequence to replace.
    ///   - replacement: The new elements to add to the collection.
    ///   - subrange: The range in the collection in which to search for `regex`.
    ///   - maxReplacements: A number specifying how many occurrences of the
    ///   sequence matching `regex` to replace. Default is `Int.max`.
    /// - Returns: A new collection in which all occurrences of subsequence
    /// matching `regex` in `subrange` are replaced by `replacement`.
    public func replacing<Replacement: Collection>(
        _ r: some RegexComponent,
        with replacement: Replacement,
        subrange: Range<Index>,
        maxReplacements: Int = .max
    ) -> Self where Replacement.Element == Element

    /// Returns a new collection in which all occurrences of a sequence matching
    /// the given regex are replaced by another collection.
    /// - Parameters:
    ///   - regex: A regex describing the sequence to replace.
    ///   - replacement: The new elements to add to the collection.
    ///   - maxReplacements: A number specifying how many occurrences of the
    ///   sequence matching `regex` to replace. Default is `Int.max`.
    /// - Returns: A new collection in which all occurrences of subsequence
    /// matching `regex` are replaced by `replacement`.
    public func replacing<Replacement: Collection>(
        _ r: some RegexComponent,
        with replacement: Replacement,
        maxReplacements: Int = .max
    ) -> Self where Replacement.Element == Element

    /// Replaces all occurrences of the sequence matching the given regex with
    /// a given collection.
    /// - Parameters:
    ///   - regex: A regex describing the sequence to replace.
    ///   - replacement: The new elements to add to the collection.
    ///   - maxReplacements: A number specifying how many occurrences of the
    ///   sequence matching `regex` to replace. Default is `Int.max`.
    public mutating func replace<Replacement: Collection>(
        _ r: some RegexComponent,
        with replacement: Replacement,
        maxReplacements: Int = .max
    ) where Replacement.Element == Element

    /// Returns a new collection in which all occurrences of a sequence matching
    /// the given regex are replaced by another regex match.
    /// - Parameters:
    ///   - regex: A regex describing the sequence to replace.
    ///   - subrange: The range in the collection in which to search for `regex`.
    ///   - maxReplacements: A number specifying how many occurrences of the
    ///   sequence matching `regex` to replace. Default is `Int.max`.
    ///   - replacement: A closure that receives the full match information,
    ///   including captures, and returns a replacement collection.
    /// - Returns: A new collection in which all occurrences of subsequence
    /// matching `regex` are replaced by `replacement`.
    public func replacing<R: RegexComponent, Replacement: Collection>(
        _ regex: R,
        subrange: Range<Index>,
        maxReplacements: Int = .max,
        with replacement: (Regex<R.RegexOutput>.Match) throws -> Replacement
    ) rethrows -> Self where Replacement.Element == Element

    /// Returns a new collection in which all occurrences of a sequence matching
    /// the given regex are replaced by another collection.
    /// - Parameters:
    ///   - regex: A regex describing the sequence to replace.
    ///   - maxReplacements: A number specifying how many occurrences of the
    ///   sequence matching `regex` to replace. Default is `Int.max`.
    ///   - replacement: A closure that receives the full match information,
    ///   including captures, and returns a replacement collection.
    /// - Returns: A new collection in which all occurrences of subsequence
    /// matching `regex` are replaced by `replacement`.
    public func replacing<R: RegexComponent, Replacement: Collection>(
        _ regex: R,
        maxReplacements: Int = .max,
        with replacement: (Regex<R.RegexOuput>.Match) throws -> Replacement
    ) rethrows -> Self where Replacement.Element == Element

    /// Replaces all occurrences of the sequence matching the given regex with
    /// a given collection.
    /// - Parameters:
    ///   - regex: A regex describing the sequence to replace.
    ///   - maxReplacements: A number specifying how many occurrences of the
    ///   sequence matching `regex` to replace. Default is `Int.max`.
    ///   - replacement: A closure that receives the full match information,
    ///   including captures, and returns a replacement collection.
    public mutating func replace<R: RegexComponent, Replacement: Collection>(
        _ regex: R,
        maxReplacements: Int = .max,
        with replacement: (Regex<R.RegexOutput>.Match) throws -> Replacement
    ) rethrows where Replacement.Element == Element
}

// In RegexBuilder module
extension Collection where SubSequence == Substring {
    /// Returns a new collection in which all matches for the regex
    /// are replaced, using the given closure to create the regex.
    ///
    /// - Parameters:
    ///   - replacement: The new elements to add to the collection in place of
    ///     each match for the regex, using `content` to create the regex.
    ///   - subrange: The range in the collection in which to search for
    ///     the regex.
    ///   - maxReplacements: A number specifying how many occurrences of
    ///     the regex to replace.
    ///   - content: A closure that returns the collection to search for
    ///     and replace.
    /// - Returns: A new collection in which all matches for regex in `subrange`
    ///   are replaced by `replacement`, using `content` to create the regex.
    public func replacing<Replacement: Collection>(
        with replacement: Replacement,
        subrange: Range<Index>,
        maxReplacements: Int = .max,
        @RegexComponentBuilder content: () -> some RegexComponent
    ) -> Self where Replacement.Element == Element

    /// Returns a new collection in which all matches for the regex
    /// are replaced, using the given closure to create the regex.
    ///
    /// - Parameters:
    ///   - replacement: The new elements to add to the collection in place of
    ///     each match for the regex, using `content` to create the regex.
    ///   - maxReplacements: A number specifying how many occurrences of regex
    ///     to replace.
    ///   - content: A closure that returns the collection to search for
    ///     and replace.
    /// - Returns: A new collection in which all matches for regex in `subrange`
    ///   are replaced by `replacement`, using `content` to create the regex.
    public func replacing<Replacement: Collection>(
        with replacement: Replacement,
        maxReplacements: Int = .max,
        @RegexComponentBuilder content: () -> some RegexComponent
    ) -> Self where Replacement.Element == Element

    /// Replaces all matches for the regex in this collection, using the given
    /// closure to create the regex.
    ///
    /// - Parameters:
    ///   - replacement: The new elements to add to the collection in place of
    ///     each match for the regex, using `content` to create the regex.
    ///   - maxReplacements: A number specifying how many occurrences of
    ///     the regex to replace.
    ///   - content: A closure that returns the collection to search for
    ///     and replace.
    public mutating func replace<Replacement: Collection>(
        with replacement: Replacement,
        maxReplacements: Int = .max,
        @RegexComponentBuilder content: () -> some RegexComponent
    ) where Replacement.Element == Element

    /// Returns a new collection in which all matches for the regex
    /// are replaced, using the given closures to create the replacement
    /// and the regex.
    ///
    /// - Parameters:
    ///   - subrange: The range in the collection in which to search for the
    ///     regex, using `content` to create the regex.
    ///   - maxReplacements: A number specifying how many occurrences of
    ///     the regex to replace.
    ///   - content: A closure that returns the collection to search for
    ///     and replace.
    ///   - replacement: A closure that receives the full match information,
    ///     including captures, and returns a replacement collection.
    /// - Returns: A new collection in which all matches for regex in `subrange`
    ///   are replaced by the result of calling `replacement`, where regex
    ///   is the result of calling `content`.
    public func replacing<R: RegexComponent, Replacement: Collection>(
        subrange: Range<Index>,
        maxReplacements: Int = .max,
        @RegexComponentBuilder content: () -> R,
        with replacement: (Regex<R.RegexOutput>.Match) throws -> Replacement
    ) rethrows -> Self where Replacement.Element == Element

    /// Returns a new collection in which all matches for the regex
    /// are replaced, using the given closures to create the replacement
    /// and the regex.
    ///
    /// - Parameters:
    ///   - maxReplacements: A number specifying how many occurrences of
    ///     the regex to replace, using `content` to create the regex.
    ///   - content: A closure that returns the collection to search for
    ///     and replace.
    ///   - replacement: A closure that receives the full match information,
    ///   including captures, and returns a replacement collection.
    /// - Returns: A new collection in which all matches for regex in `subrange`
    ///   are replaced by the result of calling `replacement`, where regex is
    ///   the result of calling `content`.
    public func replacing<R: RegexComponent, Replacement: Collection>(
        maxReplacements: Int = .max,
        @RegexComponentBuilder content: () -> R,
        with replacement: (Regex<R.RegexOutput>.Match) throws -> Replacement
    ) rethrows -> Self where Replacement.Element == Element

    /// Replaces all matches for the regex in this collection, using the
    /// given closures to create the replacement and the regex.
    ///
    /// - Parameters:
    ///   - maxReplacements: A number specifying how many occurrences of
    ///     the regex to replace, using `content` to create the regex.
    ///   - content: A closure that returns the collection to search for
    ///     and replace.
    ///   - replacement: A closure that receives the full match information,
    ///     including captures, and returns a replacement collection.
    public mutating func replace<R: RegexComponent, Replacement: Collection>(
        maxReplacements: Int = .max,
        @RegexComponentBuilder content: () -> R,
        with replacement: (Regex<R.RegexOutput>.Match) throws -> Replacement
    ) rethrows where Replacement.Element == Element
}
```

#### Split

We propose a generic collection `split` that can take a subsequence separator:

```swift
extension Collection where Element: Equatable {
    /// Returns the longest possible subsequences of the collection, in order,
    /// around elements equal to the given separator collection.
    ///
    /// - Parameters:
    ///   - separator: A collection of elements to be split upon.
    ///   - maxSplits: The maximum number of times to split the collection,
    ///     or one less than the number of subsequences to return.
    ///   - omittingEmptySubsequences: If `false`, an empty subsequence is
    ///     returned in the result for each consecutive pair of separator
    ///     sequences in the collection and for each instance of separator
    ///     sequences at the start or end of the collection. If `true`, only
    ///     nonempty subsequences are returned.
    /// - Returns: A collection of subsequences, split from this collection's
    ///   elements.
    public func split<C: Collection>(
        separator: C,
        maxSplits: Int = Int.max,
        omittingEmptySubsequences: Bool = true
    ) -> some Collection<SubSequence> where C.Element == Element
}
extension BidirectionalCollection where Element: Comparable {
    /// Returns the longest possible subsequences of the collection, in order,
    /// around elements equal to the given separator collection.
    ///
    /// - Parameters:
    ///   - separator: A collection of elements to be split upon.
    ///   - maxSplits: The maximum number of times to split the collection,
    ///     or one less than the number of subsequences to return.
    ///   - omittingEmptySubsequences: If `false`, an empty subsequence is
    ///     returned in the result for each consecutive pair of separator
    ///     sequences in the collection and for each instance of separator
    ///     sequences at the start or end of the collection. If `true`, only
    ///     nonempty subsequences are returned.
    /// - Returns: A collection of subsequences, split from this collection's
    ///   elements.
    public func split<C: Collection>(
        separator: C,
        maxSplits: Int = Int.max,
        omittingEmptySubsequences: Bool = true
    ) -> some Collection<SubSequence> where C.Element == Element
}
```

And a regex-taking variant for string types:

```swift
extension Collection where SubSequence == Substring {
    /// Returns the longest possible subsequences of the collection, in order,
    /// around subsequence that match the given separator regex.
    ///
    /// - Parameters:
    ///   - separator: A regex to be split upon.
    ///   - maxSplits: The maximum number of times to split the collection,
    ///     or one less than the number of subsequences to return.
    ///   - omittingEmptySubsequences: If `false`, an empty subsequence is
    ///     returned in the result for each consecutive pair of matches
    ///     and for each match at the start or end of the collection. If
    ///     `true`, only nonempty subsequences are returned.
    /// - Returns: A collection of substrings, split from this collection's
    ///   elements.
    public func split(
        separator: some RegexComponent,
        maxSplits: Int = Int.max,
        omittingEmptySubsequences: Bool = true
    ) -> some Collection<Substring>
}

// In RegexBuilder module
extension Collection where SubSequence == Substring {
    /// Returns the longest possible subsequences of the collection, in order,
    /// around subsequence that match the regex created by the given closure.
    ///
    /// - Parameters:
    ///   - maxSplits: The maximum number of times to split the collection,
    ///     or one less than the number of subsequences to return.
    ///   - omittingEmptySubsequences: If `false`, an empty subsequence is
    ///     returned in the result for each consecutive pair of matches
    ///     and for each match at the start or end of the collection. If
    ///     `true`, only nonempty subsequences are returned.
    ///   - separator: A closure that returns a regex to be split upon.
    /// - Returns: A collection of substrings, split from this collection's
    ///   elements.
    public func split(
        maxSplits: Int = Int.max,
        omittingEmptySubsequences: Bool = true,
        @RegexComponentBuilder separator: () -> some RegexComponent
    ) -> some Collection<Substring>
}
```

**Note:** We plan to adopt the new generics features enabled by [SE-0346][] for these proposed methods when the standard library adopts primary associated types, [pending a forthcoming proposal][stdlib-pitch]. For example, the first method in the _Replacement_ section above would instead be:

```swift
extension RangeReplaceableCollection where Element: Equatable {
    /// Returns a new collection in which all occurrences of a target sequence
    /// are replaced by another collection.
    public func replacing(
        _ other: some Collection<Element>,
        with replacement: some Collection<Element>,
        subrange: Range<Index>,
        maxReplacements: Int = .max
    ) -> Self
}
```

#### Searching for empty strings and matches

Empty matches and inputs are an important edge case for several of the algorithms proposed above. For example, what is the result of `"123.firstRange(of: /[a-z]*/)`? How do you split a collection separated by an empty collection, as in `"1234".split(separator: "")`? For the Swift standard library, this is a new consideration, as current algorithms are `Element`-based and cannot be passed an empty input.

Languages and libraries are nearly unanimous about finding the location of an empty string, with Ruby, Python, C#, Java, Javascript, etc, finding an empty string at each index in the target. Notably, Foundation's `NSString.range(of:)` does _not_ find an empty string at all.

The methods proposed here follow the consensus behavior, which makes sense if you think of `a.firstRange(of: b)` as returning the first subrange `r` where `a[r] == b`. If a regex can match an empty substring, like `/[a-z]*/`, the behavior is the same.

```swift
let hello = "Hello"
let emptyRange = hello.firstRange(of: "")
// emptyRange is equivalent to '0..<0' (integer ranges shown for readability)
```

Because searching again at the same index would yield that same empty string, we advance one position after finding an empty string or matching an empty pattern when finding all ranges. This yields the position of every valid index in the string.

```swift
let allRanges = hello.ranges(of: "")
// allRanges is equivalent to '[0..<0, 1..<1, 2..<2, 3..<3, 4..<4, 5..<5]'
```

Splitting with an empty separator (or a pattern that matches empty string), uses this same behavior, resulting in a collection of single-element substrings. Interestingly, a couple languages make different choices here. C# returns the original string instead of its parts, and Python rejects an empty separator (though it permits regexes that match empty strings).

```swift
let parts = hello.split(separator: "")
// parts == ["h", "e", "l", "l", "o"]

let moreParts = hello.split(separator: "", omittingEmptySubsequences: false)
// parts == ["", "h", "e", "l", "l", "o", ""]
```

Finally, searching for an empty string within an empty string yields, as you might imagine, the empty string:

```swift
let empty = ""
let range = empty.firstRange(of: empty)
// empty == empty[range]
```

[SE-0346]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0346-light-weight-same-type-syntax.md
[stdlib-pitch]: https://forums.swift.org/t/pitch-primary-associated-types-in-the-standard-library/56426

## Alternatives considered

### Extend `Sequence` instead of `Collection`

Most of the proposed algorithms are necessarily on `Collection` due to the use of indices or mutation. `Sequence` does not support multi-pass iteration, so even `trimmingPrefix` would problematic on `Sequence` because it needs to look one `Element` ahead to know when to stop trimming and would need to return a wrapper for the in-progress iterator instead of a subsequence.

### Cross-proposal API naming consistency

The regex work is broken down into 6 proposals based on technical domain, which is advantageous for deeper technical discussions and makes reviewing the large body of work manageable. The disadvantage of this approach is that relatively-shallow cross-cutting concerns, such as API naming consistency, are harder to evaluate until we've built up intuition from multiple proposals.

We've seen the [Regex type and overview](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0350-regex-type-overview.md), the [Regex builder DSL](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0351-regex-builder.md), and here we present lots of ways to use regex. Now's a good time to go over API naming consistency.

(The other proposal with a significant amount of API is [Unicode for String Processing](https://forums.swift.org/t/pitch-unicode-for-string-processing/56907), which is in the pitch phase. It is a technical niche and less impactful on these naming discussions. We'll still want to design those names for consistency, of course.)


```swift
protocol RegexComponent {
    associatedtype RegexOutput
}
```

The associatedtype name is "RegexOutput" to help libraries conform their parsers to this protocol (e.g. via `CustomConsumingRegexComponent`). Regex's capture representation is regexy: it has the overall matched portion as the first capture and the regex builders know how to combine these kinds of capture lists together. This could be different than how e.g. a parser combinator library's output types might be represented. Thus, we chose a more specific name to avoid any potential conflicts.

The name "RegexComponent" accentuates that any conformer can be used as part of a larger regex, while it de-emphasizes that `Regex` instances themselves can be used directly. We propose methods that are generic over `RegexComponent` and developers will be considering whether they should make their functions that otherwise take a `Regex` also be generic over `RegexComponent`.

It's possible there might be some initial confusion around the word "component", i.e. a developer may have a regex and not be sure how to make it into a component or how to get the component out. The word "component" carries a lot of value in the context of the regex DSL. An alternative name might be `RegexProtocol`, which implies that a Regex can be used at the site and would be clearly the way to make a function taking a concrete `Regex` generic. But, it's otherwise a naming workaround that doesn't carry the additional regex builder connotations.

The protocol requirement is `var regex: Regex<RegexOutput>`, i.e. any type that can produce a regex or hook into the engine's customization hooks (this is what `consuming` does) can be used as a component of the DSL and with these generic API. An alternative name could be "CustomRegexConvertible", but we don't feel that communicates component composability very well, nor is it particularly enlightening when encountering these generic API.

Another alternative is to have a second protocol just for generic API. But without a compelling semantic distinction or practical utility, we'd prefer to avoid adding protocols just for names. If a clearly superior name exists, we should just choose that.


```swift
protocol CustomConsumingRegexComponent {
    func consuming(...)
}
```

This is not a normal developer-facing protocol or concept; it's an advanced library-extensibility feature. Explicit, descriptive, and careful names are more important than concise names. The "custom" implies that we're not just vending a regex directly ourselves, we're instead customizing behavior by hooking into the run-time engine directly.

Older versions of the pitch had `func match(...) -> (String.Index, T)?` as the protocol requirement. As [Regex type and overview](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0350-regex-type-overview.md) went through review, naming convention settled on using the word "match" as a noun and in context with operations that produce a `Match` instance. Since this is the engine's customization hook, it produces the value and position to resume execution from directly, and hence different terminology is apt and avoids confusion or future ambiguities. "Consuming" is the nomenclature we're going with for something that chews off the front of its input in order to produces a value.

This protocol customizes the basic consume-from-the-front functionality. A protocol for customizing search is future work and involves accommodating different kinds of state and ways that a searcher may wish to speed up subsequent searches. Alternative names for the protocol include `CustomRegexComponent`, `CustomConsumingRegex`, etc., but we don't feel brevity is the key consideration here.


### Why `where SubSequence == Substring`?

A `Substring` slice requirement allows the regex engine to produce indicies in the original collection by operating over a portion of the input. Unfortunately, this is not one of the requirements of `StringProtocol`.

A new protocol for types that can produce a `Substring` on request (e.g. from UTF-8 contents) would have to eagerly produce a `String` copy first and would need requirements to translate indices. When higher-level algorithms are implemented via multiple calls to the lower-level algorithms, these copies could happen many times. Shared strings are future work but a much better solution to this.
   
## Future directions

### Backward algorithms

It would be useful to have algorithms that operate from the back of a collection, including ability to find the last non-overlapping range of a pattern in a string, and/or that to find the first range of a pattern when searching from the back, and trimming a string from both sides. They are deferred from this proposal as the API that could clarify the nuances of backward algorithms are still being explored.

<details>
<summary> Nuances of backward algorithms </summary>

There is a subtle difference between finding the last non-overlapping range of a pattern in a string, and finding the first range of this pattern when searching from the back.

The currently proposed algorithm that finds a pattern from the front, e.g. `"aaaaa".ranges(of: "aa")`, produces two non-overlapping ranges, splitting the string in the chunks `aa|aa|a`. It would not be completely unreasonable to expect to introduce a counterpart, such as `"aaaaa".lastRange(of: "aa")`, to return the range that contains the third and fourth characters of the string. This would be a shorthand for `"aaaaa".ranges(of: "aa").last`. Yet, it would also be reasonable to expect the function to return the first range of `"aa"` when searching from the back of the string, i.e. the range that contains the fourth and fifth characters.

Trimming a string from both sides shares a similar story. For example, `"ababa".trimming("aba")` can return either `"ba"` or `"ab"`, depending on whether the prefix or the suffix was trimmed first.
</details>

### Split preserving the separator

Future work is a split variant that interweaves the separator with the separated portions. For example, when splitting over `\p{punctuation}` it might be useful to be able to preserve the punctionation as a separate entry in the returned collection.

### Future API

Some common string processing functions are not currently included in this proposal, such as trimming the suffix from a string/collection, and finding overlapping ranges of matched substrings. This pitch aims to establish a pattern for using `RegexComponent` with string processing algorithms, so that further enhancement can to be introduced to the standard library easily in the future, and eventually close the gap between Swift and other popular scripting languages.
