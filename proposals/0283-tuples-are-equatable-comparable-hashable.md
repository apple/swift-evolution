# Tuples Conform to `Equatable`, `Comparable`, and `Hashable`

* Proposal: [SE-0283](0283-tuples-are-equatable-comparable-hashable.md)
* Author: [Alejandro Alonso](https://github.com/Azoy)
* Review Manager: [Saleem Abdulrasool](https://github.com/compnerd)
* Status: **Accepted (2020-05-19)**
* Implementation: [apple/swift#28833](https://github.com/apple/swift/pull/28833)
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-se-0283-tuples-conform-to-equatable-comparable-and-hashable/36658), [Additional Commentary](https://forums.swift.org/t/implementation-issues-with-se-0283-tuples-are-ehc/46946)

## Introduction

Introduce `Equatable`, `Comparable`, and `Hashable` conformance for all tuples whose elements are themselves `Equatable`, `Comparable`, and `Hashable`.

Swift-evolution thread: [Tuples Conform to Equatable, Comparable, and Hashable](https://forums.swift.org/t/tuples-conform-to-equatable-comparable-and-hashable/34156)

## Motivation

Tuples in Swift currently lack the ability to conform to protocols. This has led many users to stop using tuples altogether in favor of structures that can conform to protocols. The shift — from tuples to structures — has made tuples almost feel like a second-class type in the language, because of them not being able to do simple operations that should *just* work.

Consider the following snippet of code that naively tries to use tuples for simple operations, but instead is faced with ugly errors.

```swift
let points = [(x: 128, y: 316), (x: 0, y: 0), (x: 100, y: 42)]
let origin = (x: 0, y: 0)

// error: type '(x: Int, y: Int)' cannot conform to 'Equatable';
//        only struct/enum/class types can conform to protocols
if points.contains(origin) {
  // do some serious calculations here
}

// error: type '(x: Int, y: Int)' cannot conform to 'Comparable';
//        only struct/enum/class types can conform to protocols
let sortedPoints = points.sorted()

// error: type '(x: Int, y: Int)' cannot conform to 'Hashable';
//        only struct/enum/class types can conform to protocols
let uniquePoints = Set(points)
```

This also creates friction when one needs to conditionally conform to a type, or if a type is just trying to get free conformance synthesis for protocols like `Equatable` or `Hashable`.

```swift
struct Restaurant {
  let name: String
  let location: (latitude: Int, longitude: Int)
}

// error: type 'Restaurant' does not conform to protocol 'Equatable'
extension Restaurant: Equatable {}

// error: type 'Restaurant' does not conform to protocol 'Hashable'
extension Restaurant: Hashable {}
```

These are simple and innocent examples of trying to use tuples in one's code, but currently the language lacks the means to get these examples working and prevents the user from writing this code.

After all the errors, one decides to give in and create a structure to mimic the tuple layout. From a code size perspective, creating structures to mimic each unique tuple need adds a somewhat significant amount of size to one's binary.

## Proposed solution

Introduce `Equatable`, `Comparable`, and `Hashable` conformance for all tuples whose elements themselves conform to said protocols. While this isn't a general-purpose "conform any tuple to any protocol" proposal, `Equatable`, `Comparable`, and `Hashable` are crucial protocols to conform to, because it allows for all of the snippets above in Motivation to compile and run as expected, along with many other standard library operations to work nicely with tuples.

### Equatable

The rule is simple: if all of the tuple elements are themselves `Equatable` then the overall tuple itself conforms to `Equatable`.

```swift
// Ok, Int is Equatable thus the tuples are Equatable
(1, 2, 3) == (1, 2, 3) // true

struct EmptyStruct {}

// error: type '(EmptyStruct, Int, Int)' does not conform to protocol 'Equatable'
// note: value of type 'EmptyStruct' does not conform to protocol 'Equatable',
//       preventing conformance of '(EmptyStruct, Int, Int)' to 'Equatable'
(EmptyStruct(), 1, 2) == (EmptyStruct(), 1, 2)
```

It's also important to note that this conformance does not take into account the tuple labels in consideration for equality. If both tuples have the same element types, then they can be compared for equality. This mimics the current behavior of the operations introduced in [SE-0015](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0015-tuple-comparison-operators.md).

```swift
// We don't take into account the labels for equality.
(x: 0, y: 0) == (0, 0) // true
```

### Comparable

Comparable conformance for tuples works just like `Equatable`, if all the elements themselves are `Comparable`, then the tuple itself is `Comparable`. Comparing a tuple to a tuple works elementwise:

> Look at the first element, if they are equal move to the second element.
Repeat until we find elements that are not equal and compare them.

If all of the elements are equal, we cannot compare them, thus the result is `false`. Of course if we're using `<=` or `>=` and the tuples are exactly the same then the output would be `true`.

```swift
let origin = (x: 0, y: 0)
let randomPoint = (x: Int.random(in: 1 ... 10), y: Int.random(in: 1 ... 10))

// In this case, the first element of origin is 0 and the first element
// of randomPoint is somewhere between 1 and 10, so they are not equal.
// origin's element is less than randomPoint's, thus true.
print(origin < randomPoint) // true
```

Just like in `Equatable`, the comparison operations do not take tuple labels into consideration when determining comparability. This mimics the current behavior of the operations introduced in [SE-0015](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0015-tuple-comparison-operators.md).

```swift
// We don't take into account the labels for comparison.
(x: 0, y: 0) < (1, 0) // true
```

### Hashable

The same rule applies to `Hashable` as it does for `Comparable` and `Equatable`, if all the elements are `Hashable` then the tuple itself is `Hashable`. When hashing a value of a tuple, all of the elements are combined into the hasher to produce the tuple's hash value. Now that tuples are `Hashable`, one can make a set of tuples or create dictionaries with tuple keys:

```swift
let points = [(x: 0, y: 0), (x: 1, y: 2), (x: 0, y: 0)]
let uniquePoints = Set(points)

// Create a grid system to hold game entities.
var grid = [(x: Int, y: Int): Entity]()

for point in uniquePoints {
    grid[point]?.move(up: 10)
}
```

Once again, `Hashable` doesn't take tuple element labels into consideration when evaluating the hash value of a tuple. Because of this, one is able to index into a set or dictionary with an unlabeled tuple and retrieve elements whose keys were labeled tuples:

```swift
// We don't take into account the labels for hash value.
(x: 0, y: 0).hashValue == (0, 0).hashValue // true

grid[(x: 100, y: 200)] = Entity(name: "Pam")

print(grid[(100, 200)]) // Entity(name: "Pam")
```

## Source compatibility

These are completely new conformances to tuples, thus source compatibility is unaffected as they were previously not able to conform to protocols.

## Effect on ABI stability

The conformances to `Equatable`, `Comparable`, and `Hashable` are all additive to the ABI. While at the time of writing this, there is no way to spell a new conformance to an existing type. However, these conformances are being implemented within the runtime which allows us to backward deploy these conformance to Swift 5.0, 5.1, and 5.2 clients. Because these are special conformances being added before other language features allows us to create real conformances, there is a level of runtime support needed to enable these conformances to work properly. Going forward this means we'll need to keep the entry points needed for these to work even after tuples are able to properly conform to protocols.

## Alternatives considered

Besides not doing this entirely, the only alternative here is whether or not we should hold off on this before we get proper protocol conformances for tuples which allow them to conform to any protocol. Doing this now requires a lot of builtin machinery in the compiler which some may refer to as technical debt. While I agree with this statement, I don't believe we should be holding off on features like this that many are naturally reaching for until bigger and more complex proposals that allow this feature to natively exist in Swift. I also believe it is none of the user's concern for what technical debt is added to the compiler that allows them to write the Swift code that they feel comfortable writing. In any case, the technical debt to be had here should only be the changes to the runtime (or at least the symbols needed) which allow this feature to work.

## Future Directions

With this change, other conformances such as `Codable` might make sense for tuples as well. It also makes sense to implement other conformances for other structural types in the language such as metatypes being `Hashable`, existentials being `Equatable` and `Hashable`, etc.

In the future when we have proper tuple extensions along with variadic generics and such, implementing these conformances for tuples will be trivial and I imagine the standard library will come with these conformances for tuples. When that happens all future usage of those conformances will use the standard library's implementation, but older clients that have been compiled with this implementation will continue using it as normal.
