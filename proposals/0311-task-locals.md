# Task Local Values

* Proposal: [SE-0311](0311-task-locals.md)
* Authors: [Konrad 'ktoso' Malawski](https://github.com/ktoso)
* Review Manager: [John McCall](https://github.com/rjmccall)
* Status: **Implemented (Swift 5.5)**
* Previous Revision: [1](https://github.com/swiftlang/swift-evolution/blob/884df3ad6020f0724e06184534b21dd76bd6f4bf/proposals/0311-task-locals.md), [2](https://github.com/swiftlang/swift-evolution/blob/cd1aaef28802a26986094c1f851c261acc796cb6/proposals/0311-task-locals.md), [3](https://github.com/swiftlang/swift-evolution/blob/79b44f3cd15eefc675196136858a5f76a3e58656/proposals/0311-task-locals.md)
* Review: ([first review](https://forums.swift.org/t/se-0311-task-local-values/47478), of revision 1) ([second review](https://forums.swift.org/t/se-0311-2nd-review-task-local-values/47738), of revisions 2 and 3) ([third review](https://forums.swift.org/t/se-0311-3rd-review-task-local-values/49122), of revision 4) ([acceptance](https://forums.swift.org/t/accepted-se-0311-task-local-values/50120))

## Table of Contents

* [Introduction](#introduction)
* [Motivation](#motivation)
* [Proposed solution](#proposed-solution)
  + [Task Local Values](#task-local-values-1)
* [Detailed design](#detailed-design)
  + [Declaring task-local values](#declaring-task-local-values)
  + [Binding task-local values](#binding-task-local-values)
    - [Binding values for the duration of a child-task](#binding-values-for-the-duration-of-a-child-task)
    - [Binding task-local values from synchronous functions](#binding-task-local-values-from-synchronous-functions)
    - [Task-local value and tasks which outlive their scope](#task-local-value-and-tasks-which-outlive-their-scope)
  + [Task-local value lifecycle](#task-local-value-lifecycle)
  + [Reading task-local values](#reading-task-local-values)
    - [Reading task-local values: implementation details](#reading-task-local-values-implementation-details)
    - [Task-locals in contexts where no Task is available](#task-locals-in-contexts-where-no-task-is-available)
    - [Child task and value lifetimes](#child-task-and-value-lifetimes)
      * [Task-local value item allocations](#task-local-value-item-allocations)
  + [Similarities and differences with SwiftUI's `Environment`](#similarities-and-differences-with-swiftuis-environment)
* [Prior Art](#prior-art)
  + [Kotlin: CoroutineContext[T]](#kotlin-coroutinecontextt)
  + [Java/Loom: Scope Variables](#javaloom-scope-variables)
  + [Go: explicit context passing all the way](#go-explicit-context-passing-all-the-way)
* [Alternatives Considered](#alternatives-considered)
  + [Surface API: Type-based key definitions](#surface-api-type-based-key-definitions)
  + [Surface API: Key-less value definitions](#surface-api-key-less-value-definitions)
* [Rejected Alternatives](#rejected-alternatives)
  + [Plain-old Thread-Local variables](#plain-old-thread-local-variables)
  + [Dispatch Queue Specific Values](#dispatch-queue-specific-values)
* [Intended use-cases](#intended-use-cases)
  + [Use case: Distributed Tracing & Contextual Logging](#use-case-distributed-tracing--contextual-logging)
    - [Contextual Logging](#contextual-logging)
    - [Function Tracing](#function-tracing)
    - [Distributed Tracing](#distributed-tracing)
    - [Future direction: Function wrapper interaction](#future-direction-function-wrapper-interaction)
  + [Use case: Mocking internals (Swift System)](#use-case-mocking-internals-swift-system)
  + [Use case: Progress Monitoring](#use-case-progress-monitoring)
  + [Use case: Executor configuration](#use-case-executor-configuration)
* [Future Directions](#future-directions)
  + [Additional configuration options for `@TaskLocal`](#additional-configuration-options-for-tasklocal)
  + [Tracing annotations with Function Wrappers](#tracing-annotations-with-function-wrappers)
  + [Language features to avoid nesting with `withValue`](#language-features-to-avoid-nesting-with-withvalue)
  + [Specialized TaskLocal Value Inheritance Semantics](#specialized-tasklocal-value-inheritance-semantics)
    - ["Never" task-local value inheritance](#never-task-local-value-inheritance)
* [Revision history](#revision-history)
* [Source compatibility](#source-compatibility)
* [Effect on ABI stability](#effect-on-abi-stability)
* [Effect on API resilience](#effect-on-api-resilience)


## Introduction

With Swift embracing asynchronous functions and actors, asynchronous code will be everywhere. 

Therefore, the need for debugging, tracing and otherwise instrumenting asynchronous code becomes even more necessary than before. At the same time, tools which instrumentation systems could have used before to carry information along requests — such as thread locals or queue-specific values — are no longer compatible with Swift's Task-focused take on concurrency.

Previously, tool developers could have relied on thread-local or queue-specific values as containers to associate information with a task and carry it across suspension boundaries. However, these mechanisms do not compose well in general, have known "gotchas" (e.g., forgetting to carefully maintain and clear state from these containers after a task has completed to avoid leaking them), and do not compose at all with Swift's task-first approach to concurrency. Furthermore, those mechanisms do not feel "right" given Swift's focus on Structured Concurrency, because they are inherently unstructured.

This proposal defines the semantics of _Task Local Values_. That is, values which are local to a `Task`.

Task-local values set in a task _cannot_ out-live the task, solving many of the pain points relating to un-structured primitives such as thread-locals, as well as aligning this feature closely with Swift's take on Structured Concurrency.

Swift-evolution threads:

- [Review #1](https://forums.swift.org/t/se-0311-task-local-values/47478/11),
- [Pitch #1](https://forums.swift.org/t/pitch-task-local-values/42829/15).

## Motivation

Task Local Values are a significant improvement over thread-local storage in that it is a better fit for Swift's concurrency model because it takes advantage of its structured nature. In existing code, developers have used thread-local or queue-specific values to associate state with each thread/queue, however the exact semantics of those mechanisms made it difficult and error prone in reality.

Specifically, previously developers could have used thread-local or queue-specific values to achieve some of these features, however the exact semantics of those made it difficult and error prone in reality. 

> The use of thread-locals in highly asynchronous libraries is generally frowned upon because it is so difficult to get right, and generally only adds to the confusion rather than helping one achieve transparent context propagation. 
> 
> This is why currently [Swift Distributed Tracing](https://github.com/apple/swift-distributed-tracing) had to revert to using explicit context passing, making asynchronous APIs even more verbose than they already are.


Finally, those mechanisms are outright incompatible with asynchronous code that hops between execution contexts, which also includes Swift's async/await execution semantics which guarantee specific _executors_ or _actors_ to be used for execution, however never guarantees any specific queue or thread use at all.

> For discussion about alternative approaches to this problem please refer to [Prior Art](#prior-art) and [Alternatives Considered](#alternatives-considered).

## Proposed solution

### Task Local Values

Tasks already require the capability to "carry" metadata with them, and that metadata used to implement both cancellation and priority propagation for a parent task and its child tasks. Specifically Task API's exhibiting similar behavior are: `Task.currentPriority`, and `Task.isCancelled`. Task-local values do not directly use the same storage mechanism, as cancellation and priority is somewhat optimized because _all_ tasks carry them, but the semantics are the same.

We propose to expose the Task's internal ability to "carry metadata with it" via a Swift API, *aimed for library and instrumentation authors* such that they can participate in carrying additional information with Tasks, the same way as Tasks already do with priority and deadlines and other metadata.

Task local values may be read from any function running within a task context. This includes *synchronous* functions which were called from an asynchronous function. 

The functionality is also available even if no Task is available in the call stack of a function at all. In such contexts, the task-local APIs will effectively work similar to thread-local storage meaning that they cannot automatically propagate to new (unstructured) threads (e.g. pthread) created from such context. They _will continue to work as expected_ with Task APIs nested inside such scopes however: for example, if `async{}` is used to create an asynchronous task from such synchronous function with no task available, it will inherit task-locals from the synchronous context.

A task-local must be declared as a static stored property, and annotated using the `@TaskLocal` property wrapper. 

```swift
enum MyLibrary {
  @TaskLocal
  static var requestID: String?
}
```

> 💡 Note: Property wrappers are currently not allowed on global declarations.  If this is changed, it should become possible to declare top-level task locals.

Each task-local declaration represents its own, independent task-local storage. Reading from one declaration will never observe a value stored using a different declaration, even if the declarations look exactly the same.

Because of those pitfalls with creating multiple instances of the same task local identifier, we propose to diagnose and fail at compile time if the `@TaskLocal` property wrapper is not defined on a static or global property. 

> In order to do so, we will extend the internal `public static subscript<T>(_enclosingInstance object: T, ...)` subscript mechanism to require "no enclosing instance", which will cause the apropriate compile time error reporting to be triggered if such wrapper is used on a non-static or non-global property.

The diagnosed error would look like this:

```swift
enum MyLibrary {
  @TaskLocal
  var requestID: String?
    // error: @TaskLocal declaration 'requestID' must be static. 
    // Task-local declarations must be static stored properties.
}
```

It is expected that task-local property declarations will often decide to use an optional type, and default it to `nil`. 

Some declarations however may have "good defaults", such as an empty container type, or some other representation of "not present". For example if `Task.Priority` were expressed using a task local, its `.unspecified` value would be a perfect default value to use in the task-local property declaration. We are certain similar cases exist, and thus want to allow users to retain full control over the type of the property, even if most often an optional is the right thing to use.

Accessing the value of a task-local property is done by accessing the property wrapper annotated property.


```swift
func asyncPrintRequestID() async {
  let id = MyLibrary.requestID
  print(id ?? "no-request-id")
}

func syncPrintRequestID() { // also works in synchronous functions
  let id = MyLibrary.requestID
  print(id ?? "no-request-id")
}
```

The task local value is accessible using the same API from async and non async functions, even though it relies on running inside a Task. The asynchronous function always performs a lookup inside the current task, since it is guaranteed to have a current task, while the synchronous function simply immediately returns the default value if it is _not_ called from within a Task context. 

> :warning: Task-local value lookups are more expensive than a direct static property lookup. They involve a thread-local access and scanning a stack of value bindings until a value is found, or the end of the stack is reached. As such, task-local values should be used with care, and e.g. hoisted out of for loops etc, so they are only looked up _once_ whenever possible.

Binding values is the most crucial piece of this design, as it embraces the structured nature of Swift's concurrency. Unlike thread-local values, it is not possible to just "set" a task local value. This avoids the issue of values being set and forgotten about leading to leaks and hard to debug issues with unexpected values being read in other pieces of code which did not expect them. 

By using scopes and limiting a value's lifetime to the task's lifetime the implementation can use efficient task-local allocation techniques, thereby avoiding the system-wide allocator. Once the scope ends, the child task ends, and the associated task-local value is discarded.

To bind a specific task-local declaration to a specific value, we can use the `withValue(_:operation:)` function which is declared on the property wrapper type. In order to access this function the `$` sign must be prefixed to the property name, to access the property wrapper's projected value rather than the wrapped value itself:

```swift
await MyLibrary.$requestID.withValue("1234-5678") {
  await asyncPrintRequestID() // prints: 1234-5678
  syncPrintRequestID()        // prints: 1234-5678
}

await asyncPrintRequestID()  // prints: no-request-id
syncPrintRequestID()         // prints: no-request-id
```

The `withValue` operation is executed synchronously, and no additional tasks are created to execute them.

It is also possible to bind the same key multiple times while executing in the same task. This can be thought of the most recent binding shadowing the previous one, like this:

```swift
syncPrintRequestID()                             // prints: no-request-id

await MyLibrary.$requestID.withValue("1111") { 
  syncPrintRequestID()                           // prints: 1111
  
  await MyLibrary.$requestID.withValue("2222") { 
    syncPrintRequestID()                         // prints: 2222
  }
  
  syncPrintRequestID()                           // prints: 1111
}

syncPrintRequestID()                             // prints: no-request-id
```

A task local is readable by any function invoked from a context that has set the value, regardless of how nested it is. For example, it is possible for an asynchronous function to set the value, call through a few asynchronous functions, and finally one synchronous function. All the functions are able to read the bound value, like this:

```swift
func outer() async -> String? {
  await MyLibrary.$requestID.withValue("1234") { 
    MyLibrary.requestID // "1234"  
    return middle() // "1234"
  }
}

func middle() async -> String? {
  MyLibrary.requestID // "1234"
  return inner() // "1234"
}


func inner() -> String? { // synchronous function
  return MyLibrary.requestID // "1234"
}
```

The same property holds for child tasks. For example, if we used a task group to create a child task, it would inherit and read the same value that was set in the outer scope by it's parent. Thanks to guarantees of structured concurrency and child tasks never out-living their parents this still is able to use the efficient storage allocation techniques, and does not need to employ any locking to implement the reads:

```swift
await MyLibrary.$requestID.withValue("1234-5678") {
  await withTaskGroup(of: String.self) { group in 
    group.addTask { // add child task running this closure
      MyLibrary.requestID // returns "1234-5678", which was bound by the parent task
    }
                                        
    return await group.next()! // returns "1234-5678"
  } // returns "1234-5678"
}
```

The same operations also work and compose naturally with child tasks created by `async let` and any other future APIs that would allow creating child tasks.

## Detailed design

### Declaring task-local values

Task-local values need to declare a _"key"_ which will be used to access them. This key is represented by the property wrapper instance that is created around the `@TaskLocal` annotated property.

The `TaskLocal` property wrapper is used to declare task-local keys, based off a static property. 

The property wrapper is defined as:

```swift
@propertyWrapper
public final class TaskLocal<Value: Sendable>: Sendable, CustomStringConvertible {
  let defaultValue: Value

  public init(wrappedValue defaultValue: Value) {
    self.defaultValue = defaultValue
  }

  @discardableResult
  public func withValue<R>(_ valueDuringOperation: Value, 
                           operation: () async throws -> R,
                           file: String = #file, line: UInt = #line) async rethrows -> R { ... }
  
  public var wrappedValue: Value {
    ...
  }
  
  public var projectedValue: TaskLocal<Value> {
    get { self }

    @available(*, unavailable, message: "use '$myTaskLocal.withValue(_:operation:)' instead")
    set {
      fatalError("Illegal attempt to set a \(Self.self) value, use `withValue(...) { ... }` instead.")
    }
  }


  public var description: String {
    "\(Self.self)(defaultValue: \(self.defaultValue))"
  }

}
```

Values stored in task-local storage must conform to the [`Sendable` marker protocol](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md), which ensures that such values are safe to be used from different tasks. Please refer to the `Sendable` proposal for more details on the guarantees and checks it introduces.

The property wrapper itself must be a `class` because we use it's stable object identifier as *key* for the value lookups performed by the concurrency runtime.

The implementation of task locals relies on the existence of `withUnsafeCurrentTask` from the [Structured Concurrency proposal](0304-structured-concurrency.md). This is how we are able to obtain a task reference, regardless if within or outside of an asynchronous context.

### Binding task-local values

Task locals cannot be "set" explicitly, rather, a scope must be formed within which the key is bound to a specific value. 

This addresses pain-points of task-local values predecessor: thread-locals, which are notoriously difficult to work with because, among other reasons, the hardships of maintaining the set/recover-previous value correctness of scoped executions. It also is cleanly inspired by structured concurrency concepts, which also operate in terms of such scopes (child tasks).

> Please refer to [Rejected alternatives](#rejected-alternatives), for an in depth analysis of the shortcomings of thread-locals, and how task-local `withValue` scopes address them.  

Binding values is done by using the `$myTaskLocal.withValue(_:operation:)` function, which adds task-local values for the duration of the operation:

```swift
@discardableResult
public func withValue<R>(
  _ valueDuringOperation: Value,
  operation: () async throws -> R
) async rethrows -> R
```

A synchronous version of this function also exists, allowing users to spare the sometimes unnecessary `await` call, if all code called within the `operation` closure is synchronous as well:

```swift
@discardableResult
public func withValue<R>(
  _ valueDuringOperation: Value,
  operation: () throws -> R
) rethrows -> R
```

The synchronous version of this API can be called from synchronous functions, even if they are not running on behalf of a Task. The APIs will uphold their expected semantics. Details about how this is achieved will be explained in later sections.

> In the future, if the `reasync` modifier is implemented and accepted, these two APIs could be combined into one.

Task-local storage can only be modified by the "current" task itself, and it is not possible for a child task to mutate a parent's task-local values. 

#### Binding values for the duration of a child-task

The scoped binding mechanism naturally composes with child tasks.

Binding a task-local value for the entire execution of a child task is done by changing the following:

```swift
async let dinner = cookDinner()
```

which–if we desugar the syntax a little bit to what is actually happening–is equivalent to the right hand side of the `async let` being a closure that will execute concurrently:

```swift
async let dinner = { 
  cookDinner() 
}
```

With that in mind, we only need to wrap the body of the right hand-side with the task-local binding to achieve the result of the value being bound for the entire duration of a specific child task.

```swift
async let dinner = Lib.$wasabiPreference.withValue(.withWasabi) {
  cookDinner()
}
```

This will set the wasabi preference task-local value _for the duration of that child task_ to `.withWasabi`. 

If we had two meals to prepare, we could either set the value for both of them, like this:

```swift
await Lib.$wasabiPreference.withValue(.withWasabi) {
  async let firstMeal = cookDinner()
  async let secondMeal = cookDinner()
  await firstMeal, secondMeal
}
```

And finally, if we wanted to set the `withWasabi` reference for most of the tasks in some scope, except one or two of them, we can compose the scopes to achieve this, as expected:

```swift
await Lib.$wasabiPreference.withValue(.withWasabi) {
  async let firstMeal = cookDinner()
  async let secondMeal = cookDinner()
  async let noWasabiMeal = Lib.$wasabiPreference.withValue(.withoutWasabi) {
    cookDinner()
  }
  await firstMeal, secondMeal, noWasabiMeal
}
```

The example here is arguably a little silly, because we could just pass the wasabi preference to the functions directly in this case. But it serves well to illustrate the functioning of the scoping mechanisms. 

In practice, please be careful with the use of task-locals and don't use them in places where plain-old parameter passing would have done the job. Task-local values should be reserved to metadata that does not affect the logical outcome of function calls, but only affects side effects or other configuration parameters of functions. If unsure if a value should be passed directly or via a task-local, err on the side of passing it explicitly and keep in mind that task-locals are primarily designed for "context metadata" such as trace identifiers, authorization tokens etc.

#### Binding task-local values from synchronous functions

Reading and binding task-local values is also possible from synchronous functions.

The same API is used to bind and read values from synchronous functions, however the closure passed to `withValue` when binding a key to a specific value cannot be asynchronous if called from a synchronous function itself (as usual with async functions).

Sometimes, it may happen that the synchronous `withValue` function is called from a context that has no Task available to attach the task-local binding to. This should rarely be the case in typical Swift programs as all threads and calls should originate from _some_ initiating asynchronous function, however e.g. if the entry point is a call from a C-library or other library which manages it's own threads, a Task may not be available. The task-local values API _continues to work even in those (task-less) circumstances_, by simulating the task scope with the use of a special thread-local in which the task-local storage is written. 

This means that as long as the code remains synchronous, all the usual task-local operations will continue to work even if the functions are called from a task-less context.

```swift
func synchronous() { // even if no Task is available to this function, the APIs continue to work as expected
  printTaskLocal(TL.number) // 1111

  TL.$number.withValue(2222) { // same as usual
    printTaskLocal(TL.number) // 2222
  }
  
  printTaskLocal(TL.number) // 1111
}
```

#### Task-local value and tasks which outlive their scope

> Note: In the original pitch it was proposed to allow detached tasks to be forced into inheriting task-local values. We have since decided that detached tasks shall be _fully detached_, and a new API to introduce "continue work asynchronously, with carrying priority and task-local values" will be introduced shortly. 
>
> This new core primitive has been implemented here: [Add "async" operation for continuing work asynchronously. #37007](https://github.com/apple/swift/pull/37007), and will be pitched to Swift Evolution shortly. This section expressess its semantics in terms of the new construct.

Sometimes it may be necessary to "continue work asynchronously" _without waiting_ for the result of such operation. 

Today there exists the `detach` operation which steps out of the realm of Structured Concurrency entirely, and may out-live it's calling scope entirely. This is problematic for task-local values which are built and optimized entirely around the structured notion of child-tasks. Also, a detached task's purpose is to "start from a clean slate" (i.e. detach) from the context it was created from. In other words, detached tasks cannot and will not inherit task-local values (!), much in the same way as they would not inherit the execution context or priority of the calling context.

To illustrate the interaction of detached tasks and task-locals, consider the following example:

```swift
await Lib.$sugar.withValue(.noSugar) { 
  assert(Lib.sugar == .noSugar)
  
  detach { // completely detaches from enclosing context!
    assert(Lib.sugar == .noPreference) // no preference was inherited; it's a detached task!
  }
  
  assert(Lib.sugar == .noSugar)
} 
```

As expected, because the *detached task* completely discards any contextual information from the creating task, no `.sugar` preferences were automatically carried through to it. This is similar to task priority, which also is never automatically inherited in detached tasks.

If necessary, it is possible to make a detached task carry a specific priority, executor preference and even task-local value by handling the propagation manually:

```swift
let sugarPreference = Lib.sugar                 // store the sugar preference in task-1
detach(priority: Task.currentPriority) {        // manually propagate priority
  await Lib.$sugar.withValue(sugarPreference) { // restore the sugar preference in detached-task
    assert(Lib.sugar == preference)
  }
}
```

While this is quite labor intensive and boilerplate heavy, it is intentional that detached tasks never carry any of their legacy around with them. So if a detached task really has to carry some information, it should do so explicitly.

At the same time, the new `async` (naming pending, perhaps `send` (?!)) operation _does_ inherit all of the following properties of the creating task: execution context, task priority, and task-local values. 

The async operation will be pitched independently, but for the sake of this proposal we only need to focus on the fact how it propagates task-local values. Consider the following snippet:

```swift
// priority == .background
await Lib.$tea.withValue(.green) { 
  async { 
    await Task.sleep(10_000)
    // assert(Task.currentPriority == .background) // inherited from creating task (!)
    assert(Lib.tea == .green)                      // inherited from creating task
    print("inside")
  }
}

print("outside")
```

Note that the `async` operation, similar to a `detach` operation, is allowed to out-live the creating task. I.e. the operation is __not__ a child-task, and as such the usual technique of task-locals to rely on the task tree for storage of the task-locals cannot be used here.

The implementation ensures correctness of this by _copying_ all task-local value bindings over to the new async task at the point of creation (line 3 in the above example). This means that such operation is slightly heavier than creating a plain child-task, because not only does the task have to be likely heap allocated, it also needs to copy over all task-local bindings from the creating task.

Please note that what is copied here are only the bindings, i.e. if a reference counted type was bound using `withValue` in the creating task, what is copied to the new task is a reference to the previous task, along with incrementing the reference count to it to keep the referenced object alive.

---

One other situation where a task might out-live the `withValue` lexical-scope is a specific anti-pattern within task groups. This situation is reliabily detected at runtime and cause a crash when it is encountered, along with a detailed explanation of the issue.

This one situation where a `withValue` scope is not enough to encapsulate the lifetime of a child-task is if the binding is performed _exactly_ around a TaskGroup's `group.addTask`, like this:

```swift
withTaskGroup(of: String.self) { group in 

  Trace.$name.withValue("some(func:)") { // RUNTIME CRASH! 
    // error: task-local value: detected illegal task-local value binding at Example.swift:68. 
    // <... more details ... >

    group.addTask { 
      Trace.name
    }
  } // end-withValue
  
  // the added child-task lives until it is pulled out of the group by next() here:
  return group.next()!
}
```

This is an un-supported pattern because the purpose of `group.addTask` (and `group.addTaskUnlessCancelled`) is explicitly to add off a child-task and return immediately. While the _structure_ of these child-tasks is upheld by no child-task being allowed to escape the task group, the child-tasks do "escape" the scope of the `withValue` — which causes trouble for the internal workings of task locals, which are allocated using an efficient task-local allocation mechanism.

At the same time, the just shown pattern can be seen as simply wrong usage of the API and programmer error, violating the structured nature of child-tasks. Instead, what the programmer should do in this case is either, set the value for the entire task group, such that all children inherit it:

```swift
await Trace.$name.withValue("some(func:)") { // OK!
  await withTaskGroup(...) { group in
    group.addTask { ... }
  }
}
```

or, set it _within_ the added child-task, as then the task-local allocation will take place inside the child-task, and the lifetime of the value will be correct again, i.e. bounded by the closure lifetime of the added child-task:

```swift
await withTaskGroup(...) { group in
  group.addTask {
    await Trace.$name.withValue("some(func:)") { // OK!
      ...
    }
  }
}
```

### Task-local value lifecycle

Task-local values are retained until `withValue`'s `operation` scope exits. Effectively this means that the value is kept alive until all child tasks created in such scope exit as well. This is important because child tasks may be refering to this specific value in the parent task, so it cannot be released earlier. 

Both value and reference types are allowed to be stored in task-local storage, using their expected respective semantics: 

- values stored as task-locals are copied into the task's local storage,
- references stored as task-locals are retained and stored by reference in the task's local storage.

Task local "item" storage allocations are performed using an efficient task local stack-discipline allocator, since it is known that those items can never out-live a task they are set on. This makes it slightly cheaper to allocate storage for values allocated this way than going through the global allocator, however task-local storage _should not_ be abused to avoid passing parameters explicitly, because it makes your code harder to reason about due to the "hidden argument" passing rather than plain old parameters in function calls.

Task-local items which are copied to a different task, i.e. when `async{}` launches a new unstructured task, have independent lifecycles and attach to the newly spawned task. This means that, at the point of creating a new task with `async{}`, reference-counted types stored within task-local storage may be retained.

### Reading task-local values

Task-local variables are semantically _inherited_ the same way by _child tasks_ similar to some other properties of a task, such as `priority`. 

This implies that stored values may be accessed from different tasks executing concurrently. In order to guarantee safety, task-local values must conform to the `Sendable` protocol, introduced in [SE-0302](0302-concurrent-value-and-concurrent-closures.md).

Accessing task-local values is synchronous and may be done from any context. If no task is available in the calling context, the default value for the task-local will be returned. The same default value is returned if the accessor is invoked from a context in which a task is present, however the task-local was never bound in this, or any of its parent tasks.

The specific lookup mechanism used by this accessor will be explained in detail in the next sections.

The example below explains the contextual awareness of task-local accessors when evaluated as a parameter for a synchronous function call (i.e. `print`):

```swift
func simple() async {
  print("number: \(Lib.number)")     // number: 0
  await Lib.$number.withValue(42) {
    print("number: \(Lib.number)")   // number: 42
  }
}
```

The same would work if the second `print` would be multiple asynchronous function calls "deeper" from the `withValue` invocation.

The same mechanism also works with tasks added in task groups or async let declarations, because those also construct child tasks, which then inherit the bound task-local values of the outer scope.

```swift
await Lib.$number.withValue(42) {
  
  await withTaskGroup(of: Int.self) { group in 
    group.addTask { 
      Lib.number // task group child-task sees the "42" value
    }
    return group.next()! // 42
  }
  
}
```

If a synchronous function is invoked from a context that was not running within a task, it will automatically return the `defaultValue` for given key — since there is no task available to read the value from. 

```swift
func simple() {
  print("number: \(Lib.number)")
}
```

Usually it doesn’t matter if the function was invoked without first binding the task-local value, or if the execution context is outside the Task runtime, as we can simply return the default value.

To check if the value was not bound albeit executing within a task, the following pattern can be used:

````swift
withUnsafeCurrentTask { task in 
  guard task != nil else { 
    return "<not executing within a task>" 
  }

  return Library.example // e.g. "example"
}
````

#### Reading task-local values: implementation details

There are two approaches possible to implement the necessary semantics. The naive approach being copying all task-local values to every created child task, which obviously creates a large overhead for "set once and then hundreds of tasks read the value" values. Because this is the usual access pattern for such values (e.g. request identifiers and similar), another approach is taken.

Since the implementation effectively already is a linked list of tasks, where children are able to look up their parent task, we reuse this mechanism to avoid copying values into child tasks. Instead, the _read_ implementation first checks for presence of the key in the current task, if not present, it performs a lookup in its parent, and so on, until no parent is available at which point the default value for the task-local key is returned:

```
[detached] ()
  \ 
  |[child-task-1] (id:10)
  |   \
  |   |[child-task-1-1] (id:20)
  |[child-task-2] (name: "alice")
```

Looking up `name` from `child-task-2` will return "alice" immediately, while looking up the same `name` from `child-task-1-1` will have to 1) check in the child task itself, 2) check in `child-task-1`, and finally check in `detached`, all of which returning empty. Looking up `id` from `child-task-1-1` will also return immediately and return `20`, which is what we'd expect — it is the "more specific" value deeper in the call chain.

We also notice that in many situations, the following chain will exist:

```
[detached] ()
  \ 
   [child-task-1] (requestID:10)
    \
    |[child-task-2] ()
     \
     |[child-task-3] ()
      \
      |[child-task-4] ()
```

Where many tasks can exist however they do not contribute any new task-local values to the chain. Thanks to task locals being immutable at task creation, we can guarantee that their known values never change, and thus we can optimize lookups from all tasks whose parent's do not contribute any additional task-local values. 

Specifically, at creation time of e.g. `child-task-3` we can notice that the parent (`child-task-2`) does not have any task-local values, and thus we can directly point at *its* parent instead: `child-task-1`, which indeed does contribute some values. More generally, the rule is expressed as pointing "up" to the first parent task that actually has any task-local values defined. Thanks to this, looking up `requestID` from `child-task-4` is only costing a single "hop" right into `child-task-1` which happens to define this key. If it didn't contain the key we were looking for, we would continue this search (including skipping empty tasks) until a detached task is reached.

This approach is highly optimized for the kinds of use-cases such values are used for. Specifically, the following assumptions are made about the access patterns to such values:

- **relatively few tasks read task-local values**
  - there usually is one "root task" which has the task-local information set, and hundreds or thousands of small child tasks (throughout the lifetime of the "root") which may or may not read the value,
  - _most_ child tasks do not read the task-local information; and even in tracing situations where potentially many tasks will read the value, this is only true in a fraction of the code's executions,
  - **conclusion**: it is not worth aggressively copying the values into all child tasks; taking a small performance hit during lookups is acceptable.
- **there may be many tasks 'between' the task binding the values, and those reading them**
  - quite often, values are set by a framework or runtime "once" before offering control flow to user code; usually none of the user-code adds any task-local values, but only uses the existing ones (e.g. in logging or tracing)
  - **conclusion**: the "skip task-local 'empty' tasks" optimization is worth it,
- **tasks should never have to worry about "racing" access to task-local values**
  - tasks must always be able to call `Lib.myValue` and get predictable values back; specifically, this means that a task _must not_ be able to mutate its task-local values — because child tasks run concurrently with it, this would mean that a child task invoking `Lib.myValue` twice, could get conflicting results, leading to a confusing programming model
  - **conclusion**: task-local storage must be initialized at task creation time and cannot be mutated, values may only be "bound" by creating new scopes/tasks.

> Note: This approach is similar to how Go's `Context` objects work -- they also cannot be mutated, but only `With(...)` copied, however the copies actually form a chain of contexts, all pointing to their parent context. In Swift, we simply reuse the Concurrency model's inherent `Task` abstraction to implement this pattern.

#### Task-locals in contexts where no Task is available

Task-locals are also able to function in contexts where no Task is available, they function just as a "dynamic scope" and simply utilize a single thread-local variable to store the task-local storage, rather than forming the chain of storages as is the case when tasks are available.

The only context in which a Task is not available to the implementation are synchronous functions that were called from the outside of Swift Concurrency. These functions are very rare but can happen, e.g. when a callback is invoked by a C library on some thread that it managed itself. 

The following API continues to work as expected in those situations:

```swift
func synchronous() {
  withUnsafeCurrentTask { task in 
    assert(task == nil) // no task is available!
  }
  
  Example.$local.withValue(13) { 
    other()
  }
}

func other() {
  print(Example.local) // 13, works as expected
}
```

The only Swift Concurrency API that can be called from such synchronous functions and _will_ inherit task-local values is `async{}`, and it will copy any task-local values encountered as usual. This makes for a good interoperability story even with legacy libraries -- we never have to worry if we are on a thread owned by the Swift Concurrency runtime or not, and things continue to work as expected.

#### Child task and value lifetimes

It is also important to note that no additional synchronization is needed on the internal implementation of the task-local value stack / linked-list. This is because we strongly rely on guarantees of structured concurrency. Specifically, we exploit the guarantee that:

> *By the time the scope exits, the child task must either have completed, or it will be implicitly awaited.* When the scope exits via a thrown error, the child task will be implicitly cancelled before it is awaited.

Thanks to this guarantee child tasks may directly point at the head of the stack of their parent (or super-parent), and we need not implement any additional house-keeping for those references. We know that the parent task will always have values we pointed to from child tasks (defined within a `withValue` body) present, and the child tasks are guaranteed to complete before the `withValue` returns. We use this to automatically pop bound values from the value stack as we return from the `withValue` function, this is guaranteed to be safe, since by that time, all child tasks must have completed and no-one will refer to the task-local values at that point anymore.

##### Task-local value item allocations

It is worth calling out that the `withValue(_:) { ... }` API style enables crucial performance optimizations for internal storage of those tasks.

Since the lifetime of values is bounded by the scope of a `withValue` function along with guarantees made by structured concurrency with reference to parent/child task lifetimes, we are able to use task-local allocation mechanisms, which avoid using the system allocator directly and can be vastly more efficient than global allocation (e.g. malloc).

### Similarities and differences with SwiftUI's `Environment`

Readers may be aware of SwiftUI's type [SwiftUI's Environment](https://developer.apple.com/documentation/swiftui/environment) which seemingly has a very similar purpose, however it is more focused on the view hierarchies, rather than "flow of a value _through_ asynchronous calls" which this API is focused on.

One may think about the difference how these APIs differ in terms of where the "parent/child" relationship is represented. 

SwiftUI's environment considers relationships between views, while task-local values are about the relationship of asynchronous tasks. So while the general idea is similar, the actual semantics are quite different. It is best to visualize task-local values as "following" the execution path, regardless where (in which specific asynchronous function or actor) that execution takes place.

Swift UI's `@Environment` can be used to define and store custom values, like so:

```swift
struct Kitchen {
  @Environment(\.oven) var oven: Oven
}
```

where keys are defined as:

```swift
public protocol EnvironmentKey {
  associatedtype Value
  static var defaultValue: Self.Value { get }
}
```

and can be implemented as:

```swift
struct OvenKey: EnvironmentKey {
  static let defaultValue: Oven = DefaultOven()
}
extension EnvironmentValues {
  var oven: Oven {
    get {
      return self[OvenKey.self]
    }
    set {
      self[OvenKey.self] = newValue
    }
  }
}
```

Keeping the `OvenKey` `internal` or even `private` allows for fine grained control over who can set or read this value. 


This API as well as the Swift Distributed Tracing `Baggage` type all adopt the same style and should be used in the same way to set custom keys. However it is NOT the primary purpose of task-local values to help create values — it is to use them _during_ execution of asynchronous functions.

In other words:

- **SwiftUI's `@Environment`** is useful for structurally configuring views etc.
- **Task-Local Values** are useful for _carrying_ metadata along through a series of asynchronous calls, where each call may want to access it, and the context is likely different for every single "incoming request" even while the structure of the system remains the same.

## Prior Art

### Kotlin: CoroutineContext[T]

Kotlin offers an explicit API to interact with the coroutine "scope" and "context", these abstractions are very similar to Swift's `Task` abstraction. 

An explicit [`CoroutineContext`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/) API is offered to read the context from anywhere it can be accessed. It is semantically equivalent to `[CoroutineContext.Key<...>: CoroutineContext.Element]`, so again, very similar to what we discussed above.

Usage typically is as follows:

```kotlin
println("Running in ${coroutineContext[CoroutineName]}")
```

where `CoroutineName` is a `Key`, and when executed in a coroutine this yields the expected name.

Setting a context again can only be done by nesting and scopes, as follows:

```kotlin
suspend fun <T> withContext(
  context: CoroutineContext, 
  block: suspend CoroutineScope.() -> T
): T
```

used like this:

```kotlin
withContext(Dispatchers.IO) {
  // IO dispatcher context variable is in effect here
}
// IO dispatcher context variable is no longer set
```

which allows adding context variables while the `block` executes.

See also [Structured concurrency, lifecycle and coroutine parent-child hierarchy](https://github.com/Kotlin/kotlinx.coroutines/blob/master/ui/coroutines-guide-ui.md#structured-concurrency-lifecycle-and-coroutine-parent-child-hierarchy).

### Java/Loom: Scope Variables

Java, with it's coroutine and green-thread based re-thinking of the JVM's execution model, is experimenting with introducing "[*Scope Variables*](https://cr.openjdk.java.net/~rpressler/loom/loom/sol1_part2.html#scope-variables)" which address the same known pain-points of thread-local variables.

Java's Loom-based concurrency does not expose coroutines or any new concepts into the language (nor does it have async/await or function coloring, because of the use of green threads).

Snippet explaining their functioning:

> ```java
> static final Scoped<String> sv = Scoped.forType(String.class);
> 
> void foo() {
>   try (var __ = sv.bind("A")) {
>     bar();
>     baz();
>     bar();
>   }
> }
> 
> void bar() {
>   System.out.println(sv.get());
> }
> 
> void baz() {
>   try (var __ = sv.bind("B")) {
>     bar();
>   }
> }
> ```
> 
> `baz` does not mutate `sv`’s binding but, rather introduces a new binding in a nested scope that shadows its enclosing binding. So foo will print:
> 
> ```
> A
> B
> A
> ```

This again is very similar to task-local variables, however it expresses it as an actual variable through the access must be performed.

### Go: explicit context passing all the way

Go's take on asynchronous context propagation takes the form of the [`Context`](https://golang.org/pkg/context/) type which is part of the standard library. The code of the library boils down to:

```go
// A Context carries a deadline, cancelation signal, and request-scoped values
// across API boundaries. Its methods are safe for simultaneous use by multiple
// goroutines.
type Context interface {
  // Done returns a channel that is closed when this Context is canceled
  // or times out.
  Done() <-chan struct{}

  // Err indicates why this context was canceled, after the Done channel
  // is closed.
  Err() error

  // Deadline returns the time when this Context will be canceled, if any.
  Deadline() (deadline time.Time, ok bool)

  // Value returns the value associated with key or nil if none.
  Value(key interface{}) interface{}
}

var (
  background = new(emptyCtx)
  todo       = new(emptyCtx)
)
```

Go's `Context` is used for cancellation and deadline propagation as well as other values propagation, it is the _one_ bag for extra values that gets passed explicitly to all functions.

Notice though that context variables are not typed, the Value returns an `interface{}` (which is like `Any` in Swift). Otherwise though, the general shape is very similar to what Swift is offering.

The Go programming style is very strict about Context usage, meaning that _every, function that meaningfully can,_ **must** accept context parameter as its first parameter:

```go
func DoSomething(ctx context.Context, arg Arg) error {
  // ... use ctx ...
}
```

This results in the context being _everywhere_. Programmers learn to visually ignore the noise and live with it.

Contexts are immutable, and modifying them is performed by making a new context:

```go
func WithValue(parent Context, key interface{}, val interface{}) Context
```

The implementation is able to form a chain of contexts, such that each context points "back" to its parent forming a chain that is walked when we resolve a value by key.

This blog post is fairly informative on how this is used in the real world: [
Go Concurrency Patterns: Context](https://blog.golang.org/context).

## Alternatives Considered

### Surface API: Type-based key definitions

The initially pitched approach to define task-local keys was impossible to get wrong thanks to the type always being unique. However declaring and using the keys was deemed too tiresome by the community during review, thus the proposal currently is pitching `@TaskLocal` property wrapper.

The previous design required this boilerplate to declare a key:

```swift
extension TaskLocalValues {
  
  public struct RequestIDKey: TaskLocalKey {
    // alternatively, one may declare a nil default value:
    //     public static var defaultValue: String? { nil } 
    public static var defaultValue: String { "<no-request-id>" }
      
    // additional options here, like e.g.
    // static var inherit: TaskLocalValueInheritance = . never
  }
    
  public var requestID: RequestIDKey { .init() }
  
}
```

and usage would look like this:

```swift
await Task.withLocal(\.requestID, boundTo: "abcd") { 
  _ = Task.local(\.requestID) // "abcd"
}
```

It was argued that the declaration is too boilerplate heavy and thus discarded and we moved towards the property wrapper based API.

### Surface API: Key-less value definitions

Stefano De Carolis proposed on the forums to simplify the definition sites to be:

```swift
extension Task.Local {
  var foo: String { "Swift" }
}
```

Our concerns about this shape of API are: 

- it prioritizes briefity and not clarity. It is not clear that the value returned by the computed property `foo` is the default value. And there isn't a good place to hint at this. In the `...Key` proposal we have plenty room to define a function `static var defaultValue` which developers need to implement, immediately explaining what this does.
- this shape of API means that we would need to actively invoke the key-path in order to obtain the value stored in it. We are concerned about the performance impact of having to invoke the key-path rather than invoke a static function on a key, however we would need to benchmark this to be sure about the performance impact.
- it makes it harder future extension, if we needed to allow special flags for some keys. Granted, we currently do not have an use-case for this, but with Key types it is trivial to add special "do not inherit" or "force a copy" or similar behaviors for specific keys. It is currently not planned to implement any such modifiers though.

For completeness, the functions to read and bind values with this proposal would become:

```swift
enum Task {
  enum Local {}
  
  static func withLocal<Value, R>(
    _ path: KeyPath<Local, Value>,
    boundTo value: Value,
    body: @escaping () async -> R
  ) async -> R { ... }
  
  static func local<Value>(
    _ path: KeyPath<Local, Value>
  ) async -> Value { ... }
}
```

## Rejected Alternatives

### Plain-old Thread-Local variables

Thread-local storage _cannot_ work effectively with Swift's concurrency model.

Swift's concurrency model deliberately abstains from using the thread terminology, because _no guarantees_ are made about specific threads where asynchronous functions are executed. Instead, guarantees are phrased in terms of Tasks and Executors (i.e. a thread pool, event loop or dispatch queue actually running the task). 

In other words: Thread locals cannot effectively work in Swift's concurrency model, because the model does not give _any_ guarantees about specific threads it will use for operations.

We also specifically are addressing pain points of thread-locals with this proposal, as it is far too easy to make these mistakes with thread-local values:

- it is hard to use thread locals in highly asynchronous code, e.g. relying on event loops or queue-hopping, because on every such queue hop the library or end-user must remember to copy and restore the values onto the thread which later-on is woken up to resume the work (i.e. in a callback),
- it is possible to "leak" values into thread locals, i.e. forgetting to clean up a thread-local value before returning it to a thread pool, may result in:
   - a value never being released leading to memory leaks, 
   - or leading to new workloads accidentally picking up values previously set for other workloads;
- thread locals are not "inherited" so it is difficult to implement APIs which "carry all thread-local values to the underlying worker thread" or even jump to another worker thread. All involved libraries must be aware of the involved thread locals and copy them to the new thread — which is both inefficient, and error prone (easy to forget).

None of those issues are possible with task-local values, because they are inherently scoped and cannot outlive the task with which they are associated.

| **Issue**           | **Thread-Local Variables** | **Task-Local Values** |
|---------------------|-----------------------------|-----------------------|
| "Leaking" values    | Possible to forget to "unset" a value as a scope ends. | Impossible by construction; scopes are enforced on the API level, and are similar to scoping rules of async let and Task Groups. |
| Reasoning | Unstructured, no structural hints about when a variable is expected to be set, reset etc. | Simpler to reason about, follows the child-task semantics as embraced by Swift with `async let` and Task Groups. |
| Carrier type | Attached specific **threads**; difficult to work with in highly asynchronous APIs (such as async/await). | Attached to specific tasks, accessible through a task's child tasks as well, forming a hierarchy of values, which may be used to provide more specific values for children deeper in the hierarchy.
| Mutation | Thread locals may be mutated by *anyone*; A caller cannot assume that a function it called did not modify the thread-local that it has just set, and may need to be defensive about asserting this. | Task locals cannot modify their parent's values; They can only "modify" values by immutably adding new bindings in their own task; They cannot change values "in" their parent tasks. |

### Dispatch Queue Specific Values

Dispatch offers APIs that allow setting values that are _specific to a dispatch queue_: 
- [`DispatchQueue.setSpecific(key:value:)`](https://developer.apple.com/documentation/dispatch/dispatchqueue/2883699-setspecific) 
- [`DispatchQueue.getSpecific(key:)`](https://developer.apple.com/documentation/dispatch/dispatchqueue/1780751-getspecific).

These APIs serve their purpose well, however they are incompatible with Swift Concurrency's task-focused model. Even if actors and asynchronous functions execute on dispatch queues, no capability to carry values over multiple queues is given, which is necessary to work well with Swift Concurrency, as execution may hop back and forth between queues.

## Intended use-cases
It is important to keep in mind the intended use case of this API. Task-local values are not intended to replace passing parameters where doing so explicitly is the right tool for the job. Please note that task local storage is more expensive to access than parameters passed explicitly. They also are "invisible" in API, so take care to avoid accidentally building APIs which absolutely must have some task local value set when they are called as this is very suprising and hard to debug behavior.

Only use task local storage for auxiliary _metadata_ or "_execution scoped configuration_", like mocking out some runtime bits for the duration of a _specific call_ but not globally, etc.

### Use case: Distributed Tracing & Contextual Logging

> This section refers to [Apple/Swift-Distributed-Tracing](https://github.com/apple/swift-distributed-tracing)

Building complex server side systems is hard, especially as they are highly concurrent (serving many thousands of users concurrently) and distributed (spanning multiple services and nodes). Visibility into such system–i.e. if it is performing well, or lagging behind, dropping requests, or for some kinds of requests experiencing failures–is crucial for their successful deployment and operation.

#### Contextual Logging

Developers instrument their server side systems using logging, metrics and distributed tracing to gain some insight into how such systems are performing. Improving such observability of back-end systems is crucial to their success, yet also very tedious to manually propagate the context.

Today developers must pass context explicitly, and with enough cooperation of libraries it is possible to make this process relatively less painful, however it adds a large amount of noise to the already noisy asynchronous functions:

```swift
func chopVegetables(context: LoggingContext) async throws -> [Vegetable] { 
  context.log.info("\(#function)")
}
func marinateMeat(context: LoggingContext) async -> Meat { 
  context.log.info("\(#function)")
}
func preheatOven(temperature: Double, context: LoggingContext) async throws -> Oven { 
  context.log.info("\(#function)")
}

// ...

func makeDinner(context: LoggingContext) async throws -> Meal {
  context.log.info("\(#function)")
  
  async let veggies = chopVegetables(context: context)
  async let meat = marinateMeat(context: context)
  async let oven = preheatOven(temperature: 350, context: context)

  let dish = Dish(ingredients: try await [veggies, meat])
  return try await oven.cook(dish, duration: .hours(3), context: context)
}
```

Thanks to the passed `LoggingContext` implementations may be invoked with a specific `"dinner-request-id"` and even if we are preparing multiple dinners in parallel, we know "which dinner" a specific operation belongs to:

```swift
var context: LoggingContext = ...

context.baggage.dinnerID = "1234"
async let first = makeDinner(context: context)

context.baggage.dinnerID = "5678"
async let second = makeDinner(context: context)

await first
await second
```

Resulting in logs like this:

```
<timestamp> dinner-id=1234 makeDinner
<timestamp> dinner-id=1234 chopVegetables
  <timestamp> dinner-id=5678 makeDinner
  <timestamp> dinner-id=5678 chopVegetables
<timestamp> dinner-id=1234 marinateMeat
<timestamp> dinner-id=1234 preheatOven
  <timestamp> dinner-id=5678 marinateMeat
<timestamp> dinner-id=1234 cook
  <timestamp> dinner-id=5678 preheatOven
  <timestamp> dinner-id=5678 cook
```

Allowing developers to track down the request specific logs by filtering logs by the `dinner-id` that may be encountering some slowness, or other issues.

#### Function Tracing

We do not stop there however, by instrumenting all functions with swift-tracing, we can obtain a full trace of the execution (in production), and analyze it later on using tracing systems such as [zipkin](https://zipkin.io), [jaeger](https://www.jaegertracing.io/docs/1.20/#trace-detail-view),  [Grafana](https://grafana.com/blog/2020/11/09/trace-discovery-in-grafana-tempo-using-prometheus-exemplars-loki-2.0-queries-and-more/), or others.

By instrumenting all functions with tracing:

```swift
// ... 

func chopVegetables(context: LoggingContext) async throws -> [Vegetable] { 
  let span = InstrumentationSystem.tracer.startSpan(#function, context: context)
  defer { span.end() }
  ...
}

// ... 
```

we are able to obtain visualizations of the asynchronous computation similar to the diagram shown below. The specific visualization depends on the tracing system used, but generally it yields such trace that can be inspected offline using server side trace visualization systems (such as Zipkin, Jaeger, Honeycomb, etc):

```
>-o-o-o----- makeDinner ----------------o---------------x         [15s]
  \-|-|- chopVegetables--------x        |                 [2s]
    | |  \- chop -x |                   |              [1s]
    | |             \--- chop -x        |                 [1s]
    \-|- marinateMeat -----------x      |                  [3s]
      \- preheatOven -----------------x |                 [10s]
                                        \--cook---------x      [5s]
```
* diagram only for illustration purposes, generally this is displayed using fancy graphics and charts in tracing UIs.

Such diagrams allow developers to naturally "spot" and profile the parallel execution of their code. Values declared as `async let` introduce concurrency to the program's execution, which can be visualized using such diagrams and also easily spot which operations dominate the execution time and need to be sped up or perhaps parallelized more. In the above example, we notice that since `preheatOven` takes 10 seconds in any case, even if we sped up `chopVegetables` we will _not_ have sped up the entire `makeDinner` task because it is dominated by the preheating. If we were able to optimize the `cook()` function though we could shave off 5 seconds off our dinner preparation!

#### Distributed Tracing

So far this is on-par with an always on "profiler" that is sampling a production service, however it does only sample a single node — all the code is on the same machine... 

The most exciting bit about distributed tracing is that the same trace graphs can automatically be produced even _across libraries_ and across nodes in a _distributed system_. Thanks to HTTP Clients, Servers and RPC systems being aware of the metadata carried by asynchronous tasks, we are able to carry tracing beyond single-nodes, and easily trace distributed systems.

> For in depth details about this subject, please refer to [Swift Distribted Tracing](https://github.com/apple/swift-distributed-tracing).

If, for whatever reason, we had to extract `chopVegetables()` into a _separate (web) service_, the exact same code can be written — and if the networking library used to make calls to this *"ChoppingService"* are made, the trace is automatically propagated to the remote node and the full trace now will include spans from multiple machines (!). To visualize this we can show this as:

```
>-o-o-o----- makeDinner ----------------o---------------x      [15s]
  | | |                     | |         |                  
~~~~~~~~~ ChoppingService ~~|~|~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
  \-|-|- chopVegetables-----x |                            [2s]     \
    | |  \- chop -x |         |                        [1s]         | Executed on different host (!)
    | |             \- chop --x                        [1s]         /
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    \-|- marinateMeat -----------x      |                  [3s]
      \- preheatOven -----------------x |                 [10s]
                                        \--cook---------x  [5s]
```

Thanks to swift-tracing and automatic context propagation we can make tracing much simpler to adopt, and reap the benefits from it; Making complex server side systems (almost) as simple to debug as local single process concurrent applications.


#### Future direction: Function wrapper interaction

> **CAVEAT**: This proposal does not imply/promise any future work on function wrappers, however **if** they were proposed and accepted at some point, this would be a natural use-case for them.

If Swift were to get "function wrappers", tracing a set of asynchronous functions becomes trivial:

```swift
@Traced func chopVegetables() async throws -> [Vegetable] { ... }
@Traced func marinateMeat() async -> Meat { ... }
@Traced func preheatOven(temperature: Double) async throws -> Oven { ... }

// ...

@Traced
func makeDinner() async throws -> Meal {
  async let veggies = try await chopVegetables()
  async let meat = await marinateMeat()
  async let oven = try await preheatOven(temperature: 350)

  let dish = Dish(ingredients: await [veggies, meat])
  return try await oven.cook(dish, duration: .hours(3))
}
```

Which would automatically *start* a tracing `Span` with an *operation name* "makePizza" and *end* it when the function returns, saving developers multiple layers of nesting and directly interacting with the Tracer API, and making Swift a truly great citizen among observability-first languages and best-in class for distributed systems programming in general.

This way, eventually, we would have gained all benefits of (distributed) tracing and contextual logging, without any of the noise and complexity usually associated with it.

### Use case: Mocking internals (Swift System)

Some libraries may offer a special API that allows switching e.g. filesystem access to a "mock filesystem" if it is present in a task local value.

This way developers could configure tasks used in their tests to bind a "mock filesystem" under a known to the underlying library task local value, and this way avoid writing/reading from a real filesystem in tests, achieving greater test isolation without having to pass a specific `Filesystem` instance through all API calls of the library.

This pattern exists today in [Swift System](https://github.com/apple/swift-system) where the [withMockingEnabled](https://github.com/apple/swift-system/pull/8/files#diff-9e369bd109521aa185f8c63d962d415e58f03f6d8e80c3abd5e544511937452dR115-R128) function is used to set a thread local which changes how functions execute (and allows them to be traced). The mechanism used there, and in similar frameworks, _will not_ work in the future as Swift adopts `async` and `Swift System` itself would want to adopt async functions, since they are a prime candidate to suspend a task while a write is being handled asynchronously (e.g. if one were to implement APIs using `io_uring` or similar mechanisms). Task Local Values enable Swift System to keep it's mocking patterns working and efficient in the face of asynchronous functions.

### Use case: Progress Monitoring

In interactive applications asynchronous tasks frequently are linked with some progress indicator such that a user waiting for the task knows that it indeed is proceeding, and not just "stuck" on a never-ending "Loading..."-screen.

Foundation offers the [Progress](https://developer.apple.com/documentation/foundation/progress) type which is used with UI frameworks, such as [SwiftUI](https://developer.apple.com/xcode/swiftui/), to easily report back progress of tasks back to users. Currently, `Progress` can be used by either passing it manually and explicitly, or accessing it through thread-local storage. 

`Progress` naturally has it's own child-progress semantics which exactly mirror how the compiler enforces child task relationships — child tasks contribute to the task's progress after all. Using task local values we could provide a nice API for progress monitoring that naturally works with tasks and child tasks, without causing noise in the APIs, and also avoiding the issues of thread-local style APIs which are notoriously difficult to use correctly.

### Use case: Executor configuration

A frequent requirement developers have voiced is to have some control and configurability over executor details on which tasks are launched.

By using task locals we have a mechanism that flows naturally with the language, and due to inheritance of values also allows to automatically set up the preferred executor for tasks which do not have a preference.
For example, invoking such actor-independent functions `calcFoo` and `calcBar` could be scheduled on specific executors (or perhaps, allow configuring executor settings) by setting a task local value like this:

```swift
// Just ideas, not actual API proposal (!)
async let foo = $myExecutor.withValue(someSpecificExecutor) {
  calcFoo()
}
```

## Future Directions

### Additional configuration options for `@TaskLocal`

In our current work we discovered a number of special keys which we will be introducing in the future, e.g. to support operating system requirements for tracing calls, authentication or support for novel patterns such as a Swift Concurrency aware `Progress` type. 

Some of those keys will want to make different performance tradeoffs. For example, tracing IDs may want to require being propagated in an in-line storage and copied every time to a child task upon `spawn` rather than being lazily accessed on each read operation. Or certain keys may wish to propagate to child tasks only when called explicitly, so a "don't inherit" propagation policy could be used.

These configuration options are able to be introduced in binary and source compatible ways to the property wrapper and backing storage. The storage requirements for those flags are minimal, and such flags will only ever be created once per specific task-local key. 

### Tracing annotations with Function Wrappers

As discussed in the tracing use-case section, the ability to express `@Logged` or `@Traced` as annotations on existing functions to easily log and trace function invocations is definitely something various people have signalled a strong interest in. And this feature naturally enables the implementation of those features.

Such annotations depend on the arrival of [Function Wrappers](https://forums.swift.org/t/prepitch-function-wrappers/33618) or a similar feature to them, which currently are not being actively worked on, however we definitely have in the back of our minds while designing this proposal.

### Language features to avoid nesting with `withValue`

It is necessary for task-local correctness to only bind values for a given scope. 

This scoping rule is enforced by the only API to bind a task-local value being the `withValue() { ... }` function. The function essentially does two operations: 

- pushes a new binding onto the task-local bindings stack in the current task
- (executes the user provided `body` closure)
- and pops the binding from the task-local bindings stack of the current task

This structure allows us to use task-local allocation safely, and also ensure that no lingering values ever "leak" the scope where they were defined.

It is, however, slightly cumbersome on a source level to have to indent code only in order to get this property when in reality most of the time a binding is going to be set for the entirety (or remaining part) of the current function.

If Swift were to gain some "`using`" mechanism, that would encapsulate the pattern of doing one "start" operation now and an "end" operation at scope exit. That would be a very general feature with broad applicability. The same mechanism could be used by task-locals rather than resorting to nesting. 

```swift
scoped_overwrite requestID = id   // <- intentionally terrible syntax
```

which by default would desugar as:

```swift
let old = <variable>
<variable> = new
defer { <variable> = old }
```

but which property wrappers would have some ability to customize. Such feature may allow expressing the pattern necessary for task-local binding correctness without the cumbersome nesting.

### Specialized TaskLocal Value Inheritance Semantics

Some task local values may require specialized inheritance semantics. The default strategy simply means that child tasks "inherit" values from their parents. At runtime, this is not achieved by copying, but simply performing lookups through parent tasks as well, when a `TaskLocalInheritance.default` inherited key is being looked up.

Some, specialized use-cases however can declare more specific inheritance semantics. It is *not* encouraged to use these specialized semantics nonchalantly, and their use should always be carefully considered and given much thought as they can lead to unexpected behaviors otherwise.

A `TaskLocal` type may declare an inheritance semantics by defining the static `inherit` parameter when declaring the variable: `TaskLocal(inherit: .never)`.

The semantics default to `.default`, which are what one would expect normally — that child tasks are able to lookup values defined in their parents, unless overriden in that specific child. We will discuss the exact semantics of lookups in depth in [Reading task-local values](#reading-task-local-values).

In this proposal, we introduce two additional inheritance semantics: `.never` and `.alwaysBestEffort`:

```swift
/// Allows configuring specialized inheritance strategies for task local values.
///
/// By default, task local values are accessible by the current or any of its
/// child tasks (with this rule applying recursively).
///
/// Some, rare yet important, use-cases may require specialized inheritance
/// strategies, and this property allows them to configure these for their keys.
public enum TaskLocalInheritance: UInt8, Equatable {
  case `default`        = 0
  case never            = 1
  case alwaysBestEffort = 2
}
```

Note that `TaskLocalInheritance` should remain extensible.

Both these semantics are driven by specific use cases from the Swift ecosystem, highlighted during early design reviews of this proposal. First, the `.never` inheritance model allows for the design of a highly specialized task-aware `Progress` type, that is not part of this proposal. And second various Tracer implementations, including swift-distributed-tracing but also Instruments which will want to use special tracing metadata which should be carried "always" (at a best effort), even through detached tasks.

#### "Never" task-local value inheritance

The "never" inheritance semantics allow a task to set "truly local only to this specific task" values. I.e. if a parent task sets some value using an non-inherited key, it's children will not be able to read it.

It is simplest to explain those semantics with an example, so let us do just that. First we define a key that uses the `.never` inheritance semantics. We could, for example, declare a `House?` task-local and make sure it will not be inherited by our children (child tasks):

```swift
struct House {

  @TaskLocal(inherit: .never)
  static var key: House?

}
```

This way, only the current task which has bound this task local value to itself can access the house key. This key remains available throughout the entire `withValue`'s scope. However none of its child tasks, spawned either by async let, or task groups will inherit the `house`:

```swift
House.$key.withValue(House(...)) {
  async let child = assert(House.key == nil) // not available in child task
}
```

Addmitably, this is a fairly silly example, and in this small limited example it is trivial to replace this task local with a plain variable. Or rather, it *should* be replaced with a plain variable and not abuse task locals for this.

We have specific designs in mind with regards to `Progress` monitoring types however which will greatly benefit from these semantics. The `Progress` API will be it's own swift evolution proposal however, so we do not dive much deeper into it's API design in this proposal. Please look forward to upcoming proposals with regards to monitoring

## Revision history

- v5: Allow usage even in contexts where no Task is available
  - Fallback to additional thread-local storage if no Task is available to bind/get task local values from,
  - remove API on UnsafeCurrentTask, its use-cases are now addressed by the core `withValue(_:)` API,
- v4.5: Drop the Access type and use the `projectedValue` to simplify read and declaration sites.
- v4: Changed surface API to be focused around `@TaskLocal` property wrapper-style key definitions.
  - introduce API to bind task-local values in synchronous functions, through `UnsafeCurrentTask`
  - allude to `async` (or `send`) as the way to carry task-local values rather than forcing them into a detached task
  - explain an anti-pattern that will be detected and cause a crash if used around wrapping a `group.addTask` with a task local binding. Thank you to @Lantua over on the Swift Forums for noticing this specific issue.
- v3.2: Cleanups as the proposal used outdated wordings and references to proposals that since have either changed or been accepted already. 
  - No semantic changes in any of the mechanisms proposed.
  - Change mentions of `ConcurrentValue` to `Sendable` as it was since revised and accepted.
- v3.1: Move specialized task semantics to future directions section
  - Will adjust implementation to not offer the "do not inherit" mode when accepted
- v3: Prepare for review
  - polish wording and API names of "task-local value inheritance" related functions and wording,
  - discuss detached tasks and runDetached with inheritance,
  - explain the use of task-local allocation as a core idea to those task local items.
- v2: Thanks to the introduction of `Task.unsafeCurrent` in Structured Concurrency, we're able to amend this proposal to:
  - allow access to task-locals from *synchronous* functions, 
  - link to the [ConcurrentValue](https://forums.swift.org/t/pitch-3-concurrentvalue-and-concurrent-closures/43947) proposal and suggest it would be used to restrict what kinds of values may be stored inside task locals.
  - introduce specialized limited storage for specialized trace keys, to be carried even through detached tasks.
  - rewordings and clarifications.
- v1: Initial draft

## Source compatibility

This change is purely additive to the source language. 

## Effect on ABI stability

This proposal is additive in nature.

It adds one additional pointer for implementing the task local value stack in `AsyncTask`.

## Effect on API resilience

No impact. 
