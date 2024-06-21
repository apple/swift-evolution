# Opaque result types with limited availability

* Proposal: [SE-0360](0360-opaque-result-types-with-availability.md)
* Authors: [Pavel Yaskevich](https://github.com/xedin)
* Review Manager: [Joe Groff](https://github.com/jckarter)
* Implementation: [apple/swift#42072](https://github.com/apple/swift/pull/42072), [apple/swift#42104](https://github.com/apple/swift/pull/42104), [apple/swift#42167](https://github.com/apple/swift/pull/42167), [apple/swift#42456](https://github.com/apple/swift/pull/42456)
* Status: **Implemented (Swift 5.7)**
* Decision Notes: [Acceptance](https://forums.swift.org/t/accepted-se-0360-opaque-result-types-with-limited-availability/58712)

## Introduction

Since their introduction in [SE-0244](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0244-opaque-result-types.md), opaque result types have become a powerful tool of type-level abstraction that allows library authors to hide implementation details of their APIs.

Under the rules described in SE-0244 - a function returning an opaque result type *must return a value of the same concrete type `T`* from each `return` statement, and `T` must meet all of the constraints stated on the opaque type. 

The same-type `return` requirement is unnecessarily strict when it comes to availability conditions. SE-0244 states that it should be possible to change the underlying type in the future version of the library, but that would only work with pre-existing types. In other words, the same-type condition does not have to apply across executions of the same program, the same way that `Hashable` must produce the same output for the same value during program execution, but may produce a different value in the next execution. `#available` is special because it's a checkable form of that: dynamic availability will not change while the program is running, but may be different the next time the program runs.

Current model and implementation limit usefulness of opaque result types as an abstraction mechanism, because it prevents frameworks from introducing new types and using them as underlying types in existing APIs. To bridge this usability gap, I propose to relax same-type restriction for `return`s inside of availability conditions.

Swift-evolution thread: [
[Pitch] Opaque result types with limited availability](https://forums.swift.org/t/pitch-opaque-result-types-with-limited-availability/57286)

## Motivation

To illustrate the problematic interaction between opaque result types and availability conditions, let's consider a framework that already has a `Shape` protocol and a `Square` type that conforms to the `Shape` protocol. 

```
protocol Shape {
  func draw(to: Surface)
}

struct Square : Shape {
  ...
}
```

In a new version of the framework, the library authors decided to introduce a new shape - `Rectangle` with limited availability:

```
@available(macOS 100, *)
struct Rectangle : Shape {
  ...
}
```

Since a `Rectangle` is generalization of a `Square` it makes sense to allow transforming a `Square` into a `Rectangle` but that currently requires extension with limited availability:

```
@available(macOS 100, *)
extension Square {
  func asRectangle() -> some Shape {
     return Rectangle(...)
  }
}
```

The fact that the new method has to be declared in availability context to return `Rectangle` limits its usefulness because all uses of `asRectangle` would have to be encapsulated into `if #available` blocks.

If `asRectangle` already existed in the original version of the framework, it wouldn’t be possible to use a new type at all without declaring `if #available` block in its body:

```
struct Square {
  func asRectangle() -> some Shape {
     if #available(macOS 100, *) {
        return Rectangle(...)
     }
     
     return self
  }
}
```

But doing so is not allowed because all of the `return` statements in the body of the `asRectangle` function have to return the same concrete type:

```
 error: function declares an opaque return type 'some Shape', but the return statements in its body do not have matching underlying types
  func asRectangle() -> some Shape {
       ^                ~~~~~~~~~~
note: return statement has underlying type 'Rectangle'
      return Rectangle()
             ^
note: return statement has underlying type 'Square'
    return Square()
           ^
```

This is a dead-end for the library author although SE-0244 states that it should be possible to change underlying result type in the future version of the library/framework but that assumes that the type already exists so it could be used in all `return` statements.

## Proposed solution

To bridge this usability gap, I propose to relax the same-type restriction for functions with `if #available` conditions: if an `if #available` condition is always executed, it can return a different type than the type returned by the rest of the function.

The proposed changes allow functions to:

* use multiple `if #available` conditions to return different types based on their dynamic availability and
* safely fall back to a return type with no availability restrictions if none of the availability conditions are met.

Because the return type must be decidable without running the code in the function, mixing availability conditions with other conditions (such as `if`, `guard`, or `switch`) removes this special power and requires `return`s in the `if #available` to return the same type as the rest of the function.

This example satisfies these rules:

```swift
func test() -> some Shape {
  if #available(macOS 100, *) { ✅
    return Rectangle()
  }

  return self
}
```

## Detailed design

An *unconditional availability clause* is an `if` or `else if` clause that satisfies the following conditions:

  - The clause is part of an `if` statement at the top level of the containing function.
  - There are no `return` statements in the containing function prior to the `if` statement.
  - The condition of the clause is an `#available` condition.
  - The clause is either the initial `if` clause or an `else if` clause immediately following an unconditional availability clause.
  - The clause contains at least one `return` statement.
  - All paths through the block controlled by the clause terminate by either returning or throwing.

All `return` statements outside of unconditional availability clauses must return the same type as each other, and this type must be as available as the containing function.

All `return` statements within a given unconditional availability clause must return the same type as each other, and this type must be as available as the `#available` condition of the clause. This type need not be the same type returned by any `return` statement outside of the clause.

There must be at least one `return` statement in the containing function.  If there are no `return` statements outside of unconditional availability clauses, then at least one of the return types within unconditional availability clauses must be as available as the containing function.

Dynamically, the return type of the containing function is:
  - the return type of `return` statements in the first unconditional availability clause whose condition is dynamically satisfied, or if none are satisfied then
  - the return type of `return` statements outside of all unconditional availability clauses, or if there are no such statements then
  - the return type of `return` statements in the first unconditional availability clause that is as available as the containing function.
 
Now let's consider a couple of examples to better demonstrate the difference between well-formed and invalid functions under the proposed rules.

The following example is well-formed because the first `if #available` statement terminates with a `return` and the second one is associated with a valid `if #available` and also terminates with a `return`.

  ```swift
  func test() -> some Shape {
    if #available(macOS 100, *) { ✅
      return Rectangle()
    } else if #available(macOS 99, *) { ✅
      return Square()
    }
    return self
  }
  ```

  But

  ```swift
  func test() -> some Shape {
    if cond {
      ...
    } else if #available(macOS 100, *) { ❌
      return Rectangle()
    }
    return self
  }
  ```

is not accepted by the compiler because `if #available` associated with a dynamic condition.
  
The following is incorrect because `if #available` is preceded by a dynamic condition that returns:
  
```swift
func test() -> some Shape {
  guard let x = <opt-value> else {
    return ...
  }
    
  if #available(macOS 100, *) { ❌
    return Rectangle()
  }

  return self
}
```

Similarly, the following is incorrect because `if #available` appears inside of a loop:

```swift
func test() -> some Shape {
  for ... {
    if #available(macOS 100, *) { ❌
      return Rectangle()
    }
  }
  return self
}
```

The following `test()` function is well-formed because `if` statement produces the same result in both of its branches and it's statically known that the `if #available` always terminates with a `return`
  
  ```swift
  func test() -> some Shape {
    if #available(macOS 100, *) {
       if cond { ✅
         return Rectangle(...)
       } else {
         return Rectangle(...)
       }
    }
    return self
  }
  ```

  But:

  ```swift
  func test() -> some Shape {
    if #available(macOS 100, *) {
       if cond { ❌
         return Rectangle()
       } else {
         return Square()
       }
    }
    return self
  }
  ```

is not going to be accepted by the compiler because return types are different: `Rectangle` vs. `Square`.

This semantic adjustment fits well into the existing model because it makes sure that there is always a single underlying type per platform and universally.

## Source compatibility

Proposed changes do not break source compatibility and allow previously incorrect code to compile.

## Effect on ABI stability

No ABI impact since this is an additive change.

## Effect on API resilience

All of the resilience rules associated with opaque result types are preserved.

## Alternatives considered

* Only alternative is to change the API patterns used in the library, e.g. by exposing the underlying result type and overloading the method.

## Acknowledgments

[John McCall](https://forums.swift.org/u/john_mccall) for the help with `Proposed Solution` and `Detail Design` improvements.
