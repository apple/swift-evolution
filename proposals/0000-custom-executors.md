# Custom Executors

* Proposal: [SE-NNNN](NNNN-custom-executors.md)
* Authors: [John McCall](https://github.com/rjmccall)
* Review Manager: TBD
* Status: **Awaiting implementation**

## Introduction

It is sometimes important to control exactly how code is
executed. This proposal lays out a system of custom executors
which can schedule and run opaque jobs, and it describes
how actors and tasks can be directed to run on a specific
executor.

## Motivation

Swift's concurrency design is intentionally vague about the
details of how code is actually run.  Most code does not rely
on specific properties of the execution environment, such as
being run to a specific operating system thread, and instead
needs only high-level semantic properties, such as that no other
code will be accessing certain variables concurrently.
Maintaining flexibility about how work is scheduled onto threads
allows Swift to avoid certain performance pitfalls by default.

Nonetheless, it is sometimes useful to more finely control how
code is executed:

- The code may need to cooperate with an existing system that
  expects to run code in a certain way.

  For example, the system might expect certain kinds of work
  to be scheduled in special ways, like how some platforms
  require UI code to be run on the main thread, or how Apple's
  Core Data framework requires code that touches managed objects
  to be performed by the managed object context.

  For another example, a project might have a large amount
  of existing code which protects some state with a shared queue.
  In principle, this is the actor pattern, and the code could be
  rewritten to use Swift's actor support. However, it may be
  impossible to do that, or at least impractical to do it
  immediately. Using the existing queue as the executor for an
  actor allows code to adopt actors more incrementally.

- The code may depend on being run on a specific system thread.

  For example, some libraries maintain state in thread-local
  variables, and running code on the wrong thread will lead to
  broken assumptions in the library.

  For another example, not all execution environments are
  homogenous; some threads may be pinned to processors with
  extra capabilities.

- The code's performance may benefit from the programmer being
  more explicit about where code should run.

  For example, if one actor frequently makes requests of another,
  and the actors rarely benefit from running concurrently,
  configuring them to use the same executor may decrease the
  runtime costs of switching between them.

  For another example, if an asynchronous function makes many
  calls to the same actor without any intervening suspensions,
  running the function explicitly on that actor's executor may
  allow Swift to avoid a lot of switching overhead (or may even
  be necessary to perform those calls "atomically").

## Proposed solution

An *executor* is an object to which opaque jobs can be
submitted to be run later.  Executors come in two basic kinds:
*serial* executors run at most one job at a time, while
*concurrent* executors can run any number of jobs at once.
This proposal is concerned with both kinds, but their treatment
is quite different.

Swift's concurrency design includes both a default concurrent
executor, which is global to the process, and a default serial
executor implementation, which is used for actor instances. We
propose to allow this to be customized in a number of ways:

- A custom executor can be defined by defining a type that conforms
  to the `Executor` protocol.

- The concurrency library will provide functions to explicitly
  run an asynchronous function on a specific executor.

- Actors can override how code executes on them by providing
  a reference to a specific serial executor.

- The default concurrent executor can be replaced by defining
  certain symbols within the program.

## Detailed design

### Philosophy of thread usage

Before we elaborate much more on how to create custom
executors, it is important that we make a case for why the
default executors should be the way they are.

Swift's basic execution model is built on top of the target
platform's C execution model. Whenever Swift code is running,
it is running on a system thread. This is a basic guarantee
that Swift makes in order to ensure straightforward and
efficient interoperation with C code (as well as code from
other languages that build on the same basic execution model).

But a system thread simply runs code until the code blocks
or exits. Swift's concurrency design sees system threads as
expensive and rather precious resources. Blocking a thread
is usually undesirable because it means the thread sits idle,
running no code while reserving a substantial amount of
dedicated memory. If most threads are currently blocked, but
there's still more work to be done, then either that work will
be spuriously blocked or new threads must be created in
order to run it. This can easily lead to too many threads
being created and starving the system. Furthermore, if many
more threads are runnable than there are cores to run them,
the system will have to waste time switching between threads
rather than performing useful work.

It is therefore best if the system allocates a small number of
threads --- just enough to saturate the available cores ---
and for those threads only block for extended periods when
there is no pending work in the program. Individual functions
cannot effectively make this decision about blocking,
because they lack a holistic understanding of the state of
the program. Instead, the decision must be made by a centralized
system which manages most of the execution resources in the
program.

This basic philosophy of how best to use system threads drives
some of the most basic aspects of Swift's concurrency design. In
particular, the main reason to add `async` functions is to make it
far easier to write functions that, unlike standard functions, will
reliably abandon a thread when they need to wait for something
to complete. Similarly, the design avoids operations which rely
on blocking threads on arbitrary future work, like the traditional
condition variable primitive, because allowing widespread
thread-blocking of this sort can easily starve a fixed-width
thread pool or even lead to deadlock.

In order to facilitate this holistic global management of
threads, Swift does not provide low-level mechanisms for creating
or blocking threads. Instead, it relies on abstract execution
services to which work can be asynchronously submitted
(*executors*).

### Formal treatment of concurrency

This design will need to make formal statements about the
inter-thread ordering of certain events. Swift does not yet have a
formal memory model of its own, but we will sketch one here that
should be adequate in practice to reason about the correctness of
programs under concurrency.

As discussed above, Swift is built on top of the C thread model.
Accordingly, it is also built on top of the C memory model, at least
to a certain extent. In this design, we will use the term of art
*happens-before* (and its reverse, *happens-after*), which should
be understood to be consistent with the *happens before* relationship
described by the C standard, as well as related standards such as
C++. We will consistently use hyphenation in this term in order to
emphasize the formal nature of the claim being made.

All memory effects in Swift are associated with a formal access
period, the beginning and end of which are fully sequenced with other
events on the current thread of execution, up to observation by a
valid program. If the end of an access happens-before an event, the
memory effects of the access happen-before that event. Similarly, if
the beginning of an access happens-after an event, the memory effects
of the access happen-after that event.

Thus, for example, calling a C function which acquires a mutex,
mutating the contents of a class property in Swift code, and then
calling a C function to release the mutex will correctly order memory
even though the execution mixes operations in C and Swift.

### A low-level design

The API design of executors is intended to support high-performance
implementations, with an expectation that custom executors will be
primarily implemented by experts. Therefore, the following design
heavily prioritizes the reliable elimination of abstraction costs
over most other conceivable goals. In particular, the primitive
operations specified by protocols are generally expressed in terms
of opaque, unsafe types which implementations are required to use
correctly. These operations are then used to implement more convenient
APIs as well as the high-level language operations of Swift
concurrency.

### Executors and jobs

All executors must conform to the following protocol:

```swift
protocol Executor: AnyObject {
  /// Enqueue a job on this executor to run asynchronously.
  func enqueue(_ job: UnownedJobRef)

  /// Get an unowned reference to this executor. The reference
  /// must remain valid as long as the executor is.
  func asUnownedRef() -> UnownedExecutorRef
}
```

Executors are required to follow certain ordering rules when executing
their jobs:

- The call to `job.execute(currentExecutor:)` must happen-after the call
  to `enqueue(_:)`.

- If the executor is a serial executor, then the execution of all jobs
  must be totally ordered: for any two different jobs *A* and *B*
  submitted to the same executor with `enqueue(_:)`, it must be true
  that either all events in *A* happen-before all events in *B* or all
  events in *B* happen-before all events in *A*.

The `UnownedJobRef` type is the opaque type of schedulable jobs.
The job reference is "self-owning", meaning that executing it
takes over ownership of the reference, potentially invalidating
it immediately. The executor must not assume the validity of the
job after executing it. (If Swift supported move-only types,
`UnownedJobRef` could be one, and `execute(currentExecutor:)` would
be a consuming method.)

```swift
struct UnownedJobRef {
  /// Get the requested priority of the job.
  var priority: Priority { get }

  /// Execute the job on the current thread, claiming to be
  /// running on the given executor. The executor reference must
  /// remain valid during this call unless it is a serial executor
  /// which the job successfully gives up.
  ///
  /// Calling this immediately invalidates the job reference
  /// from the caller's perspective, and the caller must not
  /// refer to the job again.
  func execute(currentExecutor: UnownedExecutorRef)

  /// Create an unsafe job reference to run the given function
  /// at the given priority. This may require allocating memory.
  init(priority: Priority = .default, operation: @escaping () -> ())
}
```

The `UnownedExecutorRef` type is the opaque type of a reference
to an executor. This type packs certain highly valuable information
into the reference. It is an unmanaged (that is, unsafe) reference
to the executor. Whatever context produces an `UnownedExecutorRef`
is generally responsible for keeping the reference alive while it
is in use, often by maintaining some other stable relationship.

The identity of an `UnownedExecutorRef` is determined by object
identity.  The appropriate flags must be set consistently for the
executor.

```swift
struct UnownedExecutorRef: Equatable {
  init<T: SerialExecutor>(serialExecutor: T, supportsSwitching: Bool)
  init<T: Executor>(concurrentExecutor: T)

  static var defaultConcurrent: UnownedExecutorRef { get }

  var isSerial: Bool { get }

  func asOwned() -> Executor
}
```

### The default global concurrent executor

The default concurrent executor is used to run jobs that don't
need to run somewhere more specific. It is based on a fixed-width
thread pool that scales to the number of available cores. Programmers
therefore do not need to worry that creating too many jobs at once
will cause a thread explosion that will starve the program of
resources.

Some environments may wish to fully replace the default concurrent
executor. (For example, they may have their own fixed-width
thread pool, and having two fixed-width pools in use at once
completely defeats the purpose.) Programs should not attempt
to do this by specifying a custom executor on absolutely
everything in the program; that would be both highly invasive
and doomed to failure. We think it should be possible to directly
support hooking the default concurrent executor, perhaps by
overriding a weak symbol, the same way that C programs can
override `malloc`.  We won't discuss this further in this proposal
because it doesn't otherwise impact the language and library design.

### The default serial executor implementation

The default serial executor implementation is separately instantiated
for each actor that doesn't declare a custom executor; see "Actor
executors" below. It is based on an "asynchronous lock" design which
allows existing threads to immediately begin executing code on behalf
of the actor rather than requiring functions to suspend and resume
executing asynchronously. This process is called *switching*. In
situations where switching is impossible, such as when the actor is
already executing on a thread, a job to process the actor will be
scheduled onto the default concurrent executor.

Switching significantly complicates the design of serial executors,
which can be seen in the `SerialExecutor` protocol below. This
complexity could be eliminated by only allowing default serial
executors to participate in switching. Currently, we are reluctant
to hardcode this limitation into the protocol design.

### Actor executors

An actor's executor must conform to `SerialExecutor`, which refines
the `Executor` protocol. `SerialExecutor` makes additional guarantees
about the behavior of `execute(currentExecutor:)` and also adds
several new methods which are used to allow the executor to opt in
to supporting "switching" serial executors the same way that the
default serial executor does.

TODO: if we need to support blocking actors, we will need to extend
this significantly; otherwise, actor executors will need to assume
that returning from a job should always unblock the executor.
(Presumably we do not want to block actor executors by blocking
their thread.)

```swift
protocol SerialExecutor: Executor {
  /// Is it possible for this executor to give up the current thread
  /// and allow it to start running a different actor?
  var canGiveUpThread: Bool { get }

  /// Given that canGiveUpThread() previously returned true, give up
  /// the current thread.
  func giveUpThread()

  /// Attempt to start running a task on the current actor.  Returns
  /// true if this succeeds.
  func tryClaimThread() -> Bool
}
```

All actors implicitly conform to the `Actor` protocol.  An actor
must provide a serial executor.

```swift
protocol Actor: AnyObject {
  /// Return the serial executor for this actor.
  ///
  /// This must always return the same reference, and `isSerial()`
  /// must return true for the reference.
  ///
  /// Keeping the actor reference valid must be sufficient to keep
  /// the executor reference valid.
  var serialExecutor: UnownedExecutorRef { get }
}
```

An `actor` may derive its executor implementation in one
of the following ways. We may add more ways in the future.

- The actor may declare a property named `serialExecutor`.
  The property must not be actor-isolated.

- The actor may declare a property named `delegateActor`.
  The property must not be actor-isolated. The type of the property
  must be convertible to `Actor`. The property must always return
  the same actor. Actor safety checking should allow uses of the
  actor state and functions of `delegateActor` from the actor
  functions of the delegating actor.

  The `serialExecutor` property will be synthesized as if
  the following:

  ```swift
  public final var serialExecutor: UnownedExecutorRef {
    return delegateActor.serialExecutor
  }
  ```

  It will be `@inlinable` if `delegateActor` is.

- Otherwise, the actor will use the default serial executor
  implementation. Note that this implicitly adds storage to the
  class.

  `serialExecutor` will be synthesized as `public final`.
  It will be `@inlinable` if the actor class is `frozen`.

In some situations, it may be useful for an actor to delegate
to another actor's executor with `serialExecutor` rather than
delegating to the actor with `delegateActor`.  This allows the
system to dynamically benefit from the executor being shared
(and thus avoid extra dispatches) without semantically tying
the actors' isolation together (which could become hard-coded
and hard to eliminate).

