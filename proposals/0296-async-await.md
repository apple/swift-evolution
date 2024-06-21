# Async/await

* Proposal: [SE-0296](0296-async-await.md)
* Authors: [John McCall](https://github.com/rjmccall), [Doug Gregor](https://github.com/DougGregor)
* Review Manager: [Ben Cohen](https://github.com/airspeedswift)
* Status: **Implemented (Swift 5.5)**
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-with-modification-se-0296-async-await/43318), [Amendment to allow overloading on `async`](https://forums.swift.org/t/accepted-amendment-to-se-0296-allow-overloads-that-differ-only-in-async/50117)

## Table of Contents

   * [Async/await](#asyncawait)
      * [Introduction](#introduction)
      * [Motivation: Completion handlers are suboptimal](#motivation-completion-handlers-are-suboptimal)
      * [Proposed solution: async/await](#proposed-solution-asyncawait)
         * [Suspension points](#suspension-points)
      * [Detailed design](#detailed-design)
         * [Asynchronous functions](#asynchronous-functions)
         * [Asynchronous function types](#asynchronous-function-types)
         * [Await expressions](#await-expressions)
         * [Closures](#closures)
         * [Overloading and overload resolution](#overloading-and-overload-resolution)
         * [Autoclosures](#autoclosures)
         * [Protocol conformance](#protocol-conformance)
      * [Source compatibility](#source-compatibility)
      * [Effect on ABI stability](#effect-on-abi-stability)
      * [Effect on API resilience](#effect-on-api-resilience)
      * [Future Directions](#future-directions)
         * [reasync](#reasync)
      * [Alternatives Considered](#alternatives-considered)
         * [Make await imply try](#make-await-imply-try)
         * [Launching async tasks](#launching-async-tasks)
         * [Await as syntactic sugar](#await-as-syntactic-sugar)
      * [Revision history](#revision-history)
      * [Related proposals](#related-proposals)
      * [Acknowledgments](#acknowledgments)

## Introduction

Modern Swift development involves a lot of asynchronous (or "async") programming using closures and completion handlers, but these APIs are hard to use.  This gets particularly problematic when many asynchronous operations are used, error handling is required, or control flow between asynchronous calls gets complicated.  This proposal describes a language extension to make this a lot more natural and less error prone.

This design introduces a [coroutine model](https://en.wikipedia.org/wiki/Coroutine) to Swift. Functions can opt into being `async`, allowing the programmer to compose complex logic involving asynchronous operations using the normal control-flow mechanisms. The compiler is responsible for translating an asynchronous function into an appropriate set of closures and state machines.

This proposal defines the semantics of asynchronous functions. However, it does not provide concurrency: that is covered by a separate proposal to introduce structured concurrency, which associates asynchronous functions with concurrently-executing tasks and provides APIs for creating, querying, and cancelling tasks.

Swift-evolution thread: [Pitch #1](https://forums.swift.org/t/concurrency-asynchronous-functions/41619), [Pitch #2](https://forums.swift.org/t/pitch-2-async-await/42420)

## Motivation: Completion handlers are suboptimal

Async programming with explicit callbacks (also called completion handlers) has many problems, which we’ll explore below.  We propose to address these problems by introducing async functions into the language.  Async functions allow asynchronous code to be written as straight-line code.  They also allow the implementation to directly reason about the execution pattern of the code, allowing callbacks to run far more efficiently.

#### Problem 1: Pyramid of doom

A sequence of simple asynchronous operations often requires deeply-nested closures. Here is a made-up example showing this:

```swift
func processImageData1(completionBlock: (_ result: Image) -> Void) {
    loadWebResource("dataprofile.txt") { dataResource in
        loadWebResource("imagedata.dat") { imageResource in
            decodeImage(dataResource, imageResource) { imageTmp in
                dewarpAndCleanupImage(imageTmp) { imageResult in
                    completionBlock(imageResult)
                }
            }
        }
    }
}

processImageData1 { image in
    display(image)
}
```

This "pyramid of doom" makes it difficult to read and keep track of where the code is running. In addition, having to use a stack of closures leads to many second order effects that we will discuss next.

#### Problem 2: Error handling

Callbacks make error handling difficult and very verbose. Swift 2 introduced an error handling model for synchronous code, but callback-based interfaces do not derive any benefit from it:

```swift
// (2a) Using a `guard` statement for each callback:
func processImageData2a(completionBlock: (_ result: Image?, _ error: Error?) -> Void) {
    loadWebResource("dataprofile.txt") { dataResource, error in
        guard let dataResource = dataResource else {
            completionBlock(nil, error)
            return
        }
        loadWebResource("imagedata.dat") { imageResource, error in
            guard let imageResource = imageResource else {
                completionBlock(nil, error)
                return
            }
            decodeImage(dataResource, imageResource) { imageTmp, error in
                guard let imageTmp = imageTmp else {
                    completionBlock(nil, error)
                    return
                }
                dewarpAndCleanupImage(imageTmp) { imageResult, error in
                    guard let imageResult = imageResult else {
                        completionBlock(nil, error)
                        return
                    }
                    completionBlock(imageResult)
                }
            }
        }
    }
}

processImageData2a { image, error in
    guard let image = image else {
        display("No image today", error)
        return
    }
    display(image)
}
```

The addition of [`Result`](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0235-add-result.md) to the standard library improved on error handling for Swift APIs. Asynchronous APIs were one of the [main motivators](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0235-add-result.md#asynchronous-apis) for `Result`: 

```swift
// (2b) Using a `do-catch` statement for each callback:
func processImageData2b(completionBlock: (Result<Image, Error>) -> Void) {
    loadWebResource("dataprofile.txt") { dataResourceResult in
        do {
            let dataResource = try dataResourceResult.get()
            loadWebResource("imagedata.dat") { imageResourceResult in
                do {
                    let imageResource = try imageResourceResult.get()
                    decodeImage(dataResource, imageResource) { imageTmpResult in
                        do {
                            let imageTmp = try imageTmpResult.get()
                            dewarpAndCleanupImage(imageTmp) { imageResult in
                                completionBlock(imageResult)
                            }
                        } catch {
                            completionBlock(.failure(error))
                        }
                    }
                } catch {
                    completionBlock(.failure(error))
                }
            }
        } catch {
            completionBlock(.failure(error))
        }
    }
}

processImageData2b { result in
    do {
        let image = try result.get()
        display(image)
    } catch {
        display("No image today", error)
    }
}
```

```swift
// (2c) Using a `switch` statement for each callback:
func processImageData2c(completionBlock: (Result<Image, Error>) -> Void) {
    loadWebResource("dataprofile.txt") { dataResourceResult in
        switch dataResourceResult {
        case .success(let dataResource):
            loadWebResource("imagedata.dat") { imageResourceResult in
                switch imageResourceResult {
                case .success(let imageResource):
                    decodeImage(dataResource, imageResource) { imageTmpResult in
                        switch imageTmpResult {
                        case .success(let imageTmp):
                            dewarpAndCleanupImage(imageTmp) { imageResult in
                                completionBlock(imageResult)
                            }
                        case .failure(let error):
                            completionBlock(.failure(error))
                        }
                    }
                case .failure(let error):
                    completionBlock(.failure(error))
                }
            }
        case .failure(let error):
            completionBlock(.failure(error))
        }
    }
}

processImageData2c { result in
    switch result {
    case .success(let image):
        display(image)
    case .failure(let error):
        display("No image today", error)
    }
}
```

It's easier to handle errors when using `Result`, but the closure-nesting problem remains.

#### Problem 3: Conditional execution is hard and error-prone

Conditionally executing an asynchronous function is a huge pain. For example, suppose we need to "swizzle" an image after obtaining it. But, we sometimes have to make an asynchronous call to decode the image before we can swizzle. Perhaps the best approach to structuring this function is to write the swizzling code in a helper "continuation" closure that is conditionally captured in a completion handler, like this:

```swift
func processImageData3(recipient: Person, completionBlock: (_ result: Image) -> Void) {
    let swizzle: (_ contents: Image) -> Void = {
      // ... continuation closure that calls completionBlock eventually
    }
    if recipient.hasProfilePicture {
        swizzle(recipient.profilePicture)
    } else {
        decodeImage { image in
            swizzle(image)
        }
    }
}
```

This pattern inverts the natural top-down organization of the function: the code that will execute in the second half of the function must appear *before* the part that executes in the first half. In addition to restructuring the entire function, we must now think carefully about captures in the continuation closure, because the closure is used in a completion handler. The problem worsens as the number of conditionally-executed async functions grows, yielding what is essentially an inverted "pyramid of doom."

#### Problem 4: Many mistakes are easy to make

It's quite easy to bail-out of the asynchronous operation early by simply returning without calling the correct completion-handler block. When forgotten, the issue is very hard to debug:

```swift
func processImageData4a(completionBlock: (_ result: Image?, _ error: Error?) -> Void) {
    loadWebResource("dataprofile.txt") { dataResource, error in
        guard let dataResource = dataResource else {
            return // <- forgot to call the block
        }
        loadWebResource("imagedata.dat") { imageResource, error in
            guard let imageResource = imageResource else {
                return // <- forgot to call the block
            }
            ...
        }
    }
}
```

When you do remember to call the block, you can still forget to return after that:

```swift
func processImageData4b(recipient:Person, completionBlock: (_ result: Image?, _ error: Error?) -> Void) {
    if recipient.hasProfilePicture {
        if let image = recipient.profilePicture {
            completionBlock(image) // <- forgot to return after calling the block
        }
    }
    ...
}
```

Thankfully the `guard` syntax protects against forgetting to return to some degree, but it's not always relevant.

#### Problem 5: Because completion handlers are awkward, too many APIs are defined synchronously

This is hard to quantify, but the authors believe that the awkwardness of defining and using asynchronous APIs (using completion handlers) has led to many APIs being defined with apparently synchronous behavior, even when they can block.  This can lead to problematic performance and responsiveness problems in UI applications, e.g. a spinning cursor.  It can also lead to the definition of APIs that cannot be used when asynchrony is critical to achieve scale, e.g. on the server.

## Proposed solution: async/await

Asynchronous functions—often known as async/await—allow asynchronous code to be written as if it were straight-line, synchronous code.  This immediately addresses many of the problems described above by allowing programmers to make full use of the same language constructs that are available to synchronous code.  The use of async/await also naturally preserves the semantic structure of the code, providing information necessary for at least three cross-cutting improvements to the language: (1) better performance for asynchronous code; (2) better tooling to provide a more consistent experience while debugging, profiling, and exploring code; and (3) a foundation for future concurrency features like task priority and cancellation.  The example from the prior section demonstrates how async/await drastically simplifies asynchronous code:

```swift
func loadWebResource(_ path: String) async throws -> Resource
func decodeImage(_ r1: Resource, _ r2: Resource) async throws -> Image
func dewarpAndCleanupImage(_ i : Image) async throws -> Image

func processImageData() async throws -> Image {
  let dataResource  = try await loadWebResource("dataprofile.txt")
  let imageResource = try await loadWebResource("imagedata.dat")
  let imageTmp      = try await decodeImage(dataResource, imageResource)
  let imageResult   = try await dewarpAndCleanupImage(imageTmp)
  return imageResult
}
```

Many descriptions of async/await discuss it through a common implementation mechanism: a compiler pass which divides a function into multiple components.  This is important at a low level of abstraction in order to understand how the machine is operating, but at a high level we’d like to encourage you to ignore it.  Instead, think of an asynchronous function as an ordinary function that has the special power to give up its thread.  Asynchronous functions don’t typically use this power directly; instead, they make calls, and sometimes these calls will require them to give up their thread and wait for something to happen.  When that thing is complete, the function will resume executing again.

The analogy with synchronous functions is very strong.  A synchronous function can make a call; when it does, the function immediately waits for the call to complete. Once the call completes, control returns to the function and picks up where it left off.  The same thing is true with an asynchronous function: it can make calls as usual; when it does, it (normally) immediately waits for the call to complete. Once the call completes, control returns to the function and it picks up where it was.  The only difference is that synchronous functions get to take full advantage of (part of) their thread and its stack, whereas *asynchronous functions are able to completely give up that stack and use their own, separate storage*.  This additional power given to asynchronous functions has some implementation cost, but we can reduce that quite a bit by designing holistically around it.

Because asynchronous functions must be able to abandon their thread, and synchronous functions don’t know how to abandon a thread, a synchronous function can’t ordinarily call an asynchronous function: the asynchronous function would only be able to give up the part of the thread it occupied, and if it tried, its synchronous caller would treat it like a return and try to pick up where it was, only without a return value.  The only way to make this work in general would be to block the entire thread until the asynchronous function was resumed and completed, and that would completely defeat the purpose of asynchronous functions, as well as having nasty systemic effects.

In contrast, an asynchronous function can call either synchronous or asynchronous functions.  While it’s calling a synchronous function, of course, it can’t give up its thread.  In fact, asynchronous functions never just spontaneously give up their thread; they only give up their thread when they reach what’s called a suspension point.  A suspension point can occur directly within a function, or it can occur within another asynchronous function that the function calls, but in either case the function and all of its asynchronous callers simultaneously abandon the thread.  (In practice, asynchronous functions are compiled to not depend on the thread during an asynchronous call, so that only the innermost function needs to do any extra work.)

When control returns to an asynchronous function, it picks up exactly where it was.  That doesn’t necessarily mean that it’ll be running on the exact same thread it was before, because the language doesn’t guarantee that after a suspension.  In this design, threads are mostly an implementation mechanism, not a part of the intended interface to concurrency.  However, many asynchronous functions are not just asynchronous: they’re also associated with specific actors (which are the subject of a separate proposal), and they’re always supposed to run as part of that actor.  Swift does guarantee that such functions will in fact return to their actor to finish executing.  Accordingly, libraries that use threads directly for state isolation—for example, by creating their own threads and scheduling tasks sequentially onto them—should generally model those threads as actors in Swift in order to allow these basic language guarantees to function properly.

### Suspension points

A suspension point is a point in the execution of an asynchronous function where it has to give up its thread.  Suspension points are always associated with some deterministic, syntactically explicit event in the function; they’re never hidden or asynchronous from the function’s perspective.  The primary form of suspension point is a call to an asynchronous function associated with a different execution context.

It is important that suspension points are only associated with explicit operations.  In fact, it’s so important that this proposal requires that calls that *might* suspend be enclosed in an `await` expression. These calls are referred to as *potential suspension points*, because it is not known statically whether they will actually suspend: that depends both on code not visible at the call site (e.g., the callee might depend on asynchronous I/O) as well as dynamic conditions (e.g., whether that asynchronous I/O will have to wait to complete). 

The requirement for `await` on potential suspension points follows Swift's precedent of requiring `try` expressions to cover calls to functions that can throw errors. Marking potential suspension points is particularly important because *suspensions interrupt atomicity*.  For example, if an asynchronous function is running within a given context that is protected by a serial queue, reaching a suspension point means that other code can be interleaved on that same serial queue.  A classic but somewhat hackneyed example where this atomicity matters is modeling a bank: if a deposit is credited to one account, but the operation suspends before processing a matched withdrawal, it creates a window where those funds can be double-spent.  A more germane example for many Swift programmers is a UI thread: the suspension points are the points where the UI can be shown to the user, so programs that build part of their UI and then suspend risk presenting a flickering, partially-constructed UI.  (Note that suspension points are also called out explicitly in code using explicit callbacks: the suspension happens between the point where the outer function returns and the callback starts running.)  Requiring that all potential suspension points are marked allows programmers to safely assume that places without potential suspension points will behave atomically, as well as to more easily recognize problematic non-atomic patterns.

Because potential suspension points can only appear at points explicitly marked within an asynchronous function, long computations can still block threads.  This might happen when calling a synchronous function that just does a lot of work, or when encountering a particularly intense computational loop written directly in an asynchronous function.  In either case, the thread cannot interleave code while these computations are running, which is usually the right choice for correctness, but can also become a scalability problem.  Asynchronous programs that need to do intense computation should generally run it in a separate context.  When that’s not feasible, there will be library facilities to artificially suspend and allow other operations to be interleaved.

Asynchronous functions should avoid calling functions that can actually block the thread, especially if they can block it waiting for work that’s not guaranteed to be currently running.  For example, acquiring a mutex can only block until some currently-running thread gives up the mutex; this is sometimes acceptable but must be used carefully to avoid introducing deadlocks or artificial scalability problems.  In contrast, waiting on a condition variable can block until some arbitrary other work gets scheduled that signals the variable; this pattern goes strongly against recommendation.

## Detailed design

### Asynchronous functions

Function types can be marked explicitly as `async`, indicating that the function is asynchronous:

```swift
func collect(function: () async -> Int) { ... }
```

A function or initializer declaration can also be declared explicitly as `async`:

```swift
class Teacher {
  init(hiringFrom: College) async throws {
    ...
  }
  
  private func raiseHand() async -> Bool {
    ...
  }
}
```

> **Rationale**: The `async` follows the parameter list because it is part of the function's type as well as its declaration. This follows the precedent of `throws`.

The type of a reference to a function or initializer declared `async` is an `async` function type. If the reference is a “curried” static reference to an instance method, it is the "inner" function type that is `async`, consistent with the usual rules for such references.

Special functions like `deinit` and storage accessors (i.e., the getters and setters for properties and subscripts) cannot be `async`.

> **Rationale**: Properties and subscripts that only have a getter could potentially be `async`. However, properties and subscripts that also have an `async` setter imply the ability to pass the reference as `inout` and drill down into the properties of that property itself, which depends on the setter effectively being an "instantaneous" (synchronous, non-throwing) operation. Prohibiting `async` properties is a simpler rule than only allowing get-only `async` properties and subscripts.
 
If a function is both `async` and `throws`, then the `async` keyword must precede `throws` in the type declaration. This same rule applies if `async` and `rethrows`.

> **Rationale** : This order restriction is arbitrary, but it's not harmful, and it eliminates the potential for stylistic debates.

An `async` initializer of a class that has a superclass but lacks a call to a superclass initializer will get an implicit call to `super.init()` only if the superclass has a zero-argument, synchronous, designated initializer.

> **Rationale**: If the superclass initializer is `async`, the call to the asynchronous initializer is a potential suspension point and therefore the call (and required `await`) must be visible in the source.
 
### Asynchronous function types

Asynchronous function types are distinct from their synchronous counterparts. However, there is an implicit conversion from a synchronous function type to its corresponding asynchronous function type. This is similar to the implicit conversion from a non-throwing function to its throwing counterpart, which can also compose with the asynchronous function conversion. For example:

```swift
struct FunctionTypes {
  var syncNonThrowing: () -> Void
  var syncThrowing: () throws -> Void
  var asyncNonThrowing: () async -> Void
  var asyncThrowing: () async throws -> Void
  
  mutating func demonstrateConversions() {
    // Okay to add 'async' and/or 'throws'    
    asyncNonThrowing = syncNonThrowing
    asyncThrowing = syncThrowing
    syncThrowing = syncNonThrowing
    asyncThrowing = asyncNonThrowing
    
    // Error to remove 'async' or 'throws'
    syncNonThrowing = asyncNonThrowing // error
    syncThrowing = asyncThrowing       // error
    syncNonThrowing = syncThrowing     // error
    asyncNonThrowing = syncThrowing    // error
  }
}
```

### Await expressions

A call to a value of `async` function type (including a direct call to an `async` function) introduces a potential suspension point. 
Any potential suspension point must occur within an asynchronous context (e.g., an `async` function). Furthermore, it must occur within the operand of an `await` expression. 

Consider the following example:

```swift
// func redirectURL(for url: URL) async -> URL { ... }
// func dataTask(with: URL) async throws -> (Data, URLResponse) { ... }

let newURL = await server.redirectURL(for: url)
let (data, response) = try await session.dataTask(with: newURL)
```

In this code example, a task suspension may happen during the calls to `redirectURL(for:)` and `dataTask(with:)` because they are async functions. Thus, both call expressions must be contained within some `await` expression, because each call contains a potential suspension point. An `await` operand may contain more than one potential suspension point. For example, we can use one `await` to cover both potential suspension points from our example by rewriting it as:

```swift
let (data, response) = try await session.dataTask(with: server.redirectURL(for: url))
```

The `await` has no additional semantics; like `try`, it merely marks that an asynchronous call is being made.  The type of the `await` expression is the type of its operand, and its result is the result of its operand.
An `await` operand may also have no potential suspension points, which will result in a warning from the Swift compiler, following the precedent of `try` expressions:

```swift
let x = await synchronous() // warning: no calls to 'async' functions occur within 'await' expression
```

> **Rationale**: It is important that asynchronous calls are clearly identifiable within the function because they may introduce suspension points, which break the atomicity of the operation.  The suspension points may be inherent to the call (because the asynchronous call must execute on a different executor) or simply be part of the implementation of the callee, but in either case it is semantically important and the programmer needs to acknowledge it. `await` expressions are also an indicator of asynchronous code, which interacts with inference in closures; see the section on [Closures](#closures) for more information.

A potential suspension point must not occur within an autoclosure that is not of `async` function type.

A potential suspension point must not occur within a `defer` block.

If both `await` and a variant of `try` (including `try!` and `try?`) are applied to the same subexpression, `await` must follow the `try`/`try!`/`try?`:

```swift
let (data, response) = await try session.dataTask(with: server.redirectURL(for: url)) // error: must be `try await`
let (data, response) = await (try session.dataTask(with: server.redirectURL(for: url))) // okay due to parentheses
```

> **Rationale**: this restriction is arbitrary, but follows the equally-arbitrary restriction on the ordering of `async throws` in preventing stylistic debates.

### Closures

A closure can have `async` function type. Such closures can be explicitly marked as `async` as follows:

```swift
{ () async -> Int in
  print("here")
  return await getInt()
}
```

An anonymous closure is inferred to have `async` function type if it contains an `await` expression.

```swift
let closure = { await getInt() } // implicitly async

let closure2 = { () -> Int in     // implicitly async
  print("here")
  return await getInt()
}
```

Note that inference of `async` on a closure does not propagate to its enclosing or nested functions or closures, because those contexts are separably asynchronous or synchronous. For example, only `closure6` is inferred to be `async` in this situation:

```swift
// func getInt() async -> Int { ... }

let closure5 = { () -> Int in       // not 'async'
  let closure6 = { () -> Int in     // implicitly async
    if randomBool() {
      print("there")
      return await getInt()
    } else {
      let closure7 = { () -> Int in 7 }  // not 'async'
      return 0
    }
  }
  
  print("here")
  return 5
}
```

### Overloading and overload resolution

Existing Swift APIs generally support asynchronous functions via a callback interface, e.g.,

```swift
func doSomething(completionHandler: ((String) -> Void)? = nil) { ... }
```

Many such APIs are likely to be updated by adding an `async` form:

```swift
func doSomething() async -> String { ... }
```

These two functions have different names and signatures, even though they share the same base name. However, either of them can be called with no parameters (due to the defaulted completion handler), which would present a problem for existing code:

```swift
doSomething() // problem: can call either, unmodified Swift rules prefer the `async` version
```

A similar problem exists for APIs that evolve into providing both a synchronous and an asynchronous version of the same function, with the same signature. Such pairs allow APIs to provide a new asynchronous function which better fits in the Swift asynchronous landscape, without breaking backward compatibility. New asynchronous functions can support, for example, cancellation (covered in the [Structured Concurrency](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md) proposal).

```swift
// Existing synchronous API
func doSomethingElse() { ... }

// New and enhanced asynchronous API
func doSomethingElse() async { ... }
```

In the first case, Swift's overloading rules prefer to call a function with fewer default arguments, so the addition of the `async` function would break existing code that called the original `doSomething(completionHandler:)` with no completion handler. This would get an error along the lines of:

```
error: `async` function cannot be called from non-asynchronous context
```

This presents problems for code evolution, because developers of existing asynchronous libraries would have to either have a hard compatiblity break (e.g, to a new major version) or would need have different names for all of the new `async` versions. The latter would likely result in a scheme such as [C#'s pervasive `Async` suffix](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/async/task-asynchronous-programming-model).

The second case, where both functions have the same signature and only differ in `async`, is normally rejected by existing Swift's overloading rules. Those do not allow two functions to differ only in their *effects*, and one can not define two functions that only differ in `throws`, for example.

```
// error: redeclaration of function `doSomethingElse()`.
```

This also presents a problem for code evolution, because developers of existing libraries just could not preserve their existing synchronous APIs, and support new asynchronous features.

Instead, we propose an overload-resolution rule to select the appropriate function based on the context of the call. Given a call, overload resolution prefers non-`async` functions within a synchronous context (because such contexts cannot contain a call to an `async` function).  Furthermore, overload resolution prefers `async` functions within an asynchronous context (because such contexts should avoid stepping out of the asynchronous model into blocking APIs). When overload resolution selects an `async` function, that call is still subject to the rule that it must occur within an `await` expression.

The overload-resolution rule depends on the synchronous or asynchronous context, in which the compiler selects one and only one overload. The selection of the async overload requires an `await` expression, as all introductions of a potential suspension point:

```swift
func f() async {
  // In an asynchronous context, the async overload is preferred:
  await doSomething()
  // Compiler error: Expression is 'async' but is not marked with 'await'
  doSomething()
}
```

In non-`async` functions, and closures without any `await` expression, the compiler selects the non-`async` overload:

```swift
func f() async {
  let f2 = {
    // In a synchronous context, the non-async overload is preferred:
    doSomething()
  }
  f2()
}
```


### Autoclosures

A function may not take an autoclosure parameter of `async` function type unless the function itself is `async`. For example, the following declaration is ill-formed:

```swift
// error: async autoclosure in a function that is not itself 'async'
func computeArgumentLater<T>(_ fn: @escaping @autoclosure () async -> T) { } 
```

This restriction exists for several reasons. Consider the following example:

  ```swift
  // func getIntSlowly() async -> Int { ... }

  let closure = {
    computeArgumentLater(await getIntSlowly())
    print("hello")
  }
  ```

At first glance, the `await` expression implies to the programmer that there is a potential suspension point *prior* to the call to `computeArgumentLater(_:)`, which is not actually the case: the potential suspension point is *within* the (auto)closure that is passed and used within the body of `computeArgumentLater(_:)`. This causes a few problems. First, the fact that `await` appears to be prior to the call means that `closure` would be inferred to have `async` function type, which is also incorrect: all of the code in `closure` is synchronous. Second, because an `await`'s operand only needs to contain a potential suspension point somewhere within it, an equivalent rewriting of the call should be:

```swift
await computeArgumentLater(getIntSlowly())
```

But, because the argument is an autoclosure, this rewriting is no longer semantics-preserving. Thus, the restriction on `async` autoclosure parameters avoids these problems by ensuring that `async` autoclosure parameters can only be used in asynchronous contexts.

### Protocol conformance

A protocol requirement can be declared as `async`. Such a requirement can be satisfied by an `async` or synchronous function. However, a synchronous function requirement cannot be satisfied by an `async` function. For example:

```swift
protocol Asynchronous {
  func f() async
}

protocol Synchronous {
  func g()
}

struct S1: Asynchronous {
  func f() async { } // okay, exactly matches
}

struct S2: Asynchronous {
  func f() { } // okay, synchronous function satisfying async requirement
}

struct S3: Synchronous {
  func g() { } // okay, exactly matches
}

struct S4: Synchronous {
  func g() async { } // error: cannot satisfy synchronous requirement with an async function
}
```

This behavior follows the subtyping/implicit conversion rule for asynchronous functions, as is precedented by the behavior of `throws`.

## Source compatibility

This proposal is generally additive: existing code does not use any of the new features (e.g., does not create `async` functions or closures) and will not be impacted. However, it introduces two new contextual keywords, `async` and `await`.

The positions of the new uses of `async` within the grammar (function declarations and function types) allows us to treat `async` as a contextual keyword without breaking source compatibility. A user-defined `async` cannot occur in those grammatical positions in well-formed code.

The `await` contextual keyword is more problematic, because it occurs within an expression. For example, one could define a function `await` in Swift today:

```swift
func await(_ x: Int, _ y: Int) -> Int { x + y }

let result = await(1, 2)
```

This is well-formed code today that is a call to the `await` function. With this proposal, this code becomes an `await` expression with the subexpression `(1, 2)`. This will manifest as a compile-time error for existing Swift programs, because `await` can only be used within an asynchronous context, and no existing Swift programs have such a context. Such functions do not appear to be common, so we believe this is an acceptable source break as part of the introduction of async/await.

## Effect on ABI stability

Asynchronous functions and function types are additive to the ABI, so there is no effect on ABI stability, because existing (synchronous) functions and function types are unchanged.

## Effect on API resilience

The ABI for an `async` function is completely different from the ABI for a synchronous function (e.g., they have incompatible calling conventions), so the addition or removal of `async` from a function or type is not a resilient change.

## Future Directions

### `reasync`

Swift's `rethrows` is a mechanism for indicating that a particular function is throwing only when one of the arguments passed to it is a function that itself throws. For example, `Sequence.map` makes use of `rethrows` because the only way the operation can throw is if the transform itself throws:

```swift
extension Sequence {
  func map<Transformed>(transform: (Element) throws -> Transformed) rethrows -> [Transformed] {
    var result = [Transformed]()
    var iterator = self.makeIterator()
    while let element = iterator.next() {
      result.append(try transform(element))   // note: this is the only `try`!
    }
    return result
  }
}
```

Here are uses of `map` in practice:

```swift
_ = [1, 2, 3].map { String($0) }  // okay: map does not throw because the closure does not throw
_ = try ["1", "2", "3"].map { (string: String) -> Int in
  guard let result = Int(string) else { throw IntParseError(string) }
  return result
} // okay: map can throw because the closure can throw
```

The same notion could be applied to `async` functions. For example, we could imagine making `map` asynchronous when its argument is asynchronous with `reasync`:

```swift
extension Sequence {
  func map<Transformed>(transform: (Element) async throws -> Transformed) reasync rethrows -> [Transformed] {
    var result = [Transformed]()
    var iterator = self.makeIterator()
    while let element = iterator.next() {
      result.append(try await transform(element))   // note: this is the only `try` and only `await`!
    }
    return result
  }
}
```

*Conceptually*, this is fine: when provided with an `async` function, `map` will be treated as `async` (and you'll need to `await` the result), whereas providing it with a non-`async` function, `map` will be treated as synchronous (and won't require `await`).

*In practice*, there are a few problems here:

* This is probably not a very good implementation of an asynchronous `map` on a sequence. More likely, we would want a concurrent implementation that (say) processes up to number-of-cores elements concurrently.
* The ABI of throwing functions is intentionally designed to make it possible for a `rethrows` function to act as a non-throwing function, so a single ABI entry point suffices for both throwing and non-throwing calls. The same is not true of `async` functions, which have a radically different ABI that is necessarily less efficient than the ABI for synchronous functions.

For something like `Sequence.map` that might become concurrent, `reasync` is likely the wrong tool: overloading for `async` closures to provide a separate (concurrent) implementation is likely the better answer. So, `reasync` is likely to be much less generally applicable than `rethrows`.

There are undoubtedly some uses for `reasync`, such as the `??` operator for optionals, where the `async` implementation degrades nicely to a synchronous implementation:

```swift
func ??<T>(
    _ optValue: T?, _ defaultValue: @autoclosure () async throws -> T
) reasync rethrows -> T {
  if let value = optValue {
    return value
  }

  return try await defaultValue()
}
```

For such cases, the ABI concern described above can likely be addressed by emitting two entrypoints: one when the argument is `async` and one when it is not. However, the implementation is complex enough that the authors are not yet ready to commit to this design.

## Alternatives Considered

### Make `await` imply `try`

Many asynchronous APIs involve file I/O, networking, or other failable operations, and therefore will be both `async` and `throws`. At the call site, this means `try await` will be repeated many times. To reduce the boilerplate, `await` could imply `try`, so the following two lines would be equivalent:

```swift
let dataResource  = await loadWebResource("dataprofile.txt")
let dataResource  = try await loadWebResource("dataprofile.txt")
```

We chose not to make `await` imply `try` because they are expressing different kinds of concerns: `await` is about a potential suspension point, where other code might execute in between when you make the call and it when it returns, while `try` is about control flow out of the block.

One other motivation that has come up for making `await` imply `try` is related to task cancellation. If task cancellation were modeled as a thrown error, and every potential suspension point implicitly checked whether the task was cancelled, then every potential suspension point could throw: in such cases `await` might as well imply `try` because every `await` can potentially exit with an error.
Task cancellation is covered in the [Structured Concurrency](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md) proposal, and does *not* model cancellation solely as a thrown error nor does it introduce implicit cancellation checks at each potential suspension point.

### Launching async tasks

Because only `async` code can call other `async` code, this proposal provides no way to initiate asynchronous code. This is intentional: all asynchronous code runs within the context of a "task", a notion which is defined in the [Structured Concurrency](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md) proposal. That proposal provides the ability to define asynchronous entry points to the program via `@main`, e.g.,

```swift
@main
struct MyProgram {
  static func main() async { ... }
}
```

Additionally, top-level code is not considered an asynchronous context in this proposal, so the following program is ill-formed:

```swift
func f() async -> String { "hello, asynchronously" }

print(await f()) // error: cannot call asynchronous function in top-level code
```

This, too, will be addressed in a subsequent proposal that properly accounts for
top-level variables.

None of the concerns for top-level code affect the fundamental mechanisms of async/await as defined in this proposal.

### Await as syntactic sugar

This proposal makes `async` functions a core part of the Swift type system, distinct from synchronous functions. An alternative design would leave the type system unchanged, and instead make `async` and `await` syntactic sugar over some `Future<T, Error>` type, e.g.,

```swift
async func processImageData() throws -> Future<Image, Error> {
  let dataResource  = try loadWebResource("dataprofile.txt").await()
  let imageResource = try loadWebResource("imagedata.dat").await()
  let imageTmp      = try decodeImage(dataResource, imageResource).await()
  let imageResult   = try dewarpAndCleanupImage(imageTmp).await()
  return imageResult
}
```

This approach has a number of downsides vs. the proposed approach here:

* There is no universal `Future` type on which to build it in the Swift ecosystem. If the Swift ecosystem had mostly settled on a single future type already (e.g., if there were already one in the standard library), a syntactic-sugar approach like the above would codify existing practice. Lacking such a type, one would have to try to abstract over all of the different kinds of future types with some kind of `Futurable` protocol. This may be possible for some set of future types, but would give up any guarantees about the behavior or performance of asynchronous code.
* It is inconsistent with the design of `throws`. The result type of asynchronous functions in this model is the future type (or "any `Futurable` type"), rather than the actual returned value. They must always be `await`'ed immediately (hence the postfix syntax) or you'll end up working with futures when you actually care about the result of the asynchronous operation. This becomes a programming-with-futures model rather than an asynchronous-programming model, when many other aspects of the `async` design intentionally push away from thinking about the futures.
* Taking `async` out of the type system would eliminate the ability to do overloading based on `async`. See the prior section on the reasons for overloading on `async`.
* Futures are relatively heavyweight types, and forming one for every async operation has nontrivial costs in both code size and performance. In contrast, deep integration with the type system allows `async` functions to be purpose-built and optimized for efficient suspension. All levels of the Swift compiler and runtime can optimize `async` functions in a manner that would not be possible with future-returning functions.

## Revision history

* Post-review changes:
   * Replaced `await try` with `try await`.
   * Added syntactic-sugar alternative design.
   * Amended the proposal to allow [overloading on `async`](https://github.com/swiftlang/swift-evolution/pull/1392).
* Changes in the second pitch:
	* One can no longer directly overload `async` and non-`async` functions. Overload resolution support remains, however, with additional justification.
	* Added an implicit conversion from a synchronous function to an asynchronous function.
	* Added `await try` ordering restriction to match the `async throws` restriction.
	* Added support for `async` initializers.
	* Added support for synchronous functions satisfying an `async` protocol requirement.
	* Added discussion of `reasync`.
	* Added justification for `await` not implying `try`.
	* Added justification for `async` following the function parameter list.

* Original pitch ([document](https://github.com/DougGregor/swift-evolution/blob/092c05eebb48f6c0603cd268b7eaf455865c64af/proposals/nnnn-async-await.md) and [forum thread](https://forums.swift.org/t/concurrency-asynchronous-functions/41619)).

## Related proposals

In addition to this proposal, there are a number of related proposals covering different aspects of the Swift Concurrency model:

* [Concurrency Interoperability with Objective-C](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0297-concurrency-objc.md): Describes the interaction with Objective-C, especially the relationship between asynchronous Objective-C methods that accept completion handlers and `@objc async` Swift methods.
* [Structured Concurrency](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md): Describes the task structure used by asynchronous calls, the creation of both child tasks and detached tasks, cancellation, prioritization, and other task-management APIs.
* [Actors](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0306-actors.md): Describes the actor model, which provides state isolation for concurrent programs

## Acknowledgments

The desire for async/await in Swift has been around for a long time. This proposal draws some inspiration (and most of the Motivation section) from an earlier proposal written by
[Chris Lattner](https://github.com/lattner) and [Joe Groff](https://github.com/jckarter), available [here](https://gist.github.com/lattner/429b9070918248274f25b714dcfc7619). That proposal itself is derived from a proposal written by [Oleg Andreev](https://github.com/oleganza), available [here](https://gist.github.com/oleganza/7342ed829bddd86f740a). It has been significantly rewritten (again), and many details have changed, but the core ideas of asynchronous functions have remained the same.

Efficient implementation is critical for the introduction of asynchronous functions, and Swift Concurrency as a whole. Nate Chandler, Erik Eckstein, Kavon Farvardin, Joe Groff, Chris Lattner, Slava Pestov, and Arnold Schwaighofer all made significant contributions to the implementation of this proposal.
