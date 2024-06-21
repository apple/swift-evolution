# Typed throws

* Proposal: [SE-0413](0413-typed-throws.md)
* Authors: [Jorge Revuelta (@minuscorp)](https://github.com/minuscorp), [Torsten Lehmann](https://github.com/torstenlehmann), [Doug Gregor](https://github.com/DougGregor)
* Review Manager: [Steve Canon](https://github.com/stephentyrone)
* Status: **Accepted**
* Upcoming Feature Flag: `FullTypedThrows`
* Review: [latest pitch](https://forums.swift.org/t/pitch-n-1-typed-throws/67496), [review](https://forums.swift.org/t/se-0413-typed-throws/68507), [acceptance](https://forums.swift.org/t/accepted-se-0413-typed-throws/69099)

## Introduction

Swift's error handling model allows functions and closures marked `throws` to note that they can exit by throwing an error. The error values themselves are always type-erased to `any Error`. This approach encourages errors to be handled generically, and remains a good default for most code. However, there are some places where the type erasure is unfortunate, because it doesn't allow for more precise error typing in narrow places where it is possible and desirable to handle all errors, or where the costs of type erasure are prohibitive.

This proposal introduces the ability to specify that functions and closures only throw errors of a particular concrete type.

## Table of Contents

[Typed throws](#typed-throws)

 * [Introduction](#introduction)
 * [Motivation](#motivation)
    * [Communicates less error information than Result or Task](#communicates-less-error-information-than-result-or-task)
    * [Inability to interconvert throws with Result or Task](#inability-to-interconvert-throws-with-result-or-task)
       * [Approach 1: Chaining Results](#approach-1-chaining-results)
       * [Approach 2: Unwrap/switch/wrap on every chaining/mapping point](#approach-2-unwrapswitchwrap-on-every-chainingmapping-point)
    * [Existential error types incur overhead](#existential-error-types-incur-overhead)
 * [Proposed solution](#proposed-solution)
    * [Specific types in catch blocks](#specific-types-in-catch-blocks)
    * [Throwing any Error or Never](#throwing-any-error-or-never)
    * [An alternative to rethrows](#an-alternative-to-rethrows)
    * [When to use typed throws](#when-to-use-typed-throws)
 * [Detailed design](#detailed-design)
    * [Syntax adjustments](#syntax-adjustments)
       * [Function type](#function-type)
       * [Closure expression](#closure-expression)
       * [Function, initializer, and accessor declarations](#function-initializer-and-accessor-declarations)
       * [Examples](#examples)
    * [Throwing and catching with typed throws](#throwing-and-catching-with-typed-throws)
       * [Throwing within a function that declares a typed error](#throwing-within-a-function-that-declares-a-typed-error)
       * [Catching typed thrown errors](#catching-typed-thrown-errors)
       * [rethrows](#rethrows)
       * [Opaque thrown error types](#opaque-thrown-error-types)
       * [async let](#async-let)
    * [Subtyping rules](#subtyping-rules)
       * [Function conversions](#function-conversions)
       * [Protocol conformance](#protocol-conformance)
       * [Override checking](#override-checking)
    * [Type inference](#type-inference)
       * [Closure thrown type inference](#closure-thrown-type-inference)
       * [Associated type inference](#associated-type-inference)
    * [Standard library adoption](#standard-library-adoption)
       * [Converting between throws and Result](#converting-between-throws-and-result)
       * [Standard library operations that rethrow](#standard-library-operations-that-rethrow)
 * [Source compatibility](#source-compatibility)
 * [Effect on API resilience](#effect-on-api-resilience)
 * [Effect on ABI stability](#effect-on-abi-stability)
 * [Future directions](#future-directions)
    * [Standard library operations that rethrow](#standard-library-operations-that-rethrow)
    * [Concurrency library adoption](#concurrency-library-adoption)
    * [Specific thrown error types for distributed actors](#specific-thrown-error-types-for-distributed-actors)
 * [Alternatives considered](#alternatives-considered)
    * [Thrown error type syntax](#thrown-error-type-syntax)
    * [Multiple thrown error types](#multiple-thrown-error-types)
    * [Treat all uninhabited thrown error types as nonthrowing](#treat-all-uninhabited-thrown-error-types-as-nonthrowing)
    * [Typed rethrows](#typed-rethrows)
 * [Revision history](#revision-history)

## Motivation

Swift is known for being explicit about semantics and using types to communicate constraints that apply to specific APIs. From that perspective, the fact that all thrown errors are of type `any Error` feels like an outlier. However, it reflects the view laid out in the original [error handling rationale](https://github.com/apple/swift/blob/main/docs/ErrorHandlingRationale.md) that errors are generally propagated and rendered, but rarely handled exhaustively, and are prone to changing over time in a way that types are not.

The desire to provide specific thrown error types has come up repeatedly on the Swift forums. Here are just a few of the forum threads calling for some form of typed throws:

* [[Pitch N+1] Typed throws](https://forums.swift.org/t/pitch-n-1-typed-throws/67496)
* [Typed throw functions](https://forums.swift.org/t/typed-throw-functions/38860)
* [Status check: typed throws](https://forums.swift.org/t/status-check-typed-throws/66637)
* [Precise error typing in Swift](https://forums.swift.org/t/precise-error-typing-in-swift/52045)
* [Typed throws](https://forums.swift.org/t/typed-throws/6501)
* [[Pitch\] Typed throws](https://forums.swift.org/t/pitch-typed-throws/5233)
* [Type-annotated throws](https://forums.swift.org/t/type-annotated-throws/3875)
* [Proposal: Allow Type Annotations on Throws](https://forums.swift.org/t/proposal-allow-type-annotations-on-throws/1149)
* [Proposal: Allow Type Annotations on Throws](https://forums.swift.org/t/proposal-allow-type-annotations-on-throws/623)
* [Proposal: Typed throws](https://forums.swift.org/t/proposal-typed-throws/268)
* [Type Inferencing For Error Handling (try catch blocks)](https://forums.swift.org/t/type-inferencing-for-error-handling-try-catch-blocks/117)

In a sense, Swift started down the path toward typed throws with the introduction of the [`Result`](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0235-add-result.md) type in the standard library, which captured a specific thrown error type in its `Failure` parameter. That pattern was replicated in the [`Task` type](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md) and other concurrency APIs. The loss of information between types like `Result` and `Task` and the language's error-handling system provides partial motivation for the introduction of typed throws, and is discussed further below.

Typed throws also provides benefits in places where clients need to exhaustively handle errors. For this to make sense, the set of potential failure conditions must be relatively fixed, either because they come from the same module or package as the clients, or because they come from a library that is effectively standalone and unlikely to evolve to (e.g.) pass through an error from another lower-level library. Typed throws also provides benefits in generic code that will propagate errors from its arguments, but never generate errors itself, as a more flexible alternative to the existing `rethrows`. Finally, typed throws also open up the potential for more efficient code, because they avoid the overhead associated with existential types (`any Error`).

Even with the introduction of typed throws into Swift, the existing (untyped) `throws` remains the better default error-handling mechanism for most Swift code. The section ["When to use typed throws"](#when-to-use-typed-throws) describes the circumstances in which typed throws should be used.

### Communicates less error information than `Result` or `Task`

Assume you have this Error type

```swift
enum CatError: Error {
    case sleeps
    case sitsAtATree
}
```

Compare

```swift
func callCat() -> Result<Cat, CatError>
```

or

```swift
func callFutureCat() -> Task<Cat, CatError>
```

with

```swift
func callCatOrThrow() throws -> Cat
```

`throws` communicates less information about why the cat is not about to come to you.

### Inability to interconvert `throws` with `Result` or `Task`

The fact that`throws` carries less information than `Result` or `Task` means that conversions to `throws` loses type information, which can only be recovered by explicit casting:

```swift
func callAndFeedCat1() -> Result<Cat, CatError> {
    do {
        return Result.success(try callCatOrThrow())
    } catch {
        // won't compile, because error type guarantee is missing in the first place
        return Result.failure(error)
    }
}
```

```swift
func callAndFeedCat2() -> Result<Cat, CatError> {
    do {
        return Result.success(try callCatOrThrow())
    } catch let error as CatError {
        // compiles
        return Result.failure(error)
    } catch {
        // won't compile, because exhaustiveness can't be checked by the compiler
        // so what should we return here?
        return Result.failure(error)
    }
}
```

### `Result` is not the go to replacement for `throws` in imperative languages

Using explicit errors with `Result` has major implications for a code base. Because the exception handling mechanism ("goto catch") is not built into the language (like `throws`), you need to do that on your own, mixing the exception handling mechanism with domain logic.

#### Approach 1: Chaining Results

If you use `Result` in a functional (i.e. monadic) way, you need extensive use of `map`, `flatMap` and similar operators.

Example is taken from [Question/Idea: Improving explicit error handling in Swift (with enum operations) - Using Swift - Swift Forums](https://forums.swift.org/t/question-idea-improving-explicit-error-handling-in-swift-with-enum-operations/35335).

```swift
struct SimpleError: Error {
    let message: String
}

struct User {
    let firstName: String
    let lastName: String
}

func stringResultFromArray(_ array: [String], at index: Int, errorMessage: String) -> Result<String, SimpleError> {
    guard array.indices.contains(index) else { return Result.failure(SimpleError(message: errorMessage)) }
    return Result.success(array[index])
}

func userResultFromStrings(strings: [String]) -> Result<User, SimpleError>  {
    return stringResultFromArray(strings, at: 0, errorMessage: "Missing first name")
        .flatMap { firstName in
            stringResultFromArray(strings, at: 1, errorMessage: "Missing last name")
                .flatMap { lastName in
                    return Result.success(User(firstName: firstName, lastName: lastName))
            }
    }
}
```

That's the functional way of writing exceptions, but Swift does not provide enough functional constructs to handle that comfortably (compare with [Haskell/do notation](https://en.wikibooks.org/wiki/Haskell/do_notation)).

#### Approach 2: Unwrap/switch/wrap on every chaining/mapping point

We can also just unwrap every result by switching over it and wrapping the value or error into a result again.

```swift
func userResultFromStrings(strings: [String]) -> Result<User, SimpleError>  {
    let firstNameResult = stringResultFromArray(strings, at: 0, errorMessage: "Missing first name")
    
    switch firstNameResult {
    case .success(let firstName):
        let lastNameResult = stringResultFromArray(strings, at: 1, errorMessage: "Missing last name")
        
        switch lastNameResult {
        case .success(let lastName):
            return Result.success(User(firstName: firstName, lastName: lastName))
        case .failure(let simpleError):
            return Result.failure(simpleError)
        }
        
    case .failure(let simpleError):
        return Result.failure(simpleError)
    }
}
```

This is even more boilerplate than the first approach, because now we are writing the implementation of the `flatMap` operator over and over again.

### Existential error types incur overhead

Untyped errors have the existential type `any Error`, which incurs some [necessary overhead](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0335-existential-any.md), in code size, heap allocation overhead, and execution performance, due to the need to support values of unknown type. In constrained environments such as those supported by [Embedded Swift](https://forums.swift.org/t/embedded-swift/67057), existential types may not be permitted due to these overheads, making the existing untyped throws mechanism unusable in those environments.


## Proposed solution

In general, we want to add the possibility of using `throws` with a single, specific error type.

```swift
func callCat() throws(CatError) -> Cat {
  if Int.random(in: 0..<24) < 20 {
    throw .sleeps
  }
  // ...
}
```

The function can only throw instances of `CatError`. This provides contextual type information for all throw sites, so we can write `.sleeps` instead of the more verbose `CatError.sleeps` that's needed with untyped throws. Any attempt to throw any other kind of error out of the function will be an error:

```swift
func callCatBadly() throws(CatError) -> Cat {
  throw SimpleError(message: "sleeping")  // error: SimpleError cannot be converted to CatError
}
```

Maintaining specific error types throughout a function is much easier than when using `Result`, because one can use `try` consistently:

```swift
func stringFromArray(_ array: [String], at index: Int, errorMessage: String) throws(SimpleError) -> String {
    guard array.indices.contains(index) else { throw SimpleError(message: errorMessage) }
    return array[index]
}

func userResultFromStrings(strings: [String]) throws(SimpleError) -> User  {
    let firstName = try stringFromArray(strings, at: 0, errorMessage: "Missing first name")
    let lastName = try stringFromArray(strings, at: 1, errorMessage: "Missing last name")
    return User(firstName: firstName, lastName: lastName)
}
```

The error handling mechanism is pushed aside and you can see the domain logic more clearly. 

### Specific types in catch blocks

With typed throws, a throwing function contains the same information about the error type as `Result`, making it easier to convert between the two:

```swift
func callAndFeedCat1() -> Result<Cat, CatError> {
    do {
        return Result.success(try callCat())
    } catch {
        // would compile now, because error is `CatError`
        return Result.failure(error)
    }
}
```

Note that the implicit `error` variable within the catch block is inferred to the concrete type `CatError`; there is no need for the existential `any Error`.

When a `do` statement can throw errors with different concrete types, or involves any calls to functions using untyped throws, the `catch` block will receive a thrown error type of an `any Error` type:

```swift
func callKids() throws(KidError) -> [Kid] { ... }

do {
  try callCat()
  try callKids()
} catch {
  // error has type 'any Error', as it does today
}
```

The caught error type for a `do..catch` statement will be inferred from the various throwing sites within the body of the `do` block. One can explicitly specify this type with a `throws` clause on ` do` block itself, i.e.,

```swift
do throws(CatError) {
  if isDaylight && foodBowl.isEmpty {
    throw .sleeps   // equivalent to CatError.sleeps
  }
  try callCat()
} catch let myError {
   // myError is of type CatError
}
```

When one needs to translate errors of one concrete type to another, use a `do...catch` block around each sequence of calls that produce the same kind of error :

```swift
func firstNameResultFromArray(_ array: [String]) throws(FirstNameError) -> String {
    guard array.indices.contains(0) else { throw FirstNameError() }
    return array[0]
}

func userResultFromStrings(strings: [String]) throws(SimpleError) -> User  {
    do {
        let firstName = try firstNameResultFromArray(strings)
        return User(firstName: firstName, lastName: "")        
    } catch {
        // error is a `FirstNameError`, map it to a `SimpleError`.
        throw SimpleError(message: "Missing first name")
    }
}
```

### Throwing `any Error` or `Never`

Typed throws generalizes over both untyped throws and non-throwing functions. A function specified with `any Error` as its thrown type:

```swift
func throwsAnything() throws(any Error) { ... }
```

is equivalent to untyped throws:

```swift
func throwsAnything() throws { ... }
```

Similarly, a function specified with `Never` as its thrown type:

```swift
func throwsNothing() throws(Never) { ... }
```

is equivalent to a non-throwing function:

```swift
func throwsNothing() { }
```

There is a more general subtyping rule here that says that you can loosen the thrown type, i.e., converting a non-throwing function to a throwing one, or a function that throws a concrete type to one that throws `any Error`. 

### An alternative to `rethrows`

The ability to throw a generic error parameter that might be `Never` allows one to safely express some rethrowing patterns that are otherwise not possible with rethrows. For example, consider a function that semantically rethrows, but needs to do so by going through some code that doesn't throw:

```swift
/// Count number of nodes in the tree that match a particular predicate
func countNodes(in tree: Node, matching predicate: (Node) throws -> Bool) rethrows -> Int {
  class MyNodeVisitor: NodeVisitor {
    var error: (any Error)? = nil
    var count: Int = 0
    var predicate: (Node) throws -> Bool

    init(predicate: @escaping (Node) throws -> Bool) {
      self.predicate = predicate
    }
    
    override func visit(node: Node) {
      do {
        if try predicate(node) {
          count = count + 1
        }
      } catch let localError {
        error = error ?? localError
      } 
    }
  }
  
  return try withoutActuallyEscaping(predicate) { predicate in
    let visitor = MyNodeVisitor(predicate: predicate)
    visitor.visitTree(node)
    if let error = visitor.error {
      throw error // error: is not throwing as a consequence of 'predicate' throwing.
    } else {
      return visitor.count
    }
  }
}
```

Walking through the code, we can convince ourselves that `MyNodeVisitor.error` will only ever be set as a result of the predicate throwing an error, so this code semantically fulfills the contract of `rethrows`. However, the Swift compiler's rethrows checking cannot perform such an analysis, so it will reject this function. The limitation on `rethrows` has prompted at least [two](https://forums.swift.org/t/pitch-rethrows-unchecked/10078) [pitches](https://forums.swift.org/t/pitch-fix-rethrows-checking-and-add-rethrows-unsafe/44863) to add an "unsafe" or "unchecked" rethrows variant, turning this into a runtime-checked contract. 

Typed throws offer a compelling alternative: one can capture the error type of the closure argument in a generic parameter, and use that consistently throughout. This is immediately useful for maintaining precise typed error information in generic code that  only rethrows the error from its closure arguments, like `map`:

```swift
extension Collection {
  func map<U, E: Error>(body: (Element) throws(E) -> U) throws(E) -> [U] {
    var result: [U] = []
    for element in self {
      result.append(try body(element))
    }
    return result
  }
}
```

When given a closure that throws `CatError`, this formulation of `map` will throw `CatError`. When given a closure that doesn't throw, `E` will be `Never`, so `map` is non-throwing.

This approach extends to our `countNodes` example:

```swift
/// Count number of nodes in the tree that match a particular predicate
func countNodes<E: Error>(in tree: Node, matching predicate: (Node) throws(E) -> Bool) throws(E) -> Int {
  class MyNodeVisitor<E>: NodeVisitor {
    var error: E? = nil
    var count: Int = 0
    var predicate: (Node) throws(E) -> Bool

    init(predicate: @escaping (Node) throws(E) -> Bool) {
      self.predicate = predicate
    }
    
    override func visit(node: Node) {
      do {
        if try predicate(node) {
          count = count + 1
        }
      } catch let localError {
        error = error ?? localError // okay, error has type E?, localError has type E
      } 
    }
  }
  
  return try withoutActuallyEscaping(predicate) { predicate in
    let visitor = MyNodeVisitor(predicate: predicate)
    visitor.visitTree(node)
    if let error = visitor.error {
      throw error // okay! error has type E, which can be thrown out of this function
    } else {
      return visitor.count
    }
  }
}
```

Note that typed throws has elegantly solved our problem, because any throwing site that throws a value of type `E` is accepted. When the closure argument doesn't throw, `E` is inferred to `Never`, and (dynamically) no instance of it will ever be created.

### When to use typed throws

Typed throws makes it possible to strictly specify the thrown error type of a function, but doing so constrains the evolution of that function's implementation. Additionally, errors are usually propagated or rendered, but not exhaustively handled, so even with the addition of typed throws to Swift, untyped `throws` is better for most scenarios. Consider typed throws only in the following circumstances:

1. In code that stays within a module or package where you always want to handle the error, so it's purely an implementation detail and it is plausible to handle the error.
2. In generic code that never produces its own errors, but only passes through errors that come from user components. The standard library contains a number of constructs like this, whether they are `rethrows` functions like `map` or are capturing a `Failure` type like in `Task` or `Result`.
3. In dependency-free code that is meant to be used in a constrained environment (e.g., Embedded Swift) or cannot allocate memory, and will only ever produce its own errors.

Resist the temptation to use typed throws because there is only a single kind of error that the implementation can throw. For example, consider an operation that loads bytes from a specified file:

```swift
public func loadBytes(from file: String) async throws(FileSystemError) -> [UInt8]  // should use untyped throws
```

Internally, it is using some file system library that throws a `FileSystemError`, which it then republishes directly. However, the fact that the error was specified to always be a `FileSystemError` may hamper further evolution of this API: for example, it might be reasonable for this API to start supporting loading bytes from other sources (say, a network connection or database) when the file name matches some other schema. However, errors from those other libraries will not be `FileSystemError` instances, which poses a problem for `loadBytes(from:)`: it either needs to translate the errors from other libraries into `FileSystemError` (if that's even possible), or it needs to break its API contract by adopting a more general error type (or untyped `throws`). 

This section will be added to the [Swift API Design Guidelines](https://www.swift.org/documentation/api-design-guidelines/).

## Detailed design

### Syntax adjustments

The [Swift grammar](https://docs.swift.org/swift-book/ReferenceManual/zzSummaryOfTheGrammar.html) is updated wherever there is either `throws` or `rethrows`, to optionally include a thrown type, e.g.,

```
throws-clause -> throws thrown-type(opt)

thrown-type -> '(' type ')'
```

#### Function type

Changing from

```
function-type → attributes(opt) function-type-argument-clause async(opt) throws(opt) -> type
```

to

```
function-type → attributes(opt) function-type-argument-clause async(opt) throws-clause(opt) -> type
```

Examples

```swift
() -> Bool
() throws -> Bool
() throws(CatError) -> Bool
```

#### Closure expression

Changing from

```
closure-signature → capture-list(opt) closure-parameter-clause async(opt) throws(opt) function-result opt in
```

to

```
closure-signature → capture-list(opt) closure-parameter-clause async(opt) throws-clause(opt) function-result opt in
```

Examples

```swift
{ () -> Bool in true }
{ () throws -> Bool in true }
{ () throws(CatError) -> Bool in true }
```


#### Function, initializer, and accessor declarations

Changing from

```
function-signature → parameter-clause async(opt) throws(opt) function-result(opt)
function-signature → parameter-clause async(opt) rethrows(opt) function-result(opt)
initializer-declaration → initializer-head generic-parameter-clause(opt) parameter-clause async(opt) throws(opt)
initializer-declaration → initializer-head generic-parameter-clause(opt) parameter-clause async(opt) throws(opt)
```

to

```
function-signature → parameter-clause async(opt) throws-clause(opt) function-result(opt)
initializer-declaration → initializer-head generic-parameter-clause(opt) parameter-clause async(opt) throws-clause(opt)
```

Note that the current grammar does not account for throwing accessors, although they should receive the same transformation.

#### `do..catch` blocks

The syntax of a `do..catch` block is extended with an optional throw clause:

```
do-statement → do throws-clause(opt) code-block catch-clauses?
```

If a `throws-clause` is present, then there must be at least one `catch-clause`.

#### Examples

```swift
func callCat() -> Cat
func callCat() throws -> Cat
func callCat() throws(CatError)  -> Cat

init()
init() throws
init() throws(CatError)

var value: Success {
  get throws(Failure) { ... }
}
```

### Throwing and catching with typed throws

#### Throwing within a function that declares a typed error

Any function, closure or function type that is marked as `throws` can declare which type the function throws. That type, which is called the *thrown error type*, must conform to the `Error` protocol.

Every uncaught error that can be thrown from the body of the function must be convertible to the thrown error type. This applies to both explicit `throw` statements and any errors thrown by other calls (as indicated by a `try`). For example:

```swift
func throwingTypedErrors() throws(CatError) {
  throw CatError.asleep // okay, type matches
  throw .asleep // okay, can infer contextual type from the thrown error type
  throw KidError() // error: KidError is not convertible to CatError
  
  try callCat() // okay
  try callKids() // error: throws KidError, which is not convertible to CatError
  
  do {
    try callKids() // okay, because this error is caught and suppressed below
  } catch {
    // eat the error
  }
}
```

Because a value of any `Error`-conforming type implicitly converts to `any Error`, this implies that an function declared with untyped `throws` can throw anything:

```swift
func untypedThrows() throws {
  throw CatError.asleep // okay, CatError converts to any Error
  throw KidError() // okay, KidError converts to any Error
  try callCat() // okay, thrown CatError converts to any Error
  try callKids() // okay, thrown KidError converts to any Error
}
```

Therefore, these rules subsume those of untyped throws, and no existing code will change behavior.

Note that the constraint that the thrown error type must conform to `Error` means that one cannot use an existential type such as `any Error & Codable` as the thrown error type:

```swift
// error: any Error & Codable does not conform to Error
func remoteCall(function: String) async throws(any Error & Codable) -> String { ... }
```

The `any Error` existential has [special semantics](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0235-add-result.md#adding-swifterror-self-conformance) that allow it to conform to the `Error` protocol, introduced along with `Result`. A separate language change would be required to allow other existential types to conform to the `Error` protocol.

#### Catching typed thrown errors 

A `do...catch` block is used to catch and process thrown errors. With only untyped errors, the type of the error thrown from inside the `do` block is always `any Error`. In the presence of typed throws, the type of the error thrown from inside the `do` block can either be explicitly specified with a `throws` clause following the `do`, or inferred from the specific throwing sites.

When the `do` block specifies a thrown error type, that error type can be used for inferring the contextual type of `throw` statements. For example:

```swift
do throws(CatError) {
  if isDaytime && foodBowl.isEmpty {
    throw .sleep
  }
} catch {
  // implicit 'error' value has type CatError
}
```

As with other uses of untyped throws, `do throws` is equivalent to `do throws(any Error)`.

When there is no throws clause, the thrown error type is inferred from the body of the `do` block. When all throwing sites within a `do` block produce the same error type (ignoring any that throw `Never`), that error type is used as the type of the thrown error. For example:

```swift
do /*infers throws(CatError)*/ {
  try callCat() // throws CatError
  if something {
    throw CatError.asleep // throws CatError
  }
} catch {
  // implicit 'error' value has type CatError
  if error == .asleep { 
    openFoodCan()
  }
}
```

This also implies that one can use the thrown type context to perform type-specific checks in the catch clauses, e.g.,

```swift
do /*infers throws(CatError)*/ {
  try callCat() // throws CatError
  if something {
    throw CatError.asleep // throws CatError
  }
} catch .asleep {
  openFoodCan()
} // note: CatError can be thrown out of this do...catch block when the cat isn't asleep
```

> **Rationale**: By inferring a concrete result type for the thrown error type, we can entirely avoid having to reason about existential error types within `catch` blocks, leading to a simpler syntax. Additionally, it preserves the notion that a `do...catch` block that has a `catch` site accepting anything (i.e., one with no conditions) can exhaustively suppress all errors. 

When throw sites within the `do` block throw different (non-`Never`) error types, the inferred error type is `any Error`. For example:

```swift
do /*infers throws(any Error)*/ {
  try callCat() // throws CatError
  try callKids() // throw KidError
} catch {
  // implicit 'error' variable has type 'any Error'
}
```

In essence, when there are multiple possible thrown error types, we immediately resolve to the untyped equivalent of `any Error`. We will refer to this notion as a type function `errorUnion(E1, E2, ..., EN)`, which takes `N` different error types (e.g., for throwing sites within a `do` block) and produces the union error type of those types. Our definition and use of `errorUnion`  for typed throws subsumes the existing rule for untyped throws, in which every throw site produces an error of type `any Error`.

> **Rationale**: While it would be possible to compute a more precise "union" type of different error types, doing so is potentially an expensive operation at compile time and run time, as well as being harder for the programmer to reason about. If in the future it becomes important to tighten up the error types, that could be done in a mostly source-compatible manner.

The semantics specified here are not fully source compatible with existing Swift code. A `do...catch` block that contains `throw` statements of a single concrete type (and no other throwing sites) might depend on the error being caught as `any Error`. Here is a contrived example:

```swift
do /*infers throws(CatError) in Swift 6 */ {
  throw CatError.asleep
} catch {
  var e = error   // currently has type any Error, will have type CatError
  e = KidsError() // currently well-formed, will become an error
}
```

> **Swift 6**: To prevent this source compatibility issue, we can refine the rule slightly for Swift 5 code bases to specify that any `throw` statement always throws a value of type `any Error`. That way, one can only get a caught error type more specific than `any Error` when the both of the `do..catch` contains no `throw` statements and all of the `try` operations are using functions that make use of typed throws.

Note that the only way to write an exhaustive `do...catch` statement is to have an unconditional `catch` block. The dynamic checking provided by `is` or `as` patterns in the `catch` block cannot be used to make a catch exhaustive, even if the type specified is the same as the type thrown from the body of the `do`:

```swift
func f() {
  do /*infers throws(CatError)*/ {
    try callCat()
  } catch let ce as CatError {
    
  } // error: do...catch is not exhaustive, so this code rethrows CatError and is ill-formed
}
```

>  **Note**: Exhaustiveness checking in the general is expensive at compile time, and the existing language uses the presence of an unconditional `catch` block as the indicator for an exhaustive `do...catch`. See the section on closure thrown type inference for more details about inferring throwing closures.

#### `rethrows`

A function marked `rethrows` throws only when one of its closure parameters throws. It is typically used with higher-order functions, such as the `map` operation on a collection:

```swift
extension Collection {
  func map<U>(body: (Element) throws -> U) rethrows -> [U] {
    var result: [U] = []
    for element in self {
      result.append(try body(element))
    }
    return result
  }
}
```

When provided with a throwing closure, `map` can throw, and it chooses to directly throw the same error as the body. This contract can be more precisely modeled using typed throws:

```swift
extension Collection {
  func map<U, E: Error>(body: (Element) throws(E) -> U) throws(E) -> [U] {
    var result: [U] = []
    for element in self {
      result.append(try body(element))
    }
    return result
  }
}
```

Now, when `map` is provided with a closure that throws `E`, it can only throw an `E`. For a non-throwing closure, `E` will be `Never` and `map` is non-throwing. For an untyped throwing closure, `E` will be `any Error` and we get the same type-level behavior as the `rethrows` version of `map`.

However, because `rethrows` uses untyped errors, `map` would be permitted to substitute a different error type that, for example, provides more information about the failing element:

```swift
struct MapError<Element>: Error {
  var failedElement: Element
  var underlyingError: any Error
}

extension Collection {
  func map<U>(body: (Element) throws -> U) rethrows -> [U] {
    var result: [U] = []
    for element in self {
      do {
        result.append(try body(element))
      } catch {
        // Provide more information about the failure
        throw MapError(failedElement: element, underlyingError: error)
      }
    }
    return result
  }
}
```

Typed throws, as presented here, is not able to express the contract of this function.

The Swift standard library does not perform error substitution of this form, and its contract for operations like `map` is best expressed by typed throws as shown above. It is likely that many existing `rethrows` functions are better expressed with typed throws. However, not *all* `rethrows` functions can be expressed by typed throws, if they are performing error substitution like this last `map`. 

Therefore, this proposal does not change the primary semantics of `rethrows`: it remains untyped, and it is ill-formed to attempt to provide a thrown error type to a `rethrows` function. The Alternatives Considered section provides several options for `rethrows`, which can become the subject of a future proposal.

However, there is a small change in the type checking behavior of a `rethrows` function to improve source compatibility in certain cases. Specifically, consider a `rethrows` function that calls into a function with typed throws:

```swift
extension Collection {
  func filter<E: Error>(_ isIncluded: (Element) throws(E) -> Bool) throws(E) -> [Element] { ... }

  func filterOdds<E: Error>(_ isIncluded: (Element) throws -> Bool) rethrows -> [Element {
    var onOdd = true
    return try filter { element in
      defer { onOdd = !onOdd }
      return onOdd && isIncluded(element)
    } // error: call to filter isn't "rethrows"
  }
}
```

The standard `rethrows` checking rejects the call to `filter` because, technically, it could throw `any Error` under any circumstances. Unfortunately, this behavior is a source compatibility problem for the standard library's adoption of typed throws, because an existing `rethrows` function calling into something like `map` or `filter` would be rejected once those introduce typed throws. This proposal introduces a small compatibility feature that considers a function that

1. Has a thrown error type that is a generic parameter (call it `E`) of the function itself,
2. Has no protocol requirements on `E` other than that it conform to the `Error` protocol, and
3. Any parameters of throwing function type throw the specific error type `E`.

to be a rethrowing function for the purposes of `rethrows` checking in its caller. This compatibility feature introduces a small soundness hole in `rethrows` functions, so it is temporary: it is only available in Swift 5, and is removed when the `FullTypedThrows` upcoming feature is enabled.

#### Opaque thrown error types

The thrown error type of a function can be specified with an [opaque result type](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0244-opaque-result-types.md). For example:

```swift
func doSomething() throws(some Error) { ... }
```

The opaque thrown error type is like a result type, so the concrete type of the error is chosen by the `doSomething` function itself, and could change from one version to the next. The caller only knows that the error type conforms to the `Error` protocol; the concrete type won't be knowable until runtime.

Opaque result types can be used as an alternative to existentials (`any Error`) when there is a fixed number of potential error types that might be thrown , and we either can't (due to being in an embedded environment) or don't want to (for performance or code-evolution reasons) expose the precise error type. For example, one could use a suitable `Either` type under the hood:

```swift
func doSomething() throws(some Error) { 
  do {
    try callCat()
  } catch {
    throw Either<CatError, KidError>.left(error)
  }
  
  do {
    try callKids()
  } catch {
    throw Either<CatError, KidError>.right(error)
  }
}
```

Due to the contravariance of parameters, an opaque thrown error type that occurs within a function parameter will be an [opaque parameter](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0341-opaque-parameters.md). This means that the closure argument itself will choose the type, so

```swift
func map<T>(_ transform: (Element) throws(some Error) -> T) rethrows -> [T]
```

is equivalent to

```swift
func map<T, E: Error>(_ transform: (Element) throws(E) -> T) rethrows -> [T]
```

#### `async let`

An `async let` initializer can throw an error, and that error is effectively rethrown at any point where one of the variables defined in the `async let` is referenced. For example:

```swift
async let answer = callCat()
// ... 
try await answer // could rethrow the result from the initializer here
```

The type thrown by the variables of an `async let` is determined using the same rules as for the `do` part of a `do...catch` block. In the example above, accesses to `answer` can throw an error of type `CatError`.

### Subtyping rules

A function type that throws an error of type `A` is a subtype of a function type that differs only in that it throws an error of type `B` when `A` is a subtype of `B`.  As previously noted, a `throws` function that does not specify the thrown error type will have a thrown type of `any Error`, and a non-throwing function has a thrown error type of `Never`. For subtyping purposes, `Never` is assumed to be a subtype of all error types.

The subtyping rule manifests in a number of places, including function conversions, protocol conformance checking and refinements, and override checking, all of which are described below.

#### Function conversions

Having related errors and a non-throwing function

```swift
class BaseError: Error {}
class SubError: BaseError {}

let f1: () -> Void
```

Converting a non-throwing function to a throwing one is allowed

```swift
let f2: () throws(SubError) -> Void = f1
```

It's also allowed to assign a subtype of a thrown error, though the subtype information is erased and the error of f2 will be casted up.

```swift
let f3: () throws(BaseError) -> Void = f2
```

Erasing the specific error type is possible

```swift
let f4: () throws -> Void = f3
```

#### Protocol conformance

Protocols should have the possibility to conform and refine other protocols containing throwing functions based on the subtype relationship of their functions. This way it would be possible to throw a more specialised error or don't throw an error at all.

```swift
protocol Throwing {
    func f() throws
}

struct ConcreteNotThrowing: Throwing {
    func f() { } // okay, doesn't have to throw  
}

enum SpecificError: Error { ... }

struct ConcreteThrowingSpecific: Throwing {
    func f() throws(SpecificError) { } // okay, throws a specific error
}
```

#### Override checking

A declaration in a subclass that overrides a superclass declaration can be a subtype of the superclass declaration, for example:

```swift
class BlueError: Error { ... }
class DeepBlueError: BlueError { ... }

class Superclass {
  func f() throws { }
  func g() throws(BlueError) { }
}

class Subclass: Superclass {
  override func f() throws(BlueError) { }       // okay
  override func g() throws(DeepBlueError) { }   // okay
}

class Subsubclass: Subclass {
  override func f() { } // okay
  override func g() { }  // okay
}
```

### Type inference

The type checker can infer thrown error types in a number of different places, making it easier to carry specific thrown type information through a program without additional annotation. This section covers the various ways in which thrown errors interact with type inference.

#### Closure thrown type inference

Function declarations must always explicitly specify whether they throw, optionally providing a specific thrown error type. For closures, whether they throw or not is inferred by the Swift compiler. Specifically, the Swift compiler looks at the structure of body of the closure. If the body of the closure contains a throwing site (either a `throw` statement or a `try` expression) that is not within an exhaustive `do...catch`  (i.e., one that has an unconditional `catch` clause), then the closure is inferred to be `throws`. Otherwise, it is non-throwing. Here are some examples:

```swift
{ throw E() } // throws

{ try call() } // throws

{ 
  do {
    try call()
  } catch let e as CatError {
    // ...
  }
} // throws, the do...catch is not exhaustive

{ 
  do {
    try call()
  } catch e {}
    // ...
  }
} // does not throw, the do...catch is exhaustive
```

With typed throws, the closure type could be inferred to have a typed error by considering all of the throwing sites that aren't caught (let each have a thrown type `Ei`) and then inferring the closure's thrown error type to be `errorUnion(E1, E2, ... EN)`. 

> **Swift 6**: This inference rule will change the thrown error types of existing closures that throw concrete types. For example, the following closure:
>
> ```swift
> { 
>     if Int.random(in: 0..<24) < 20 {
>         throw CatError.asleep
>     }
> }
> ```
>
> will currently be inferred as `throws`. With the rule specified here, it will be inferred as `throws(CatError)`. This could break some code that depends on the precisely inferred type. To prevent this from becoming a source compatibility problem, we apply the same rule as for `do...catch` statements to limit inference: `throw` statements within the closure body are treated as having the type `any Error` in Swift 5. This way, one can only infer a more specific thrown error type in a closure when the `try` operations are calling functions that make use of typed errors.
>
> Note that one can explicitly specify the thrown error type of a closure to disable this type inference, which has the nice effect of also providing a contextual type for throw statements:
>
> ```swift
> { () throws(CatError) in
>     if Int.random(in: 0..<24) < 20 {
>        throw .asleep
>     }
> }
> ```

#### Associated type inference

An associated type can be used as the thrown error type in other protocol requirements. For example:

```swift
protocol CatFeeder {
    associatedtype FeedError: Error 
    
    func feedCat() throws(FeedError) -> CatStatus
}
```

When a concrete type conforms to such a protocol, the associated type can be inferred from the declarations that satisfy requirements that mention the associated type in a typed throws clause. For the purposes of this inference, a non-throwing function has `Never` as its error type and an untyped `throws` function has `any Error` as its error type. For example:

```swift
struct Tabby: CatFeeder {
  func feedCat() throws(CatError) -> CatStatus { ... } // okay, FeedError is inferred to CatError
}

struct Sphynx: CatFeeder {
  func feedCat() throws -> CatStatus { ... } // okay, FeedError is inferred to any Error
}

struct Ragdoll: CatFeeder {
  func feedCat() -> CatStatus { ... } // okay, FeedError is inferred to Never
}
```

#### `Error` requirement inference

When a function signature uses a generic parameter or associated type as a thrown type, that generic parameter or associated type is implicitly inferred to conform to the `Error` type. For example, given this declaration for `map`:

```swift
func map<T, E>(body: (Element) throws(E) -> T) throws(E) { ... }
```

the function has an inferred requirement `E: Error`. 

### Standard library adoption

#### Converting between `throws` and `Result`

`Result`'s [init(catching:)](https://developer.apple.com/documentation/swift/result/3139399-init) operation translates a throwing closure into a `Result` instance. It's currently defined only when the `Failure` type is `any Error`, i.e.,

```swift
init(catching body: () throws -> Success) where Failure == any Error { ... }
```

Replace this with an initializer that uses typed throws:

```swift
init(catching body: () throws(Failure) -> Success)
```

The new initializer is more flexible: in addition to retaining the error type from typed throws, it also supports non-throwing closure arguments by inferring `Failure` to be equal to `Never`.

Additionally, `Result`'s `get()` operation:

```swift
func get() throws -> Success
```

should use `Failure` as the thrown error type:

```swift
func get() throws(Failure) -> Success
```

#### Standard library operations that `rethrow`

The standard library contains a large number of operations that `rethrow`. In all cases, the standard library will only throw from a call to one of the closure arguments: it will never substitute a different thrown error. Therefore, each `rethrows` operation in the standard library should be replaced with one that uses typed throws to propagate the same error type. For example, the `Optional.map` operation would change from:

```swift 
public func map<U>(
  _ transform: (Wrapped) throws -> U
) rethrows -> U?
```

to

```swift
public func map<U, E>(
  _ transform: (Wrapped) throws(E) -> U
) throws(E) -> U?
```

This is a mechanical transformation that is applied throughout the standard library.

## Source compatibility

This proposal has called out two specific places where the introduction of typed throws into the language will affect source compatibility. In both cases, the type inference behavior of the language will differ when there are `throw` statements that throw a specific concrete type.

To mitigate this source compatibility problem in Swift 5, `throw` statements will be treated as always throwing `any Error`. In Swift 6, they will be treated as throwing the type of their thrown expression. One can enable the Swift 6 behavior with the [upcoming feature flag](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0362-piecemeal-future-features.md) named `FullTypedThrows`.

Note that the source compatibility arguments in this proposal are there to ensure that Swift code that does not use typed throws will continue to work in the same way it always has. Once a function adopts typed throws, the effect of typed throws can then ripple to its callers.

## Effect on API resilience

An API that uses typed throws cannot make its thrown error type more general (or untyped) without breaking existing clients that depend on the specific thrown error type:

```swift
// Library
public enum DataLoaderError {
  case missing
}

public class DataLoader {
  func load() throws(DataLoaderError) -> Data { ... }
}

// Client code
func processError(_ error: DataLoaderError) { ... }

func load(from dataLoader: dataLoader) {
  do {
    try dataLoader.load()
  } catch {
    processError(error)
  }
}
```

Any attempt to generalize the thrown type of `DataLoader.load()` will break the client code, which depends on getting a `DataLoaderError` in the `catch` block.

Going in the other direction, of making the thrown error type *more* specific than it used to be (or adopting typed throws in an API that previously used untyped throws) can also break clients, but in much more limited cases. For example, let's consider the same API above, but in reverse:

```swift
// Library
public enum DataLoaderError {
  case missing
}

public class DataLoader {
  func load() throws -> Data { ... }
}

// Client 
func processError(_ error: any Error) { ... }

func load(from dataLoader: dataLoader) {
  do {
    try dataLoader.load()
  } catch {
    processError(error)
  }
}
```

Here, the `DataLoader.load()` function could be updated to throw `DataLoaderError` and this particular client code would still work, because `DataLoaderError` is convertible to `any Error`. Note that clients could still be broken by this kind of change, for example overrides of an `open` function, declarations that satisfy a protocol requirement, or code that relies on the precide error type (say, by overloading). However, such a change is far less likely to break clients of an API than loosening thrown type informance.

A `rethrows` function can generally be replaced with a function that is generic over the thrown error type of its closure argument and propagates that thrown error. For example, one can replace this API:

```swift
public func last(
    where predicate: (Element) throws -> Bool
) rethrows -> Element?
```

with

```swift
public func last<E>(
    where predicate: (Element) throws(E) -> Bool
) throws(E) -> Element?
```

When calling this function, the closure argument supplies the thrown error type (`E`), which can also be inferred to `any Error` (for untyped `throws`) or `Never` (for non-throwing functions). Existing clients of this new function therefore see the same behavior as with the `rethrows` version.

There is one difference between the two functions that could break client code that is referring to such functions without calling them. For example, consider the following code:

```swift 
let primes = [2, 3, 5, 7]
let getLast = primes.last(where:)
```

With the `rethrows` formulation of the `last(where:)` function, `getLast` will have the type `((Int) throws -> Bool) throws -> Int?`. With the typed-errors formulation, this code will result in an error because the an argument for the generic parameter `E` cannot be inferred without context. Note that this is only a problem when there is no context type for `getLast`, and can be fixed by providing it with a type:

```swift
let getLast: ((Int) -> Bool) -> Int? = primes.last(where:) // okay, E is inferred to Never
```

Note that one would have to do the same thing with the `rethrows` formulation to produce a non-throwing `getLast`, because `rethrows` is not a part of the formal type system. Given that most `rethrows` operations are already generic in other parameters (unlike `last(where:)`), and most uses of such APIs are either calls or have type context, it is expected that the actual source compatibilty impact of replacing `rethrows` with typed errors will be small.

## Effect on ABI stability

The ABI between a function with an untyped throws and one that uses typed throws will be different, so that typed throws can benefit from knowing the precise type.

Replacing a `rethrows` function with one that uses typed throws, as proposed for the standard library, is an ABI-breaking change. However, it can be done in a manner that doesn't break ABI by retaining the `rethrows` function only for binary-compatibility purposes. The existing `rethrows` functions will be renamed at the source level (so they don't conflict with the new ones) and made  `@usableFromInline internal`, which retains the ABI while making the function invisible to clients of the standard library:

```swift
@usableFromInline 
@_silgen_name(<mangled name of the existing function>)
internal func _oldRethrowingMap<U>(
  _ transform: (Wrapped) throws -> U
) rethrows -> U?
```

Then, the new typed-throws version will be introduced with [back-deployment support](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0376-function-back-deployment.md):

```swift
@backDeploy(...)
public func map<U, E>(
  _ transform: (Wrapped) throws(E) -> U
) throws(E) -> U?
```

This way, clients compiled against the updated standard library will always use the typed-throws version. Note that many of these functions are quite small and will be generic, so implementers may opt to use `@_alwaysEmitIntoClient` rather than `@backDeploy`.

## Future directions

### Concurrency library adoption

The concurrency library has a number of places that could benefit from the adoption of typed throws, including `Task` creation and completion, continuations, task cancellation, task groups, and async sequences and streams. 

`Task` is similar to `Result` because it also carries a `Failure` type that could benefit from typed throws. Continuations and task groups could propagate typed throws information from closures to make more of the library usable with precise thrown type information.

`AsyncSequence`, and the asynchronous `for..in` loop that depends on it, could be improved by using typed throws. Both `AsyncIteratorProtocol` and `AsyncSequence` could be augmented with a `Failure` associated type that is used for the thrown error type of `next()`, and will be used by the asynchronous `for..in` loop to determine whether the sequence can throw. This can be combined with [primary associated types](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0346-light-weight-same-type-syntax.md) to make it possible to use existentials such as `any AsyncSequence<Image, NetworkError>`:

```swift
public protocol AsyncIteratorProtocol<Element, Failure> {
  associatedtype Element
  associatedtype Failure: Error = any Error
  mutating func next() async throws(Failure) -> Element?
}

public protocol AsyncSequence<Element, Failure> {
  associatedtype AsyncIterator: AsyncIteratorProtocol
  associatedtype Element where AsyncIterator.Element == Element
  associatedtype Failure where AsyncIterator.Failure == Failure
  __consuming func makeAsyncIterator() -> AsyncIterator
}
```

The scope of potential changes to the concurrency library to make full use of typed throws is large. Unlike with the standard library, the adoption of typed throws in the concurrency library requires some interesting design. Therefore, we leave it to a follow-on proposal, noting only that whatever form `AsyncSequence` takes with typed throws, the language support for asynchronous `for..in` will need to adjust.

### Specific thrown error types for distributed actors

The transport mechanism for [distributed actors](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0344-distributed-actor-runtime.md), `DistributedActorSystem`, can throw an error due to transport failures. This error is currently untyped, but it should be possible to adopt typed throws (with a `Failure` associated type in `DistributedActorSystem` and mirrored in `DistributedActor`) so that the distributed actor system can be more specific about the kind of error it throws. Calls to a distributed actor from outside the actor (i.e., that could be on a different node) would then throw `errorUnion(Failure, E)` where the `E` is the type that the function normally throws.

## Alternatives considered

### Thrown error type syntax

There have been several alternatives to the `throws(E)` syntax proposed here. The `throws(E)` syntax was chosen because it is syntactically unambiguous, allows arbitrary types for `E`,  and is consistent with the way in which attributes (like property wrappers or macros with arguments) and modifiers (like `unowned(unsafe)`) are written.

The most commonly proposed syntax omits the parentheses, i.e., `throws E`. However, this syntax introduces some syntactic ambiguities that would need to be addressed and might cause problems for future evolution of the language:

* The following code is syntactically ambiguous if `E` is parsed with the arbitrary `type` grammar:

  ```swift
  func f() throws (E) -> Int { ... }
  ```

  because the error type would parse as either `(E)` or `(E) -> Int`. One could parse a subset of the `type` grammar that doesn't include function types to dodge this ambiguity, with a more complicated grammar.

* The identifier following `throws` could end up conflicting with a future effect:

  ```swift
  func f() throws E { ... }
  ```

  If `E` were an effect name in some later Swift version, then there is an ambiguity between typed throws and that effect that we would need to resolve. Future effect modifiers might require more than one argument (and therefore need parentheses), which would make them inconsistent with `throws E`.

Another suggestion uses angle brackets around the thrown type, i.e.,

```swift
func f() throws<E> -> Int { ... }
```

This follows more closely with generic syntax, and highlights the type nature of the arguments more clearly. It's inconsistent with the use of parentheses in modifiers, but has some precedent in attached macros where one can explicitly specify the generic arguments to the macro, e.g., `@OptionSet<UInt16>`. 

### Multiple thrown error types

This proposal specifies that a function may throw at most one error type, and if there is any reason to throw more than one error type, one should use `any Error` (or the equivalent untyped `throws` spelling). It would be possible to support multiple error types, e.g.,

```swift
func fetchData() throws(FileSystemError, NetworkError) -> Data
```

However, this change would introduce a significant amount of complexity in the type system, because everywhere that deals with thrown errors would have to deal with an arbitrary set of thrown errors.

A more reasonable direction to support this use case would be to introduce a form of anonymous enum (often called a *sum* type) into the language itself, where the type `A | B` can be either an `A` or ` B`. With such a feature in place, one could express the function above as:

```swift
func fetchData() throws(FileSystemError | NetworkError) -> Data
```

Trying to introduce multiple thrown error types directly into the language would introduce nearly all of the complexity of sum types, but without the generality, so this proposal only considers a single thrown error type.

### Treat all uninhabited thrown error types as nonthrowing

This proposal specifies that a function type whose thrown error type is `Never` is equivalent to a function type that does not throw. This rule could be generalized from `Never` to any *uninhabited* type, i.e., any type for which we can structurally determine that there is no runtime value. The simplest uninhabited type is a frozen enum with no cases, which is how `Never` itself is defined:

```swift
@frozen public enum Never {}
```

However, there are other forms of uninhabited type: a `struct` or `class` with a stored property of uninhabited type is uninhabited, as is an enum where all cases have an associated value containing an uninhabited type (a generalization of the "no cases" rule mentioned above). This can happen generically. For example, a simple `Pair` struct:

```swift
struct Pair<First, Second> {
  var first: First
  var second Second
}
```

will be uninhabited when either `First` or `Second` is uninhabited. The `Either` enum will be uninhabited when both of its generic arguments are uninhabited. `Optional` is never uninhabited, because it's always possible to create a `nil` value.

It is possible to generalize the rule about non-throwing function types to consider any function type with an uninhabited thrown error type to be equivalent to a non-throwing function type (all other things remaining equal). However, we do not do so due to implementation concerns: the check for a type being uninhabited is nontrivial, requiring one to walk all of the storage of the type, and (in the presence of indirect enum cases and reference types) is recursive, making it a potentially expensive computation.  Crucially, this computation will need to be performed at runtime, to produce proper function type metadata within generic functions:

```swift
func f<E: Error>(_: E.Type)) {
  typealias Fn = () throws(E) -> Void
  let meta = Fn.self
}

f(Never.self)                // Fn should be equivalent to () -> Void
f(Either<Never, Never>.self) // Fn should be equivalent to () -> Void
f(Pair<Never, Int>.self)     // Fn should be equivalent to () -> Void
```

The runtime computation of "uninhabited" therefore carries significant cost in terms of the metadata required (one may need to walk all of the storage of the type) as well as the execution time to evaluate that metadata during runtime type formation. 

The most plausible route here involves the introduction of an `Uninhabited` protocol, which could then be used with conditional conformances to propagate the "uninhabited" type information. For example, `Never` would conform to `Uninhabited`, and one could conditionally conform a generic error type. For example:

```swift
struct WrappedError<E: Error>: Error {
  var wrapped: E
}

extension WrappedError: Uninhabited where E: Uninhabited { }
```

With this, one can express "rethrowing" behavior that wraps the underlying error via typed throws:

```swift
func translatesError<E: Error>(f: () throws(E) -> Void) throws(WrappedError<E>) { ... }
```

Here, when give a non-throwing closure for `f` (which infers `E = Never`), `translatesError` is known not to throw because `WrappedError<Never>` is known to be uninhabited (via the conditional conformance). This approach extends to the use of an `Either` type to capture errors:

```swift
extension Either: Uninhabited when Left: Uninhabited, Right: Uninhabited { }
```

However, it breaks down when there are two such generic error parameters for something like `WrappedError`, because having either one of them be `Uninhabited` makes the struct uninhabited, and the generics system does not permit disjunctive constraints like that.

Extending from `Never` to arbitrary uninhabited types has some benefits, but requires enough additional design work and complexity that it should constitute a separate proposal. Therefore, we stick with the simpler rule where `Never` is the only uninhabited type considered to be special.

### Typed `rethrows`

A function marked `rethrows` throws only when one or more of its closure arguments throws. As note previously, typed throws allows one to more precisely express when the function only rethrows exactly the error from its closure, without translation, as demonstrated with `map`:

```swift
func map<T, E: Error>(_ transform: (Element) throws(E) -> T) throws(E) -> [T]
```

However, it cannot express rethrowing behavior when the function is performing translation of errors. For example, consider the following:

```swift
func translateErrors<E1: Error, E2: Error>(
  f: () throws(E1) -> Void, 
  g: () throws(E2) -> Void
) ??? {
  do {
    try f()
  } catch {
    throw SimpleError(message: "E1: \(error)")
  }
  
  do {
    try g()
  } catch {
    throw SimpleError(message: "E2: \(error)")
  }
}
```

This function will only throw when `f` or `g` throw, and in both cases will translate the errors into `SimpleError`. With this proposal, there are two options for specifying the error-handling behavior of `translateErrors`, neither of which is precise:

* `rethrows` correctly communicates that this function throws only when the arguments for `f` or `g` do, but the thrown error type is treated as `any Error`.
* `throws(SimpleError)` correctly communicates that this function throws errors of type `SimpleError`, but not that it throws when the argument for `f` or `g` do.

One way to address this would be to allow `rethrows` to specify the thrown error type, e.g., `rethrows(SimpleError)`, which captures both of the aspects of how this function behavies---when it throws, and what specific error type it `throws`.  

With typed `rethrows`, a bare `rethrows` could be treated as syntactic sugar for `rethrows(any Error)`, similarly to how `throws` is syntactic sugar for `throws(any Error)`. This extension is source-compatible and allows one to express more specific error types with throwing behavior.

However, this definition of `rethrows` is somewhat unfortunate in a typed-throws world, because it is likely the wrong default. Many use cases for `rethrows` do not involve error translation, and would be better served by using typed throws in the manner that `map` does. If `rethrows` were not already part of the Swift language prior to this proposal, it's likely that we either would not introduce the feature at all, or would treat it as syntactic sugar for typed throws that introduces a generic parameter for the error type that is used for the thrown type of the closure parameters and the function itself. For example:

```swift
// rethrows could try rethrows as syntactic sugar..
func map<T>(_ transform: (Element) throws -> T) rethrows -> [T]
// for typed errors:
func map<T, E: Error>(_ transform: (Element) throws(E) -> T) throws(E) -> [T]
```

Removing or changing the semantics of `rethrows` would be a source-incompatible change, so we leave such concerns to a later proposal.

## Revision history

* Revision 5 (first review):
  * Add `do throws(MyError)` { ... } syntax to allow explicit specification of the thrown error type within the body of a `do..catch` block, suppressing type inference of the thrown error type. Thank you to Becca Royal-Gordon for the idea!
* Revision 4:
  * Update the introduction, motivation, and "when to use typed throws" to be more direct.
  * Re-incorporate the replacement of `rethrows` functions in the standard library with generic typed throws into the actual proposal. It's so mechanical and straightforward that it doesn't need a separate proposal.
  * Extend the discussion on API resilience to talk through the source compatibility impacts of replacing a `rethrows` function with one that uses typed throws, since it is quite relevant to this proposal.
  * Explain that one cannot currently have a thrown error type of `any Error & Codable` or similar because it doesn't conform to `Error`.
  * Introduce a compatibility feature to `rethrows` functions to cope with their callees moving to typed throws.
* Revision 3:
  * Move the the typed `rethrows` feature out of this proposal, and into Alternatives Considered. Once we gain more experience with typed throws, we can decide what to do with `rethrows`.
  * Expand the discussion on allowing all uninhabited error types to mean "non-throwing".
  * Provide a better example for inferring `Error` conformance on generic parameters.
  * Move the replacement of `rethrows` in the standard library with typed throws into "Future Directions", because it is large enough that it needs a separate proposal.
  * Move the concurrency library changes for typed throws into "Future Directions", because it is large enough that it needs a separate proposal.
  * Add an extended example of replacing the need for `rethrows(unsafe)` with typed throws.
  * Provide a more significant example of opaque thrown errors that makes use of `Either` internally.
* Revision 2:
  * Add a short section on when to use typed throws
  * Add an Alternatives Considered section for other syntaxes
  * Make it clear that only unconditional catches make `do...catch` exhaustive
  * Update continuation APIs with typed throws
  * Add an example of an existential thrown error type
  * Describe semantics of `async let` with respect to thrown errors
  * Add updates to task cancellation APIs
