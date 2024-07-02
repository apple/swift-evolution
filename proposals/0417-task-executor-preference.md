# Task Executor Preference

* Proposal: [SE-0417](0417-task-executor-preference.md)
* Author: [Konrad 'ktoso' Malawski](https://github.com/ktoso), [John McCall](https://github.com/rjmccall), [Franz Busch](https://github.com/FranzBusch)
* Review Manager: [Doug Gregor](https://github.com/DougGregor)
* Status:  **Implemented (Swift 6.0)**
* Review: ([pitch](https://forums.swift.org/t/pitch-task-executor-preference/68191)), ([review](https://forums.swift.org/t/se-0417-task-executor-preference/68958)), ([acceptance](https://forums.swift.org/t/accepted-se-0417-task-executor-preference/69705))

## Introduction

Swift Concurrency uses tasks and actors to model concurrency and primarily relies on actor isolation to determine where a specific piece of code shall execute.

The recent introduction of custom actor executors in [SE-0392](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0392-custom-actor-executors.md) allows specifying a `SerialExecutor` implementation code should be running on while isolated to a specific actor. This allows developers to gain some control over exact threading semantics of actors, by e.g. making sure all work made by a specific actor is made on a dedicated queue or thread.

Today, the same flexibility is not available to tasks in general, and nonisolated asynchronous functions are always executed on the default global concurrent thread pool managed by Swift concurrency.

## Motivation

Custom actor executors allow developers to customize where execution of a task “on” an actor must happen (e.g. on a specific queue or thread, represented by a `SerialExecutor`), the same capability is currently missing for code that is not isolated to an actor.

Notably, since Swift 5.7’s [SE-0338: Clarify the Execution of Non-Actor-Isolated Async Functions](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0338-clarify-execution-non-actor-async.md), functions which are not isolated to an actor will always hop to the default global concurrent executor, which is great for correctness and understanding the code and avoids “hanging onto” actors  longer than necessary. This is also a desirable semantic for code running on the `MainActor` calling into `nonisolated` functions, since it allows the main actor to be quickly freed up to proceed with other work, however it has a detrimental effect on applications which want to *avoid* hops in order to maximize request processing throughput. This is especially common with event-loop based systems, such as network servers or other kinds of tight request handling loops.

As Swift concurrency is getting adopted in a wider variety of performance sensitive codebases, it has become clear that the lack of control over where nonisolated functions execute is a noticeable problem. 
At the same time, the defensive "hop-off" semantics introduced by [SE-0338](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0338-clarify-execution-non-actor-async.md) are still valuable, but sometimes too restrictive and some use-cases might even say that the exact opposite behavior might be desirable instead.

This proposal acknowledges the different needs of various use-cases, and provides a new flexible mechanism for developers to tune their applications and avoid potentially unnecessary context switching when possible.

## Proposed solution

We propose to introduce an additional layer of control over where a task can be executed, and have this executor setting be “sticky” to the task.

**Currently** the decision where an async function or closure is going to execute is binary:

```
// `func` execution semantics before this proposal

[ func / closure ] - /* where should it execute? */
                                 |
                           +--------------+          +==========================+
                   +- no - | is isolated? | - yes -> | default (actor) executor |
                   |       +--------------+          +==========================+
                   |
                   |                                 +==========================+
                   +-------------------------------> | on global conc. executor |
                                                     +==========================+
```

This proposal introduces a way to control hopping off to the global concurrent pool for `nonisolated` functions and closures. This is expressed as **task executor preference** and is sticky to the task and entire *structured task hierarchy* created from a task with a specified preference. This changes the current decision diagram to the following:

```
// `func` execution semantics with this proposal

[ func / closure ] - /* where should it execute? */
                               |
                     +--------------+          +===========================+
           +-------- | is isolated? | - yes -> | actor has unownedExecutor |
           |         +--------------+          +===========================+
           |                                       |                |      
           |                                      yes               no
           |                                       |                |
           |                                       v                v
           |                  +=======================+    /* task executor preference? */
           |                  | on specified executor |        |                   |
           |                  +=======================+       yes                  no
           |                                                   |                   |
           |                                                   |                   v
           |                                                   |    +==========================+
           |                                                   |    | default (actor) executor |
           |                                                   v    +==========================+
           v                                   +==============================+
/* task executor preference? */ ---- yes ----> | on Task's preferred executor |
           |                                   +==============================+
           no
           |
           v
  +===============================+
  | on global concurrent executor |
  +===============================+
```

In other words, this proposal introduces the ability to specify where code may execute from a Task, and not just by using a custom actor executor,
and even influence the thread use of default actors.

With this proposal a **`nonisolated` function** will execute, as follows:

* if task preference **is not** set:
  * it is equivalent to current semantics, and will execute on the global concurrent executor,

* if a task preference **is** set,
  * **(new)** nonisolated functions will execute on the selected executor.


The preferred executor also may influence where **actor-isolated code** may execute, specifically:

- if task preference **is** set:
  - **(new)** default actors will use the task's preferred executor
  - actors with a custom executor execute on that specified executor (i.e. "preference" has no effect), and are not influenced by the task's preference

The task executor preference can be specified either at task creation time:

```swift
Task(executorPreference: executor) {
  // starts and runs on the 'executor'
  await nonisolatedAsyncFunc()
}

Task.detached(executorPreference: executor) {
  // starts and runs on the 'executor'
  await nonisolatedAsyncFunc()
}

await withDiscardingTaskGroup { group in 
  group.addTask(executorPreference: executor) { 
    // starts and runs on the 'executor'
    await nonisolatedAsyncFunc()
  }
}

func nonisolatedAsyncFunc() async -> Int {
  // if the Task has a specific executor preference,
  // runs on that 'executor' rather than on the default global concurrent executor
  return 42
} 
```

or, for a specific scope using the `withTaskExecutorPreference` method. Notably, the task executor preference is in effect for the entire structured task hierarchy while running in a task or scope where a task executor preference is set. For example, the following snippet illustrates child tasks created inside of a `withTaskExecutorPreference`:

```swift
await withTaskExecutorPreference(executor) { 
  // if not already running on specified 'executor'
  // the withTaskExecutorPreference would hop to it, and run this closure on it.

  // task groups
  await withDiscardingTaskGroup { group in 
    group.addTask {
      // starts and runs on the 'executor'
      await nonisolatedAsyncFunc() // also runs on 'executor'
    }
  }
  
  // async let 
  async let number = nonisolatedAsyncFunc() // starts and runs on 'executor'
  await number
}
```

If a task with such executor preference encounters code which is `isolated` to some specific actor, the isolation properties of the actor still are upheld, however, unless that actor has a custom executor configured, the source of the thread actually running the actor's functions will be from the preferred executor:

```swift
let capy: Capybara = Capybara()
actor Capybara { func eat() {} } 

Task(executorPreference: executor) {
  // starts on 'executor'
  try await capy.eat() // execution is isolated to the 'capy' actor, however execution happens on the 'executor' TaskExecutor
}
```

In a way, one should think of the `SerialExecutor` of the actor and `TaskExecutor` both being tracked and providing different semantics. 
The `SerialExecutor` guarantees mutual exclusion, and the `TaskExecutor` provides a source of threads.

## Detailed design

### Setting task executor preference

A new concept of task executor preference is added to Swift Concurrency tasks. This preference is stored in a task and propagated throughout child tasks (such as ones created by TaskGroups and async let).

The preference can be set using various APIs that will be discussed in detail in their respective sections. The first of those APIs is `withTaskExecutorPreference` which can be called inside an asynchronous context to both ensure we’re executing on the expected executor, as well as set the task executor preference for the duration of the operation closure:

```swift
await withTaskExecutorPreference(someExecutor) { 
 // guaranteed to be executing on someExecutor
}
```

Once set, the effect of an executor preference is such that a nonisolated func instead of immediately hopping to the global pool, it may hop to the preferred executor, e.g.:

```swift
nonisolated func doSomething() async { 
 // ...
}

let preferredExecutor: SomeConcreteTaskExecutor = ...
Task(executorPreference: preferredExecutor) {
  // executes on 'preferredExecutor'
  await doSomething() // doSomething body would execute on 'preferredExecutor'
}

 await doSomething() // doSomething body would execute on 'default global concurrent executor'
```

### The `TaskExecutor` protocol

In order to fulfil the requirement that we'd like default actors to run on a task executor, if it was set, we need to introduce a new kind of executor.

This stems from the fact that `SerialExecutor` and how a default actor effectively acts as an executor "for itself" in Swift.
A default actor (so an actor which does not use a custom executor), has a "default" serial executor that is created by the Swift runtime and uses the actor is the executor's identity.
This means that the runtime executor tracking necessarily needs to track that some code is executing on a specific serial executor in order for things like `assumeIsolated` or the built-in runtime thread-safety checks can utilize them.

The new protocol mirrors `Executor` and `SerialExecutor` in API, however it provides different semantics, and is tracked using a different mechanism at runtime -- by obtaining it from a task's executor preference record.

The `TaskExecutor` is defined as:

```swift
public protocol TaskExecutor: Executor {
  func enqueue(_ job: consuming ExecutorJob)

  func asUnownedTaskExecutor() -> UnownedTaskExecutor
}
```

As an intuitive way to think about `TaskExecutor` and `SerialExecutor`, one can think of the prior as being a "source of threads" to execute work on,
and the latter being something that "provides serial isolation" and is a crucial part of Swift actors. The two share similarities, however the task executor has a more varied application space.

### Task executor preference inheritance in Structured Concurrency

Task executor preference is inherited by child tasks and actors which do not declare an explicit executor (so-called "default actors"), and is *not* inherited by un-structured tasks. 

Specifically:

* **Do** inherit task executor preference
    * TaskGroup’s `addTask()`, unless overridden with explicit parameter
    * `async let`
    * methods on default actors (actors which do not use a custom executor)
* **Do not** inherit task executor preference
    * Unstructured tasks: `Task {}` and `Task.detached {}`
    * methods on actors which **do** use a custom executor (including e.g. the `MainActor`)

This also means that an entire tree can be made to execute their nonisolated work on a specific executor, just by means of setting the preference on the top-level task.

### Task executor preference and async let

Since `async let` are the simplest form of structured concurrency, they do not offer much in the way of customization.

An async let currently always executes on the global concurrent executor, and with the inclusion of this proposal, it does take into account task executor preference. In other words, if an executor preference is set, it will be used by async let to enqueue its underlying task:

```swift
func test() async -> Int {
  return 42
}

await withTaskExecutorPreference(someExecutor) { 
  async let value = test() // async let's "body" and target function execute on 'someExecutor'
  // ... 
  await value
}
```

### Task executor preference and TaskGroups

A `TaskGroup` and its various friends (`ThrowingTaskGroup`, `DiscardingTaskGroup`, ...) are the most powerful, but also most explicit and verbose API for structured concurrency. A group allows creating multiple child tasks using the `addTask` method, and always awaits all child tasks to complete before returning.

This proposal adds overloads to the `addTask` method, which changes the executor the child tasks will be enqueued on:

```swift
extension (Discarding)(Throwing)TaskGroup { 
  mutating func addTask(
    on executor: (any TaskExecutor)?,
    priority:  TaskPriority? = nil,
    operation:  @Sendable @escaping () async (throws) -> Void
  )
}
```

Which allows users to require child tasks be enqueued and run on specific executors:

```swift
Task(executorPreference: specialExecutor) {
  _ = await withTaskGroup(of: Int.self) { group in 
    group.addTask {
      // using 'specialExecutor' (inherited preference)
      return 12 
    }
    group.addTask(executorPreference: differentExecutor) {
      // using 'differentExecutor', overriden preference
      return 42 
    }
    group.addTask(executorPreference: nil) {
      // using 'specialExecutor' (inherited preference)
      //
      // explicitly documents that this task has "no task executor preference".
      // this is semantically equivalent to the addTask() call without specifying
      // an executor, and therefore since the surrounding scope has a specialExecutor preference,
      // that's the executor used.
      return 84
    } 
    group.addTask(executorPreference: globalConcurrentExecutor) {
      // using 'globalConcurrentExecutor', overridden preference
      // 
      // using the global concurrent executor -- effectively overriding
      // the task executor preference set by the outer scope back to the 
      // default semantics of child tasks -- to execute on the global concurrent executor.
      return 84
    } 
    return await group.next()!
  }
```

This gives developers explicit control over where a task group child task shall be executed. Notably, this gives callers of libraries more control over where work should be performed. Do note that code executing on an actor will always hop to that actor; and task executor preference has no impact on code which *requires* to be running in some specific isolation.

If a library really wants to ensure that hops to the global concurrent executor *are* made by child tasks it may use the newly introduced `globalConcurrentExecutor` global variable.

### Task executor preference and Unstructured Tasks

We propose adding new APIs and necessary runtime changes to allow a Task to be enqueued directly on a specific `Executor`, by using a new `Task(executorPreference:)` initializer:

```swift
extension Task where Failure == Never {
  @discardableResult
  public init(
    executorPreference taskExecutor: (any TaskExecutor)?,
    priority: TaskPriority? = nil,
    operation: @Sendable @escaping () async -> Success
  )
  
  @discardableResult
  static func detached(
    executorPreference taskExecutor: (any TaskExecutor)?,
    priority: TaskPriority? = nil,
    operation: @Sendable @escaping () async -> Success
  )
}

extension Task where Failure == Error { 
  @discardableResult
  public init(
    executorPreference taskExecutor: (any TaskExecutor)?,
    priority: TaskPriority? = nil,
    operation: @Sendable @escaping () async throws -> Success
  )
  
  @discardableResult
  static func detached(
    executorPreference taskExecutor: (any TaskExecutor)?,
    priority: TaskPriority? = nil,
    operation: @Sendable @escaping () async throws -> Success
  )
}
```

Tasks created this way are **immediately enqueued** on given executor.

It is possible to pass `nil` to all task executor accepting APIs introduced in this proposal. Passing `nil` to an `executorPreference:` parameter means "no preference", and for structured tasks means to inherit the surrounding context's executor preference; and for unstructured tasks (`Task.init`, `Task.detached`) it serves as a way of documenting no specific executor preference was selected for this task. In both cases, passing `nil` is equivalent to calling the methods which do not accept an executor preference.

By default, serial executors are not task executors, and therefore cannot be directly used with these APIs. 
This is because it would cause confusion in the runtime about having two "mutual exclusion" contexts at the same time, which could result in difficult to understand behaviors.

It is possible however to write a custom `SerialExecutor` and conform to the `TaskExecutor` protocol at the same time, if indeed one intended to use it for both purposes.
The serial executor conformance can be used for purposes of isolation (including the asserting and "assuming" of isolation), and the task executor conformance allows
using a type to provide a hint where tasks should execute although cannot be used to fulfil isolation requirements.

#### Task executor preference and default actor isolated methods

It is also worth explaining the interaction with actors which do not use a custom executor -- which is the majority of actors usually defined in a typical codebase.
Such actors are referred to as "default actors" and are the default way of how actors are declared:

```swift
actor RunsAnywhere { // a "default" actor == without an executor requirement
  func hello() {
    return "Hello"
  }
}
```

Such actor has no requirement as to where it wants to execute. This means that if we were to call the `hello()` isolated 
actor method from a task that has defined an executor preference -- the hello() method would still execute on a thread owned by that executor (!),
however isolation is still guaranteed by the actor's semantics:

```swift
let anywhere = RunsAnywhere()
Task { await anywhere.hello() } // runs on "default executor", using a thread from the global pool

Task(executorPreference: myExecutor) {
  // runs on preferred executor, using a thread owned by that executor
  await anywhere.hello()
}
```

Methods which assert isolation, such as `Actor/assumeIsolated` and similar still function as expected.

The task executor can be seen as a "source of threads" for the execution, while the actor's serial executor is used to ensure the serial and isolated execution of the code.

## Inspecting task executor preference

It is possible to inspect the current preferred task executor of a task, however because doing so is inherently unsafe -- due to lack of guarantees surrounding the lifetime of an executor referred to using an `UnownedTaskExecutor`,
this operation is only exposed on the `UnsafeCurrentTask`. 

Furthermore, the API purposefully does not expose an `any TaskExecutor` because this would risk incurring atomic ref-counting on an executor object that may have been already deallocated.

This API is intended only for fine-tuning and checking if we are executing on the "expected" task executor, and therefore the `UnownedTaskExecutor` also implements the `Equatable` protocol,
and implements it using pointer equality. This comparison is not strictly safe, in case if an executor was deallocated, and a new executor was allocated in the same memory location,
however for purposes of executors -- especially long-lived ones, we believe this is not going to prove to be a problem in practical uses of task executors.

An example use of this API might be something like this:

``` swift
struct MyEventLoopTaskExecutor: TaskExecutor {}

func test(expected eventLoop: MyEventLoopTaskExecutor) {
  withUnsafeCurrentTask { task in
    guard let task else {
      fatalError("Missing task?")
    }
    guard let currentTaskExecutor = task.unownedTaskExecutor else {
      fatalError("Expected to have task executor")
    }
    
    precondition(currentTaskExecutor == eventLoop.asUnownedTaskExecutor())
    
    // perform action that is required to run on the expected executor
  }
}
```

This may be useful in synchronous functions; however should be used sparingly, and with caution.
Asynchronous functions, or functions on actors should instead rely on the usual ways to statically ensure to be running on an expected executor:
by providing the right annotations or custom executors to their enclosing actors.

Instead, functions which have strict execution requirements may be better served as declaring them inside of an actor
that has the required specific executor specified (by using custom actor executors), or by using an asynchronous function
and wrapping the code that is required to run on a specific executor in an `withTaskExecutorPreference(eventLoop) { ... }` block.

Nevertheless, because we understand there may be situations where synchronous code may want to compare task executors, this capability is exposed for advanced use cases.

Another use case may be carrying the same task executor into an un-structured Task -- although this should only be done with **extreme caution**,
because it breaks structured concurrency lifetime expectations of executors. For example, the following code is correct under structured concurrency's
default and automatic behavior surrounding task executors:

```swift
func computeThings() async {
  let eventLoop = MyCoolEventLoop()
  defer { eventLoop.shutdown() }

  let computed = await withTaskExecutorPreference(eventLoop) {
    async let first = computation(1)
    async let second = computation(2)
    return await first + second
  }

  return computed // event loop will be shutdown and the executor destroyed(!)
}

func computation(_ int: Int) -> Int { return int * 2 }
```

The above code is structurally correct and we guarantee the lifetime of `MyCoolEventLoop` throughout all of its uses 
by structured concurrency tasks in this snippet.

The following snippet is **not safe**, which is why task executors are not inherited to un-structured tasks:

```swift 
// !!! POTENTIALLY UNSAFE !!! 
// Do not do this, unless you can guarantee the lifetime of TaskExecutor 
// exceeds all potential for any task to be running on it (!)

func computeThings() async {
  let eventLoop: any TaskExecutor = MyCoolEventLoop()
  defer { eventLoop.shutdown() }

  let computed = withTaskExecutorPreference(eventLoop) {
    async let first = computation(1)
    async let second = computation(2)
    return await first + second
  }

  return computed // event loop will be shutdown and the executor destroyed(!)
}

// DANGEROUS; MUST ENSURE THE EXECUTOR REMAINS ALIVE FOR AS LONG AS ANY TASK MAY BE RUNNING ON IT
func computation(_ int: Int) -> Int {
  withUnsafeCurrentTask { task in
    let unownedExecutor: UnownedTaskExecutor? = task?.unownedTaskExecutor
    let eventLoop: MyCoolEventLoop? = EventLoops.find(unownedExecutor)
    
    // Dangerous because there is no structured guarantee that eventLoop will be kept alive
    // for as long as there are any of its child tasks and functions running on it
    Task(executorPreference: eventLoop) { ... }
  }
}
```

## Combining `SerialExecutor` and `TaskExecutor`

It is possible to declare a single executor type and have it conform to *both* the `SerialExecutor` (introduced in the custom actor executors proposal),
as well as the `TaskExecutor` (introduce in this proposal).

If declaring an executor which conforms to both protocols, it truly **must** adhere to the `SerialExecutor` 
semantics of not running work concurrently, as it may be used as an *isolation context* by an actor. 

```swift
// naive executor for illustration purposes; we'll assert on the dispatch queue and isolation.
final class NaiveQueueExecutor: TaskExecutor, SerialExecutor {
  let queue: DispatchQueue

  init(_ queue: DispatchQueue) {
    self.queue = queue
  }

  public func enqueue(_ _job: consuming ExecutorJob) {
  let job = UnownedJob(_job)
  queue.async {
    job.runSynchronously(
        isolatedOn: self.asUnownedSerialExecutor(),
        taskExecutor: self.asUnownedTaskExecutor())
    }
  }

  @inlinable
  public func asUnownedSerialExecutor() -> UnownedSerialExecutor {
    UnownedSerialExecutor(ordinary: self)
  }

  @inlinable
  public func asUnownedTaskExecutor() -> UnownedTaskExecutor {
    UnownedTaskExecutor(ordinary: self)
  }
}
```

Since the enqueue method shares the same signature between the two protocols it is possible to just implement it once.
It is of crucial importance to run the job using the new `runSynchronously(isolatedOn:taskExecutor:)` overload
of the `runSynchronously` method. This will set up all the required thread-local state for both isolation assertions
and task-executor preference semantics to be handled properly.

Given such an executor, we are able to have it both be used by an actor (thanks to being a `SerialExecutor`), and have 
any structured tasks or nonisolated async functions execute on it (thanks to it being a `TaskExecutor`):

```swift
nonisolated func nonisolatedFunc(expectedExecutor: NaiveQueueExecutor) async {
  dispatchPrecondition(condition: .onQueue(expectedExecutor.queue))
  expectedExecutor.assertIsolated()
}

actor Worker {
  let executor: NaiveQueueExecutor

  init(on executor: NaiveQueueExecutor) {
    self.executor = executor
  }

  func test(_ expectedExecutor: NaiveQueueExecutor) async {
    // we are isolated to the serial-executor (!)
    dispatchPrecondition(condition: .onQueue(expectedExecutor.queue))
    expectedExecutor.preconditionIsolated()

    // the nonisolated async func properly executes on the task-executor
    await nonisolatedFunc(expectedExecutor: expectedExecutor)

    /// the task-executor preference is inherited properly:
    async let val = {
      dispatchPrecondition(condition: .onQueue(expectedExecutor.queue))
      expectedExecutor.preconditionIsolated()
      return 12
    }()
    _ = await val

    // as expected not-inheriting
    _ = await Task.detached {
      dispatchPrecondition(condition: .notOnQueue(expectedExecutor.queue))
    }.value

    // we properly came back to the serial executor, just to make sure
    dispatchPrecondition(condition: .onQueue(expectedExecutor.queue))
    expectedExecutor.preconditionIsolated()
  }
}
```

### The `globalConcurrentExecutor`

This proposal also introduces a way to obtain a reference to the global concurrent executor which is used by default by all tasks and asynchronous functions unless they require some specific executor.

The implementation of this executor is not exposed as a type, however it is accessible through the `globalConcurrentExecutor` global variable:

```swift
nonisolated(unsafe)
public var globalConcurrentExecutor: any _TaskExecutor { get }
```

Accessing this global computed property is thread-safe and can be done without additional synchronization.

At present, it is not possible to customize the returned executor from this property, however customizing it is something we are interested in exploring in the future (as well as the main actor's executor).

This executor does not introduce new functionality to Swift Concurrency per se, as it was always there since the beginning of the concurrency runtime, however it is the first time it is possible to obtain a reference to the global concurrent executor in pure Swift. Generally just creating tasks and calling nonisolated asynchronous functions would automatically enqueue them onto this underlying global thread-pool.

This proposal introduces the `globalConcurrentExecutor` variable in order to be able to effectively "disable" a task executor preference, because setting a task's executor preference to the default executor is equivalent to the task having the default behavior, as if no executor preference was set. This matters particularly which child tasks, which do want to execute on the default executor under any circumstances: 

```swift
async let noPreference = computation() // child task executes on the global concurrent executor

await withTaskExecutorPreference(specific) {
  async let compute = computation() // child task executes on 'specific' executor
  
  await withTaskGroup(of: Int.self) { group in
    //  child task executes on 'specific' executor
    group.addTask { computation() }
    
    // child task executes on global concurrent executor 
    group.addTask(executorPreference: globalConcurrentExecutor) {
      async let compute = computation() // child task executes on the global concurrent executor
      
      computation() // executed on the global concurrent executor
    } 
  }
}
```

## Execution semantics discussion

### Not a Golden Hammer

As with many new capabilities in libraries and languages, one may be tempted to use task executors to solve various problems.

We advise care when doing so with task executors, because while they do minimize the "hopping off" of executors and the associated context switching,
this is also a behavior that may be entirely _undesirable_ in some situations. For example, over-hanging on the MainActor's executor is one of the main reasons
earlier Swift versions moved to make `nonisolated` asynchronous functions always hop off their calling execution context; and this proposal brings back this behavior 
for specific executors. 

Applying task executors to solve a performance problem should be done after thoroughly understanding the problem an application is facing,
and only then determining the right "sticky"-ness behavior and specific pieces of code which might benefit from it.

Examples of good candidates for task executor usage would be systems which utilize some form of "specific thread" to minimize synchronization overhead, 
like for example event-loop based systems (often, network applications), or IO systems which willingly perform blocking operations and need to perform them off the global concurrency pool. 

### Analysis of use-cases and the "sticky" preference semantics

The semantics explained in this proposal may at first seem tricky, however in reality the rule is quite straightforward:

- when there is a strict requirement for code to run on some specific executor, *it will* (and therefore disregard the "preference"),
- when there is no requirement where asynchronous code should execute, this proposal allows to specify a preference and therefore avoid hopping and context switches, leading to more efficient programs.

It is worth discussing how user-control is retained with this proposal. Most notably, we believe this proposal follows Swift's core principle of progressive disclosure. 

When developing an application at first one does not have to optimize for fewer context switches, however if as applications grow performance analysis diagnoses context switching being a problem this proposal gives developers the tools to, selectively, in specific parts of a code-base introduce sticky task executor behavior.

### Separating blocking code off the global shared pools

This proposal gives control to developers who know that they'd like to isolate their code off from callers. For example, imagine an IO library which wraps blocking IO primitives like read/write system calls. You may not want to perform those on the width-limited default pool of Swift Concurrency, but instead wrap APIs which will be calling such APIs with the executor preference of some "`DedicatedIOExecutor`" (not part of this proposal):

```swift
// MyCoolIOLibrary.swift

func blockingRead() -> Bytes { ... } 

public func callRead() async -> Bytes { 
  await withTaskExecutorPreference(DedicatedIOExecutor.shared) { // sample executor
    blockingRead() // OK, we're on our dedicated thread
  }
}

public func callBulk() async -> Bytes {
  // The same executor is used for both public functions
  await withTaskExecutorPreference(DedicatedIOExecutor.shared) { // sample executor
    await callRead() 
    await callRead()
  }
}
```

This way we won't be blocking threads inside the shared pool, and are not risking thread starving the entire application.

We can call `callRead` from inside `callBulk` and avoid unnecessary context switching as the same thread servicing the IO operation may be used for those asynchronous functions -- and no actual context switch may need to be performed when `callBulk` calls into `callRead` either.

End-users of this library don't need to worry about any of this, but the author of such a library is in full control over where execution will happen -- be it using task executor preference, or custom actor executors.

This also works the other way around, when a user of a library notices that it is doing blocking work which they would rather separate out onto a different executor. This is true even if the library has declared asynchronous methods but still is taking too long to yield the thread for some reason, causing issues to the shared pool.

```swift
// SomeLibrary
nonisolated func slowSlow() async { ... } // causes us issues by blocking
```

 In such situation, we, as users of given library can notice and work around this issue by wrapping it with an executor preference:

```swift
// our code
func caller() async {
  // on shared global pool...
  // let's make sure to run slowSlow on a dedicated IO thread:
  await withTaskExecutorPreference(DedicatedIOExecutor.shared) { // sample executor
    await slowSlow() // will not hop to global pool, but stay on our IOExecutor
  }
}
```

In other words, task executor preference gives control to developers when and where care needs to be taken.

The default of hop-avoiding when a preference is set has the benefit of optimizing for less context switching and can lead to better performance. 

It is possible to effectively restore the default behavior as-if no task executor preference was present, by setting the preference to the `globalConcurrentExecutor` which is the executor used by default actors, tasks, and free async functions when no task executor preference is set:

```swift
func function() async {
  // make sure to ignore caller's task executor preference, 
  // and always use the global concurrent executor.
  await withTaskExecutorPreference(globalConcurrentExecutor) { ... }
}
```

## Prior-Art

It is worth comparing with other concurrency runtimes with similar concepts to make sure if there are some common ideas or something different other projects have researched.

For example, in Kotlin, a `launch` which equivalent to Swift’s creation of a new task, takes a coroutine context which can contain an executor preference. The official documentation showcases the following example:

```kotlin
launch { // context of the parent, main runBlocking coroutine
    println("main runBlocking      : I'm working in thread ${Thread.currentThread().name}")
}
launch(Dispatchers.Unconfined) { // not confined -- will work with main thread
    println("Unconfined            : I'm working in thread ${Thread.currentThread().name}")
}
launch(Dispatchers.Default) { // will get dispatched to DefaultDispatcher 
    println("Default               : I'm working in thread ${Thread.currentThread().name}")
}
launch(newSingleThreadContext("MyOwnThread")) { // will get its own new thread
    println("newSingleThreadContext: I'm working in thread ${Thread.currentThread().name}")
}
```

Which is similar to the here proposed semantics of passing a specific executor preference. Notably though, because Swift has the concept of actor `isolation` the executor semantics introduced in this proposal are only a preference and will never override the executor requirements of actually strongly `isolated` code.

Kotlin jobs also inherit the coroutine context from their parent, which is similar to the here proposed executor inheritance works.

## Future directions

### Task executor preference and global actors

Thanks to the improvements to treating @SomeGlobalActor isolation proposed in [SE-NNNN: Improved control over closure actor isolation](https://github.com/swiftlang/swift-evolution/pull/2174) we would be able to that a Task may prefer to run on a specific global actor’s executor, and shall be isolated to that actor.

Thanks to the equivalence between `SomeGlobalActor.shared` instance and `@SomeGlobalActor` annotation isolations (introduced in the linked proposal), this does not require a new API, but uses the previously described API that accepts an actor as parameter, to which we can pass a global actor’s `shared` instance.

```swift
@MainActor 
var example: Int = 0

Task(executorPreference: MainActor.shared) { 
   example = 12 // not crossing actor-boundary
}
```

It is more efficient to write `Task(executorPreference: MainActor.shared) {}` than it is to `Task { @MainActor in }` because the latter will first launch the task on the inferred context (either enclosing actor, or global concurrent executor), and then hop to the main actor. The `executorPreference: MainActor.shared` spelling allows Swift to immediately enqueue on the actor itself.

### Static closure isolation 

It would be interesting to allow starting a task on a specific actor's executor, and have this infer the specific isolation.

Today the proposal does not allow using serial executors, which are strictly associated with actors to start a Task "on" such executor.
We could consider adding some form of such ability, and then be able to infer that the closure of a Task is isolated to the actor passed to `Task(executorPreference: some Actor)`.

The upcoming [SE-NNNN: Improved control over closure actor isolation](https://github.com/swiftlang/swift-evolution/pull/2174) proposal includes a future direction which would allow isolating a closure to a known other value.

This could be utilized to spell the `Task` initializer like this:

```swift
extension Task where ... {
  init<TargetActor>(
    executorPreference target: TargetActor,
    // ..., 
    operation: @isolated(target) () async -> ()
  ) where TargetActor: Actor
}
```

This would allow us to cut down on the noise of passing the isolated-on parameter explicitly and avoid a hop to the global executor before the task eventually hops back to the intended actor.
Today, if we were to allow a default actor's executor to be used as `TaskExecutor`, a similar API could be made that would look like this: 

```swift
actor Worker { func work() {} }
let worker: Worker = Worker()

Task(executorPreference: worker) { worker in // noisy parameter; though required for isolation purposes
  worker.work()
}
```

However, it would be noisy in the sense of having to repeat the `worker` parameter for purposes of isolation.



## Alternatives considered

### Do not provide any control over task executors

We considered if not introducing this feature could be beneficial and forcing developers to always pass explicit `isolated` parameters instead. We worry that this becomes a) very tedious and b) impossibly ties threading semantics with public API and ABI of methods. We are concerned that the lack of executor “preference” which only affects the nonisolated functions in a task hierarchy would cause developers to defensively and proactively create multiple versions of APIs. It would also only allow passing actors as the executors, because isolation is an actor concept, and therefore we’d only be able to isolate using serial executors, while we may want to isolate using general purpose `Executor` types.


## Revisions
- 1.6
  - introduce the global `var defaultConcurrentExecutor: any TaskExecutor` we we can express a task specifically wanting to run on the default global concurrency pool.
- 1.5
  - document that an executor may be both SerialExecutor and TaskExecutor at the same time 
- 1.4
  - added `unownedTaskExecutor` to UnsafeCurrentTask
- 1.3
  - introduce TaskExecutor in order to be able to implement actor isolation properly and still use a different thread for running default actors
  - wording cleanups
  - removal of the `Task(executorPreference: Actor)` APIs; we could perhaps revisit this if we made default actors' executors somehow aware of being a thread source as well etc. 
- 1.2
  - preference also has effect on default actors
- 1.1
  - added future direction about simplifying the isolation of closures without explicit parameter passing
  - removed ability to observe current executor preference of a task
