# Introducing Namespacing for Common Swift Error Scenarios

* Proposal: [SE-TBD](0tbd-fatal.md)
* Authors: [Dave DeLong](http://github.com/davedelong), [Tim Vermeulen](https://github.com/timvermeulen), [Erica Sadun](https://github.com/erica)
* Review Manager: TBD
<!---
* Implementation: [apple/swift#NNNNN](https://github.com/apple/swift/pull/NNNNN)
* Decision Notes: [Rationale](https://lists.swift.org/pipermail/swift-evolution/), [Additional Commentary](https://lists.swift.org/pipermail/swift-evolution/)
* Bugs: [SR-NNNN](https://bugs.swift.org/browse/SR-NNNN), [SR-MMMM](https://bugs.swift.org/browse/SR-MMMM)
* Previous Revision: [1](https://github.com/apple/swift-evolution/blob/...commit-ID.../proposals/NNNN-filename.md)
* Previous Proposal: [SE-XXXX](XXXX-filename.md)
* -->

## Introduction

This proposal provides a namespaced umbrella for common exit scenarios, modernizing a holdover from C-like languages.

Discussion: [Introducing Namespacing for Common Swift Error Scenarios](https://forums.swift.org/t/introducing-namespacing-for-common-swift-error-scenarios/10773)

## Motivation

Swift's `fatalError` is a one-size-fits-all hammer. Swift lacks a modern extensible solution that differentiates fatal error scenarios for specific contexts. Because of this, many developer libraries now implement custom exit points like the following example to provide that functionality:

```swift
/// Handles unimplemented functionality with site-specific information
/// courtesy of Soroush Khanlou, who discovered it from Caleb Davenport, 
/// who discovered it from Brian King
func unimplemented(_ function: String = #function, _ file: String = #file) -> Never {
    fatalError("\(function) in \(file) has not been implemented")
}
```

Namespacing these marker functions to a common type results in groupable, extensible, and discoverable IDE elements.

There are four universal core language concepts that can be better represented in Swift. They are:

* This code is unreachable.
* This code is not yet implemented.
* This method must be overriden in a subtype.
* This code should never be called.

These items represent common scenarios that are fundamental to language design.

### Unreachable

The Swift compiler cannot detect that the following code is exhaustive. It emits an error and recommends a default clause.

```swift
switch Int.random(in: 0...2) {
case 0: print("got 0")
case 1: print("got 1")
case 2: print("got 2")
}

// error: switch must be exhaustive
// note: do you want to add a default clause?
```

That default clause should annotate the code so it's clear that the clause is unreachable and will never run. If it does run, there is a serious breach in logic that should result in a fatal outcome.

### Not yet implemented

In the `TODO` scenario, a type member is reachable but calling it should trap at runtime because the logic has not yet been implemented. 

```swift
public func semanticallyResonantMethodName() {
    // TODO: implement this
}
```

This call should establish why the code cannot yet be run. Swift's compile-time diagnostics are insufficient as they don't provide runtime suport:

* Using `#error` prevents compilation and debugging. 
* Using `#warning` is insufficient. It can be missed at compile time and error in "warn-as-errors" development houses.  

There should be a way to trap at run-time to indicate an currently-unsafe avenue of execution.

### Must override

It's not uncommon to build an abstract supertype that requires methods and properties to be implemented in concrete subtypes:

```swift
class AbstractSupertype {
    func someKeyBehavior() {
        // Must override in subclass
    }
}
```

This method should never be called, as the type is abstract.

### Must not be called

Silencing warnings should not contradict Swift safety best practices. Some required members may add functionality that should never be called:

```swift
required init?(coder: NSCoder) {
  super.init(coder: coder)
  // Xcode requires unimplemented initializer
}
```

A method that is left _deliberately_ unimplemented rather than as a `TODO` item should be annotated as such.

## Proposed solution

Overload the global `fatalError` function as `fatalError(because:)` and introduce `FatalReason`, a new type that vends static members to error under these universal scenarios. The precise wording and choice of each member and error message can be bikeshedded after acceptance of this proposal's concept.

```swift
/// Reasons why code should die at runtime
public struct FatalReason: CustomStringConvertible {
  /// Die because this code branch should be unreachable.
  public static let unreachable = FatalReason("Should never be reached during execution.")
  
  /// Die because this method or function has not yet been implemented.
  public static let notYetImplemented = FatalReason("Not yet implemented.")
  
  /// Die because a default method must be overriden by a
  /// subtype or extension.
  public static let subtypeMustOverride = FatalReason("Must be overriden in subtype.")
  
  /// Die because this functionality should never be called,
  /// typically to silence requirements.
  public static let mustNotBeCalled = FatalReason("Should never be called.")
  
  /// An underlying string-based cause for a fatal error.
  public let reason: String
  
  /// Establishes a new instance of a `FatalReason` with a string-based explanation.
  public init(_ reason: String) {
    self.reason = reason
  }
  
  /// Conforms to CustomStringConvertible, allowing reason to 
  /// print directly to complaint.
  public var description: String {
    return reason
  }
}

/// Unconditionally prints a given message and stops execution.
///
/// - Parameters:
///   - reason: A predefined `FatalReason`.
///   - function: The name of the calling function to print with `message`. The 
///     default is the calling scope where `fatalError(because:, function:, file:, line:)`
///     is called.
///   - file: The file name to print with `message`. The default is the file
///     where `fatalError(because:, function:, file:, line:)` is called.
///   - line: The line number to print along with `message`. The default is the
///     line number where `fatalError(because:, function:, file:, line:)` is called.
@inlinable // FIXME(sil-serialize-all)
@_transparent
public func fatalError(
  because reason: FatalReason, 
  function: StaticString = #function,
  file: StaticString = #file, 
  line: UInt = #line
) -> Never {
  fatalError("\(function): \(reason)", file: file, line: line)
}
```

You call `fatalError(because:)` using the static entry points:

```swift
fatalError(because: .unreachable)
fatalError(because: .notYetImplemented)
fatalError(because: .subtypeMustOverride)
fatalError(because: .mustNotBeCalled)
```

A few points about this design:

* The contextual members (`function`, `file`, and `line`) are automatically picked up by the defaulted properties and passed to `fatalError`.
* The `FatalReason` namespacing allows each reason to be auto-completed by a hosting IDE.
* The text is intentionally developer-facing and unlocalized, as with other warning and error content.
* The `FatalReason` namespace encourages additions specific to fatal outcomes. As its name suggests, it is not a shared repository for developer-facing strings.

This design allows you to extend `FatalReason` to introduce custom causes of programmatic run-time failure common to your in-house work, such as:

```swift
// Custom outlet complaint
extension FatalReason {
  static func missingOutlet(_ name: String) -> FatalReason {
    return FatalReason("Outlet \"\(name)\" is not connected")
  }
}

// for example:
fatalError(because: .missingOutlet("myCheckbox"))
```
## Source compatibility

This proposal is strictly additive. There is no reason to deprecate or remove `fatalError`.

## Effect on ABI stability

This proposal does not affect ABI stability.

## Effect on API resilience

This proposal does not affect ABI resilience.

## Alternatives considered

* Introduce a standalone type (such as `Abort`, `Crash`, `Fatal` or `Trap`) and a static method to avoid using a global function. This approach is less intuitive and breaks the mold established by the current `fatalError` function.

* Reconfigure the `fatalError` callsite to drop an external label, allowing `fatalError("some string")` and `fatalError(.someReason)`.

* Extend `Never` instead of introducing a new type. `Never.because()` and `Never.reason()` are confusing to read. `Never.die()` is contradictory.

## Acknowlegements

Thanks [Ian Keen](https://github.com/IanKeen/), [Brent Royal-Gordon](https://github.com/brentdax), Lance Parker, Lily Vulcano.