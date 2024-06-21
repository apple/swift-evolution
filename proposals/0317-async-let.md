# `async let` bindings

* Proposal: [SE-0317](0317-async-let.md)
* Authors: [John McCall](https://github.com/rjmccall), [Joe Groff](https://github.com/jckarter), [Doug Gregor](https://github.com/DougGregor), [Konrad 'ktoso' Malawski](https://github.com/ktoso)
* Review Manager: [Ben Cohen](https://github.com/airspeedswift)
* Status: **Implemented (Swift 5.5)**

## Introduction

[Structured concurrency](0304-structured-concurrency.md) provides a paradigm for spawning concurrent *child tasks* in scoped *task groups*, establishing a well-defined hierarchy of tasks which allows for cancellation, error propagation, priority management, and other tricky details of concurrency management to be handled transparently. 

This proposal aims to make the common task of spawning child tasks to run asynchronously and pass their eventual
results up to their parent, using lightweight syntax similar to `let` bindings.

Discussion threads:

- Originally pitched as part of Structured Concurrency:
  - [Pitch #1](https://forums.swift.org/t/concurrency-structured-concurrency/41622),
  - [Pitch #2](https://forums.swift.org/t/pitch-2-structured-concurrency/43452).
- and later separated into its own proposal:
  - [Pitch #3](https://forums.swift.org/t/pitch-3-async-let/48336).
- Separate discussion on [scoped suspension points](https://forums.swift.org/t/async-let-and-scoped-suspension-points/49846).

[TOC]



## Motivation

In [SE-0304: Structured Concurrency](0304-structured-concurrency.md) we introduced the concept of tasks and task groups, which can be used to spawn multiple concurrently executing child-tasks and collect their results before exiting out of the task group.

Task groups are a very powerful, yet low-level, building block useful for creating powerful parallel computing patterns, such as collecting the "first few" successful results, and other typical fan-out or scatter/gather patterns. They work best for spreading out computation of same-typed operations. For example, a parallelMap could be implemented in terms of a TaskGroup. In that sense, task groups are a low level implementation primitive, and not the end-user API that developers are expected to interact with a lot, rather, it is expected that more powerful primitives are built on top of task groups.

Task Groups also automatically propagate task cancellation, priority, and task-local values through to child-tasks and offer a flexible API to collect results from those child-tasks _in completion order_, which is impossible to achieve otherwise using other structured concurrency APIs. They do all this while upholding the structured concurrency guarantees that a child-task may never "out-live" (i.e. keep running after the task group scope has exited) the parent task.

While task groups are indeed very powerful, they are hard to use with *heterogeneous results* and step-by-step initialization patterns. 

The following example, an asynchronous `makeDinner` function, consists of both of those patterns. It consists of three tasks which can be performed in parallel, all yielding different result types. To proceed to the final step of the cooking process, all those results need to be obtained, and fed into the final `oven.cook(...)` function. In a way, this is the trickiest situation to implement well using task groups. Let us examine it more closely:

```swift
func makeDinner() async -> Meal {
  // Create a task group to scope the lifetime of our three child tasks
  return try await withThrowingTaskGroup(of: CookingTask.self) { group in
    // spawn three cooking tasks and execute them in parallel:
    group.async {
      CookingTask.veggies(try await chopVegetables())
    }
    group.async {
      CookingTask.meat(await marinateMeat())
    }
    group.async {
      CookingTask.oven(await preheatOven(temperature: 350))
    }

    // prepare variables to collect the results
    var veggies: [Vegetable]? = nil
    var meat: Meat? = nil
    var oven: Oven? = nil

    // collect the results
    for try await task in group {
      switch task {
      case .veggies(let v):
        veggies = v
      case .meat(let m):
        meat = m
      case .oven(let o):
        oven = o
      }
    }

    // ensure every variable was initialized as expected
    assert(veggies != nil)
    assert(meat != nil)
    assert(oven != nil)

    // prepare the ingredients
    var ingredients: [Ingredient] = veggies!
    ingredients.append(meat!)

    // and, finally, cook the meal, awaiting inside the group
    let dish = Dish(ingredients: ingredients)
    return try await oven!.cook(dish, duration: .hours(3))
  }
}
```

The `withThrowingTaskGroup` scope explicitly delineates any potential concurrency, because it guarantees that any child tasks spawned within it are awaited on as the group scope exits. Any results can be collected by iterating through the group. Errors and cancellation are handled automatically for us by the group.

However, this example showcases the weaknesses of the TaskGroups very well: heterogeneous result processing and variable initialization become very boilerplate heavy. While there exist ideas to make this boilerplate go away in future releases, with smarter analysis and type checking, the fundamental issue remains. 

If we step back a little, we can notice that in the example each child task is really producing a *single value* and returning it back to the *parent task*, which then needs to assemble the pieces and proceed with calling some other function. We achieve this by preparing, and assigning into `Optional` variables dedicated for each of the spawned tasks. This is not ideal, since
although the code is correct as written, modifying this code to add a variable is not only boilerplate heavy, but also potentially quite error prone, leading to runtime crashes due to the force-unwraps which a well written Swift program usually would not have to resort to. 

This dataflow pattern from child tasks to parent is very common, and we want to make it as lightweight and safe as possible.

## Proposed solution

This proposal introduces a simple way to create child tasks and await their results: `async let` declarations.

Using `async let`, our example looks like this:

```swift
// given: 
//   func chopVegetables() async throws -> [Vegetables]
//   func marinateMeat() async -> Meat
//   func preheatOven(temperature: Int) async -> Oven

func makeDinner() async throws -> Meal {
  async let veggies = chopVegetables()
  async let meat = marinateMeat()
  async let oven = preheatOven(temperature: 350)

  let dish = Dish(ingredients: await [try veggies, meat])
  return try await oven.cook(dish, duration: .hours(3))
}
```

`async let` is similar to a `let`, in that it defines a local constant that is initialized by the expression on the right-hand side of the `=`. However, it differs in that the initializer expression is evaluated in a separate, concurrently-executing child task. 

The child task begins running as soon as the `async let` is encountered. By default, child tasks use the global, width-limited,  concurrent executor, in the same manner as task group child-tasks do. It is a future direction to allow customizing which executor these should be executing on. On normal completion, the child task will initialize the variables in the `async let`.

The right-hand side of a `async let` expression can be thought of as an implicit `@Sendable closure`, similar to how the `Task.detached { ... }` API works, however the resulting task is a *child task* of the currently executing task. Because of this, and the need to suspend to await the results of such expression, `async let` declarations may only occur within an asynchronous context, i.e. an `async` function or closure.

For single statement expressions in the `async let` initializer, the `await` and `try` keywords may be omitted. The effects they represent carry through to the introduced constant and will have to be used when waiting on the constant. In the example shown above, the veggies are declared as `async let veggies = chopVegetables()`, and even through `chopVegetables` is `async` and `throws`, the `await` and `try` keywords do not have to be used on that line of code. Once waiting on the value of that `async let` constant, the compiler will enforce that the expression where the `veggies` appear must be covered by both `await` and some form of `try`.

Because the main body of the function executes concurrently with its child tasks, it is possible that the parent task (the body of `makeDinner` in this example) will reach the point where it needs the value of a `async let` (say,`veggies`) before that value has been produced. To account for that, reading a variable defined by a `async let` is treated as a potential suspension point,
and therefore must be marked with `await`. 

## Detailed design

### Declaring `async let` constants

`async let` declarations are similar to `let` declarations, however they can only appear in specific contexts.

Because the asynchronous task must be able to be awaited on in the scope it is created, it is only possible to declare `async let`s in contexts where it would also be legal to write an explicit `await`, i.e. asynchronous functions:

```swift
func greet() async -> String { "hi" }

func asynchronous() async {
  async let hello = greet()
  // ... 
  await hello
}
```

and inside asynchronous closures:

```swift
func callMe(_ maybe: () async -> String) async -> String 
  return await maybe()
}

callMe { // async closure
  async let hello = greet()
  // ... 
  return await hello
}
```

It is not allowed to declare `async let` as top-level code, in synchronous functions or closures:

```swift
async let top = ... // error: 'async let' in a function that does not support concurrency

func sync() { // note: add 'async' to function 'sync()' to make it asynchronous
  async let x = ... // error: 'async let' in a function that does not support concurrency
}

func syncMe(later: () -> String) { ... }
syncMe {
  async let x = ... // error: invalid conversion from 'async' function of type '() async -> String' to synchronous function type '() -> String'
}
```

A `async let` creates a child-task, which inherits its parent task's priority as well as task-local values. Semantically, this is equivalent to creating a one-off `TaskGroup` which spawns a single task and returns its result, however the implementation of `async let`s can make more assumptions and optimizations around the lifetime and usage of those values.

The child-task created to initialize the `async let` by default runs on the global concurrent, width-limited, executor that comes with the Swift Concurrency runtime. 

> Customizing the execution context of async lets is a future direction we are likely to explore with the introduction of Custom Executors.

The initializer of the `async let` can be thought of as a closure that runs the code contained within it in a separate task, very much like the explicit `group.async { <work here/> }` API of task groups.

Similarly to the `group.async()` function, the closure is `@Sendable` and `nonisolated`, meaning that it cannot access non-sendable state of the enclosing context. For example, it will result in a compile-time error, preventing a potential race condition, for a `async let` initializer to attempt mutating a closed-over variable:

```swift
var localText: [String] = ...
async let w = localText.removeLast() // error: mutation of captured var 'localText' in concurrently-executing code
```

The async let initializer may refer to any sendable state, same as any non-isolated sendable closure.

The initializer of a `async let` permits the omission of the `await` keyword if it is directly calling an asynchronous function, like this:

```swift
func order() async -> Order { ... }

async let o1 = await order()
// should be written instead as
async let o2 = order()
```

This is because by looking at the async let declaration, it is obvious that the right-hand side function will be used to initialize the left-hand side, by waiting on it. This is similar to single-expression `return` keyword omission, and also applies only to single expression initializers.

It is illegal to declare an `async var`. This is due to the complex initialization that a `async let` represents, it does not make sense to allow further external modification of them. Doing so would tremendously complicate the understandability of such asynchronous code, and undermine potential optimizations by making it harder to make assumptions about the data-flow of the values.

```swift
async var x = nope() // error: 'async' can only be used with 'let' declarations
```

Other than having to be awaited to access its value, a `async let` behaves just like a typical `let`, as such it is not possible to pass it `inout` to other functions - simply because it is a `let`, and those may not be passed as `inout`.

#### Declaring `async let` with patterns

It is possible to create a `async let` where the left-hand side is a pattern, e.g. a tuple, like this:

```swift
func left() async -> String { "l" }
func right() async -> String { "r" }

async let (l, r) = (left(), right())

await l // at this point `r` is also assumed awaited-on
```

To understand the execution semantics of the above snippet, we can remember the sugaring rule that the right-hand side of a `async let` effectively is just a concurrently executing asynchronous closure:

```swift
async let (l, r) = {
  return await (left(), right())
  // -> 
  // return (await left(), await right())
}
```

meaning that the entire initializer of the `async let` is a single task, and if multiple asynchronous function calls are made inside it, they are performed one-by one. This is a specific application of the general rule of `async let` initializers being allowed to omit a single leading `await` keyword before their expressions. Because in this example, we invoke two asynchronous functions to form a tuple, the `await` can be moved outside the expression, and that await is what is omitted in the shorthand form of the `async let` that we've seen in the first snippet.

This also means that as soon as we enter continue past the line of `await l` it is known that the `r` value also has completed successfully (and will not need to emit an "implicit await" which we'll discuss in detail below).

Another implication of these semantics is that if _any_ piece of the initializer throws, any await on such pattern declared `async let` shall be considered throwing, as they are initialized "together". To visualize this, let us consider the following:

```swift
async let (yay, nay) = ("yay", throw Boom())
try await yay // because the (yay, nay) initializer is throwing
```

Because we know that the right-hand side is simply a single closure, performing the entire initialization, we know that if any of the operations on the right-hand size is throwing, the entire initializer will be considered throwing. As such, awaiting even the `yay` here must be ready for that initializer to have thrown and therefore must include the `try` keyword in addition to `await`.

### Awaiting `async let` values

Since `async let`s introduce constants that will be "filled in later" by their right-hand-side concurrently-executing task, referring to them must be covered by an `await` keyword:

```swift
async let name = getName() 
async let surname = getSurname() 
await name
await surname
```

It is also possible to simply cover the entire expression where a `async let` is used with just a single `await`, similar to how the same can be done with `try`:

```swift
greet(await name, await surname)
await greet(name, surname)
// or even
await print("\(name) \(surname)")
```

If the initializer of the specific `async let` was throwing, then awaiting on the `async let` constant must be covered using a variant of the `try` keyword:

```swift
async let ohNo = throwThings()
try await ohNo
try? await ohNo
try! await ohNo
```

Currently, it is required to cover every reference to a `async let` using the appropriate `try` and `await` keywords, like this:

```swift
async let yes = ""
async let ohNo = throwThings()

_ = await yes
_ = await yes
_ = try await ohNo
_ = try await ohNo
```

This is a simple rule and allows us to bring the feature forward already. It might be possible to employ control flow based analysis to enable "only the first reference to the specific `async let` on each control flow path has to be an `await`", as technically speaking, every following await will be a no-op and will not suspend as the value is already completed, and the placeholder has been filled in.

### Implicit `async let` awaiting 

A `async let` that was declared but never awaited on *explicitly* as the scope in which it was declared exits, will be awaited on implicitly. These semantics are put in place to uphold the Structured Concurrency guarantees provided by `async let`.

To showcase these semantics, let us have a look at this function which spawns two child tasks, `fast` and `slow` but does not await on any of them:

```swift
func go() async { 
  async let f = fast() // 300ms
  async let s = slow() // 3seconds
  return "nevermind..."
  // implicitly: cancels f
  // implicitly: cancels s
  // implicitly: await f
  // implicitly: await s
}
```

Assuming the execution times of `fast()` and `slow()` are as the comments next to them explain, the `go()` function will _always_ take at least 3 seconds to execute. Or to state the rule more generally, any structured invocation will take as much time to return as the longest of its child tasks takes to complete.

As we return from the `go()` function without ever having awaited on the `f` or `s` values, both of them will be implicitly cancelled and awaited on before returning from the function `go()`. This is the very nature of structured concurrency, and avoiding this can _only_ be done by creating non-child tasks, e.g. by using `Task.detached` or other future APIs which would allow creation of non-child tasks.

If we instead awaited on one of the values, e.g. the fast one (`f`) the emitted code would not need to implicitly cancel or await it, as this was already taken care of explicitly:

```swift
func go2() async {
  async let f = fast()
  async let s = slow()
  _ = await f
  return "nevermind..."
  // implicitly: cancels s
  // implicitly: awaits s
}
```

The duration of the `go2()` call remains the same, it is always `time(go2) == max(time(f), time(s))`.

Special attention needs to be given to the `async let _ = ...` form of declarations. This form is interesting because it creates a child-task of the right-hand-side initializer, however it actively chooses to ignore the result. Such a declaration (and the associated child-task) will run and be cancelled and awaited-on implicitly, as the scope it was declared in is about to exit — the same way as an unused `async let` declaration would be.

### `async let` and closures

Because `async let` tasks cannot out-live the scope in which they are defined, passing them to closures needs some further discussion for what is legal and not.

It is legal to capture a `async let` in a non-escaping asynchronous closure, like this:

```swift
func greet(_ f: () async -> String) async -> String { await f() }

async let name = "Alice"
await greet { await name }
```

Notice how we are required to write the `await` inside the closure as well as in front of the `greet` function. This is on purpose as we do want to be explicit about the `await` inside the closure. 

The same applies to auto closures, in order to make it explicit that the `await` is happening _inside_ the closure rather than before it, it is required to await explicitly in parameter position where the auto closure is formed for the argument:

```swift
func greet(_ f: @autoclosure () async -> String) async -> String { await f() }

async let name = "Bob"
await greet(await name) // await on name is required, because autoclosure
```

It is *not* legal to escape a `async let` value to an escaping closure. This is because structures backing the async let implementation may be allocated on the stack rather than the heap. This makes them very efficient, and makes great use of the structured guarantees they have to adhere to. These optimizations, however, make it unsafe to pass them to any escaping contexts:

```swift
func greet(_ f: @escaping () async -> String) async -> String { somewhere = f; somewhere() }

async let name = "Bob"
await greet { await name } // error: cannot escape 'async let' value
```



### `async let` error propagation

While it is legal to declare a `async let` and never explicitly `await` on it, it also implies that we do not particularly care about its result.

This is the same as spawning a number of child-tasks in a task group, and not collecting their results, like so:

```swift
try await withThrowingTaskGroup(of: Int.self) { group in 
  group.async { throw Boom() }
                             
  return 0 // we didn't care about the child-task at all(!)
} // returns 0
```

The above TaskGroup example will ignore the `Boom` thrown by its child task. However, it _will_ await for the task (and any other tasks it had spawned) to run to completion before the `withThrowingTaskGroup` returns. If we wanted to surface all potential throws of tasks spawned in the group, we should have written: `for try await _ in group {}` which would have re-thrown the `Boom()`.

The same concept carries over to `async let`, where the scope of the group is replaced by the syntactic scope in which the `async let` was declared. For example, the following snippet is semantically equivalent to the above TaskGroup one:

```swift
// func boom() throws -> Int { throw Boom() }

func work() async -> Int {
  async let work: Int = boom()
  // never await work...
  return 0
  // implicitly: cancels work
  // implicitly: awaits work, discards errors
}
```

This `work()` function will never throw, because we didn't await on the throwing `async let`. If we modified it to explicitly await on it, the compiler would force us to spell out not only the `await` but also the `try` keyword. The presence of the `try` keyword would then force us to annotate the `work()` function as `throws`, as expected from normal, non-asynchronous code in Swift:

```swift
// func boom() throws -> Int { throw Boom() }

func work() async throws -> Int { // throws is enforced due to 'try await'
  async let work: Int = boom()
  // ... 
  return try await work // 'try' is enforced since 'boom()' was throwing
}
```

Alternatively, we could have handled the error of `work` by wrapping it in a `do/catch`.

### Cancellation and `async let` child tasks

Cancellation propagates recursively through the task hierarchy from parent to child tasks.

Because tasks spawned by `async let` are child tasks, they naturally participate in their parent's cancellation.

Cancellation of the parent task means that the context in which the `async let` declarations exist is cancelled, and any tasks created by those declarations will be cancelled as well. Because cancellation in Swift is co-operative, it does not prevent the spawning of tasks, however tasks spawned from a cancelled context are *immediately* marked as cancelled. This exhibits the same semantics as `TaskGroup.async` which, when used from an already cancelled task, _will_ spawn more child-tasks, however they will be immediately created as cancelled tasks — which they can inspect by calling `Task.isCancelled`.

We can observe this in the following example:

```swift
let handle = Task.detached { 
  // don't write such spin loops in real code (!!!)
  while !Task.isCancelled {
    // keep spinning
    await Task.sleep(...)
  }
  
  assert(Task.isCancelled) // parent task is cancelled
  async let childTaskCancelled = Task.isCancelled // child-task is spawned and is cancelled too
  
  assert(await childTaskCancelled)
}

handle.cancel() 
```

The example uses APIs defined in the Structured Concurrency proposal: `Task.detached` to obtain a handle for the detached task which we can cancel explicitly. This allows us to easily illustrate that a `async let` entered within a task that _already is cancelled_ still spawns the child task, yet the spawned task will be immediately cancelled - as witnessed by the `true` returned into the `childTaskCancelled` variable.

This works well with the co-operative nature of task cancellation in Swift's concurrency story. Tasks which are able and willing to participate in cancellation handling, need to check for its status using `Task.isCancelled` or `try Task.checkCancellation()` where appropriate.

### Analysis of limitations and benefits of `async let`

#### Comparing with `TaskGroup`

Semantically, one might think of a `async let` as sugar for manually using a task group, spawning a single task within it and collecting the result from `group.next()` wherever the `async let` declared value is `await`-ed on. As we saw in the [Motivation](#motivation) section of the proposal, such explicit usage of groups ends up very verbose and error prone in practice, thus the need for a "sugar" for the specific pattern.

A `async let` declaration, in reality, is not just a plain sugar-syntax for task groups, and can make use of additional known-at-compile-time structure of the declared tasks. For example, it is possible to avoid heap allocations for small enough `async let` child tasks, avoid queues and other mechanisms which a task group must make use of to implement its "by completion order" yielding of values out of `next()`. 

This comes at a price though, async let declarations are less flexible than groups, and this is what we'll explore in this section.

Specifically, `async let` declarations are not able to express dynamic numbers of tasks executing in parallel, like this group showcases:

```swift
func toyParallelMap<A, B>(_ items: [A], f: (A) async -> B) async -> [B] { 
  return await withTaskGroup(of: (Int, B).self) { group in
    var bs = [B?](repeating: nil, count: items.count)
    
    // spawn off processing all `f` mapping functions in parallel
    // in reality, one might want to limit the "width" of these
    for i in items.indices { 
      group.async { (i, await f(items[i])) }
    }
    
    // collect all results
    for await (i, mapped) in group {
      bs[i] = mapped
    }
    
    return bs.map { $0! }
  }
}
```

In the above `toyParallelMap` the number of child-tasks is _dynamic_ because it depends on the count of elements in the `items` array _at runtime_. Such patterns are not possible to express using `async let` because we'd have to know how many `async let` declarations to create *at compile time*. One might attempt to simulate these by:

```swift
// very silly example to show limitations of `async let` when facing dynamic numbers of tasks
func toyParallelMapExactly2<A, B>(_ items: [A], f: (A) async -> B) async -> [B] { 
  assert(items.count == 2)
  async let f0 = f(items[0])
  async let f1 = f(items[1])
  
  return await [f0, f1]
}
```

And while the second example reads very nicely, it cannot work in practice to implement such parallel map function, because the size of the input `items` is not known (and we'd have to implement `1...n` versions of such function).

Another API which is not implementable with `async let` and will require using a task group is anything that requires some notion of completion order. Because `async let` declarations must be awaited on it is not possible to express "whichever completes first", and a task group must be used to implement such API. 

For example, the `race(left:right:)` function shown below, runs two child tasks in parallel, and returns whichever completed first. Such API is not possible to implement using async let and must be implemented using a group:

```swift
func race(left: () async -> Int, right: () async -> Int) async -> Int {
  await withTaskGroup(of: Int.self) { group in 
    group.async { left() }
    group.async { right() }

    let first = await group.next()! // !-safe, there is at-least one result to collect
    group.cancelAll() // cancel the other task
    return first
  }
}
```

#### Comparing with Task, and (not proposed) futures

It is worth comparing `async let` declarations with the one other API proposed so far that is able to start asynchronous tasks: `Task {}`, and `Task.detached {}`, proposed in [SE-0304: Structured Concurrency](0304-structured-concurrency.md).

First off, `Task.detached` most of the time should not be used at all, because it does _not_ propagate task priority, task-local values or the execution context of the caller. Not only that but a detached task is inherently not _structured_ and thus may out-live its defining scope.

This immediately shows how `async let` and the general concept of child-tasks are superior to detached tasks. They automatically propagate all necessary information about scheduling and metadata necessary for execution tracing. And they can be allocated more efficiently than detached tasks.

So while in theory one can think of `async let` as introducing a (hidden) `Task` or future, which is created at the point of declaration of the `async let` and whose value is retrieved at the `await` in practice, this comparison fails to notice the primary strength of async lets: structured concurrency child-tasks.

Child tasks in the proposed structured-concurrency model are (intentionally) more restricted than general-purpose futures. Unlike in a typical futures' implementation, a child task does not persist beyond the scope in which it was created. By the time the scope exits, the child task must either have completed, or it will be implicitly awaited. When the scope exits via a thrown error, the child task will be implicitly cancelled before it is awaited. These limitations intentionally preserve the same properties of structured concurrency that explicit task groups provide.

It is also on purpose, and unlike Tasks and futures that it is not possible to pass a "still being computed" value to another function. With handles or futures one is quite used to "pass the handle" to another function like this:

```swift
func take(h: Task<String, Error>) async -> String {
  return await h.get()
}
```


## Source compatibility

This change is purely additive to the source language.

## Effect on ABI stability

This change is purely additive to the ABI.

## Effect on API resilience

All the changes described in this document are additive to the language and are locally scoped, e.g., within function bodies. Therefore, there is no effect on API resilience.

## Future directions

### Await in closure capture lists

Because a `async let` cannot be closed over by an escaping closure, as it would unsafely extend its lifetime beyond the lifetime of the function in which it was declared, developers who need to wait for a value of an `async let` before passing it off to an escaping closure will have to write:

```swift
func run() async { 
  async let alcatraz = "alcatraz"
  // ... 
  escapeFrom { // : @escaping () async -> Void
    alcatraz // error: cannot refer to 'async let' from @escaping closure
  }
  // implicitly: await alcatraz
}
```

The only legal way to achieve this in the present proposal is to introduce another value and store the awaited value in it:

```swift
func run() async { 
  async let alcatraz = "alcatraz"
  // ... 
  let awaitedAlcatraz = await alcatraz
  escapeFrom { // : @escaping () async -> Void
    awaitedAlcatraz // ok
  }
}
```

This is correct, yet slightly annoying as we had to invent a new name for the awaited value. Instead, we could utilize capture lists enhanced with the ability to await on such value _at the creation point of the closure_:

```swift
func escapeFrom(_ f: @escaping () -> ()) -> () {}

func run() async { 
  async let alcatraz = "alcatraz"
  // ... 
  escapeFrom { [await alcatraz] in // value awaited on at closure creation
    alcatraz // ok
  }
}
```

This snippet is semantically equivalent to the one before it, in that the `await alcatraz` happens before the `escapeFrom` function is able to run. 

While it is only a small syntactic improvement over the second snippet in this section, it is a welcome and consistent one with prior patterns in swift, where it is possible to capture a `[weak variable]` in closures.

The capture list is only necessary for `@escaping` closures, as non-escaping ones are guaranteed to not "out-live" the scope from which they are called, and thus cannot violate the structured concurrency guarantees an `async let` relies on.

### Custom executors and `async let` 

It is reasonable to request that specific `async let` initializers run on specific executors. 

While this usually not necessary to actor based code, because actor invocations will implicitly "hop" to the right actor as it is called, like in the example below:

```swift
actor Worker { func work() {} }
let worker: Worker = ...

async let x = worker.work() // implicitly hops to the worker to perform the work
```

The reasons it may be beneficial to specify an executor child-tasks should run are multiple, and the list is by no means exhaustive, but to give an idea, specifying the executor of child-tasks may:

- pro-actively fine-tune executors to completely avoid any thread and executor hopping in such tasks,
- execute child-tasks concurrently however _not_ in parallel with the creating task (e.g. make child tasks run on the same serial executor as the calling actor),
- if the child-task work is known to be heavy and blocking, it may be beneficial to delegate it to a specific "blocking executor" which would have a dedicated, small, number of threads on which it would execute the blocking work; Thanks to such separation, the main global thread-pool would not be impacted by starvation issues which such blocking tasks would otherwise cause.
- various other examples where tight control over the execution context is required...

We should be able to allow such configuration based on scope, like this:

```swift
await withTask(executor: .globalConcurrentExecutor) { 
  async let x = ...
  async let y = ...
  // x and y execute in parallel; this is equal to the default semantics
}

actor Worker {
  func work(first: Work, second: Work) async {
    await withTask(executor: self.serialExecutor) {
      // using any serial executor, will cause the tasks to be completed one-by-one,
      // concurrently, however without any real parallelism.
      async let x = process(first)
      async let y = process(second)
      // x and y do NOT execute in parallel
    }
  }
}
```

The details of the API remain to be seen, but the general ability to specify an executor for child-tasks is useful and will be considered in the future.

## Alternatives considered

### Explicit futures

As discussed in the [structured concurrency proposal](0304-structured-concurrency.md#prominent-futures), we choose not to expose futures or `Task`s for child tasks in task groups, because doing so either can undermine the hierarchy of tasks, by escaping from their parent task group and being awaited on indefinitely later, or would result in there being two kinds of future, one of which dynamically asserts that it's only used within the task's scope. `async let` allows for future-like data flow from child tasks to parent, without the need for general-purpose futures to be exposed.

### "Don't spawn tasks when in cancelled parent"

It would be very confusing to have `async let` tasks automatically "not run" if the parent task were cancelled. Such semantics are offered by task groups via the `group.asyncUnlessCancelled` API, however would be quite difficult to express using plain `let` declarations, as effectively all such declarations would have to become implicitly throwing, which would sacrifice their general usability. We are convinced that following through with the co-operative cancellation strategy works well for `async let` tasks, because it composes well with how all asynchronous functions should be handling cancellation to begin with: only when they want to, in appropriate places within their execution, and deciding by themselves if they prefer to throw a `Task.CancellationError` or rather return a partial result when cancellation occurs.

### Requiring an `await`on any execution path that waits for an `async let`

In initial versions of this proposal, we considered a rule to force an `async let` declaration to be awaited on each control-flow path that the execution of a function might take. This rule turned out to be too simplistic, because it isn't generally possible to annotate all of the control-flow edges that would result in waiting for a child task to complete. The most problematic case involves a control-flow edge due to a thrown exception, e.g.,

```swift
func runException() async {
  do {
    async let a = f()
    try mayFail() // no way to "await a" only along the thrown-error edge; it is an implicit suspension point
    ... await a ...
  } catch {
    ...
  }
}
```

When `mayFail()` returns normally, we'll later `await a` so that `async let` will be associated with an explicit suspension point. However, when `mayFail()` throws an error, control flow jumps to the `catch` block and must wait for the child task that produces `a` to complete. This latter suspension point is implicit, and there is no direct way to make it explicit that doesn't also involve moving the definition of `a` outside of the `do...catch` block. 

There are other places where there are control-flow edges that will implicitly await the child tasks for `async let`s in scope, e.g., a function with an `async let` in a loop:

```swift
func runLoop() async {
  for e in list {
    async let a = f(e)
    guard <condition> else {
      break // cancels and implicitly awaits the task that produces "a"
    }
    ... await a ...
  }
  foo()
}
```

The most promising approach to marking all `async let` suspension points explicitly involves marking the control-flow edges that can result in a potential suspension point with `await`. For the most recent example, this means using `await break`:

```swift
func runLoop() async {
  for e in list {
    async let a = f(e)
    guard <condition> else {
      await break   // awaits the child task that produces the value "a"
    }
    ... await a ...
  }
  foo()
}
```

One would similarly need an `await continue`. For the first example, this means marking the call to `mayFail()` with an `await`, because the potentially-throwing call creates a control-flow edge out of the scope:

```swift
func runException() async {
  do {
    async let a = f()
    try await mayFail() // awaits the child task that produces a; mayFail() itself may not even be "async"
    ... await a ...
  } catch {
    ...
  }
}
```

It is somewhat ambiguous what `try await` means in this case, because `mayFail()` may or may not be `async` at all. If it is, then `await` does double-duty covering both the potential suspension points for the call to `mayFail()` as well as the potential suspension point when waiting for the child task along the thrown-error control-flow-edge.

Similarly, one would need `await throw` for cases where a directly-thrown expression would imply a suspension point to wait for an `async let` child task to complete:

```swift
func runThrow() async {
  do {
    async let a = f()
    if <condition> {
      await throw SomeError() // awaits the child task that produces a
    }
    ... await a ...
  } catch {
    ...
  }
}
```

However, not all control-flow edges involving implicit `async let` suspension points have a specific keyword to which we can attach `await`, because some come from fall-through to subsequent code. For such cases, one could have a standalone `await` statement marking that fall through:

```swift
func runIfFallthrough() async {
  if <condition> {
    async let a = f()
    ... code ...
    // falling out of this block must await the child task that produces a, so require a freestanding "await"
    await
  }
  ... more code ...
}
```

The same would be required in, e.g., the cases of a `switch` statement that introduce an `async let`:

```swift
func runSwitchCase() async {
  switch <expression> {
  case .a:
    async let a = f()
    // falling out of this block must await the child task that produces a, so require a freestanding "await"
    await

  default:
    ... code ...
  }
  ... code ...
}
```

The above is a significant expansion of the grammar: introducing the `await` keyword in front of `break`, `continue`, `throw`, and `fallthrough`; requiring `await` on certain throwing expressions that don't otherwise involve `async` operations; and adding the freestanding `await` statement. It would also need to be coupled with rules that only require the new `await` when it is semantically meaningful. For example, the additional `await` shouldn't be required if all of the `async let` child tasks have already been explicitly awaited in some other manner, e.g.,

```swift
func runIfFallthroughOkay() async {
  if <condition> {
    async let a = f()
    ... code ...
    if <other condition> {
      ... await a ...
    } else {
      ... await a ...
    }
    // no need for "await" here because we've already waited for "a" along all paths
  }
  ... more code ...
}
```

Additionally, every `async` function is already called with an `await`, which covers any suspension points that occur when the function exits. Therefore, a control-flow edge that exits the function should not require any additional `await` for any `async let` child tasks that are awaited. For this reason, there is no `await return`. It also means that other control-flow edges that exit the function need not be annotated. For example:

```swift
func runThrowsOkay() async {
  async let a = f()
  if <condition> {
    throw SomeError() // no need for "await" because this edge exits the function
  } 

  // no need for "await" at the end because we are exiting the function
}
```

The rules above attempt to limit the places in which the new `await` syntaxes are required to only those where they are semantically meaningful, i.e., those places where the `async let` child tasks will not already have had their completion explicitly awaited. The rules are complicated enough that we would not expect programmers to be able to correctly write `await` in all of the places where it is required. Rather, the Swift compiler would need to provide error messags with Fix-Its to indicate the places where additional `await` annotations are required, and those `await`s will remain as an artifact for the reader.

We feel that the complexity of the solution for marking all suspension points, which includes both the grammar expansion for marking control-flow edges and the flow-sensitive analysis to only require the additional `await` marking when necessary, exceeds the benefits of adding it. Instead, we feel that the presence of `async let` in a block with complicated control flow is sufficient to imply the presence of additional suspension points.

### Property wrappers instead of `async let`

The combination of [property wrappers](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0258-property-wrappers.md) and [effectful properties](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0310-effectful-readonly-properties.md) implies that one could approximate the behavior of `async let` with a property wrapper, e.g.,

```swift
@AsyncLet var veggies = try await chopVegetables()
```

One problem with this approach is that property wrappers cannot provide the semantics of structured concurrency. This becomes more apparent when trying to implement such a property wrapper:

```swift
@propertyWrapper
class AsyncLet<Wrapped: Sendable> {
  var task: Task<Wrapped, Error>
  
  init(wrappedValue fn: @Sendable @escaping @autoclosure () async throws -> Wrapped) {
    self.task = Task.detached {  // have to produce a detached task; cannot create a child task
      try await fn()
    }
  }
  
  var wrappedValue: Wrapped {
    get async throws {
      try await task.value
    }
  }
  
  deinit {
    // we can cancel the task...
    task.cancel()
    
    // ... but we cannot wait for it to complete, because deinits cannot be async
  }
}
```

A property-wrapper approach is forced to create unstructured concurrency to capture the task, which is then subject to escaping (e.g.,  the synthesized backing storage property `_veggies`). Once we have unstructured concurrency, there is no way to get the structure back: the deinitializer cannot wait on completion of the task, so the task would keep running after the `@AsyncLet` property has been destroyed. The lack of structure also affects the compiler's ability to reason about (and therefore optimize) the use of this feature: as a structured concurrency primitive, `async let` can be optimized by the compiler to (e.g.) share storage of its async stack frames with its parent async task, eliminating spurious allocations, and provide more optimal access patterns for the resulting value. To address the semantic and performance issues with using property wrappers, an `@AsyncLet` property wrapper would effectively be hard-coded syntax in the compiler that is property-wrapper-like, but not actually a property wrapper.

One thing that is lost with the property-wrapper approach that the definition of a property such as

```swift
@AsyncLet var veggies = try await chopVegetables()
```

loses the `async` keyword. With `async let`, the names introduced are clearly `async` and therefore must be `await`'ed when they are used, as with other `async` entities in the language:

```swift
async let veggies = chopVegetables()
...
await veggies
```

### Braces around the `async let` initializer

The expression on the right-hand side of an `async let` declaration is executed in a separate, child task that is running concurrently with the function that initiates the `async let`. It has been suggested that the task should be called out more explicitly by adding a separate set of braces around the expression, e.g.,

```swift
async let veggies = { try await chopVegetables() }
```

The problem with requiring braces is that it breaks the equivalence between the type of the entity being declared (`veggies` is of type `[Vegetable]`) and the value it is initialized with (which now appears to be `@Sendable () async throws -> [Vegetable]`). This equivalence holds throughout nearly all of the language; the only real exception is the `if let` syntax, which which strips a level of optionality and is often considered a design mistake in Swift. For `async let`, requiring the braces would become particularly awkward if one were defining a value of closure type:

```swift
async let closure = { { try await getClosure() } }
```

Requiring braces on the right-hand side of `async let` would be a departure from Swift's existing precedent with `let` declarations. In the cases where one is defining a syntactically larger child task, it is reasonable to create and immediately call a closure, which is common practice with `lazy` variables:

```swift
async let image: Image = {
  let data = try await download(url: url)
  return try await Image(from: data)
}()
```

## Revision history

After the first review:

* Expanded the discussion of implicit suspension points in Alternatives Considered with a more comprehensive design sketch for making all suspension points explicit.
* Added discussion of the use of property wrappers instead of `async let` to Alternatives Considered.
* Added discussion about requiring braces around an `async let` initializer expression to Alternatives Considered.

After initial pitch (as part of Structured Concurrency):

- renamed back to `async let` to be consistent with updated naming in structured concurrency APIs, 
- renamed `async let` to `spawn let` to be consistent with `spawn` usage in the rest of structured concurrency APIs,
- added details of cancellation handling
- added details of await handling
