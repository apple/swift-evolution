# Forward-scan matching for trailing closures

* Proposal: [SE-0286](0286-forward-scan-trailing-closures.md)
* Author: [Doug Gregor](https://github.com/DougGregor)
* Review Manager: [John McCall](https://github.com/rjmccall)
* Status: **Implemented (Swift 5.3)**
* Upcoming Feature Flag: `ForwardTrailingClosures` (implemented in Swift 5.8)
* Implementation: [apple/swift#33092](https://github.com/apple/swift/pull/33092)
* Toolchains: [Linux](https://ci.swift.org/job/swift-PR-toolchain-Linux/404//artifact/branch-master/swift-PR-33092-404-ubuntu16.04.tar.gz), [macOS](https://ci.swift.org/job/swift-PR-toolchain-osx/579//artifact/branch-master/swift-PR-33092-579-osx.tar.gz)
* Discussion: ([Pitch #1](https://forums.swift.org/t/pitch-1-forward-scan-matching-for-trailing-closures-source-breaking/38162)), ([Pitch #2](https://forums.swift.org/t/pitch-2-forward-scan-matching-for-trailing-closures/38491))
* Previous Revision: [1](https://github.com/swiftlang/swift-evolution/blob/07bcb908125e1795a08d47391b5d866eb782639e/proposals/0286-forward-scan-trailing-closures.md)
* Review: ([Review](https://forums.swift.org/t/se-0286-forward-scan-for-trailing-closures/38529)), ([Acceptance](https://forums.swift.org/t/accepted-with-modifications-se-0286-forward-scan-for-trailing-closures/38836))

## Introduction

[SE-0279 "Multiple Trailing Closures"](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0279-multiple-trailing-closures.md) threaded the needle between getting the syntax we wanted for multiple trailing closures without breaking source compatibility. One aspect of that compromise was to extend (rather than replace) the existing rule for matching a trailing closure to a parameter by scanning *backward* from the end of the parameter list.

However, the backward-scan matching rule makes it hard to write good API that uses trailing closures, especially multiple trailing closures. This proposal replaces the backward scan with a forward scan wherever possible, which is simpler, more in line with normal argument matching in a call, and works better for APIs that support trailing closures (whether single or multiple) and default arguments. This change introduces a *minor source break* for code involving multiple, defaulted closure parameters, but that source break is staged over multiple Swift versions.

## Motivation

Several folks noted the downsides of the "backward" matching rule. The rule itself is described in the [detailed design](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0279-multiple-trailing-closures.md#detailed-design) section of SE-0279 (search for "backward"). To understand the problem with the backward rule, let's try to declare the UIView [`animate(withDuration:animations:completion:)`](https://developer.apple.com/documentation/uikit/uiview/1622515-animate) method in the obvious way to make use of SE-0279:

```swift
class func animate(
    withDuration duration: TimeInterval, 
    animations: @escaping () -> Void, 
    completion: ((Bool) -> Void)? = nil
)
```

SE-0279 matches the named trailing closure arguments backward, matching the last (labeled) trailing closure argument from the back of the parameter list, then proceeding to move to earlier trailing closures and function parameters. Consider the following example (straight from SE-0279):

```swift
UIView.animate(withDuration: 0.3) {
  self.view.alpha = 0
} completion: { _ in
  self.view.removeFromSuperview()
}
```

The `completion:` trailing closure matches the last parameter, and the unnamed trailing closure matches `animations:`. The backward rule worked fine here.

However, things fall apart when a single (therefore unnamed) trailing closure is provided to this API:

```swift
UIView.animate(withDuration: 0.3) {
  self.view.alpha = 0
}
```

Now, the backward rule matches the unnamed trailing closure to `completion:`. The compiler produces an error:

```
error: missing argument for parameter 'animations' in call
  animate(withDuration: 0.3) {
```

Note that the "real" UIView API actually has two different methods---`animate(withDuration:animations:completion:)` and `animate(withDuration:animations:)`---where the latter looks like this:

```swift
class func animate(
    withDuration duration: TimeInterval, 
    animations: @escaping () -> Void
)
```

This second overload only has a closure argument, so the backward-matching rule handles the single-trailing-closure case. These overloads exist because these UIView APIs were imported from Objective-C, which does not have default arguments. A new Swift API would not be written this way---except that SE-0279 forces it due to the backward-matching rule.

## Proposed solution

The idea of the forward-scan matching rule is to match trailing closure arguments to parameters in the same forward, left-to-right manner that other arguments are matched to parameters. The unlabeled trailing closure will be matched to the next parameter that is either unlabeled or has a declared type that structurally resembles a function type (defined below). For the example above, this means the following:

```swift
UIView.animate(withDuration: 0.3) {
  self.view.alpha = 0
}
// equivalent to
UIView.animate(withDuration: 0.3, animations: {
  self.view.alpha = 0
})
```

and

```swift
UIView.animate(withDuration: 0.3) {
  self.view.alpha = 0
} completion: { _ in
  self.view.removeFromSuperview()
}
// equivalent to
UIView.animate(withDuration: 0.3, animations: {
  self.view.alpha = 0
}, completion: { _ in
  self.view.removeFromSuperview()
})
```

Note that the unlabeled trailing closure matches `animations:` in both cases; specifying additional trailing closures fills out later parameters but cannot shift the unlabeled trailing closure to an earlier parameter.

Note that you can still have the unlabeled trailing closure match a later parameter, by specifying earlier ones:

```swift
UIView.animate(withDuration: 0.3, animations: self.doAnimation) { _ in
  self.view.removeFromSuperview()
}
// equivalent to
UIView.animate(withDuration: 0.3, animations: self.doAnimation, completion: { _ in
  self.view.removeFromSuperview()
})
```

This is both a consequence of forward matching and also a necessity for source compatibility.

### Structural resemblance to a function type

When a function parameter does not require an argument (e.g., because it is variadic or has a default argument), the call site can skip mentioning that parameter entirely, and the default will be used instead (e.g., an empty variadic argument or the specified default argument). The matching of arguments to parameters tends to rely on argument labels to determine when a particular parameter has been skipped. For example:

```swift
func nameMatchingExample(x: Int = 1, y: Int = 2, z: Int = 3) { }

nameMatchingExample(x: 5) // equivalent to nameMatchingExample(x: 5, y: 2, z: 3)
nameMatchingExample(y: 4) // equivalent to nameMatchingExample(x: 1, y: 4, z: 3)
nameMatchingExample(x: -1, z: -3) // equivalent to nameMatchingExample(x: -1, y: 2, z: -3)
```

The unlabeled trailing closure ignores the (otherwise required) argument label, which would prevent the use of argument labels for deciding which parameter should be matched with the unlabeled trailing closure. Let's bring that back to the UIView example by adding a default argument to `withDuration:`

```swift
class func animate(
    withDuration duration: TimeInterval = 1.0, 
    animations: @escaping () -> Void, 
    completion: ((Bool) -> Void)? = nil
)
```

Consider a call:

```swift
UIView.animate {
  self.view.alpha = 0
}
```

The first parameter is `withDuration`, but there is no argument in parentheses. Unlabeled trailing closures ignore the parameter name, so without some additional rule, the unlabeled trailing closure would try to match `withDuration:` and this call would be ill-formed.

The forward-scan matching rule skips over any parameters that do not "structurally resemble" a function type. A parameter structurally resembles a function type if both of the following are true:

* The parameter is not  `inout`
* The adjusted type of the parameter (defined below) is a function type

The adjusted type of the parameter is the parameter's type as declared in the function, looking through any type aliases whenever they appear, and performing three adjustments:

* If the parameter is an  `@autoclosure` , use the result type of the parameter's declared (function) type, before performing the second adjustment.
* If the parameter is variadic, looking at the element type of the (implied) array type.
* Remove all outer "optional" types.

Following this rule, the `withDuration` parameter (a `TimeInterval`) does not resemble a function type. However, `@escaping () -> Void` does, so the unlabeled trailing closure matches `animations`. `@autoclosure () -> ((Int) -> Int)` and `((Int) -> Int)?` would also resemble a function type.

### Mitigating the source compatibility impact (all language versions)

The forward-scanning rule, as described above, is source-breaking. A run over Swift's [source compatibility suite](https://swift.org/source-compatibility/) with this change enabled in all language modes turned up source compatibility breaks in three projects. The first problem occurs with a SwiftUI API [`View.sheet(isPresented:onDismiss:content:)`](https://developer.apple.com/documentation/swiftui/view/sheet(ispresented:ondismiss:content:)):

```swift
func sheet(
  isPresented: Binding<Bool>,
  onDismiss: (() -> Void)? = nil,
  content: @escaping () -> Content
) -> some View
```

Note that `onDismiss` and `content` both structurally resemble a function type. This API fits well with the backward-matching rule, because the unlabeled trailing closure in the following example is always ascribed to `content:`. The `onDismiss:` argument gets the default argument `nil`:

```swift
sheet(isPresented: $isPresented) { Text("Hello") }
```

With the forward-scanning rule, the unlabeled trailing closure matches the `onDismiss:` parameter, and there is no suitable argument for `content:`. Therefore, the well-formed code above would be rejected by the rule as proposed above.

However, it is clear from the function signature that (1) `onDismiss:` could have used the default argument, and (2) `content:` therefore won't have an argument if it is not paired with the unlabeled trailing closure. We can turn this into an heuristic to accept more existing code, reducing the source breaking impact of the proposal. Specifically, if

* the parameter that would match the unlabeled trailing closure
argument does not require an argument (because it is variadic or has a default argument), and
* there are parameters  *after*  that parameter that require an argument, up until the first parameter whose label matches that of the *next* trailing closure (if any)

then do not match the unlabeled trailing closure to that parameter. Instead, skip it and examine the next parameter to see if that should be matched against the unlabeled trailing closure. For the `View.sheet(isPresented:onDismiss:content:)` API, this means that `onDismiss`, which has a default argument, will be skipped in the forward match so that the unlabeled trailing closure will match `content:`, allowing this code to continue to compile correctly.

This heuristic is remarkably effective: in addition to fixing 2 of the 3 failures from the Swift source compatibility suite (the remaining failure will be discussed below), it resolved most of the failures we observed in a separate (larger) testsuite comprising a couple of million lines of Swift.

One practical effect of this heuristic is that it makes the forward scan as proposed here produce the same results as the existing backward scan in many, many more cases.

### Mitigating the source compatibility impact (Swift < 6)

Even with the heuristic, the forward-scan matching rule will still fail to compile some existing code, and can change the meaning of some code, when there are multiple, defaulted parameters of closure type. As an example, the remaining source compatibility failure in the Swift source compatibility suite, a project called [ModelAssistant](https://github.com/ssamadgh/ModelAssistant), is due to [this API](https://github.com/ssamadgh/ModelAssistant/blob/c96335280a3aba5f8e14955ecaf38dc25a0872b6/Source/Libraries/AOperation/Observers/BlockObserver.swift#L22-L26):

```swift
init(
    startHandler: ((AOperation) -> Void)? = nil,
    produceHandler: ((AOperation, Foundation.Operation) -> Void)? = nil,
    finishHandler: ((AOperation, [NSError]) -> Void)? = nil
) {
    self.startHandler = startHandler
    self.produceHandler = produceHandler
    self.finishHandler = finishHandler
}
```

Note that this API takes three closure parameters. The (existing) backward scan will match `finishHandler:`, while the forward scan will match `startHandler:`. The heuristic described in the previous section does not apply, because all of the closure parameters have default arguments. Existing code that uses trailing closures with this API will break.

Note that this API interacts poorly with SE-0279 multiple trailing closures, because the unlabeled trailing closure "moves" backwards as additional trailing closures are provided at the call site:

```swift
// SE-0279 backward scan behavior
BlockObserver { (operation, errors) in
  print("finishHandler!")
}

// label finishHandler, unlabeled moves "back" to produceHandler

BlockObserver { (aOperation, foundationOperation) in
  print("produceHandler!")
} finishHandler: { (operation, errors) in
  print("finishHandler!")
}

// label produceHandler, unlabeled moves "back" to startHandler
BlockObserver { aOperation in 
  print("startHandler!")
} produceHandler: { (aOperation, foundationOperation) in
  print("produceHandler!")
} finishHandler: { (operation, errors) in
  print("finishHandler!")
}
```

The forward scan provides a consistent unlabeled trailing closure anchor, and later (labeled) trailing closures can be tacked on:

```swift
// Proposed forward scan
BlockObserver { aOperation in 
  print("startHandler!") {
}

// add another
BlockObserver { aOperation in 
  print("startHandler!")
} produceHandler: { (aOperation, foundationOperation) in
  print("produceHandler!")
}

// specify everything 
BlockObserver { aOperation in 
  print("startHandler!")
} produceHandler: { (aOperation, foundationOperation) in
  print("produceHandler!")
} finishHandler: { (operation, errors) in
  print("finishHandler!")
}

// skip the middle one!
BlockObserver { aOperation in 
  print("startHandler!")
} finishHandler: { (operation, errors) in
  print("finishHandler!")
}
```

The forward-scan matching rule provides more predictable results, making it easier to understand how to use this API properly. However, maintaining backward compatibility requires that the backward scan be considered in places where it differs from the forward scan.

To address this remaining source compatibility problem, Swift minor versions (prior to Swift 6) shall implement an additional rule for calls that involve a single (unlabeled) trailing closure. If the forward and backward-scan rules produce *different* assignments of arguments to parameters, then the Swift compiler will attempt both: if only one succeeds, use it. If both succeed, prefer the backward-scanning rule (for source compatibility reasons) and produce a warning about the use of the backward scan. For example:

```swift
BlockObserver { (operation, errors) in
  print("finishHandler!")
}
```

Here, the forward scan fails to type-check, because the closure accepts two parameters whereas `startHandler` accepts a single parameter. Therefore, the backward scan is selected, maintaining source compatibility, and produces a warning with a Fix-It to make the trailing closure a regular argument:

```
warning: backward matching of the unlabeled trailing closure is deprecated; label the argument with 'finishHandler' to suppress this warning
BlockObserver { (operation, errors) in
              ^
             (finishHandler:
```

If there truly is an ambiguity, where both the forward scan and backward scan type-check but would do so differently, we prefer the backward scan to maintain source compatibility:

```swift
func trailingClosureBothDirections(
  f: (Int, Int) -> Int = { $0 + $1 }, g: (Int, Int) -> Int = { $0 - $1 }
) { }
trailingClosureBothDirections { $0 * $1 }
```

Here, the forward scan would bind the trailing closure to `f:` (for Swift 6 and newer) while the backward scan would bind the trailing closure to `g:` (for Swift < 6). The same warning will apply when the backward scan result is chosen, with a Fix-It to rewrite the code to:

```swift
trailingClosureBothDirections(g: { $0 * $1 })
```

This suppresses the warning and eliminates the ambiguity, so the code behaves the same across all overload sets.

The Swift 6 and newer behavior can be enabled in existing language modes with the [upcoming feature flag](0362-piecemeal-future-features.md) `ForwardTrailingClosures`.

### Workaround via overload sets

APIs like the above that depend on the backward scan can be reworked to provide the same client API. The basic technique involves removing the default arguments, then adding additional overloads to create the same effect. For example, drop the default argument of `finishHandler` so that the heuristic will kick in to fix calls with a single unlabeled trailing closure:

```swift
init(
    startHandler: ((AOperation) -> Void)? = nil,
    produceHandler: ((AOperation, Foundation.Operation) -> Void)? = nil,
    finishHandler: ((AOperation, [NSError]) -> Void)?
) {
    self.startHandler = startHandler
    self.produceHandler = produceHandler
    self.finishHandler = finishHandler
}
```

One can then add overloads to handle other cases, e.g., the zero-argument case:

```swift
init() {
  self.init(startHandler: nil, produceHandler: nil, finishHandler: nil)
}
```

## Future directions

The proposal specifies that the "backward" scan be removed in Swift 6, which introduces a small source break that is staged in over time. However, the heuristic (that skips matching the unnamed trailing closure argument to a parameter that doesn't require an argument when the unnamed trailing closure is needed to match a later parameter) is retained. However, some future language version (Swift 6 or even later) might accept more source breakage by removing this heuristic---leaving only the forward scan in place---and find a better way to express APIs such as `View.sheet(isPresented:onDismiss:content:)` in the language. Possibilities include (but are not limited to):

* A parameter attribute `@noTrailingClosure` that prevents the use of trailing closure syntax for a given parameter entirely.
* Eliminating the allowance for matching the first (unlabeled) trailing closure to a parameter that has an argument label, so normal argument matching rules would apply.
* Allowing an argument label on the first trailing closure to let the caller select which parameter to match explicitly.

This proposal leaves open all of these possibilities, and makes their changes less drastic because each of the ideas involves moving to a forward-scan matching rule. As such, this proposal makes SE-0279's multiple trailing closures immediately useful with minimal (or no) source breakage, paving the way for more significant changes if we so choose in the future.

## Revision history

* **Version 2**: Improved source compatibility by performing both the forward and backward scans in Swift < 6 mode ([originally suggested](https://forums.swift.org/t/se-0286-forward-scan-for-trailing-closures/38529/9) by Pavel Yaskevich) and adopting the [specific proposal](https://forums.swift.org/t/se-0286-forward-scan-for-trailing-closures/38529/30) from Xiaodi Wu to prefer the backward scan result in Swift < 6 when the two scans differ.
