# Improved control over actor isolation

* Proposal: [SE-0313](0313-actor-isolation-control.md)
* Authors: [Doug Gregor](https://github.com/DougGregor), [Chris Lattner](https://github.com/lattner)
* Review Manager: [Ted Kremenek](https://github.com/tkremenek)
* Status: **Implemented (Swift 5.5)**
* Previous revision: [1](https://github.com/swiftlang/swift-evolution/blob/ca2e3b43be77b7f20303e1c5cba98f22ebb0fcb0/proposals/0313-actor-isolation-control.md)
* Implementation: Partially available in [recent `main` snapshots](https://swift.org/download/#snapshots) behind the flag `-Xfrontend -enable-experimental-concurrency`

## Table of Contents

* [Introduction](#introduction)
* [Motivation](#motivation)
* [Proposed design](#proposed-design)
   * [Actor-isolated parameters](#actor-isolated-parameters)
   * [Non-isolated declarations](#non-isolated-declarations)
   * [Protocol conformances](#protocol-conformances)
   * [Pre-async asynchronous protocols](#pre-async-asynchronous-protocols)
* [Source compatibility](#source-compatibility)
* [Effect on ABI stability](#effect-on-abi-stability)
* [Effect on API resilience](#effect-on-api-resilience)
* [Future Directions](#future-directions)
   * [Multiple isolated parameters](#multiple-isolated-parameters)
   * [Isolated protocol conformances](#isolated-protocol-conformances)
* [Alternatives Considered](#alternatives-considered)
   * [Isolated or sync actor types](#isolated-or-sync-actor-types)
* [Revision history](#revision-history)
  
## Introduction

The [Swift actors proposal][actors] introduces the notion of *actor-isolated* declarations, which are declarations that can safely access an actor's isolated state. In that proposal, all instance methods, instance properties, and instance subscripts on an actor type are actor-isolated, and they can synchronously use those declarations on `self`. This proposal generalizes the notion of actor isolation to allow better control, including the ability to have actor-isolated declarations that aren't part of an actor type (e.g., they can be non-member functions) and have non-isolated declarations that are instance members of an actor type (e.g., because they are based on immutable, non-isolated actor state). This allows better abstraction of the use of actors, additional actor operations that are otherwise not expressible safely in the system, and enables some conformances to existing, synchronous protocols.

## Motivation

The actors proposal uses a simple actor `BankAccount`, which has some immutable and some mutable state in it:

```swift
actor BankAccount {
  let accountNumber: Int
  var balance: Double

  init(accountNumber: Int, initialDeposit: Double) {
    self.accountNumber = accountNumber
    self.balance = initialDeposit
  }
  
  func deposit(amount: Double) {
    assert(amount >= 0)
    balance = balance + amount
  }
}
```

There are a few seemingly obvious things that one cannot do with this actor:

* We can't extract an operation like `deposit(amount:)` into a global function; it can only be written as a member of the actor.
* We can't write a computed property that provides a convenient display name for a bank account instance that's usable synchronously from outside the actor.
* We can't create a `Set<BankAccount>` because there is no way to make `BankAccount` conform to the `Hashable` protocol.

## Proposed design

All of the limitations described above stem from the fact that instance methods (and properties, and subscripts) on an actor type are *always* actor-isolated; no other functions can be actor-isolated and there is no way to make an instance method (etc.) not be isolated. This proposal generalizes the notion of actor-isolated functions such that any function can choose to be actor-isolated by indicating which of its actor parameters is isolated, as well as making an instance declaration on an actor not be actor-isolated at all.

### Actor-isolated parameters

A function can become actor-isolated by indicating that one of its parameters is `isolated`. For example, the `deposit(amount:)` operation can now be expressed as a module-scope function as follows:

```swift
func deposit(amount: Double, to account: isolated BankAccount) {
  assert(amount >= 0)
  account.balance = account.balance + amount
}
```

Because the `account` parameter is isolated, `deposit(amount:to:)` is actor-isolated (to its `account` parameter) and can access actor-isolated state directly on that parameter. The same actor-isolation rules apply:

```swift
extension BankAccount {
  func giveSomeGetSome(amount: Double, friend: BankAccount) async {
    deposit(amount: amount, to: self)         // okay to call synchronously, because self is isolated
    await deposit(amount: amount, to: friend) // must call asynchronously, because friend is not isolated
  }
}
```

This makes instance methods on actor types less special, because now they are expressible in terms of a general feature: they are methods for which the `self` parameter is `isolated`, which one can see when referencing the method's curried type:

```swift
let fn = BankAccount.deposit(amount:)   // type of fn is (isolated BankAccount) -> (Double) -> Void
```

A given function cannot have multiple `isolated` parameters:

```swift
func f(a: isolated BankAccount, b: isolated BankAccount) {  // error: multiple isolated parameters in function `f(a:b:)`.
  // ...
}

extension BankAccount {
  func quickTransfer(amount: Double, to other: isolated BankAccount) {  // error: multiple isolated parameters in function 'quickTransfer(amount:to:)'
    // ...
  }
}
```

### Non-isolated declarations

Instance declarations on an actor type implicitly have an `isolated self`. However, one can disable this implicit behavior using the `nonisolated` keyword:

```swift
actor BankAccount {
  nonisolated let accountNumber: Int
  var balance: Double

  // ...
}

extension BankAccount {
  // Produce an account number string with all but the last digits replaced with "X", which
  // is safe to put on documents.
  nonisolated func safeAccountNumberDisplayString() -> String {
    let digits = String(accountNumber)   // okay, because accountNumber is also nonisolated
    return String(repeating: "X", count: digits.count - 4) + String(digits.suffix(4))
  }
}

let fn2 = BankAccount.safeAccountNumberDisplayString   // type of fn is (BankAccount) -> () -> String
```

Note that, because `self` is not actor-isolated, `safeAccountNumberDisplayString` can only refer to non-isolated data on the actor. An attempt to refer to any actor-isolated declaration will produce an error or require asynchronous access, as appropriate:

```swift
extension BankAccount {
  nonisolated func steal(amount: Double) {
    balance -= amount  // error: actor-isolated property 'balance' can not be referenced on non-isolated parameter 'self'
  }
}  
```

The types involved in a non-isolated declaration must all be `Sendable`, because a non-isolated declaration can be used from any actor or concurrently-executing code. For example, one could not return a non-`Sendable` class from a `nonisolated` function:

```swift
class SomeClass { } // not Sendable

extension BankAccount {
  nonisolated func f() -> SomeClass? { nil } // error: `nonisolated` declaration returns non-Sendable type `SomeClass?`
}
```

### Protocol conformances

The actors proposal describes the rule that an actor-isolated function cannot satisfy a protocol requirement that is neither actor-isolated nor asynchronous, because doing so would allow synchronous access to actor state. However, non-isolated functions don't have access to actor state, so they are free to satisfy synchronous protocol requirements of any kind. For example, we can make `BankAccount` conform to `Hashable` by basing the hashing on the account number:

```swift
extension BankAccount: Hashable {
  nonisolated func hash(into hasher: inout Hasher) {
    hasher.combine(accountNumber) 
  }  
}

let fn = BankAccount.hash(into:) // type is (BankAccount) -> (inout Hasher) -> Void
```

Similarly, one can use a `nonisolated` computed property to conform to, e.g. `CustomStringConvertible`:

```swift
extension BankAccount: CustomStringConvertible {
  nonisolated var description: String {
    "Bank account #\(safeAccountNumberDisplayString())"
  }
}
```

### Pre-`async` asynchronous protocols

Non-isolated declarations are particularly useful for adapting existing asynchronous protocols, expressed using completion handlers, to actors. For example, consider an existing simple "server" protocol that uses a completion handler:

```swift
protocol OldServer {
  func send<Message: MessageType>(
    message: Message,
    completionHandler: (Result<Message.Reply>) -> Void
  )
}
```

Over time, this protocol should evolve to provide `async` requirements. However, one can make an actor type conform to this protocol using a non-isolated declaration that launches a detached task:

```swift
actor MyActorServer {
  func send<Message: MessageType>(message: Message) async throws -> Message.Reply { ... }  // this is the "real" asynchronous implementation we want
}

extension MyActorServer : OldServer {
  nonisolated func send<Message: MessageType>(
    message: Message,
    completionHandler: (Result<Message.Reply>) -> Void
  ) {
    detach {
      do {
        let reply = try await send(message: message)
        completionHandler(.success(reply))
      } catch {
        completionHandler(.failure(error))
      } 
    }
  }
}
```

This allows actors to more smoothly integrate into existing code bases, without having to first adopt `async` throughout.

## Source compatibility

This proposal is additive, extending the grammar in a space where new contextual keywords are commonly introduced (declaration modifiers), so it will not affect source compatibility.

## Effect on ABI stability

This is purely additive to the ABI. Function parameters can be marked `isolated`, which will be captured as part of the function type. However, this (like other modifiers on a function parameter) is an additive change that won't affect existing ABI.

## Effect on API resilience

Nearly all changes in actor isolation are breaking changes, because the actor isolation rules require consistency between a declaration. Therefore, a parameter cannot be changed between `isolated` and non-`isolated` (either directly, or indirectly via `nonisolated`) without breaking the API.

## Future Directions

### Multiple `isolated` parameters

This proposal prohibits a function declaration that has more than one `isolated` parameter. We could lift this restriction in the future, to allow code such as:

```swift
func f(a: isolated BankAccount, b: isolated BankAccount) { 
  // ...
}
```

However, there are very few ways to call such a function in [base actors proposal][actors], because one can only run on a single actor at a time. Therefore, the only way to safely call `f` is to pass the same actor twice:

```swift
extension BankAccount {
  func g() {
    f(a: self, b: self)
  }

  func h(other: BankAccount) async {
    await f(a: self, b: other) // error: isolated parameters `a` and `b` passed values with potentially-different actors
  }
}
```

There are unsafe mechanisms (e.g., unsafe casting of pointer types) that could be used to pass two different actors that are both isolated. The [custom executors proposal](https://forums.swift.org/t/support-custom-executors-in-swift-concurrency/44425) provides control over the concurrency domains in which actors execute, which could be used to dynamically ensure that two actors execute in the same concurrency domain. That proposal could be modified or extended to guarantee *statically* that some set of actors share a concurrency domain to make functions with more than one `isolated` parameter more useful in the future.

### Isolated protocol conformances

The conformance of an actor type to a protocol assumes that the client of the protocol is outside of the actor's isolation domain. Therefore, [protocol conformances](#protocol-conformance) require either the protocol to have `async` requirements or the actor to use non-isolated members to establish protocol conformance. The [Type System Considerations for Actor Protocol](https://forums.swift.org/t/exploration-type-system-considerations-for-actor-proposal/44540) pitch argues that actor types should be able to conform to protocols with the assumption that the conformance is only used within the actor's isolation context. That pitch provides the following example:

```swift
public protocol DataProcessible {
    var data: Data { get }
}
extension DataProcessible {
  func compressData() -> Data {
    use(data) 
    /// details omitted
  }
}

actor MyDataActor : DataProcessible {
  // error: cannot fulfill sync requirement with isolated actor member.
  var data: Data

  func doThing() {
    // All sync, no problem!
    let compressed = compressData()
  }
}
```

That pitch suggests that the conformance of `MyDataActor : DataProcessible` be permitted, and introduces the notion of a `@sync` actor type to describe the actor when in its own isolation domain. Specifically, the type `@sync MyDataActor` conforms to `DataProcessible` but the type `@async MyDataActor` (which represents the actor outside of its isolation domain) does not.

This proposal does not separate isolated from non-isolated actor types, and instead uses an `isolated` parameter to describe the actor that the code is executing on. The same notion can be extended to introduce isolated protocol conformances, which are conformances that can only be used with isolated values. For example, the conformance itself could have `isolated` applied to it to mark it as an isolated conformance:

```swift
actor MyDataActor : isolated DataProcessible {
  var data: Data   // okay: satisfies "data" requirement

  func doThing() {
    // okay, because self is isolated
    let compressed = compressData()
  }
  
  nonisolated failToDoTheThing() {
    // error: isolated conformance MyDataActor : DataProcessible cannot be used when non-isolated
    // value of type MyDataActor is passed to the generic function.
    let compressed = compressData()    
  }
}
```

The use of isolated protocol conformances would require a number of other restrictions to ensure that the protocol conformance cannot be used on non-isolated instances of the actor. For example, this means that a non-isolated conformance can never be used along with `Sendable` on the same type, because that would permit a non-isolated instance of the actor to be passed outside of the actor's isolation domain along with a protocol conformance that assumes it is within the actor's isolation domain.

## Alternatives Considered

### Isolated or sync actor types

The notion of "isolated" parameters grew out of a [proposal](https://forums.swift.org/t/exploration-type-system-considerations-for-actor-proposal/44540) that generalized the notion of actor isolation from something that only made sense on `self` to one that made sense for any parameter. That proposal modeled isolation directly in the type system by introducing a new kind of type: `@sync` actor types were used for values that have synchronous access to the actors they describe. Therefore, instead of saying that `self` is an `isolated` parameter of type `MyActor`, the proposal would say that `self` has the type `@sync MyActor`. The "isolated conformances" described in the future directions above are similar to (and directly influenced by) the notion that `@sync` actor types can conform to (synchronous) protocols as described in that proposal.

At a high level, isolated parameters and isolated conformances are similar to parameters of `@sync` type and conformances of `@sync` types to protocols, and can address similar sets of use cases. This proposal chose to treat `isolated` as a parameter modifier rather than as a type because it provides a simpler, value-centric model that aligns more closely with the behavior of a similarly-constrained construct, `inout`. There are several inconsistencies to the `@sync` type approach that made it less desirable:

* The type of an actor's `self` can change within nested contexts, such as closures, between `@sync` and non-`@sync`:

    ```swift
    func f<T>(_: T) { }
    
    actor MyActor {
      func g() {
        f(self) // T = @sync MyActor

        asyncDetached {
          f(self) // T = MyActor
        }
      }
    }  
    ```
  Generally speaking, a variable in Swift has the same type when it's captured in a nested context as it does in its enclosing context, which provides a level of predictability that would be lost with `@sync` types. In the example above, type inference for the call to `f` differs significantly whether you're in the closure or not. A [recent discussion on the forums](https://forums.swift.org/t/implicit-casts-for-verified-type-information/41035) about narrowing types showed resistence to the idea of changing the type of a variable in a nested context, even when doing so could eliminate additional boilerplate. 

* The design relies heavily on the implicit conversion from `@sync MyActor` to `MyActor`, e.g.,

    ```swift
    func acceptActor(_: MyActor) { }
    func acceptSendable<T: Sendable>(_: T) { }
    
    extension MyActor {
      func h() {
        acceptActor(h)  // okay, requires conversion of @sync MyActor to MyActor
        acceptSendable(h) // okay, requires T=MyActor and conversion of @sync MyActor to MyActor
      }
    }
  ```

* Conformance to `Sendable` doesn't follow the normal subtyping rules. Per the conversion above, a `@sync` actor type is a subtype of the corresponding (non-`@sync`) actor type. By definition, a subtype has all of the conformances of its supertype, and may of course add more capabilities. This is a general principle of type system design, and shows up in Swift in a number of places, e.g., with subclassing:

    ```swift
    protocol P { }
    
    class C: P { }
    class D: C { }
    
    func test(c: C, d: D) {
      let _: P = c   // okay, C conforms to P
      let _: P = d   // okay, D conforms to P because it is a subtype of C, which itself conforms to P
    }
    ```
    
    However, `@sync` types don't behave this way with respect to `Sendable`. A non-`@sync` actor type conforms to `Sendable` (it's safe to share it across concurrency domains), but its corresponding `@sync` subtype does *not* conform to `Sendable`. This is why in the prior example's call to `acceptSendable`, the implicit conversion from `@sync MyActor` to `MyActor` is required.

## Revision history

* Changes in the accepted version of this proposal:
  * Removed `isolated` captures.
  * Prohibit multiple `isolated` parameters.

[actors]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0306-actors.md
