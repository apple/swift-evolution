# Continuations for interfacing async tasks with synchronous code

* Proposal: [SE-0300](0300-continuation.md)
* Authors: [John McCall](https://github.com/rjmccall), [Joe Groff](https://github.com/jckarter), [Doug Gregor](https://github.com/DougGregor), [Konrad Malawski](https://github.com/ktoso)
* Review Manager: [Ben Cohen](https://github.com/airspeedswift)
* Status: **Implemented (Swift 5.5)**
* Previous Revisions: [1](https://github.com/swiftlang/swift-evolution/blob/5f79481244329ec2860951c0c49c101aef5069e7/proposals/0300-continuation.md), [2](https://github.com/swiftlang/swift-evolution/blob/61c788cdb9674c99fc8731b49056cebcb5497edd/proposals/0300-continuation.md)

## Introduction

Asynchronous Swift code needs to be able to work with existing synchronous
code that uses techniques such as completion callbacks and delegate methods to
respond to events. Asynchronous tasks can suspend themselves on
**continuations** which synchronous code can then capture and invoke to
resume the task in response to an event.

Swift-evolution thread:

- [Structured concurrency](https://forums.swift.org/t/concurrency-structured-concurrency/41622)
- [Continuations for interfacing async tasks with synchronous code](https://forums.swift.org/t/concurrency-continuations-for-interfacing-async-tasks-with-synchronous-code/43619)

## Motivation

Swift APIs often provide asynchronous code execution by way of a callback. This
may occur either because the code itself was written prior to the introduction
of async/await, or (more interestingly in the long term) because it ties in
with some other system that is primarily event-driven. In such cases, one may
want to provide an async interface to clients while using callbacks internally.
In these cases, the calling async task needs to be able to suspend itself,
while providing a mechanism for the event-driven synchronous system to resume
it in response to an event.

## Proposed solution

The library will provide APIs to get a **continuation** for the current
asynchronous task. Getting the task's continuation suspends the task, and
produces a value that synchronous code can then use a handle to resume the
task. Given a completion callback based API like:

```swift
func beginOperation(completion: (OperationResult) -> Void)
```

we can turn it into an `async` interface by suspending the task and using its
continuation to resume it when the callback is invoked, turning the argument
passed into the callback into the normal return value of the async function:

```swift
func operation() async -> OperationResult {
  // Suspend the current task, and pass its continuation into a closure
  // that executes immediately
  return await withUnsafeContinuation { continuation in
    // Invoke the synchronous callback-based API...
    beginOperation(completion: { result in
      // ...and resume the continuation when the callback is invoked
      continuation.resume(returning: result)
    }) 
  }
}
```

## Detailed design

### Raw unsafe continuations

The library provides two functions, `withUnsafeContinuation` and
`withUnsafeThrowingContinuation`, that allow one to call into a callback-based
API from inside async code. Each function takes an *operation* closure,
which is expected to call into the callback-based API. The closure
receives a continuation instance that must be resumed by the callback,
either to provide the result value or (in the throwing variant) the thrown
error that becomes the result of the `withUnsafeContinuation` call when the
async task resumes:


```swift
struct UnsafeContinuation<T, E: Error> {
  func resume(returning: T)
  func resume(throwing: E)
  func resume(with result: Result<T, E>)
}

extension UnsafeContinuation where T == Void {
  func resume() { resume(returning: ()) }
}

extension UnsafeContinuation where E == Error {
  // Allow covariant use of a `Result` with a stricter error type than
  // the continuation:
  func resume<ResultError: Error>(with result: Result<T, ResultError>)
}

func withUnsafeContinuation<T>(
    _ operation: (UnsafeContinuation<T, Never>) -> ()
) async -> T

func withUnsafeThrowingContinuation<T>(
    _ operation: (UnsafeContinuation<T, Error>) throws -> ()
) async throws -> T
```

`withUnsafe*Continuation` will run its `operation` argument immediately in the
task's current context, passing in a *continuation* value that can be
used to resume the task. The `operation` function must arrange for the
continuation to be resumed at some point in the future; after the `operation`
function returns, the task is suspended. The task must then be brought out
of the suspended state by invoking one of the continuation's `resume` methods.
Note that `resume` immediately returns control to the caller after transitioning
the task out of its suspended state; the task itself does not actually resume
execution until its executor reschedules it. The argument to
`resume(returning:)` becomes the return value of `withUnsafe*Continuation`
when the task resumes execution.
`resume(throwing:)` can be used instead to make the task resume by propagating
the given error. As a convenience, given a `Result`, `resume(with:)` can be used
to resume the task by returning normally or raising an error according to the
state of the `Result`. If the `operation` raises an uncaught error before
returning, this behaves as if the operation had invoked `resume(throwing:)` with
the error.

If the return type of `withUnsafe*Continuation` is `Void`, one must specify
a value of `()` when calling `resume(returning:)`. Doing so produces some
unsightly code, so `Unsafe*Continuation<Void>` has an extra member `resume()`
that makes the function call easier to read.

After invoking `withUnsafeContinuation`, exactly one `resume` method must be
called *exactly-once* on every execution path through the program.
`Unsafe*Continuation` is an unsafe interface, so it is undefined behavior if
a `resume` method is invoked on the same continuation more than once. The
task remains in the suspended state until it is resumed; if the continuation
is discarded and never resumed, then the task will be left suspended until
the process ends, leaking any resources it holds.
Wrappers can provide checking for these misuses of continuations, and the
library will provide one such wrapper, discussed below.

Using the `Unsafe*Continuation` API, one may for example wrap such
(purposefully convoluted for the sake of demonstrating the flexibility of
the continuation API) function:

```swift
func buyVegetables(
  shoppingList: [String],
  // a) if all veggies were in store, this is invoked *exactly-once*
  onGotAllVegetables: ([Vegetable]) -> (),

  // b) if not all veggies were in store, invoked one by one *one or more times*
  onGotVegetable: (Vegetable) -> (),
  // b) if at least one onGotVegetable was called *exactly-once*
  //    this is invoked once no more veggies will be emitted
  onNoMoreVegetables: () -> (),
  
  // c) if no veggies _at all_ were available, this is invoked *exactly once*
  onNoVegetablesInStore: (Error) -> ()
)
// returns 1 or more vegetables or throws an error
func buyVegetables(shoppingList: [String]) async throws -> [Vegetable] {
  try await withUnsafeThrowingContinuation { continuation in
    var veggies: [Vegetable] = []

    buyVegetables(
      shoppingList: shoppingList,
      onGotAllVegetables: { veggies in continuation.resume(returning: veggies) },
      onGotVegetable: { v in veggies.append(v) },
      onNoMoreVegetables: { continuation.resume(returning: veggies) },
      onNoVegetablesInStore: { error in continuation.resume(throwing: error) },
    )
  }
}

let veggies = try await buyVegetables(shoppingList: ["onion", "bell pepper"])
```

Thanks to weaving the right continuation resume calls into the complex
callbacks of the `buyVegetables` function, we were able to offer a much nicer
overload of this function, allowing async code to interact with this function in
a more natural straight-line way.

### Checked continuations

`Unsafe*Continuation` provides a lightweight mechanism for interfacing
sync and async code, but it is easy to misuse, and misuse can corrupt the 
process state in dangerous ways. In order to provide additional safety and
guidance when developing interfaces between sync and async code, the
library will also provide a wrapper which checks for invalid use of the
continuation:

```swift
struct CheckedContinuation<T, E: Error> {
  func resume(returning: T)
  func resume(throwing: E)
  func resume(with result: Result<T, E>)
}

extension CheckedContinuation where T == Void {
  func resume()
}

extension CheckedContinuation where E == Error {
  // Allow covariant use of a `Result` with a stricter error type than
  // the continuation:
  func resume<ResultError: Error>(with result: Result<T, ResultError>)
}

func withCheckedContinuation<T>(
    _ operation: (CheckedContinuation<T, Never>) -> ()
) async -> T

func withCheckedThrowingContinuation<T>(
  _ operation: (CheckedContinuation<T, Error>) throws -> ()
) async throws -> T
```

The API is intentionally identical to the `Unsafe` variants, so that code
can switch easily between the checked and unchecked variants. For instance,
the `buyVegetables` example above can opt into checking merely by turning
its call of `withUnsafeThrowingContinuation` into one of `withCheckedThrowingContinuation`:

```swift
// returns 1 or more vegetables or throws an error
func buyVegetables(shoppingList: [String]) async throws -> [Vegetable] {
  try await withCheckedThrowingContinuation { continuation in
    var veggies: [Vegetable] = []

    buyVegetables(
      shoppingList: shoppingList,
      onGotAllVegetables: { veggies in continuation.resume(returning: veggies) },
      onGotVegetable: { v in veggies.append(v) },
      onNoMoreVegetables: { continuation.resume(returning: veggies) },
      onNoVegetablesInStore: { error in continuation.resume(throwing: error) },
    )
  }
}
```

Instead of leading to undefined behavior, `CheckedContinuation` will instead
trap if the program attempts to resume the continuation multiple times.
`CheckedContinuation` will also log a warning if the continuation
is discarded without ever resuming the task, which leaves the task stuck in its
suspended state, leaking any resources it holds. These checks happen regardless
of the optimization level of the program.

## Additional examples

Continuations can be used to interface with more complex event-driven
interfaces than callbacks as well. As long as the entirety of the process
follows the requirement that the continuation be resumed exactly once, there
are no other restrictions on where the continuation can be resumed. For
instance, an `Operation` implementation can trigger resumption of a
continuation when the operation completes:

```swift
class MyOperation: Operation {
  let continuation: UnsafeContinuation<OperationResult, Never>
  var result: OperationResult?

  init(continuation: UnsafeContinuation<OperationResult, Never>) {
    self.continuation = continuation
  }

  /* rest of operation populates `result`... */

  override func finish() {
    continuation.resume(returning: result!)
  }
}

func doOperation() async -> OperationResult {
  return await withUnsafeContinuation { continuation in
    MyOperation(continuation: continuation).start()
  }
}
```

Using APIs from the [structured concurrency proposal](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md),
one can wrap up a `URLSession` in a task, allowing the task's cancellation
to control cancellation of the session, and using a continuation to respond
to data and error events fired by the network activity:

```swift
func download(url: URL) async throws -> Data? {
  var urlSessionTask: URLSessionTask?

  return try Task.withCancellationHandler {
    urlSessionTask?.cancel()
  } operation: {
    let result: Data? = try await withUnsafeThrowingContinuation { continuation in
      urlSessionTask = URLSession.shared.dataTask(with: url) { data, _, error in
        if case (let cancelled as NSURLErrorCancelled)? = error {
          continuation.resume(returning: nil)
        } else if let error = error {
          continuation.resume(throwing: error)
        } else {
          continuation.resume(returning: data)
        }
      }
      urlSessionTask?.resume()
    }
    if let result = result {
      return result
    } else {
      Task.cancel()
      return nil
    }
  }
}
```

It is also possible for wrappers around callback based APIs to respect their parent/current tasks's cancellation, as follows:

```swift
func fetch(items: Int) async throws -> [Items] {
  let worker = ... 
  return try Task.withCancellationHandler(
    handler: { worker?.cancel() }
  ) { 
    return try await withUnsafeThrowingContinuation { c in 
      worker.work(
        onNext: { value in c.resume(returning: value) },
        onCancelled: { value in c.resume(throwing: CancellationError()) },
      )
    } 
  }
}
```

If tasks were allowed to have instances, which is under discussion in the structured concurrency proposal, it would also be possible to obtain the task in which the fetch(items:) function was invoked and call isCanceled on it whenever the insides of the withUnsafeThrowingContinuation would deem it worthwhile to do so.

## Alternatives considered

### Name `CheckedContinuation` just `Continuation`

We could position `CheckedContinuation` as the "default" API for doing
sync/async interfacing by leaving the `Checked` word out of the name. This
would certainly be in line with the general philosophy of Swift that safe
interfaces are preferred, and unsafe ones used selectively where performance
is an overriding concern. There are a couple of reasons to hesitate at doing
this here, though:

- Although the consequences of misusing `CheckedContinuation` are not as
  severe as `UnsafeContinuation`, it still only does a best effort at checking
  for some common misuse patterns, and it does not render the consequences of
  continuation misuse entirely moot: dropping a continuation without resuming
  it will still leak the un-resumed task, and attempting to resume a
  continuation multiple times will still cause the information passed through
  the continuation to be lost. It is still a serious programming error if
  a `with*Continuation` operation misuses the continuation;
  `CheckedContinuation` only helps make the error more apparent.
- Naming a type `Continuation` now might take the "good" name away if,
  after we have move-only types at some point in the future, we want to
  introduce a continuation type that statically enforces the exactly-once
  property.

### Don't expose `UnsafeContinuation`

One could similarly make an argument that `UnsafeContinuation` shouldn't be
exposed at all, since the `Checked` form can always be used instead. We think
that being able to avoid the cost of checking when interacting with
performance-sensitive APIs is valuable, once users have validated that their
interfaces to those APIs are correct.

### Have `CheckedContinuation` trap on all misuses, or log all misuses

`CheckedContinuation` is proposed to trap when the program attempts to
resume the same continuation twice, but only log a warning if a continuation
is abandoned without getting resumed. We think this is the right tradeoff
for these different situations for the following reasons:

- With `UnsafeContinuation`, resuming multiple times corrupts the process and
  leaves it in an undefined state. By trapping when the task is resumed
  multiple times, `CheckedContinuation` turns undefined behavior into a well-
  defined trap situation.  This is analogous to other checked/unchecked
  pairings in the standard library, such as `!` vs. `unsafelyUnwrapped` for
  `Optional`.
- By contrast, failing to resume a continuation with `UnsafeContinuation`
  does not corrupt the task, beyond leaking the suspended task's resources;
  the rest of the program can continue executing normally. Furthermore,
  the only way we can currently detect and report such a leak is by using
  a class `deinit` in its implementation. The precise moment at which such
  a deinit would execute is not entirely predictable because of refcounting
  variability from ARC optimization. If `deinit` were made to trap, whether that
  trap is executed and when could vary with optimization level, which we
  don't think would lead to a good experience.

### Expose more `Task` API on `*Continuation`, or allow a `Handle` to be recovered from a continuation

The full `Task` and `Handle` API provides additional control over the task
state to holders of the handle, particularly the ability to query and set
cancellation state, as well as await the final result of the task, and one
might wonder why the `*Continuation` types do not also expose this functionality.
The role of a `Continuation` is very different from a `Handle`, in that a handle
represents and controls the entire lifetime of the task, whereas a continuation
only represents a *single suspension point* in the lifetime of the task.
Furthermore, the `*Continuation` API is primarily designed to allow for
interfacing with code outside of Swift's structured concurrency model, and
we believe that interactions between tasks are best handled inside that model
as much as possible.

Note that `*Continuation` also does not strictly need direct support for any
task API on itself. If, for instance, someone wants a task to cancel itself
in response to a callback, they can achieve that by funneling a sentinel
through the continuation's resume type, such as an Optional's `nil`:

```swift
let callbackResult: Result? = await withUnsafeContinuation { c in
  someCallbackBasedAPI(
    completion: { c.resume($0) },
    cancellation: { c.resume(nil) })
}

if let result = callbackResult {
  process(result)
} else {
  cancel()
}
```

### Provide API to resume the task immediately to avoid "queue-hopping"

Some APIs, in addition to taking a completion handler or delegate, also allow
the client to control *where* that completion handler or delegate's methods are
invoked; for instance, some APIs on Apple platforms take an argument for the
dispatch queue the completion handler should be invoked by. In these cases,
it would be optimal if the original API could resume the task directly on the
dispatch queue (or whatever other scheduling mechanism, such as a thread or 
run loop) that the task would normally be resumed on by its executor. To
enable this, we could provide a variant of `with*Continuation` that, in
addition to providing a continuation, also provides the dispatch queue that
the task expects to be resumed on. The `*Continuation` type in turn could
provide an `unsafeResumeImmediately` set of APIs, which would immediately
resume execution of the task on the current thread. This would enable something
like this:

```swift
// Given an API that takes a queue and completion handler:
func doThingAsynchronously(queue: DispatchQueue, completion: (ResultType) -> Void)

// We could wrap it in a Swift async function like:
func doThing() async -> ResultType {
  await withUnsafeContinuationAndCurrentDispatchQueue { c, queue in
    // Schedule to resume on the right queue, if we know it
    doThingAsynchronously(queue: queue) {
      c.unsafeResumeImmediately(returning: $0)
    }
  }
}
```

However, such an API would have to be used very carefully; the programmer
would have to be careful that `unsafeResumeImmediately` is in fact invoked
in the correct context, and that it is safe to take over control of the
current thread from the caller for a potentially unbounded amount of time.
If the task is resumed in the wrong context, it will break assumptions in the
written code as well as those made by the compiler and runtime, which will
lead to subtle bugs that would be difficult to diagnose. We can investigate
this as an addition to the core proposal, if "queue hopping" in continuation-
based adapters turns out to be a performance problem in practice.

## Revision history

Third revision:

- Replaced separate `*Continuation<T>` and `*ThrowingContinuation<T>` types with a
  single `Continuation<T, E: Error>` type parameterized on the error type.
- Added a convenience `resume()` equivalent to `resume(returning: ())` for
  continuations with a `Void` return type.
- Changed `with*ThrowingContinuation` to take an `operation` block that may
  throw, and to immediately resume the task throwing the error if an uncaught
  error propagates from the operation.

Second revision:

- Clarified the execution behavior of `with*Continuation` and
  `*Continuation.resume`, namely that `with*Continuation` immediately executes
  its operation argument in the current context before suspending the task,
  and that `resume` immediately returns to its caller after un-suspending the
  task, leaving the task to be scheduled by its executor.
- Removed an unnecessary invariant on when `resume` must be invoked; it is valid
  to invoke it exactly once at any point after the `with*Continuation` operation
  has started executing; it does not need to run exactly when the operation
  returns.
- Added "future direction" discussion of a potential more advanced API that
  could allow continuations to directly resume their task when the correct
  dispatch queue to do so is known.
- Added `resume()` on `Void`-returning `Continuation` types.