### Explicit scheduling

The following operation is provided in order to perform explicit
scheduling onto an executor:

```swift
extension Executor {
  /// Run the given async operation explicitly on this executor.
  func run<T>(operation: () -> async throws T) async rethrows -> T
}
```

The following operation is provided in order to perform explicit
scheduling onto an actor's executor:

```swift
extension Actor {
  /// Run the given async operation explicitly on the actor's executor.
  ///
  /// Ideally, when the operation is an explicit closure expression, the
  /// body of that closure should isolation-checked as if it were an actor
  /// function of this actor. That may be somewhat challenging to
  /// define in the design, since it's only reliable when this method
  /// is called on a constant actor reference. Since isolation checking
  /// is defined to be independent of ordinary type-checking, it should
  /// be possible to improve this in later versions of the compiler
  /// without affecting source compatibility.
  func run<T>(operation: () -> async throws T) async rethrows -> T
}
```

## Source compatibility

This feature describes new APIs that are expected to be used in
conjunction with the `async`/`await` feature; there is no anticipated
impact on existing code.

## Effect on ABI stability

Swift's scheduling runtime must maintain references to executors in
order to correctly implement the semantics of task/actor scheduling.
Custom executors mean that the runtime must propagate and store
slightly more information, but we believe this can be done in a fairly
extensible manner.  In general, the ABI here will likely favor the
built-in (default) executors.

The design of the `Executor` and `SerialExecutor` protocols will be
ABI, and that ABI may limit the sorts of scheduling tricks that can be
done with non-default executor implementations in the future.  The
design of `SerialExecutor` currently does not support non-reentrant
actors, and it does not support executors for which dispatch is
always synchronous (e.g. that just acquire a traditional mutex).

## Effect on API resilience

While some APIs may depend on being executed on particular executors,
this proposal makes no effort to formalize that in interfaces, as opposed
to being an implementation detail of implementations, and so has no API
resilience implications.

If this is extended in the future to automatic, declaration-driven
executor switching, as actors do, that would have API resilience
implications.

## Alternatives considered

The proposed ways for actors to opt in to custom executors are brittle,
in the sense that a typo or some similar error could accidentally leave
the actor using the default executor. This could be fully mitigated by
requiring actors to explicitly opt in to using the default executor;
however, that would be an unacceptable burden on the common case. Short
of that, it would be possible to have a modifier that marks a declaration
as having special significance, and then complain if the compiler
doesn't recognize that significance. However, there are a number of
existing features that use a name-sensitive design like this, such as
dynamic member lookup ([SE-0195](0195-dynamic-member-lookup.md)).
A "special significance" modifier should be designed and considered more
holistically.
