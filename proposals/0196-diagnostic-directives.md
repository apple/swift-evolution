# Compiler Diagnostic Directives

* Proposal: [SE-0196](0196-diagnostic-directives.md)
* Author: [Harlan Haskins](https://github.com/harlanhaskins)
* Review Manager: [Ted Kremenek](https://github.com/tkremenek)
* Implementation: [apple/swift#14048](https://github.com/apple/swift/pull/14048)
* Previous Revision: [1](https://github.com/swiftlang/swift-evolution/blob/ab0c22a2340be9bfcb82e6f237752b4d959a93b7/proposals/0196-diagnostic-directives.md)
* Status: **Implemented (Swift 4.2)**

## Introduction

This proposal introduces `#warning` and `#error` directives that will cause
the Swift compiler to emit a custom warning or an error during compilation.

## Motivation

During the development lifecycle, it's common to leave bits of code unfinished
while focusing on other parts. Frequently, developers leave `TODO` or `FIXME`
comments around unfinished or suboptimal portions of their code to signal
to themselves and their teammates that the code needs work. Many editors pick
up on these comments and present them in code navigators, but these don't
materialize in command-line builds or continuous integration. Allowing #warning
gives a predictable, universal way to ensure a message will be displayed during
compilation.  

Additionally, it's possible for two build configurations to be mutually
exclusive, and there isn't a canonical way to ensure those configurations
don't happen. `#error` solves this problem:

```swift
#if MYLIB_VERSION < 3 && os(macOS)
#error("MyLib versions < 3 are not supported on macOS")
#endif
```

`#warning` can be used for code templates where the user is meant to fill in
the values of global constants or implement missing routines:

```swift
enum APICredentials {
  #warning("fill in your API key below")
  static let key = ""
  #warning("fill in your API secret below")
  static let secret = ""
}
```

## Proposed solution

Add `#warning` and `#error` as compiler directives that emit an appropriate
diagnostic with the contents, pointing to the start of the message.

```swift
func configPath() -> String {
  #warning("this should be made more safe")
  return Bundle.main().path(forResource: "Config", ofType: "plist")!
}
```

## Detailed design

This will add four new productions to the Swift grammar:

```
compiler-control-statement → warning-directive
compiler-control-statement → error-directive
warning-directive → #warning '(' static-string-literal ')'
error-directive → #error '(' static-string-literal ')'
```

Upon parsing a `#error` directive, the Swift compiler will emit the provided
string literal as an error, pointing to the beginning of the
string, and ignore the directive.

Upon parsing a `#warning` directive, the Swift compiler will emit the provided
string literal as a warning, pointing to the beginning of the
string, and ignore the directive.

If a `#warning` or `#error` exists inside a branch of a `#if` directive that is
not taken, then no diagnostic is emitted.

```swift
#if false
#warning("this will not trigger a warning")
#error("this will not trigger an error")
#endif

#if true
#warning("this will trigger a warning")
#error("this will trigger an error")
#endif
```

## Impact on existing code

This change is purely additive; no migration will be required.

## Alternatives considered

- We could do some kind of comment-parsing based approach to surface
  `TODO`s and `FIXME`s, but `#warning` serves as a general-purpose facility
  for reporting at compile time. It is also likely that there are `TODO` and
  `FIXME` comments that are not urgent, but that should be revisited. 

- [Alexander Momchilov](https://forums.swift.org/t/pitch-warning/2819/41) brought
  up the idea of using `TODO` and `warning` as functions in the standard
  library with special compiler magic that will warn on their uses.

  ```swift
  func TODO(_ message: StaticString? = nil) {
    if let s = message { print("TODO: \(s)") }
  }

  @discardableResult
  func TODO<T>(_ message: StaticString? = nil, _ temporaryValue: T) -> T {
    if let s = message { print("TODO: \(s)") }
    return temporaryValue
  }
  ```
  While these could be useful, I think `#warning` and `#error` have uses beyond
  just marking unfinished code that would be unwieldy or impossible with
  just an expression-oriented approach.

- [Erik Little](https://forums.swift.org/t/pitch-warning/2819/42?) refined that
  to instead use special directives `#warning` and `#error` in expression
  position, like:

  ```swift
  let somethingSuspect = #warning("This is really the wrong function to call, but I'm being lazy", suspectFunction())
  ```
  However, I think there's not much of a benefit to this syntax vs. just
  adding a `#warning` above the line:
  ```swift
  #warning "This is really the wrong function to call, but I'm being lazy"
  let somethingSuspect = suspectFunction()
  ```

- [A few](https://forums.swift.org/t/pitch-warning/2819/39)
  [people](https://forums.swift.org/t/pitch-warning/2819/37) have requested
  `#message` or `#info`, as an analogue for Clang's `#pragma message`. This may
  be something we want, but I didn't include it in this proposal because as of
  this writing, Clang treats `#pragma message` as a warning and flags it
  as `-W#pragma-message`.

## Future directions

Both `#message` and an expression-based `#warning` are additive with respect
to this proposal, and both could be addressed in future proposals.

-------------------------------------------------------------------------------

# Rationale

On February 1, 2018 the Core Team decided to **accept** this proposal with
slight revision over the [original proposal](https://github.com/swiftlang/swift-evolution/blob/ab0c22a2340be9bfcb82e6f237752b4d959a93b7/proposals/0196-diagnostic-directives.md).

The only revision over the original proposal is to change the syntax to use
`#warning(<Message>)` instead of `#warning <Messsage>`.  This fits well with
most of Swift's existing compiler directives, and was strongly supported in
the [review discussion](https://forums.swift.org/t/se-0196-compiler-diagnostic-directives/8734).

The review discussion also covered a variety of possible extensions or
variants to this proposal, including support for using `#warning` [as an
expression](https://forums.swift.org/t/se-0196-compiler-diagnostic-directives/8734/21)
instead of a line directive and [support for runtime issues](https://forums.swift.org/t/se-0196-compiler-diagnostic-directives/8734/6).
The Core Team decided that while these directions are interesting and worth
exploring, they are complementary to the core functionality serviced by this
proposal.  Further, keeping `#warning` as a line directive allows it to be
used in a wide variety of contexts, and serves a different need than using it
as a placeholder expression.
