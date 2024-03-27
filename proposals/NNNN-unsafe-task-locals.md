# Unsafe Task Local Binding

* Proposal: SE-NNNN
* Author: [Konrad 'ktoso' Malawski](https://github.com/ktoso)
* Review Manager: TBD
* Status: Implemented [PR #70062](https://github.com/apple/swift/pull/70062)
* Review: TBD

## Introduction

With the introduction of Swift Concurrency and the concept of tasks to the language, Swift also gained the ability to
associate values with the structured task hierarchy thanks to [Task Local Values (SE-0311)](https://github.com/apple/swift-evolution/blob/main/proposals/0311-task-locals.md).

Task local values are a **safe** and powerful alternative to what in traditional thread-based concurrency systems
would be handled by thread-local values, however they are also automatically inherited by child tasks created
by the task which bound such value, and are automatically cleaned up when the task or scope completes.

This proposal aims to introduce "unsafe" versions of the task local APIs in Swift.

## Motivation

Task local values achieve, in large part, their safety and _structured_ properties that work so well with
Structured Concurrency by forcing the APIs to form scopes during which the value binding applies.

In practice this means that there is no API to "set" a task-local value, but instead a key is bound to a specific value
in the task's storage by calling the `withValue` method, like this:

```swift
@TaskLocal
var currentCustomer: CustomerID?

let customerID = makeCustomerID()

$currentCustomer.withValue(customerID) {
    let sushi = makeSushi()
    // ... 
}

func makeSushi() -> Sushi {
    print("Preparing sushi for customer \(currentCustomer)")
    // ... 
}
```

This capability is used primarily to propagate additional metadata throughout a chain of calls, 
without which the system would still work fine, but thanks to this metadata we can provide
enriched logging, security, or other semantics that improve the usability and understandability of the system.

Notably, the `withValue` closure makes it possible for the runtime to reliably "push" and "pop" values onto the 
task-local bindings stack. Developers cannot "forget" to remove a value when it should be cleaned up, and they cannot
remove more values than they should. This API is inherently safe and does not need much explanation to use it correctly.

This API however does not compose well when code being called cannot be wrapped in a closure like this.
And more specifically, it makes writing method attached macros problematic because they would have to "replace"
the entire method body, sacrificing performance, safety, and auto-completion in the process.

This proposal offers an unsafe solution tailored to advanced users to remedy this problem.

## Proposed solution

We propose to introduce a pair of "unsafe" methods used to push and pop value bindings onto the task-local bindings stack:

- `unsafePushValue(_:)`
- `unsafePopValue()`

They are equivalent to what the `withValue` method is currently doing, however they can be used without forcing the 
nesting that the with closure accepting API forces (`$key.withValue(...) { ... }`).

These APIs are unsafe because an un-balanced pop could either attempt to pop more values than the stack contains 
leading to incorrect value bindings. Or even worse attempt to pop a value stored in a parent task (which can happen
by popping more values than a task has pushed). 

> **WARNING:** Mis-use of these APIs will lead to incorrect behavior, crashes, or even undefined behavior, so they must be handled very carefully. Preferably, only in macro generated code which can _guarantee_ the correct balance of push/pop calls.

The following code using `withValue`:

```swift
func order() async {
  $wasabiPreference.withValue(.with) {
    let sushi = await orderSushi()
  }
}
```

can be expressed using the new unsafe APIs as follows:

```swift
func order() async {
  $wasabiPreference.unsafePushValue(.with)
  defer { TL.$number.unsafePopValue() } 
  let sushi = await orderSushi() 
}
```

More interestingly, if [function body macros (PITCH)](https://forums.swift.org/t/function-body-macros/66471/72) would be accepted,
it would become possible to express the wasabi preference as follows:

```swift
// function body macros are subject to a separate evolution proposal,
// see here: https://github.com/apple/swift-evolution/pull/2219/
// 
// @attached(preamble)
// macro WasabiPreference(_ preference: WasabiPreference) = #externalMacro(...)

@WasabiPreference(.with)
func order() async {
  let sushi = await orderSushi() 
}
```

which expands to the same piece of code we've seen before, using the unsafe push and pop method calls:

```swift
@WasabiPreference(.with)
func order() async {
  //   $wasabiPreference.unsafePushValue(.with)
  //   defer { TL.$number.unsafePopValue() }
  let sushi = await orderSushi() 
}
```

One of the expected adopters of this pattern is [swift-distributed-tracing](https://github.com/apple/swift-distributed-tracing),
or libraries similar to it, which need to setup a task-local "context" for the duration of a method. 

### Detailed design

The exact method signatures that are being proposed here are as follows:

```swift
extension TaskLocal {

  /// Unsafe API for binding a task-local value that must be paired with an
  /// ``unsafePopValue(file:line:)`` call to remove the task local when exiting
  /// the scope in which it was set.
  ///
  /// Generally, the safe ``withValue(_:operation:file:line:)`` method should be
  /// used instead of this pair of unsafe calls.
  ///
  /// Using this unsafe pair of methods can lead to runtime crashes, or
  /// unexpected behavior (including memory leaks), when not used correctly.
  ///
  /// Every value *push* **must** be paired with exactly one *pop* of a value.
  /// The safe `withValue` version of this API handles this automatically by
  /// pushing the value before the operation closure executes, and popping it
  /// after the closure completes. Using these the
  /// ``unsafePushValue(_:file:line:)`` and ``unsafePopValue(file:line:)`` pair
  /// of methods **must** uphold the same semantic - when exiting a scope (e.g.
  /// method, or code block), the value must be popped properly.
  ///
  /// Omitting a pop operation may lead to leaking and over-hanging a task-local
  /// value binding beyond it's expected scope. Issuing an additional pop
  /// operation will pop a different value binding and risk incorrect behavior.
  /// Attempting to pop a value when the task-local value bindings stack is
  /// empty will result in a runtime crash.
  ///
  /// - Parameters:
  ///   - valueDuringOperation: the value to be pushed onto the task-local
  ///                           binding stack
  ///
  /// - SeeAlso: ``withValue(_:file:line:)``
  /// - SeeAlso: ``unsafePopValue(_:file:line:)``
  @inlinable
  public func unsafePushValue(_ valueDuringOperation: __owned Value,
                              file: String = #fileID, line: UInt = #line) {
    // check if we're not trying to bind a value from an illegal context; this may crash
    _checkIllegalTaskLocalBindingWithinWithTaskGroup(file: file, line: line)

    _taskLocalValuePush(key: key, value: consume valueDuringOperation)
  }

  /// Unsafe API for removing a task-local binding from the bindings stack.
  ///
  /// Generally, the safe ``withValue(_:operation:file:line:)`` method should be
  /// used instead of this pair of unsafe calls.
  ///
  /// Must only be used in conjunction with the
  /// ``unsafePushValue(_:file:line:)`` method, to remove any value binding the
  /// prior has established.
  ///
  /// - SeeAlso: ``withValue(_:file:line:)``
  /// - SeeAlso: ``unsafePushValue(_:file:line:)``
  @inlinable
  public func unsafePopValue(file: String = #fileID, line: UInt = #line)
}
```

## Alternatives considered

### Do not provide unsafe APIs for task-locals

This proposal is primarily driven by the need to offer a great user experience when using function body macros.
We had considered making it possible for those to "wrap the body" of a function which would mean no need to
introduce these unsafe APIs. However, this would come at a large cost other pieces of the language, 
and provide a not-great user experience (e.g. not working auto completion) inside methods using "body replacement" macros.

Thus we decided it is safer and more composable to offer an unsafe push/pop API that macros can use to build safe
macros with, even if the push/pop building block by itself is not safe in general.

## Revisions

- 1.0 
  - initial revision 