# Init Accessors

* Proposal: [SE-0400](0400-init-accessors.md)
* Authors: [Holly Borla](https://github.com/hborla), [Doug Gregor](https://github.com/douggregor)
* Review Manager: [Frederick Kellison-Linn](https://github.com/Jumhyn)
* Status: **Implemented (Swift 5.9)**
* Implementation: On `main` behind experimental feature flag `InitAccessors`
* Review: ([pitch](https://forums.swift.org/t/pitch-init-accessors/64881)) ([review](https://forums.swift.org/t/se-0400-init-accessors/65583)) ([acceptance](https://forums.swift.org/t/accepted-se-0400-init-accessors/66212))

## Introduction

Init accessors generalize the out-of-line initialization feature of property wrappers to allow any computed property on types to opt into definite initialization analysis, and subsume initialization of a set of stored properties with custom initialization code.

## Motivation

Swift applies [definite initialization analysis](https://en.wikipedia.org/wiki/Definite_assignment_analysis) to stored properties, stored local variables, and variables with property wrappers. Definite initialization ensures that memory is initialized on all paths before it is accessed. A common pattern in Swift code is to use one property as backing storage for one or more computed properties, and abstractions like [property wrappers](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0258-property-wrappers.md) and [attached macros](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0389-attached-macros.md) help facilitate this pattern. Under this pattern, the backing storage is an implementation detail, and most code works with the computed property, including initializers.

Property wrappers support bespoke definite initialization that allows initializing the backing property wrapper storage via the computed property, always re-writing initialization-via-wrapped-property in the form `self.value = value` to initialization of the backing storage in the form of `_value = Wrapper(wrappedValue: value)`:

```swift
@propertyWrapper
struct Wrapper<T> {
  var wrappedValue: T
}

struct S {
  @Wrapper var value: Int

  init(value: Int) {
    self.value = value  // Re-written to self._value = Wrapper(wrappedValue: value)
  }

  init(other: Int) {
    self._value = Wrapper(wrappedValue: other) // Okay, initializes storage '_value' directly
  }
}
```

The ad-hoc nature of property wrapper initializers mixed with an exact definite initialization pattern prevent property wrappers with additional arguments from being initialized out-of-line. Furthermore, property-wrapper-like macros cannot achieve the same initializer usability, because any backing storage variables added must be initialized directly instead of supporting initialization through computed properties. For example, the [`@Observable` macro](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0395-observability.md) applies a property-wrapper-like transform that turns stored properties into computed properties backed by the observation APIs, but it provides no way to write an initializer using the original property names like the programmer expects:

```swift
@Observable
struct Proposal {
  var title: String
  var text: String

  init(title: String, text: String) {
    self.title = title // error: 'self' used before all stored properties are initialized
    self.text = text // error: 'self' used before all stored properties are initialized
  } // error: Return from initializer without initializing all stored properties
}
```

## Proposed solution

This proposal adds _`init` accessors_ to opt computed properties on types into definite initialization that subsumes initialization of a set of zero or more specified stored properties, which allows assigning to computed properties in the body of a type's initializer:

```swift
struct Angle {
  var degrees: Double
  var radians: Double {
    @storageRestrictions(initializes: degrees)
    init(initialValue)  {
      degrees = initialValue * 180 / .pi
    }

    get { degrees * .pi / 180 }
    set { degrees = newValue * 180 / .pi }
  }

  init(degrees: Double) {
    self.degrees = degrees // initializes 'self.degrees' directly
  }

  init(radiansParam: Double) {
    self.radians = radiansParam // calls init accessor for 'self.radians', passing 'radiansParam' as the argument
  }
}
```

The signature of an `init` accessor specifies up to two sets of stored properties: the properties that are accessed (via `accesses`) and the properties that are initialized (via `initializes`) by the accessor. `initializes` and `accesses` are side-effects of the `init` accessor. Access effects specify the other stored properties that can be accessed from within the `init` accessor (no other uses of `self` are allowed), and therefore must be initialized before the computed property's `init` accessor is invoked. The `init` accessor must initialize each of the initialized stored properties on all control flow paths. The `radians` property in the example above specifies no access effect, but initializes the `degrees` property, so it specifies only `initializes: degrees`.

Access effects allow a computed property to be initialized by placing its contents into another stored property:

```swift
struct ProposalViaDictionary {
  private var dictionary: [String: String]

  var title: String {
    @storageRestrictions(accesses: dictionary)
    init(newValue)  {
      dictionary["title"] = newValue
    }

    get { dictionary["title"]! }
    set { dictionary["title"] = newValue }
  }

   var text: String {
    @storageRestrictions(accesses: dictionary)
    init(newValue) {
      dictionary["text"] = newValue
    }

    get { dictionary["text"]! }
    set { dictionary["text"] = newValue }
  }

  init(title: String, text: String) {
    self.dictionary = [:] // 'dictionary' must be initialized before init accessors access it
    self.title = title // calls init accessor to insert title into the dictionary
    self.text = text   // calls init accessor to insert text into the dictionary

    // it is an error to omit either initialization above
  }
}
```

Both `init` accessors document that they access `dictionary`, which allows them to insert the new values into the dictionary with the appropriate key as part of initialization. This allows one to fully abstract away the storage mechanism used in the type.

Finally, computed properties with `init` accessors are privileged in the synthesized member-wise initializer. With this proposal, property wrappers have no bespoke definite and member-wise initialization support. Instead, the desugaring for property wrappers with an `init(wrappedValue:)` includes an `init` accessor for wrapped properties and a member-wise initializer including wrapped values instead of the respective backing storage. The property wrapper code in the Motivation section will desugar to the following code:

```swift
@propertyWrapper
struct Wrapper<T> {
  var wrappedValue: T
}

struct S {
  private var _value: Wrapper<Int>
  var value: Int {
    @storageRestrictions(initializes: _value)
    init(newValue)  {
      self._value = Wrapper(wrappedValue: newValue)
    }

    get { _value.wrappedValue }
    set { _value.wrappedValue = newValue }
  }

  // This initializer is the same as the generated member-wise initializer.
  init(value: Int) {
    self.value = value  // Calls 'init' accessor on 'self.value'
  }
}

S(value: 10)
```

This proposal allows macros to model the following property-wrapper-like patterns including out-of-line initialization of the computed property:
* A wrapped property with attribute arguments
* A wrapped property that is backed by an explicit stored property
* A set of wrapped properties that are backed by a single stored property

## Detailed design

### Syntax

The proposal adds a new kind of accessor, an `init` accessor, which can be written in the accessor list of a computed property. Init accessors add the following production rules to the grammar:

```
init-accessor -> 'init' init-accessor-parameter[opt] function-body

init-accessor-parameter -> '(' identifier ')'

accessor-block -> init-accessor
```

The `identifier` in an `init-accessor-parameter`, if provided, is the name of the parameter that contains the initial value. If not provided, a parameter with the name `newValue` is automatically created. The minimal init accessor has no parameter list and no initialization effects:

```swift
struct Minimal {
  var value: Int {
    init {
      print("init accessor called with \(newValue)")
    }

    get { 0 }
  }
}
```

This proposal also adds a new `storageRestrictions` attribute to describe the storage restrictions for `init` accessor blocks. The attribute can only be used on `init` accessors.  The attribute is described by the following production rules in the grammar:

```
attribute ::= storage-restrictions-attribute

storage-restrictions-attribute ::= '@' storageRestrictions '(' storage-restrictions[opt] ')'

storage-restrictions-initializes ::= 'initializes' ':' identifier-list
storage-restrictions-accesses ::= 'accesses' ':' identifier-list

storage-restrictions ::= storage-restrictions-accesses
storage-restrictions ::= storage-restrictions-initializes
storage-restrictions ::= storage-restrictions-initializes ',' storage-restrictions-accesses
```

The storage restriction attribute can include a list of stored properties that are initialized by this accessor (the identifier list in `storage-restrictions-initializes`), and a list of stored properties that are accessed by this accessor (the identifier list in `storage-restrictions-accesses`), each of which are optional:

```swift
struct S {
  var readMe: String

  var _x: Int

  var x: Int {
    @storageRestrictions(initializes: _x, accesses: readMe)
    init(newValue) {
      print(readMe)
      _x = newValue
    }

    get { _x }
    set { _x = newValue }
  }
}
```

If the accessor uses the default parameter name `newValue` and neither initializes nor accesses any stored property, the signature is not required.

Init accessors can subsume the initialization of a set of stored properties. Subsumed stored properties are specified through the `initializes` argument to the attribute. The body of an `init` accessor is required to initialize the subsumed stored properties on all control flow paths.

Init accessors can also require a set of stored properties to already be initialized when the body is evaluated, which are specified through the `accesses` argument to the attribute. These stored properties can be accessed in the accessor body; no other properties or methods on `self` are available inside the accessor body, nor is `self` available as a whole object (i.e., to call methods on it).

### Definite initialization of properties on `self`

The semantics of an assignment inside of a type's initializer depend on whether or not all of `self` is initialized on all paths at the point of assignment. Before all of `self` is initialized, assignment to a computed property with an `init` accessor is re-written to an `init` accessor call; after `self` has been initialized, assignment to a computed property is re-written to a setter call.

With this proposal, all of `self` is initialized if:
* All stored properties are initialized on all paths, and
* All computed properties with `init` accessors are initialized on all paths.

An assignment to a computed property with an `init` accessor before all of `self` is initialized will call the computed property's `init` accessor and initialize all of the stored properties specified in its `initializes` clause:

```swift
struct S {
  var x1: Int
  var x2: Int
  
  var computed: Int {
    @storageRestrictions(initializes: x1, x2)
    init(newValue) { ... }
  }

  init() {
    self.computed = 1 // initializes 'computed', 'x1', and 'x2'; 'self' is now fully initialized
  }
}
```

An assignment to a computed property that has not been initialized on all paths will be re-written to an `init` accessor call:

```swift
struct S {
  var x: Int
  var y: Int
  
  var point: (Int, Int) {
    @storageRestrictions(initializes: x, y)
    init(newValue) {
	    (self.x, self.y) = newValue
    }
    get { (x, y) }
    set { (x, y) = newValue }
  }

  init(x: Int, y: Int) {
    if (x == y) {
      self.point = (x, x) // calls 'init' accessor
    }

    // 'self.point' is not initialized on all paths here

    self.point = (x, y) // calls 'init' accessor

    // 'self.point' is initialized on all paths here
  }
}
```

An assignment to a stored property before all of `self` is initialized will initialize that stored property. When all of the stored properties listed in the `initializes` clause of a computed property with an `init` accessor have been initialized, that computed property is considered initialized:

```swift
struct S {
  var x1: Int
  var x2: Int
  var x3: Int
  
  var computed: Int {
    @storageRestrictions(initializes: x1, x2)
    init(newValue) { ... }
  }

  init() {
    self.x1 = 1 // initializes 'x1'; neither 'x2' or 'computed' is initialized
    self.x2 = 1 // initializes 'x2' and 'computed'
    self.x3 = 1 // initializes 'x3'; 'self' is now fully initialized
  }
}
```

An assignment to a computed property where at least one of the stored properties listed in `initializes` is initialized, but `self` is not initialized, is an error. This prevents double-initialization of the underlying stored properties:

```swift
struct S {
  var x: Int
  var y: Int
  
  var point: (Int, Int) {
    @storageRestrictions(initializes: x, y)
    init(newValue) {
      (self.x, self.y) = newValue
    }
    get { (x, y) }
    set { (x, y) = newValue }
  }

  init(x: Int, y: Int) {
    self.x = x // Only initializes 'x'
    self.point = (x, y) // error: neither the `init` accessor nor the setter can be called here
  }
}
```

### Memberwise initializers

If a struct does not declare its own initializers, it receives an implicit memberwise initializer based on the stored properties of the struct, because the storage is what needs to be initialized. Because many use-cases for `init` accessors are fully abstracting a single computed property to be backed by a single stored property, such as in the property-wrapper use case, an `init` accessor provides a preferred mechanism for initializing storage because the programmer will primarily interact with that storage through the computed property. As such, the memberwise initializer parameter list will include computed properties that have init accessors along with only those stored properties that have not been subsumed by an init accessor.

```swift
struct S {
  var _x: Int
  
  var x: Int {
    @storageRestrictions(initializes: _x)
    init(newValue) {
      _x = newValue
    }

    get { _x }
    set { _x = newValue }
  }

  var y: Int  
}

S(x: 10, y: 100)
```

The above struct `S` receives a synthesized initializer:

```swift
init(x: Int, y: Int) {
  self.x = x
  self.y = y
}
```

The parameters of the memberwise initializer follow source order. However, if an init accessor `accesses` a stored property that precedes it in the memberwise initializer, then the properties cannot be initialized in the same order as the parameters occur in the memberwise initializer. For example:

```swift
struct S {
  var _x: Int

  var x: Int {
    @storageRestrictions(initializes: _x, accesses: y)
    init(newValue) {
      _x = newValue
    }

    get { _x }
    set { _x = newValue }
  }

  var y: Int 
}
```

If the memberwise initializer of the above struct were written to initialize the properties in the same order as the parameters, it would produce an error:

```swift
init(x: Int, y: Int) {
  self.x = x // error
  self.y = y
}
```

Therefore, the compiler will order the initializations in the synthesized memberwise initializer to respect the `accesses` clauses:

```swift
init(x: Int, y: Int) {
  self.y = y
  self.x = x
}
```

The initial review of this proposal suppressed the memberwise initializer in such cases, based on a concern that out-of-order initialization would cause surprises. However, given the fact that the fields are initialized independently (or have `accessses` relationships that define their relative ordering), and that side effects here are limited to those of the `init` accessors themselves, one has to introduce global side effects during initialization to observe any difference.

There remain cases where a memberwise initializer cannot be synthesized. For example, if a type contains several computed properties with `init` accessors that initialize the same stored property, it is not clear which computed property should be used within the member-wise initializer. In such cases, a member-wise initializer will not be synthesized.

### Init accessors on computed properties

An init accessor can be provided on a computed property, in which case it is used for initialization and as a default argument in the memberwise initializer. For example, given the following:

```swift
struct Angle {
  var degrees: Double
  
  var radians: Double {
    @storageRestrictions(initializes: degrees)
    init(initialValue) {
      degrees = initialValue * 180 / .pi
    }

    get { degrees * .pi / 180 }
    set { degrees = newValue * 180 / .pi }
  }
}
```

The implicit memberwise initializer will contain `radians`, but not the `degrees` stored property that it subsumes:

```swift
init(radians: Double) {
  self.radians = radians // calls init accessor, subsumes initialization of 'degrees'
}
```

### Init accessors for read-only properties

Init accessors can be provided for properties that lack a setter. Such properties act much like a `let` property, able to be initialized (exactly) once and not set thereafter:

```swift
struct S {
  var _x: Int

  var x: Int {
    @storageRestrictions(initializes: _x)
    init(initialValue) {
      self._x = x
    }

    get { _x }
  }

  init(halfOf y: Int) {
    self.x = y / 2 // okay, calls init accessor for x
    self.x = y / 2 // error, 'x' cannot be set
  }
}

```

### Initial values on properties with an init accessor

A property with an init accessor can have an initial value, e.g.,

```swift
struct WithInitialValues {
  var _x: Int

  var x: Int = 0 {
    @storageRestrictions(initializes: _x)
    init(initialValue) {
      _x = initialValue
    }

    get { ... }
    set { ... }
  }

  var y: Int
}
```

The synthesized memberwise initializer will use the initial value as a default argument, so it will look like the following:

```swift
init(x: Int = 0, y: Int) {
  self.x = x  // calls init accessor, which initializes _x
  self.y = y
}
```

In a manually written initializer, the initial value will be used to initialize the property with the init accessor prior to any user-written code:

```swift
init() {
  // implicitly initializes self.x = 0
  self.y = 10
  self.x = 20 // calls setter
}
```

### Restrictions

A property with an `init` accessor can only be declared in the primary
declaration of a type.

## Source compatibility

`init` accessors are an additive capability with new syntax; there is no impact on existing source code.

## ABI compatibility

`init` accessors are an ABI-additive change; they are at most `internal` but can
be ABI-public.
Calling an `init` accessor from an `inlinable` type initializer requires that
the `init` accessor is ABI-public.

## Implications on adoption

Because `init` accessors are always called from within the defining module, adopting `init` accessors is an ABI-compatible change. Adding an `init` accessor to an existing property also cannot have any source compatibility impact outside of the defining module; the only possible source incompatibilities are on the generated memberwise initializer (if new entries are added), or on the type's `init` implementation (if new initialization effects are added).

## Alternatives considered

### Syntax for "initializes" and "accesses"

A number of different syntaxes have been considered for specifying the set of stored properties that are initialized or accessed by a property that has an `init` accessor. The original pitch specified them in the parameter list using special labels:

```swift
struct S {
  var _x: Int
  var x: Int {
    init(newValue,  initializes: _x, accesses: y) {
      _x = newValue
    }

    get { _x }
    set { _x = newValue }
  }

  var y: Int
}
```

This syntax choice is misleading because the effects look like function parameters, while `initializes` behaves more like the output of an init accessor, and `accesses` are not explicitly provided at the call-site. Conceptually, `initializes` and `accesses` are side effects of an `init` accessor, so the proposal was revised to place these modifiers in the effects clause.

The first reviewed version of this proposal placed `initializes` and `accesses` along with other *effects*, e.g.,

```swift
struct S {
  var _x: Int
  var x: Int {
    init(newValue) initializes(_x), accesses(y) {
      _x = newValue
    }

    get { _x }
    set { _x = newValue }
  }

  var y: Int
}
```

However, `initializes` and `effects` don't behave in the same manner as other effects in Swift, such as `throws` and `async`, for several reasons. First, there's no annotation like `try` or `await` at the call site. Second, these aren't part of the type of the entity (e.g., there is not function type that has an `initializes` clause). Therefore, using the effects clause is not a good match for Swift's semantic model.

The current proposal uses an attribute. With attributes, there is question of whether we can remove the `@` to turn it into a declaration modifier:

```swift
struct S {
  var _x: Int
  var x: Int {
    storageRestrictions(initializes: _x, accesses: y)
    init(newValue) {
      _x = newValue
    }

    get { _x }
    set { _x = newValue }
  }

  var y: Int
}
```

This is doable within the confines of this proposal's init accessors, but would prevent further extensions of this proposal that would allow the use of `initializes` or `accesses` on arbitrary functions. For example, such an extension might allow the following

```swift
var _x, _y: Double

storageRestrictions(initializes: _x, _y)
func initCoordinates(radius: Double, angle: Double) { ... }

if let (r, theta) = decodeAsPolar() {
  initCoordinates(radius: r, angle: theta)
} else {
  // ...
}
```

However, there is a parsing ambiguity in the above because `storageRestrictions(initializes: _x, _y)` could be a call to a function names `storageRestrictions(initializes:)` or it could be a declaration modifier specifying that `initCoordinates` initializes `_x` and `_y`. 

Other syntax suggestions from pitch reviewers included:

* Using a capture-list-style clause, e.g. `init { [&x, y] in ... }`
* Using more concise effect names, e.g. `writes` and `reads` instead of `initializes` and `accesses`
* And more!

However, the current syntax in this proposal, which uses an attribute, most accurately models the semantics of initialization effects. An `init` accessor is a function -- not a closure -- that has side-effects related to initialization. _Only_ the `init` accessor has these effects; though the `set` accessor often contains code that looks the same as the code in the `init` accessor, the effects of these accessors are different. Because `init` accessors are called before all of `self` is initialized, they do not recieve a fully-initialized `self` as a parameter like `set` accessors do, and assignments to `initializes` stored properties in `init` accessors have the same semantics as that of a standard initializer, such as suppressing `willSet` and `didSet` observers. 

## Future directions

### `init` accessors for local variables

`init` accessors for local variables have different implications on definite initialization, because re-writing assignment to `init` or `set` is not based on the initialization state of `self`. Local variable getters and setters can also capture any other local variables in scope, which raises more challenges for diagnosing escaping uses before initialization during the same pass where assignments may be re-written to `init` or `set`. As such, local variables with `init` accessors are a future direction.

### Generalization of storage restrictions to other functions

In the future, the `storageRestrictions` attribute could be be generalized to apply to other functions. For example, this could allow one to implement a common initialization function within a class:

```swift
class C {
  var id: String
  var state: State

  @storageRestrictions(initializes: state, accesses: id)
  func initState() {
    self.state = /* initialization code here */
  }

  init(id: String) {
    self.id = id
    initState() // okay, accesses id and initializes state
  }
}
```

The principles are the same as with `init` accessors: a function's implementation can be restricted to only access certain stored properties, and to initialize others along all paths. A call to the function then participates in definite initialization.

This generalization comes with limitations that were not relevant to `init` accessors, because the functions are more akin to fragments of an initializer. For example, the `initState` function cannot be called after `state` is initialized (because it would re-initialize `state`), nor can it be used as a "first-class" function:

```swift
  init(id: String) {
    self.id = id
    initState() // okay, accesses id and initializes state

    initState() // error, 'state' is already initialized
    let fn = self.initState // error: can't treat it like a function value
  }
```

These limitations are severe enough that this future direction would require a significant amount of justification on its own to pursue, and therefore is not part of the `init` accessors proposal.

## Revision history

* Following the initial review:
  * Replaced the "effects" syntax with the `@storageRestrictions` attribute.
  * Add section on init accessors for computed properties.
  * Add section on init accessors for read-only properties.
  * Allow reordering of the initializations in the synthesized memberwise initializer to respect `accesses` restrictions.
  * Add a potential future direction for the generalization of storage restrictions to other functions.
  * Clarify the behavior of properties that have init accessors and initial values.

## Acknowledgments

Thank you to TJ Usiyan, Michel Fortin, and others for suggesting alternative syntax ideas for `init` accessor effects; thank you to Pavel Yaskevich for helping with the implementation.
