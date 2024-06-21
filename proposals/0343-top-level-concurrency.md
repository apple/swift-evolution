# Concurrency in Top-level Code

* Proposal: [SE-0343](0343-top-level-concurrency.md)
* Authors: [Evan Wilde](https://github.com/etcwilde)
* Review Manager: [Saleem Abdulrasool](https://github.com/compnerd)
* Status: **Implemented (Swift 5.7)**
* Implementation: [Fix top-level global-actor isolation crash](https://github.com/apple/swift/pull/40963), [Add `@MainActor @preconcurrency` to top-level variables](https://github.com/apple/swift/pull/40998), [Concurrent top-level inference](https://github.com/apple/swift/pull/41061)

## Introduction

Bringing concurrency to top-level code is an expected continuation of the
concurrency work in Swift. This pitch looks to iron out the details of how
concurrency will work in top-level code, specifically focusing on how top-level
variables are protected from data races, and how a top-level code context goes
from a synchronous context to an asynchronous context.

Swift-evolution thread: [Discussion thread topic for concurrency in top-level code](https://forums.swift.org/t/concurrency-in-top-level-code/55001)

## Motivation

The top-level code declaration context works differently than other declaration
spaces. As such, adding concurrency features to this spaces results in questions
that have not yet been addressed.

Variables in top-level code behave as a global-local hybrid variable; they exist
in the global scope and are accessible as global variables within the module,
but are initialized sequentially like local variables. Global variables are
dangerous, especially with concurrency. There are no isolation guarantees made,
and are therefore subject to race conditions.

As top-level code is intended as a safe space for testing out features and
writing pleasant little scripts, this simply will not do.

In addition to the strange and dangerous behavior of variables, changing whether
a context is synchronous or asynchronous has an impact on how function overloads
are resolved, so simply flipping a switch could result in some nasty hidden
semantic changes, potentially breaking scripts that already exist.

## Proposed solution

The solutions will only apply when the top-level code is an asynchronous
context. As a synchronous context, the behavior of top-level code does not
change. In order trigger making the top-level context an asynchronous context, I
propose using the presence of an `await` in one of the top-level expressions.

An await nested within a function declaration or a closure will not trigger the
behavior.

```swift
func doAsyncStuff() async {
  // ...
}

let countCall = 0

let myClosure = {
  await doAsyncStuff() // `await` does not trigger async top-level
  countCall += 1
}

await myClosure() // This `await` will trigger an async top-level
```

Top-level global variables are implicitly assigned a `@MainActor` global actor
isolation to prevent data races. To avoid breaking sources, the variable is
implicitly marked as pre-concurrency up to Swift 6.

```swift
var a = 10

func bar() {
  print(a)
}

bar()

await something() // make top-level code an asynchronous context
```

After Swift 6, full actor-isolation checking will take place. The usage of `a`
in `bar` will result in an error due to `bar` not being isolated to the
`MainActor`. In Swift 5, this will compile without errors.

## Detailed design

### Asynchronous top-level context inference

The rules for inferring whether the top-level context is an asynchronous context
are the same for anonymous closures, specified in [SE-0296 Async/Await](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0296-async-await.md#closures).

The top-level code is inferred to be an asynchronous context if it contains a
suspension point in the immediate top-level context.

```swift
func theAnswer() async -> Int { 42 }

async let a = theAnswer() // implicit await, top-level is async

await theAnswer() // explicit await, top-level is async

let numbers = AsyncStream(Int.self) { continuation in
  Task {
    for number in 0 .. < 10 {
      continuation.yield(number)
    }
    continuation.finish()
  }
}

for await number in numbers { // explicit await, top-level is asnyc
  print(number)
}
```

The above example demonstrates each kind of suspension point, triggering an
asynchronous top-level context. Specifically, `async let a = theAnswer()`
involves an implicit suspension, `await theAnswer()` involves an explicit
suspension, as does `for await number in numbers`. Any one of these is
sufficient to trigger the switch to an asynchronous top-level context.

Not that the inference of `async` in the top-level does not propagate to
function and closure bodies, because those contexts are separably asynchronous
or synchronous.

```swift
func theAnswer() async -> Int { 42 }

let closure1 = { @MainActor in print(42) }
let closure2 = { () async -> Int in await theAnswer() }
```

The top-level code in the above example is not an asynchronous context because
the top-level does not contain a suspension point, either explicit or implicit.

The mechanism for inferring whether a closure body is an asynchronous context
lives in the `FindInnerAsync` ASTWalker. With minimal effort, the
`FindInnerAsync` walker can be generalized to handle top-level code bodies,
maintaining the nice parallel inference behaviour between top-level code and
closure body asynchronous detection.

### Variables

Variables in top-level code are initialized sequentially like a local variable,
but are in the global scope and are otherwise treated as global variables. To
prevent data races, variables should implicitly be isolated to the main actor.
It would be a shame if every top-level variable access had to go through an
`await` though. Luckily, like the other entrypoints, top-level code runs on the
main thread, so we can make the top-level code space implicitly main-actor
isolated so the variables can be accessed and modified directly. This is still
source-breaking though; a synchronous global function written in the top-level
code will emit an error because the function is not isolated to the main actor
when the variable is. While the diagnostic is correct in stating that there is a
potential data-race, the source-breaking effect is also unfortunate. To
alleviate the source break, the variable is implicitly annotated with the
`@preconcurrency` attribute. The attribute only applies to Swift 5 code, and
once the language mode is updated to Swift 6, these data races will become hard
errors.

If `-warn-concurrency` is passed to the compiler and there is an `await` in
top-level code, the warnings are hard errors in Swift 5, as they would in any
other asynchronous context. If there is no `await` and the flag is passed,
variables are implicitly protected by the main actor and concurrency checking is
strictly enforced, even though the top-level is not an asynchronous context.
Since the top-level is not an asynchronous context, no run-loops are created
implicitly and the overload resolution behavior does not change.

In summary, top-level variable declarations behave as though they were declared
with `@MainActor @preconcurrency` in order to strike a nice balance between
data-race safety and reducing source breaks.

Going back to the global behaviour variables, there are some additional design
details that I should point out.

I would like to propose removing the ability to explicitly specify a global
actor on top-level variables. Top-level variables are treated like a hybrid of
global and local variables, which has some nasty consequences. The variables are
declared in the global scope, so they are assumed to be available anywhere. This
results in some nasty memory safety issues, like the following example:

```swift
print(a)
let a = 10
```

The example compiles and prints "0" when executed. The declaration `a` is
available at the `print` statement because it is a global variable, but it is
not yet initialized because initialization happens sequentially. Integer types
and other primitives are implicitly zero-initialized; however, classes are
referential types, initialized to zero, so this results in a segmentation fault
if the variable is a class type.

Eventually, we would like to plug this hole in the memory model. The design for
that is still in development, but will likely move toward making top-level
variables local variables of the implicit main function. I am proposing that we
disallow explicit global actors to facilitate that change and reduce the source
breakage caused by that change.

## Source compatibility

The `await` expression cannot appear in top-level code today since the top-level
is not an asynchronous context. As the features proposed herein are enabled by
the presence of an `await` expression in the top level, there are no scripts
today that will be affected by the changes proposed in this proposal.

## Effect on ABI stability

This proposal has no impact on ABI. Functions and variables have the same
signature as before.

## Acknowledgments

Thank you, Doug, for lots of discussion on how to break this down into something
that minimizes source breakage to a level where we can introduce this to Swift 5.
