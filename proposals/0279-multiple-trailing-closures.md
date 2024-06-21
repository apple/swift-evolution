# Multiple Trailing Closures

* Proposal: [SE-0279](0279-multiple-trailing-closures.md)
* Authors: [Kyle Macomber](https://github.com/kylemacomber), [Pavel Yaskevich](https://github.com/xedin), [Doug Gregor](https://github.com/douggregor), [John McCall](https://github.com/rjmccall)
* Review Manager: [Ben Cohen](https://github.com/airspeedswift)
* Status: **Implemented (Swift 5.3)**
* Previous Revisions: [1st](https://github.com/swiftlang/swift-evolution/blob/d923209a05c3c38c8b735510cf1525d27ed4bd14/proposals/0279-multiple-trailing-closures.md)
* Reviews: [1st](https://forums.swift.org/t/se-0279-multiple-trailing-closures/34255),
           [2nd](https://forums.swift.org/t/se-0279-multiple-trailing-closures-amended/35435),
           [3rd](https://forums.swift.org/t/accepted-se-0279-multiple-trailing-closures/36141)
* Implementation: [apple/swift#31052](https://github.com/apple/swift/pull/31052)

## Motivation

Since its inception, Swift has supported *trailing closure* syntax: a bit of syntactic sugar that lets you "pop" the final argument to a function out of the parentheses when it's a closure.

This example uses [`UIView.animate(withDuration:animations:)`](https://developer.apple.com/documentation/uikit/uiview/1622418-animate) to fade out a view:

```swift
// Without trailing closure:
UIView.animate(withDuration: 0.3, animations: {
  self.view.alpha = 0
})
// With trailing closure:
UIView.animate(withDuration: 0.3) {
  self.view.alpha = 0
}
```

Trailing closure syntax has proven to be very popular, and it's not hard to guess why. Especially when an API is crafted with trailing closure syntax in mind, the call site is *easier to read*: it is more concise and less nested, without loss of clarity.

However, the restriction of trailing closure syntax to *only the final closure* has limited its applicability. This limitation was noticed [very early on in Swift's lifetime](https://www.natashatherobot.com/swift-trailing-closure-syntax/) as "the" problem with trailing closure syntax.

Consider using [`UIView.animate(withDuration:animations:completion:)`](https://developer.apple.com/documentation/uikit/uiview/1622515-animate) to remove the view once it has finished fading out:

```swift
// Without trailing closure:
UIView.animate(withDuration: 0.3, animations: {
  self.view.alpha = 0
}, completion: { _ in
  self.view.removeFromSuperview()
})
// With trailing closure
UIView.animate(withDuration: 0.3, animations: {
  self.view.alpha = 0
}) { _ in
  self.view.removeFromSuperview()
}
```

In this case, the trailing closure syntax is *harder to read*: the role of the trailing closure is unclear, the first closure remains nested, and something about the asymmetry is unsettling.

Concerns about call-site confusion have led Swift style guides to include rules that prohibit the use of trailing closure syntax when a function call has multiple closure arguments ([1](https://google.github.io/swift/#trailing-closures), [2](https://rules.sonarsource.com/swift/RSPEC-2958)).

As a result, if we ever need to append an additional closure argument to a function, many of us find ourselves having to rejigger our code more than may seem necessary:

```swift
// Single closure argument -> trailing closure
UIView.animate(withDuration: 0.3) {
  self.view.alpha = 0
}
// Multiple closure arguments -> no trailing closure
UIView.animate(withDuration: 0.3, animations: {
  self.view.alpha = 0
}, completion: { _ in
  self.view.removeFromSuperview()
})
```

## Proposed Solution

This proposal extends trailing closure syntax to allow additional *labeled* closures to follow the initial unlabeled closure:

```swift
// Single trailing closure argument
UIView.animate(withDuration: 0.3) {
  self.view.alpha = 0
}
// Multiple trailing closure arguments
UIView.animate(withDuration: 0.3) {
  self.view.alpha = 0
} completion: { _ in
  self.view.removeFromSuperview()
}
```

This extends the concision and denesting of trailing closure syntax to function calls with multiple closures arguments. And there's no rejiggering required to append an additional trailing closure argument!
Informally, the new syntax rules are:

* The first trailing closure drops its argument label (like today).
* Subsequent trailing closures require argument labels.

These rules seem to work well in practice, because functions with multiple closure arguments tend to have one that is more primary. Often any additional closure arguments are optional (via default parameter values or overloading), in order to provide progressive disclosure:


1. We've already seen UIKit's [`UIView.animate(withDuration:animations:)`](https://developer.apple.com/documentation/uikit/uiview/1622418-animate) and [`UIView.animate(withDuration:animations:completion:)`](https://developer.apple.com/documentation/uikit/uiview/1622515-animate) functions.

2. Consider Combine's [sink](https://developer.apple.com/documentation/combine/publisher/3343978-sink) operator, which today contorts itself to the existing trailing closure rules:

  ```swift
  ipAddressPublisher
    .sink { identity in
      self.hostnames.insert(identity.hostname!)
    }

  ipAddressPublisher
    .sink(receiveCompletion: { completion in
      // handle error
    }) { identity in
      self.hostnames.insert(identity.hostname!)
    }
  ```

  ... but could be re-worked in light of multiple trailing closure syntax:

  ```swift
  ipAddressPublisher
    .sink { identity in
      self.hostnames.insert(identity.hostname!)
    }

  ipAddressPublisher
    .sink { identity in
      self.hostnames.insert(identity.hostname!)
    } receiveCompletion: { completion in
      // handle error
    }
  ```

3. Consider SwiftUI's [Section](https://developer.apple.com/documentation/swiftui/section) view, which today avoids using `@ViewBuilder` closures for its optional header and footer:

  ```swift
  Section {
    // content
  }
  Section(header: ...) {
    // content
  }
  Section(footer: ...) {
    // content
  }
  Section(
    header: ...,
    footer: ...
  ) {
    // content
  }
  ```

  ... but could be re-worked in light of multiple trailing closure syntax:

  ```swift
  Section {
    // content
  }
  Section {
    // content
  } header: {
    ...
  }
  Section {
    // content
  } footer: {
    ...
  }
  Section {
    // content
  } header: {
    ...
  } footer: {
    ...
  }
  ```

When using multiple trailing closure syntax, these APIs are all [clear at the point of use](https://swift.org/documentation/api-design-guidelines/#fundamentals), without the need to label the first trailing closure.

If labelling the first trailing closure were allowed, users would have to evaluate whether to include the label on a case by case basis, which would inevitably lead to linter and style guide rules to prohibit it. So, in conjunction with the new syntax rules, we propose an amendment to the [API Design Guidelines](https://swift.org/documentation/api-design-guidelines):

> Name functions assuming that the argument label of the first trailing closure will be dropped. Include meaningful argument labels for all subsequent trailing closures.

## Detailed Design

The grammar is modified as follows to accommodate labeled trailing closures following an unlabeled trailing closure:

```
expr-trailing-closure:
  expr-postfix(trailing-closure) trailing-closures

trailing-closures:
  expr-closure
  trailing-closures (identifier|keyword|'_') ':' expr-closure
```

This introduces a zero-lookahead ambiguity between the start of a labeled trailing closure and the start of either a new expression, a labeled statement, or a `default:` label followed by `'{'`.  The first two ambiguities can be resolved by looking forward at most two tokens, because `(identifier|keyword|'_') ':'` can never start an expression, only a labeled statement, and the statement in a labeled statement can never start with `'{'` while `expr-closure` must start with it.  The ambiguity with `default:` can be resolved by not allowing the unescaped `default` keyword as a label in this syntax; it can still be used if necessary by escaping it (i.e. `` `default`: ``).  Source compatibility requires the existing use of `default:` to be preferred, and it's better to do this uniformly (even if the syntax does not appear in a `switch`) in order to discourage the use of `default` as a trailing-closure label in APIs, rather than leaving a trap in the language if an API like that is used within a `switch`.

The labeled trailing closures are associated with the base expression in the same way as the unlabeled trailing closure is today:

* if the base expression is a call, they are added as extra arguments to that call;
* if the base expression is a subscript, they are added as extra index arguments to that subscript,
* otherwise, an implicit call of the base expression is created using only the trailing closures as arguments.

The existing trailing-closure feature requires special treatment of the trailing-closure argument by the type checker because of the special power of label omission: the trailing closure can be passed to a parameter that would ordinarily require an argument label.  Currently, the special treatment is specific to the final argument.  Because this proposal still has an unlabeled trailing-closure argument, we have to generalize that treatment to allow label omission at an intermediate argument.

Note that labeled trailing closures are required to match labels with a parameter.  A labeled trailing closure can use the special label `_` to indicate that it matches an unlabeled parameter, but this *only* matches an unlabeled parameter; it does not have the special label-omission powers of the initial unlabeled trailing closure.  For example:

```swift
func pointFromClosures(
  x: () -> Int,
  _ y: () -> Int
) -> (Int, Int) {
  (x(), y())
}
pointFromClosures { 10 } _: { 20 }  // Ok

func performAsync(
  action: @escaping () -> Void,
  completionOnMainThread: @escaping () -> Void
) {
  ...
}
performAsync {
  // some action
} _: {               // Not okay: must use completionOnMainThread:
  window.exit()
}
```

In the current representation, the AST maintains a flag indicating whether the last argument was a trailing closure.  This is no longer sufficient, and instead the AST must maintain the exact position of the first trailing closure in the argument list.

The current type-checking rule for trailing closures performs a limited backwards scan through the parameters looking for a parameter that is compatible with a trailing closure.  The proposed type-checking rule builds on this while seeking to degenerate to the old behavior when there are no labeled trailing closures.  This is done by performing a backwards scan through the parameters to bind all the labeled trailing closures to parameters using label-matching, then doing the current limited scan for the unlabeled trailing closure starting from the last labeled parameter that was matched.

For example, given this function:

```swift
func when<T>(
  _ condition: @autoclosure () -> Bool,
  then: () -> T,
  `else`: () -> T
) -> T {
  condition() ? then() : `else`()
}
```

The following call using the new syntax:

```swift
when(2 < 3) {
  print("then")
} else: {
  print("else")
}
```

is equivalent to:

```swift
when(2 < 3, then: { print("then") }, else: { print("else") })
```

It's important to note that the handling of default arguments in relation to trailing closures is maintained as-is.  For example:

```swift
func foo(a: () -> Int = { 42 }, b: Any? = nil) {}

foo {
  42
}
```

Although trailing closure matches parameter for `a:` by type, existing trailing closure behavior would match trailing closure argument to parameter labeled as `b:`, which means that previous call to foo is equivalent to:

```swift
foo(b: { 42 })
```

Now let's add one more parameter to foo to see how this applies to the new multiple trailing closures syntax:

```swift
func foo(a: () -> Int = { 42 }, b: Any? = nil, c: () -> Void) {}

foo {
  42
} c: {
  ...
}
```

Since the new type-checking rule dictates a backwards scan starting for the last (labeled) trailing closure before attempting to match an unlabeled argument, this call is equivalent to a following "old" syntax:

```swift
foo(b: { 42 }, c: { ... })
```

This shows that unlabeled trailing closure matching behaves exactly the same way in both scenarios.

There are reasonable arguments against the backwards-scan design for type-checking trailing closures.  Perhaps the strongest argument is that this interaction with default arguments is unintuitive and limiting.  For example, it is natural to want to take a primary, required closure, followed by some optional closures:

```swift
func resolve(
  id: UUID,
  action: (Object) -> Void,
  completion: (() -> Void)? = nil,
  onError: ((Error) -> Void)? = nil
) {
  ...
}
```

Under the proposed type-checking rule, code like the following will not type-check as expected:

```swift
resolve(id: paulID) { paul in
  // do something with object
} onError: { error in
  // handle error
}
```

It is tempting to try to take advantage of the introduction of this new syntax to use a better type-checking rule that would handle this correctly.  This would help in some cases.  However, unfortunately, when the programmer omits both of the optional closures, they’re no longer using this new syntax, because they’ve dropped back to the existing trailing-closure case:

```swift
resolve(id: paulID) { paul in
  // do something with object
}
```

The behavior of this call can’t be changed without potentially breaking source compatibility.  That might be worthwhile to do in order to enable these sorts of APIs and get more consistent type-checking behavior for trailing closures; however, it will need its own proposal, and it will only be feasible under a new source-compatibility mode.  We recommend considering this for Swift 6.  In the meantime, library designers will have to use overloading to get this effect instead of default arguments.

## Alternatives Considered

### Trailing Block of Closures

Multiple trailing closures could alternatively be specified within a trailing block, with each trailing closure indicated by an argument label:

```swift
UIView.animate(withDuration: 0.3) {
  animations: {
    self.view.alpha = 0
  }
  completion: { _ in
    self.view.removeFromSuperview()
  }
}
```

While this syntax is clear at the point of use, pleasant to read, and provides contextual cues by separating the trailing closures from the rest of the arguments, it risks evolving into an alternative calling syntax. The proposed syntax is more concise and less nested, without loss of clarity:

```swift
UIView.animate(withDuration: 0.3) {
  self.view.alpha = 0
} completion: { _ in
  self.view.removeFromSuperview()
}
```

### Optionally Labeled First Trailing Closure

The proposed syntax could be extended to allow users to optionally label the first trailing closure:

```swift
ipAddressPublisher
  .sink receiveCompletion: { completion in
    // handle error
  }
```

This would allow the user to disambiguate when the backwards-scan would have otherwise resolved differently, in this case for the declaration:

```swift
public func sink(
    receiveCompletion: ((Subscribers.Completion<Failure>) -> Void)? = nil,
    receiveValue: ((Output) -> Void)? = nil
) -> Subscribers.Sink<Self>
```

However, it shouldn’t be used to disambiguate for fellow humans. Recall: API authors should be naming functions assuming that the argument label of the first trailing closure will be dropped. Swift users aren’t used to seeing function names and argument labels juxtaposed without parenthesis. Many find this spelling unsettling.

Improving the type-checking rule, as described in Detailed Design, is a more promising avenue for addressing this use case. In the meantime, users that find themselves in this situation can use the existing syntax:

```swift
ipAddressPublisher
  .sink(receiveCompletion: { completion in
    // handle error
  })
```
