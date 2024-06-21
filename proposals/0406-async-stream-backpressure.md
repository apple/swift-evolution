# Backpressure support for AsyncStream

* Proposal: [SE-0406](0406-async-stream-backpressure.md)
* Author: [Franz Busch](https://github.com/FranzBusch)
* Review Manager: [Xiaodi Wu](https://github.com/xwu)
* Status: **Returned for revision**
* Implementation: [apple/swift#66488](https://github.com/apple/swift/pull/66488)
* Review: ([pitch](https://forums.swift.org/t/pitch-new-apis-for-async-throwing-stream-with-backpressure-support/65449)) ([review](https://forums.swift.org/t/se-0406-backpressure-support-for-asyncstream/66771)) ([return for revision](https://forums.swift.org/t/returned-for-revision-se-0406-backpressure-support-for-asyncstream/67248))

## Introduction

[SE-0314](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0314-async-stream.md)
introduced new `Async[Throwing]Stream` types which act as root asynchronous
sequences. These two types allow bridging from synchronous callbacks such as
delegates to an asynchronous sequence. This proposal adds a new way of
constructing asynchronous streams with the goal to bridge backpressured systems
into an asynchronous sequence. Furthermore, this proposal aims to clarify the
cancellation behaviour both when the consuming task is cancelled and when
the production side indicates termination.

## Motivation

After using the `AsyncSequence` protocol and the `Async[Throwing]Stream` types
extensively over the past years, we learned that there are a few important
behavioral details that any `AsyncSequence` implementation needs to support.
These behaviors are:

1. Backpressure
2. Multi/single consumer support
3. Downstream consumer termination
4. Upstream producer termination

In general, `AsyncSequence` implementations can be divided into two kinds: Root
asynchronous sequences that are the source of values such as
`Async[Throwing]Stream` and transformational asynchronous sequences such as
`AsyncMapSequence`. Most transformational asynchronous sequences implicitly
fulfill the above behaviors since they forward any demand to a base asynchronous
sequence that should implement the behaviors. On the other hand, root
asynchronous sequences need to make sure that all of the above behaviors are
correctly implemented. Let's look at the current behavior of
`Async[Throwing]Stream` to see if and how it achieves these behaviors.

### Backpressure

Root asynchronous sequences need to relay the backpressure to the producing
system. `Async[Throwing]Stream` aims to support backpressure by providing a
configurable buffer and returning
`Async[Throwing]Stream.Continuation.YieldResult` which contains the current
buffer depth from the `yield()` method. However, only providing the current
buffer depth on `yield()` is not enough to bridge a backpressured system into
an asynchronous sequence since this can only be used as a "stop" signal but we
are missing a signal to indicate resuming the production. The only viable
backpressure strategy that can be implemented with the current API is a timed
backoff where we stop producing for some period of time and then speculatively
produce again. This is a very inefficient pattern that produces high latencies
and inefficient use of resources.

### Multi/single consumer support

The `AsyncSequence` protocol itself makes no assumptions about whether the
implementation supports multiple consumers or not. This allows the creation of
unicast and multicast asynchronous sequences. The difference between a unicast
and multicast asynchronous sequence is if they allow multiple iterators to be
created. `AsyncStream` does support the creation of multiple iterators and it
does handle multiple consumers correctly. On the other hand,
`AsyncThrowingStream` also supports multiple iterators but does `fatalError`
when more than one iterator has to suspend. The original proposal states:

> As with any sequence, iterating over an AsyncStream multiple times, or
creating multiple iterators and iterating over them separately, may produce an
unexpected series of values.

While that statement leaves room for any behavior we learned that a clear distinction
of behavior for root asynchronous sequences is beneficial; especially, when it comes to
how transformation algorithms are applied on top.

### Downstream consumer termination

Downstream consumer termination allows the producer to notify the consumer that
no more values are going to be produced. `Async[Throwing]Stream` does support
this by calling the `finish()` or `finish(throwing:)` methods of the
`Async[Throwing]Stream.Continuation`. However, `Async[Throwing]Stream` does not
handle the case that the `Continuation` may be `deinit`ed before one of the
finish methods is called. This currently leads to async streams that never
terminate. The behavior could be changed but it could result in semantically
breaking code.

### Upstream producer termination

Upstream producer termination is the inverse of downstream consumer termination
where the producer is notified once the consumption has terminated. Currently,
`Async[Throwing]Stream` does expose the `onTermination` property on the
`Continuation`. The `onTermination` closure is invoked once the consumer has
terminated. The consumer can terminate in four separate cases:

1. The asynchronous sequence was `deinit`ed and no iterator was created
2. The iterator was `deinit`ed and the asynchronous sequence is unicast
3. The consuming task is canceled
4. The asynchronous sequence returned `nil` or threw

`Async[Throwing]Stream` currently invokes `onTermination` in all cases; however,
since `Async[Throwing]Stream` supports multiple consumers (as discussed in the
`Multi/single consumer support` section), a single consumer task being canceled
leads to the termination of all consumers. This is not expected from multicast
asynchronous sequences in general.

## Proposed solution

The above motivation lays out the expected behaviors from a root asynchronous
sequence and compares them to the behaviors of `Async[Throwing]Stream`. These
are the behaviors where `Async[Throwing]Stream` diverges from the expectations.

- Backpressure: Doesn't expose a "resumption" signal to the producer
- Multi/single consumer:
  - Divergent implementation between throwing and non-throwing variant
  - Supports multiple consumers even though proposal positions it as a unicast
  asynchronous sequence
- Consumer termination: Doesn't handle the `Continuation` being `deinit`ed
- Producer termination: Happens on first consumer termination 

This section proposes new APIs for `Async[Throwing]Stream` that implement all of
the above-mentioned behaviors.

### Creating an AsyncStream with backpressure support

You can create an `Async[Throwing]Stream` instance using the new `makeStream(of:
backpressureStrategy:)` method. This method returns you the stream and the
source. The source can be used to write new values to the asynchronous stream.
The new API specifically provides a multi-producer/single-consumer pattern.

```swift
let (stream, source) = AsyncStream.makeStream(
    of: Int.self,
    backpressureStrategy: .watermark(low: 2, high: 4)
)
```

The new proposed APIs offer three different ways to bridge a backpressured
system. The foundation is the multi-step synchronous interface. Below is an
example of how it can be used:

```swift
do {
    let writeResult = try source.write(contentsOf: sequence)
    
    switch writeResult {
    case .produceMore:
       // Trigger more production
    
    case .enqueueCallback(let callbackToken):
        source.enqueueCallback(token: callbackToken, onProduceMore: { result in
            switch result {
            case .success:
                // Trigger more production
            case .failure(let error):
                // Terminate the underlying producer
            }
        })
    }
} catch {
    // `write(contentsOf:)` throws if the asynchronous stream already terminated
}
```

The above API offers the most control and highest performance when bridging a
synchronous producer to an asynchronous sequence. First, you have to write
values using the `write(contentsOf:)` which returns a `WriteResult`. The result
either indicates that more values should be produced or that a callback should
be enqueued by calling the `enqueueCallback(callbackToken: onProduceMore:)`
method. This callback is invoked once the backpressure strategy decided that
more values should be produced. This API aims to offer the most flexibility with
the greatest performance. The callback only has to be allocated in the case
where the producer needs to be suspended.

Additionally, the above API is the building block for some higher-level and
easier-to-use APIs to write values to the asynchronous stream. Below is an
example of the two higher-level APIs.

```swift
// Writing new values and providing a callback when to produce more
try source.write(contentsOf: sequence, onProduceMore: { result in
    switch result {
    case .success:
        // Trigger more production
    case .failure(let error):
        // Terminate the underlying producer
    }
})

// This method suspends until more values should be produced
try await source.write(contentsOf: sequence)
```

With the above APIs, we should be able to effectively bridge any system into an
asynchronous stream regardless if the system is callback-based, blocking or
asynchronous.

### Downstream consumer termination

> When reading the next two examples around termination behaviour keep in mind
that the newly proposed APIs are providing a strict unicast asynchronous sequence.

Calling `finish()` terminates the downstream consumer. Below is an example of
this:

```swift
// Termination through calling finish
let (stream, source) = AsyncStream.makeStream(
    of: Int.self,
    backpressureStrategy: .watermark(low: 2, high: 4)
)

_ = try await source.write(1)
source.finish()

for await element in stream {
    print(element)
}
print("Finished")

// Prints
// 1
// Finished
```

The other way to terminate the consumer is by deiniting the source. This has the
same effect as calling `finish()` and makes sure that no consumer is stuck
indefinitely. 

```swift
// Termination through deiniting the source
let (stream, _) = AsyncStream.makeStream(
    of: Int.self,
    backpressureStrategy: .watermark(low: 2, high: 4)
)

for await element in stream {
    print(element)
}
print("Finished")

// Prints
// Finished
```

Trying to write more elements after the source has been finish will result in an
error thrown from the write methods.

### Upstream producer termination

The producer will get notified about termination through the `onTerminate`
callback. Termination of the producer happens in the following scenarios:

```swift
// Termination through task cancellation
let (stream, source) = AsyncStream.makeStream(
    of: Int.self,
    backpressureStrategy: .watermark(low: 2, high: 4)
)

let task = Task {
    for await element in stream {

    }
}
task.cancel()
```

```swift
// Termination through deiniting the sequence
let (_, source) = AsyncStream.makeStream(
    of: Int.self,
    backpressureStrategy: .watermark(low: 2, high: 4)
)
```

```swift
// Termination through deiniting the iterator
let (stream, source) = AsyncStream.makeStream(
    of: Int.self,
    backpressureStrategy: .watermark(low: 2, high: 4)
)
_ = stream.makeAsyncIterator()
```

```swift
// Termination through calling finish
let (stream, source) = AsyncStream.makeStream(
    of: Int.self,
    backpressureStrategy: .watermark(low: 2, high: 4)
)

_ = try source.write(1)
source.finish()

for await element in stream {}

// onTerminate will be called after all elements have been consumed
```

Similar to the downstream consumer termination, trying to write more elements after the
producer has been terminated will result in an error thrown from the write methods. 

## Detailed design

All new APIs on `AsyncStream` and `AsyncThrowingStream` are as follows:

```swift
/// Error that is thrown from the various `write` methods of the
/// ``AsyncStream.Source`` and ``AsyncThrowingStream.Source``.
/// 
/// This error is thrown when the asynchronous stream is already finished when
/// trying to write new elements.
public struct AsyncStreamAlreadyFinishedError: Error {}

extension AsyncStream {
    /// A mechanism to interface between producer code and an asynchronous stream.
    ///
    /// Use this source to provide elements to the stream by calling one of the `write` methods, then terminate the stream normally
    /// by calling the `finish()` method.
    public struct Source: Sendable {
        /// A strategy that handles the backpressure of the asynchronous stream.
        public struct BackpressureStrategy: Sendable {
            /// When the high watermark is reached producers will be suspended. All producers will be resumed again once
            /// the low watermark is reached.
            public static func watermark(low: Int, high: Int) -> BackpressureStrategy {}
        }

        /// A type that indicates the result of writing elements to the source.
        @frozen
        public enum WriteResult: Sendable {
            /// A token that is returned when the asynchronous stream's backpressure strategy indicated that production should
            /// be suspended. Use this token to enqueue a callback by  calling the ``enqueueCallback(_:)`` method.
            public struct CallbackToken: Sendable {}

            /// Indicates that more elements should be produced and written to the source.
            case produceMore

            /// Indicates that a callback should be enqueued.
            ///
            /// The associated token should be passed to the ``enqueueCallback(_:)`` method.
            case enqueueCallback(CallbackToken)
        }

        /// A callback to invoke when the stream finished.
        ///
        /// The stream finishes and calls this closure in the following cases:
        /// - No iterator was created and the sequence was deinited
        /// - An iterator was created and deinited
        /// - After ``finish(throwing:)`` was called and all elements have been consumed
        /// - The consuming task got cancelled
        public var onTermination: (@Sendable () -> Void)?

        /// Writes new elements to the asynchronous stream.
        ///
        /// If there is a task consuming the stream and awaiting the next element then the task will get resumed with the
        /// first element of the provided sequence. If the asynchronous stream already terminated then this method will throw an error
        /// indicating the failure.
        ///
        /// - Parameter sequence: The elements to write to the asynchronous stream.
        /// - Returns: The result that indicates if more elements should be produced at this time.
        public func write<S>(contentsOf sequence: S) throws -> WriteResult where Element == S.Element, S: Sequence {}

        /// Write the element to the asynchronous stream.
        ///
        /// If there is a task consuming the stream and awaiting the next element then the task will get resumed with the
        /// provided element. If the asynchronous stream already terminated then this method will throw an error
        /// indicating the failure.
        ///
        /// - Parameter element: The element to write to the asynchronous stream.
        /// - Returns: The result that indicates if more elements should be produced at this time.
        public func write(_ element: Element) throws -> WriteResult {}

        /// Enqueues a callback that will be invoked once more elements should be produced.
        ///
        /// Call this method after ``write(contentsOf:)`` or ``write(_:)`` returned ``WriteResult/enqueueCallback(_:)``.
        ///
        /// - Important: Enqueueing the same token multiple times is not allowed.
        ///
        /// - Parameters:
        ///   - token: The callback token.
        ///   - onProduceMore: The callback which gets invoked once more elements should be produced.
        public func enqueueCallback(token: WriteResult.CallbackToken, onProduceMore: @escaping @Sendable (Result<Void, Error>) -> Void) {}

        /// Cancel an enqueued callback.
        ///
        /// Call this method to cancel a callback enqueued by the ``enqueueCallback(callbackToken:onProduceMore:)`` method.
        ///
        /// - Note: This method supports being called before ``enqueueCallback(callbackToken:onProduceMore:)`` is called and
        /// will mark the passed `token` as cancelled.
        ///
        /// - Parameter token: The callback token.
        public func cancelCallback(token: WriteResult.CallbackToken) {}

        /// Write new elements to the asynchronous stream and provide a callback which will be invoked once more elements should be produced.
        ///
        /// If there is a task consuming the stream and awaiting the next element then the task will get resumed with the
        /// first element of the provided sequence. If the asynchronous stream already terminated then `onProduceMore` will be invoked with
        /// a `Result.failure`.
        ///
        /// - Parameters:
        ///   - sequence: The elements to write to the asynchronous stream.
        ///   - onProduceMore: The callback which gets invoked once more elements should be produced. This callback might be
        ///   invoked during the call to ``write(contentsOf:onProduceMore:)``.
        public func write<S>(contentsOf sequence: S, onProduceMore: @escaping @Sendable (Result<Void, Error>) -> Void) where Element == S.Element, S: Sequence {}

        /// Writes the element to the asynchronous stream.
        ///
        /// If there is a task consuming the stream and awaiting the next element then the task will get resumed with the
        /// provided element. If the asynchronous stream already terminated then `onProduceMore` will be invoked with
        /// a `Result.failure`.
        ///
        /// - Parameters:
        ///   - sequence: The element to write to the asynchronous stream.
        ///   - onProduceMore: The callback which gets invoked once more elements should be produced. This callback might be
        ///   invoked during the call to ``write(_:onProduceMore:)``.
        public func write(_ element: Element, onProduceMore: @escaping @Sendable (Result<Void, Error>) -> Void) {}

        /// Write new elements to the asynchronous stream.
        ///
        /// If there is a task consuming the stream and awaiting the next element then the task will get resumed with the
        /// first element of the provided sequence. If the asynchronous stream already terminated then this method will throw an error
        /// indicating the failure.
        ///
        /// This method returns once more elements should be produced.
        ///
        /// - Parameters:
        ///   - sequence: The elements to write to the asynchronous stream.
        public func write<S>(contentsOf sequence: S) async throws where Element == S.Element, S: Sequence {}

        /// Write new element to the asynchronous stream.
        ///
        /// If there is a task consuming the stream and awaiting the next element then the task will get resumed with the
        /// provided element. If the asynchronous stream already terminated then this method will throw an error
        /// indicating the failure.
        ///
        /// This method returns once more elements should be produced.
        ///
        /// - Parameters:
        ///   - sequence: The element to write to the asynchronous stream.
        public func write(_ element: Element) async throws {}

        /// Write the elements of the asynchronous sequence to the asynchronous stream.
        ///
        /// This method returns once the provided asynchronous sequence or the asynchronous stream finished.
        ///
        /// - Important: This method does not finish the source if consuming the upstream sequence terminated.
        ///
        /// - Parameters:
        ///   - sequence: The elements to write to the asynchronous stream.
        public func write<S>(contentsOf sequence: S) async throws where Element == S.Element, S: AsyncSequence {}

        /// Indicates that the production terminated.
        ///
        /// After all buffered elements are consumed the next iteration point will return `nil`.
        ///
        /// Calling this function more than once has no effect. After calling finish, the stream enters a terminal state and doesn't accept
        /// new elements.
        public func finish() {}
    }

    /// Initializes a new ``AsyncStream`` and an ``AsyncStream/Source``.
    ///
    /// - Parameters:
    ///   - elementType: The element type of the stream.
    ///   - backpressureStrategy: The backpressure strategy that the stream should use.
    /// - Returns: A tuple containing the stream and its source. The source should be passed to the
    ///   producer while the stream should be passed to the consumer.
    public static func makeStream(
        of elementType: Element.Type = Element.self,
        backpressureStrategy: Source.BackpressureStrategy
    ) -> (`Self`, Source) {}
}

extension AsyncThrowingStream {
    /// A mechanism to interface between producer code and an asynchronous stream.
    ///
    /// Use this source to provide elements to the stream by calling one of the `write` methods, then terminate the stream normally
    /// by calling the `finish()` method. You can also use the source's `finish(throwing:)` method to terminate the stream by
    /// throwing an error.
    public struct Source: Sendable {
        /// A strategy that handles the backpressure of the asynchronous stream.
        public struct BackpressureStrategy: Sendable {
            /// When the high watermark is reached, producers will be suspended. All producers will be resumed again once
            /// the low watermark is reached.
            public static func watermark(low: Int, high: Int) -> BackpressureStrategy {}
        }

        /// A type that indicates the result of writing elements to the source.
        @frozen
        public enum WriteResult: Sendable {
            /// A token that is returned when the asynchronous stream's backpressure strategy indicated that production should
            /// be suspended. Use this token to enqueue a callback by  calling the ``enqueueCallback(_:)`` method.
            public struct CallbackToken: Sendable {}

            /// Indicates that more elements should be produced and written to the source.
            case produceMore

            /// Indicates that a callback should be enqueued.
            ///
            /// The associated token should be passed to the ``enqueueCallback(_:)`` method.
            case enqueueCallback(CallbackToken)
        }

        /// A callback to invoke when the stream finished.
        ///
        /// The stream finishes and calls this closure in the following cases:
        /// - No iterator was created and the sequence was deinited
        /// - An iterator was created and deinited
        /// - After ``finish(throwing:)`` was called and all elements have been consumed
        /// - The consuming task got cancelled
        public var onTermination: (@Sendable () -> Void)? {}

        /// Writes new elements to the asynchronous stream.
        ///
        /// If there is a task consuming the stream and awaiting the next element then the task will get resumed with the
        /// first element of the provided sequence. If the asynchronous stream already terminated then this method will throw an error
        /// indicating the failure.
        ///
        /// - Parameter sequence: The elements to write to the asynchronous stream.
        /// - Returns: The result that indicates if more elements should be produced at this time.
        public func write<S>(contentsOf sequence: S) throws -> WriteResult where Element == S.Element, S: Sequence {}

        /// Write the element to the asynchronous stream.
        ///
        /// If there is a task consuming the stream and awaiting the next element then the task will get resumed with the
        /// provided element. If the asynchronous stream already terminated then this method will throw an error
        /// indicating the failure.
        ///
        /// - Parameter element: The element to write to the asynchronous stream.
        /// - Returns: The result that indicates if more elements should be produced at this time.
        public func write(_ element: Element) throws -> WriteResult {}

        /// Enqueues a callback that will be invoked once more elements should be produced.
        ///
        /// Call this method after ``write(contentsOf:)`` or ``write(_:)`` returned ``WriteResult/enqueueCallback(_:)``.
        ///
        /// - Important: Enqueueing the same token multiple times is not allowed.
        ///
        /// - Parameters:
        ///   - token: The callback token.
        ///   - onProduceMore: The callback which gets invoked once more elements should be produced.
        public func enqueueCallback(token: WriteResult.CallbackToken, onProduceMore: @escaping @Sendable (Result<Void, Error>) -> Void) {}

        /// Cancel an enqueued callback.
        ///
        /// Call this method to cancel a callback enqueued by the ``enqueueCallback(callbackToken:onProduceMore:)`` method.
        ///
        /// - Note: This method supports being called before ``enqueueCallback(callbackToken:onProduceMore:)`` is called and
        /// will mark the passed `token` as cancelled.
        ///
        /// - Parameter token: The callback token.
        public func cancelCallback(token: WriteResult.CallbackToken) {}

        /// Write new elements to the asynchronous stream and provide a callback which will be invoked once more elements should be produced.
        ///
        /// If there is a task consuming the stream and awaiting the next element then the task will get resumed with the
        /// first element of the provided sequence. If the asynchronous stream already terminated then `onProduceMore` will be invoked with
        /// a `Result.failure`.
        ///
        /// - Parameters:
        ///   - sequence: The elements to write to the asynchronous stream.
        ///   - onProduceMore: The callback which gets invoked once more elements should be produced. This callback might be
        ///   invoked during the call to ``write(contentsOf:onProduceMore:)``.
        public func write<S>(contentsOf sequence: S, onProduceMore: @escaping @Sendable (Result<Void, Error>) -> Void) where Element == S.Element, S: Sequence {}

        /// Writes the element to the asynchronous stream.
        ///
        /// If there is a task consuming the stream and awaiting the next element then the task will get resumed with the
        /// provided element. If the asynchronous stream already terminated then `onProduceMore` will be invoked with
        /// a `Result.failure`.
        ///
        /// - Parameters:
        ///   - sequence: The element to write to the asynchronous stream.
        ///   - onProduceMore: The callback which gets invoked once more elements should be produced. This callback might be
        ///   invoked during the call to ``write(_:onProduceMore:)``.
        public func write(_ element: Element, onProduceMore: @escaping @Sendable (Result<Void, Error>) -> Void) {}

        /// Write new elements to the asynchronous stream.
        ///
        /// If there is a task consuming the stream and awaiting the next element then the task will get resumed with the
        /// first element of the provided sequence. If the asynchronous stream already terminated then this method will throw an error
        /// indicating the failure.
        ///
        /// This method returns once more elements should be produced.
        ///
        /// - Parameters:
        ///   - sequence: The elements to write to the asynchronous stream.
        public func write<S>(contentsOf sequence: S) async throws where Element == S.Element, S: Sequence {}

        /// Write new element to the asynchronous stream.
        ///
        /// If there is a task consuming the stream and awaiting the next element then the task will get resumed with the
        /// provided element. If the asynchronous stream already terminated then this method will throw an error
        /// indicating the failure.
        ///
        /// This method returns once more elements should be produced.
        ///
        /// - Parameters:
        ///   - sequence: The element to write to the asynchronous stream.
        public func write(_ element: Element) async throws {}

        /// Write the elements of the asynchronous sequence to the asynchronous stream.
        ///
        /// This method returns once the provided asynchronous sequence or the  the asynchronous stream finished.
        ///
        /// - Important: This method does not finish the source if consuming the upstream sequence terminated.
        ///
        /// - Parameters:
        ///   - sequence: The elements to write to the asynchronous stream.
        public func write<S>(contentsOf sequence: S) async throws where Element == S.Element, S: AsyncSequence {}

        /// Indicates that the production terminated.
        ///
        /// After all buffered elements are consumed the next iteration point will return `nil` or throw an error.
        ///
        /// Calling this function more than once has no effect. After calling finish, the stream enters a terminal state and doesn't accept
        /// new elements.
        ///
        /// - Parameters:
        ///   - error: The error to throw, or `nil`, to finish normally.
        public func finish(throwing error: Failure?) {}
    }

    /// Initializes a new ``AsyncThrowingStream`` and an ``AsyncThrowingStream/Source``.
    ///
    /// - Parameters:
    ///   - elementType: The element type of the stream.
    ///   - failureType: The failure type of the stream.
    ///   - backpressureStrategy: The backpressure strategy that the stream should use.
    /// - Returns: A tuple containing the stream and its source. The source should be passed to the
    ///   producer while the stream should be passed to the consumer.
    public static func makeStream(
        of elementType: Element.Type = Element.self,
        throwing failureType: Failure.Type = Failure.self,
        backpressureStrategy: Source.BackpressureStrategy
    ) -> (`Self`, Source) where Failure == Error {}
}
```

## Comparison to other root asynchronous sequences

### swift-async-algorithm: AsyncChannel

The `AsyncChannel` is a multi-consumer/multi-producer root asynchronous sequence
which can be used to communicate between two tasks. It only offers asynchronous
production APIs and has no internal buffer. This means that any producer will be
suspended until its value has been consumed. `AsyncChannel` can handle multiple
consumers and resumes them in FIFO order.

### swift-nio: NIOAsyncSequenceProducer

The NIO team have created their own root asynchronous sequence with the goal to
provide a high performance sequence that can be used to bridge a NIO `Channel`
inbound stream into Concurrency. The `NIOAsyncSequenceProducer` is a highly
generic and fully inlinable type and quite unwieldy to use. This proposal is
heavily inspired by the learnings from this type but tries to create a more
flexible and easier to use API that fits into the standard library.

## Source compatibility

This change is additive and does not affect source compatibility.

## ABI compatibility

This change is additive and does not affect ABI compatibility. All new methods
are non-inlineable leaving us flexiblity to change the implementation in the
future.

## Future directions

### Adaptive backpressure strategy

The high/low watermark strategy is common in networking code; however, there are
other strategies such as an adaptive strategy that we could offer in the future.
An adaptive strategy regulates the backpressure based on the rate of
consumption and production. With the proposed new APIs we can easily add further
strategies.

### Element size dependent strategy

When the stream's element is a collection type then the proposed high/low
watermark backpressure strategy might lead to unexpected results since each
element can vary in actual memory size. In the future, we could provide a new
backpressure strategy that supports inspecting the size of the collection.

### Deprecate `Async[Throwing]Stream.Continuation`

In the future, we could deprecate the current continuation based APIs since the
new proposed APIs are also capable of bridging non-backpressured producers by
just discarding the `WriteResult`. The only use-case that the new APIs do not
cover is the _anycast_ behaviour of the current `AsyncStream` where one can
create multiple iterators to the stream as long as no two iterators are
consuming the stream at the same time. This can be solved via additional
algorithms such as `broadcast` in the `swift-async-algorithms` package.

To give developers more time to adopt the new APIs the deprecation of the
current APIs should be deferred to a future version. Especially since those new
APIs are not backdeployed like the current Concurrency runtime.

### Introduce a `Writer` and an `AsyncWriter` protocol

The newly introduced `Source` type offers a bunch of different write methods. We
have seen similar types used in other places such as file abstraction or
networking APIs. We could introduce a new `Writer` and `AsyncWriter` protocol in
the future to enable writing generic algorithms on top of writers. The `Source`
type could then conform to these new protocols.

## Alternatives considered

### Providing an `Async[Throwing]Stream.Continuation.onConsume`

We could add a new closure property to the `Async[Throwing]Stream.Continuation`
which is invoked once an element has been consumed to implement a backpressure
strategy; however, this requires the usage of a synchronization mechanism since
the consumption and production often happen on separate threads. The
added complexity and performance impact led to avoiding this approach.

### Provide a getter for the current buffer depth

We could provide a getter for the current buffer depth on the
`Async[Throwing]Stream.Continuation`. This could be used to query the buffer
depth at an arbitrary time; however, it wouldn't allow us to implement
backpressure strategies such as high/low watermarks without continuously asking
what the buffer depth is. That would result in a very inefficient
implementation. 

### Extending `Async[Throwing]Stream.Continuation`

Extending the current APIs to support all expected behaviors is problematic
since it would change the semantics and might lead to currently working code
misbehaving. Furthermore, extending the current APIs to support backpressure
turns out to be problematic without compromising performance or usability.

### Introducing a new type

We could introduce a new type such as `AsyncBackpressured[Throwing]Stream`;
however, one of the original intentions of `Async[Throwing]Stream` was to be
able to bridge backpressured systems. Furthermore, `Async[Throwing]Stream` is
the best name. Therefore, this proposal decided to provide new interfaces to
`Async[Throwing]Stream`.

### Stick with the current `Continuation` and `yield` naming

The proposal decided against sticking to the current names since the existing
names caused confusion to them being used in multiple places. Continuation was
both used by the `AsyncStream` but also by Swift Concurrency via
`CheckedContinuation` and `UnsafeContinuation`. Similarly, yield was used by
both `AsyncStream.Continuation.yield()`, `Task.yield()` and the `yield` keyword.
Having different names for these different concepts makes it easier to explain
their usage. The currently proposed `write` names were chosen to align with the
future direction of adding an `AsyncWriter` protocol. `Source` is a common name
in flow based systems such as Akka. Other names that were considered:

- `enqueue`
- `send`

### Provide the `onTermination` callback to the factory method

During development of the new APIs, I first tried to provide the `onTermination`
callback in the `makeStream` method. However, that showed significant usability
problems in scenarios where one wants to store the source in a type and
reference `self` in the `onTermination` closure at the same time; hence, I kept
the current pattern of setting the `onTermination` closure on the source.

### Provide a `onConsumerCancellation` callback

During the pitch phase, it was raised that we should provide a
`onConsumerCancellation` callback which gets invoked once the asynchronous
stream notices that the consuming task got cancelled. This callback could be
used to customize how cancellation is handled by the stream e.g. one could
imagine writing a few more elements to the stream before finishing it. Right now
the stream immediately returns `nil` or throws a `CancellationError` when it
notices cancellation. This proposal decided to not provide this customization
because it opens up the possibility that asynchronous streams are not terminating
when implemented incorrectly. Additionally, asynchronous sequences are not the
only place where task cancellation leads to an immediate error being thrown i.e.
`Task.sleep()` does the same. Hence, the value of the asynchronous not
terminating immediately brings little value when the next call in the iterating
task might throw. However, the implementation is flexible enough to add this in
the future and we can just default it to the current behaviour.

### Create a custom type for the `Result` of the `onProduceMore` callback

The `onProduceMore` callback takes a `Result<Void, Error>` which is used to
indicate if the producer should produce more or if the asynchronous stream
finished. We could introduce a new type for this but the proposal decided
against it since it effectively is a result type.

### Use an initializer instead of factory methods

Instead of providing a `makeStream` factory method we could use an initializer
approach that takes a closure which gets the `Source` passed into. A similar API
has been offered with the `Continuation` based approach and
[SE-0388](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0388-async-stream-factory.md)
introduced new factory methods to solve some of the usability ergonomics with
the initializer based APIs.

## Acknowledgements

- [Johannes Weiss](https://github.com/weissi) - For making me aware how
important this problem is and providing great ideas on how to shape the API.
- [Philippe Hausler](https://github.com/phausler) - For helping me designing the
APIs and continuously providing feedback
- [George Barnett](https://github.com/glbrntt) - For providing extensive code
reviews and testing the implementation.
