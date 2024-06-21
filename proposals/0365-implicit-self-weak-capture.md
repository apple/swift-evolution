# Allow implicit `self` for `weak self` captures, after `self` is unwrapped

* Proposal: [SE-0365](0365-implicit-self-weak-capture.md)
* Author: [Cal Stephens](https://github.com/calda)
* Review Manager: [Saleem Abdulrasool](https://github.com/compnerd)
* Status: **Implemented (Swift 5.8)**
* Implementation: [apple/swift#40702](https://github.com/apple/swift/pull/40702), [apple/swift#61520](https://github.com/apple/swift/pull/61520)

## Introduction

As of [SE-0269](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0269-implicit-self-explicit-capture.md), implicit `self` is permitted in closures when `self` is written explicitly in the capture list. We should extend this support to `weak self` captures, and permit implicit `self` as long as `self` has been unwrapped.

```swift
class ViewController {
    let button: Button

    func setup() {
        button.tapHandler = { [weak self] in
            guard let self else { return }
            dismiss()
        }
    }

    func dismiss() { ... }
}
```

Swift-evolution thread: [Allow implicit `self` for `weak self` captures, after `self` is unwrapped](https://forums.swift.org/t/allow-implicit-self-for-weak-self-captures-after-self-is-unwrapped/54262)

## Motivation

Explicit `self` has historically been required in closures, in order to help prevent users from inadvertently creating retain cycles. [SE-0269](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0269-implicit-self-explicit-capture.md) relaxed these rules in cases where implicit `self` is unlikely to introduce a hidden retain cycle, such as when `self` is explicitly captured in the closure's capture list:

```swift
button.tapHandler = { [self] in
    dismiss()
}
```

SE-0269 left the handling of `weak self` captures as a future direction, so explicit `self` is currently required in this case:

```swift
button.tapHandler = { [weak self] in
    guard let self else { return }
    self.dismiss()
}
```

Since `self` has already been captured explicitly, there is limited value in requiring authors to use explicit `self`. This is inconsistent, and adds unnecessary visual noise to the body of closures using `weak self` captures.

## Proposed solution

We should permit implicit `self` for `weak self` captures, once `self` has been unwrapped.

This code would now be allowed to compile:

```swift
class ViewController {
    let button: Button

    func setup() {
        button.tapHandler = { [weak self] in
            guard let self else { return }
            dismiss()
        }
    }

    func dismiss() { ... }
}
```

## Detailed design

### Enabling implicit `self`

All of the following forms of optional unwrapping are supported, and enable implicit self for the following scope where `self` is non-optional:

```swift
button.tapHandler = { [weak self] in
  guard let self else { return }
  dismiss()
}

button.tapHandler = { [weak self] in
  guard let self = self else { return }
  dismiss()
}

button.tapHandler = { [weak self] in
  if let self {
    dismiss()
  }
}

button.tapHandler = { [weak self] in
  if let self = self {
    dismiss()
  }
}

button.tapHandler = { [weak self] in
  while let self {
    dismiss()
  }
}

button.tapHandler = { [weak self] in
  while let self = self {
    dismiss()
  }
}
```

Like with implicit `self` for `strong` and `unowned` captures, the compiler will synthesize an implicit `self.` for calls to properties / methods on `self` inside a closure that uses `weak self`.

If `self` has not been unwrapped yet, the following error will be emitted:

```swift
button.tapHandler = { [weak self] in
    // error: explicit use of 'self' is required when 'self' is optional,
    // to make control flow explicit
    // fix-it: reference 'self?.' explicitly
    dismiss()
}
```

### Nested closures

Nested closures can be a source of subtle retain cycles, so have to be handled more carefully. For example, if the following code was allowed to compile, then the implicit `self.bar()` call could introduce a hidden retain cycle:

```swift
couldCauseRetainCycle { [weak self] in
  guard let self else { return }
  foo()

  couldCauseRetainCycle {
    bar()
  }
}
```

Following the precedent of [SE-0269](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0269-implicit-self-explicit-capture.md), additional closures nested inside the `[weak self]` closure must capture `self` explicitly in order to use implicit `self`.

```swift
// Not allowed:
couldCauseRetainCycle { [weak self] in
  guard let self else { return }
  foo()

  couldCauseRetainCycle {
    // error: call to method 'method' in closure requires 
    // explicit use of 'self' to make capture semantics explicit
    bar()
  }
}

// Allowed:
couldCauseRetainCycle { [weak self] in
  guard let self else { return }
  foo()

  couldCauseRetainCycle { [weak self] in
    guard let self else { return }
    bar()
  }
}

// Also allowed:
couldCauseRetainCycle { [weak self] in
  guard let self else { return }
  foo()

  couldCauseRetainCycle {
    self.bar()
  }
}

// Also allowed:
couldCauseRetainCycle { [weak self] in
  guard let self else { return }
  foo()

  couldCauseRetainCycle { [self] in
    bar()
  }
}
```

Also following [SE-0269](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0269-implicit-self-explicit-capture.md), implicit `self` will only be permitted if the `self` optional binding specifically, and exclusively, refers the closure's `self` capture:

```swift
button.tapHandler = { [weak self] in
    guard let self = self ?? someOptionalWithSameTypeOfSelf else { return }

    // error: call to method 'method' in closure requires explicit use of 'self' 
    // to make capture semantics explicit
    method()
}
```

## Source compatibility

This change is purely additive and does not break source compatibility of any valid existing Swift code.

## Effect on ABI stability

This change is purely additive, and is a syntactic transformation to existing valid code, so has no effect on ABI stability.

## Effect on API resilience

This change is purely additive, and is a syntactic transformation to existing valid code, so has no effect on API resilience.

## Alternatives considered

It is technically possible to also support implicit `self` _before_ `self` has been unwrapped, like:

```swift
button.tapHandler = { [weak self] in
    dismiss() // as in `self?.dismiss()`
}
```

That would effectively add implicit control flow, however. `dismiss()` would only be executed when `self` is not nil, without any indication that it may not run. We could create a new way to spell this that still implies optional chaining, like `?.dismiss()`, but that is not meaningfully better than the existing `self?.dismiss()` spelling.

## Acknowledgments

Thanks to the authors of [SE-0269](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0269-implicit-self-explicit-capture.md) for laying the foundation for this proposal.

Thanks to Kyle Sluder for [the suggestion](https://forums.swift.org/t/allow-implicit-self-for-weak-self-captures-after-self-is-unwrapped/54262/2) to not permit implicit `self` in cases where the unwrapped `self` value doesn't necessarily refer to the closure's `self` capture, like in `let self = self ?? C.global`.

Many thanks to Pavel Yaskevich, John McCall, and Xiaodi Wu for providing significant feedback and advice regarding the implementation of this proposal.