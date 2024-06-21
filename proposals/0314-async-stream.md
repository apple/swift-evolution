# `AsyncStream` and `AsyncThrowingStream`

* Proposal: [SE-0314](0314-async-stream.md)
* Authors: [Philippe Hausler](https://github.com/phausler), [Tony Parker](https://github.com/parkera), [Ben D. Jones](https://github.com/bendjones), [Nate Cook](https://github.com/natecook1000)
* Review Manager: [Doug Gregor](https://github.com/DougGregor)
* Status: **Implemented (Swift 5.5)**
* Review: ([first review](https://forums.swift.org/t/se-0314-asyncstream-and-asyncthrowingstream/48198), [revision announcement](https://forums.swift.org/t/returned-for-revision-se-0314-asyncstream-and-asyncthrowingstream/49718), [second review](https://forums.swift.org/t/se-0314-second-review-asyncstream-and-asyncthrowingstream/49803), [acceptance announcement](https://forums.swift.org/t/accepted-se-0314-asyncstream-and-asyncthrowingstream/50699))
* Implementation: [apple/swift#36921](https://github.com/apple/swift/pull/36921)

#### Change Log

Changes for the second review:
* added `YieldResult` to express the action of yielding’s impact, either something is enqueued, dropped or the continuation is already terminated
* added `init(unfolding: @escaping () async -> Element?)` to offer an initializer for unfolding to handle back-pressure based APIs.
* made `AsyncThrowingStream` generic on Failure but the initializers only afford for creation `where Failure == Error`
* removed the example of `DispatchSource` signals since the other `DispatchSource` types might be actively harmful to use in *any* async context
* initialization now takes a buffering policy to both restrict the buffer size as well as configure how elements are dropped

## Introduction

The continuation types added in [SE-0300](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0300-continuation.md) act as adaptors for synchronous code that signals completion by calling a delegate method or callback function. For code that instead yields multiple values over time, this proposal adds new types to support implementing an `AsyncSequence` interface.

Swift-evolution threads:

- [[Pitch] AsyncStream and AsyncThrowingStream](https://forums.swift.org/t/pitch-asyncstream-and-asyncthrowingstream/47820)
- [[Concurrency] YieldingContinuation](https://forums.swift.org/t/concurrency-yieldingcontinuation/47126)

## Motivation

Swift’s new `async` / `await` features include support for adapting callback- or delegate-based APIs using `UnsafeContinuation` or `CheckedContinuation` . These single-use continuations work well for adapting APIs like the `getInt(completion:)` function defined here into asynchronous ones:

```swift
func getInt(completion: @escaping (Int) -> Void) {
    DispatchQueue(label: "myQueue").async {
        sleep(1)
        completion(42)
    }
}
```

By calling one of the `with``*Continuation(_:)` functions, you can suspend the current task and resume it with a result or an error using the provided continuation.

```swift
func getInt() async -> Int {
    await withUnsafeContinuation { continuation in
        getInt(completion: { result in
            continuation.resume(returning: result)
        }
    }
}
```

This provides a great experience for APIs that asynchronously produce a single result, but some operations produce many values over time instead. Rather than being adapted to an `async` function, the appropriate solution for these operations is to create an `AsyncSequence` .

Repeating asynchronous operations typically separate the consumption of values from their location of use, either within a callback function or a delegate method. Given an existing API that offers an interface as the following:

```swift
class QuakeMonitor {
  var quakeHandler: (Quake) -> Void
  func startMonitoring()
  func stopMonitoring()
}
```

The usage of this pattern would work similarly to this:

```swift
let monitor = QuakeMonitor()
monitor.quakeHandler { quake in
  // ...
}
monitor.startMonitoring() // start sending quakes to the handler
...
monitor.stopMonitoring() // cancel the quakes being sent to the handler
```

The same is true for delegates that are informative only, and need no feedback or exclusivity of execution to be valid. As one example, the AppKit [ `NSSpeechRecognizerDelegate` ](https://developer.apple.com/documentation/appkit/nsspeechrecognizerdelegate) is called whenever the system recognizes a spoken command:

```swift
if let recognizer = NSSpeechRecognizer() {
    class CommandDelegate: NSObject, NSSpeechRecognizerDelegate {
        func speechRecognizer(
             _ sender: NSSpeechRecognizer,
            didRecognizeCommand command: String) 
        {
            // do something on each recognized command
        }
    }
    let delegate = CommandDelegate()
    recognizer.delegate = delegate
    ...
}
```

Both of these examples represent common design patterns in many applications, frameworks, and libraries. These delegate methods and callbacks are asynchronous in nature, but cannot be annotated as `async` functions because they don’t have a singular return value. While not all delegates or callbacks are suitable to be represented as asynchronous sequences, there are numerous enough cases that we would like to offer a safe and expressive way to represent producers that can yield multiple values or errors.

## Proposed Solution

In order to fill this gap, we propose adding two new types: `AsyncStream` and `AsyncThrowingStream` . These types fill a role similar to continuations, bridging the gap from non `async` / `await` based asynchronous behavior into the world of `async` / `await` . We anticipate that types that currently provide multiple-callback or delegate interfaces to asynchronous behavior can use these `AsyncStream` types to provide an interface for within `async` contexts.

The two `AsyncStream` types each include a nested `Continuation` type; these outer and inner types represent the consuming and producing sides of operation, respectively. You send values, errors, and “finish” events via the `Continuation` , and clients consume those values and errors through the `AsyncStream` type’s `AsyncSequence` interface.

### Creating a non-throwing `AsyncStream`

When you create an `AsyncStream` instance, you specify the element type and pass a closure that operates on the series’s `Continuation` . You can yield values to this continuation type *multiple* times, and the series buffers any yielded elements until they are consumed via iteration.

The `QuakeMonitor` above can be given an `AsyncStream` interface this way:

```swift
extension QuakeMonitor {
  static var quakes: AsyncStream<Quake> {
    AsyncStream { continuation in
      let monitor = QuakeMonitor()
      monitor.quakeHandler { quake in
        continuation.yield(quake)
      }
      continuation.onTermination = { _ in 
        monitor.stopMonitoring()
      }
      monitor.startMonitoring()
    }
  }
}

// elsewhere...

for await quake in QuakeMonitor.quakes {
    // ...
}
```

As each value is passed to the `QuakeMonitor` ’s event handler closure, the call to `continuation.yield(_:)` stores the value for access by a consumer of the sequence. With this implementation, quake events are buffered as they come in, and only consumed when an iterator requests a value.

Alternatively if a source is just an async function (one that represents a backpressure) an AsyncStream can be constructed by unfolding a producing function and a cancellation handler. This affords the case in which that unfolded function can leave the specification adherence of AsyncSequence to AsyncStream. In short the `init(unfolding:onCancel:)` initializer handles the terminal cases as well as the cancellation. 

### Creating an `AsyncThrowingStream`

Along with the potentially infinite sequence in the example above, `AsyncSeries` can also adapt APIs like the slightly contrived one below. The `findVegetables` function uses callback closures that are called with each retrieved vegetable, as well as when the vegetables have all been returned or an error occurs.

```swift
func buyVegetables(
  shoppingList: [String],

  // a) invoked once for each vegetable in the shopping list
  onGotVegetable: (Vegetable) -> Void,

  // b) invoked once all available veggies have been retrieved
  onAllVegetablesFound: () -> Void,

  // c) invoked if a non-vegetable food item is encountered
  // in the shopping list
  onNonVegetable: (Error) -> Void
)

// Returns a stream of veggies
func findVegetables(shoppingList: [String]) -> AsyncThrowingStream<Vegetable> {
  AsyncThrowingStream { continuation in
    buyVegetables(
      shoppingList: shoppingList,
      onGotVegetable: { veggie in continuation.yield(veggie) },
      onAllVegetablesFound: { continuation.finish() },
      onNonVegetable: { error in continuation.finish(throwing: error) }
    )
  }
}
```

Note that a call to the `finish()` method is required to end iteration for the consumer of an `AsyncStream` . Any buffered elements are provided to the sequence consumer before finishing with either a simple terminating `nil` or a thrown error.

### Awaiting Values

An `AsyncStream` provides an `AsyncSequence` interface to its values, so you can iterate over the elements in an `AsyncStream` by using `for` - `in` , or use any of the `AsyncSequence` methods added as part of [SE-0298](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0298-asyncsequence.md).

```swift
for await notif in NotificationCenter
      .notifications(for: ...)
      .prefix(3) 
{
    // update with notif
}
```

You may also create an iterator directly, and call its `next()` method for more control over iteration. Each call to `next()` , either through `for` - `in` iteration, an `AsyncSequence` method, or direct calls on an iterator, either immediately returns the earliest buffered element or `await` s the next element yielded to the stream’s continuation.

As with any sequence, iterating over an `AsyncStream` multiple times, or creating multiple iterators and iterating over them separately, may produce an unexpected series of values. Neither `AsyncStream` nor its iterator are `@Sendable` types, and concurrent iteration is considered a programmer error.

## Detailed design

The full API of `AsyncStream` , `AsyncThrowingStream` , and their nested `Continuation` and `Iterator` types are as follows:

```swift
/// An ordered, asynchronously generated sequence of elements.
///
/// AsyncStream is an interface type to adapt from code producing values to an
/// asynchronous context iterating them. This is intended to allow
/// callback or delegation based APIs to participate with async/await.
///
/// AsyncStream can be initialized with the option to buffer to a given limit.
/// The default value for this limit is Int.max. The buffering is only for
/// values that have yet to be consumed by iteration. Values can be yielded 
/// to the continuation passed into the closure. That continuation
/// is Sendable, in that it is intended to be used from concurrent contexts
/// external to the iteration of the stream.
///
/// A trivial use case producing values from a detached task would work as such:
///
///     let digits = AsyncStream(Int.self) { continuation in
///       detach {
///         for digit in 0..<10 {
///           continuation.yield(digit)
///         }
///         continuation.finish()
///       }
///     }
///
///     for await digit in digits {
///       print(digit)
///     }
///
public struct AsyncStream<Element> {
  public struct Continuation: Sendable {
    /// Indication of the type of termination informed to 
    /// `onTermination`. 
    public enum Termination {
      
      /// The stream was finished via the `finish` method
      case finished
      
      /// The stream was cancelled
      case cancelled
    }
    
    /// A result of yielding values.
    public enum YieldResult {
    
      /// When a value is successfully enqueued, either buffered
      /// or immediately consumed to resume a pending call to next
      /// and a count of remaining slots available in the buffer at
      /// the point in time of yielding. Note: transacting upon the
      /// remaining count is only valid when then calls to yield are
      /// mutually exclusive.
      case enqueued(remaining: Int)
      
      /// Yielding resulted in not buffering an element because the 
      /// buffer was full. The element is the dropped value.
      case dropped(Element)
      
      /// Indication that the continuation was yielded when the 
      /// stream was already in a terminal state: either by cancel or
      /// by finishing.
      case terminated
    }
    
    /// A strategy that handles exhaustion of a buffer’s capacity.
    public enum BufferingPolicy {
      case unbounded

      /// When the buffer is full, discard the newly received element.
      /// This enforces keeping the specified amount of oldest values.
      case bufferingOldest(Int)
      
      /// When the buffer is full, discard the oldest element in the buffer.
      /// This enforces keeping the specified amount of newest values.
      case bufferingNewest(Int)
    }
    
    /// Resume the task awaiting the next iteration point by having it return
    /// normally from its suspension point. Buffer the value if nothing is awaiting
    /// the iterator.
    ///
    /// - Parameter value: The value to yield from the continuation.
    ///
    /// This can be called more than once and returns to the caller immediately
    /// without blocking for any awaiting consumption from the iteration.
    ///
    /// The `yield(_:)` function returns the state of any value yielded to the
    /// continuation. This can be one of three states: `enqueued`, `dropped` or
    /// `terminated`. Each of the states respectively represents if the value
    /// was either buffered or resumed to active iteration, dropped because
    /// the limit of the buffer was reached, or dropped because the AsyncStream
    /// was at a terminal state either from being finished or cancelled.
    @discardableResult
    public func yield(_ value: Element) -> YieldResult

    /// Resume the task awaiting the next iteration point by having it return
    /// nil. This signifies the end of the iteration.
    ///
    /// Calling this function more than once is idempotent. All values received
    /// from the iterator after it is finished and after the buffer is exhausted 
    /// are nil.
    public func finish()

    /// A callback to invoke when iteration of a AsyncStream is canceled.
    ///
    /// If an onTermination callback is set, when iteration of an AsyncStream is
    /// canceled via task cancellation that callback is invoked. The callback
    /// is disposed of after any terminal state is reached.
    ///
    /// Canceling an active iteration will first invoke the onCancel callback
    /// and then resume yielding nil. This means that any cleanup state can be
    /// emitted accordingly in the cancellation handler.
    public var onTermination: (@Sendable (Termination) -> Void)? { get nonmutating set }
  }

  /// Construct an AsyncStream buffering given an Element type.
  ///
  /// - Parameter elementType: The type the AsyncStream will produce.
  /// - Parameter bufferingPolicy: The policy in which to buffer elements.
  /// This controls the amount that can potentially be stored in the buffer
  /// and the mechanism in which to drop values. 
  /// - Parameter build: The work associated with yielding values to the 
  ///   AsyncStream.
  ///
  /// The maximum number of pending elements is enforced by dropping the oldest
  /// value when a new value comes in. By default this limit is unlimited.
  /// A value of 0 results in immediate dropping of values if there is no current
  /// await on the iterator.
  ///
  /// The build closure passes in a Continuation which can be used in
  /// concurrent contexts. It is thread safe to yield and finish. All calls are
  /// to the continuation are serialized. However, calling yield from multiple
  /// concurrent contexts could result in out of order delivery.
  public init(
    _ elementType: Element.Type = Element.self,
    bufferingPolicy limit: Continuation.BufferingPolicy = .unbounded,
    _ build: (Continuation) -> Void
  ) {
  
  /// Construct an AsyncStream by unfolding the application of a function.
  ///
  /// - Parameter produce: The function to call when calculating the next value.
  /// - Parameter onCancel: A closure to call when the AsyncStream is cancelled.
  ///
  /// Construction with this initializer handles the rules of AsyncSequence in
  /// that after a nil is produced subsequent calls must produce nil.
  public init(
    unfolding produce: @escaping () async -> Element?, 
    onCancel: (@Sendable () -> Void)? = nil
  )
}

extension AsyncStream: AsyncSequence {
  /// The asynchronous iterator for iterating an AsyncStream.
  ///
  /// This type is not Sendable. It is not intended to be used
  /// from multiple concurrent contexts. Any such case that next is invoked
  /// concurrently and contends with another call to next is a programmer error
  /// and will fatalError.
  public struct Iterator: AsyncIteratorProtocol {
    public mutating func next() async -> Element?
  }

  /// Construct an iterator.
  public func makeAsyncIterator() -> Iterator
}

extension AsyncStream.Continuation {
  /// Resume the task awaiting the next iteration point by having it return
  /// normally from its suspension point or buffer the value if no awaiting
  /// next iteration is active.
  ///
  /// - Parameter result: A result to yield from the continuation.
  ///
  /// This can be called more than once and returns to the caller immediately
  /// without blocking for any awaiting consumption from the iteration.
  @discardableResult
  public func yield(
    with result: Result<Element, Never>
  ) -> YieldResult

  /// Resume the task awaiting the next iteration point by having it return
  /// normally from its suspension point or buffer the value if no awaiting
  /// next iteration is active where the `Element` is `Void`.
  ///
  /// This can be called more than once and returns to the caller immediately
  /// without blocking for any awaiting consumption from the iteration.
  @discardableResult
  public func yield() -> YieldResult where Element == Void
}

public struct AsyncThrowingStream<Element, Failure: Error> {
  public struct Continuation: Sendable {
    public enum Termination {
      case finished(Failure?)
      case cancelled
    }
    
    public enum YieldResult {
      case enqueued
      case dropped
      case terminated
    }
    
    /// * See AsyncStream.Continuation.yield(_:) *
    @discardableResult
    public func yield(_ value: Element) -> YieldResult

    /// Resume the task awaiting the next iteration point with a terminal state.
    /// If error is nil, this is a completion with out error. If error is not 
    /// nil, then the error is thrown. Both of these states indicate the 
    /// end of iteration.
    ///
    /// - Parameter error: The error to throw or nil to signify termination.
    ///
    /// Calling this function more than once is idempotent. All values received
    /// from the iterator after it is finished and after the buffer is exhausted 
    /// are nil.
    public func finish(throwing error: Failure? = nil)

    /// * See AsyncStream.Continuation.onTermination *
    public var onTermination: (@Sendable (Termination) -> Void)? { get nonmutating set }
  }

  /// * See AsyncStream.init *
  public init(
    _ elementType: Element.Type,
    maxBufferedElements limit: Int = .max,
    _ build: (Continuation) -> Void
  ) where Failure == Error
  
  /// Construct an AsyncStream by unfolding the application of a function.
  ///
  /// - Parameter produce: The function to call when calculating the next value.
  /// - Parameter onCancel: A closure to call when the AsyncStream is cancelled.
  ///
  /// Construction with this initializer handles the rules of AsyncSequence in
  /// that after a nil is produced subsequent calls must produce nil.
  public init(
    unfolding produce: @escaping () async throws -> Element?, 
    onCancel: (@Sendable () -> Void)? = nil
  )
}

extension AsyncThrowingStream: AsyncSequence {
  public struct Iterator: AsyncIteratorProtocol {
    public mutating func next() async throws -> Element?
  }

  public func makeAsyncIterator() -> Iterator
}

extension AsyncThrowingStream.Continuation {
  /// Resume the task awaiting the next iteration point by having it return
  /// normally from its suspension point or buffer the value if no awaiting
  /// next iteration is active.
  ///
  /// - Parameter result: A result to yield from the continuation.
  ///
  /// This can be called more than once and returns to the caller immediately
  /// without blocking for any awaiting consumption from the iteration.
  @discardableResult
  public func yield(
    with result: Result<Element, Failure>
  ) -> YieldResult

  /// * See AsyncStream.yield() *
  @discardableResult
  public func yield() -> YieldResult where Element == Void
}
```

### Yielding Values

In some cases it is meaningful to manage the potential emissions in accordance with what the state of yielding values to the continuation might do. There are three potential states of yielding a value, it may be enqueued to the buffer (or even immediately resumed to an active iterator), dropped because the limit of the buffer has been reached, or dropped because the `AsyncStream` or `AsyncThrowingStream` have been terminated (either by finish or cancel). This is a meaningful return value in some cases but generally this is not always a meaningful/useful state. In the cases that it is meaningful; consumers may want to emit an error when the buffer has reached its limit. Or consumers may want to concatenate the value to the next emitted value. 

The yield function of the `Continuation` type for both `AsyncStream` and `AsyncThrowingStream` returns the state in which that yield transacted - it can return that the value was enqueued with a remaining count of slots available at the time of the yield for the backing buffer, or if the buffer is full it will return the element that was dropped, or if the stream was already terminal it returns an indication of that terminal state.

### Buffering Values

By default, every element yielded to an `AsyncStream` ’s continuation is buffered until consumed by iteration. This matches the expectation for most of the APIs we anticipate being adapted via `AsyncStream` — with a stream of notifications, database records, or other similar types, the caller needs to receive every single one.

If the caller specifies a different value *n* for `maxBufferedElements` , then the most recent *n* elements are buffered until consumed by iteration. If the caller specifies `0` , the stream switches to a dropping behavior, dropping the value if nothing is `await` ing the iterator’s `next` .

### Backpressure

`AsyncStream` and `AsyncThrowingStream` both are types intended to interface systems in which back pressure is not present to the `AsyncSequence` interface which is a back pressure based system. `AsyncSequence` via it's iterator is back pressure based via the `next` method on the iterator. Each call to next is an asynchronous call that represents an applied demand of 1. This means that systems in which back pressure is not present; like callbacks that are called more than once, or some "informative" style delegates there must be some intermediary to offer behavior of either buffering, dropping or blocking. These three options are the only available mechanisms to stride that gap between the back pressure world and the non back pressure world. `AsyncStream` does not aim to resolve the blocking scenario (any callbacks that require that type of functionality are probably ill suited for async/await anyhow). The buffering and dropping scenarios can be represented with just buffering since the dropping scenario is just a buffer of 0 items.

### Finishing the Stream

Calling a continuation’s `finish()` method moves its stream into a “terminated” state. An implementor of an `AsyncThrowingStream` can optionally pass an error to be thrown by the iterator. After providing all buffered elements, the stream’s iterator will return `nil` or throw an error, as appropriate.

The first call to `finish()` sets the terminating behavior of the stream (either returning `nil` or throwing); further calls to `finish()` or `yield(_:)` once a stream is in a terminated state have no effect.

### Cancellation Handlers

When defined, a continuation’s `onTermination` handler function is called when iteration ends, when the stream goes out of scope, or when the task containing the stream is canceled. You can safely use the `onTermination` handler to clean up resources that were opened or allocated at the start of the stream.

This `onTermination` behavior is shown in the following example — once the task containing the stream is canceled, the `onTermination` handler for the stream’s continuation is called:

```swift
let t = detach {
  func make123Stream() -> AsyncStream<Int> {
    AsyncStream { continuation in
      continuation.onTermination = { termination in
        switch termination {
        case .finished:
            print("Regular finish")
        case .cancelled:
            print("Cancellation")
        }
      }
      detach {
        for n in 1...3 {
          continuation.yield(n)
          sleep(2)
        }
        continuation.finish()
      }
    }
  }

  for await n in make123Stream() {
    print("for-in: \(n)")
  }
  print("After")
}
sleep(3)
t.cancel()

// for-in: 1
// for-in: 2
// Cancellation
// After
```

### Convenience Methods

As conveniences, both `AsyncStream` continuations include a `yield(with:)` method that takes a `Result` as a parameter and, when the stream’s `Element` type is `Void` , a `yield()` method that obviates the need for passing an empty tuple.

## Alternatives considered

### YieldingContinuation

A `YieldingContinuation` type was pitched previously in [this forum thread](https://forums.swift.org/t/concurrency-yieldingcontinuation/47126), designed as a single type that could be yielded to and awaited on. `AsyncStream` adds automatic buffering and a simpler API to support most common use cases.

### Users use `os_unfair_lock` / `unlock` or other locking primitives

The buffering behavior in `AsyncStream` requires the use of locks in order to be thread safe. Without a native locking mechanism, Swift developers would be left to write their own custom (and potentially per-platform) locks. We think it is better for a standard library solution to be provided that removes this complexity and potential source of errors.

## Source compatibility

This proposal is purely additive and has no direct impact on existing source.

## Effect on ABI stability

This proposal is purely additive and has no direct impact on ABI stability.

## Effect on API resilience

The current implementation leaves room for future development of this type to offer different construction mechanisms and is encapsulated into it's own type hierarchy so it has no immediate impact upon API resilience and is future-proofed for future development.

## Acknowledgments

We would like to thank [Jon Shier](https://forums.swift.org/u/jon_shier) and [David Nadoba](https://forums.swift.org/u/dnadoba) for their helpful insight and feedback driving the discussion on YieldingContinuation to make us look further into making a more well rounded solution. Special thanks go to the creators of the [reactive-streams specification ](https://github.com/reactive-streams/reactive-streams-jvm/blob/v1.0.3/README.md#specification), without which numerous behavioral edge cases would not have been considered.
