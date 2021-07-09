# Super call enforcement

* Proposal: [SE-XXXX](xxxx-super-call-enforcement.md)
* Author: [Suyash Srijan](https://github.com/theblixguy)
* Review Manager: **TBD**
* Status: **Pending Review**
* Bug: [SR-6417](https://bugs.swift.org/browse/SR-6417)
* Implementation: [apple/swift#32712](https://github.com/apple/swift/pull/32712)

## Introduction

Introduce a new attribute to enforce that an overridden function should call the `super` method in its body.

Swift-evolution thread: [Super call enforcement](https://forums.swift.org/t/super-call-enforcement/38177)

## Motivation

It is quite common to override a method in order to add additional functionality to it, rather than to completely replace it. For example - one might override `UIViewController.awakeFromNib()` to add additional setup code to it:

```swift
class MyViewController: UIViewController {
  override func awakeFromNib() {
    super.awakeFromNib()
    // Configure some views
  }
}
```

On top of that, the superclass method may also sometimes require a call to itself in an override for the overall functionality to work correctly. Here's an example (provided by [Rob Mayoff](https://forums.swift.org/t/super-call-enforcement/38177/37)) that illustrates the problem:

```swift
open class StyledText {
    // Make `boldness(at:)` return true for every point in `range`.
    public func embolden(in range: Range<String.Iterator>) {
        ... implementation details ...
        self.styleDidChange(in: range)
    }

    // When NOT inside a call to `embolden(in:)`, the returned
    // `extent` is the largest range that contains point and in which
    // every character's boldness is `isBold`
    public func boldness(at point: String.Iterator) -> (isBold: Bool, extent: Range<String.Iterator>) {
        ... implementation details ...
    }

    public func styleDidChange(in range: Range<String.Iterator>) {
        // Coalesce adjacent bold ranges to simplify the implementation
        // of `boldness(at:)`.
        ... implementation details ...
    }
}
```

The implementation of `embolden(in)` relies on the behavior of `styleDidChange(in:)` to ensure that `boldness(at:)` returns the correct range of boldness.

Suppose one adds a subclass of `StyledText`:

```swift
open class VeryStyledText: StyledText {
    public override func styleDidChange(in range: Range<String.Iterator>) {
        ... implementation details ...
    }
}
```

If `VeryStyledText` forgets to call `super.styleDidChange(in:)` in the implementation, then `StyledText` cannot maintain its own correct behavior. Similarly, if one further subclasses `VeryStyledText` (and so on) and forgets to call `super` in the chain, then that particular subclass will end up breaking the overall functionality.

At present, there is no way to communicate this need, other than adding it to the documentation. Even experienced developers sometimes overlook this small detail, and later run into various issues at runtime. In Objective-C, one can annotate a superclass method with the [`objc_requires_super`](https://clang.llvm.org/docs/AttributeReference.html#objc-requires-super) attribute, and the compiler emits a warning when an overridden method is missing a call to the `super` method in its body.

However, there is no such attribute in Swift. There are some sub-optimal solutions to this problem, for example:

```swift
class C1 {
  final func foo() {
    bar()
  }
  
  func bar() {}
}

class C2: C1 {
  override func bar() {
    // It's okay to elide the super.bar() because 'foo()' calls 'bar()'
  }
}
```

However, this has a couple of problems:

1. It doesn't work when you have a subclass of `C2`.
2. The users of `bar()` lose control over when `super` is called.
3. If your class is `public` and you cannot give `foo()` an access level of `private` or `internal` (maybe because it's called from outside its module), then you now have an additional API method in your interface that you have to document as something that no one should ever call directly.

## Proposed solution

Introduce a new `@requiresSuper` attribute to control `super` calls in overridden methods.

When a `@requiresSuper` attribute is present on a method, any overrides of that method should call the `super` method in their body. If the `super` call is missing, the compiler will emit an error. The error can be suppressed by inserting the `super` call and the compiler places no restrictions on where or how many times the call appears.

```swift
class C1 {
  @requiresSuper 
  func foo() {}
}

class C2: C1 {
  override func foo() {} // error: method override is missing 'super.foo()' call
}

class C3: C1 {
  override func foo() { // Okay
    super.foo()
  }
}
```

## Detailed Design

The `super` call must be made to the base method that is annotated with `@requiresSuper` and not to some other base method. For example:

```swift
class C1 {
  @requiresSuper
  func foo() {}
  func foo(arg: Int) {}
  func foo(_ arg: Bool) {}
  func bar()
}

class C2: C1 {
  override func foo() { super.foo(arg: 0) } // error: method override is missing 'super.foo()' call
}

class C3: C1 {
  override func foo() { super.foo(false) } // error: method override is missing 'super.foo()' call
}

class C4: C1 {
  override func foo() { super.bar() } // error: method override is missing 'super.foo()' call
}

class C5: C1 {
  override func foo() { super.foo() } // Okay
}
```

A message can be (optionally) specified on the `@requiresSuper` attribute to provide any additional information, which will be shown with the error message:

```swift
class C1 {
  @requiresSuper("Call super as the final step in your implementation")
  func foo() {}
}

class C2: C1 {
  override func foo() {} // error: method override is missing 'super.foo()' call: Call super as the final step in your implementation
}
```

Any overrides of a method annotated with `@requiresSuper` will also implicitly inherit the attribute, to make sure that all the overrides in a subclass chain call back to the `super` method. This will be suppressed if the override is in a `final` class:

```swift
class C1 {
  @requiresSuper
  func foo() {}
}

class C2: C1 {
  // Implicitly inherits '@requiresSuper' from 'C1.foo'
  override func foo() {
    super.foo()
    // Do some other stuff
  }
}

final class C3: C1 {
  // Does not implicitly inherit '@requiresSuper' from 'C1.foo', since 'C3.foo' cannot be overridden
  override func foo {
    super.foo()
    // Do some other stuff
  }
}

class C4: C2 {
  override func foo() {} // error: method override is missing 'super.foo()' call
}
```

Finally, a `@requiresSuper` attribute will also be implicitly added for imported ObjC methods which have been annotated with `objc_requires_super`. This enables warnings (instead of errors) for any method with such an attribute, such as `UIViewController.awakeFromNib()`:

```swift
import UIKit

class MyViewController1: UIViewController {
  override func awakeFromNib() {} // warning: method override is missing 'super.awakeFromNib()' call
}

class MyViewController2: UIViewController {
  override func awakeFromNib() { super.awakeFromNib() } // Okay
}
```

## Source compatibility

This is an additive change, so it does not break any existing Swift code. An imported ObjC declaration annotated with `objc_requires_super` will trigger a warning (instead of an error) if the overridden method skips the `super` call, which helps preserve source compatibility while giving users time to update their code.

## Effect on ABI stability

This change has no effect on the ABI.

## Effect on API resilience

Adding `@requiresSuper` to any existing API method can break code if an override of that method does not already call `super` in its body. Removing the attribute does not have any effect.

## Alternatives considered

- Do not diagnose missing `super` calls.
- Diagnose missing `super` calls but also enforce their placement: The ordering of a `super` call (first, middle, last, etc) is not a syntactic requirement. Adding such a requirement will complicate API design and usage and will also not be possible to prove statically in some scenarios. For simplicity, it would be better to only dictate that a superclass method is called at some point, rather than also dictating its order and how many times it's called.
- Diagnose missing `super` calls but also allow a way to opt-out using an `@ignoresSuper` attribute. It would be strange to allow users to bypass something that was deemed as a "requirement" by an API author.

## Future Directions

We could make the `super` call check the default and get rid of the `@requiresSuper` attribute. This would be source-breaking in practice, but can be mitigated by downgrading the error to a warning in a compatibility mode. We could offer an `@ignoresSuper` attribute instead to silence the diagnostic.