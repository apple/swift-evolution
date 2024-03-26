# Add `with` method for modifying values within a single expression

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [aggie33](https://github.com/aggie33), [Cal Stephens](https://github.com/calda)
* Review Manager: TBD
* Status: **Awaiting review**
* Implementation: [apple/swift#67768](https://github.com/apple/swift/pull/67768)
* Review: [Pitch](https://forums.swift.org/t/pitch-with-functions-in-the-standard-library/65716)

## Introduction

We should add a `with(_:)` method on all types to allow for easier modification of values within a single expression.

## Motivation

It is common for Swift APIs to require you to create a value, modify various properties on the value, and then either return it or use it in some other expression. This typically requires you to declare it as a mutable `var` instead of a constant, even if this "configuration" stage is the only time you modify the value.

```swift
var components = URLComponents()
components.scheme = "https"
components.host = "forums.swift.org"
components.path = "/c/evolution"

navigate(to: components.url)
```

One approach for avoiding the mutable variable is using a closure which is called immediately. This adds a bit of extra boilerplate, but removes the unwanted mutable property.

```swift
let components = {
  var components = URLComponents()
  components.scheme = "https"
  components.host = "forums.swift.org"
  components.path = "/c/evolution"
  return components
}()

navigate(to: components.url)
```

Since this uses a single expression to both create and customize the value, it has the benefit of being usable in-line within other expressions. The extra boilerplate here is particularly noticable.

```swift
navigate(to: {
  var components = URLComponents()
  components.scheme = "https"
  components.host = "forums.swift.org"
  components.path = "/c/evolution"
  return components
}().url)
```

Also, in some APIs, like in SwiftUI, it is common to want to make a simple modification to a copy of a value, and then return the copy. These functions would make this easier. Currently, you have to write code like this.

```swift
extension FooView { 
  func bar() -> some View {
    var copy = self
    copy.bar = true
    return copy
  }
}
```

This might not seem too difficult, but it can become tedious and repetitive. In a component with lots of different modifiers, this pattern has to be repeated for each individual modifier:

```swift
// For each modifier method, you need to repeat the same copying boilerplate.
extension CustomButtonView {
  func role(_ role: UIButton.Role) -> MyButtonView {
    var copy = self
    copy.role = role
    return copy
  }
  
  func behavioralStyle(_ style: UIBehavioralStyle) -> MyButtonView {
    var copy = self
    copy.behavioralStyle = style
    return copy
  }
}
```

This is tedious, and it hides what the method is actually doing. Two-thirds of each method is the same copying boilerplate. 

## Proposed solution

We propose adding a `with` method to all types, which let you modify values within a single expression. The above examples can be simplified by adopting this method:

``` swift
let components = URLComponents().with { components in
  components.scheme = "https"
  components.host = "forums.swift.org"
  components.path = "/c/evolution"
}

navigate(to: components.url)

// or:

navigate(to: URLComponents().with { components in
  components.scheme = "https"
  components.host = "forums.swift.org"
  component.path = "/c/evolution"
}.url)
```

```swift
extension FooView {
  func bar() -> some View {
    self.with { $0.bar = true }
  }
}

extension CustomButtonView {
  func role(_ role: UIButton.Role) -> SwiftyButton {
    self.with { $0.role = role }
  }

  func behavioralStyle(_ style: UIBehavioralStyle) -> SwiftyButton {
    self.with { $0.behavioralStyle = style } 
  }
}
```

Since the `with` method takes a closure, you aren't limited to just setting properties. It supports the full range of functionality supported elsewhere in the language:

```swift
// Using control flow to conditionally set certain properties:
let components = URLComponents().with { components in
  components.scheme = "https"
  components.host = "forums.swift.org"
  components.path = "/c/evolution/18"

  if let credentials {
    components.user = credentials.username
    components.password = credentials.password
  }
}
```

```swift
// Accessing current values when determining the updated value:
let originalComponents = makeURLComponents()

let updatedComponents1 = originalComponents.with { components in
  components.port? += 1
}

let updatedComponents2= originalComponents.with { components in
  if (components.queryItems ?? []).isEmpty {
    components.queryItems = [defaultQueryItem]
  }
}
```


## Detailed design

We would introduce a `with` method, available on all types, with both a synchronous and asynchronous overload.

It is not currently possible to add methods to all types (e.g. in an `extension Any { ... }`), but the end result should be functionally identical to adding the below theoretical extension to the standard library:

```swift
extension Any {
  /// Makes a copy of `self` and invokes `transform` on it, then returns the modified value.
  @inlinable
  @_disfavoredOverload
  public func with(_ transform: (inout Self) throws -> Void) rethrows -> Self {
    var copy = self
    try transform(&copy)
    return copy
  }

  /// Makes a copy of `self` and invokes `transform` on it, then returns the modified value.
  @inlinable
  @_disfavoredOverload
  public func with(_ transform: (inout Self) async throws -> Void) async rethrows -> Self {
    var copy = self
    try await transform(&copy)
    return copy
  }
}
```

Since `extension Any` is not expressible in the surface language today, the actual implementation will require changes to the compiler (which are beyond the scope of a proposal review, although discussion is available in the [implementation PR](https://github.com/apple/swift/pull/67768)).

### `@_disfavoredOverload`

`@_disfavoredOverload` is required to maximize source compatibility. 

Since this proposal adds a new method named `with` to all types / values, this new method potentially conflicts with any existing method named `with`. 

There are several examples that would fail to compile if this method were added without being a disfavored overload. For example:

```swift
// An existing type that has a `with` function of the same signature
struct Foo {
  var bar: String?
  
  func with(_ modify: (inout Foo) -> Void) -> Foo {    
    var copy = self
    modify(&copy)
    return copy
  }
}

// Without @_disfavoredOverload, produces an error "ambiguous use of 'with'"
let value = Foo().with { $0.bar = "bar" }
```

```swift
// An existing type that has a `with` function of a different signature
struct Foo {
  var bar: String?
  
  func with(bar: String) -> Foo {    
    var copy = self
    copy.bar = bar
    return copy
  }
}

// Without @_disfavoredOverload, produces an error "ambiguous use of 'with'"
let withFunc = Foo.with

let value = Foo().with(bar: "bar")
```

Annotating the new `with` methods with `@_disfavoredOverload` allows these examples to continue compiling / functioning as they did before. 

`@_disfavoredOverload` could potentially be removed in Swift 6 mode, where a source break could be more acceptable.

### `@dynamicMemberLookup`

`@dynamicMemberLookup` is currently the overload-of-last-resort, only used if there are no other overloads that match the given name. It is possible to create examples that no longer compile after adding the new `with` method:

```swift
@dynamicMemberLookup
struct Foo {
  var bar: String?
  
  subscript(dynamicMember member: String) -> String {
    "dymamic member \(member)"
  }
}

// Today this prints "DYNAMIC MEMBER WITH"
//
// With the new `with` method, by default this would produce an error:
// value of type '((inout Test) -> Void) -> Test' has no member 'uppercased'
print(Foo().with.uppercased())

// Today this prints "DYNAMIC MEMBER WITH"
//
// With the new `with` method, by default this would produce an error: 
// ambiguous use of 'with'
let string = Foo().with
print(string.uppercased())
```

To prevent this source break, we can make it so that the new `with` overload doesn't suppress the dynamic member subscrpt overload when accessing a dynamic member named "with". Combining this with `@_disfavoredOverload` allows the above example to continue compiling / functioning exactly as it did before.

This additional behavior could potentially be removed in Swift 6 mode, where a source break could be acceptable.

## Source compatibility

We are not currently aware of any cases where this results in a source break, since the new method is a disfavored overload.

## ABI compatibility

This is a purely additive change, so it should not affect ABI compatibility.

## Implications on adoption

This feature can be freely adopted and un-adopted in source code with no deployment constraints and without affecting source or ABI compatibility.

## Future directions

### Add `with` overloads that allow you to return a value inside of the closure
This would allow you to more easily use certain mutating methods that return a value.

### Allow arbitrary extensions on `Any`

In the future it may make sense to allow extensions on `Any`, so that similar helpers can be added to all types / values without requiring compiler support. This is not required for this specific proposal, however.

If `Any` extensions are permitted in the future, most of the custom implementation in the compiler could be removed in favor of a "real" extension in the standard library. 

### Permit source breaks in Swift 6

`@_disfavoredOverload` and support for `@dynamicMemberLookup` are included only to avoid source breaks in Swift 5 mode. It could be reasonable to remove this in Swift 6 mode, and instead accept source breaks in these cases.

## Alternatives considered

### Introduce a `with` free function

It is not currently possible to implement this proposal purely using Swift code (e.g. in the standard library, or in an extension in your own project). One alternative design that is possible today without any additional compiler support is a introducing a free function named `with`. In fact, this was proposed as long ago as [2016](https://github.com/beccadax/swift-evolution/blob/with-function/proposals/NNNN-introducing-with-to-stdlib.md).

```swift
public func with<T>(_ value: T, transform: (inout T) throws -> Void) rethrows -> T {
    var copy = value
    try transform(&copy)
    return copy
}
```

Free functions are less discoverable, less idiomatic, and result in less fluent callsites:

```swift
with(URLComponents()) {
  $0.scheme = "https"
  $0.host = "forums.swift.org"
  $0.path = "/c/evolution"
}.url
```

```swift
extension FooView {
  func bar() -> some View {
    with(self) { $0.bar = true }
  }
}
```

### Use an operator instead of `with`.

Another option is to introduce an operator with the same functionality. For example:

```swift
infix operator |>

public func |> <T>(_ value: T, transform: (inout T) throws -> Void) rethrows -> T {
    var copy = value
    try transform(&copy)
    return copy
}

URLComponents() |> {
  $0.scheme = "https"
  $0.host = "forums.swift.org"
  $0.path = "/c/evolution/18"
}
```

One very tangible downside of using an operator is that any sequential property / method access on the result would require using parenthesis. For example:

```swift
// error: value of type '(_) -> ()' has no member 'url'
let url1 = URLComponents() |> {
  $0.scheme = "https"
  $0.host = "forums.swift.org"
  $0.path = "/c/evolution/18"
}.url

// requires parens:
let url2 = (URLComponents() |> {
  $0.scheme = "https"
  $0.host = "forums.swift.org"
  $0.path = "/c/evolution/18"
}).url
```

Introducing new operators to the standard library also requires meeting a very high bar. Operators are maximally terse, but can be difficult to understand if you aren't familiar with the specific symbol yet. Swift typically prefers using established operators from the C family of languages, and there isn't an obvious existing precedent to follow for this operation.

### Builder pattern

Another common approach for modifying values in a single expression, especially in SwiftUI, is the builder pattern. If applied to the above examples, this could look like:

```swift
let components = URLComponents()
  .scheme("https")
  .host("forums.swift.org")
  .path("/c/evolution")
```

We could conceivably automatically support this pattern for all propeties, by synthesizing implicit functions of the form:

```swift
func scheme(_ value: String) -> Self {
  var copy = self
  copy.scheme = value
  return copy
}
```

While this pattern is visually pleasing for simple cases, it is less expressive than a `with` function that takes a closure. There are many types of modifications that are simple when using a `with` closure but are not available if only using an altnertive synthesized builder pattern.

In the builder pattern it is more difficult to express conditional logic:

```swift
// Using the with method:
let components = URLComponents().with { components in
  components.scheme = "https"
  components.host = "forums.swift.org"
  components.path = "/c/evolution/18"

  if let credentials {
    components.user = credentials.username
    components.password = credentials.password
  }
}

// Using the builder pattern:
var components = URLComponents()
  .scheme("https")
  .host("forums.swift.org")
  .path("/c/evolution")

if let credentials {
  components = components
    .user(credentials.username)
    .password(credentials.password)
}
```

and you can't easily reference existing values of properties being modified:

```swift
let originalComponents = makeURLComponents()

// Using the with method:
let updatedComponents = originalComponents.with { components in
  components.port? += 1
}

// Using the builder pattern:
let updatedComponents = 
  if let originalPort = originalComponents.port {
    originalComponents.port(originalPort + 1)
  } else {
    originalComponents
  }
```

## Acknowledgments

Thank you to everyone who participated in the [pitch thread](https://forums.swift.org/t/pitch-with-functions-in-the-standard-library/65716)!
