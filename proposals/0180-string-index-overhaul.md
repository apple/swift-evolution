# String Index Overhaul

* Proposal: [SE-0180](0180-string-index-overhaul.md)
* Author: [Dave Abrahams](https://github.com/dabrahams)
* Review Manager: [Ted Kremenek](https://github.com/tkremenek)
* Status: **Implemented (Swift 4.0)**
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-se-0180-string-index-overhaul/6286)
* Implementation: [apple/swift#9806](https://github.com/apple/swift/pull/9806)
* Previous Revision: [1](https://github.com/swiftlang/swift-evolution/blob/72b8d90becd60b7cc7695607ae908ef251f1e966/proposals/0180-string-index-overhaul.md)

## Introduction

Today `String` shares an `Index` type with its `CharacterView` but not
with its `UTF8View`, `UTF16View`, or `UnicodeScalarView`.  This
proposal redefines `String.UTF8View.Index`, `String.UTF16View.Index`,
and `String.CharacterView.Index` as typealiases for `String.Index`,
and exposes a public `encodedOffset` property and initializer that can
be used to serialize and deserialize positions in a `String` or
`Substring`.

Swift-evolution thread: [Pitch: String Index Overhaul](https://forums.swift.org/t/pitch-string-index-overhaul/6017)

## Motivation

The different index types are supported by a set of `Index`
initializers, which are failable whenever the source index might not
correspond to a position in the target view:

```swift
if let j = String.UnicodeScalarView.Index(
  someUTF16Position, within: s.unicodeScalars) {
  ... 
}
```

The current API is as follows:

```swift
public extension String.Index {
  init?(_: String.UnicodeScalarIndex, within: String)
  init?(_: String.UTF16Index, within: String)
  init?(_: String.UTF8Index, within: String)
}

public extension String.UTF16View.Index {
  init?(_: String.UTF8Index, within: String.UTF16View)
  init(_: String.UnicodeScalarIndex, within: String.UTF16View)
  init(_: String.Index, within: String.UTF16View)
}

public extension String.UTF8View.Index {
  init?(_: String.UTF16Index, within: String.UTF8View)
  init(_: String.UnicodeScalarIndex, within: String.UTF8View)
  init(_: String.Index, within: String.UTF8View)
}

public extension String.UnicodeScalarView.Index {
  init?(_: String.UTF16Index, within: String.UnicodeScalarView)
  init?(_: String.UTF8Index, within: String.UnicodeScalarView)
  init(_: String.Index, within: String.UnicodeScalarView)
}
```

These initializers are supplemented by a corresponding set of
convenience conversion methods:

```swift
if let j = someUTF16Position.samePosition(in: s.unicodeScalars) {
  ... 
}
```

with the following API:

```swift
public extension String.Index {
  func samePosition(in: String.UTF8View) -> String.UTF8View.Index
  func samePosition(in: String.UTF16View) -> String.UTF16View.Index
  func samePosition(
    in: String.UnicodeScalarView) -> String.UnicodeScalarView.Index
}

public extension String.UTF16View.Index {
  func samePosition(in: String) -> String.Index?
  func samePosition(in: String.UTF8View) -> String.UTF8View.Index?
  func samePosition(
    in: String.UnicodeScalarView) -> String.UnicodeScalarView.Index?
}

public extension String.UTF8View.Index {
  func samePosition(in: String) -> String.Index?
  func samePosition(in: String.UTF16View) -> String.UTF16View.Index?
  func samePosition(
    in: String.UnicodeScalarView) -> String.UnicodeScalarView.Index?
}

public extension String.UnicodeScalarView.Index {
  func samePosition(in: String) -> String.Index?
  func samePosition(in: String.UTF8View) -> String.UTF8View.Index
  func samePosition(in: String.UTF16View) -> String.UTF16View.Index
}
```

The result is a great deal of API surface area for apparently little
gain in ordinary code, that normally only interchanges indices among
views when the positions match up exactly (i.e. when the conversion is
going to succeed).  Also, the resulting code is needlessly awkward.

Finally, the opacity of these index types makes it difficult to record
`String` or `Substring` positions in files or other archival forms,
and reconstruct the original positions with respect to a deserialized
`String` or `Substring`.

## Proposed solution

All `String` views will use a single index type (`String.Index`), so
that positions can be interchanged without awkward explicit
conversions:

```swift
let html: String = "See <a href=\"http://swift.org\">swift.org</a>"

// Search the UTF16, instead of characters, for performance reasons:
let open = "<".utf16.first!, close = ">".utf16.first!
let tagStart = html.utf16.index(of: open)
let tagEnd = html.utf16[tagStart...].index(of: close)

// Slice the String with the UTF-16 indices to retrieve the tag.
let tag = html[tagStart...tagEnd]
```

A property and an intializer will be added to `String.Index`, exposing
the offset of the index in code units (currently only UTF-16) from the
beginning of the string:

```swift
let n: Int = html.endIndex.encodedOffset
let end = String.Index(encodedOffset: n)
assert(end == String.endIndex)
```

# Comparison and Subscript Semantics

When two indices being compared correspond to positions that are valid
in any single `String` view, comparison semantics are already fully
specified by the `Collection` requirements.  The other cases occur
when indices fall between Unicode scalar boundaries in views having
distinct encodings.  For example, the string `"\u{1f773}"` (“🝳”) is
encoded as `0xD83D, 0xDF73` in UTF-16 and `0xF0, 0x9F, 0x9D, 0xB3` in
UTF-8, and there is no obvious way to compare the second positions in
each of those sequences.  The proposed rule is that such indices are
compared by comparing their `encodedOffset`s.  Such index values are
not totally ordered but do satisfy strict weak ordering requirements,
which is sufficient for algorithms such as `sort` to exhibit sensible
behavior.  We might consider loosening the specified requirements on
these algorithms and on `Comparable` to support strict weak ordering,
but for now we can treat such index pairs as being formally outside
the domain of comparison, like any other indices from completely
distinct collections.

With respect to subscripts, an index that does not fall on an exact
boundary in a given `String` or `Substring` view will be treated as
falling at its `encodedOffset` in the underlying code units, with the
actual contents of the result being an emergent property of applying
the usual Unicode rules for decoding those code units.  For example,
when slicing a string with an index `i` that falls between two
`Character` boundaries, `i.encodedOffset` is treated as a position in
the string's underlying code units, and the `Character`s of the result
are determined by performing standard Unicode grapheme breaking on the
resulting sequence of code units.

```swift
let s = "e\u{301}galite\u{301}"           // "égalité"
let i = Array(s.unicodeScalars.indices)
print(s[i[1]...])                         // "◌́galité"
print(s[..<i.last!])                      // "égalite"
print(s[i[1])                             // "◌́"
```

Similarly, assignment to a slice of a string is performed by replacing
the corresponding code units, and again the resulting `Characters` are
determined by re-applying standard grapheme breaking rules.

Replacing the failable APIs listed [above](#motivation) that detect
whether an index represents a valid position in a given view, and
enhancement that explicitly round index positions to nearby boundaries
in a given view, are left to a later proposal.  For now, we do not
propose to remove the existing index conversion APIs.

## Detailed design

`String.Index` acquires an `encodedOffset` property and initializer:

```swift
public extension String.Index {
  /// Creates a position corresponding to the given offset in a
  /// `String`'s underlying (UTF-16) code units.
  init(encodedOffset: Int)

  /// The position of this index expressed as an offset from the
  /// beginning of the `String`'s underlying (UTF-16) code units.
  var encodedOffset: Int
}
```

`Index` types of `String.UTF8View`, `String.UTF16View`, and
`String.UnicodeScalarView` are replaced by `String.Index`:

```swift
public extension String.UTF8View {
  typealias Index = String.Index
}
public extension String.UTF16View {
  typealias Index = String.Index
}
public extension String.UnicodeScalarView {
  typealias Index = String.Index
}
```

Because the index types are collapsing, index conversion methods and
initializers are reduced to the following:

```swift
public extension String.Index {
  init?(_: String.Index, within: String)
  init?(_: String.Index, within: String.UTF8View)
  init?(_: String.Index, within: String.UTF16View)
  init?(_: String.Index, within: String.UnicodeScalarView)

  func samePosition(in: String) -> String.Index?
  func samePosition(in: String.UTF8View) -> String.Index?
  func samePosition(in: String.UTF16View) -> String.Index?
  func samePosition(in: String.UnicodeScalarView) -> String.Index?
}
```

## Source compatibility

Because of the collapse of index
types, [existing non-failable APIs](#motivation) become failable.  To
avoid breaking Swift 3 code, the following overloads of existing
functions are added, allowing the resulting optional indices to be
used where previously non-optional indices were used.  These overloads
were driven by making the new APIs work with existing code, including
the Swift source compatibility test suite, and should be viewed as
migration aids only, rather than additions to the Swift 3 API.

```swift
extension Optional where Wrapped == String.Index {
  @available(
    swift, deprecated: 3.2, obsoleted: 4.0,
    message: "Any String view index conversion can fail in Swift 4; please unwrap the optional indices")
  public static func ..<(
    lhs: String.Index?, rhs: String.Index?
  ) -> Range<String.Index> {
    return lhs! ..< rhs!
  }

  @available(
    swift, deprecated: 3.2, obsoleted: 4.0,
    message: "Any String view index conversion can fail in Swift 4; please unwrap the optional indices")
  public static func ...(
    lhs: String.Index?, rhs: String.Index?
  ) -> ClosedRange<String.Index> {
    return lhs! ... rhs!
  }
}

// backward compatibility for index interchange.  
extension String.UTF16View {
  @available(
    swift, deprecated: 3.2, obsoleted: 4.0,
    message: "Any String view index conversion can fail in Swift 4; please unwrap the optional index")
  public func index(after i: Index?) -> Index {
    return index(after: i)
  }
  @available(
    swift, deprecated: 3.2, obsoleted: 4.0,
    message: "Any String view index conversion can fail in Swift 4; please unwrap the optional index")
  public func index(
    _ i: Index?, offsetBy n: IndexDistance) -> Index {
    return index(i!, offsetBy: n)
  }
  @available(
    swift, deprecated: 3.2, obsoleted: 4.0,
    message: "Any String view index conversion can fail in Swift 4; please unwrap the optional indices")
  public func distance(from i: Index?, to j: Index?) -> IndexDistance {
    return distance(from: i!, to: j!)
  }
  @available(
    swift, deprecated: 3.2, obsoleted: 4.0,
    message: "Any String view index conversion can fail in Swift 4; please unwrap the optional index")
  public subscript(i: Index?) -> Unicode.UTF16.CodeUnit {
    return self[i!]
  }
}

extension String.UTF8View {
  @available(
    swift, deprecated: 3.2, obsoleted: 4.0,
    message: "Any String view index conversion can fail in Swift 4; please unwrap the optional index")
  public func index(after i: Index?) -> Index {
    return index(after: i!)
  }
  @available(
    swift, deprecated: 3.2, obsoleted: 4.0,
    message: "Any String view index conversion can fail in Swift 4; please unwrap the optional index")
  public func index(_ i: Index?, offsetBy n: IndexDistance) -> Index {
    return index(i!, offsetBy: n)
  }
  @available(
    swift, deprecated: 3.2, obsoleted: 4.0,
    message: "Any String view index conversion can fail in Swift 4; please unwrap the optional indices")
  public func distance(
    from i: Index?, to j: Index?) -> IndexDistance {
    return distance(from: i!, to: j!)
  }
  @available(
    swift, deprecated: 3.2, obsoleted: 4.0,
    message: "Any String view index conversion can fail in Swift 4; please unwrap the optional index")
  public subscript(i: Index?) -> Unicode.UTF8.CodeUnit {
    return self[i!]
  }
}

// backward compatibility for index interchange.  
extension String.UnicodeScalarView {
  @available(
    swift, deprecated: 3.2, obsoleted: 4.0,
    message: "Any String view index conversion can fail in Swift 4; please unwrap the optional index")
  public func index(after i: Index?) -> Index {
    return index(after: i)
  }
  @available(
    swift, deprecated: 3.2, obsoleted: 4.0,
    message: "Any String view index conversion can fail in Swift 4; please unwrap the optional index")
  public func index(_ i: Index?,  offsetBy n: IndexDistance) -> Index {
    return index(i!, offsetBy: n)
  }
  @available(
    swift, deprecated: 3.2, obsoleted: 4.0,
    message: "Any String view index conversion can fail in Swift 4; please unwrap the optional indices")
  public func distance(from i: Index?, to j: Index?) -> IndexDistance {
    return distance(from: i!, to: j!)
  }
  @available(
    swift, deprecated: 3.2, obsoleted: 4.0,
    message: "Any String view index conversion can fail in Swift 4; please unwrap the optional index")
  public subscript(i: Index?) -> Unicode.Scalar {
    return self[i!]
  }
}
```

- **Q**: Will existing correct Swift 3 applications stop compiling due
  to this change?

  **A**: it is possible but unlikely.  The existing index conversion
  APIs are relatively rarely used, and the overloads listed above
  handle the common cases in Swift 3 compatibility mode.
  
- **Q**: Will applications still compile but produce
  different behavior than they used to? 

  **A**: No.
  
- **Q**: Is it possible to automatically migrate from the old syntax
  to the new syntax? 

  **A**: Yes, although usages of these APIs may be rare enough that it
  isn't worth the trouble.

- **Q**: Can Swift applications be written in a common subset that works
   both with Swift 3 and Swift 4 to aid in migration?

  **A**: Yes, the Swift 4 APIs will all be available in Swift 3 mode.

## Effect on ABI stability

This proposal changes the ABI of the standard library.

## Effect on API resilience

This proposal makes no changes to the resilience of any APIs.

## Alternatives considered

The only alternative considered was no action.
