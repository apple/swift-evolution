## Introduction

Property Wrappers have empowered users to abstract common property implementation details into expressive components. This proposal aims to make property wrappers more flexible and efficient by allowing them to opt-in to a shared storage.

## Motivation

Property Wrappers are responsible for wrapping common getting and setting boilerplate and also for storing any auxiliary helper properties. Often, these helpers are constant across different instances of the wrapper, not changing after initialization. Thus, having to store these properties in each individual wrapper instance should be avoided. In the following `Clamped` example, every wrapped instance will store its own `range` — even though there isn't a way for this range to change across different `Hud` initializations.

```swift
@propertyWrapper
struct Clamped<Value: Comparable> {
  private var value: Value
  let range: ClosedRange<Value>
  
  init(wrappedValue: Value, _ range: ClosedRange<Value>) {
    self.value = range.clamping(wrappedValue) 
    self.range = range
  }
  
  var wrappedValue: Value {
    get { value }
    set { value = range.clamping(newValue) }
  }
}

struct Hud {
  @Clamped(0...100) var value = 100
}

// the `range` property is constant and has the same value
// on both `hud1` and `hud2` 
let hud1 = Hud()
let hud2 = Hud()
```

### API-level property wrappers

Another motivation for this feature is mentioned in the [Static property-wrapper attribute arguments](https://github.com/apple/swift-evolution/blob/main/proposals/0293-extend-property-wrappers-to-function-and-closure-parameters.md#static-property-wrapper-attribute-arguments) Future Direction's section of SE-0293. To achieve consistency across the multiple initialization kinds API-level property wrappers are not allowed to have arguments in their wrapper attribute. 

## Proposed solution

We propose introducing storage that is shared per property wrapper instance. The storage is immutable, initialized once, and stored outside of the instance scope. It's a tool for property wrapper authors to optimize their wrapper abstractions and avoid repeated unnecessary storage.

```swift
@propertyWrapper
struct Clamped<Value: Comparable> {
  @shared let storage: RangeStorage<Value>
  
  var wrappedValue: Value { ... }
  
  // a plain struct, with properties that will be shared across wrappers
  struct RangeStorage<Value: Comparable> { ... }
}
```

## Detailed design

The storage is declared using the new `@shared` declaration attribute inside a Property Wrapper declaration. This property will be initialized and stored globally by the compiler while remaining accessible to the property wrapper instance like any other private property defined in the wrapper type. 

In the next example, the `RangeStorage` struct will be used for the `Clamped` wrapper. 

```swift
struct RangeStorage<Value: Comparable> {
  let range: ClosedRange<Value>
  init(_ range: ClosedRange<Value>) { 
    self.range = range
  } 
  
  func clamp(_ value: Value) -> Value {}
}

@propertyWrapper
struct Clamped<Value: Comparable> {
  @shared let storage: RangeStorage<Value>
  private var value: Value
  
  init(wrappedValue: Value, @shared: RangeStorage<Value>) {
    self.value = shared.clamp(wrappedValue)
  }
  
  var wrappedValue: Value {
    get { value }
    set {
      // `storage` is available like any other property
      value = storage.clamp(newValue) 
    }
  }
}  
```

And at the point of use, the compiler will make sure `RangeStorage` is initialized once for each wrapper application and stored at a scope outside of the instances. Later on, when multiple instances of `Hud` are initialized, they'll all be given access to the same `$shared` properties. Suggestions on how to name the `$shared` property would be much appreciated. 🙂

```swift
struct Hud {
  @Clamped(RangeStorage(0...14)) var bar = 5
  @Clamped(RangeStorage(1...9)) var foo = 1
}

var hud1 = Hud()
var hud2 = Hud()

// desugars to

struct Hud {
  static let bar$shared = RangeStorage(0...14)
  static let foo$shared = RangeStorage(1...9)
  
  var bar = Clamped(wrappedValue: 5, @shared: bar$shared)
  var foo = Clamped(wrappedValue: 1, @shared: foo$shared)
}

// both Hud's get access to the same $shared properties.
var hud1 = Hud() 
var hud2 = Hud() 
```

### Initialization

Inside the wrapper's initializer, assigning the shared value to the `shared` property is handled by the compiler, so there's no need to explicitly do it. 

```swift
@shared let storage: RangeStorage<Value>

init(wrappedValue: Value, @shared: RangeStorage<Value>) {
  self.value = shared.clamp(wrappedValue)
}
```

The initialization of the storage value itself follows the same principles as static variables: it can't use instance variables or methods that depend on `self` being initialized. Though literals and other type variables can be used. 

```swift
struct RangeStorage {
  init(_ range: String) { ... } 
}

struct Container { 
  @Clamped(RangeStorage(1...7)) var weekday = 3
}

// not okay
struct ContainerB {
  var minDay: Int
  var maxDay: Int
  @Clamped(RangeStorage(minDay...maxDay)) var weekday = 3
}
```

Property wrappers can be initialized in multiple ways (through a `wrappedValue` initializer, a `projectedValue`, or default inits). For property wrappers passed as function arguments, which initializer is called depends on the value that is passed to the function. For those reasons, property wrappers that declare a dependence of a `shared` storage will need to include it on all initializers.

```swift
@propertyWrapper
struct Wrapper<Value> {
  var wrappedValue: Value
  var projectedValue: Wrapper
  @shared let storage: SomeStorage
  
  init(wrappedValue: Value, @shared: SomeStorage) { // }
  
  init(projectedValue: Wrapper, @shared: SomeStorage) { //	}
  
  init(@shared: SomeStorage = SomeStorage()) { // }
}

// ...

@Wrapper(SomeStorage()) var value = ""
```

The initialization of the shared storage must be resolved and stored at the call site. So providing a default value on the initializer argument is allowed but initializing it inside the wrapper declaration is not. 

```swift
@propertyWrapper 
struct Wrapper {
  // ...
  @shared let storage = SomeStorage() // * error
}
```

Since the goal of this feature is to allow instances of the type containing a property wrapped property to share the same storage instance, injecting the shared storage into the `Container` is also a violation. 

```swift
class Container {
  @Wrapper var someProperty: String 
  
  // this way instances of `Container` could have different `storage` values 
  init(value: String, @shared storage: SomeStorage) {
    self._someProperty = Wrapper(wrappedValue: value, shared: storage)  // error
  }
}
```

### Access control

The shared property is accessible anywhere in the `Wrapper` scope, like any other property. However, unlike other generated property wrapper properties, it's not *directly* visible to the container type. It can only be accessed through the backing storage property (unless it was declared private).

```swift
class Container { 
  // @shared let someProperty$shared = SomeStorage("hi") 
  @Wrapper(SomeStorage("hi")) var someProperty = ""
  
  func accessStorage() {
    print(someProperty$shared) // not allowed
    print(_someProperty.storage) // okay
  }
}
```

### Lifecycle

There are a few important aspects of the lifecycle of the shared storage. About its initialization, it happens only once. And then it's reused for subsequent instances that need it. 

```swift
class Container { 
  @Wrapper(SomeStorage()) var someProperty = ""
}

let firstContainer = Container() // `shared let someProperty$shared` initialized

// the following `Container` instances use the `someProperty$shared` 
let secondContainer = Container()
let anotherContainer = Container()

```

The shared storage can be declared with classes, structs, and enums. Multiple containers will end up using the storage, potentially at the same time, so it should be read-only. Since the storage is immutable and Swift uses copy on write to avoid needlessly copying values, only one instance of the storage will be alive in the memory regardless of how many container instances use it. 

The storage lifecycle is not tied to the "original" instance that caused it to be initialized in the first place. Instead, it follows the rules of other Type properties: it must be given a default value, and it is lazily initialized.

### API-level Property Wrappers on function parameters

Implementing this feature also unlocks the possibility for API-level wrappers to pass arguments to the `shared storage` when passed as function parameters. And unlike the strategy mentioned in the Future Directions section of the SE-0293, using the shared storage won't require accessing wrapped and projected values through subscripts.

```swift
struct Layout {} 

struct SharedStorage { 
  let layout: Layout
  
  static func italic() -> Layout {}
} 

@propertyWrapper
struct Style {
  @shared let storage: SharedStorage
  var wrappedValue: UIView 
  var projectedValue: Style { self }
  
  init(wrappedValue: UIView) {}
  
  init(projectedValue: Style, @shared: SharedStorage) { // }  
}

func emphasized(@Style(.italic()) label: UILabel) {}
```

### Composition of Property Wrappers

When a property declares multiple property wrappers, they get composed and their effects are combined through a composition chain. For wrappers with a shared storage dependency, the same can be applied. 

Take for example the following composition chain, where one of the wrappers has shared storage and the other does not. 

```swift
@WithShared(.init()) @Without var foo: Bool = false
```

The composition chain will be resolved by nesting the inner wrapper into the outer wrapper type and initializing the shared property as needed. The same logic applies to the reversed order of application ( `@Without @WithShared var foo`).

```swift
shared let foo$shared = Shared()
var foo: WithShared<Without<Bool>> = WithShared(wrappedValue: Without(wrappedValue: false), foo$shared)
```

In the case of a property with multiple applications of the same wrapper with shared storage, the composition chain would be resolved in the same way. Each wrapper gets its own shared storage property regardless. 

```swift
@WithShared(.init()) @WithShared(.init()) var foo: Bool = false 
```

```swift
shared let baz$shared = Shared()
shared let baz2$shared = Shared()

var baz: WithShared<WithShared<Bool>> = WithShared(wrappedValue: WithShared(wrappedValue: false, baz$shared), baz2$shared)
```

## Impact on existing code

This is an additive feature, and it shouldn't impact existing source code. 

### Backward compatibility

However, from a library evolution standpoint, making an existing property wrapper opt-in into the shared storage model can be a non-resilient change for ABI-public property wrappers. 

Consider a type that exposes a property with a property wrapper to its public API. 

```swift
@propertyWrapper
public struct Wrapper<Value> {
  var wrappedValue: Value { ... }
  var projectedValue: Wrapper { ... } 
}

public struct Container {
  @Wrapper public var someValue: String
}

// -------
// the generated interface
public struct Container {
  public var someValue: String 
  public var $someValue: Wrapper 
}

@propertyWrapper
public struct Wrapper<Value> { ... }
```

Suppose that on a later version, the author of this property wrapper decides to change it by adding shared storage. Even if the shared storage is given a default argument in the property wrapper initializer, this is a non-resilient change. The same would be true for the opposite scenario: removing the shared storage from an ABI-public wrapper.

The example shows an API-level property wrapper, but the same would apply to an ABI-public implementation detail wrapper.  

### Alternatives considered

**Static shared storage**

Instead of introducing a new attribute, we could store the generated storage property as a normal static variable in its enclosing instance.

```swift
@propertyWrapper(shared: Storage)
struct Clamped {
  private var value: Value
  
  var wrappedValue: Value { fatalError("use the subscript!") }
  
  // wrappedValue would be accessed through a subscript
  subscript(shared storage: Storage) -> Value {
    get { value }
    set { value = storage.range.clamping(newValue) }
  }
  
  struct Storage {
    let range: ClosedRange<Value>
    init(range: ClosedRange<Value>) { // ... } 
  }
} 

// .... using it

struct Hud {
  @Clamped(range: 0...100) var value = 100
}

// Desugared version:
struct Hud {
  private var _value: Clamped<Int> = .init(wrappedValue: 100)
  private static let _value$shared: Clamped<Int>.Storage = .init(range: 0...100)
  var value: Int {
    get { _value[shared: Hud._value$shared] }
    set { _value[shared: Hud._value$shared] = newValue }
  }
}
```

Readability is one of the main disadvantages of this approach, as it would require passing the storage around through subscripts. 

### Related Work

The Future Directions sections on both SE-0258 and SE-0293, and the threads discussing this feature, especially this [thread](https://forums.swift.org/t/future-directions-of-property-wrappers/27934/11), were essential to this proposal.
