# Observation

* Proposal: [SE-0395](0395-observability.md)
* Authors: [Philippe Hausler](https://github.com/phausler), [Nate Cook](https://github.com/natecook1000)
* Review Manager: [Ben Cohen](https://github.com/airspeedswift)
* Status: **Implemented (Swift 5.9)**
* Review: ([pitch](https://forums.swift.org/t/pitch-observation-revised/63757)), ([first review](https://forums.swift.org/t/se-0395-observability/64342/)), ([second review](https://forums.swift.org/t/second-review-se-0395-observability/65261/)), ([acceptance](https://forums.swift.org/t/accepted-with-revision-se-0395-observability/66760))

#### Changes

* Version 1: [Initial pitch](https://forums.swift.org/t/pitch-observation/62051)
* Version 2: Previously Observation registered observers directly to `Observable`, the new approach registers observers to an `Observable` via a `ObservationTransactionModel`. These models control the "edge" of where the change is emitted. They are the responsible component for notifying the observers of events. This allows the observers to focus on just the event and not worry about "leading" or "trailing" (will/did) "edges" of the signal. Additionally the pitch was shifted from the type wrapper feature over to the more appropriate macro features.
* Version 3: The `Observer` protocol and `addObserver(_:)` method are gone in favor of providing async sequences of changes and transactions.
* Version 4: In order to support observation for subclasses and to provide space to address design question around the asynchronous `values(for:)` and `changes(for:)` methods, the proposal now focuses on an `Observable` marker protocol and the `withTracking(_:changes:)` function.

#### Suggested Reading

* [Expression Macros](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0382-expression-macros.md)
* [Attached Macros](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0389-attached-macros.md)

## Introduction

Making responsive apps often requires the ability to update the presentation when underlying data changes. The _observer pattern_ allows a subject to maintain a list of observers and notify them of specific or general state changes. This has the advantages of not directly coupling objects together and allowing implicit distribution of updates across potential multiple observers. An observable object needs no specific information about its observers.

This design pattern is a well-traveled path by many languages, and Swift has an opportunity to provide a robust, type-safe, and performant implementation. This proposal defines what an observable reference is, what an observer needs to conform to, and the connection between a type and its observers.

## Motivation

There are already a few mechanisms for observation in Swift. These include key-value observing (KVO) and `ObservableObject`, but each of those have limitations. KVO can only be used with `NSObject` descendants, and `ObservableObject` requires using Combine, which is restricted to Darwin platforms and does not use current Swift concurrency features. By taking experience from those existing systems, we can build a more generally useful feature that applies to all Swift reference types, not just those that inherit from `NSObject`, and have it work cross-platform with the advantages from language features like `async`/`await`.

The existing systems get a number of behaviors and characteristics right. However, there are a number of areas that can provide a better balance of safety, performance, and expressiveness. For example, grouping dependent changes into an independent transaction is a common task, but this is complex when using Combine and unsupported when using KVO. In practice, observers want access to transactions, with the ability to specify how transactions are interpreted.

Annotations clarify what is observable, but can also be cumbersome. For example, Combine requires not just that a type conform to `ObservableObject`, but also requires each property that is being observed to be marked as `@Published`. Furthermore, computed properties cannot be directly observed. In reality, having non-observed fields in a type that is observable is uncommon.

Throughout this document, references to both KVO and Combine will illustrate what capabilities are benefits and can be incorporated into the new approach, and what drawbacks are possible to solve in a more robust manner.

### Prior Art

#### KVO

Key-value observing has served the Cocoa/Objective-C programming model well, but is limited to class hierarchies that inherit from `NSObject`. The APIs only offer the intercepting of events, meaning that the notification of changes is between the `willSet` and `didSet` events. KVO has great flexibility with granularity of events, but lacks in composability. KVO observers must also inherit from `NSObject`, and rely on the Objective-C runtime to track the changes that occur. Even though the interface for KVO has been updated to utilize the more modern Swift strongly-typed key paths, under the hood its events are still stringly typed.

#### Combine

Combine's `ObservableObject` produces changes at the beginning of a change event, so all values are delivered before the new value is set. While this serves SwiftUI well, it is restrictive for non-SwiftUI usage and can be surprising to developers first encountering that behavior. `ObservableObject` also requires all observed properties to be marked as `@Published` to interact with change events. In most cases, this requirement is applied to every single property and becomes redundant to the developer; folks writing an `ObservableObject` conforming type must repeatedly (with little to no true gained clarity) annotate each property. In the end, this results in meaning fatigue of what is or isn't a participating item.

## Proposed solution

A formalized observer pattern needs to support the following capabilities:

* Marking a type as observable
* Tracking changes within an instance of an observable type
* Observing and utilizing those changes from somewhere else

In addition, the design and implementation should meet these criteria:

* Observable types are easy to annotate (without fatigue of meaning)
* Access control should be respected
* Adopting the features for observability should require minimal effort to get started
* Using advanced features should progressively disclose to more complex systems
* Observation should be able to handle more than one observed member at once
* Observation should be able to work with computed properties that reference other properties
* Observation should be able to work with computed properties that store their values in external storage
* Integration of observation should work in transactions of graphs and not just singular objects

We propose a new standard library module named `Observation` that includes the required functionality to implement such a pattern.

Primarily, a type can declare itself as observable simply by using the `@Observable` macro annotation:

```swift
@Observable class Car {
    var name: String
    var awards: [Award]
}
```

The `@Observable` macro implements conformance to the `Observable` marker protocol and tracking for each stored property. Unlike `ObservableObject` and `@Published`, the properties of an `@Observable` type do not need to be individually marked as observable. Instead, all stored properties are implicitly observable.

The `Observation` module also provides the top-level function `withObservationTracking`, which detects accesses to tracked properties within a specific scope. Once those properties are identified, any changes to the tracked properties triggers a call to the provided `onChange` closure.

```swift
let cars: [Car] = ...

@MainActor
func renderCars() {
    withObservationTracking {
        for car in cars {
            print(car.name)
        }
    } onChange: {
        Task { @MainActor in
            renderCars()
        }
    }
}
```

In the example above, the `render` function accesses each car's `name` property. When any of the cars change `name`, the `onChange` closure is then called on the first change. However, if a car has an award added, the `onChange` call won't happen. This design supports uses that require implicit observation tracking, ensuring that updates are only performed in response to relevant changes.

## Detailed Design

The `Observable` protocol, `@Observable` macro, and a handful of supporting types comprise the `Observation` module. As described below, this design allows adopters to use a straightforward syntax for simple cases, while allowing full control over the details the implementation when necessary.

### `Observable` protocol

Observable types conform to the `Observable` marker protocol. While the `Observable` protocol doesn't have formal requirements, it includes a semantic requirement that conforming types must implement tracking for each stored property using an `ObservationRegistrar`. Most types can meet that requirement simply by using the `@Observable` macro:

```swift
@Observable public final class MyObject {
    public var someProperty = ""
    public var someOtherProperty = 0
    fileprivate var somePrivateProperty = 1
}
```

### `@Observable` Macro

In order to make implementation as simple as possible, the `@Observable` macro automatically synthesizes conformance to the `Observable` protocol, transforming annotated types into a type that can be observed. When fully expanded, the `@Observable` macro does the following:

- declares conformance to the `Observable` protocol,
- adds a property for the registrar,
- and adds internal helper methods for tracking accesses and mutations.

Additionally, for each stored property, the macro:

- annotates each stored property with the `@ObservationTracked` macro,
- converts each stored property to a computed property,
- and adds an underscored, `@ObservationIgnored` version of each stored property.

Since all of the code generated by the macro could be manually written, developers can write or customize their own implementation when they need more fine-grained control.

As an example of the `@Observable` macro expansion, consider the following `Model` type:

```swift
@Observable class Model {
    var order: Order?
    var account: Account?
  
    var alternateIconsUnlocked: Bool = false
    var allRecipesUnlocked: Bool = false
  
    func purchase(alternateIcons: Bool, allRecipes: Bool) {
        alternateIconsUnlocked = alternateIcons
        allRecipesUnlocked = allRecipes
    }
}
```

Expanding the `@Observable` macro, as well as the generated macros, results in the following declaration:

```swift
class Model: Observable {
    internal let _$observationRegistrar = ObservationRegistrar<Model>()
  
    internal func access<Member>(
        keyPath: KeyPath<Model, Member>
    ) {
        _$observationRegistrar.access(self, keyPath: keyPath)
    }

    internal func withMutation<Member, T>(
        keyPath: KeyPath<Model, Member>,
        _ mutation: () throws -> T
    ) rethrows -> T {
        try _$observationRegistrar.withMutation(of: self, keyPath: keyPath, mutation)
    }
          
    var order: Order? {
        get { 
            self.access(keyPath: \.order)
            return _order
        }
        set {
            self.withMutation(keyPath: \.order) {
                _order = newValue
            }
        }
    }
  
    var account: Account? {
        get { 
            self.access(keyPath: \.account)
            return _account
        }
        set {
            self.withMutation(keyPath: \.account) {
                _account = newValue
            }
        }
    }

    var alternateIconsUnlocked: Bool {
        get { 
            self.access(keyPath: \.alternateIconsUnlocked)
            return _alternateIconsUnlocked
        }
        set {
            self.withMutation(keyPath: \.alternateIconsUnlocked) {
                _alternateIconsUnlocked = newValue
            }
        }
    }

    var allRecipesUnlocked: Bool {
        get { 
            self.access(keyPath: \.allRecipesUnlocked)
            return _allRecipesUnlocked
        }
        set {
            self.withMutation(keyPath: \.allRecipesUnlocked) {
                _allRecipesUnlocked = newValue
            }
        }
    }

    var _order: Order?
    var _account: Account?
  
    var _alternateIconsUnlocked: Bool = false
    var _allRecipesUnlocked: Bool = false
}
```

### `@ObservationTracked` and `@ObservationIgnored` macros

The `Observation` module includes two additional macros that can annotate properties of observable types. The `@ObservationTracked` macro is added to stored properties by the `@Observable` macro expansion, and, when expanded, converts a stored property to a computed one with access and mutation tracking. Developers generally won't use `@ObservationTracked` themselves.

The `@ObservationIgnored` macro, on the other hand, doesn't add anything to a source file when expanded. Instead, it acts as a marker for properties that shouldn't be tracked. The `@Observable` macro expansion adds `@ObservationIgnored` to the underscored stored properties it creates. Developers can also apply `@ObservationIgnored` to stored properties that shouldn't be included in observation tracking.

### Computed properties

Computed properties that derive their values from stored properties are automatically tracked due to their reliance on tracked properties. Computed properties that source their value from remote storage or via indirection, however, must manually add tracking using the generated `access(keyPath:)` and `withMutation(keyPath:)` methods.

For example, consider the `AtomicModel` in the following code sample. `AtomicModel` stores a score in an `AtomicInt`, with a computed property providing an `Int` interface. The atomic property is annotated with the `@ObservationIgnored` macro because it isn't useful to track the constant value for observation. For the computed `score` property, which is the public interface of the type, the getter and setter include manually-written calls to track accesses and mutations.

```swift
@Observable
public class AtomicModel {
    @ObservationIgnored
    fileprivate let _scoreStorage = AtomicInt(initialValue: 0)

    public var score: Int {
        get {
            self.access(keyPath: \.score)
            return _scoreStorage.value
        }
        set {
            self.withMutation(keyPath: \.score) {
                _scoreStorage.value = newValue
            }
        }
    }
}
```

### `willSet`/`didSet`

Observation is supported for properties with `willSet` and `didSet` property observers. For example, the `@Observable` macro on the `PropertyExample` type here:

```swift
@Observable class PropertyExample {
    var a = 0 {
        willSet { print("will set triggered") }
        didSet { print("did set triggered") }
    }
    var b = 0
    var c = ""
}
```

...transforms the `a` property as follows, preserving the `willSet` and `didSet` behavior:

```swift
var a: Int {
    get {
        self.access(keyPath: \.a)
        return _a 
    }
    set {
        self.withMutation(keyPath: \.a) {
            _a = newValue
        }
    }
}

var _a = 0 {
    willSet { print("will set triggered") }
    didSet { print("did set triggered") }
}
```

### Initializers

Because observable types generally use the implicitly generated initializers, the `@Observable` macro requires that all stored properties have a default value. This guarantees definitive initialization, so that additional initializers can be added to observable types in an extension.

The default value requirement could be relaxed in a future version; see the Future Directions section for more.

### Subclasses

Developers can create `Observable` subclasses of either observable or non-observable types. Only the properties of a type that implements the `Observable` tracking requirements will be observed. That is, when working with an observable subclass of a non-observable type, the superclass's stored properties will not be tracked under observation.

### `withObservationTracking(_:onChange:)`

In order to provide automatically scoped observation, the `ObservationModule` provides a function to capture accesses to properties within a given scope, and then call out upon the first change to any of those properties. This can be used by user interface libraries, such as SwiftUI, to provide updates to the specific properties which are accessed within a particular scope, limiting interface updates or renders to only the relevant changes. For more detail, see the SDK Impact section below.

```swift
public func withObservationTracking<T>(
    _ apply: () -> T, 
    onChange: @autoclosure () -> @Sendable () -> Void
) -> T
```

The `withObservationTracking` function takes two closures. Any access to a tracked property within the `apply` closure will flag the property; any change to a flagged property will trigger a call to the `onChange` closure.

Accesses are recognized for:
- tracked properties on observable objects
- tracked properties of properties that have observable type
- properties that are accessed via computed property accesses

For example, this `Person` class has multiple tracked properties, some of which are internal:

```swift
@Observable public class Person: Sendable {    
    internal var firstName = ""
    internal var lastName = ""
    public var age: Int?
    
    public var fullName: String {
        "\(firstName) \(lastName)"
    }
    
    public var friends: [Person] = []
}
```

Accessing the `fullName` and `friends` properties will result in the `firstName`, `lastName`, and `friends` properties being tracked for changes:

```swift
@MainActor
func renderPerson(_ person: Person) {
    withObservationTracking {
        print("\(person.fullName) has \(person.friends.count) friends.")
    } onChange: {
        Task { @MainActor in
            renderPerson(person)
        }
    }
}
```

Whenever the person's `firstName` or `lastName` properties are updated, the `onChange` closure will be called, even though those properties are internal, since their accesses are linked to a public computed property. Mutations to the `friends` array will also cause a call to `onChange`, though changes to individual members of the array are not tracked.

### `ObservationRegistrar`

`ObservationRegistrar` is the required storage for tracking accesses and mutations. The `@Observable` macro synthesizes a registrar to handle these mechanisms as a generalized feature. By default, the registrar is thread safe and must be as `Sendable` as containers could potentially be; therefore it must be designed to handle independent isolation for all actions.

```swift
public struct ObservationRegistrar: Sendable {
    public init()
      
    public func access<Subject: Observable, Member>(
        _ subject: Subject,
        keyPath: KeyPath<Subject, Member>
    )
      
    public func willSet<Subject: Observable, Member>(
        _ subject: Subject,
        keyPath: KeyPath<Subject, Member>
    )
      
    public func didSet<Subject: Observable, Member>(
        _ subject: Subject,
        keyPath: KeyPath<Subject, Member>
    )
      
    public func withMutation<Subject: Observable, Member, T>(
        of subject: Subject, 
        keyPath: KeyPath<Subject, Member>, 
        _ mutation: () throws -> T
    ) rethrows -> T
}
```

The `access` and `withMutation` methods identify transactional accesses. These methods register access to the underlying tracking system for access and identify mutations to the transactions registered for observers.

## SDK Impact (a preview of SwiftUI interaction)

When using the existing `ObservableObject`-based observation, there are a number of edge cases that can be surprising unless developers have an in-depth understanding of SwiftUI. Formalizing observation can make these edge cases considerably more approachable by reducing the complexity of the different systems needed to be understood.

The following is adapted from the [Fruta sample app](https://developer.apple.com/documentation/swiftui/fruta_building_a_feature-rich_app_with_swiftui), modified for clarity:

```swift
class Model: ObservableObject {
    @Published var order: Order?
    @Published var account: Account?
    
    var hasAccount: Bool {
        return userCredential != nil && account != nil
    }
    
    @Published var favoriteSmoothieIDs = Set<Smoothie.ID>()
    @Published var selectedSmoothieID: Smoothie.ID?
    
    @Published var searchString = ""
    
    @Published var isApplePayEnabled = true
    @Published var allRecipesUnlocked = false
    @Published var unlockAllRecipesProduct: Product?
}

struct SmoothieList: View {
    var smoothies: [Smoothie]
    @ObservedObject var model: Model
    
    var listedSmoothies: [Smoothie] {
        smoothies
            .filter { $0.matches(model.searchString) }
            .sorted(by: { $0.title.localizedCompare($1.title) == .orderedAscending })
    }
    
    var body: some View {
        List(listedSmoothies) { smoothie in
            ...
        }
    }
} 
```

The `@Published` attribute identifies each field that participates in changes in the object, but it does not provide any differentiation or distinction as to the source of changes. This unfortunately results in additional layouts, rendering, and updates.

The proposed API not only reduces the `@Published` repetition, but also simplifies the SwiftUI view code too! With the proposed `@Observable` macro, the previous example can instead be written as the following:

```swift
@Observable class Model {
    var order: Order?
    var account: Account?
    
    var hasAccount: Bool {
        userCredential != nil && account != nil
    }
    
    var favoriteSmoothieIDs: Set<Smoothie.ID> = []
    var selectedSmoothieID: Smoothie.ID?
    
    var searchString = ""
    
    var isApplePayEnabled = true
    var allRecipesUnlocked = false
    var unlockAllRecipesProduct: Product?
}

struct SmoothieList: View {
    var smoothies: [Smoothie]
    var model: Model
    
    var listedSmoothies: [Smoothie] {
        smoothies
            .filter { $0.matches(model.searchString) }
            .sorted(by: { $0.title.localizedCompare($1.title) == .orderedAscending })
    }
    
    var body: some View {
        List(listedSmoothies) { smoothie in
            ...
        }
    }
} 
```

There are some other interesting differences that follow from using the proposed observation system. For example, tracking observation of access within a view can be applied to an array, an optional, or even a custom type. This opens up new and interesting ways that developers can utilize SwiftUI more easily.

This is a potential future direction for SwiftUI, but is not part of this proposal.

## Source compatibility

This proposal is additive and provides no impact to existing source code.

## Effect on ABI stability

This proposal is additive and no impact is made upon existing ABI stability. This does have implication to the marking of inline to functions and back-porting of this feature. In the cases where it is determined to be performance critical to the distribution of change events the methods will be marked as inlineable. 

Changing a type from not observable to `@Observable` has the same ABI impact as changing a property from stored to computed (which is not ABI breaking). Removing `@Observable` not only transitions from computed to stored properties but also removes a conformance (which is ABI breaking).

## Effect on API resilience

This proposal is additive and no impact is made upon existing API resilience. The types that adopt `@Observable` cannot remove it without breaking API contract.

## Location of API

This API will be housed in a module that is part of the Swift language but outside of the standard library. To use this module `import Observation` must be used (and provisionally using the preview `import _Observation`).

## Future Directions

The requirement that all stored properties of an observable type have initial values could be relaxed in the future, if language features are added that would support that. For example, property wrappers have a feature that allows their underlying wrapped value to be provided in an initializer rather than as a default value. Generalizing that feature to all properties could allow the `@Observable` macro to enable a more typical initialization implementation.

Another area of focus for future enhancements is support for observable `actor` types. This would require specific handling for key paths that currently does not exist for actors.

An earlier version of this proposal included asynchronous sequences of coalesced transactions and individual property changes, named `values(for:)` and `changes(for:)`. Similar invariant-preserving asynchronous sequences could be added in a future proposal.

## Alternatives considered

An earlier consideration instead of defining transactions used direct will/did events to the observer. This, albeit being more direct, promoted mechanisms that did not offer the correct granularity for supporting the required synchronization between dependent members. It was determined that building transactions are worth the extra complexity to encourage developers using the API to consider what models for transactionality they need, instead of thinking just in terms of will/did events.

Another design included an `Observer` protocol that could be used to build callback-style observer types. This has been eliminated in favor of the `AsyncSequence` approach.

The `ObservedChange` type could have the `Sendable` requirement relaxed by making the type only conditionally `Sendable` and then allowing access to the subject in all cases; however this poses some restriction to the internal implementations and may have a hole in the sendable nature of the type. Since it is viewed that accessing values is most commonly by one property the values `AsyncSequence` fills most of that role and for cases where more than one field is needed to be accessed on a given actor the iteration can be done with a weak reference to the observable subject.

## Acknowledgments

* [Holly Borla](https://github.com/hborla) - For providing fantastic ideas on how to implement supporting infrastructure to this pitch
* [Pavel Yaskevich](https://github.com/xedin) - For tirelessly iterating on prototypes for supporting compiler features
* Rishi Verma - For bouncing ideas and helping with the design of integrating this idea into other work
* [Kyle Macomber](https://github.com/kylemacomber) - For connecting resources and providing useful feedback
* Matt Ricketson - For helping highlight some of the inner guts of SwiftUI

## Related systems

* [Swift `Combine.ObservableObject`](https://developer.apple.com/documentation/combine/observableobject/)
* [Objective-C Key Value Observing](https://developer.apple.com/documentation/objectivec/nsobject/nskeyvalueobserving?language=objc)
* [C# `IObservable`](https://learn.microsoft.com/en-us/dotnet/api/system.iobservable-1?view=net-6.0)
* [Rust `Trait rx::Observable`](https://docs.rs/rx/latest/rx/trait.Observable.html)
* [Java `Observable`](https://docs.oracle.com/javase/7/docs/api/java/util/Observable.html)
* [Kotlin `observable`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.properties/-delegates/observable.html)
