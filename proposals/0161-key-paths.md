# Smart KeyPaths: Better Key-Value Coding for Swift

* Proposal: [SE-0161](0161-key-paths.md)
* Authors: [David Smith](https://github.com/Catfish-Man), [Michael LeHew](https://github.com/mlehew), [Joe Groff](https://github.com/jckarter)
* Review Manager: [Doug Gregor](https://github.com/DougGregor)
* Status: **Implemented (Swift 4.0)**
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-se-0161-smart-keypaths-better-key-value-coding-for-swift/5690)
* Previous Revision: [1](https://github.com/swiftlang/swift-evolution/blob/55e61f459632eca2face40e571a517919f846cfb/proposals/0161-key-paths.md)

## Introduction
We propose a family of concrete _Key Path_ types that represent uninvoked references to properties that can be composed to form paths through many values and directly get/set their underlying values.

## Motivation
#### We Can Do Better than String
On Darwin platforms Swift's existing `#keyPath()` syntax provides a convenient way to safely *refer* to properties. Unfortunately, once validated, the expression becomes a `String` which has a number of important limitations:

* Loss of type information (requiring awkward `Any` APIs)
* Unnecessarily slow to parse
* Only applicable to `NSObjects` 
* Limited to Darwin platforms

#### Use/Mention Distinctions
While methods can be referred to without invoking them (`let x = foo.bar` instead of  `let x = foo.bar()`), this is not currently possible for properties and subscripts.

Making indirect references to a properties' concrete types also lets us expose metadata about the property, and in the future additional behaviors.

#### More Expressive KeyPaths
We would also like to support being able to use _Key Paths_ to access into collections and other subscriptable types, which is not currently possible.

## Proposed solution
We propose introducing a new expression similar to function type references (e.g. `Type.method`), but for properties and subscripts.  To avoid ambiguities with type properties, we propose we escape such expressions using `\` to indicate that you are talking about the property, not its invocation. A key path expression takes the general form `\<Type>.<path>`, where `<Type>` is a type name, and `<path>` is a chain of one or more property, subscript, or optional chaining/forcing operators. If the type name can be inferred from context, then it can be elided, leaving `\.<path>`.

These property reference expressions produce `KeyPath` objects, rather than `Strings`. `KeyPaths` are a family of generic classes _(structs and protocols here would be ideal, but requires generalized existentials)_ which encapsulate a property reference or chain of property references, including the type, mutability, property name(s), and ability to set/get values.

Here's a sample of it in use:

```swift
class Person {
    var name: String
    var friends: [Person] = []
    var bestFriend: Person? = nil
    init(name: String) {
        self.name = name
    }
}

var han = Person(name: "Han Solo")
var luke = Person(name: "Luke Skywalker")
luke.friends.append(han)

// create a key path and use it
let firstFriendsNameKeyPath = \Person.friends[0].name
let firstFriend = luke[keyPath: firstFriendsNameKeyPath] // "Han Solo"

// or equivalently, with type inferred from context
luke[keyPath: \.friends[0].name] // "Han Solo"
// The path must always begin with a dot, even if it starts with a
// subscript component
luke.friends[keyPath: \.[0].name] // "Han Solo"
luke.friends[keyPath: \[Person].[0].name] // "Han Solo"

// rename Luke's first friend
luke[keyPath: firstFriendsNameKeyPath] = "A Disreputable Smuggler"

// optional properties work too
let bestFriendsNameKeyPath = \Person.bestFriend?.name
let bestFriendsName = luke[keyPath: bestFriendsNameKeyPath]  // nil, if he is the last Jedi
```

## Detailed design
### Core KeyPath Types
`KeyPaths` are a hierarchy of progressively more specific classes, based on whether we have prior knowledge of the path through the object graph we wish to traverse. 

##### Unknown Path / Unknown Root Type
`AnyKeyPath` is fully type-erased, referring to 'any route' through an object/value graph for 'any root'.  Because of type-erasure many operations can fail at runtime and are thus optional. 

```swift
class AnyKeyPath: CustomDebugStringConvertible, Hashable {
    // MARK - Composition
    // Returns nil if path.rootType != self.valueType
    func appending(path: AnyKeyPath) -> AnyKeyPath?
    
    // MARK - Runtime Information        
    class var rootType: Any.Type
    class var valueType: Any.Type
    
    static func == (lhs: AnyKeyPath, rhs: AnyKeyPath) -> Bool
    var hashValue: Int
}
```
##### Unknown Path / Known Root Type
If we know a little more type information (what kind of thing the key path is relative to), then we can use `PartialKeyPath<Root>`, which refers to an 'any route' from a known root:

```swift
class PartialKeyPath<Root>: AnyKeyPath {
    // MARK - Composition
    // Returns nil if Value != self.valueType
    func appending(path: AnyKeyPath) -> PartialKeyPath<Root>?
    func appending<Value, AppendedValue>(path: KeyPath<Value, AppendedValue>) -> KeyPath<Root, AppendedValue>?
    func appending<Value, AppendedValue>(path: ReferenceKeyPath<Value, AppendedValue>) -> ReferenceKeyPath<Root, AppendedValue>?
}
```

##### Known Path / Known Root Type
When we know both what the path is relative to and what it refers to, we can use `KeyPath<Root, Value>`.  Thanks to the knowledge of the `Root` and `Value` types, all of the failable operations lose their `Optional`.  

```swift
public class KeyPath<Root, Value>: PartialKeyPath<Root> {
    // MARK - Composition
    func appending<AppendedValue>(path: KeyPath<Value, AppendedValue>) -> KeyPath<Root, AppendedValue>
    func appending<AppendedValue>(path: WritableKeyPath<Value, AppendedValue>) -> Self
    func appending<AppendedValue>(path: ReferenceWritableKeyPath<Value, AppendedValue>) -> ReferenceWritableKeyPath<Root, AppendedValue>
}
```

##### Value/Reference Mutation Semantics Mutation
Finally, we have a pair of subclasses encapsulating value/reference mutation semantics. These have to be distinct because mutating a copy of a value is not very useful, so we need to mutate an inout value.

```swift
class WritableKeyPath<Root, Value>: KeyPath<Root, Value> {
    // MARK - Composition
    func appending<AppendedPathValue>(path: WritableKeyPath<Value, AppendedPathValue>) -> WritableKeyPath<Root, AppendedPathValue>
}

class ReferenceWritableKeyPath<Root, Value>: WritableKeyPath<Root, Value> {
    override func appending<AppendedPathValue>(path: WritableKeyPath<Value, AppendedPathValue>) -> ReferenceWritableKeyPath<Root, AppendedPathValue>
}
```

### Access and Mutation Through KeyPaths
To get or set values for a given root and key path we effectively add the following subscripts to all Swift types. 

```swift
extension Any {
    subscript(keyPath path: AnyKeyPath) -> Any? { get }
    subscript<Root: Self>(keyPath path: PartialKeyPath<Root>) -> Any { get }
    subscript<Root: Self, Value>(keyPath path: KeyPath<Root, Value>) -> Value { get }
    subscript<Root: Self, Value>(keyPath path: WritableKeyPath<Root, Value>) -> Value { set, get }
}
```

This allows for code like

```swift
let someKeyPath = ... 
person[keyPath: someKeyPath]
```

which is both appealingly readable, and doesn't require read-modify-write copies (subscripts access `self` inout). Conflicts with existing subscripts are avoided by using a named parameter and generics to only accept key paths with a `Root` of the type in question.

### Referencing Key Paths
Forming a `KeyPath` utilizes a new escape sigil `\`. We feel this best serves our needs of disambiguating from existing `#keyPath` expressions (which will continue to produce `Strings`) and existing type properties.

Optionals are handled via optional-chaining. Multiply dotted expressions are allowed as well, and work just as if they were composed via the `appending` methods on `KeyPath`.

Forming a key path through subscripts (e.g. Array / Dictionary) will have the limitation that the parameter's type(s) must be `Hashable`.  Should the archival and serialization proposal be accepted, we would also like to include `Codable` with an eye towards being able to make key paths `Codable` themselves in the future. 

### Performance
The performance of interacting with a property/subscript via `KeyPaths` should be close to the cost of calling the property directly.

## Source compatibility
This change is additive and there should no affect on existing source. 

## Effect on ABI stability
This feature adds the following requirements to ABI stability: 

- mechanism to access key paths of public properties

We think a protocol-based design would be preferable once the language has sufficient support for generalized existentials to make that ergonomic. By keeping the class hierarchy closed and the concrete implementations private to the implementation it should be tractable to provide compatibility with an open protocol-based design in the future.

## Effect on API resilience
This should not significantly impact API resilience, as it merely provides a new mechanism for operating on existing APIs.

## Alternatives considered

#### More Features
Various drafts of this proposal have included additional features (decomposable key paths, prefix comparisons, support for custom `KeyPath` subclasses, creating a `KeyPath` from a `String` at runtime, `KeyPaths` conforming to `Codable`, bound key paths as a concrete type, etc.).  We anticipate approaching these enhancements additively once the core `KeyPath` functionality is in place. 

#### Spelling
We also explored many different spellings, each with different strengths. We have chosen the current syntax for the clarity and discoverability it provides in practice.

| Case | `#keyPath` | Function Type Reference | Escape |
| --- | --- | --- | --- |
| Fully qualified | `#keyPath(Person, .friends[0].name)` | `Person.friends[0].name` | `\Person.friends[0].name` |
| Type Inferred | `#keyPath(.friends[0].name)` |`Person.friends[0].name`  | `\.friends[0].name` |

While the crispness of the function-type-reference is appealing, it becomes ambiguous when working with type properties.  The escape-sigil variant avoids this, and remains quite readable.

#### Why `\`?
During review many different sigils were considered: 

**No Sigil**: This matches function type references, but suffers from ambiguity with wanting to actually call a type property. Having to type `let foo: KeyPath<Baz, Bar>` while consistent with function type references, really is not that great (even for  function type references). 

**Back Tick**: Borrowing from lisp, back-tick was what we used in initial discussions of this proposal (it was easy to write on a white-board), but it was not chosen because it is hard to type in markdown, and comes dangerously close to conflicting with other parser intrinsics.

**Pound**: We considered `#` as well, and while it is appealing, we'd like to save it for the future. `#` also has a slightly more computational connotation in Swift so far. For instance, `#keyPath` 'identifies if its valid and returns a String', `#available` does the necessary computation to verify availability and yields a boolean. 

**Back Slash**: Where `#` is computational, `\` in Swift has more of a 'behave differently for a moment' connotation, and that seems to fit exactly what we want when forming a key path. 

#### Function Type References
We think the disambiguating benefits of the escape-sigil would greatly benefit function type references, but such considerations are outside the scope of this proposal.
