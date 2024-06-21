# Async/Await: Sequences

* Proposal: [SE-0298](0298-asyncsequence.md)
* Authors: [Tony Parker](https://github.com/parkera), [Philippe Hausler](https://github.com/phausler)
* Review Manager: [Doug Gregor](https://github.com/DougGregor)
* Status: **Implemented (Swift 5.5)**
* Implementation: [apple/swift#35224](https://github.com/apple/swift/pull/35224)
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-with-modification-se-0298-async-await-sequences/44231)
* Revision: Based on [forum discussion](https://forums.swift.org/t/pitch-clarify-end-of-iteration-behavior-for-asyncsequence/45548)

## Introduction

Swift's [async/await](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0296-async-await.md) feature provides an intuitive, built-in way to write and use functions that return a single value at some future point in time. We propose building on top of this feature to create an intuitive, built-in way to write and use functions that return many values over time.

This proposal is composed of the following pieces:

1. A standard library definition of a protocol that represents an asynchronous sequence of values
2. Compiler support to use `for...in` syntax on an asynchronous sequence of values
3. A standard library implementation of commonly needed functions that operate on an asynchronous sequence of values

## Motivation

We'd like iterating over asynchronous sequences of values to be as easy as iterating over synchronous sequences of values. An example use case is iterating over the lines in a file, like this:

```swift
for try await line in myFile.lines() {
  // Do something with each line
}
```

Using the `for...in` syntax that Swift developers are already familiar with will reduce the barrier to entry when working with asynchronous APIs. Consistency with other Swift types and concepts is therefore one of our most important goals. The requirement of using the `await` keyword in this loop will distinguish it from synchronous sequences.
### `for/in` Syntax

To enable the use of `for in`, we must define the return type from `func lines()` to be something that the compiler understands can be iterated. Today, we have the `Sequence` protocol. Let's try to use it here:

```swift
extension URL {
  struct Lines: Sequence { /* ... */ }
  func lines() async -> Lines
}
```

Unfortunately, what this function actually does is wait until *all* lines are available before returning. What we really wanted in this case was to await *each* line. While it is possible to imagine modifications to `lines` to behave differently (e.g., giving the result reference semantics), it would be better to define a new protocol to make this iteration behavior as simple as possible.

```swift
extension URL {
  struct Lines: AsyncSequence { /* ... */ }
  func lines() async -> Lines
}
```

`AsyncSequence` allows for waiting on each element instead of the entire result by defining an asynchronous `next()` function on its associated iterator type.

### Additional AsyncSequence functions

Going one step further, let's imagine how it might look to use our new `lines` function in more places. Perhaps we want to process lines until we reach one that is greater than a certain length.

```swift
let longLine: String?
do {
  for try await line in myFile.lines() {
    if line.count > 80 {
      longLine = line
      break
    }
  }
} catch {
  longLine = nil // file didn't exist
}
```

Or, perhaps we actually do want to read all lines in the file before starting our processing:

```swift
var allLines: [String] = []
do {
  for try await line in myFile.lines() {
    allLines.append(line)
  }
} catch {
  allLines = []
}
```

There's nothing wrong with the above code, and it must be possible for a developer to write it. However, it does seem like a lot of boilerplate for what might be a common operation. One way to solve this would be to add more functions to `URL`:

```swift
extension URL {
  struct Lines : AsyncSequence { }

  func lines() -> Lines
  func firstLongLine() async throws -> String?
  func collectLines() async throws -> [String]
}
```

It doesn't take much imagination to think of other places where we may want to do similar operations, though. Therefore, we believe the best place to put these functions is instead as an extension on `AsyncSequence` itself, specified generically -- just like `Sequence`.

## Proposed solution

The standard library will define the following protocols:

```swift
public protocol AsyncSequence {
  associatedtype AsyncIterator: AsyncIteratorProtocol where AsyncIterator.Element == Element
  associatedtype Element
  __consuming func makeAsyncIterator() -> AsyncIterator
}

public protocol AsyncIteratorProtocol {
  associatedtype Element
  mutating func next() async throws -> Element?
}
```

The compiler will generate code to allow use of a `for in` loop on any type which conforms with `AsyncSequence`. The standard library will also extend the protocol to provide familiar generic algorithms. Here is an example which does not actually call an `async` function within its `next`, but shows the basic shape:

```swift
struct Counter : AsyncSequence {
  let howHigh: Int

  struct AsyncIterator : AsyncIteratorProtocol {
    let howHigh: Int
    var current = 1
    mutating func next() async -> Int? {
      // We could use the `Task` API to check for cancellation here and return early.
      guard current <= howHigh else {
        return nil
      }

      let result = current
      current += 1
      return result
    }
  }

  func makeAsyncIterator() -> AsyncIterator {
    return AsyncIterator(howHigh: howHigh)
  }
}
```

At the call site, using `Counter` would look like this:

```swift
for await i in Counter(howHigh: 3) {
  print(i)
}

/* 
Prints the following, and finishes the loop:
1
2
3
*/


for await i in Counter(howHigh: 3) {
  print(i)
  if i == 2 { break }
}
/*
Prints the following:
1
2
*/
```

## Detailed design

Returning to our earlier example:

```swift
for try await line in myFile.lines() {
  // Do something with each line
}
```

The compiler will emit the equivalent of the following code:

```swift
var it = myFile.lines().makeAsyncIterator()
while let line = try await it.next() {
  // Do something with each line
}
```

All of the usual rules about error handling apply. For example, this iteration must be surrounded by `do/catch`, or be inside a `throws` function to handle the error. All of the usual rules about `await` also apply. For example, this iteration must be inside a context in which calling `await` is allowed like an `async` function.

### Cancellation

`AsyncIteratorProtocol` types should use the cancellation primitives provided by Swift's `Task` API, part of [structured concurrency](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md). As described there, the iterator can choose how it responds to cancellation. The most common behaviors will be either throwing `CancellationError` or returning `nil` from the iterator. 

If an `AsyncIteratorProtocol` type has cleanup to do upon cancellation, it can do it in two places:

1. After checking for cancellation using the `Task` API.
2. In its `deinit` (if it is a class type).

### Rethrows

This proposal will take advantage of a separate proposal to add specialized `rethrows` conformance in a protocol, pitched [here](https://forums.swift.org/t/pitch-rethrowing-protocol-conformances/42373). With the changes proposed there for `rethrows`, it will not be required to use `try` when iterating an `AsyncSequence` which does not itself throw.

The `await` is always required because the definition of the protocol is that it is always asynchronous.

### End of Iteration

After an `AsyncIteratorProtocol` types returns `nil` or throws an error from its `next()` method, all future calls to `next()` must return `nil`. This matches the behavior of `IteratorProtocol` types and is important, since calling an iterator's `next()` method is the only way to determine whether iteration has finished.

## AsyncSequence Functions

The existence of a standard `AsyncSequence` protocol allows us to write generic algorithms for any type that conforms to it. There are two categories of functions: those that return a single value (and are thus marked as `async`), and those that return a new `AsyncSequence` (and are not marked as `async` themselves).

The functions that return a single value are especially interesting because they increase usability by changing a loop into a single `await` line. Functions in this category are `first`, `contains`, `min`, `max`, `reduce`, and more. Functions that return a new `AsyncSequence` include `filter`, `map`, and `compactMap`.

### AsyncSequence to single value

Algorithms that reduce a for loop into a single call can improve readability of code. They remove the boilerplate required to set up and iterate a loop.

For example, here is the `contains` function:

```swift
extension AsyncSequence where Element : Equatable {
  public func contains(_ value: Element) async rethrows -> Bool
}
```

With this extension, our "first long line" example from earlier becomes simply:

```swift
let first = try? await myFile.lines().first(where: { $0.count > 80 })
```

Or, if the sequence should be processed asynchonously and used later:

```swift
async let first = myFile.lines().first(where: { $0.count > 80 })

// later

warnAboutLongLine(try? await first)
```

The following functions will be added to `AsyncSequence`:

| Function | Note |
| - | - |
| `contains(_ value: Element) async rethrows -> Bool` | Requires `Equatable` element |
| `contains(where: (Element) async throws -> Bool) async rethrows -> Bool` | The `async` on the closure allows optional async behavior, but does not require it |
| `allSatisfy(_ predicate: (Element) async throws -> Bool) async rethrows -> Bool` | |
| `first(where: (Element) async throws -> Bool) async rethrows -> Element?` | |
| `min() async rethrows -> Element?` | Requires `Comparable` element |
| `min(by: (Element, Element) async throws -> Bool) async rethrows -> Element?` | |
| `max() async rethrows -> Element?` | Requires `Comparable` element |
| `max(by: (Element, Element) async throws -> Bool) async rethrows -> Element?` | |
| `reduce<T>(_ initialResult: T, _ nextPartialResult: (T, Element) async throws -> T) async rethrows -> T` | |
| `reduce<T>(into initialResult: T, _ updateAccumulatingResult: (inout T, Element) async throws -> ()) async rethrows -> T` | |

### AsyncSequence to AsyncSequence

These functions on `AsyncSequence` return a result which is itself an `AsyncSequence`. Due to the asynchronous nature of `AsyncSequence`, the behavior is similar in many ways to the existing `Lazy` types in the standard library. Calling these functions does not eagerly `await` the next value in the sequence, leaving it up to the caller to decide when to start that work by simply starting iteration when they are ready.

As an example, let's look at `map`:

```swift
extension AsyncSequence {
  public func map<Transformed>(
    _ transform: @escaping (Element) async throws -> Transformed
  ) -> AsyncMapSequence<Self, Transformed>
}

public struct AsyncMapSequence<Upstream: AsyncSequence, Transformed>: AsyncSequence {
  public let upstream: Upstream
  public let transform: (Upstream.Element) async throws -> Transformed
  public struct Iterator : AsyncIterator { 
    public mutating func next() async rethrows -> Transformed?
  }
}
```

For each of these functions, we first define a type which conforms with the `AsyncSequence` protocol. The name is modeled after existing standard library `Sequence` types like `LazyDropWhileCollection` and `LazyMapSequence`. Then, we add a function in an extension on `AsyncSequence` which creates the new type (using `self` as the `upstream`) and returns it.

| Function |
| - |
| `map<T>(_ transform: (Element) async throws -> T) -> AsyncMapSequence` |
| `compactMap<T>(_ transform: (Element) async throws -> T?) -> AsyncCompactMapSequence` |
| `flatMap<SegmentOfResult: AsyncSequence>(_ transform: (Element) async throws -> SegmentOfResult) async rethrows -> AsyncFlatMapSequence` |
| `drop(while: (Element) async throws -> Bool) async rethrows -> AsyncDropWhileSequence` |
| `dropFirst(_ n: Int) async rethrows -> AsyncDropFirstSequence` |
| `prefix(while: (Element) async throws -> Bool) async rethrows -> AsyncPrefixWhileSequence` |
| `prefix(_ n: Int) async rethrows -> AsyncPrefixSequence` |
| `filter(_ predicate: (Element) async throws -> Bool) async rethrows -> AsyncFilterSequence` |

## Future Proposals

The following topics are things we consider important and worth discussion in future proposals:

### Additional `AsyncSequence` functions

We've aimed for parity with the most relevant `Sequence` functions. There may be others that are worth adding in a future proposal.

API which uses a time argument must be coordinated with the discussion about `Executor` as part of the [structured concurrency proposal](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md).

We would like a `first` property, but properties cannot currently be `async` or `throws`. Discussions are ongoing about adding a capability to the language to allow effects on properties. If those features become part of Swift then we should add a `first` property to `AsyncSequence`.

### AsyncSequence Builder

In the standard library we have not only the `Sequence` and `Collection` protocols, but concrete types which adopt them (for example, `Array`). We will need a similar API for `AsyncSequence` that makes it easy to construct a concrete instance when needed, without declaring a new type and adding protocol conformance.

## Source compatibility

This new functionality will be source compatible with existing Swift.

## Effect on ABI stability

This change is additive to the ABI.

## Effect on API resilience

This change is additive to API.

## Alternatives considered

### Explicit Cancellation

An earlier version of this proposal included an explicit `cancel` function. We removed it for the following reasons:

1. Reducing the requirements of implementing `AsyncIteratorProtocol` makes it simpler to use and easier to understand. The rules about when `cancel` would be called, while straightforward, would nevertheless be one additional thing for Swift developers to learn.
2. The structured concurrency proposal already includes a definition of cancellation that works well for `AsyncSequence`. We should consider the overall behavior of cancellation for asynchronous code as one concept.

### Asynchronous Cancellation

If we used explicit cancellation, the `cancel()` function on the iterator could be marked as `async`. However, this means that the implicit cancellation done when leaving a `for/in` loop would require an implicit `await` -- something we think is probably too much to hide from the developer. Most cancellation behavior is going to be as simple as setting a flag to check later, so we leave it as a synchronous function and encourage adopters to make cancellation fast and non-blocking.

### Opaque Types

Each `AsyncSequence`-to-`AsyncSequence` algorithm will define its own concrete type. We could attempt to hide these details behind a general purpose type eraser. We believe leaving the types exposed gives us (and the compiler) more optimization opportunities. A great future enhancement would be for the language to support `some AsyncSequence where Element=...`-style syntax, allowing hiding of concrete `AsyncSequence` types at API boundaries.

### Reusing Sequence

If the language supported a `reasync` concept, then it seems plausible that the `AsyncSequence` and `Sequence` APIs could be merged. However, we believe it is still valuable to consider these as two different types. The added complexity of a time dimension in asynchronous code means that some functions need more configuration options or more complex implementations. Some algorithms that are useful on asynchronous sequences are not meaningful on synchronous ones. We prefer not to complicate the API surface of the synchronous collection types in these cases.

### Naming

The names of the concrete `AsyncSequence` types is designed to mirror existing standard library API like `LazyMapSequence`. Another option is to introduce a new pattern with an empty enum or other namespacing mechanism.

We considered `AsyncGenerator` but would prefer to leave the `Generator` name for future language enhancements. `Stream` is a type in Foundation, so we did not reuse it here to avoid confusion.

### `await in`

We considered a shorter syntax of `await...in`. However, since the behavior here is fundamentally a loop, we feel it is important to use the existing `for` keyword as a strong signal of intent to readers of the code. Although there are a lot of keywords, each one has purpose and meaning to readers of the code.

### Add APIs to iterator instead of sequence

We discussed applying the fundamental API (`map`, `reduce`, etc.) to `AsyncIteratorProtocol` instead of `AsyncSequence`. There has been a long-standing (albeit deliberate) ambiguity in the `Sequence` API -- is it supposed to be single-pass or multi-pass? This new kind of iterator & sequence could provide an opportunity to define this more concretely.

While it is tempting to use this new API to right past wrongs, we maintain that the high level goal of consistency with existing Swift concepts is more important. 

For example, `for...in` cannot be used on an `IteratorProtocol` -- only a `Sequence`. If we chose to make `AsyncIteratorProtocol` use `for...in` as described here, that leaves us with the choice of either introducing an inconsistency between `AsyncIteratorProtocol` and `IteratorProtocol` or giving up on the familiar `for...in` syntax. Even if we decided to add `for...in` to `IteratorProtocol`, it would still be inconsistent because we would be required to leave `for...in` syntax on the existing `Sequence`.

Another point in favor of consistency is that implementing an `AsyncSequence` should feel familiar to anyone who knows how to implement a `Sequence`.

We are hoping for widespread adoption of the protocol in API which would normally have instead used a `Notification`, informational delegate pattern, or multi-callback closure argument. In many of these cases we feel like the API should return the 'factory type' (an `AsyncSequence`) so that it can be iterated again. It will still be up to the caller to be aware of any underlying cost of performing that operation, as with iteration of any `Sequence` today.
