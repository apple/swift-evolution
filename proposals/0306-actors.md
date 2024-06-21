# Actors

* Proposal: [SE-0306](0306-actors.md)
* Authors: [John McCall](https://github.com/rjmccall), [Doug Gregor](https://github.com/DougGregor), [Konrad Malawski](https://github.com/ktoso), [Chris Lattner](https://github.com/lattner)
* Review Manager: [Joe Groff](https://github.com/jckarter)
* Status: **Implemented (Swift 5.5)**
* Implementation: Partially available in [recent `main` snapshots](https://swift.org/download/#snapshots) behind the flag `-Xfrontend -enable-experimental-concurrency`
* Review: ([first review](https://forums.swift.org/t/se-0306-actors/45734)), ([second review](https://forums.swift.org/t/se-0306-second-review-actors/47291)), ([acceptance](https://forums.swift.org/t/accepted-with-modification-se-0306-actors/47662))

## Table of Contents

* [Introduction](#introduction)
* [Proposed solution](#proposed-solution)
   * [Actors](#actors-1)
   * [Actor isolation](#actor-isolation)
   * [Cross-actor references and Sendable types](#cross-actor-references-and-sendable-types)
   * [Closures](#closures)
   * [Actor reentrancy](#actor-reentrancy)
      * ["Interleaving" execution with reentrant actors](#interleaving-execution-with-reentrant-actors)
      * [Deadlocks with non-reentrant actors](#deadlocks-with-non-reentrant-actors)
      * [Unnecessary blocking with non-reentrant actors](#unnecessary-blocking-with-non-reentrant-actors)
      * [Existing practice](#existing-practice)
      * [Reentrancy Summary](#reentrancy-summary)
   * [Protocol conformances](#protocol-conformances)
* [Detailed design](#detailed-design)
   * [Actors](#actors-2)
   * [Actor isolation checking](#actor-isolation-checking)
      * [References and actor isolation](#references-and-actor-isolation)
      * [Protocol conformance](#protocol-conformance)
   * [Partial applications](#partial-applications)
   * [Key paths](#key-paths)
   * [inout parameters](#inout-parameters)
   * [Actor interoperability with Objective-C](#actor-interoperability-with-objective-c)
* [Source compatibility](#source-compatibility)
* [Effect on ABI stability](#effect-on-abi-stability)
* [Effect on API resilience](#effect-on-api-resilience)
* [Future Directions](#future-directions)
   * [Non-reentrancy](#non-reentrancy)
   * [Task-chain reentrancy](#task-chain-reentrancy)
* [Alternatives considered](#alternatives-considered) 
   * [Actor inheritance](#actor-inheritance)
   * [Cross-actor lets](#cross-actor-lets)
* [Revision history](#revision-history)

## Introduction

The Swift concurrency model intends to provide a safe programming model that statically detects [data races](https://en.wikipedia.org/wiki/Race_condition#Data_race) and other common concurrency bugs. The [Structured Concurrency][sc] proposal introduces a way to define concurrent tasks and provides data-race safety for functions and closures. This model is suitable for a number of common design patterns, including things like parallel maps and concurrent callback patterns, but is limited to working with state that is captured by closures.

Swift includes classes, which provide a mechanism for declaring mutable state that is shared across the program. Classes, however, are notoriously difficult to correctly use within concurrent programs, requiring error-prone manual synchronization to avoid data races. We want to provide the ability to use shared mutable state while still providing static detection of data races and other common concurrency bugs.

The [actor model](https://en.wikipedia.org/wiki/Actor_model) defines entities called *actors* that are perfect for this task. Actors allow you as a programmer to declare that a bag of state is held within a concurrency domain and then define multiple operations that act upon it. Each actor protects its own data through *data isolation*, ensuring that only a single thread will access that data at a given time, even when many clients are concurrently making requests of the actor. As part of the Swift Concurrency Model, actors provide the same race and memory safety properties as structured concurrency, but provide the familiar abstraction and reuse features that other explicitly declared types in Swift enjoy.

Swift-evolution threads:
  [Pitch #1](https://forums.swift.org/t/concurrency-actors-actor-isolation/41613),
  [Pitch #2](https://forums.swift.org/t/pitch-2-actors/44094),
  [Pitch #3](https://forums.swift.org/t/pitch-3-actors/44470),
  [Pitch #4](https://forums.swift.org/t/pitch-4-actors/45215),
  [Pitch #5](https://forums.swift.org/t/pitch-4-actors/45215/36),
  [Pitch #6](https://forums.swift.org/t/pitch-6-actors/45519),
  [Review #1](https://forums.swift.org/t/se-0306-actors/45734)

## Proposed solution

### Actors

This proposal introduces *actors* into Swift. An actor is a reference type that protects access to its mutable state, and is introduced with the keyword `actor`:

```swift
actor BankAccount {
  let accountNumber: Int
  var balance: Double

  init(accountNumber: Int, initialDeposit: Double) {
    self.accountNumber = accountNumber
    self.balance = initialDeposit
  }
}
```

Like other Swift types, actors can have initializers, methods, properties, and subscripts. They can be extended and conform to protocols, be generic, and be used with generics.

The primary difference is that actors protect their state from data races. This is enforced statically by the Swift compiler through a set of limitations on the way in which actors and their instance members can be used, collectively called *actor isolation*.   

### Actor isolation

Actor isolation is how actors protect their mutable state. For actors, the primary mechanism for this protection is by only allowing their stored instance properties to be accessed directly on `self`. For example, here is a method that attempts to transfer money from one account to another:

```swift
extension BankAccount {
  enum BankError: Error {
    case insufficientFunds
  }
  
  func transfer(amount: Double, to other: BankAccount) throws {
    if amount > balance {
      throw BankError.insufficientFunds
    }

    print("Transferring \(amount) from \(accountNumber) to \(other.accountNumber)")

    balance = balance - amount
    other.balance = other.balance + amount  // error: actor-isolated property 'balance' can only be referenced on 'self'
  }
}
```

If `BankAccount` were a class, the `transfer(amount:to:)` method would be well-formed, but would be subject to data races in concurrent code without an external locking mechanism. 

With actors, the attempt to reference `other.balance` triggers a compiler error, because `balance` may only be referenced on `self`. The error message notes that `balance` is *actor-isolated*, meaning that it can only be accessed directly from within the specific actor it is tied to or "isolated by". In this case, it's the instance of `BankAccount` referenced by `self`. All declarations on an instance of an actor, including stored and computed instance properties (like `balance`), instance methods (like `transfer(amount:to:)`), and instance subscripts, are all actor-isolated by default. Actor-isolated declarations can freely refer to other actor-isolated declarations on the same actor instance (on `self`). Any declaration that is not actor-isolated is *non-isolated* and cannot synchronously access any actor-isolated declaration.

A reference to an actor-isolated declaration from outside that actor is called a *cross-actor reference*. Such references are permissible in one of two ways. First, a cross-actor reference to immutable state is allowed from anywhere in the same module as the actor is defined because, once initialized, that state can never be modified (either from inside the actor or outside it), so there are no data races by definition. The reference to `other.accountNumber` is allowed based on this rule, because `accountNumber` is declared via a `let` and has value-semantic type `Int`.

The second form of permissible cross-actor reference is one that is performed with an asynchronous function invocation. Such asynchronous function invocations are turned into "messages" requesting that the actor execute the corresponding task when it can safely do so. These messages are stored in the actor's "mailbox", and the caller initiating the asynchronous function invocation may be suspended until the actor is able to process the corresponding message in its mailbox. An actor processes the messages in its mailbox one-at-a-time, so that a given actor will never have two concurrently-executing tasks running actor-isolated code. This ensures that there are no data races on actor-isolated mutable state, because there is no concurrency in any code that can access actor-isolated state. For example, if we wanted to make a deposit to a given bank account `account`, we could make a call to a method `deposit(amount:)` on another actor, and that call would become a message placed in the actor's mailbox and the caller would suspend. When that actor processes messages, it will eventually process the message corresponding to the deposit, executing that call within the actor's isolation domain when no other code is executing in that actor's isolation domain.

> **Implementation note**: At an implementation level, the messages are partial tasks (described by the [Structured Concurrency][sc] proposal) for the asynchronous call, and each actor instance contains its own serial executor (also in the [Structured Concurrency][sc] proposal). The default serial executor is responsible for running the partial tasks one-at-a-time. This is conceptually similar to a serial [`DispatchQueue`](https://developer.apple.com/documentation/dispatch/dispatchqueue), but with an important difference: tasks awaiting an actor are **not** guaranteed to be run in the same order they originally awaited that actor. Swift's runtime system aims to avoid priority inversions whenever possible, using techniques like priority escalation. Thus, the runtime system considers a task's priority when selecting the next task to run on the actor from its queue. This is in contrast with a serial DispatchQueue, which are strictly first-in-first-out. In addition, Swift's actor runtime uses a lighter-weight queue implementation than Dispatch to take full advantage of Swift's `async` functions.

Compile-time actor-isolation checking determines which references to actor-isolated declarations are cross-actor references, and ensures that such references use one of the two permissible mechanisms described above. This ensures that code outside of the actor does not interfere with the actor's mutable state.

Based on the above, we can implement a correct version of `transfer(amount:to:)` that is asynchronous:

```swift
extension BankAccount {
  func transfer(amount: Double, to other: BankAccount) async throws {
    assert(amount > 0)

    if amount > balance {
      throw BankError.insufficientFunds
    }

    print("Transferring \(amount) from \(accountNumber) to \(other.accountNumber)")

    // Safe: this operation is the only one that has access to the actor's isolated
    // state right now, and there have not been any suspension points between
    // the place where we checked for sufficient funds and here.
    balance = balance - amount
    
    // Safe: the deposit operation is placed in the `other` actor's mailbox; when
    // that actor retrieves the operation from its mailbox to execute it, the
    // other account's balance will get updated.
    await other.deposit(amount: amount)
  }
}
```

The `deposit(amount:)` operation needs involve the state of a different actor, so it must be invoked asynchronously. This method could itself be implemented as `async`:

```swift
extension BankAccount {
  func deposit(amount: Double) async {
    assert(amount >= 0)
    balance = balance + amount
  }
}
```

However, this method doesn't really need to be `async`: it makes no asynchronous calls (note the lack of `await`). Therefore, it would be better defined as a synchronous function:

```swift
extension BankAccount {
  func deposit(amount: Double) {
    assert(amount >= 0)
    balance = balance + amount
  }
}
```

Synchronous actor functions can be called synchronously on the actor's `self`, but cross-actor references to this method require an asynchronous call. The `transfer(amount:to:)` function calls it asynchronously (on `other`), while the following function `passGo` calls it synchronously (on the implicit `self`):

```swift
extension BankAccount {
  // Pass go and collect $200
  func passGo() {
    self.deposit(amount: 200.0)  // synchronous is okay because `self` is isolated
  }
}
```

Cross-actor references to an actor property are permitted as an asynchronous call so long as they are read-only accesses:

```swift
func checkBalance(account: BankAccount) async {
  print(await account.balance)   // okay
  await account.balance = 1000.0 // error: cross-actor property mutations are not permitted
}
```

> **Rationale**: it is possible to support cross-actor property sets. However, cross-actor `inout` operations cannot be reasonably supported because there would be an implicit suspension point between the "get" and the "set" that could introduce what would effectively be race conditions. Moreover, setting properties asynchronously may make it easier to break invariants unintentionally if, e.g., two properties need to be updated at once to maintain an invariant.

From outside a module, immutable `let`s must be referenced asynchronously from outside the actor. For example:

```swift
// From another module
func printAccount(account: BankAccount) {
  print("Account #\(await account.accountNumber)")
}
```

This preserves the ability for the module that defines `BankAccount` to evolve the `let` into a `var` without breaking clients, which is a property Swift has always maintained.:

```swift
actor BankAccount { // version 2
  var accountNumber: Int
  var balance: Double  
}
```

Only code within the module will need to change to account for `accountNumber` becoming a `var`; existing clients will already use asynchronous access and be unaffected.

### Cross-actor references and `Sendable` types

[SE-0302][se302] introduces the `Sendable` protocol. Values of types that conform to the `Sendable` protocol are safe to share across concurrently-executing code. There are various kinds of types that work well this way: value-semantic types like `Int` and `String`, value-semantic collections of such types like `[String]` or `[Int: String]`, immutable classes, classes that perform their own synchronization internally (like a concurrent hash table), and so on.

Actors protect their mutable state, so actor instances can be freely shared across concurrently-executing code, and the actor itself will internally maintain synchronization. Therefore, every actor type implicitly conforms to the `Sendable` protocol.

All cross-actor references are, necessarily, working with values of types that are being shared across different concurrently-executed code. For example, let's say that our `BankAccount` includes a list of owners, where each owner is modeled by a `Person` class:

```swift
class Person {
  var name: String
  let birthDate: Date
}

actor BankAccount {
  // ...
  var owners: [Person]

  func primaryOwner() -> Person? { return owners.first }
}
```

The `primaryOwner` function can be called asynchronously from another actor, and then the `Person` instance can be modified from anywhere:

```swift
if let primary = await account.primaryOwner() {
  primary.name = "The Honorable " + primary.name  // problem: concurrent mutation of actor-isolated state
}
```

Even non-mutating access is problematic, because the person's `name` could be modified from within the actor at the same time as the original call is trying to access it. To prevent this potential for concurrent mutation of actor-isolated state, all cross-actor references can only involve types that conform to `Sendable`. For a cross-actor asynchronous call, the argument and result types must conform to `Sendable`. For a cross-actor reference to an immutable property, the property type must conform to `Sendable`. By insisting that all cross-actor references only use `Sendable` types, we can ensure that no references to shared mutable state flow into or out of the actor's isolation domain. The compiler will produce a diagnostic for such issues. For example, the call to `account.primaryOwner()` about would produce an error like the following:

```
error: cannot call function returning non-Sendable type 'Person?' across actors
```

Note that the `primaryOwner()` function as defined above can still be used with actor-isolated code. For example, we can define a function to get the name of the primary owner, like this:

```swift
extension BankAccount {
  func primaryOwnerName() -> String? {
    return primaryOwner()?.name
  }
}
```

The `primaryOwnerName()` function is safe to asynchronously call across actors because `String` (and therefore `String?`) conforms to `Sendable`.

### Closures

The restrictions on cross-actor references only work so long as we can ensure that the code that might execute concurrently with actor-isolated code is considered to be non-isolated. For example, consider a function that schedules report generation at the end of the month:

```swift
extension BankAccount {
  func endOfMonth(month: Int, year: Int) {
    // Schedule a task to prepare an end-of-month report.
    detach {
      let transactions = await self.transactions(month: month, year: year)
      let report = Report(accountNumber: self.accountNumber, transactions: transactions)
      await report.email(to: self.accountOwnerEmailAddress)
    }
  }
}
```

A task created with `detach` runs concurrently with all other code. If the closure passed to `detach` were to be actor-isolated, we would introduce a data race on access to the mutable state on `BankAccount`. Actors prevent this data race by specifying that a `@Sendable` closure (described in [`Sendable` and `@Sendable` closures][se302], and used in the definition of `detach` in the [Structured Concurrency][sc] proposal) is always non-isolated. Therefore, it is required to asynchronously access any actor-isolated declarations.

A closure that is not `@Sendable` cannot escape the concurrency domain in which it was formed. Therefore, such a closure will be actor-isolated if it is formed within an actor-isolated context. This is useful, for example, when applying sequence algorithms like `forEach` where the provided closure will be called serially:

```swift
extension BankAccount {
  func close(distributingTo accounts: [BankAccount]) async {
    let transferAmount = balance / accounts.count

    accounts.forEach { account in                        // okay, closure is actor-isolated to `self`
      balance = balance - transferAmount            
      await account.deposit(amount: transferAmount)
    }
    
    await thief.deposit(amount: balance)
  }
}
```

A closure formed within an actor-isolated context is actor-isolated if it is non-`@Sendable`, and non-isolated if it is `@Sendable`. For the examples above:

* The closure passed to `detach` is non-isolated because that function requires a `@Sendable` function to be passed to it.
* The closure passed to `forEach` is actor-isolated to `self` because it takes a non-`@Sendable` function.

### Actor reentrancy

Actor-isolated functions are [reentrant](https://en.wikipedia.org/wiki/Reentrancy_(computing)). When an actor-isolated function suspends, reentrancy allows other work to execute on the actor before the original actor-isolated function resumes, which we refer to as *interleaving*. Reentrancy eliminates a source of deadlocks, where two actors depend on each other, can improve overall performance by not unnecessarily blocking work on actors, and offers opportunities for better scheduling of (e.g.) higher-priority tasks. However, it means that actor-isolated state can change across an `await` when an interleaved task mutates that state, meaning that developers must be sure not to break invariants across an await. In general, this is the [reason for requiring `await`](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0296-async-await.md#suspension-points) on asynchronous calls, because various state (e.g., global state) can change when a call suspends.

This section explores the issue of reentrancy with examples that illustrate both the benefits and problems with both reentrant and non-reentrant actors, and settles on re-entrant actors. Alternatives Considered provides potential future directions to provide more control of re-entrancy, including [non-reentrant actors](#non-reentrancy) and [task-chain reentrancy](#task-chain-reentrancy).

#### "Interleaving" execution with reentrant actors

Reentrancy means that execution of asynchronous actor-isolated functions may "interleave" at suspension points, leading to increased complexity in programming with such actors, as every suspension point must be carefully inspected if the code *after* it depends on some invariants that could have changed before it suspended.

Interleaving executions still respect the actor's "single-threaded illusion", i.e., no two functions will ever execute *concurrently* on any given actor. However they may *interleave* at suspension points. In broad terms this means that reentrant actors are *thread-safe* but are not automatically protecting from the "high level" kinds of races that may still occur, potentially invalidating invariants upon which an executing asynchronous function may be relying on. To further clarify the implications of this, let us consider the following actor, which thinks of an idea and then returns it, after telling its friend about it.

```swift
actor DecisionMaker {
  let friend: Friend
  
  // actor-isolated opinion
  var opinion: Decision = .noIdea

  func thinkOfGoodIdea() async -> Decision {
    opinion = .goodIdea                       // <1>
    await friend.tell(opinion, heldBy: self)  // <2>
    return opinion // 🤨                      // <3>
  }

  func thinkOfBadIdea() async -> Decision {
    opinion = .badIdea                       // <4>
    await friend.tell(opinion, heldBy: self) // <5>
    return opinion // 🤨                     // <6>
  }
}
```

In the example above the `DecisionMaker` can think of a good or bad idea, shares that opinion with a friend, and returns that opinion that it stored. Since the actor is reentrant this code is wrong and will return an arbitrary opinion if the actor begins to think of a few ideas at the same time.

This is exemplified by the following piece of code, exercising the `decisionMaker` actor:

```swift
let goodThink = detach { await decisionMaker.thinkOfGoodIdea() }  // runs async
let badThink = detach { await decisionMaker.thinkOfBadIdea() } // runs async

let shouldBeGood = await goodThink.get()
let shouldBeBad = await badThink.get()

await shouldBeGood // could be .goodIdea or .badIdea ☠️
await shouldBeBad
```

This snippet _may_ result (depending on timing of the resumptions) in the following execution:

```swift
opinion = .goodIdea                // <1>
// suspend: await friend.tell(...) // <2>
opinion = .badIdea                 // | <4> (!)
// suspend: await friend.tell(...) // | <5>
// resume: await friend.tell(...)  // <2>
return opinion                     // <3>
// resume: await friend.tell(...)  // <5>
return opinion                     // <6>
```

But it _may_ also result in the "naively expected" execution, i.e. without interleaving, meaning that the issue will only show up intermittently, like many race conditions in concurrent code.

The potential for interleaved execution at suspension points is the primary reason for the requirement that every suspension point be [marked by `await`](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0296-async-await.md#suspension-points) in the source code, even though `await` itself has no semantic effect. It is an indicator that any shared state might change across the `await`, so one should avoid breaking invariants across an `await`, or otherwise depending on the state "before" to be identical to the state "after".

Generally speaking, the easiest way to avoid breaking invariants across an `await` is to encapsulate state updates in synchronous actor functions. Effectively, synchronous code in an actor provides a [critical section](https://en.wikipedia.org/wiki/Critical_section), whereas an `await` interrupts a critical section. For our example above, we could effect this change by separating "opinion formation" from "telling a friend your opinion". Indeed, telling your friend your opinion might reasonably cause you to change your opinion!

#### Deadlocks with non-reentrant actors

The opposite of reentrant actor functions are "non-reentrant" functions and actors. This means that while an actor is processing an incoming actor function call (message), it will *not* process any other message from its mailbox until it has completed running this initial function. Essentially, the entire actor is blocked from executing until that task completes.

If we take the example from the previous section and use a non-reentrant actor, it will execute correctly, because no work can be scheduled on the actor until `friend.tell` has completed:

```swift
// assume non-reentrant
actor DecisionMaker {
  let friend: DecisionMaker
  var opinion: Decision = .noIdea

  func thinkOfGoodIdea() async -> Decision {
    opinion = .goodIdea                                   
    await friend.tell(opinion, heldBy: self)
    return opinion // ✅ always .goodIdea
  }

  func thinkOfBadIdea() async -> Decision {
    opinion = .badIdea
    await friend.tell(opinion, heldBy: self)
    return opinion // ✅ always .badIdea
  }
}
```

However, non-entrancy can result in deadlock if a task involves calling back into the actor. For example, let's stretch this example further and have our friend try to convince us to change a bad idea:

```swift
extension DecisionMaker {
  func tell(_ opinion: Decision, heldBy friend: DecisionMaker) async {
    if opinion == .badIdea {
      await friend.convinceOtherwise(opinion)
    }
  }
}
```

With non-reentrant actors, `thinkOfGoodIdea()` will succeed under this implementation, because `tell` essentially does nothing. However, `thinkOfBadIdea()` will deadlock because the original decision maker (call it `A`) is locked when it calls `tell` on another decision maker (call it `B`). `B` then tries to convince `A` otherwise, but that call cannot execute because `A` is already locked. Hence, the actor itself deadlocks and cannot progress.

> The term "deadlock" used in these discussions refer to actors asynchronously waiting on "each other," or on "future work of self". No thread blocking is necessary to manifest this issue.

In theory, a fully non-reentrant model would also deadlock when calling asynchronous functions on `self`. However, since such calls are statically determinable to be on `self`, they would execute immediately and therefore not block.

Deadlocks with non-reentrant actors could be detected with runtime tools that detect cyclic call graphs once they've occurred, much like tools exist to find reference cycles in data structures at runtime. However, such deadlocks cannot generally be identified statically (e.g., with the compiler or static analysis), because call graphs require whole-program knowledge and can change dynamically depending on the data provided to the program.

Deadlocked actors would be sitting around as inactive zombies forever. Some runtimes solve deadlocks like this by making every single actor call have a timeout (such timeouts are already useful for distributed actor systems). This would mean that each `await` could potentially `throw`, and that either timeouts or deadlock detection would have to always be enabled. We feel this would be prohibitively expensive, because we envision actors being used in the vast majority of concurrent Swift applications. It would also muddy the waters with respect to cancellation, which is intentionally designed to be explicit and cooperative. Therefore, we feel that the approach of automatically cancelling on deadlocks does not fit well with the direction of Swift Concurrency.

#### Unnecessary blocking with non-reentrant actors

Consider an actor that handles the download of various images and maintains a cache of what it has downloaded to make subsequent accesses faster:

```swift
// assume non-reentrant
actor ImageDownloader { 
  var cache: [URL: Image] = [:]

  func getImage(_ url: URL) async -> Image {
    if let cachedImage = cache[url] {
      return cachedImage
    }
    
    let data = await download(url)
    let image = await Image(decoding: data)
    return cache[url, default: image]
  }
}
```

This actor is functionally correct, whether it is re-entrant or not. However, if it is non-reentrant, it will completely serialize the download of images: once a single client asked for an image, all other clients are blocked from starting any requests--even ones that would hit the cache or which ask for images at different URLs---until that first client has had its image fully downloaded and decoded.

With a reentrant actor, multiple clients can fetch images independently, so that (say) they can all be at different stages of downloading and decoding an image. The serialized execution of partial tasks on the actor ensures that the cache itself can never get corrupted. At worst, two clients might ask for the same image URL at the same time, in which there will be some redundant work. 

#### Existing practice

There are a number of existing actor implementations that have considered the notion of reentrancy:

* Erlang/Elixir ([gen_server](https://medium.com/@eduardbme/erlang-gen-server-never-call-your-public-interface-functions-internally-c17c8f28a1ee)) showcases a simple "loop/deadlock" scenario and how to detect and fix it,
* Akka ([Persistence persist/persistAsync](https://doc.akka.io/docs/akka/current/persistence.html#relaxed-local-consistency-requirements-and-high-throughput-use-cases) is effectively _non-reentrant behavior by default_, and specific APIs are designed to allow programmers to _opt into_ reentrant whenever it would be needed. In the linked documentation `persistAsync` is the re-entrant version of the API, and it is used _very rarely_ in practice. Akka persistence and this API has been used to implement bank transactions and process managers, by relying on the non-reentrancy of `persist()` as a killer feature, making implementations simple to understand and _safe_. Note that Akka is built on top of Scala, which does not provide `async`/`await`. This means that mailbox-processing methods are more synchronous in nature, and rather than block the actor while waiting for a response, they would handle the response as a separate message receipt.
* Orleans ([grains](https://dotnet.github.io/orleans/docs/grains/reentrancy.html)) are also non-reentrant by default, but offer extensive configuration around reentrancy. Grains and specific methods can be marked as being re-entrant, and there is even a dynamic mechanism by which one can implement a run-time predicate to determine whether an invocation can interleave. Orleans is perhaps closest to the Swift approach described here, because it is built on top of a language that provides `async`/`await` (C#). Note that Orleans *had* a feature called [call-chain reentrancy](https://dotnet.github.io/orleans/docs/grains/reentrancy.html#reentrancy-within-a-call-chain), which we feel is a promising potential direction: we cover it later in this proposal in our section on [task-chain reentrancy](#task-chain-reentrancy).

#### Reentrancy Summary

This proposal provides only reentrant actors. However, the [Future Directions](#future-directions) section describes potential future design directions that could add opt-in non-reentrancy.

> **Rationale**: Reentrancy by default all but eliminates the potential for deadlocks. Moreover, it helps ensure that actors can make timely progress within a concurrent system, and that a particular actor does not end up unnecessarily blocked on a long-running asynchronous operation (say, downloading a file). The mechanisms for ensuring safe interleaving, such as using synchronous code when performing mutations and being careful not to break invariants across `await` calls, are already present in the proposal.

### Protocol conformances

All actor types implicitly conform to a new protocol, `Actor`:

```swift
protocol Actor : AnyObject, Sendable { }
```

> **Note**: The definition of the `Actor` protocol is intentionally left blank. The [custom executors proposal][customexecs] will introduce requirements into the `Actor` protocol. These requirements will be implicitly synthesized by the implementation when not explicitly provided, but can be explicitly provided to allow actors to control their own serialized execution.

The `Actor` protocol can be used to write generic operations that work across all actors, including extending all actor types with new operations. As with actor types, instance properties, functions, and subscripts defined on the `Actor` protocol (including extensions thereof) are actor-isolated to the `self` actor. For example, 

```swift
protocol DataProcessible: Actor {  // only actor types can conform to this protocol
  var data: Data { get }           // actor-isolated to self
}

extension DataProcessible {
  func compressData() -> Data {    // actor-isolated to self
    // use data synchronously
  }
}

actor MyProcessor : DataProcessible {
  var data: Data                   // okay, actor-isolated to self
  
  func doSomething() {
    let newData = compressData()   // okay, calling actor-isolated method on self
    // use new data
  }
}

func doProcessing<T: DataProcessible>(processor: T) async {
  await processor.compressData() // not actor-isolated, so we must interact asynchronously with the actor
}
```

No other kind of concrete type (class, enum, struct, etc.) can conform to the `Actor` protocol, because they cannot define actor-isolated operations.

Actors can also conform to protocols with `async` requirements, because all clients will already have to interact with those requirements asynchronously, giving the actor the ability to protect its isolated state. For example:

```swift
protocol Server {
  func send<Message: MessageType>(message: Message) async throws -> Message.Reply
}

actor MyActor: Server {
  func send<Message: MessageType>(message: Message) async throws -> Message.Reply { // okay: this method is actor-isolated to 'self', satisfies asynchronous requirement
  }
}
```

Actors cannot otherwise be made to conform to non-`Actor` protocols with synchronous requirements. However, there is a separate proposal on [controlling actor isolation][isolationcontrol] that allows such conformances when they can implemented in a manner that does not reference any mutable actor state.

## Detailed design

### Actors

An actor type can be declared with the `actor` keyword:

```swift
/// Declares a new type BankAccount
actor BankAccount {
  // ...
}
```

Each instance of the actor represents a unique actor. The term "actor" can be used to refer to either an instance or the type; where necessary, one can refer to the "actor instance" or "actor type" to disambiguate.

Actors are similar to other concrete nominal types in Swift (enums, structs, and classes). Actor types can have `static` and instance methods, properties, and subscripts. They have stored properties and initializers like structs and classes. They are reference types like classes, but do not support inheritance, and therefore do not have (or need) features such as `required` and `convenience` initializers, overriding, or `class` members, `open` and `final`. Where actor types differ in behavior from other types is primarily driven by the rules of actor isolation, described below.

By default, the instance methods, properties, and subscripts of an actor have an isolated `self` parameter. This is true even for methods added retroactively on an actor via an extension, like any other Swift type. Static methods, properties, and subscripts do not have a `self` parameter that is an instance of the actor, so they are not actor-isolated.

```swift
extension BankAccount {
  func acceptTransfer(amount: Double) async { // actor-isolated
    balance += amount
  }
}  
```

### Actor isolation checking

Any given declaration in a program is either actor-isolated or is non-isolated. A function (including accessors) is actor-isolated if it is defined on an actor type (including protocols where `Self` conforms to `Actor`, and extensions thereof). A mutable instance property or instance subscript is actor-isolated if it is defined on an actor type. Declarations that are not actor-isolated are called non-isolated.

The actor isolation rules are checked in a number of places, where two different declarations need to be compared to determine if their usage together maintains actor isolation. There are several such places:
* When the definition of one declaration (e.g., the body of a function) references another declaration, e.g., calling a function, accessing a property, or evaluating a subscript.
* When one declaration satisfies a protocol requirement.

We'll describe each scenario in detail.

#### References and actor isolation

An actor-isolated non-`async` declaration can only be synchronously accessed from another declaration that is isolated to the same actor. For synchronous access to an actor-isolated function, the function must be called from another actor-isolated function. For synchronous access to an actor-isolated instance property or instance subscript, the instance itself must be actor-isolated.

An actor-isolated declaration can be asynchronously accessed from any declaration, whether it is isolated to another actor or is non-isolated. Such accesses are asynchronous operations, and therefore must be annotated with `await`. Semantically, the progam will switch to the actor to perform the synchronous operation, and then switch back to the caller's executor afterward.

For example:

```swift
actor MyActor {
  let name: String
  var counter: Int = 0
  func f()
}

extension MyActor {
  func g(other: MyActor) async {
    print(name)          // okay, name is non-isolated
    print(other.name)    // okay, name is non-isolated
    print(counter)       // okay, g() is isolated to MyActor
    print(other.counter) // error: g() is isolated to "self", not "other"
    f()                  // okay, g() is isolated to MyActor
    await other.f()      // okay, other is not isolated to "self" but asynchronous access is permitted
  }
}
```

#### Protocol conformance

When a given declaration (the "witness") satisfies a protocol requirement (the "requirement"), the protocol requirement can be satisfied by the witness if:

* The requirement is `async`, or
* the requirement and witness are both actor-isolated.

An actor can satisfy an asynchronous requirement because any uses of the requirement are asynchronous, and can therefore suspend until the actor is available to execute them. Note that an actor can satisfy an asynchronous requirement with a synchronous one, in which case the normal notion of asynchronously accessing a synchronous declaration on an actor applies. For example:

```swift
protocol Server {
  func send<Message: MessageType>(message: Message) async throws -> Message.Reply
}

actor MyServer : Server {
  func send<Message: MessageType>(message: Message) throws -> Message.Reply { ... }  // okay, asynchronously accessed from clients of the protocol
}
```

### Partial applications

Partial applications of isolated functions are only permitted when the expression is a direct argument whose corresponding parameter is non-Sendable. For example:

```swift
func runLater<T>(_ operation: @Sendable @escaping () -> T) -> T { ... }

actor A {
  func f(_: Int) -> Double { ... }
  func g() -> Double { ... }
  
  func useAF(array: [Int]) {
    array.map(self.f)                     // okay
    detach(operation: self.g)             // error: self.g has non-sendable type () -> Double that cannot be converted to a @Sendable function type
    runLater(self.g)                      // error: cannot convert value of non-sendable function type () -> Double to sendable function type
  }
}
```

These restrictions follow from the actor isolation rules for the "desugaring" of partial applications to closures. The two erroneous cases above fall out from the fact that the closure would be non-isolated in a closure that performs the call, so access to the actor-isolated function `g` would have to be asynchronous. Here are the "desugared" forms of the partial applications:


```swift
extension A {
  func useAFDesugared(a: A, array: [Int]) {
    array.map { f($0) } )      // okay
    detach { g() }             // error: self is non-isolated, so call to `g` cannot be synchronous
    runLater { g() }           // error: self is non-isolated, so the call to `g` cannot be synchronous
  }
}
```

### Key paths

A key path cannot involve a reference to an actor-isolated declaration:

```swift
actor A {
  var storage: Int
}

let kp = \A.storage  // error: key path would permit access to actor-isolated storage
```

> **Rationale**: Allowing the formation of a key path that references an actor-isolated property or subscript would permit accesses to the actor's protected state from outside of the actor isolation domain. As an alternative to this rule, we could remove the `Sendable` conformance from key paths, such that one could form key paths to actor-isolated state but they could not be shared.

### inout parameters

Actor-isolated stored properties can be passed into synchronous functions via `inout` parameters, but it is ill-formed to pass them to asynchronous functions via `inout` parameters. For example:

```swift
func modifiesSynchronously(_: inout Double) { }
func modifiesAsynchronously(_: inout Double) async { }

extension BankAccount {
  func wildcardBalance() async {
    modifiesSynchronously(&balance)        // okay
    await modifiesAsynchronously(&balance) // error: actor-isolated property 'balance' cannot be passed 'inout' to an asynchronous function
  }
}  

class C { var state : Double }
struct Pair { var a, b : Double }
actor A {
  let someC : C
  var somePair : Pair

  func inoutModifications() async {
    modifiesSynchronously(&someC.state)        // okay
    await modifiesAsynchronously(&someC.state) // not okay
    modifiesSynchronously(&somePair.a)         // okay
    await modifiesAsynchronously(&somePair.a)  // not okay
  }
}
```

> **Rationale**: this restriction prevents exclusivity violations where the modification of the actor-isolated `balance` is initiated by passing it as `inout` to a call that is then suspended, and another task executed on the same actor then attempts to access `balance`. Such an access would then result in an exclusivity violation that will terminate the program. While the `inout` restriction is not required for memory safety (because errors will be detected at runtime), the default re-entrancy of actors makes it very easy to introduce non-deterministic exclusivity violations. Therefore, we introduce this restriction to eliminate that class of problems that where a race would trigger an exclusivity violation.

### Actor interoperability with Objective-C

An actor type can be declared `@objc`, which implicitly provides conformance to `NSObjectProtocol`:

```swift
@objc actor MyActor { ... }
```

A member of an actor can only be `@objc` if it is either `async` or is not isolated to the actor. Synchronous code that is within the actor's isolation domain can only be invoked on `self` (in Swift). Objective-C does not have knowledge of actor isolation, so these members are not permitted to be exposed to Objective-C. For example:

```swift
@objc actor MyActor {
    @objc func synchronous() { } // error: part of actor's isolation domain
    @objc func asynchronous() async { } // okay: asynchronous, exposed to Objective-C as a method that accepts a completion handler
    @objc nonisolated func notIsolated() { } // okay: non-isolated
}
```

## Source compatibility

This proposal is mostly additive, and should not break source compatibility. The `actor` contextual keyword to introduce actors is a parser change that does not break existing code, and the other changes are carefully staged so they do not change existing code. Only new code that introduces actors or actor-isolation attributes will be affected.

## Effect on ABI stability

This is purely additive to the ABI. Actor isolation itself is a static notion that is not part of the ABI.

## Effect on API resilience

Nearly all changes in actor isolation are breaking changes, because the actor isolation rules require consistency between a declaration and its users:

* A class cannot be turned into an actor or vice versa.
* The actor isolation of a public declaration cannot be changed.

## Future Directions

### Non-reentrancy

We could introduce a `@reentrant` attribute may be added to any actor-isolated function, actor, or extension of an actor to describe how it is reentrant. The attribute would have several forms:

* `@reentrant`: Indicates that each potential suspension point within the function bodies covered by the attribute is reentrant.
* `@reentrant(never)`: Indicates that each potential suspension point within the function bodies covered by the attribute is non-reentrant.

A non-reentrant potential suspension point prevents any other asynchronous call from executing on the actor until it has completed. Note that asynchronous calls to non-reentrant async functions directly on `self` are exempted from this check, so an actor can asynchronously call itself without producing a deadlock.

> **Rationale**: Allowing direct calls on `self` eliminates an obvious set of deadlocks, and requires only the same static knowledge as actor-isolation checking for synchronous access to actor-isolated state.

It is an error to have a `@reentrant` attribute on a non-isolated function, non-actor type, or extension of a non-actor type. Only one `@reentrant` attribute may occur on a given declaration. The reentrancy of an actor-isolated non-type declaration is determined by finding a suitable `@reentrant` attribute. The search is as follows:

1. The declaration itself.
2. If the declaration is a non-type member of an extension, the extension.
3. If the declaration is a non-type member of a type (or extension thereof), the type definition.

If there is no suitable `@reentrant` attribute, an actor-isolated declaration is reentrant.

Here's an example illustrating how the `@reentrant` attribute can be applied at various points:

```swift
actor Stage {
  @reentrant(never) func f() async { ... }    // not reentrant
  func g() async { ... }                      // reentrant
}

@reentrant(never)
extension Stage {
  func h() async { ... }                      // not reentrant
  @reentrant func i() async { ... }           // reentrant

  actor InnerChild {                          // reentrant, not affected by enclosing extension
    func j() async { ... }                    // reentrant
  }

  nonisolated func k() async { .. }     // okay, reentrancy is uninteresting
  nonisolated @reentrant func l() async { .. } // error: @reentrant on non-actor-isolated
}

@reentrant func m() async { ... } // error: @reentrant on non-actor-isolated
```

The attribute approach is not the only possible design here. At an implementation level, the actual blocking will be handled at each asynchronous call site. Instead of an attribute that affects potentially many asynchronous calls, we could introduce a different form of `await` that does the blocking, e.g.,

```swift
await(blocking) friend.tell(opinion, heldBy: self)
```

### Task-chain reentrancy

The discussion of reentrant and non-reentrant actors treats reentrancy as a binary choice, where all forms of reentrancy are considered to be equally likely to introduce hard-to-reason-about data races. However, a frequent and usually quite understandable way of interacting between actors which are simply "conversations" between two or more actors in order fo fulfill some initial request. In synchronous code, it's common to have two or more different classes call back into each other with synchronous calls. For example, here is a silly implementation of `isEven` that uses mutual recursion between two classes:

```swift
class OddOddySync {
  let evan: EvenEvanSync!

  func isOdd(_ n: Int) -> Bool {
    if n == 0 { return true }
    return evan.isEven(n - 1)
  }
}

class EvenEvanSync {
  let oddy: OddOddySync!

  func isEven(_ n: Int) -> Bool {
    if n == 0 { return false }
    return oddy.isOdd(n - 1)
  }
}
```

This code is depending on the two methods of these classes to effectively be "reentrant" within the same call stack, because one will call into the other (and vice-versa) as part of the computation. Now, take this example and make it asynchronous using actors:

```swift
@reentrant(never)
actor OddOddy {
  let evan: EvenEvan!

  func isOdd(_ n: Int) async -> Bool {
    if n == 0 { return true }
    return await evan.isEven(n - 1)
  }
}

@reentrant(never)
actor EvenEvan {
  let oddy: OddOddy!

  func isEven(_ n: Int) async -> Bool {
    if n == 0 { return false }
    return await oddy.isOdd(n - 1)
  }
}
```

Under `@reentrant(never)`, this code will deadlock, because a call from `EvanEvan.isEven` to `OddOddy.isOdd` will then depend on another call to `EvanEvan.isEven`, which cannot proceed until the original call completes. One would need to make these methods to be reentrant to eliminate the deadlock.

With Swift embracing [Structured Concurrency][sc] as a core building block of its concurrency story, we may be able to do better than outright banning reentrancy. In Swift, every asynchronous operation is part of a `Task` which encapsulates the general computation taking place, and every asynchronous operation spawned from such task becomes a child task of the current task. Therefore, it is possible to know whether a given asynchronous call is part of the same task hierarchy, which is the rough equivalent to being in the same call stack in synchronous code.

We could introduce a new kind of reentrancy, *task-chain reentrancy*, which allows reentrant calls on behalf of the given task or any of its children. This resolves both the deadlock we encountered in the `convinceOtherwise` example from the section on [deadlocks](#deadlocks-with-non-reentrant-actors) as well as the mutually-recursive `isEven` example above, while still preventing reentrancy from unrelated tasks. This reentrancy therefore mimics synchronous code more closely, eliminating many deadlocks without allowing unrelated interleavings to break the high-level invariants of an actor.

There are a few reasons why we are not currently comfortale including task-chain reentrancy in the proposal:
* The task-based reentrancy approach doesn't seem to have been tried at scale. Orleans documents support for [reentrancy in a call chain](https://dotnet.github.io/orleans/docs/grains/reentrancy.html#reentrancy-within-a-call-chain), but the implementation was fairly limited and it was eventually [removed](https://twitter.com/reubenbond/status/1349725703634251779). From the Orleans experience, it is hard to assess whether the problem is with the idea or the specific implementation.
* We do not yet know of an efficient implementation technique for this approach within the actor runtime.

If we can address the above, task-chain reentrancy can be introduced into the actor model with another spelling of the reentrancy attribute such as `@reentrant(task)`, and may provide the best default.

## Alternatives considered

### Actor inheritance

Earlier pitches and the first reviewed version of this proposal allowed actor inheritance. Actor inheritance followed the rules of class inheritance, albeit with specific additional rules required to maintain actor isolation:

* An actor could not inherit from a class, and vice-versa.
* An overriding declaration must not be more isolated than the overridden declaration.

Subsequent review discussion determined that the conceptual cost of actor inheritance outweighed its usefulness, so it has been removed from this proposal. The form that actor inheritance would take in the language is well-understand from prior iterations of this proposal and its implementation, so this feature could be re-introduced at a later time.

### Cross-actor lets

This proposal allows synchronous access to `let` properties on an actor instance from anywhere within the same module as the actor is defined:

```swift
// in module BankActors
public actor BankAccount {
  public let accountNumber: Int
}

func print(account: BankAccount) {
  print(account.accountNumber) // okay: synchronous access to an actor's let property
}  
```

Outside of the module, access must be asynchronous:

```swift
import BankActors

func otherPrint(account: BankAccount) async {
  print(account.accountNumber)         // error: cannot synchronously access immutable 'let' outside the actor's module
  print(await account.accountNumber)   // okay to asynchronously access
}
```

The requirement for asynchronous access from outside of the module preserves a longstanding freedom for library implementors, which allows a public `let` to be refactored into a `var` without breaking any clients. It is consistent with Swift's policy of maximizing the freedom for library implementors to alter the implementation without breaking clients. Without requiring asynchronous access from other modules, the `otherPrint(account:)` function above were permitted to reference `accountNumber` synchronously. If the author of `BankActors` then changed the account number into a `var`, it would break existing client code:

```swift
public actor BankAccount {
  public var accountNumber: Int     // version 2 makes this mutable, but would break clients if synchronous access to 'let's were allowed outside the module
}
```

There are a number of other language features that take this same approach of reducing boilerplate and simplifying the language within a module, then requiring the use of additional language features when an entity is used from outside the module. For example:

* Access control defaults to `internal`, so you can use a declaration across your whole module but have to explicitly opt in to making it available outside your module (e.g., via `public`). In other words, you can ignore access control until the point where you need to make something `public` for use from another module.
* The implicit memberwise initializer of a struct is `internal`. You need to write a `public` initializer yourself to commit to allowing that struct to be initialized with exactly that set of parameters.
* Inheritance from a class is permitted by default when the superclass is in the same module. To inherit from a superclass defined in a different module, that superclass must be explicitly marked `open`. You can ignore `open` until you want to guarantee this ability to clients outside of the module.
* Overriding a declaration in a class is permitted by default when the overridden declaration is in the same module. To override from a declaration in a different module, that overrides declaration must be explicitly marked `open`.

SE-0313 "[Improved control over actor isolation][isolationcontrol]" provides an explicit way to give clients the freedom to synchronously access immutable actor state via the `nonisolated` keyword, e.g.,

```swift
// in module BankActors
public actor BankAccount {
  public nonisolated let accountNumber: Int  // can be accessed synchronously from any module due to the explicit 'nonisolated'
}
```

The original accepted version of this proposal required *all* access to immutable actor storage be asynchronous, and left any synchronous access to explicit `nonisolated` annotations as spelled out in [SE-0313][isolationcontrol]. However, experience with that model showed that it had a number of problems that affected teachability of the model:

* Developers were almost immediately confronted with the need to use `nonisolated` when writing actor code. This goes against the principle of [progressive disclosure](https://www.interaction-design.org/literature/topics/progressive-disclosure) that Swift tries to follow for advanced features. Aside from `nonisolated let`, uses of `nonisolated` are fairly rare.
* Immutable state is a key tool for writing safe concurrency code. A `let` of `Sendable` type is conceptually safe to reference from concurrency code, and works in other contexts (e.g., local variables). Making some immutable state concurrency-safe while other state is not complicates the story about data-race-safe concurrent programming. Here's an example of the existing restrictions around `@Sendable`, which were defined in [SE-0302][se302]:

  ```swift
  func test() {
    let total = 100
    var counter = 0
  
    asyncDetached {
      print(total) // okay to reference immutable state
      print(counter) // error, cannot reference a `var` from a @Sendable closure
    }
    
    counter += 1
  }
  ```

By allowing synchronous access to actor `let`s within a module, we provide a smoother learning curve for actor isolation and embrace (rather than subvert) the longstanding and pervasive idea that immutable data is safe for concurrency, while still addressing the concerns from the second review that unrestricted synchronous access to actor `let`s is implicitly committing a library author to never make that state mutable. It follows existing precedent in the Swift language of making in-module interactions simpler than interactions across modules.

## Revision history

* Changes in the post-review amendment to the proposal:
  * Cross-after references to instance `let` properties from a different module must be asynchronous; within the same module they will be synchronous.
* Changes in the final accepted version of the proposal:
  * Cross-actor references to instance `let` properties must be asynchronous.
* Changes in the second reviewed proposal:
  * Escaping closures can now be actor-isolated; only `@Sendable` prevents isolation.
  * Removed actor inheritance. It can be considered at some future point.
  * Added "cross-actor lets" to Alternatives Considered. While there is no change to the proposed direction, the issue is explained here for further discussion.
  * Replaced `Task.runDetached` with `detach` to match updates to the [Structured Concurrency proposal][sc].
* Changes in the seventh pitch:
  * Removed isolated parameters and `nonisolated` from this proposal. They'll come in a follow-up proposal on [controlling actor isolation][isolationcontrol].
* Changes in the sixth pitch:
  * Make the instance requirements of `Actor` protocols actor-isolated to `self`, and allow actor types to conform to such protocols using actor-isolated witnesses.
  * Reflow the "Proposed Solution" section to get the bigger ideas out earlier.
  * Remove `nonisolated(unsafe)`.
* Changes in the fifth pitch:
  * Drop the prohibition on having multiple `isolated` parameters. We don't need to ban it.
  * Add the `Actor` protocol back, as an empty protocol whose details will be filled in with a subsequent proposal for [custom executors][customexecs].
  * Replace `ConcurrentValue` with `Sendable` and `@concurrent` with `@Sendable` to track the evolution of [SE-0302][se302].
  * Clarify the presentation of actor isolation checking.
  * Add more examples for non-isolated declarations.
  * Added a section on isolated or "sync" actor types.
* Changes in the fourth pitch:
  * Allow cross-actor references to actor properties, so long as they are reads (not writes or `inout` references)
  * Added `isolated` parameters, to generalize the previously-special behavior of `self` in an actor and make the semantics of `nonisolated` more clear.
  * Limit `nonisolated(unsafe)` to stored instance properties. The prior definition was far too broad.
  * Clarify that `super` is isolated if `self` is.
  * Prohibit references to actor-isolated declarations in key paths.
  * Clarify the behavior of partial applications.
  * Added a "future directions" section describing isolated protocol conformances.
* Changes in the third pitch:
  * Narrow the proposal down to only support re-entrant actors. Capture several potential non-reentrant designs in the Alternatives Considered as possible future extensions.
  * Replaced `@actorIndependent` attribute with a `nonisolated` modifier, which follows the approach of `nonmutating` and ties in better with the "actor isolation" terminology (thank you to Xiaodi Wu for the suggestion).
  * Replaced "queue" terminology with the more traditional "mailbox" terminology, to try to help alleviate confusion with Dispatch queues.
  * Introduced "cross-actor reference" terminology and the requirement that cross-actor references always traffic in `Sendable` types.
  * Reference `@concurrent` function types from their separate proposal.
  * Moved Objective-C interoperability into its own section.
  * Clarify the "class-like" behaviors of actor types, such as satisfying an `AnyObject` conformance.
* Changes in the second pitch:
  * Added a discussion of the tradeoffs with actor reentrancy, performance, and deadlocks, with various examples, and the addition of new attribute `@reentrant(never)` to disable reentrancy at the actor or function level.
  * Removed global actors; they will be part of a separate document.
  * Separated out the discussion of data races for reference types.
  * Allow asynchronous calls to synchronous actor methods from outside the actor.
  * Removed the `Actor` protocol; we'll tackle customizing actors and executors in a separate proposal.
  * Clarify the role and behavior of actor-independence.
  * Add a section to "Alternatives Considered" that discusses actor inheritance.
  * Replace "actor class" with "actor".
* Original pitch [document](https://github.com/DougGregor/swift-evolution/blob/6fd3903ed348b44496b32a39b40f6b6a538c83ce/proposals/nnnn-actors.md)


[sc]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md
[se302]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md
[customexecs]: https://github.com/rjmccall/swift-evolution/blob/custom-executors/proposals/0000-custom-executors.md
[isolationcontrol]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0313-actor-isolation-control.md
