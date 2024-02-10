# Refining property-wrapper-related initialization

* Proposal: [SE-NNNN](NNNN-refining-property-wrapper-related-initialization.md)
* Authors: [Amritpan Kaur](https://github.com/amritpan), [Filip Sakel](https://github.com/filip-sakel), [Frederick Kellison-Linn](https://github.com/jumhyn)
* Review Manager: TBD
* Status: **Awaiting implementation**

## Introduction

[SE 0258](https://github.com/apple/swift-evolution/blob/master/proposals/0258-property-wrappers.md) introduced property wrappers and [SE 0293](https://github.com/apple/swift-evolution/blob/main/proposals/0293-extend-property-wrappers-to-function-and-closure-parameters.md#detailed-design) expanded them with function-like declarations. Today, property wrapper initialization exhibits inconsistencies due to its growing versatility. Specifically, memberwise initializers use complex, poorly documented rules and projection initialization remains limited. This proposal will simplify synthesized memberwise initialization for types with wrapped properties and extend projection value initialization to include global, type, and local wrapped properties.

## Motivation

Property wrappers were initially adopted in  [SE 0258](https://github.com/apple/swift-evolution/blob/main/proposals/0258-property-wrappers.md) and expanded to function parameters and closures in  [SE 0293](https://github.com/apple/swift-evolution/blob/main/proposals/0293-extend-property-wrappers-to-function-and-closure-parameters.md). This two-phase adoption of property wrappers throughout the language has left property wrappers considered as a whole with some inconsistencies and complexity that is no longer necessary.

Today, the rules for how wrapped properties are represented in the synthesized memberwise initializer for structs are poorly documented and quite complex. Here's an example to illustrate some of the points from [this thread](https://forums.swift.org/t/does-the-new-swift-5-5-init-projectedvalue-functionality-not-work-with-synthesized-memberwise-initializers/51232/7):

```swift
@propertyWrapper
struct Wrapper {
    let wrappedValue: Int
    
    init(wrappedValue: Int = 5) {
        self.wrappedValue = wrappedValue
    }
}

@propertyWrapper
struct ArgumentWrapper {
    let wrappedValue: Int
    let arg: Int
}

struct Client {
    @Wrapper var a
    @Wrapper var b = 2
    @ArgumentWrapper var c: Int
    @ArgumentWrapper(arg: 0) var d = 17
}

let client = Client(
    // Backing storage, because @Wrapper is default initialized.
    a: Wrapper(wrappedValue: 1),
    // Wrapped value, because we've provided a default value in Client.
    b: 2,
    // Our wrapper has uninitialized arguments, so we pass a wrapper instance.
    c: ArgumentWrapper(wrappedValue: 3, arg: 0),
    // We've provided the arguments and default value in Client.
    d: 4
)
```

It is evident that the author of a type with wrapped properties cannot easily determine or alter the signature of their synthesized initializer. This synthesis depends on subtle interactions between the property-wrapper type, the declaration of the wrapped property, and how that property is initialized.

Furthermore, the current ruleset can implicitly leak the private storage of the property wrapper via the (implicitly `internal`) synthesized initializer. This not only takes away control from wrapper authors, but may also cause users to abandon their synthesized initializer altogether.

SE 0293 also added a new way of initializing property wrapper storage from a projected value using a new special initializer, `init(projectedValue:)`. While the new `$`-initialization syntax works for function arguments and closure parameters, global, type, and local properties have no such equivalent.

To sum up, the current state of affairs is ripe for refinement. Property wrappers have matured to a point where we can easily simplify certain special cases, and extend the general functionality to support a consistent model for property wrappers everywhere.

## Proposed Solution

We propose two additions to the feature set of property wrappers that will improve consistency with the SE 0293 model.
First, we propose an update to the rules of the synthesized memberwise initializer for types with wrapped properties such that wrapper attributes are mapped directly into the initializer. 
For example, a type such as this one from [TSPL](https://docs.swift.org/swift-book/LanguageGuide/Properties.html#ID617):

```swift
struct MixedRectangle {
    @SmallNumber var height: Int = 1
    @SmallNumber(maximum: 9) var width: Int = 2
}
```

would receive a synthesized memberwise initializer that looks like:

```swift
init(@SmallNumber height: Int = 1, @SmallNumber(maximum: 9) width: Int = 2) {
  ...
}
```

Second, we propose allowing property wrapper storage for global, type, and local wrapped properties to be initialized via the `init(projectedValue:)` system (as discussed in the [future directions of SE 0293](https://github.com/apple/swift-evolution/blob/main/proposals/0293-extend-property-wrappers-to-function-and-closure-parameters.md#generalized-property-wrapper-initialization-from-a-projection)). E.g.:

```swift
@Wrapper
var property: Int

$property = someProjectedValue
```

## Detailed Design

### Synthesized memberwise initializer and property wrappers

We propose a new algorithm for generating the synthesized memberwise initializer as it relates to wrapped properties. Rather than trying to decide whether to expose the storage type or the wrapped type in the initializer, a property declared in a type `MyType` as:

```swift
@Wrapper(arg: value)
var property: X = initialValue
```

will always be represented in the synthesized memberwise initializer signature as:

```swift
init(..., @Wrapper(arg: value) property: X = initialValue, ...)
```

(where the argument list after `Wrapper` and the `initialValue` may or may not be present).

Note that the rules for property wrappers in parameters are outlined in [SE 0293](https://github.com/apple/swift-evolution/blob/main/proposals/0293-extend-property-wrappers-to-function-and-closure-parameters.md#detailed-design). Most importantly:
- If the property declaration _does_ provide arguments to the wrapper attribute, `Wrapper` will always be an implementation-detail property wrapper.
- If `Wrapper` provides an `init(projectedValue:)`, then when calling the initializer the corresponding parameter may be prefixed with `$` and accept the projected-value type:
    ```swift
    let _ = MyType(..., $property: ProjectedValue(), ...)
    ```
    
    
If the specific initializers declared by the `Wrapper` type would result in an un-callable synthesized initializer (for example, because `Wrapper` provides no appropriate `init(wrappedValue:)` or `init(projectedValue:)`), an error will be emitted and the user will have to adjust their property declaration or define a custom initializer (see [**Source Compatibility**](#source-compatibility) below for more information).

### Projected value initialization

We propose extending the `$` syntax for argument labels and closure parameters to allow the initialization of *any* property wrapper storage from a projected value (provided that the wrapper type supplies an `init(projectedValue:)`).

Specifically, at a point in the program where a wrapped property's storage is uninitialized, we allow assignment of the form:
```swift
$property = someProjectedValue
```

which will be transformed to:

```swift
_property = .init(projectedValue: someProjectedValue)
```

Anywhere the storage has been initialized, `$property` retains its usual meaning and will refer to the `projectedValue` property of the wrapper. This transformation takes place even if `_property.projectedValue` does not provide a setter, since we are formally assigning `_property`, not the projected value.

> Note that such initialization can be used in more complex expressions such as `($clampedValue, editCount) = (clampedProjection, 0)` 

## Source Compatibility

### Staging in the new memberwise initializer

Changing the algorithm which generates the memberwise initializer is a source-breaking change, and therefore must be introduced in a new language version. As a concrete example, this proposal will change initialization syntax for SwiftUI views which use `@Binding` and rely on the synthesized initializer:

```swift
struct MyView: View {
    @Binding
    var x: Int

    var body: some View {
        Text("\(x)")
    }
}

MyView(x: .constant(5))  // ❌ old init
MyView($x: .constant(5)) // ✅ new init
```

To ease the transition to the new memberwise initializer, we propose the following plan:
- In the Swift 5 language mode, both the old and the new memberwise initializers will be synthesized. 
  - In otherwise ambiguous cases, the old initializer is unconditionally preferred over the new one during overload resolution. 
  - The old initializer will not be suggested in code completion or appear in the "sanitized" swiftinterface (presented for frameworks by Xcode).
- In the Swift 6 language mode, both the old and the new memberwise initializer will be synthesized. 
  - In otherwise ambiguous cases, the _new_ initializer is unconditionally preferred over the old one during overload resolution. 
  - The old initializer will not be suggested in code completion or appear in the "sanitized" swiftinterface, and uses of the old initializer will warn about its impending removal.
  - If the new initializer would not be callable, a warning is emitted.

- In a future language version (Swift 7?), only the new memberwise initializer will be synthesized. Any attempt to use the old initializer is an error.
  - If the new initializer would not be callable, an error is emitted.
The flags `-warn-wrapper-init` and `-force-wrapper-init` may be used to opt into the Swift 6 or Swift 7 behavior with regards to the synthesized memberwise initializer.
### Projected value initialization

This aspect of the proposal is completely additive and will be source-compatible.

## Effect on ABI Stability

This feature and the features it depends on are either entirely non-public or implemented entirely in terms of frontend transformations and do not have an impact on ABI.

## Effect on API Resilience

Because the synthesized memberwise initializer is always non-`public`, this proposal does not introduce any functionality that would affect API resilience.

## Alternatives Considered

*TBC*

## Future Directions

### Inline projected value initialization

The `$`-initialization syntax could be extended to allow for its use in the declaration of the wrapped property itself, e.g.:

```swift
struct S {
  @Wrapper
  var $property: Wrapper = someProjectedValue
}
```

would expand to:

```swift
struct S {
  private var _property: Wrapper = .init(projectedValue: someProjectedValue)
    
  var property: Int {
    get { _property.wrappedValue }
    set { _property.wrappedValue = newValue }
  }
}
```

The authors elect to exclude such a construction from this proposal. The suggested syntax above tries to do many things at once:
1. Declare a property named `property`.
2. Declare a wrapper/storage type `Wrapper` for that property.
3. Declare the type of the projected value.
4. Declare that the property should be initialized via `init(projectedValue:)`.
5. Declare the projected value to be passed to `init(projectedValue:)`.

Notably, this syntax does not provide an easy place to indicate the type of `property` itself. The reader may be able to infer it based on the type of the wrapper or the projected value, but this could require inspection of the type declaration.


While it is possible that the syntax here could be massaged into something that made a bit more sense, there are enough open questions that the authors would rather see this form receive further, separate consideration.

### Accept the Backing Storage in Parameters

[SE 0293](https://github.com/apple/swift-evolution/blob/main/proposals/0293-extend-property-wrappers-to-function-and-closure-parameters.md) settled on parameters accepting either the wrapped or projected value. If neither was supported, the resulting declaration would be uncallable. That can be limiting, though, especially for memberwise initializers:

```swift
@propertyWrapper struct BackingStorageOnly {
  let wrappedValue = 0
}

struct Client {
  // ℹ️ `@BackingStorageOnly` doesn't declare an `init(wrappedValue:)` or `init(projectedValue:)` initializer.
  @BackingStorageOnly var property: Int 
  
  // ❌ Invoking uncallable memberwise initializer.
  static let `default` = Client()
}
```

The rationale was that the _private_ backing storage must not be exposed to function clients. This is a valid concern, but doesn't preclude a `private`, backing-storage-accepting function that follows these rules:

* A wrapper with neither an `init(wrappedValue:)` or `init(projectedValue:)` special initializer is considered **private API level**;
* A function-like declaration with at least one private-API wrapper must be `private`; and
* Private-API-wrapped parameters have underscore-prefixed names and accept their backing-storage type.

```swift
struct Client {
  // ...
  
  static let `default` = Client(_property: BackingStorageOnly())
}
```

One issue with introducing a new API significance characterizations that combines API and implementation detail is how it will be inferred. This is not a straightforward decision, because API-level inference depends solely on the primary wrapper type declaration; `init(wrappedValue:)` is recognized as a special wrapper init even in extensions.

## Acknowledgments

*TBC*
