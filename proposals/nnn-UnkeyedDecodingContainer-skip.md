# Add skip() method for UnkeyedDecodingContainer to the standard library

* Proposal: [SE-NNNN](nnn-UnkeyedDecodingContainer-skip.md)
* Authors: [Igor Kulman](https://github.com/igorkulman)
* Review Manager: TBD
* Status: **Awaiting review**
* Implementation: [apple/swift#23707](https://github.com/apple/swift/pull/23707)

<!--

*During the review process, add the following fields as needed:*

* Implementation: [apple/swift#NNNNN](https://github.com/apple/swift/pull/NNNNN)
* Decision Notes: [Rationale](https://forums.swift.org/), [Additional Commentary](https://forums.swift.org/)
* Bugs: [SR-NNNN](https://bugs.swift.org/browse/SR-NNNN), [SR-MMMM](https://bugs.swift.org/browse/SR-MMMM)
* Previous Revision: [1](https://github.com/apple/swift-evolution/blob/...commit-ID.../proposals/NNNN-filename.md)
* Previous Proposal: [SE-XXXX](XXXX-filename.md)

-->

## Introduction

This proposal introduces a new `skip()` method for `UnkeyedDecodingContainer`. This design simplifies skipping over items when manually decoding arrays of heterogeneous types and eliminates the need for a workaround to do so by providing a method directly in the standard library.

Swift-evolution thread: [Pitch: UnkeyedDecodingContainer.moveNext() to skip items in deserialization](https://forums.swift.org/t/pitch-unkeyeddecodingcontainer-movenext-to-skip-items-in-deserialization/22151)

## Motivation

When decoding an array of heterogeneous types, for example from JSON, there is no obvious way to skip an item in this array without processing it

```swift
struct Feed: Decodable {
    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: FeedKeys.self)
        var messagesArrayForType = try container.nestedUnkeyedContainer(forKey: FeedKeys.messages)
        var messages = [Message]()

        var messagesArray = messagesArrayForType
        while(!messagesArrayForType.isAtEnd)
        {
            let message = try messagesArrayForType.nestedContainer(keyedBy: MessageTypeKey.self)
            let type = try message.decode(String.self, forKey: MessageTypeKey.type)
            switch type {
            case .avatar:
                messages.append(try messagesArray.decode(AvatarMessage.self))
            case .add:
                messages.append(try messagesArray.decode(AddMessage.self))
            case .remove:
                // skip, no longer needed in the app
                // how to move to the next item in the JSON array?
            }       
        }
        self.messages = messages
    }
}
```

At the moment, to skip over an item, you need to how the  `UnkeyedDecodingContainer` works internally for the `JSONDecoder`; that it has a `currentIndex: Int` and it moves to the next item by incrementing it only after successfully decoding a type.

With this knowledge you can introduce a workaround by decoding an empty `struct` to move to the next item

```swift
private struct EmptyStruct: Codable {}


struct Feed: Decodable {
    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: FeedKeys.self)
        var messagesArrayForType = try container.nestedUnkeyedContainer(forKey: FeedKeys.messages)
        var messages = [Message]()

        var messagesArray = messagesArrayForType
        while(!messagesArrayForType.isAtEnd)
        {
            let message = try messagesArrayForType.nestedContainer(keyedBy: MessageTypeKey.self)
            let type = try message.decode(String.self, forKey: MessageTypeKey.type)
            switch type {
            case .avatar:
                messages.append(try messagesArray.decode(AvatarMessage.self))
            case .add:
                messages.append(try messagesArray.decode(AddMessage.self))
            case .remove:
                _ = try? messagesArray.decode(EmptyStruct.self)
            }       
        }
        self.messages = messages
    }
}
```

## Proposed solution

The proposed solution, adding `skip()` method for `UnkeyedDecodingContainer`, would make the code simpler and much more obvious without the need for any workaround

```swift
struct Feed: Decodable {
    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: FeedKeys.self)
        var messagesArrayForType = try container.nestedUnkeyedContainer(forKey: FeedKeys.messages)
        var messages = [Message]()

        var messagesArray = messagesArrayForType
        while(!messagesArrayForType.isAtEnd)
        {
            let message = try messagesArrayForType.nestedContainer(keyedBy: MessageTypeKey.self)
            let type = try message.decode(String.self, forKey: MessageTypeKey.type)
            switch type {
            case .avatar:
                messages.append(try messagesArray.decode(AvatarMessage.self))
            case .add:
                messages.append(try messagesArray.decode(AddMessage.self))
            case .remove:
                try messagesArray.skip()
            }       
        }
        self.messages = messages
    }
}
```

This solution would also solve decoding arrays with failable items as described in [https://bugs.swift.org/browse/SR-5953](SR-5953).

## Detailed design

The core of the implementation is adding a new method to the `UnkeyedDecodingContainer` protocol

```swift
  /// Skips the current value
  ///
  /// - throws: `DecodingError.valueNotFound` if `self` is already at the end and
  /// there is no next value to skip to
  mutating  func skip() throws
```

A default implementation is also added

```swift
// Default implementation of skip() in terms of decoding an empty struct
struct Empty: Decodable { }

extension UnkeyedDecodingContainer {
  public mutating func skip() throws {
    _ = try decode(Empty.self)
  }
}
```

so the existing decoders do not need to implement anything. 

This should be a reasonable default implementation, working for `JSONDecoder`, `PropertyListDecoder` and possibly many custom decoders. Any decoder can override this default implementation to add a more efficient implementation, like just incrementing the `currentIndex` in `JSONDecoder`.

## Source compatibility

This change is purely additive.

## Effect on ABI stability

N/A

## Effect on API resilience

N/A

## Alternatives considered

At first I considered only adding the `skip()` method and its implementation in `JSONDecoder` but then, thanks to the comments in the forum, I realized there are other custom decoders out there so adding a default implementation is a better idea. 

I also considered adding `skip(by:)` instead to allow to skip by more than 1 item at a time but could not think about good use case when it would be needed, you can always call `skip()` multiple times. 