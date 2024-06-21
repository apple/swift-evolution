# Effectful Read-only Properties

* Proposal: [SE-0310](0310-effectful-readonly-properties.md)
* Author: [Kavon Farvardin](https://github.com/kavon)
* Review Manager: [Doug Gregor](https://github.com/DougGregor)
* Status: **Implemented (Swift 5.5)**
* Decision Notes: [Pitch](https://forums.swift.org/t/pitch-effectful-read-only-properties/44090),
             [Acceptance](https://forums.swift.org/t/accepted-se-0310-effectful-read-only-properties/47739)
* Implementation: [apple/swift#36430](https://github.com/apple/swift/pull/36430),
                  [apple/swift#36670](https://github.com/apple/swift/pull/36670),
                  [apple/swift#37225](https://github.com/apple/swift/pull/37225)
* Available in [recent `main` snapshots](https://swift.org/download/#snapshots).

## Introduction

Nominal types such as classes, structs, and enums in Swift support [computed properties](https://docs.swift.org/swift-book/LanguageGuide/Properties.html) and [subscripts](https://docs.swift.org/swift-book/LanguageGuide/Subscripts.html), which are members of the type that invoke programmer-specified computations when getting or setting them.  The recently accepted proposal [SE-0296](0296-async-await.md) introduced asynchronous functions via `async`, in conjunction with `await`, but did not specify that computed properties or subscripts can support effects like asynchrony.  Furthermore, to take full advantage of `async` properties, the ability to specify that a property `throws` is also important.  This document aims to partially fill in this gap by proposing a syntax and semantics for effectful read-only computed properties and subscripts.

#### Terminology
A *read-only computed property* is a computed property that only defines a `get` accessor. Similarly, a *read-only subscript* is a subscript that only defines a `get` accessor. Throughout the remainder of this proposal, any unqualified mention of a "property" or "subscript" refers to a read-only version of that member.  Furthermore, unless otherwise specified, the concepts of synchrony, asynchrony, and the definition of something being "async" or "sync" are as described in [SE-0296](0296-async-await.md).

An *effect* is an observable behavior of a function. Swift's type system tracks a few kinds of effects: `throws` indicates that the function may return along an exceptional failure path with an `Error`, `rethrows` indicates that a throwing closure passed into the function may be invoked, and `async` indicates that the function may reach a suspension point.

This proposal's examples use features from a number of other recent proposals, such as [structured concurrency](0304-structured-concurrency.md) and [actors](0306-actors.md).  Overviews of those features are out of the scope of this proposal, but basic understanding of the importance of those features is required to fully grasp the motivation of this proposal.

## Motivation

An asynchronous function is designed for computations that may or always will suspend to perform a context switch before returning.  Of primary concern in this proposal are scenarios where the use of Swift concurrency features are limited due to the lack of *effectful read-only computed properties and subscripts* (which will be referred to as simply "effectful properties" from now on), so we will consider those first.  Then, we will consider programming patterns in existing Swift code where the availability of effectful properties would help simplify the code.

#### Swift Concurrency

An asynchronous call cannot appear within a synchronous context. This fundamental restriction means that computed properties and subscripts would be severely limited in their ability to use Swift's new concurrency features. The only concurrency capability available to them is creating detached tasks, but the completion of those tasks cannot be awaited in synchronous contexts in order to produce an answer:

```swift
// ...
class Socket {
  // ...
  public var alive: Bool {
    get {
      let handle = detach { await self.checkSocketStatus() }
      return await handle.get()
      //     ^~~~~ error: cannot 'await' in a sync context
    }
  }

  private func checkSocketStatus() async -> Bool { /* ... */ }
}
```

It would be better if the property could announce that it may require a suspension to retrieve an answer by allowing it to be marked as `async`. This way, `alive` could directly `await` the result of `checkSocketStatus`.

As one might imagine, a type that would like to take advantage of actors to isolate concurrent access to resources, while exposing information about those resources through properties, is not possible because one must use `await` to interact with the actor from outside of its isolation context:

```swift
struct Transaction { /* ... */ }
enum BankError: Error { /* ... */}

actor AccountManager {
  // NOTE: `getLastTransaction` is viewed as async 
  // when called from outside of the actor
  func getLastTransaction() -> Transaction { /* ... */ }
  func getTransactions(onDay: Date) async -> [Transaction] { /* ... */ }
}

class BankAccount {
  // ...
  private let manager: AccountManager?
  var lastTransaction: Transaction {
    get {
      guard let manager = manager else {
         throw BankError.NoManager
      // ^~~~~ error: cannot 'throw' in a non-throwing context
      }
      return await manager.getLastTransaction()
      //     ^~~~~ error: cannot 'await' in a sync context
    }
  }

  subscript(_ d: Date) -> [Transaction] {
    return await manager?.getTransactions(onDay: d) ?? []
    //     ^~~~~ error: cannot 'await' in a sync context
  }
}
```

<!-- realistic uses of `throw` in properties have been [detailed in a prior pitch](https://github.com/beccadax/swift-evolution/blob/72c55f33b94749e22637bd8277661599e9cd8007/proposals/0000-throwing-properties.md) -->

The use of `throw` in `lastTransaction` highlights a design pattern for properties and subscripts that is not available in Swift. Currently, `lastTransaction` would need to return values of type `Optional<Transaction>`, or some structurally similar `enum` or tuple, to account for the possibility of signaling failure. With the ability to `throw`, the property could describe what went wrong to its users, as opposed to simply returning `nil`, in a form compatible with the established error handling mechanisms in Swift.

Furthermore, a computed property getter cannot accept any explicit arguments, such as a completion handler, because the syntax for accessing a property is fundamentally designed _not_ to accept such arguments. Such restrictions around input arguments are one of the key differences between computed properties and methods. But, with the advent of `async` functions, an explicit completion-handler argument is no longer required for the function to be asynchronous. Thus, having `async` computed properties does not go against the existing syntax for computed property accesses: it's mainly a distinction in the type system.


#### Existing Code

According to the [API design guidelines](https://swift.org/documentation/api-design-guidelines/), computed properties that do not quickly return, which includes asynchronous operations, are not what programmers typically expect:

> **Document the complexity of any computed property that is not O(1).** People often assume that property access involves no significant computation, because they have stored properties as a mental model. Be sure to alert them when that assumption may be violated.

but, computed properties that may block or fail do appear in practice (see the motivation in [this pitch](https://github.com/beccadax/swift-evolution/blob/72c55f33b94749e22637bd8277661599e9cd8007/proposals/0000-throwing-properties.md)). 

As a real-world example of the need for effectful properties, [the SDK defines a protocol](https://developer.apple.com/documentation/avfoundation/avasynchronouskeyvalueloading) `AVAsynchronousKeyValueLoading`, which is solely dedicated to querying the status of a type's property, while offering an asynchronous mechanism to load the properties.  The types that conform to this protocol include [AVAsset](https://developer.apple.com/documentation/avfoundation/avasset), which relies on this protocol because its read-only properties are blocking and failable.

Let's distill the problem solved by `AVAsynchronousKeyValueLoading` into a simple example.  In existing code, it is impossible for property `get` access to also accept a completion handler, i.e., a closure for the property to invoke with the result of the operation. Thus, existing code that wished to use computed properties in scenarios where the computation may be blocking must use various workarounds. One workaround is to define an additional asynchronous version of the property as a method that accepts a completion handler:

```swift
class NetworkResource {
  var isAvailable: Bool {
    get { /* a possibly blocking operation */ }
  }
  func isAvailableAsync(completionHandler: ((Bool) -> Void)?) {
    // method that returns without blocking.
    // completionHandler is invoked once operation completes.
  }
}
```

The problem with this code is that, even with a comment on `isAvailable` to document that a `get` on this property may block, the programmer may mistakenly use it instead of `isAvailableAsync` because it is easy to ignore a comment. But, if `isAvailable`'s `get` were marked with `async`, then the type system will force the programmer to use `await`, which tells the programmer that the property's access may suspend until the operation completes. Thus, this `async` effect specifier enhances the recommendation made in the [API design guidelines](https://swift.org/documentation/api-design-guidelines/) by leveraging the type checker to warn users that the property access may involve significant computation.


## Proposed solution

For the problems detailed in the motivation section, the proposed solution is to allow `async`, `throws`, or both of these effect specifiers to be marked on a read-only computed property or subscript:

```swift
// ...
class BankAccount {
  // ...
  var lastTransaction: Transaction {
    get async throws {   // <-- NEW: effects specifiers!
      guard manager != nil else {
        throw BankError.notInYourFavor
      }
      return await manager!.getLastTransaction()
    }
  }

  subscript(_ day: Date) -> [Transaction] {
    get async { // <-- NEW: effects specifiers!
      return await manager?.getTransactions(onDay: day) ?? []
    }
  }
}
```

At corresponding access-sites of these properties, the expression will be treated as having the effects listed on the `get`-ter, requiring the usual `await` or `try` to surround it as-needed:

```swift
extension BankAccount {
  func meetsTransactionLimit(_ limit: Amount) async -> Bool {
    return try! await self.lastTransaction.amount < limit
    //                    ^~~~~~~~~~~~~~~~
    //                    this access is async & throws
  }                
}

  
func hadWithdrawlOn(_ day: Date, from acct: BankAccount) async -> Bool {
  return await !acct[day].allSatisfy { $0.amount >= Amount.zero }
  //            ^~~~~~~~~
  //            this access is async
}
```

Computed properties or subscripts only support effects specifiers if the only kind of accessor defined is a `get`. The main purpose of imposing this read-only restriction is to limit the scope of this proposal to a simple, useful, and easy-to-understand feature. Limiting effects specifiers to read-only properties and subscripts in this proposal does _not_ prevent future proposals from offering them for mutable members. For more discussion of why effectful setters are tricky, see the "Extensions considered" section of this proposal.

## Detailed design

This section takes a deep-dive into the changes made to Swift and its implementation as a result of this proposal.

#### Syntax and Semantics

Under the [grammar rules for declarations](https://docs.swift.org/swift-book/ReferenceManual/Declarations.html), under "Type Variable Properties", the proposed modifications and additions are:


```
getter-clause  → attributes? mutation-modifier? "get" getter-effects? code-block
getter-effects → "throws"
getter-effects → "async" "throws"?
```

where `getter-effects` is a new production in the grammar. This production allows one of the three possible combinations of effects specifiers between `get` and `{`, while enforcing an order between `async` and `throws` that mirrors the existing one on functions.  Additionally, one can declare (but not define) an effectful property (such as for a protocol) by adding the effect keywords following the `get`, as specified by this grammar:

```
getter-setter-keyword-block → "{" getter-keyword-clause setter-keyword-clause? "}"
getter-setter-keyword-block → "{" setter-keyword-clause getter-keyword-clause "}"
getter-keyword-clause → attributes? mutation-modifier? "get" getter-effects?
```

For example, one can write:

```swift
protocol Account {
  associatedtype Transaction

  var lastTransaction: Transaction { get async throws }

  subscript(_ day: Date) -> [Transaction] { get async }
}
```

to enforce that a type conforming to `Account` provides property and subscript witnesses that have the same or fewer effects than what is allowed by the protocol.

The interpretation of an effectful property definition is straightforward: the `code-block` appearing in such a `get`-ter definition will be allowed to exhibit the effects specified, i.e., throwing and/or suspending such that `await` and `try` expressions are allowed in that `code-block`. Furthermore, expressions that evaluate to an access of the property or subscript will be treated as having the effects that are declared on that property. One can think of such expressions as a simple desugaring to a method call on the object. It is always possible to determine whether a property has such effects, because the declaration of the property is always known statically. Thus, it is a static error to omit the appropriate `await`, `try`, *etc*.


#### Protocol conformance

In order for a type to conform to a protocol containing effectful properties, the type must contain a property (or subscript) that exhibits *the same or fewer effects than the protocol specifies* for that requirement. This rule mirrors how conformance checking happens for functions with effects: a witness can be missing an effect, but it cannot exhibit an effect that is not accounted for by the requirement. Here is a well-typed example without any superfluous `await`s or `try`s that follows this rule:

```swift
protocol P {
  var someProp: Int { get async throws }
}

class NoEffects: P { var someProp: Int { get { 1 } } }

class JustAsync: P { var someProp: Int { get async { 2 } } }

struct JustThrows: P { var someProp: Int { get throws { 3 } } }

struct Everything: P { var someProp: Int { get async throws { 4 } } }

func exampleExpressions() async throws {
  let _ = NoEffects().someProp
  let _ = try! await (NoEffects() as P).someProp

  let _ = await JustAsync().someProp
  let _ = try! await (JustAsync() as P).someProp

  let _ = try! JustThrows().someProp
  let _ = try! await (JustThrows() as P).someProp

  let _ = try! await Everything().someProp
  let _ = try! await (Everything() as P).someProp
}
```

Formally speaking, let us consider a getter `G` to have a set of effects `effects(G)` associated with it. This proposal adds one additional rule to conformance checking: if a getter definition `W` is said to satisfy the requirements of a protocol's getter declaration `R`, then `effects(W)` is a subset of `effects(R)`.

#### Class Inheritance

Effectful properties and subscripts can be inherited from a base class, and follow the usual visibility rules. The key difference is that, to override an inherited effectful property (or subscript) from the base class, *the subclass's property must have the same or fewer effects than the property being overridden*. This rule is a natural consequence of the subtyping relation for classes, where the base class must account for all of the effects that its subclasses may exhibit. In essence, this rule is the same as the one for protocol conformance.

#### Objective-C bridging

Some API designers may want to take advantage of Swift's effectful properties by having an Objective-C method imported as a property. Objective-C methods are normally imported as Swift methods, so their import as an effectful Swift property will be controlled through an opt-in annotation. This avoids any source compatibility issues for imported declarations.

Due to the read-only restriction on Swift properties, and the fact that a large number of failable Objective-C methods are already imported as `throws` methods in Swift, support for Objective-C bridging in this proposal is scoped for the Swift concurrency features. Importing as an effectful subscript is not included in this proposal. Furthermore, exporting effectful properties to Objective-C as methods are left to future work.

To import an Objective-C method as a Swift effectful property, the method must be compatible with the import rules for `async` Swift methods, as described by [SE-0297](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0297-concurrency-objc.md). An annotation changes this import behavior to produce an effectful Swift computed property, instead of an `async` Swift method. The original ObjC method is still imported as a normal Swift method, alongside the property.

To summarize, an Objective-C method that meets the following requirements:
  1. The method takes exactly one argument, a completion handler, as recognized by 
  [SE-0297](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0297-concurrency-objc.md).
  2. The method returns `void`. 
  3. The method is annotated with `__attribute__((swift_async_name("getter:myProp()")))`. Note the use of `getter:` to specify that it should be a property instead of a method.
  
will be imported as an effectful read-only Swift property named `myProp`, instead of a Swift `async` (and possibly also `throws`) method. The following are Objective-C method examples from the SDK that have been annotated for import as an effectful Swift property:

```objc
// from Safari Services
@interface SFSafariTab: NSObject
- (void)getPagesWithCompletionHandler:(void (^)(NSArray<SFSafariPage *> *pages))completionHandler
__attribute__((swift_async_name("getter:pages()")));
// ...
@end

// from Exposure Notification
@interface ENManager: NSObject
- (void)getUserTraveledWithCompletionHandler:(void (^)(BOOL traveled, NSError *error))completionHandler
__attribute__((swift_async_name("getter:userTraveled()")));;
// ...
@end
```

which would be imported into Swift as:

```swift
class SFSafariTab: NSObject {
  var pages: [SFSafariPage] {
    get async { /* ... */ }
  }
  // ...
}

class ENManager: NSObject {
  var userTraveled: Bool {
    get async throws { /* ... */ }
  }
}
```

## Source compatibility

The proposed syntactic changes are such that if they appeared in previous versions of the language, they would have been rejected as an error by the parser.

## Effect on ABI stability

This proposal is additive and limits its scope intentionally to avoid breaking ABI stability.

## Effect on API resilience

As an additive feature, this will not affect API resilience. But, existing APIs that adopt effectful read-only properties will break backwards compatibility, because users of the API will be required to wrap accesses of the property with `await` and/or `try`.

## Extensions considered

In this section, we will discuss extensions and additions to this proposal, and why they are not included in the proposed design above.

#### Effectful settable properties

Defining the interactions between async and/or throwing writable properties and features such as:

1. `inout`
2. `_modify`
3. property observers, i.e., `didSet`, `willSet`
4. property wrappers
5. writable subscripts

is a large project that requires a significant implementation effort. This proposal is primarily motivated by allowing the use of Swift concurrency features in computed properties and subscripts. The proposed design for effectful read-only properties is small and straightforward to implement, while still providing a notable benefit to real-world programs.

#### Key Paths

A [key-path expression](https://docs.swift.org/swift-book/ReferenceManual/Expressions.html) is syntactic sugar for instances of the `KeyPath` class and its type-erased siblings.  The introduction of effectful properties would require changes to the synthesis of `subscript(keyPath:)` for each type. It is also likely to require restrictions on type-erasure for key-paths that can access effectful properties.

For example, because we do not allow for function overloading based only on differences in effects, some sort of mechanism like `rethrows` and an equivalent version for `async` (such as a "reasync") would be required on `subscript(keyPath:)` as a starting-point.  While a key-path literal can be [automatically treated as a function](0249-key-path-literal-function-expressions.md), a general `KeyPath` value is not a function, so it cannot carry effects in its type. This causes problems when trying to make, for example, a `rethrows` version of `subscript(keyPath:)` work.

We could also introduce additional kinds of key-paths that have various capabilities, like the existing `WritableKeyPath` and `ReferenceWritableKeyPath`. Then, we could synthesize versions of `subscript` with the right effects specifiers on it, for example, `subscript<T: ThrowingKeyPath>(keyPath: T) throws`. This would require `KeyPath` kinds for all three new combinations of effects beyond "no effects".

So, a non-trivial restructuring of the type system, or significant extensions to the `KeyPath` API, would be required to make key-paths work for effectful properties. Thus, for now, we will disallow accesses to effectful properties via key-paths.  There already exist restrictions on key-paths to mutable properties based on the instance type (e.g., `WritableKeyPath`), so it would not be unusual to disallow key-paths to effectful properties.

## Alternatives considered

In this section, alternative designs for this proposal are discussed.

<!-- Describe alternative approaches to addressing the same problem, and
why you chose this approach instead. -->

#### Effects Specifiers Positions

There are a number of places where the effects specifiers be placed:

```
<A> var prop: Type <B> {
  <C> get <D> { }
}
```

Where `<X>` refers to "position X" in the example. Consider each of these positions:

* **Position A** is primarily used by access modifiers like `private(set)` or declaration modifiers like `override`. The more effect-like `mutating`/`nonmutating` is only allowed in Position C, which precedes the accessor declaration, just like a method within a struct. This position was not chosen because phrases like `override async throws var prop` or `async throws override var prop` do not read particularly well.
* **Position B** does not make much sense, because effects are only carried as part of a function type, not other types. So, it would be very confusing, leading people to think `Int async throws` is a type, when that is not. Introducing a new kind of punctuation here was ruled out because there are alteratives to this position.
* **Position C** is not bad; it's only occupied by `mutating`/`nonmutating`, but placing effects specifiers here is not consistent with the positioning for functions, which is *after* the subject. Since Position D is available, it makes more sense to use that instead of Position C.
* **Position D** is the one ultimately chosen for this proposal. It is an unused place in the grammar, places the effects on the accessor and not the variable or its type. Plus, it is consistent with where effects go on a function declaration, after the subject: `get throws` and `get async throws`, where get is the subject. Another benefit is that it is away from the variable, so it prevents confusion between the accessor's effects and the effects of a function being returned:

```swift
var predicate: (Int) async throws -> Bool {
  get throws { /* ... */ }
}
```

The access of `predicate` may throw, but if it doesn't, it results in a function that is async throws.

There was also a desire to take advantage of the implicit-getter shorthand for the above:

```swift
var predicate: (Int) async throws -> Bool { /* ... */ }
```

but there is no good place for effects specifiers here. Because this syntax is a short-hand / syntactic sugar, which necessarily has to trade some of its flexibility for conciseness. So, it was decided that it's OK to not allow effectful properties to be declared using this short-hand. The full syntax for computed properties explicitlys defines its accessors, and thus can declare effects on them.

##### Subscripts
The major difference for subscripts is the method-like header syntax and support for the implicit-getter short-hand, which combined make it look like a method:

```
class C {
  subscript(_ : InType) <E> -> RetType { /* ... */ }
}
```

**Position E** in the above is a tempting place for effects specifiers for a subscript, but subscripts are not methods. They cannot be accessed a first-class function value with `c.subscript`, nor called with `c.subscript(0)`; they use an indexing syntax `c[0]`. Methods cannot be assigned to, but subscript index expressions can be. Thus, they are closer to properties that can accept an argument.

Much like the short-hand for get-only properties, trying to find a position for effects specifiers on the short-hand form of get-only subscripts (whether its Position E or otherwise) will trap this feature in a corner if writable subscripts can support effects in the future. Why? Position E is a logically valid spot in the full-syntax *and* the short-hand syntax. Creating an inconsistency between the two would be bad. Then, using Position E + the full syntax creates an opportunity for confusion in situations like this:

```swift
subscript(_ i : Int) throws -> Bool {
  get async { }
  set { }
}
```

Here, the only logical interpretation is that `set` is throws and `get` is async throws. The programmer needs to look in multiple places to add up the effects in their head when trying to determine what effects are allowed in an accessor. This may not seem so bad in this short example, but consider having to skip over a large `get` accessor definition to learn about *all* of the effects the set accessor is allowed to have for this subscript, when you do *not* need to do that for a computed property.

So, Position D was chosen as the one true place where you can look to see whether there are effects for that type of accessor, both for subscripts and computed properties.

#### Miscellany

The `rethrows` specifier is excluded from this proposal because one cannot pass a closure (or any other explicit value) during a property `get` operation.

The `async`/`await` feature is purpose-built for enabling asynchronous programming, so no consideration is given for alternative solutions that do not rely on that feature for asynchronous properties. The same reasoning applies to `throws`/`try`.

## Acknowledgments

Thanks to Doug Gregor and John McCall for their guidance while crafting this proposal. The feasibility and design choices for this proposal were influenced by [Becca Royal-Gordon's proposal for throwing property accessors](https://github.com/beccadax/swift-evolution/blob/72c55f33b94749e22637bd8277661599e9cd8007/proposals/0000-throwing-properties.md) and recent discussions with her.
