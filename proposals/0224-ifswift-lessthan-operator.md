# Support 'less than' operator in compilation conditions

* Proposal: [SE-0224](0224-ifswift-lessthan-operator.md)
* Authors: [Daniel Martín](https://github.com/danielmartin)
* Review Manager: [Ted Kremenek](https://github.com/tkremenek)
* Status: **Implemented (Swift 5.0)**
* Decision Notes: [Rationale](https://forums.swift.org/t/se-0224-support-less-than-operator-in-compilation-conditions/15213/5)
* Bugs: [SR-6852](https://bugs.swift.org/browse/SR-6852)
* Implementations: [apple/swift#14503](https://github.com/apple/swift/pull/14503) (Stale?), [apple/swift#17690](https://github.com/apple/swift/pull/17960)

## Introduction

This proposal augments the functionality implemented for proposal
[SE-0020](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0020-if-swift-version.md)
with the introduction of a new valid operator in compilation
condition: "<". The aim is that the syntax `#if swift(<4.2)` is
supported by the language.

Swift-evolution thread: [Discussion thread topic for that proposal](https://forums.swift.org/t/support-for-more-operators-in-if-swift-build-configuration-option/14343)

## Motivation

The main motivation for introducing a new "<" operator in compilation
conditions is to be able to write Swift code that is easier to read.

For example, if we want to only compile some piece of code if the
Swift version is less than 4.2, right now we have to write the following
code:

```swift
#if !swift(>=4.2)
// This will only be executed if the Swift version is less than 4.2.
#endif

#if !compiler(>=4.2)
// This will only be executed if the Swift compiler version is less than 4.2.
#endif
```

With the introduction of support for the "<" unary operator, the
refactored code would be more clear and readable:

```swift
#if swift(<4.2)
// This will only be executed if the Swift version is less than 4.2.
#endif

#if compiler(<4.2)
// This will only be executed if the Swift compiler version is less than 4.2.
#endif
```

In the former snippet, the `!` can be easily missed in a code
review. The latter snippet reads more like plain English.

Support for other operators like "<=" and ">" is not desired, as they
make a statement about future releases and they don't account for
patch releases. That means that client code will need to be updated if
a patch release didn't fix a particular issue with the compiler, for
example.

## Proposed solution

The solution is small change in the parser so that the operator "<" is
supported for both the `#if swift` and `#if compiler` conditions. Diagnostic
messages about invalid unary operators must be updated as well.

## Detailed design

The place in the parser where `#if swift(...)` is parsed is
`ParseIfConfig.cpp`. There are two classes that will require
modification: `ValidateIfConfigCondition`, to take into account the
"<" operator, and `EvaluateIfConfigCondition`, to actually evaluate
the new operator semantics. A new '<' operator for `Version` will also
need to be implemented.

The diagnostic message when the operator is not valid also needs to
change. I propose changing it from

```
unexpected platform condition argument: expected a unary comparison, such as '>=2.2'
```

to

```
unexpected platform condition argument: expected a unary comparison '>=' or '<'; for example, '>=2.2' or '<2.2'
```

## Source compatibility

This has no effect in source compatibility.

## Effect on ABI stability

This has no effect in ABI stability.

## Effect on API resilience

This has no effect in API resilience.
