# Adding in-place `removeAll(where:)` to the Standard Library

* Proposal: [SE-0197](0197-remove-where.md)
* Author: [Ben Cohen](https://github.com/airspeedswift)
* Review Manager: [John McCall](https://github.com/rjmccall)
* Status: **Implemented (Swift 4.2)**
* Implementation: [apple/swift#11576](https://github.com/apple/swift/pull/11576)
* Review: [Thread](https://forums.swift.org/t/se-0197-add-in-place-remove-where/8872)
* Previous Revision: [1](https://github.com/swiftlang/swift-evolution/blob/feec7890d6c193e9260ac9905456f25ef5656acd/proposals/0197-remove-where.md)

## Introduction

It is common to want to remove all occurrences of a certain element from a
collection. This proposal is to add a `removeAll` algorithm to the
standard library, which will remove all entries in a collection in-place
matching a given predicate.

## Motivation

Removing all elements matching some criteria is a very common operation.
However, it can be tricky to implement correctly and
efficiently.

The easiest way to achieve this effect in Swift 3 is to use `filter` and assign
back, negating the thing you want to remove (because `filter` takes a closure
of items to "keep"):

```swift
var nums = [1,2,3,4,5]
// remove odd elements
nums = nums.filter { !isOdd($0) }
```

In addition to readability concerns, this has two performance problems: fresh
memory allocation, and a copy of all the elements in full even if none need to
be removed.

The alternative is to open-code a `for` loop. The simplest performant solution
is the "shuffle-down" approach. While not especially complex, it is certainly
non-trivial:

```swift
if var i = nums.index(where: isOdd) {
  var j = i + 1
  while j != nums.endIndex {
    let e = nums[j]
    if !isOdd(nums[j]) {
      nums[i] = nums[j]
      i += 1
    }
    j += 1
  }
  nums.removeSubrange(i...)
}
```

Possibilities for logic and performance errors abound. There are probably some
in the above code.

Additionally, this approach does not work for range-replaceable collections
that are _not_ mutable i.e. collections that can replace subranges, but can't
guarantee replacing a single element in constant time. `String` is the most
important example of this, because its elements (graphemes) are variable width.

## Proposed solution

Add the following method to `RangeReplaceableCollection`:

```swift
nums.removeAll(where: isOdd)
```

The default implementation will use the protocol's `init()` and `append(_:)`
operations to implement a copy-based version. Collections which also conform to
`MutableCollection` will get the more efficient "shuffle-down" implementation,
but still require `RangeReplaceableCollection` as well because of the need to
trim at the end. Other types may choose

Collections which are range replaceable but _not_ mutable (like `String`) will
be able to implement their own version which makes use of their internal
layout. Collections like `Array` may also implement more efficient versions
using memory copying operations.

Since `Dictionary` and `Set` would benefit from this functionality as well, but
are not range-replaceable, they should be given concrete implementations for
consistency.

## Detailed design

Add the following to `RangeReplaceableCollection`:

```swift
protocol RangeReplaceableCollection {
  /// Removes every element satisfying the given predicate from the collection.
  mutating func removeAll(where: (Iterator.Element) throws -> Bool) rethrows
}

extension RangeReplaceableCollection {
  mutating func removeAll(where: (Iterator.Element) throws -> Bool) rethrows {
    // default implementation similar to self = self.filter
  }
}
```

Other protocols or types may also have custom implementations for a faster
equivalent. For example `RangeReplaceableCollection where Self:
MutableCollection` can provide a more efficient non-allocating default
implementation. `String` is also likely to benefit from a custom implementation.

## Source compatibility

This change is purely additive so has no source compatibility consequences.

## Effect on ABI stability

This change is purely additive so has no ABI stability consequences.

## Effect on API resilience

This change is purely additive so has no API resilience consequences.

## Alternatives considered

`removeAll(where:)` takes a closure with `true` for elements to remove.
`filter` takes a closure with elements to keep. In both cases, `true` is the
"active" case, so likely to be what the user wants without having to apply a
negation. The naming of `filter` is unfortunately ambiguous as to whether it's
a removing or keeping operation, but re-considering that is outside the scope
of this proposal.

Several collection methods in the standard library (such as `index(where:)`)
have an equivalent for collections of `Equatable` elements. A similar addition
could be made that removes every element equal to a given value. This could
easily be done as a further additive proposal later.

The initial proposal of this feature named it `remove(where:)`.  During review,
it was agreed that this was unnecessarily ambiguous about whether all the
matching elements should be removed or just the first, and so the method was
renamed to `removeAll(where:)`.

