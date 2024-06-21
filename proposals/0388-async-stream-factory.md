# Convenience Async[Throwing]Stream.makeStream methods

* Proposal: [SE-0388](0388-async-stream-factory.md)
* Authors: [Franz Busch](https://github.com/FranzBusch)
* Review Manager: [Becca Royal-Gordon](https://github.com/beccadax)
* Status: **Implemented (Swift 5.9)**
* Implementation: [apple/swift#62968](https://github.com/apple/swift/pull/62968)
* Review: ([pitch](https://forums.swift.org/t/pitch-convenience-async-throwing-stream-makestream-methods/61030)) ([review](https://forums.swift.org/t/se-0388-convenience-async-throwing-stream-makestream-methods/63139)) ([acceptance](https://forums.swift.org/t/accepted-with-modifications-se-0388-convenience-async-throwing-stream-makestream-methods/63568)) 

## Introduction

We propose introducing helper methods for creating `AsyncStream` and `AsyncThrowingStream`
instances which make the stream's continuation easier to access.

## Motivation

With [SE-0314](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0314-async-stream.md)
we introduced `AsyncStream` and `AsyncThrowingStream` which act as a root
`AsyncSequence` that the standard library offers.

After having used `Async[Throwing]Stream` for some time, a common usage
is to pass the continuation and the `Async[Throwing]Stream` to different places.
This requires escaping the `Async[Throwing]Stream.Continuation` out of 
the closure that is passed to the initialiser.
Escaping the continuation is slightly inconvenient since it requires a dance
around an implicitly unwrapped optional. Furthermore, the closure implies
that the continuation lifetime is scoped to the closure which it isn't. This is how
an example usage of the current `AsyncStream` API looks like.

```swift
var cont: AsyncStream<Int>.Continuation!
let stream = AsyncStream<Int> { cont = $0 }
// We have to assign the continuation to a let to avoid sendability warnings
let continuation = cont

await withTaskGroup(of: Void.self) { group in
  group.addTask {
    for i in 0...9 {
      continuation.yield(i)
    }
    continuation.finish()
  }

  group.addTask {
    for await i in stream {
      print(i)
    }
  }
}
```

## Proposed solution

In order to fill this gap, I propose to add a new static method `makeStream` on
`AsyncStream` and `AsyncThrowingStream` that returns both the stream
and the continuation. An example of using the new proposed convenience methods looks like this:

```swift
let (stream, continuation) = AsyncStream.makeStream(of: Int.self)

await withTaskGroup(of: Void.self) { group in
  group.addTask {
    for i in 0...9 {
      continuation.yield(i)
    }
    continuation.finish()
  }

  group.addTask {
    for await i in stream {
      print(i)
    }
  }
}
```

## Detailed design

I propose to add the following code to `AsyncStream` and `AsyncThrowingStream`
respectively. These methods are also marked as backdeployed to previous Swift versions.

```swift
@available(SwiftStdlib 5.1, *)
extension AsyncStream {
  /// Initializes a new ``AsyncStream`` and an ``AsyncStream/Continuation``.
  ///
  /// - Parameters:
  ///   - elementType: The element type of the stream.
  ///   - limit: The buffering policy that the stream should use.
  /// - Returns: A tuple containing the stream and its continuation. The continuation should be passed to the
  /// producer while the stream should be passed to the consumer.
  @backDeployed(before: SwiftStdlib 5.9)
  public static func makeStream(
      of elementType: Element.Type = Element.self,
      bufferingPolicy limit: Continuation.BufferingPolicy = .unbounded
  ) -> (stream: AsyncStream<Element>, continuation: AsyncStream<Element>.Continuation) {
    var continuation: AsyncStream<Element>.Continuation!
    let stream = AsyncStream<Element>(bufferingPolicy: limit) { continuation = $0 }
    return (stream: stream, continuation: continuation!)
  }
}

@available(SwiftStdlib 5.1, *)
extension AsyncThrowingStream {
  /// Initializes a new ``AsyncThrowingStream`` and an ``AsyncThrowingStream/Continuation``.
  ///
  /// - Parameters:
  ///   - elementType: The element type of the stream.
  ///   - failureType: The failure type of the stream.
  ///   - limit: The buffering policy that the stream should use.
  /// - Returns: A tuple containing the stream and its continuation. The continuation should be passed to the
  /// producer while the stream should be passed to the consumer.
  @backDeployed(before: SwiftStdlib 5.9)
  public static func makeStream(
      of elementType: Element.Type = Element.self,
      throwing failureType: Failure.Type = Failure.self,
      bufferingPolicy limit: Continuation.BufferingPolicy = .unbounded
  ) -> (stream: AsyncThrowingStream<Element, Failure>, continuation: AsyncThrowingStream<Element, Failure>.Continuation) where Failure == Error {
    var continuation: AsyncThrowingStream<Element, Failure>.Continuation!
    let stream = AsyncThrowingStream<Element, Failure>(bufferingPolicy: limit) { continuation = $0 }
    return (stream: stream, continuation: continuation!)
  }
}
```

## Source compatibility
This change is additive and does not affect source compatibility.

## Effect on ABI stability
This change introduces new concurrency library ABI in the form of the `makeStream` methods, but it does not affect the ABI of existing declarations.

## Effect on API resilience
None; adding static methods is permitted by the existing resilience model.

## Alternatives considered

### Return a concrete type instead of a tuple
My initial pitch was using a tuple as the result type of the factory;
however, I walked back on it before the review since I think we can provide better documentation on
the concrete type. However during the review the majority of the feedback was leaning towards the tuple based approach.
After comparing the two approaches, I agree with the review feedback. The tuple based approach has two major benefits:

1. It nudges the user to destructure the returned typed which we want since the continuation and stream should be retained by the
producer and consumer respectively.
2. It allows us to back deploy the method.

### Pass a continuation to the `AsyncStream<Element>.init()`
During the pitch it was brought up that we could let users pass a continuation to the
`AsyncStream<Element>.init()`; however, this opens up a few problems:
1. A continuation could be passed to multiple streams
2. A continuation which is not passed to a stream is useless

In the end, the `AsyncStream.Continuation` is deeply coupled to one instance of an
`AsyncStream` hence we should create an API that conveys this coupling and prevents
users from misuse. 

### Do nothing alternative
We could just leave the current creation of `Async[Throwing]Stream` as is;
however, since it is part of the standard library we should provide
a better method to create a stream and its continuation.

## Revision history

- After review: Changed the return type from a concrete type to a tuple

