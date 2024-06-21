# Make non-escaping closures the default

* Proposal: [SE-0103](0103-make-noescape-default.md)
* Author: [Trent Nadeau](https://github.com/tanadeau)
* Review Manager: [Chris Lattner](https://github.com/lattner)
* Status: **Implemented (Swift 3.0)**
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-se-0103-make-non-escaping-closures-the-default/3212)
* Bug: [SR-1952](https://bugs.swift.org/browse/SR-1952)
* Previous Revision: [1](https://github.com/swiftlang/swift-evolution/blob/833afd64b5d24a777fe2c42800d4b4dcd52bb487/proposals/0103-make-noescape-default.md)

## Introduction

The current default of closure arguments to functions (i.e., arguments to functions that themselves have function type such as `(T) -> U`) is to be "escaping", meaning they can escape the function body such as saving it to a field in a struct or a global variable. In order to say that a closure argument cannot possibly escape the function body ("non-escaping"), the developer must explicitly add an `@noescape` annotation to the argument type.

This proposal switches the default to be non-escaping and requires an `@escaping` annotation if a closure argument can escape the function body. Since the escaping case can be statically detected, this annotation can be added via an error with a fixit. Other annotations that have consequences for escape semantics (e.g., `@autoclosure(escaping)`) will be altered to make use of the new `@escaping` annotation.

Swift-evolution threads:

* [Make non-escaping closures the default](https://forums.swift.org/t/proposal-make-non-escaping-closures-the-default/2889)

## Motivation

Per Chris Lattner [on swift-evolution](https://forums.swift.org/t/rejected-se-0097-normalizing-naming-for-negative-attributes/2854/2):

> To provide some more details, this approach has the following advantages:
>
> - Most functional algorithms written in pure Swift will benefit because they are naturally noescape.  The core team feels that this will reduce the boilerplate involved with writing these algorithms.
>
> - The compiler has enough logic in it to provide a great QoI experience when a developer doesn’t think about escaping, and tries to escape a closure - it can provide a fixit that suggests adding @escaping.
>
> - Recent changes (to disallow escaping closures to close over an inout parameter) are pushing the language to prefer noescape closures.  noescape closures have also always been the preferred default, since they eliminate a class of retain cycle issues.
>
> - "@autoclosure(escaping)" can be simplified and standardized to "@autoclosure @escaping”

## Detailed design

The `@noescape` annotation is removed from the language. The compiler will emit an error with a fixit to remove the annotation.

The compiler will emit an error if a closure argument is found to possibly escape the function body. In order to silence the warning, the developer must add, manually or via fixit, the `@escaping` annotation to the argument type.

The compiler's semantic analysis implementation can be simplified as the more constrained escaping case that conflicts with other attributes is now no longer the default.

The standard library should be changed to use the new default whenever possible by removing all uses of `@noescape` and only adding `@escaping` where the compiler detects the need.

A helper that will convert a non-escaping closure to an "escaping" one will be added to the standard library. This helper is useful when a function has a non-escaping closure argument (the closure is called before it returns), but the closure may be used as an argument to a function requiring an escaping closure, such as various `LazySequence` methods. The helper should verify that the closure has not actually escaped and trap if it does. It should have the following signature:

```swift
func withoutActuallyEscaping<ClosureType, ResultType>(
    _ closure: ClosureType,
    do: (fn: @escaping ClosureType) throws -> ResultType) rethrows -> ResultType {
  // ...
}
```

An example of its use:

```swift
func yourFunction(fn: (Int) -> Int) {  // fn defaults to non-escaping.
  withoutActuallyEscaping(fn) { fn in  // fn is now marked @escaping inside the closure
    // ...
    somearray.lazy.map(fn)             // pass fn to something that is notationally @escaping
    // ...
  }
}
```

### Imported C/Objective-C APIs

Per the Core Team, most Cocoa closure/block parameters are escaping (e.g., delegates). As such the Clang importer will automatically add the `@escaping` annotation to closure/block parameters encountered in imported Objective-C APIs unless they are explicitly marked with the Clang `((noescape))` attribute. This will also be done with imported C APIs with function pointer or block parameters.

## Impact on existing code

Existing code using the `@noescape` attribute will need to be migrated to remove the attribute since it will be the default. In addition, the compiler will need to detect escaping closures that are not marked with `@escaping` and create an error with a fixit to add the required attribute.

Uses of `@autoclosure(escaping)` must be changed to `@autoclosure @escaping`.

There should be few, if any, changes required for uses of Cocoa APIs as these will be mostly marked as `@escaping`, and escaping closure arguments are *more* constrained than non-escaping ones.

## Future directions

The `@noescape(once)` annotation proposed in [SE-0073](0073-noescape-once.md) would, if some future version is accepted, just become `@once`.

## Alternatives considered

1. Keep the existing semantics and the `@noescape` attribute as they are now.
2. Keep the existing semantics but change the attribute name to `@nonescaping`.

## Acknowledgements

Thanks to Chris Lattner, John McCall, and anyone else who reviewed and contributed to this proposal.
