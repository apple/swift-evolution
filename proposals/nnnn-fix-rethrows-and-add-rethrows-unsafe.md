# Fixing `rethrows` checking and introducing `rethrows(unsafe)`

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Slava Pestov](https://github.com/slavapestov)
* Review Manager: TBD
* Status: **Prototype implemented**
* Implementation: [apple/swift#36007](https://github.com/apple/swift/pull/36007)
* Bugs: [SR-680](https://bugs.swift.org/browse/SR-680)

*During the review process, add the following fields as needed:*

* Decision Notes: [Rationale](https://forums.swift.org/), [Additional Commentary](https://forums.swift.org/)
* Previous Revision: [1](https://github.com/apple/swift-evolution/blob/...commit-ID.../proposals/NNNN-filename.md)
* Previous Proposal: [SE-XXXX](XXXX-filename.md)

## Introduction

This proposal fixes a soundness hole in `rethrows` checking, and introduces
a new `rethrows(unsafe)` escape hatch for situations where a function has
the correct behavior at runtime but the compiler is unable to prove that this
is the case.

Swift-evolution thread: [Discussion thread topic for that proposal](https://forums.swift.org/)

## Motivation

Swift allows you to write a higher-order function which is known to only
throw an error if one if the provided function-typed arguments throws
an error. This is indicated by writing the `rethrows` keyword in place of
`throws` before the return value:

```swift
extension Optional {
  func map<Result>(_ fn: (Wrapped) throws -> Result) rethrows -> Result {
    guard .some(let wrapped) = self else { return nil }
    return .some(try fn(wrapped))
  }
}

let x: Int? = 123
let y = x.map { $0 * 2 } // no 'try' since nothing can be thrown
let z = try x.map { someThrowingFunc($0) } // 'try' is needed
```

Today it is also possible to defeat the compiler's `rethrows` checking
and write a function which always throws even when the passed-in closure
does not throw:

```swift
enum MyError : Error {
  case bad
}
func rethrowsViaClosure(_ fn: () throws -> ()) rethrows {
  try fn()
}
func invalidRethrows(_ fn: () throws -> ()) rethrows {
  try rethrowsViaClosure { throw MyError.bad }
}

invalidRethrows() // no 'try', but crashes at runtime
```

This was originally due to a bug in the implementation, but some
projects have come to rely on it to implement functions that still
only throw if the passed-in closure throws, but the compiler cannot
prove this.

For example, the implementation of `DispatchQueue.sync()` in
[swift-corelibs-libdispatch](https://github.com/apple/swift-corelibs-libdispatch/blob/main/src/swift/Queue.swift) looks like this:

```swift
private func _syncHelper<T>(
    fn: (() -> ()) -> (),
    execute work: () throws -> T,
    rescue: ((Swift.Error) throws -> (T))) rethrows -> T {
  var result: T?
  var error: Swift.Error?
  withoutActuallyEscaping(work) { _work in
    fn {
      do {
        result = try _work()
      } catch let e {
        error = e
      }
    }
  }
  if let e = error {
    return try rescue(e)
  } else {
    return result!
  }
}

public func sync<T>(execute work: () throws -> T) rethrows -> T {
  return try self._syncHelper(fn: sync, execute: work, rescue: { throw $0 })
}
```

The `_syncHelper()` function catches any errors thrown by the `work` parameter inside the inner closure passed to `fn`, then rethrows the error, if there was one, from outside of the closure by passing it to the `rescue` closure.

With the proposed language, the compiler now rejects the definition of `sync()`, since it calls `_syncHelper()`, which `rethrows`, with a `rescue` closure that unconditionally throws.

These situations are rare but do come up, so to handle them, we propose introducing a new `rethrows(unsafe)` variant of the `rethrows` keyword:

```swift
public func sync<T>(execute work: () throws -> T) rethrows(unsafe) -> T {
  return try self._syncHelper(fn: sync, execute: work, rescue: { throw $0 })
}
```

From the perspective of the caller, `sync()` behaves like any other `rethrows` function; the unsafe aspect is an implementation detail.

## Detailed design

Today, a `rethrows` function must obey the following restrictions:

1. It cannot `throw` an error directly.
2. It can call any one of its throwing closure arguments, with the `try` keyword required as usual.
3. It can call any other `rethrows` function.

The soundness issue is with rule 3, because we do not impose restrictions on what arguments may be passed in to the other `rethrows` function.

This proposal leaves rules 1 and 2 unchanged, but introduces a new rule 3:

1. It cannot `throw` an error directly.
2. It can call any one of its throwing closure arguments, with the `try` keyword required as usual.
3. It can call any other `rethrows` function, **as long as the closures that are passed in to the callee are either among the function's original throwing closure arguments, or another closure that in turn obeys the restrictions of rule 1, 2 and 3.**

If the function is declared with the `rethrows(unsafe)` keyword in place of `rethrows`, the rules are not enforced, and it is up to the programmer to ensure that the function only throws if one if its original closure arguments throws.

## Source compatibility

This proposal breaks source compatibility with any code that previously
relied on the soundness hole, either intentionally or unintentionally.

So far, there are three known examples of soundness violations:

1. The implementations of `DispatchQueue.sync()` in [swift-corelibs-libdispatch](https://github.com/apple/swift-corelibs-libdispatch/blob/main/src/swift/Queue.swift) and the [Darwin Dispatch overlay](https://github.com/apple/swift/blob/main/stdlib/public/Darwin/Dispatch/Queue.swift).
2. The implementations of `IndexSet.filteredIndexSet()` in [swift-corelibs-foundation](https://github.com/apple/swift-corelibs-foundation/blob/main/Sources/Foundation/IndexSet.swift) and the [Darwin Foundation overlay](https://github.com/apple/swift/blob/main/stdlib/public/Darwin/Foundation/IndexSet.swift).
3. The implementation of `DatabaseQeueue.read()` in the [GRDB.swift](https://github.com/groue/GRDB.swift/blob/d290102d9cb5c425fee7260034beaa997d581d86/GRDB/Core/DatabaseQueue.swift) open source project from Swift's source compatibility suite.

All three can be addressed with the appropriate use of `rethrows(unsafe)`.

## Effect on ABI stability

The proposed change to `rethrows` checking does not change the ABI of existing code. Changing an existing `rethrows` function to use `rethrows(unsafe)`, or vice versa, is an ABI-preserving change.

## Effect on API resilience

The proposed change to `rethrows` checking does not change the API of existing code. Changing an existing `rethrows` function to use `rethrows(unsafe)`, or vice versa, is an API-preserving change.

## Alternatives considered

We could leave the soundness hole unfixed, but this is suboptimal since users have hit it on accident and reported it as a bug.

We could downgrade violations of the new proposed rules to a warning, preserving source compatibility with code that would otherwise have to use `rethrows(unsafe)`. However, having to change code to use this keyword should be rare in practice, and a warning for a behavior that can crash at runtime will confuse developers.

We could use a separate attribute for `rethrows(unsafe)` instead of new syntax, for example something like this:

```swift
@rethrowsUnsafe func rethrowsUnsafely(_: () throws -> ()) rethrows {}
```
