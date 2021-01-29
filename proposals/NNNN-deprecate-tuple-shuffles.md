Deprecate Tuple Shuffles
========================

-   Proposal: [SE-NNNN](NNNN-filename.md)
-   Authors: [Robert Widmann](https://github.com/codafi), [David Hart](https://github.com/hartbit)
-   Review Manager: TBD
-   Status: **Awaiting review**

Introduction
------------

This proposal seeks to deprecate certain forms of a little-known feature of Swift called a "Tuple Shuffle"†. A Tuple Shuffle is an undocumented feature of Swift in which one can reference tuple labels out of order in certain expressions.

### Tuple-Shuffle Examples

Tuple Shuffles are best seen in action:

* Assigning to a tuple with the same labels in a different order:

```swift
let a = (x: 1, y: 2)
let b: (y: Int, x: Int)
b = a
print(b) // prints (y: 2, x: 1)
```

* Destructuring a tuple by referencing labels out of order:

```swift
let a = (x: 1, y: 2)
let (y: b, x: c) = a
print(b) // prints 2
print(c) // prints 1
```

* Mapping parameter labels out of order in a call expression:

```swift
func foo(_ : (x: Int, y: Int)) {}
foo((y: 5, x: 10))
```

### Non-Examples

Similar syntax that does not reorder tuple labels are **not** Tuple Shuffles:

* An assignment to a tuple with the same labels in the same order:

```swift
let a = (x: 1, y: 2)
let b: (x: Int, y: Int)
b = a
```

* An assignment to a tuple without labels:

```swift
let a = (x: 1, y: 2)
let b: (Int, Int)
b = a
```

* Destructuring a tuple and by referencing labels in order:

```swift
let a = (x: 1, y: 2)
let (x: b, y: c) = a
```

* Re-assignment through a tuple pattern:

```swift
var x = 5
var y = 10
var z = 15
(z, y, x) = (x, z, y)
```

Motivation
----------

The inclusion of Tuple Shuffles in the language complicates every part of the compiler stack, contradicts the goals of earlier SE's (see [SE-0060](https://github.com/apple/swift-evolution/blob/9cf2685293108ea3efcbebb7ee6a8618b83d4a90/proposals/0060-defaulted-parameter-order.md)), and makes non-sensical behaviors possible in surprising places.

Consider the following:

```swift
var a: (Int, y: Int) = (2, 1)
var b: (y: Int, Int) = (1, 2)

a = b
print(a == b) // false!
```

This reveals an inconsistency between the language and its standard library (where equality of tuples is defined).  Where the language permits the first assignment to succeed by virtue of an implicit Tuple Shuffle, the equality fails because the Swift Standard Library considers tuples equal when their elements are index-by-index equal. The rest of the Swift has seemingly agreed that operations on tuples preserve and respect their *parallel structure*. The rest of Swift, that is, except Tuple Shuffles.

This proposal seeks to deprecate Tuple Shuffles in Swift 4 compatibility mode and enforce that deprecation as a hard error in Swift 5 to facilitate their eventual removal from the language.

Proposed solution
-----------------

Construction of Tuple Shuffles will become a warning in Swift 4 compatibility mode and will be a hard error in Swift 5.

Detailed design
---------------

†Throughout this proposal, we have referred to "Tuple Shuffles" as a monolithic feature.  However,
the compiler currently models many things with *Tuple Shuffle Expressions* including variadic and default 
arguments. For the purpose of this discussion a Tuple Shuffle is defined to be any Tuple Shuffle Expression 
that causes the labels in its type to be reordered.

All of the examples above fit this model:

```swift
let a = (x: 1, y: 2)
let b: (y: Int, x: Int)
b = a // Shuffles x -> y, y -> x
```

```swift
let a = (x: 1, y: 2)
let (y: b, x: c) = a // Shuffles x -> y, y -> x
```

```swift
func foo(_ : (x : Int, y : Int)) {}
foo((y: 5, x: 10)) // Shuffles x -> y, y -> x
```

The compiler shall continue to accept and construct all forms of Tuple Shuffle Expression under Swift 4 compatibility mode.  In Swift 5, Tuple Shuffles will be removed from the language.

Impact on Existing Code
-----------------------

Because very little code is intentionally using Tuple Shuffles, impact on existing code will be
negligible. In fact, turning on the error-producing behavior we intended for Swift 5 in
all compiler modes passes the Swift Source Compatibility Suite.

Alternatives considered
-----------------------

Continue to keep the architecture in place to facilitate this feature.
