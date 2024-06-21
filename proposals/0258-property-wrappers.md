# Property Wrappers

* Proposal: [SE-0258](0258-property-wrappers.md)
* Authors: [Doug Gregor](https://github.com/DougGregor), [Joe Groff](https://github.com/jckarter)
* Review Manager: [John McCall](https://github.com/rjmccall)
* Status: **Implemented (Swift 5.1)**
* Implementation: [Linux toolchain](https://ci.swift.org/job/swift-PR-toolchain-Linux/251//artifact/branch-master/swift-PR-25781-251-ubuntu16.04.tar.gz), [macOS toolchain](https://ci.swift.org/job/swift-PR-toolchain-osx/327//artifact/branch-master/swift-PR-25781-327-osx.tar.gz)
* Review: ([review #1](https://forums.swift.org/t/se-0258-property-delegates/23139)) ([revision announcement #1](https://forums.swift.org/t/returned-for-revision-se-0258-property-delegates/24080)) ([review #2](https://forums.swift.org/t/se-0258-property-wrappers-second-review/25843)) ([review #3](https://forums.swift.org/t/se-0258-property-wrappers-third-review/26399)) ([acceptance](https://forums.swift.org/t/accepted-with-modification-se-0258-property-wrappers/26828))
* Previous versions: [Revision #3](https://github.com/swiftlang/swift-evolution/blob/e99ae69370f56ae84256b78902ab377cb8249cdd/proposals/0258-property-wrappers.md), [Revision #2](https://github.com/swiftlang/swift-evolution/blob/bb8709c2ddca25c21a3c1e0298ce9457911dbfba/proposals/0258-property-wrappers.md), [Revision #1](https://github.com/swiftlang/swift-evolution/commit/8c3499ec5bc22713b150e2234516af3cb8b16a0b)

## Contents

+ [Introduction](#introduction)
+ [Motivation](#motivation)
+ [Proposed solution](#proposed-solution)
+ [Examples](#examples)
  - [Delayed Initialization](#delayed-initialization)
  - [`NSCopying`](#nscopying)
  - [Thread-specific storage](#thread-specific-storage)
  - [User defaults](#user-defaults)
  - [Copy-on-write](#copy-on-write)
  - [`Ref` / `Box`](#ref--box)
  - ["Clamping" a value within bounds](#clamping-a-value-within-bounds)
  - [Property wrapper types in the wild](#property-wrapper-types-in-the-wild)
+ [Composition of property wrappers](#composition-of-property-wrappers)
+ [Detailed design](#detailed-design)
  - [Property wrapper types](#property-wrapper-types)
  - [Initialization of synthesized storage properties](#initialization-of-synthesized-storage-properties)
  - [Type inference with property wrappers](#type-inference-with-property-wrappers)
  - [Custom attributes](#custom-attributes)
  - [Mutability of properties with wrappers](#mutability-of-properties-with-wrappers)
  - [Out-of-line initialization of properties with wrappers](#out-of-line-initialization-of-properties-with-wrappers)
  - [Memberwise initializers](#memberwise-initializers)
  - [Codable, Hashable, and Equatable synthesis](#codable-hashable-and-equatable-synthesis)
  - [$ identifiers](#-identifiers)
  - [Projections](#projections)
  - [Restrictions on the use of property wrappers](#restrictions-on-the-use-of-property-wrappers)
+ [Impact on existing code](#impact-on-existing-code)
+ [Backward compatibility](#backward-compatibility)
+ [Alternatives considered](#alternatives-considered)
  - [Composition](#composition)
  - [Composition via nested type lookup](#composition-via-nested-type-lookup)
  - [Composition without nesting](#composition-without-nesting)
  - [Using a formal protocol instead of `@propertyWrapper`](#using-a-formal-protocol-instead-of-propertywrapper)
  - [Kotlin-like `by` syntax](#kotlin-like-by-syntax)
  - [Alternative spellings for the `$` projection property](#alternative-spellings-for-the--projection-property)
  - [The 2015-2016 property behaviors design](#the-2015-2016-property-behaviors-design)
+ [Future Directions](#future-directions)
  - [Finer-grained access control](#finer-grained-access-control)
  - [Referencing the enclosing 'self' in a wrapper type](#referencing-the-enclosing-self-in-a-wrapper-type)
  - [Delegating to an existing property](#delegating-to-an-existing-property)
+ [Revisions](#revisions)
  - [Changes from the accepted proposal](#changes-from-the-accepted-proposal)
  - [Changes from the third reviewed version](#changes-from-the-third-reviewed-version)
  - [Changes from the second reviewed version](#changes-from-the-second-reviewed-version)
  - [Changes from the first reviewed version](#changes-from-the-first-reviewed-version)
+ [Acknowledgments](#acknowledgments)

## Introduction

There are property implementation patterns that come up repeatedly.
Rather than hardcode a fixed set of patterns into the compiler (as we have done for `lazy` and `@NSCopying`),
we should provide a general "property wrapper" mechanism to allow
these patterns to be defined as libraries.

This is an alternative approach to some of the problems intended to be addressed by the [2015-2016 property behaviors proposal](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0030-property-behavior-decls.md). Some of the examples are the same, but this proposal takes a completely different approach designed to be simpler, easier to understand for users, and less invasive in the compiler implementation. There is a section that discusses the substantive differences from that design near the end of this proposal.

[Pitch #1](https://forums.swift.org/t/pitch-property-delegates/21895)<br/>
[Pitch #2](https://forums.swift.org/t/pitch-2-property-delegates-by-custom-attributes/22855)<br/>
[Pitch #3](https://forums.swift.org/t/pitch-3-property-wrappers-formerly-known-as-property-delegates/24961)<br/>

## Motivation

We've tried to accommodate several important patterns for properties with
targeted language support, like `lazy` and `@NSCopying`, but this support has been narrow in scope and utility.  For instance, Swift provides `lazy` properties as a primitive language feature, since lazy initialization is common and is often necessary to avoid having properties be exposed as `Optional`. Without this language support, it takes a lot of boilerplate to get the same effect:

```swift
struct Foo {
  // lazy var foo = 1738
  private var _foo: Int?
  var foo: Int {
    get {
      if let value = _foo { return value }
      let initialValue = 1738
      _foo = initialValue
      return initialValue
    }
    set {
      _foo = newValue
    }
  }
}
```

Building `lazy` into the language has several disadvantages. It makes the
language and compiler more complex and less orthogonal. It's also inflexible;
there are many variations on lazy initialization that make sense, but we
wouldn't want to hardcode language support for all of them.

There are important property patterns outside of lazy initialization.  It often
makes sense to have "delayed", once-assignable-then-immutable properties to
support multi-phase initialization:

```swift
class Foo {
  let immediatelyInitialized = "foo"
  var _initializedLater: String?

  // We want initializedLater to present like a non-optional 'let' to user code;
  // it can only be assigned once, and can't be accessed before being assigned.
  var initializedLater: String {
    get { return _initializedLater! }
    set {
      assert(_initializedLater == nil)
      _initializedLater = newValue
    }
  }
}
```

Implicitly-unwrapped optionals allow this in a pinch, but give up a lot of
safety compared to a non-optional 'let'. Using IUO for multi-phase
initialization gives up both immutability and nil-safety.

The attribute `@NSCopying` introduces a use of `NSCopying.copy()` to
create a copy on assignment. The implementation pattern may look familiar:

```swift
class Foo {
  // @NSCopying var text: NSAttributedString
  var _text: NSAttributedString
  var text: NSAttributedString {
    get { return _text }
    set { _text = newValue.copy() as! NSAttributedString }
  }
}
```

## Proposed solution

We propose the introduction of **property wrappers**, which allow a
property declaration to state which **wrapper** is used to implement
it.  The wrapper is described via an attribute:

```swift
@Lazy var foo = 1738
```

This implements the property `foo` in a way described by the *property wrapper type* for `Lazy`:

```swift
@propertyWrapper
enum Lazy<Value> {
  case uninitialized(() -> Value)
  case initialized(Value)

  init(wrappedValue: @autoclosure @escaping () -> Value) {
    self = .uninitialized(wrappedValue)
  }

  var wrappedValue: Value {
    mutating get {
      switch self {
      case .uninitialized(let initializer):
        let value = initializer()
        self = .initialized(value)
        return value
      case .initialized(let value):
        return value
      }
    }
    set {
      self = .initialized(newValue)
    }
  }
}
```

A property wrapper type provides the storage for a property that
uses it as a wrapper. The `wrappedValue` property of the wrapper type
provides the actual
implementation of the wrapper, while the (optional)
`init(wrappedValue:)` enables initialization of the storage from a
value of the property's type. The property declaration

```swift
@Lazy var foo = 1738
```

translates to:

```swift
private var _foo: Lazy<Int> = Lazy<Int>(wrappedValue: 1738)
var foo: Int {
  get { return _foo.wrappedValue }
  set { _foo.wrappedValue = newValue }
}
```

The use of the prefix `_` for the synthesized storage property name is
deliberate: it provides a predictable name for the synthesized storage property that
fits established conventions for `private` stored properties. For example,
we could provide a `reset(_:)` operation on `Lazy` to set it back to a new
value:

```swift
extension Lazy {
  /// Reset the state back to "uninitialized" with a new,
  /// possibly-different initial value to be computed on the next access.
  mutating func reset(_ newValue:  @autoclosure @escaping () -> Value) {
    self = .uninitialized(newValue)
  }
}

_foo.reset(42)
```

The backing storage property can also be explicitly initialized. For example:

```swift
extension Lazy {
  init(body: @escaping () -> Value) {
    self = .uninitialized(body)
  }
}

func createAString() -> String { ... }

@Lazy var bar: String  // not initialized yet
_bar = Lazy(body: createAString)
```

The property wrapper instance can be initialized directly by providing the initializer arguments in parentheses after the name. The above code can be written equivalently in a single declaration as:

```swift
@Lazy(body: createAString) var bar: String
```

Property wrappers can be applied to properties at global, local, or type scope. Those properties can have observing accessors (`willSet`/`didSet`), but not explicitly-written getters or setters.

The `Lazy` property wrapper has little or no interesting API outside of its initializers, so it is not important to export it to clients. However, property wrappers can also describe rich relationships that themselves have interesting API. For example, we might have a notion of a property wrapper that references a database field established by name (example inspired by [Tanner](https://forums.swift.org/t/se-0258-property-wrappers-second-review/25843/14)):

```swift
@propertyWrapper
public struct Field<Value: DatabaseValue> {
  public let name: String
  private var record: DatabaseRecord?
  private var cachedValue: Value?
  
  public init(name: String) {
    self.name = name
  }

  public func configure(record: DatabaseRecord) {
    self.record = record
  }
  
  public var wrappedValue: Value {
    mutating get {
      if cachedValue == nil { fetch() }
      return cachedValue!
    }
    
    set {
      cachedValue = newValue
    }
  }
  
  public func flush() {
    if let value = cachedValue {
      record!.flush(fieldName: name, value)
    }
  }
  
  public mutating func fetch() {
    cachedValue = record!.fetch(fieldName: name, type: Value.self)
  }
}
```

We could define our model based on the `Field` property wrapper:

```swift
public struct Person: DatabaseModel {
  @Field(name: "first_name") public var firstName: String
  @Field(name: "last_name") public var lastName: String
  @Field(name: "date_of_birth") public var birthdate: Date
}
```

`Field` itself has API that is important to users of `Person`: it lets us flush existing values, fetch new values, and retrieve the name of the corresponding field in the database. However, the underscored variables for each of the properties of our model (`_firstName`, `_lastName`, and `_birthdate`) are `private`, so our clients cannot manipulate them directly.

To vend API, the property wrapper type `Field` can provide a *projection* that allows us to manipulate the relationship of the field to the database. Projection properties are prefixed with a `$`, so the projection of the `firstName` property is called `$firstName` and is visible wherever `firstName` is visible. Property wrapper types opt into provided a projection by defining a `projectedValue` property:

```swift
@propertyWrapper
public struct Field<Value: DatabaseValue> {
  // ... API as before ...
  
  public var projectedValue: Self {
    get { self }
    set { self = newValue }
  }
}
```

When `projectedValue` is present, the projection variable is created as a wrapper around `projectedValue`. For example, the following property:

```swift
@Field(name: "first_name") public var firstName: String
```

expands to:

```swift
private var _firstName: Field<String> = Field(name: "first_name")

public var firstName: String {
  get { _firstName.wrappedValue }
  set { _firstName.wrappedValue = newValue }
}

public var $firstName: Field<String> {
  get { _firstName.projectedValue }
  set { _firstName.projectedValue = newValue }
}
```

This allows clients to manipulate both the property and its projection, e.g.,

```swift
somePerson.firstName = "Taylor"
$somePerson.flush()
```

## Examples

Before describing the detailed design, here are some more examples of
wrappers.

### Delayed Initialization

A property wrapper can model "delayed" initialization, where
the definite initialization (DI) rules for properties are enforced
dynamically rather than at compile time.  This can avoid the need for
implicitly-unwrapped optionals in multi-phase initialization. We can
implement both a mutable variant, which allows for reassignment like a
`var`:

```swift
@propertyWrapper
struct DelayedMutable<Value> {
  private var _value: Value? = nil

  var wrappedValue: Value {
    get {
      guard let value = _value else {
        fatalError("property accessed before being initialized")
      }
      return value
    }
    set {
      _value = newValue
    }
  }

  /// "Reset" the wrapper so it can be initialized again.
  mutating func reset() {
    _value = nil
  }
}
```

and an immutable variant, which only allows a single initialization like
a `let`:

```swift
@propertyWrapper
struct DelayedImmutable<Value> {
  private var _value: Value? = nil

  var wrappedValue: Value {
    get {
      guard let value = _value else {
        fatalError("property accessed before being initialized")
      }
      return value
    }

    // Perform an initialization, trapping if the
    // value is already initialized.
    set {
      if _value != nil {
        fatalError("property initialized twice")
      }
      _value = newValue
    }
  }
}
```

This enables multi-phase initialization, like this:

```swift
class Foo {
  @DelayedImmutable var x: Int

  init() {
    // We don't know "x" yet, and we don't have to set it
  }

  func initializeX(x: Int) {
    self.x = x // Will crash if 'self.x' is already initialized
  }

  func getX() -> Int {
    return x // Will crash if 'self.x' wasn't initialized
  }
}
```

### `NSCopying`

Many Cocoa classes implement value-like objects that require explicit copying.
Swift currently provides an `@NSCopying` attribute for properties to give
them behavior like Objective-C's `@property(copy)`, invoking the `copy` method
on new objects when the property is set. We can turn this into a wrapper:

```swift
@propertyWrapper
struct Copying<Value: NSCopying> {
  private var _value: Value
  
  init(wrappedValue value: Value) {
    // Copy the value on initialization.
    self._value = value.copy() as! Value
  }

  var wrappedValue: Value {
    get { return _value }
    set {
      // Copy the value on reassignment.
      _value = newValue.copy() as! Value
    }
  }
}
```

This implementation would address the problem detailed in
[SE-0153](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0153-compensate-for-the-inconsistency-of-nscopyings-behaviour.md). Leaving the `copy()` out of `init(wrappedValue:)` implements the pre-SE-0153 semantics.

### Thread-specific storage

Thread-specific storage (based on pthreads) can be implemented as a property wrapper, too (example courtesy of Daniel Delwood):

```swift
@propertyWrapper
final class ThreadSpecific<T> {
  private var key = pthread_key_t()
  private let initialValue: T

  init(key: pthread_key_t, wrappedValue: T) {
    self.key = key
    self.initialValue = wrappedValue
  }

  init(wrappedValue: T) {
    self.initialValue = wrappedValue
    pthread_key_create(&key) {
      // 'Any' erasure due to inability to capture 'self' or <T>
      $0.assumingMemoryBound(to: Any.self).deinitialize(count: 1)
      $0.deallocate()
    }
  }

  deinit {
    fatalError("\(ThreadSpecific<T>.self).deinit is unsafe and would leak")
  }

  private var box: UnsafeMutablePointer<Any> {
    if let pointer = pthread_getspecific(key) {
      return pointer.assumingMemoryBound(to: Any.self)
    } else {
      let pointer = UnsafeMutablePointer<Any>.allocate(capacity: 1)
      pthread_setspecific(key, UnsafeRawPointer(pointer))
      pointer.initialize(to: initialValue as Any)
      return pointer
    }
  }

  var wrappedValue: T {
    get { return box.pointee as! T }
    set (v) {
      box.withMemoryRebound(to: T.self, capacity: 1) { $0.pointee = v }
    }
  }
}
```


### User defaults

Property wrappers can be used to provide typed properties for
string-keyed data, such as [user defaults](https://developer.apple.com/documentation/foundation/userdefaults) (example courtesy of Harlan Haskins),
encapsulating the mechanism for extracting that data in the wrapper type.
For example:

```swift
@propertyWrapper
struct UserDefault<T> {
  let key: String
  let defaultValue: T
  
  var wrappedValue: T {
    get {
      return UserDefaults.standard.object(forKey: key) as? T ?? defaultValue
    }
    set {
      UserDefaults.standard.set(newValue, forKey: key)
    }
  }
}

enum GlobalSettings {
  @UserDefault(key: "FOO_FEATURE_ENABLED", defaultValue: false)
  static var isFooFeatureEnabled: Bool
  
  @UserDefault(key: "BAR_FEATURE_ENABLED", defaultValue: false)
  static var isBarFeatureEnabled: Bool
}
```

### Copy-on-write

With some work, property wrappers can provide copy-on-write wrappers (original example courtesy of Becca Royal-Gordon):

```swift
protocol Copyable: AnyObject {
  func copy() -> Self
}

@propertyWrapper
struct CopyOnWrite<Value: Copyable> {
  init(wrappedValue: Value) {
    self.wrappedValue = wrappedValue
  }
  
  private(set) var wrappedValue: Value
  
  var projectedValue: Value {
    mutating get {
      if !isKnownUniquelyReferenced(&wrappedValue) {
        wrappedValue = wrappedValue.copy()
      }
      return wrappedValue
    }
    set {
      wrappedValue = newValue
    }
  }
}
```

`projectedValue` provides projection for the synthesized storage property, allowing the copy-on-write wrapper to be used directly:

```swift
@CopyOnWrite var storage: MyStorageBuffer

// Non-modifying access:
let index = storage.index(of: …)

// For modification, access $storage, which goes through `projectedValue`:
$storage.append(…)
```

### `Ref` / `Box`

We can define a property wrapper type `Ref` that is an abstracted reference
to some value that can be get/set, which is effectively a programmatic computed property:

```swift
@propertyWrapper
struct Ref<Value> {
  let read: () -> Value
  let write: (Value) -> Void

  var wrappedValue: Value {
    get { return read() }
    nonmutating set { write(newValue) }
  }

  subscript<U>(dynamicMember keyPath: WritableKeyPath<Value, U>) -> Ref<U> {
    return Ref<U>(
        read: { self.wrappedValue[keyPath: keyPath] },
        write: { self.wrappedValue[keyPath: keyPath] = $0 })
  }
}
```

The subscript is using [SE-0252 "Key Path Member Lookup"](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0252-keypath-dynamic-member-lookup.md) so that a `Ref` instance provides access to the properties of its value. Building on the example from SE-0252:

```swift
@Ref(read: ..., write: ...)
var rect: Rectangle

print(rect)          // accesses the Rectangle
print(rect.topLeft)  // accesses the topLeft component of the rectangle

let rect2 = _rect    // get the Ref<Rectangle>
let topLeft2 = _rect.topLeft // get a Ref<Point> referring to the Rectangle's topLeft
```

The `Ref` type encapsulates read/write, and making it a property wrapper lets
us primarily see the underlying value. Often, one does not want to explicitly
write out the getters and setters, and it's fairly common to have a `Box` type that boxes up a value and can vend `Ref` instances referring into that box. We can do so with another property wrapper:

```swift
@propertyWrapper
class Box<Value> {
  var wrappedValue: Value

  init(wrappedValue: Value) {
    self.wrappedValue = wrappedValue
  }

  var projectedValue: Ref<Value> {
    return Ref<Value>(read: { self.wrappedValue }, write: { self.wrappedValue = $0 })
  }
}
```

Now, we can define a new `Box` directly:

```swift
@Box var rectangle: Rectangle = ...

print(rectangle)  // access the rectangle
print(rectangle.topLeft) // access the top left coordinate of the rectangle
let rect2 = $rectangle   // through projectedValue, produces a Ref<Rectangle>
let topLeft2 = $rectangle.topLeft   // through projectedValue, produces a Ref<Point>
```

The use of `projectedValue` hides the box from the client (`_rectangle` remains private), providing direct access to the value in the box (the common case) as well as access to the box contents via `Ref` (referenced as `$rectangle`).

### "Clamping" a value within bounds

A property wrapper could limit the stored value to be within particular bounds. For example, the `Clamping` property wrapper provides min/max bounds within which values will be clamped:

```swift
@propertyWrapper
struct Clamping<V: Comparable> {
  var value: V
  let min: V
  let max: V

  init(wrappedValue: V, min: V, max: V) {
    value = wrappedValue
    self.min = min
    self.max = max
    assert(value >= min && value <= max)
  }

  var wrappedValue: V {
    get { return value }
    set {
      if newValue < min {
        value = min
      } else if newValue > max {
        value = max
      } else {
        value = newValue
      }
    }
  }
}
```

Most interesting in this example is how `@Clamping` properties can be
initialized given both an initial value and initializer arguments. In such cases, the `wrappedValue:` argument is placed first. For example, this means we can define a `Color` type that clamps all values in the range [0, 255]:

```swift
struct Color {
  @Clamping(min: 0, max: 255) var red: Int = 127
  @Clamping(min: 0, max: 255) var green: Int = 127
  @Clamping(min: 0, max: 255) var blue: Int = 127
  @Clamping(min: 0, max: 255) var alpha: Int = 255
}
```

The synthesized memberwise initializer demonstrates how the initialization itself is formed:

```swift
init(red: Int = 127, green: Int = 127, blue: Int = 127, alpha: Int = 255) {
  _red = Clamping(wrappedValue: red, min: 0, max: 255)
  _green = Clamping(wrappedValue: green, min: 0, max: 255)
  _blue = Clamping(wrappedValue: blue, min: 0, max: 255)
  _alpha = Clamping(wrappedValue: alpha, min: 0, max: 255)
}
```

(Example [courtesy of Avi](https://forums.swift.org/t/pitch-3-property-wrappers-formerly-known-as-property-delegates/24961/65))

### Property wrapper types in the wild

There are a number of existing types that already provide the basic structure of a property wrapper type. One fun case is `Unsafe(Mutable)Pointer`, which we could augment to allow easy access to the pointed-to value:

```swift
@propertyWrapper
struct UnsafeMutablePointer<Pointee> {
  var pointee: Pointee { ... }
  
  var wrappedValue: Pointee {
    get { return pointee }
    set { pointee = newValue }
  }
}
```

From a user perspective, this allows us to set up the unsafe mutable pointer's address once, then mostly refer to the pointed-to value:

```
@UnsafeMutablePointer(mutating: addressOfAnInt)
var someInt: Int

someInt = 17 // equivalent to _someInt.pointee = 17
print(someInt)

_someInt.deallocate()
```

RxCocoa's [`BehaviorRelay`](https://github.com/ReactiveX/RxSwift/blob/master/RxCocoa/Traits/BehaviorRelay.swift) replays the most recent value provided to it for each of the subscribed observers. It is created with an initial value, has `wrappedValue` property to access the current value and a `projectedValue` to expose a projection providing API to `subscribe` a new observer: (Thanks to Adrian Zubarev for pointing this out)

```swift
@BehaviorRelay
var myValue: Int = 17

let observer = $myValue.subscribe(...)   // subscribe an observer
$myValue.accept(42)  // set a new value via the synthesized storage property

print(myValue)   // print the most recent value
```

[Combine's `Published`](https://developer.apple.com/documentation/combine/published) property wrapper is similar in spirit, allowing clients to subscribe to `@Published` properties (via the `$` projection) to receive updates when the value changes.

[SwiftUI](https://developer.apple.com/xcode/swiftui/) makes extensive use of
property wrappers to declare local state (`@State`) and express data dependencies on other state that can effect the UI (`@EnvironmentObject`, `@Environment`, `@ObjectBinding`). It makes extensive use of projections to the [`Binding`](https://developer.apple.com/documentation/swiftui/binding) property wrapper to allow controlled mutation of the state that affects UI.

## Composition of property wrappers

When multiple property wrappers are provided for a given property,
the wrappers are composed together to get both effects. For example, consider the composition of `DelayedMutable` and `Copying`:

```swift
@DelayedMutable @Copying var path: UIBezierPath
```

Here, we have a property for which we can delay initialization until later. When we do set a value, it will be copied via `NSCopying`'s `copy` method.

Composition is implemented by nesting later wrapper types inside earlier wrapper types, where the innermost nested type is the original property's type. For the example above, the backing storage will be of type `DelayedMutable<Copying<UIBezierPath>>`, and the synthesized getter/setter for `path` will look through both levels of `.wrappedValue`:

```swift
private var _path: DelayedMutable<Copying<UIBezierPath>> = .init()
var path: UIBezierPath {
  get { return _path.wrappedValue.wrappedValue }
  set { _path.wrappedValue.wrappedValue = newValue }
}  
```

Note that this design means that property wrapper composition is not commutative, because the order of the attributes affects how the nesting is performed:

```swift
@DelayedMutable @Copying var path1: UIBezierPath   // _path1 has type DelayedMutable<Copying<UIBezierPath>>
@Copying @DelayedMutable var path2: UIBezierPath   // error: _path2 has ill-formed type Copying<DelayedMutable<UIBezierPath>>
```

In this case, the type checker prevents the second ordering, because `DelayedMutable` does not conform to the `NSCopying` protocol. This won't always be the case: some semantically-bad compositions won't necessarily be caught by the type system. Alternatives to this approach to composition are presented in "Alternatives considered." 

## Detailed design

### Property wrapper types

A *property wrapper type* is a type that can be used as a property
wrapper. There are two basic requirements for a property wrapper
type:

1. The property wrapper type must be defined with the attribute
`@propertyWrapper`. The attribute indicates that the type is meant to
be used as a property wrapper type, and provides a point at which the
compiler can verify any other consistency rules.
2. The property wrapper type must have a property named `wrappedValue`, whose
access level is the same as that of the type itself. This is the
property used by the compiler to access the underlying value on the
wrapper instance.

### Initialization of synthesized storage properties

Introducing a property wrapper to a property makes that property
computed (with a getter/setter) and introduces a stored property whose
type is the wrapper type. That stored property can be initialized
in one of three ways:

1. Via a value of the original property's type (e.g., `Int` in `@Lazy var
   foo: Int`, using the property wrapper type's
   `init(wrappedValue:)` initializer. That initializer must have a single
   parameter of the same type as the `wrappedValue` property (or
   be an `@autoclosure` thereof) and have the same access level as the 
   property wrapper type itself. When `init(wrappedValue:)` is present,
   is always used for the initial value provided on the property
   declaration. For example:

   ```swift
   @Lazy var foo = 17

   // ... implemented as
   private var _foo: Lazy = Lazy(wrappedValue: 17)
   var foo: Int { /* access via _foo.wrappedValue as described above */ }
   ```

   When there are multiple, composed property wrappers, all of them must provide an `init(wrappedValue:)`, and the resulting initialization will wrap each level of call:

   ```swift
   @Lazy @Copying var path = UIBezierPath()

   // ... implemented as
   private var _path: Lazy<Copying<UIBezierPath>> = .init(wrappedValue: .init(wrappedValue: UIBezierPath()))
   var path: UIBezierPath { /* access via _path.wrappedValue.wrappedValue as described above */ }
   ```

2. Via a value of the property wrapper type, by placing the initializer
   arguments after the property wrapper type:

   ```swift
   var addressOfInt: UnsafePointer<Int> = ...

   @UnsafeMutablePointer(mutating: addressOfInt) 
   var someInt: Int

   // ... implemented as
   private var _someInt: UnsafeMutablePointer<Int> = UnsafeMutablePointer(mutating: addressOfInt)
   var someInt: Int { /* access via _someInt.wrappedValue */ }
   ```

   When there are multiple, composed property wrappers, only the first (outermost) wrapper may have initializer arguments.

3. Implicitly, when no initializer is provided and the property wrapper type provides a no-parameter initializer (`init()`). In such cases, the wrapper type's `init()` will be invoked to initialize the stored property.

   ```swift
   @DelayedMutable var x: Int

   // ... implemented as
   private var _x: DelayedMutable<Int> = DelayedMutable<Int>()
   var x: Int { /* access via _x.wrappedValue */ }
   ```

   When there are multiple, composed property wrappers, only the first (outermost) wrapper needs to have an `init()`.

### Type inference with property wrappers

If the first property wrapper type is generic, its generic arguments must either be given explicitly in the attribute or Swift must be able to deduce them from the variable declaration. That deduction proceeds as follows:

* If the variable has an initial value expression `E`, then the first wrapper type is constrained to equal the type resulting from a call to `A(wrappedValue: E, argsA...)`, where `A` is the written type of the attribute and `argsA` are the arguments provided to that attribute. For example:

  ```swift
  @Lazy var foo = 17
  // type inference as in...
  private var _foo: Lazy = Lazy(wrappedValue: 17)
  // infers the type of '_foo' to be 'Lazy<Int>'
  ```

  If there are multiple wrapper attributes, the argument to this call will instead be a nested call to `B(wrappedValue: E, argsB...)` for the written type of the next attribute, and so on recursively. For example:
  
  ```swift
  @A @B(name: "Hello") var bar = 42
  // type inference as in ...
  private var _bar = A(wrappedValue: B(wrappedValue: 42, name: "Hello"))
  // infers the type of '_bar' to be 'A<B<Int>'
  ```

* Otherwise, if the first wrapper attribute has direct initialization arguments `E...`, the outermost wrapper type is constrained to equal the type resulting from `A(E...)`, where `A` is the written type of the first attribute. Wrapper attributes after the first may not have direct initializers. For example:

  ```swift
  @UnsafeMutablePointer(mutating: addressOfInt)
  var someInt
  // type inference as in...
  private var _someInt: UnsafeMutablePointer = UnsafeMutablePointer.init(mutating: addressOfInt)
  // infers the type of `_someInt` to be `UnsafeMutablePointer<Int>`
  ```

* Otherwise, if there is no initialization, and the original property has a type annotation, the type of the `wrappedValue` property in the last wrapper type is constrained to equal the type annotation of the original property. For example:

  ```swift
  @propertyWrapper
  struct Function<T, U> {
    var wrappedValue: (T) -> U? { ... }
  }

  @Function var f: (Int) -> Float?   // infers T=Int, U=Float 
  ```

In any case, the first wrapper type is constrained to be a specialization of the first attribute's written type. Furthermore, for any secondary wrapper attributes, the type of the wrappedValue property of the previous wrapper type is constrained to be a specialization of the attribute's written type. Finally, if a type annotation is given, the type of the wrappedValue property of the last wrapper type is constrained to equal the type annotation. If these rules fail to deduce all the type arguments for the first wrapper type, or if they are inconsistent with each other, the variable is ill-formed. For example:

```swift
@Lazy<Int> var foo: Int  // okay
@Lazy<Int> var bar: Double  // error: Lazy<Int>.wrappedValue is of type Int, not Double
```

The deduction can also provide a type for the original property (if a type annotation was omitted) or deduce generic arguments that have been omitted from the type annotation. For example:

```swift
@propertyWrapper
struct StringDictionary {
  var wrappedValue: [String: String]
}

@StringDictionary var d1.            // infers Dictionary<String, String>
@StringDictionary var d2: Dictionary // infers <String, String>
```

### Custom attributes

Property wrappers are a form of custom attribute, where the attribute syntax
is used to refer to entities declared in Swift. Grammatically, the use of property wrappers is described as follows:

```
attribute ::= '@' type-identifier expr-paren?
```

The *type-identifier* must refer to a property wrapper type, which can include generic arguments. Note that this allows for qualification of the attribute names, e.g.,

```swift
@Swift.Lazy var foo = 1742
```

The *expr-paren*, if present, provides the initialization arguments for the wrapper instance.

This formulation of custom attributes fits in with a [larger proposal for custom attributes](https://forums.swift.org/t/pitch-introduce-custom-attributes/21335/47), which uses the same custom attribute syntax as the above but allows for other ways in which one can define a type to be used as an attribute. In this scheme, `@propertyWrapper` is just one kind of custom attribute: there will be other kinds of custom attributes that are available only at compile time (e.g., for tools) or runtime (via some reflection capability).

### Mutability of properties with wrappers

Generally, a property that has a property wrapper will have both a getter and a setter. However, the setter may be missing if the `wrappedValue` property of the property wrapper type lacks a setter, or its setter is inaccessible.

The synthesized getter will be `mutating` if the property wrapper type's `wrappedValue` property is `mutating` and the property is part of a `struct`. Similarly, the synthesized setter will be `nonmutating` if either the property wrapper type's `wrappedValue` property has a `nonmutating` setter or the property wrapper type is a `class`. For example:

```swift
@propertyWrapper
struct MutatingGetterWrapper<Value> {
  var wrappedValue: Value {
    mutating get { ... }
    set { ... }
  }
}

@propertyWrapper
struct NonmutatingSetterWrapper<Value> {
  var wrappedValue: Value {
    get { ... }
    nonmutating set { ... }
  }
}

@propertyWrapper
class ReferenceWrapper<Value> {
  var wrappedValue: Value
}

struct Usewrappers {
  // x's getter is mutating
  // x's setter is mutating
  @MutatingGetterWrapper var x: Int

  // y's getter is nonmutating
  // y's setter is nonmutating
  @NonmutatingSetterWrapper var y: Int
  
  // z's getter is nonmutating
  // z's setter is nonmutating
  @ReferenceWrapper var z: Int  
}
```

### Out-of-line initialization of properties with wrappers

A property that has a wrapper can be initialized after it is defined,
either via the property itself (if the wrapper type has an
`init(wrappedValue:)`) or via the synthesized storage property. For
example:


```swift
@Lazy var x: Int
// ...
x = 17   // okay, treated as _x = .init(wrappedValue: 17)
```

The synthesized storage property can also be initialized directly,
e.g.,

```swift
@UnsafeMutable var y: Int
// ...
_y = UnsafeMutable<Int>(pointer: addressOfInt) // okay
```

Note that the rules of [definite
initialization](https://developer.apple.com/swift/blog/?id=28) (DI)
apply to properties that have wrappers. Let's expand the example of
`x` above to include a re-assignment and use `var`:

```swift
@Lazy var x2: Int
// ...
x2 = 17   // okay, treated as _x2 = .init(wrappedValue: 17)
// ...
x2 = 42   // okay, treated as x2 = 42 (calls the Lazy.wrappedValue setter)
```

### Memberwise initializers

Structs implicitly declare memberwise initializers based on the stored
properties of the struct. With a property that has a wrapper, the
property is technically computed because it's the synthesized property
(of the wrapper's type) that is stored. Instance properties that have a
property wrapper will have a corresponding parameter in the memberwise
initializer, whose type will either be the original property type or
the wrapper type, depending on the wrapper type and the initial value
(if provided). Specifically, the memberwise initializer parameter for
an instance property with a property wrapper will have the original
property type if either of the following is true:

* The corresponding property has an initial value specified with the
`=` syntax, e.g., `@Lazy var i = 17`, or
- The corresponding property has no initial value, but the property
wrapper type has an `init(wrappedValue:)`.

Otherwise, the memberwise initializer parameter will have the same
type as the wrapper. For example:

```swift
struct Foo {
  @UserDefault(key: "FOO_FEATURE_ENABLED", defaultValue: false)
  var x: Bool
  @Lazy var y: Int = 17
  @Lazy(closure: { getBool() }) var z: Bool
  @CopyOnWrite var w: Image

  // implicit memberwise initializer:
  init(x: UserDefault<Bool> = UserDefault(key: "FOO_FEATURE_ENABLED", defaultValue: false),
       y: Int = 17,
       z: Lazy<Bool> = Lazy(closure: { getBool() }),
       w: Image) {
    self._x = x
    self._y = Lazy(wrappedValue: y)
    self._z = z
    self._w = CopyOnWrite(wrappedValue: w)
  }
}
```

### Codable, Hashable, and Equatable synthesis

Synthesis for `Encodable`, `Decodable`, `Hashable`, and `Equatable`
use the backing storage property. This allows property wrapper types to determine their own serialization and equality behavior. For `Encodable` and `Decodable`, the name used for keyed archiving is that of the original property declaration (without the `_`).

### $ identifiers

Currently, identifiers starting with a `$` are not permitted in Swift programs. Today, such identifiers are only used in LLDB, where they can be used to name persistent values within a debugging session.

This proposal loosens these rules slightly: the Swift compiler will introduce identifiers that start with `$` (for the projection property), and Swift code can reference those properties. However, Swift code cannot declare any new entities with an identifier that begins with `$`. For example:

```swift
@CopyOnWrite var x = UIBezierPath()
print($x)                 // okay to refer to compiler-defined $x
let $y = UIBezierPath()   // error: cannot declare entity with $-prefixed name '$y'
```

### Projections

A property wrapper type can choose to provide a projection property (e.g., `$foo`) to expose more API for each wrapped property by defining a `projectedValue` property.  
As with the `wrappedValue` property and `init(wrappedValue:)`, the `projectedValue` property must have the
same access level as its property wrapper type. For example:

```swift
class StorageManager {
  func allocate<T>(_: T.Type) -> UnsafeMutablePointer<T> { ... }
}

@propertyWrapper
struct LongTermStorage<Value> {
  let pointer: UnsafeMutablePointer<Value>

  init(manager: StorageManager, initialValue: Value) {
    pointer = manager.allocate(Value.self)
    pointer.initialize(to: initialValue)
  }

  var wrappedValue: Value {
    get { return pointer.pointee }
    set { pointer.pointee = newValue }
  }

  var projectedValue: UnsafeMutablePointer<Value> {
    return pointer
  }
}
```

When we use the `LongTermStorage` wrapper, it handles the coordination with the `StorageManager` and provides either direct access or an `UnsafeMutablePointer` with which to manipulate the value:

```swift
let manager = StorageManager(...)

@LongTermStorage(manager: manager, initialValue: "Hello")
var someValue: String

print(someValue)     // prints "Hello"
someValue = "World"  // update the value in storage to "World"

// $someValue accesses the projectedValue property of the wrapper instance, which
// is an UnsafeMutablePointer<String>
let world = $someValue.move()   // take value directly from the storage
$someValue.initialize(to: "New value")
```

The projection property has the same access level as the original property:
```swift
@LongTermStorage(manager: manager, initialValue: "Hello")
public var someValue: String
```

is translated into:

```swift
private var _someValue: LongTermStorage<String> = LongTermStorage(manager: manager, initialValue: "Hello")

public var $someValue: UnsafeMutablePointer<String> {
  get { return _someValue.projectedValue }
}

public var someValue: String {
  get { return _someValue.wrappedValue }
  set { _someValue.wrappedValue = newValue }
}
```

Note that, in this example, `$someValue` is not writable, because `projectedValue` is a get-only property. 

When multiple property wrappers are applied to a given property, only the outermost property wrapper's `projectedValue` will be considered.

### Restrictions on the use of property wrappers

There are a number of restrictions on the use of property wrappers when defining a property:

* A property with a wrapper may not be declared inside a protocol.
* An instance property with a wrapper may not be declared inside an extension.
* An instance property may not be declared in an `enum`.
* A property with a wrapper that is declared within a class cannot override another property. 
* A property with a wrapper cannot be `lazy`, `@NSCopying`, `@NSManaged`, `weak`, or `unowned`.
* A property with a wrapper must be the only property declared within its enclosing declaration (e.g., `@Lazy var (x, y) = /* ... */` is ill-formed).
* A property with a wrapper shall not define a getter or setter.
* The `wrappedValue` property and (if present) `init(wrappedValue:)` of a property wrapper type shall have the same access as the property wrapper type.
* The `projectedValue` property, if present, shall have the same access as the property wrapper type.
* The `init()` initializer, if present, shall have the same access as the property wrapper type.

## Impact on existing code

By itself, this is an additive feature that doesn't impact existing
code. However, with some of the property wrappers suggested, it can
potentially obsolete existing, hardcoded language
features. `@NSCopying` could be completely replaced by a `Copying`
property wrapper type introduced in the `Foundation` module. `lazy`
cannot be completely replaced because it's initial value can refer to
the `self` of the enclosing type; see 'deferred evaluation of
initialization expressions_. However, it may still make sense to
introduce a `Lazy` property wrapper type to cover many of the common
use cases, leaving the more-magical `lazy` as a backward-compatibility
feature.

## Backward compatibility

The property wrappers language feature as proposed has no impact on the ABI or runtime. Binaries that use property wrappers can be backward-deployed to the Swift 5.0 runtime.

## Alternatives considered

### Composition

Composition was left out of the [first revision](https://github.com/swiftlang/swift-evolution/commit/8c3499ec5bc22713b150e2234516af3cb8b16a0b) of this proposal, because one can manually compose property wrapper types. For example, the composition `@A @B` could be implemented as an `AB` wrapper:

```swift
@propertyWrapper
struct AB<Value> {
  private var storage: A<B<Value>>
  
  var wrappedValue: Value {
    get { storage.wrappedValue.wrappedValue }
    set { storage.wrappedValue.wrappedValue = newValue }
  }
}
```

The main benefit of this approach is its predictability: the author of `AB` decides how to best achieve the composition of `A` and `B`, names it appropriately, and provides the right API and documentation of its semantics. On the other hand, having to manually write out each of the compositions is a lot of boilerplate, particularly for a feature whose main selling point is the elimination of boilerplate. It is also unfortunate to have to invent names for each composition---when I try to compose `A` and `B` via `@A @B`, how do I know to go look for the manually-composed property wrapper type `AB`? Or maybe that should be `BA`?

### Composition via nested type lookup
One proposed approach to composition addresses only the last issue above directly, treating the attribute-composition syntax `@A @B` as  a lookup of the nested type `B` inside `A` to find the wrapper type:

```swift
@propertyWrapper
struct A<Value> {
  var wrappedValue: Value { ... }
}

extension A {
  typealias B = AB<Value>
}
```

This allows the natural composition syntax `@A @B` to work, redirecting to manually-written property wrappers that implement the proper semantics and API. Additionally, this scheme allows one to control which compositions are valid: if there is no nested type `B` in `A`, the composition is invalid. If both `A.B` and `B.A` exist, we have a choice: either enforce commutative semantics as part of the language (`B.A` and `A.B` must refer to the same type or the composition `@A @B` is ill-formed), or allow them to differ (effectively matching the semantics of this proposal).

This approach addresses the syntax for composition while maintaining control over the precise semantics of composition via manually-written wrapper types. However, it does not address the boilerplate problem.
  
### Composition without nesting

There has been a desire to effect composition of property wrappers without having to wrap one property wrapper type in the other. For example, to have `@A @B` apply the policies of both `A` and `B` without producing a nested type like `A<B<Int>>`. This would potentially make composition more commutative, at least from the type system perspective. However, this approach does not fit with the "wrapper" approach taken by property wrappers. In a declaration

```swift
@A @B var x: Int
```

the `Int` value is conceptually wrapped by a property wrapper type, and the property wrapper type's `wrappedValue` property guards access to that (conceptual) `Int` value. That `Int` value cannot be wrapped both by instances of both `A` and `B` without either duplicating data (both `A` and `B` have a copy of the `Int`) or nesting one of the wrappers inside the other. With the copying approach, one must maintain consistency between the copies (which is particularly hard when value types are involved) and there will still be non-commutative compositions. Nesting fits better with the "wrapper" model of property wrappers.

### Using a formal protocol instead of `@propertyWrapper`

Instead of a new attribute, we could introduce a `PropertyWrapper`
protocol to describe the semantic constraints on property wrapper
types. It might look like this:

```swift
protocol PropertyWrapper {
  associatedtype Value
  var wrappedValue: Value { get }
}
```

There are a few issues here. First, a single protocol
`PropertyWrapper` cannot handle all of the variants of `wrappedValue` that
are implied by the section on mutability of properties with wrappers,
because we'd need to cope with `mutating get` as well as `set` and
`nonmutating set`. Moreover, protocols don't support optional
requirements, like `init(wrappedValue:)` (which also has two
forms: one accepting a `Value` and one accepting an `@autoclosure ()
-> Value`) and `init()`. To cover all of these cases, we would need
several related-but-subtly-different protocols.

The second issue is that, even if there were a single `PropertyWrapper`
protocol, we don't know of any useful generic algorithms or data
structures that seem to be implemented in terms of only
`PropertyWrapper`.


### Kotlin-like `by` syntax

A previous iteration of this proposal (and its [implementation](https://github.com/apple/swift/pull/23440)) used `by` syntax similar to that of [Kotlin's delegated
properties](https://kotlinlang.org/docs/reference/delegated-properties.html), where the `by` followed the variable declaration. For example:

```swift
var foo by Lazy = 1738

static var isFooFeatureEnabled: Bool by UserDefault(key: "FOO_FEATURE_ENABLED", defaultValue: false)
```

There are some small advantages to this syntax over the attribute formulation:

* For cases like `UserDefault` where the wrapper instance is initialized directly, the initialization happens after the original variable declaration, which reads better because the variable type and name come first, and how it's implemented come later. (Counter point: Swift developers are already accustomed to reading past long attributes, which are typically placed on the previous line)
* The `by wrapperType` formulation leaves syntactic space for add-on features like specifying the access level of the wrapper instance (`by private wrapperType`) or delegating to an existing property (`by someInstanceProperty`).

The main problem with `by` is its novelty: there isn't anything else in Swift quite like the `by` keyword above, and it is unlikely that the syntax would be re-used for any other feature. As a keyword, `by` is quite meaningless, and brainstorming  during the [initial pitch](https://forums.swift.org/t/pitch-property-delegates/21895) didn't find any clearly good names for this functionality. 

### Alternative spellings for the `$` projection property

The prefix `$` spelling for the projection property has been the source of
much debate. A number of alternatives have been proposed, including longer `#`-based spellings (e.g., `#storage(of: foo)`) and postfix `$` (e.g., `foo$`). The postfix `$` had the most discussion, based on the idea that it opens up more extension points in the future (e.g., `foo$storage` could refer to the backing storage, `foo$databaseHandle` could refer to a specific "database handle" projection for certain property wrappers, etc.). However, doing so introduces yet another new namespace of names to the language ("things that follow `$`) and isn't motivated by enough strong use cases.

### The 2015-2016 property behaviors design

Property wrappers address a similar set of use cases to *property behaviors*, which were [proposed and
reviewed](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0030-property-behavior-decls.md)
in late 2015/early 2016. The design did not converge, and the proposal
was deferred. This proposal picks up the thread, using much of the
same motivation and some design ideas, but attempting to simplify the
feature and narrow the feature set. Some substantive differences from
the prior proposal are:

* Behaviors were introduced into a property with the `[behavior]`
  syntax, rather than the attribute syntax described here. See the
  property behaviors proposal for more information.
* Wrappers are always expressed by a (generic) type. Property behaviors
  had a new kind of declaration (introduced by the
  `behavior` keyword). Having a new kind of declaration allowed for
  the introduction of specialized syntax, but it also greatly
  increased the surface area (and implementation cost) of the
  proposal. Using a generic type makes property wrappers more of a
  syntactic-sugar feature that is easier to implement and explain.
* Wrappers cannot declare new kinds of accessors (e.g., the
  `didChange` example from the property behaviors proposal).
* Wrappers used for properties declared within a type cannot refer to
  the `self` of their enclosing type. This eliminates some use cases
  (e.g., implementing a `Synchronized` property wrapper type that
  uses a lock defined on the enclosing type), but simplifies the
  design.
* Wrappers can be initialized out-of-line, and one
  can use the `$`-prefixed name to refer to the storage property.
  These were future directions in the property behaviors proposal.


## Future Directions

### Finer-grained access control

By default, the synthesized storage property will have `private` access, and the projection property (when available) will have the same access as the original wrapped property. However, there are various circumstances where it would be beneficial to expose the synthesized storage property. This could be performed "per-property", e.g., by introducing a syntax akin to `private(set)`:

```swift
// both foo and _foo are publicly visible, $foo remains private
@SomeWrapper
public public(storage) private(projection) var foo: Int = 1738
```

One could also consider having the property wrapper types themselves declare that the synthesized storage properties for properties using those wrappers should have the same access as the original property. For example:

```swift
@propertyWrapper(storageIsAccessible: true)
struct SomeWrapper<T> {
  var wrappedValue: T { ... }
}

// both bar and _bar are publicly visible
@SomeWrapper
public var bar: Int = 1738
```

The two features could also be combined, allowing property wrapper types to provide the default behavior and the `access-level(...)` syntax to change the default. The current proposal's rules are meant to provide the right defaults while allowing for a separate exploration into expanding the visibility of the synthesized properties.

### Referencing the enclosing 'self' in a wrapper type

Manually-written getters and setters for properties declared in a type often refer to the `self` of their enclosing type. For example, this can be used to notify clients of a change to a property's value:

```swift
public class MyClass: Superclass {
  private var backingMyVar: Int
  public var myVar: Int {
    get { return backingMyVar }
    set {
      if newValue != backingMyVar {
        self.broadcastValueWillChange(newValue: newValue)
      }
      backingMyVar = newValue
    }
  }
}
```

This "broadcast a notification that the value has changed" implementation cannot be cleanly factored into a property wrapper type, because it needs access to both the underlying storage value (here, `backingMyVar`) and the `self` of the enclosing type. We could require a separate call to register the `self` instance with the wrapper type, e.g.,

```swift
protocol Observed {
  func broadcastValueWillChange<T>(newValue: T)
}

@propertyWrapper
public struct Observable<Value> {
  public var stored: Value
  var observed: Observed?
  
  public init(wrappedValue: Value) {
    self.stored = wrappedValue
  }
  
  public func register(_ observed: Observed) {
    self.observed = observed
  }
  
  public var wrappedValue: Value {
    get { return stored }
    set {
      if newValue != stored {
        observed?.broadcastValueWillChange(newValue: newValue)
      }
      stored = newValue
    }
  }
}
```

However, this means that one would have to manually call `register(_:)` in the initializer for `MyClass`:

```swift
public class MyClass: Superclass {
  @Observable public var myVar: Int = 17
  
  init() {
    // self._myVar gets initialized with Observable(wrappedValue: 17) here
    super.init()
    self._myVar.register(self)    // register as an Observable
  }
}
```

This isn't as automatic as we would like, and it requires us to have a separate reference to the `self` that is stored within `Observable`. Moreover, it is hiding a semantic problem: the observer code that runs in the `broadcastValueWillChange(newValue:)` must not access the synthesized storage property in any way (e.g., to read the old value through `myVal` or subscribe/unsubscribe an observer via `_myVal`), because doing so will trigger a [memory exclusivity](https://swift.org/blog/swift-5-exclusivity/) violation (because we are calling `broadcastValueWillChange(newValue:)` from within the a setter for the same synthesized storage property).

To address these issues, we could extend the ad hoc protocol used to access the storage property of a `@propertyWrapper` type a bit further. Instead of a `wrappedValue` property, a property wrapper type could provide a static `subscript(instanceSelf:wrapped:storage:)`that receives `self` as a parameter, along with key paths referencing the original wrapped property and the backing storage property. For example:


```swift
@propertyWrapper
public struct Observable<Value> {
  private var stored: Value
  
  public init(wrappedValue: Value) {
    self.stored = wrappedValue
  }
  
  public static subscript<OuterSelf: Observed>(
      instanceSelf observed: OuterSelf,
      wrapped wrappedKeyPath: ReferenceWritableKeyPath<OuterSelf, Value>,
      storage storageKeyPath: ReferenceWritableKeyPath<OuterSelf, Self>
    ) -> Value {
    get {
      observed[keyPath: storageKeyPath].stored
    }
    set {
      let oldValue = observed[keyPath: storageKeyPath].stored
      if newValue != oldValue {
        observed.broadcastValueWillChange(newValue: newValue)
      }
      observed[keyPath: storageKeyPath].stored = newValue
    }
  }
}
```

The (generic) subscript gets access to the enclosing `self` type via its subscript parameter, eliminating the need for the separate `register(_:)` step and the (type-erased) storage of the outer `self`. The desugaring within `MyClass` would be as follows:

```swift
public class MyClass: Superclass {
  @Observable public var myVar: Int = 17
  
  // desugars to...
  private var _myVar: Observable<Int> = Observable(wrappedValue: 17)
  public var myVar: Int {
    get { Observable<Int>[instanceSelf: self, wrapped: \MyClass.myVar, storage: \MyClass._myVar] }
    set { Observable<Int>[instanceSelf: self, wrapped: \MyClass.myVar, storage: \MyClass._myVar] = newValue }
  }
}
```

The design uses a `static` subscript and provides key paths to both the original property declaration (`wrapped`) and the synthesized storage property (`storage`). A call to the static subscript's getter or setter does not itself constitute an access to the synthesized storage property, allowing us to address the memory exclusivity violation from the early implementation. The subscript's implementation is given the means to access the synthesized storage property (via the enclosing `self` instance and `storage` key path). In our `Observable` property wrapper, the static subscript setter performs two distinct accesses to the synthesized storage property via `observed[keyPath: storageKeyPath]`:

1. The read of the old value
2. A write of the new value

In between these operations is the broadcast operation to any observers. Those observers are permitted to read the old value, unsubscribe themselves from observation, etc., because at the time of the `broadcastValueWillChange(newValue:)` call there is no existing access to the synthesized storage property.

There is a secondary benefit to providing the key paths, because it allows the property wrapper type to reason about its different instances based on the identity of the `wrapped` key path.

This extension is backward-compatible with the rest of the proposal. Property wrapper types could opt in to this behavior by providing a `static subscript(instanceSelf:wrapped:storage:)`, which would be used in cases where the property wrapper is being applied to an instance property of a class. If such a property wrapper type is applied to a property that is not an instance property of a class, or for any property wrapper types that don't have such a static subscript, the existing `wrappedValue` could be used. One could even allow `wrappedValue` to be specified to be unavailable within property wrapper types that have the static subscript, ensuring that such property wrapper types could only be applied to instance properties of a class:

```swift
@availability(*, unavailable) 
var wrappedValue: Value {
  get { fatalError("only works on instance properties of classes") }
  set { fatalError("only works on instance properties of classes") }
}
```

The same model could be extended to static properties of types (passing the metatype instance for the enclosing `self`) as well as global and local properties (no enclosing `self`), although we would also need to extend key path support to static, global, and local properties to do so.
 
### Delegating to an existing property

When specifying a wrapper for a property, the synthesized storage property is implicitly created. However, it is possible that there already exists a property that can provide the storage. One could provide a form of property delegation that creates the getter/setter to forward to an existing property, e.g.:

```swift
lazy var fooBacking: SomeWrapper<Int>
@wrapper(to: fooBacking) var foo: Int
```

One could express this either by naming the property directly (as above) or, for an even more general solution, by providing a keypath such as `\.someProperty.someOtherProperty`.

## Revisions

### Changes from the accepted proposal

This proposal originally presented an example of implementing atomic operations using a property wrapper interface. This example was misleading because it would require additional compiler and library features to work correctly.

Programmers looking for atomic operations can use the [Swift Atomics](https://github.com/apple/swift-atomics) package.

For those who have already attempted to implement something similar, here is the original example, and why it is incorrect:

```swift
@propertyWrapper
class Atomic<Value> {
  private var _value: Value

  init(wrappedValue: Value) {
    self._value = wrappedValue
  }

  var wrappedValue: Value {
    get { return load() }
    set { store(newValue: newValue) }
  }

  func load(order: MemoryOrder = .relaxed) { ... }
  func store(newValue: Value, order: MemoryOrder = .relaxed) { ... }
  func increment() { ... }
  func decrement() { ... }
}
```

As written, this property wrapper does not access its wrapped value atomically. `wrappedValue.getter` reads the entire `_value` property nonatomically, *before* calling the atomic `load` operation on the copied value. Similarly, `wrappedValue.setter` writes the entire `_value` property nonatomically *after* calling the atomic `store` operation. So, in fact, there is no atomic access to the shared class property, `_value`.

Even if the getter and setter could be made atomic, useful atomic operations, like increment, cannot be built from atomic load and store primitives. The property wrapper in fact encourages race conditions by allowing mutating methods, such as `atomicInt += 1`, to be directly invoked on the nonatomic copy of the wrapped value.

### Changes from the third reviewed version

* `init(initialValue:)` has been renamed to `init(wrappedValue:)` to match the name of the property.

### Changes from the second reviewed version

* The synthesized storage property is always named with a leading `_` and is always `private`.
* The `wrapperValue` property has been renamed to `projectedValue` to make it sufficiently different from `wrappedValue`. This also gives us the "projection" terminology to talk about the `$` property.
* The projected property (e.g., `$foo`) always has the same access as the original wrapped property, rather than being artificially limited to `internal`. This reflects the idea that, for property wrapper types that have a projection, the projection is equal in importance to the wrapped value.

### Changes from the first reviewed version

* The name of the feature has been changed from "property delegates" to "property wrappers" to better communicate how they work and avoid the existing uses of the term "delegate" in the Apple developer community
* When a property wrapper type has a no-parameter `init()`, properties that use that wrapper type will be implicitly initialized via `init()`.
* Support for property wrapper composition has been added, using a "nesting" model.
* A property with a wrapper can no longer have an explicit `get` or `set` declared, to match with the behavior of existing, similar features (`lazy`, `@NSCopying`).
* A property with a wrapper does not need to be `final`.
* Reduced the visibility of the synthesized storage property to `private`.
* When a wrapper type provides `wrapperValue`, the (computed) `$` variable is `internal` (at most) and the backing storage variable gets the prefix `$$` (and remains private).
* Removed the restriction banning property wrappers from having names that match the regular expression `_*[a-z].*`.
* `Codable`, `Hashable`, and `Equatable` synthesis are now based on the backing storage properties, which is a simpler model that gives more control to the authors of property wrapper types.
* Improved type inference for property wrapper types and clarified that the type of the `wrappedValue` property is used as part of this inference. See the "Type inference" section.
* Renamed the `value` property to `wrappedValue` to avoid conflicts.
* Initial values and explicitly-specified initializer arguments can both be used together; see the `@Clamping` example.

## Acknowledgments

This proposal was greatly improved throughout its [first pitch](https://forums.swift.org/t/pitch-property-delegates/21895) by many people. Harlan Haskins, Becca Royal-Gordon, Adrian Zubarev, Jordan Rose and others provided great examples of uses of property wrappers (several of which are in this proposal). Adrian Zubarev and Kenny Leung helped push on some of the core assumptions and restrictions of the original proposal, helping to make it more general. Vini Vendramini and David Hart helped tie this proposal together with custom attributes, which drastically reduced the syntactic surface area of this proposal.
