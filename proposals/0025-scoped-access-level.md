# Scoped Access Level

* Proposal: [SE-0025](0025-scoped-access-level.md)
* Author: Ilya Belenkiy
* Status: **Implemented (Swift 3.0)**
* Review Manager: [Doug Gregor](https://github.com/DougGregor)
* Decision Notes: [Rationale](https://forums.swift.org/t/se-0025-scoped-access-level-next-steps/1797/131)
* Bug: [SR-1275](https://bugs.swift.org/browse/SR-1275)
* Previous revision: [1](https://github.com/swiftlang/swift-evolution/blob/e4328889a9643100177aef19f6f428855c5d0cf2/proposals/0025-scoped-access-level.md)

## Introduction

Scoped access level allows hiding implementation details of a class or a class extension at the class/extension level, instead of a file. It is a concise expression of the intent that a particular part of a class or extension definition is there only to implement a public API for other classes or extensions and must not be used directly anywhere outside of the scope of the class or the extension.

[Swift Evolution Discussion](https://forums.swift.org/t/review-se-0025-scoped-access-level/1588), [Next Steps Discussion](https://forums.swift.org/t/se-0025-scoped-access-level-next-steps/1797)

## Motivation

Currently, the only reliable way to hide implementation details of a class is to put the code in a separate file and mark it as private. This is not ideal for the following reasons:

- It is not clear whether the implementation details are meant to be completely hidden or can be shared with some related code without the danger of misusing the APIs marked as private. If a file already has multiple classes, it is not clear if a particular API is meant to be hidden completely or can be shared with the other classes.

- It forces a one class per file structure, which is very limiting. Putting related APIs and/or related implementations in the same file helps ensure consistency and reduces the time to find a particular API or implementation. This does not mean that the classes in the same file need to share otherwise hidden APIs, but there is no way to express such sharability with the current access levels.

Another, less reliable, way is to prefix APIs that are meant to be hidden with a `_` or do something similar. That works, but it’s not enforced by the compiler, and those APIs show up in tools like code completion, so the programmer has to filter out the noise — although these tools could quite easily support hiding methods with the `_` prefix standard. Also, there is a greater danger of using private APIs if they do something similar to public APIs but are somehow more optimized (because they make additional assumptions about the internal state).

The existing solutions are in some ways similar to those for untyped collections. It is usually possible to give a collection a name that would imply the type of elements it holds (similar to using _ to indicate private), but it is not the same as specifying it explicitly. Just as with generics, the intent not to share the implementation details with any other class is much clearer with support from the language as opposed to relying on where the code is in the project. Also, with untyped collections, it is possible to add an element of a different type (deliberately or not). Generics make that impossible, and it’s enforced by the compiler. Similarly, a dedicated access level modifier could enforce hiding implementation details at the compiler level and make it impossible to accidentally misuse or (deliberately use) implementation details in a context that the class meant not to share.

## Proposed solution

Add another access level modifier that is meant to express that the API is visible only within the scope in which it is defined. Properties, functions, and nested types marked this way would be completely hidden outside the class or class extension definition.

After the first review, the core team decided that it would be best to use `private` for this access level and rename other access level modifiers for consistency. The most popular set of names is:

- public: symbol visible outside the current module
- internal: symbol visible within the current module
- fileprivate: symbol visible within the current file
- private: symbol visible within the current declaration

(names proposed by Chris Lattner as an adjustment from names proposed by James Berry)

## Detailed design

When a function, variable, constant, subscript, or initializer is defined with `private` access modifier, it is visible only within that lexical scope. For example:

```swift
class A {
   private var counter = 0

   // public API that hides the internal state
   func incrementCount() { ++counter }

   // hidden API, not visible outside of this lexical scope
   private func advanceCount(dx: Int) { counter += dx }

   // incrementTwice() is not visible here
}

extension A {
   // counter is not visible here
   // advanceCount() is not visible here

   // may be useful only to implement some other methods of the extension
   // hidden from anywhere else, so incrementTwice() doesn’t show up in 
   // code completion outside of this extension
   private func incrementTwice() {
      incrementCount()
      incrementCount()
   }
}
```

### Complications with private types

When a type is defined with the `private` access modifier, things become a little more complicated. Of course the type itself is visible only within the lexical scope it is defined in, but what about members of the type?

```swift
class Outer {
  private class Inner {
    var value = 0
  }

  func test() {
    // Can Outer.test reference Inner's initializer?
    let inner = Inner()
    // Can Outer.test reference Inner's 'value' property?
    print(inner.value)
  }
}
```

If the members of a private type are themselves considered `private`, it is very clear that they cannot be used outside of the type itself. However, it is also not currently permitted for a member to have an access level greater than its enclosing type. This restriction produces a conundrum: the type can be referenced within its enclosing lexical scope, but none of its members can.

Ignoring formal concerns, the most likely expected behavior is that members not explicitly marked `private` are permitted to be accessed within the enclosing scope of the private type. To achieve this goal, we relax a few of the existing rules:

- The default level of access control anywhere is `internal`.

- The compiler should not warn when a broader level of access control is used within a type with more restrictive access, such as `internal` within a `private` type. This allows the designer of the type to select the access they would use were they to make the type more widely accessible. (The members still cannot be accessed outside the enclosing lexical scope because the type itself is still restricted, i.e. outside code will never encounter a value of that type.)

- The type of a member can reference only declarations that are accessible wherever the member is accessible. (This relaxes an existing rule that states that the type of a declaration may not reference any declarations that have broader access.) The change permits the following code:

  ```swift
  struct Outer {
    private typealias Value = Int
    private struct Inner {
      var value: Value
    }
  }
  ```

  and continues to treat this code as illegal:

  ```swift
  struct Outer {
    private struct Inner {
      private typealias Value = Int
      var value: Value
    }
  }
  ```

- A member that satisfies a protocol requirement may never be `private`. Similarly, a `required` initializer may never be `private`.

- As before, an extension with an explicit access modifier overrides the default `internal` access by specifying a default *scope*. Therefore, within an extension marked `private`, the default access level is `fileprivate` (since extensions are always declared at file scope). This matches the behavior of types declared `private` at file scope.
  
- As before, an explicit access modifier on an extension sets the maximum allowed access within that extension, and the compiler will warn on overly broad access within an extension that has an explicit access modifier.

## Impact on existing code

Existing code will need to rename `private` to `fileprivate` to achieve the same semantics, although in many cases the new meaning of `private` is likely to still compile and to run exactly as before.

## Alternatives considered

1. Do nothing and use `_` and `/` or split the code into more files and use the `private` modifier. The proposed solution makes the intent much clearer and enforced by the compiler, and the language does not dictate how the code must be organized.

2. Introduce a scoped namespace that would make it possible to hide APIs in part of the file. This introduces an extra level of grouping and nesting and forces APIs to be grouped by access level instead of a logical way that may make more sense.

3. Introduce a different access modifier and keep the current names unchanged. The proposal originally followed this approach to be completely compatible with the existing code, but the core team decided that it was better to use `private` for this modifier because it’s much closer to what the term means in other languages.

### Alternatives considered for "the private type issue"

1. Use `fileprivate` rather than `internal` as the default access level within `private` and `fileprivate` types. This is a more narrow change from the original model, but didn't have any benefits once we determined that the warning for unnecessarily broad access wasn't useful.

2. Introduce a new "parent" access level that declares an entity to be accessible within the *parent* lexical scope, rather than the immediately enclosing scope. This idea seems effective for `private` but is overly specific within types with any broader access and not worth the added complexity. We would also have to determine its name within the language, or decide that this level of access could not be spelled explicitly and was available only as the default access within private types.

3. Introduce a new "default" access level that names the default access within a scope. Within a `private` type, this would have the "parent" semantics from (2); elsewhere it would follow the rules laid down in previous versions of Swift. This idea likewise added complexity to the model for only a small gain in expressivity, and we would likewise have to determine a name for it within the language.

## Changes from revision 1

- The proposal was amended post-acceptance by [Robert Widmann][] and [Jordan Rose][] to account for "[the private type issue](#complications-with-private-types)". Only that section was added; there were no semantic changes to the rest of the proposal. This amendment requires a small amount of work to implement compared to the [alternatives considered](#alternatives-considered-for-the-private-type-issue), and was determined by the Core Team to be a small enough set of changes in the spirit of the original proposal that a full review was not necessary.

[Robert Widmann]: https://github.com/CodaFi
[Jordan Rose]: https://github.com/jrose-apple
