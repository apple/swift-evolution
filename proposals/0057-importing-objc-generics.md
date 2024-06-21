# Importing Objective-C Lightweight Generics

* Proposal: [SE-0057](0057-importing-objc-generics.md)
* Author: [Doug Gregor](https://github.com/DougGregor)
* Review Manager: [Chris Lattner](https://github.com/lattner)
* Status: **Implemented (Swift 3.0)**
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-se-0057-importing-objective-c-lightweight-generics/2185)
* Previous Revision: [Originally Accepted Proposal](https://github.com/swiftlang/swift-evolution/blob/3abbed3edd12dd21061181993df7952665d660dd/proposals/0057-importing-objc-generics.md)


## Introduction

Objective-C's *lightweight generics* feature allows Objective-C
classes to be parameterized on the types they work with, similarly to
Swift's generics syntax. Their adoption in Foundation's collection
classes allow Objective-C APIs to be bridged more effectively into
Swift. For example, an `NSArray<NSString *> *` bridges to `[String]`
rather than the far-weaker `[AnyObject]`. However, parameterized
Objective-C classes lose their type parameters when they are imported
into Swift, so uses of type parameters outside of bridged, typed
collections (`NSArray`, `NSDictionary`, `NSSet`) don't benefit in
Swift. This proposal introduces a way to import the type parameters of
Objective-C classes into Swift.

Swift-evolution thread: [here](https://forums.swift.org/t/proposal-draft-importing-objective-c-lightweight-generics/991)

## Motivation

Cocoa and Cocoa Touch include a number of APIs that have adopted
Objective-C lightweight generics to improve static type safety and
expressiveness. However, because the type parameters are lost when
these APIs are imported into Swift, they are effectively *less* type
safe in Swift than in Objective-C, a situation we clearly cannot
abide. This proposal aims to improve the projection of these
Objective-C APIs in Swift.

## Proposed solution

A parameterized class written in Objective-C will be imported into
Swift as a generic class with the same number of type parameters. The
bounds on the type parameters in Objective-C will be translated into
requirements on the generic type parameters in Swift:

* The generic type parameters in Swift will always be class-bound,
  i.e., the generic class will have the requirement `T : AnyObject`.
* If the bound includes a class type (e.g., `T : NSValue *` in
  Objective-C), the generic Swift class will have the corresponding
  superclass requirement (`T : NSValue`).
* If the bound includes protocol qualification (e.g., `T :
  id<NSCopying>` in Objective-C), each protocol bound is turned into
  a conformance requirement (`T : NSCopying`) on the generic Swift
  class.

The following Objective-C code:

```
@interface MySet<T : id<NSCopying>> : NSObject
-(MySet<T> *)unionWithSet:(MySet<T> *)otherSet;
@end

@interface MySomething : NSObject
- (MySet<NSValue *> *)valueSet;
@end
```

will be imported as:

```swift
class MySet<T : NSCopying> : NSObject {
  func unionWithSet(otherSet: MySet<T>) -> MySet<T>
}

class MySomething : NSObject {
  func valueSet() -> MySet<NSValue>
}
```

### Importing unspecialized types

When importing an unspecialized Objective-C type into Swift, we
will substitute the bounds for the type arguments. For example:

```
@interface MySomething (ObjectSet)
- (MySet *)objectSet;    // note: no type arguments to MySet
@end
```

will be imported as:

```swift
extension MySomething {
  func objectSet() -> MySet<NSCopying> // note: uses the type bound
}
```

### Restrictions on uses of Objective-C parameterized classes

While the Swift and Objective-C generics systems look similar on the
surface, they use fundamentally different semantic
models. Specifically, Objective-C lightweight generics are based on
*type erasure*, so we cannot in general recover the type arguments
from the metaclass of an Objective-C parameterized class (i.e.,
because `MySet`, `MySet<NSString *>`, and `MySet<NSNumber *>` all
share a metaclass). This leads to several restrictions:

* Downcasting to an instance or metatype of a parameterized
  Objective-C class is inherently uncheckable, so we place limits on
  such casts. For example,

  ```swift
  let obj: AnyObject = ...
  if let set1 = obj as? MySet<NSCopying> {
    // okay: every MySet is a MySet<NSCopying> by construction, so
    // we're just checking that this is a 'MySet'.
  }

  if let set2 = obj as? MySet<NSNumber> {
    // error: conditional cast to specialized Objective-C instance
    // doesn't check type argument 'NSNumber'
  }

  let set3 = obj as! MySet<NSNumber> // okay: we assert that it is safe

  if let set4 = obj as? MySet<NSCopying> {
    let set5 = set4 as! MySet<NSNumber> // here's how to get a MySet<NSNumber>
  }
  ```

* By default, extensions of parameterized Objective-C classes cannot reference the
  type parameters in any way. For example:

  ```swift
  extension MySet {
    func someNewMethod(x: T) { ... } // error: cannot use `T`.
  }
  ```
  
### Subclassing parameterized Objective-C classes from Swift

When subclassing a parameterized Objective-C class from Swift, the
Swift compiler has the complete type metadata required, because it is
stored in the (Swift) type metadata.

## Impact on existing code

In Swift 2, parameterized Objective-C classes are imported as
non-parameterized classes. Importing them as parameterized classes
will break any existing references to the affecting APIs. There are a
handful of cases where type inference may paper over the problems:

```swift
let array: NSArray = ["hello", "world"] // okay, infer NSArray<NSString>
// old

var mutArray = NSMutableArray() // error: need type arguments for NSMutableArray
```

A migrator could introduce the type bounds as arguments, e.g.,
`NSArray` would get migrated to `NSArray<AnyObject>`. It is not the
best migration---many developers would likely want to tighten up the
bounds to improve their Swift code---but it should migrate existing
code.

## Alternatives considered

The only major alternative design involves bringing type erasure into
the Swift generics system as an alternative implementation. It would
lift the restrictions on extensions of parameterized Objective-C
classes by treating type parameters in such contexts as the type
bounds:

```swift
extension MySet {
  func someNewMethod(x: T) { ... } // okay: `T` is treated like `NSCopying`
}
```

Doing so could allow the use of "unspecialized" generic types within
Swift, e.g., `NSMutableArray` with no type bounds (possibly spelled
`NSMutableArray<*>`), which would more accurately represent
Objective-C semantics in Swift.

However, doing so comes at the cost of having two incompatible
generics models implemented in Swift, which produces both a high
conceptual burden as well as a high implementation cost. The proposed
solution implies less implementation cost and puts the limitations on
what one can express when working with parameterized Objective-C
classes without fundamentally changing the Swift model.

## Revision history

The [originally accepted proposal](https://github.com/swiftlang/swift-evolution/blob/3abbed3edd12dd21061181993df7952665d660dd/proposals/0057-importing-objc-generics.md)
included a mechanism by which Objective-C generic classes could implement
an informal protocol to provide reified generic arguments to Swift clients:

> A parameterized Objective-C class can opt in to providing information
> about its type argument by implementing a method
> `classForGenericArgumentAtIndex:` either as a class method (for the first case
> described above) or as an instance method (for the second case
> described above). The method returns the metaclass for the type
> argument at the given, zero-based index.

As of Swift 5, this feature has not been implemented, so it has been withdrawn
from the proposal.
