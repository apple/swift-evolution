# Remove Actor Isolation Inference caused by Property Wrappers

* Proposal: [SE-0401](0401-remove-property-wrapper-isolation.md)
* Authors: [BJ Homer](https://github.com/bjhomer)
* Review Manager: [Holly Borla](https://github.com/hborla)
* Status: **Implemented (Swift 5.9)**
* Implementation: [apple/swift#63884](https://github.com/apple/swift/pull/63884)
* Upcoming Feature Flag: `DisableOutwardActorInference`
* Review: ([pitch](https://forums.swift.org/t/pitch-stop-inferring-actor-isolation-based-on-property-wrapper-usage/63262)) ([review](https://forums.swift.org/t/se-0401-remove-actor-isolation-inference-caused-by-property-wrappers/65618)) ([acceptance](https://forums.swift.org/t/accepted-with-modifications-se-0401-remove-actor-isolation-inference-caused-by-property-wrappers/66241))

## Introduction

[SE-0316: Global Actors](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0316-global-actors.md) introduced annotations like `@MainActor` to isolate a type, function, or property to a particular global actor. It also introduced various rules for how that global actor isolation could be inferred. One of those rules was:

> Declarations that are not explicitly annotated with either a global actor or `nonisolated` can infer global actor isolation from several different places:
>
> [...]
> 
> - A struct or class containing a wrapped instance property with a global actor-qualified wrappedValue infers actor isolation from that property wrapper:
> 
>   ```swift 
>   @propertyWrapper
>   struct UIUpdating<Wrapped> {
>     @MainActor var wrappedValue: Wrapped
>   }
> 
>   struct CounterView { // infers @MainActor from use of @UIUpdating
>     @UIUpdating var intValue: Int = 0
>   }
>   ```


This proposal advocates for **removing this inference rule** when compiling in the Swift 6 language mode. Given the example above, CounterView would no longer infer `@MainActor` isolation in Swift 6.

## Motivation

This particular inference rule is surprising and nonobvious to many users. Some developers have trouble understanding the Swift Concurrency model because it's not obvious to them when actor isolation applies. When something is inferred, it is not visible to the user, and that makes it harder to understand. This frequently arises when using the property wrappers introduced by Apple's SwiftUI framework, although it is not limited to that framework. For example:

### An example using SwiftUI

```swift
struct MyView: View {
  // Note that `StateObject` has a MainActor-isolated `wrappedValue`
  @StateObject private var model = Model()
  
  var body: some View {
    Text("Hello, \(model.name)")
      .onAppear { viewAppeared() }
  }

  // This function is inferred to be `@MainActor`
  func viewAppeared() {
    updateUI()
  }
}

@MainActor func updateUI() { /* do stuff here */ }
```

The above code compiles just fine. But if we change `@StateObject` to `@State`, we get an error:

```diff
-   @StateObject private var model = Model()
+   @State private var model = Model()
```
  
```swift  
  func viewAppeared() {
    // error: Call to main actor-isolated global function
    // 'updateUI()' in a synchronous nonisolated context
    updateUI()
  }
```

Changing `@StateObject var model` to `@State var model` caused `viewAppeared()` to stop compiling, even though that function didn't use `model` at all. It feels non-obvious that changing the declaration of one property should cause a _sibling_ function to stop compiling. In fact, we also changed the isolation of the entire `MyView` type by changing one property wrapper. 

### An example not using SwiftUI

This problem is not isolated to SwiftUI. For example:


```swift
// A property wrapper for use with our database library
@propertyWrapper
struct DBParameter<T> {
  @DatabaseActor public var wrappedValue: T
}

// Inferred `@DatabaseActor` isolation because of use of `@DBParameter`
struct DBConnection {
  @DBParameter private var connectionID: Int

  func executeQuery(_ query: String) -> [DBRow] { /* implementation here */ }
}


// In some other file...

@DatabaseActor
func fetchOrdersFromDatabase() async -> [Order] {
  let connection = DBConnection()

  // No 'await' needed here, because 'connection' is also isolated to `DatabaseActor`.
  connection.executeQuery("...")
}
```

Removing the property wrapper on `DBConnection.connectionID` would remove the inferred actor isolation of `DBConnection`, which would in turn cause `fetchOrdersFromDatabase` to fail to compile. **It's unprecedented in Swift that changes to a _private_ property should cause compilation errors in some entirely separate file**. Upward inference of actor isolation (from property wrappers to their containing type) means that we can no longer locally reason about the effects of even *private* properties within a type. Instead, we get "spooky action at a distance".
  
### Does this cause actual problems?

This behavior has caused quite a bit of confusion in the community. For example, see [this tweet](https://twitter.com/teilweise/status/1580105376913297409?s=61&t=hwuO4NDJK1aIxSntRwDuZw), [this blog post](https://oleb.net/2022/swiftui-task-mainactor/), and [this entire Swift Forums thread](https://forums.swift.org/t/reconsider-inference-of-global-actor-based-on-property-wrappers/60821). One particular callout comes from [this post](https://forums.swift.org/t/reconsider-inference-of-global-actor-based-on-property-wrappers/60821/6/), where this inference made it hard to adopt Swift Concurrency in some cases, because the actor isolation goes "viral" beyond the intended scope:

```swift
class MyContainer {
     let contained = Contained() // error: Call to main actor-isolated initializer 'init()' in a synchronous nonisolated context
}

class Contained {
    @OnMainThread var i = 1
}
```

The author created an `@OnMainThread` property wrapper, intended to declare that a particular property was isolated to the main thread. However, they cannot enforce that by using `@MainActor` within the property wrapper, because doing so causes the entire contained type to become unexpectedly isolated.

The [original motivation](https://forums.swift.org/t/se-0401-remove-actor-isolation-inference-caused-by-property-wrappers/65618/10) for this inference rule was to reduce the annotation burden when using property wrappers like SwiftUI's `@ObservedObject`. But it's not clear it actually makes anything significantly easier; it only saves us from writing a single annotation on the type, and the loss of that annotation introduces violations of the [principle of least surprise](https://en.wikipedia.org/wiki/Principle_of_least_astonishment).


## Proposed solution

The proposal is simple: In the Swift 6 language mode, property wrappers used within a type will not affect the type's actor isolation. We simply disable this inference step entirely.

In the Swift 5 language mode, isolation will continue to be inferred as it currently is. The new behavior can be requested using the **`-enable-upcoming-feature DisableOutwardActorInference`** compiler flag.

## Detailed design

[`ActorIsolationRequest.getIsolationFromWrappers()`](https://github.com/apple/swift/blob/85d59d2e55e5e063c552c15f12a8abe933d8438a/lib/Sema/TypeCheckConcurrency.cpp#L3618) implements the actor isolation inference described in this proposal. That function will be adjusted to avoid producing any inference when running in the Swift 6 language mode or when the compiler flag described above is passed.

## Source compatibility

This change _does_ introduce potential for source incompatibility, because there may be code which was relying on the inferred actor isolation. That code can be explicitly annotated with the desired global actor in a source-compatible way right now. For example, if a type is currently inferred to have `@MainActor` isolation, you could explicitly declare that isolation on the type right now to avoid source compatibility. (See note about warnings in Alternatives Considered.)

There may be cases where the source incompatibility could be mitigated by library authors in a source-compatible way. For example, if Apple chose to make SwiftUI's `View` protocol `@MainActor`-isolated, then all conforming types would consistently be isolated to the Main Actor, rather than being inconsistently isolated based on the usage of certain property wrappers. This proposal only notes that this mitigation may be _possible_, but does not make any recommendation as to whether that is necessary.

### Source compatibility evaluation

In an effort to determine the practical impact of this change, I used a macOS toolchain containing these changes and evaluated various open-source Swift projects (from the Swift Source Compatibility Library and elsewhere). I found no instances of actual source incompatibility as a result of the proposed changes. Most open-source projects are libraries that use no property wrappers at all, but I tried to specifically seek out a few projects that *do* use property wrappers and may be affected by this change. The results are as follows:

Project | Outcome | Notes
---|---|---
[ACNHBrowserUI](https://github.com/Dimillian/ACHNBrowserUI) | Fully Compatible | Uses SwiftUI property wrappers
[AlamoFire](https://github.com/Alamofire/Alamofire) | Fully Compatible | Uses custom property wrappers, but none are actor isolated
[Day One (Mac)](https://dayoneapp.com) | Fully Compatible | Uses SwiftUI property wrappers. (Not open source)
[Eureka](https://github.com/xmartlabs/Eureka) | Fully Compatible | Does not use property wrappers at all
[NetNewsWire](https://github.com/Ranchero-Software/NetNewsWire) | Fully Compatible | Uses SwiftUI property wrappers
[swift-nio](https://github.com/apple/swift-nio) | Fully Compatible | Does not use property wrappers at all
[SwiftyJSON](https://github.com/SwiftyJSON/SwiftyJSON) | Fully Compatible | Does not use property wrappers at all
[XcodesApp](https://github.com/RobotsAndPencils/XcodesApp) | Fully Compatible | Uses SwiftUI property wrappers

All of the above had a `Swift Concurrency Checking` setting of **Minimal** by default. When I changed the concurrency checking level to **Targeted**, all of the above continued to compile with no errors, both with and without the proposed changes.

When I changed the concurrency checking level to **Complete**, most of the above projects had compilation errors, _even without the changes proposed here_. The changes proposed here likely contributed a few _additional_ errors under "Complete" checking, but they did not break source compatibility in projects that would have otherwise been source compatible.

## Effect on ABI stability

This change is ABI stable, as the actor isolation of a type is not reflected in its runtime calling convention in any way.

## Effect on API resilience

This proposal has no effect on API resilience.

## Alternatives considered

#### Warn about Property Wrapper-based inference in Swift 5

In certain cases, we produce a warning that code will become invalid in a future Swift release. (For example, this has been done with the planned changes to Swift Concurrency in Swift 6.) I considered adding a warning to the Swift 5 language mode along these lines:

```swift

// ⚠️ Warning: `MyView` is inferred to use '@MainActor' isolation because
//    it uses `@StateObject`. This inference will go away in Swift 6.
//
// Add `@MainActor` to the type to silence this warning.

struct MyView: View {
  @StateObject private var model = Model()

  var body: some View {
    Text("Hello")
  }
}
```

However, I found two problems:

1. This would produce a _lot_ of warnings, even in code that will not break under the Swift 6 language mode.

2. There's no way to silence this warning _without_ isolating the type. If I actually _didn't_ want the type to be isolated, there's no way to express that. You can't declare a non-isolated type:

```swift
nonisolated   // 🛑 Error:  'nonisolated' modifier cannot be applied to this declaration
struct MyView: View {
  /* ... */
}
```

Given that users cannot silence the warning in a way that matches the new Swift 6 behavior, it seems inappropriate to produce a warning here.


## Acknowledgments

Thanks to Dave DeLong for reviewing this proposal, and to the many members of the Swift community who have engaged in discussion on this topic.
