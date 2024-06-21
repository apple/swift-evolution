# Swift 2.2 - Released on March 21, 2016

[This release](https://swift.org/blog/swift-2-2-released/) focused on fixing
bugs, improving quality-of-implementation (QoI)
with better warnings and diagnostics, improving compile times, and improving
performance.  It put some finishing touches on features introduced in Swift 2.0, 
and included some small additive features that don't break Swift code or
fundamentally change the way Swift is used. As a step toward Swift 3, it
introduced warnings about upcoming source-incompatible changes in Swift 3
so that users can begin migrating their code sooner.

Aside from warnings, a major goal of this release was to be as source compatible
as practical with Swift 2.0.

## Evolution proposals included in Swift 2.2

* [SE-0001: Allow (most) keywords as argument labels](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0001-keywords-as-argument-labels.md)
* [SE-0015: Tuple comparison operators](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0015-tuple-comparison-operators.md)
* [SE-0014: Constraining `AnySequence.init`](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0014-constrained-AnySequence.md)
* [SE-0011: Replace `typealias` keyword with `associatedtype` for associated type declarations](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0011-replace-typealias-associated.md)
* [SE-0021: Naming Functions with Argument Labels](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0021-generalized-naming.md)
* [SE-0022: Referencing the Objective-C selector of a method](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0022-objc-selectors.md)
* [SE-0020: Swift Language Version Build Configuration](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0020-if-swift-version.md)
