# Clarify the Swift memory consistency model ⚛︎

* Proposal: [SE-0282](0282-atomics.md)
* Author: [Karoy Lorentey](https://github.com/lorentey)
* Review Manager: [Joe Groff](https://github.com/jckarter)
* Bug: [SR-9144](https://bugs.swift.org/browse/SR-9144)
* Implementation: Proof of concept [swift-atomics package][package]
* Previous Revision: [v1][SE-0282v1] ([Returned for revision](https://forums.swift.org/t/se-0282-low-level-atomic-operations/35382/69))
* Status: **Implemented (Swift 5.3)**
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-se-0282-interoperability-with-the-c-atomic-operations-library/38050)

[SE-0282v1]: https://github.com/swiftlang/swift-evolution/blob/3a358a07e878a58bec256639d2beb48461fc3177/proposals/0282-atomics.md
[package]: https://github.com/apple/swift-atomics

## Introduction

This proposal adopts a C/C++-style weak concurrency memory model in Swift, describing how Swift code interoperates with concurrency primitives imported from C.

This enables intrepid library authors to start building concurrency constructs in (mostly) pure Swift.

Original swift-evolution thread: [Low-Level Atomic Operations](https://forums.swift.org/t/low-level-atomic-operations/34683)

## Revision History

- 2020-04-13: Initial proposal version.
- 2020-06-05: First revision.
    - Removed all new APIs; the proposal is now focused solely on C interoperability.

## Table of Contents

  * [Motivation](#motivation)
  * [Proposed Solution](#proposed-solution)
    * [Amendment to The Law of Exclusivity](#amendment-to-the-law-of-exclusivity)
  * [Considerations for Library Authors](#considerations-for-library-authors)
    * [Interaction with Non\-Instantaneous Accesses](#interaction-with-non-instantaneous-accesses)
    * [Interaction with Implicit Pointer Conversions](#interaction-with-implicit-pointer-conversions)
  * [Source Compatibility](#source-compatibility)
  * [Effect on ABI Stability](#effect-on-abi-stability)
  * [Effect on API Resilience](#effect-on-api-resilience)
  * [Alternatives Considered](#alternatives-considered)
  * [References](#references)

## Motivation

In Swift today, application developers use dispatch queues and Foundation's NSLocking protocol to synchronize access to mutable state across concurrent threads of execution.

However, for Swift to be successful as a systems programming language, it needs to also be possible to implement such synchronization constructs (and many more!) directly within Swift. To allow this, we need to start describing a concurrency memory model.

Given how deeply Swift interoperates with C, it seems reasonable to assume that Swift's concurrency memory model is compatible with the one described in the C standard. In fact, given that a large amount of existing Swift code deeply depends on concurrency constructs imported from C (most prominently the Dispatch library, but also POSIX Threads, `stdatomic.h` and others), fully embracing interoperability is very likely to be the only practical choice. Therefore, this proposal does exactly that -- it describes how C's atomic operations and memory orderings interact with Swift code.

Having a reasonably well-defined meaning for the low-level atomic constructs defined for the C (and C++) memory model is crucial for people who wish to implement synchronization constructs or concurrent data structures directly in Swift. (Note that this is a hazardous area that is full of pitfalls. We expect that the higher-level synchronization tools that can be built on top of these atomic primitives will provide a nicer abstraction layer.)

Note that while this proposal doesn't include a high-level concurrency design for Swift, it also doesn't preclude the eventual addition of one. Indeed, we expect that embracing a compatible concurrency memory model will serve as an important step towards language-level concurrency, by making it easier for motivated people to explore the design space on a library level.

## Proposed Solution

We propose to adopt a C/C++-style concurrency memory model for Swift code:

* Concurrent write/write or read/write access to the same location in memory generally remains undefined/illegal behavior, unless all such access is done through a special set of primitive *atomic operations*.

* The same atomic operations can also apply *memory ordering* constraints that establish strict before/after relationships for accesses across multiple threads of execution. Such constraints can also be established by explicit *memory fences* that aren't tied to a particular atomic operation.

This document does not define a formal concurrency memory model in Swift, although we believe the methodology and tooling introduced for the C and C++ memory model and other languages could be adapted to work for Swift, too [[C18], [C++17], [Boehm 2008], [Batty 2011], [Nienhuis 2016], [Mattarei 2018]]. This proposal also doesn't come with any native concurrency primitives; it merely describes how C's preexisting constructs (`atomic_load_explicit`, `atomic_thread_fence`, etc.) can be used to synchronize Swift code.

When applied carefully, atomic operations and memory ordering constraints can be used to implement higher-level synchronization algorithms that guarantee well-defined behavior for arbitrary variable accesses across multiple threads, by strictly confining their effects into some sequential timeline.

For now, we will be heavily relying on the Law of Exclusivity as defined in [[SE-0176]] and the [[Ownership Manifesto]], and we'll explain to what extent C's memory orderings apply to Swift's variable accesses. The intention is that Swift's memory model will be fully interoperable with its C/C++ counterparts.

This proposal does not specify whether/how dependency chains arising from the C/C++ `memory_order_consume` memory ordering work in Swift. The consume ordering as specified in the C/C++ standards is not implemented in any C/C++ compiler, and we join the current version of the C++ standard in encouraging Swift programmers not to use it. We expect to tackle the problem of efficient traversal of concurrent data structures in future proposals. Meanwhile, Swift programmers can start building useful concurrency constructs using relaxed, acquire/release, and sequentially consistent memory orderings imported from C.


### Amendment to The Law of Exclusivity

While the declarations in C's `stdatomic.h` header don't directly import into Swift, it is still possible to access these constructs from Swift code by [wrapping them into plain structs and functions][package] that can be imported. This way, `_Atomic` values can end up being stored within a Swift variable. 

When Swift code is able to acquire a stable pointer to the storage location of such a variable (by e.g. manually allocating it), it ought to be possible to pass this pointer to C's atomic functions to perform atomic operations on its value. Because C's atomic operations (`atomic_load`, `atomic_store`, `atomic_compare_exchange`, etc.) are inherently safe to execute concurrently, we must make sure that the Law of Exclusivity won't disallow them.

While [[SE-0176]] didn't introduce any active enforcement of the Law of Exclusivity for unsafe pointers, it still defined overlapping read/write access to their pointee as an exclusivity violation.

To resolve this problem, we propose to introduce the concept of *atomic access*, and to amend the Law of Exclusivity as follows:

> Two accesses to the same variable aren't allowed to overlap unless both accesses are reads **or both accesses are atomic**.

We define *atomic access* as a call to one of the following functions in the C atomic operation library:

```text
    atomic_flag_test_and_set         atomic_flag_test_and_set_explicit
    atomic_flag_clear                atomic_flag_clear_explicit
    atomic_store                     atomic_store_explicit
    atomic_load                      atomic_load_explicit
    atomic_exchange                  atomic_exchange_explicit
    atomic_compare_exchange_strong   atomic_compare_exchange_strong_explicit
    atomic_compare_exchange_weak     atomic_compare_exchange_weak_explicit
    atomic_fetch_add                 atomic_fetch_add_explicit
    atomic_fetch_sub                 atomic_fetch_sub_explicit
    atomic_fetch_or                  atomic_fetch_or_explicit
    atomic_fetch_xor                 atomic_fetch_xor_explicit
    atomic_fetch_and                 atomic_fetch_and_explicit
```

We consider two of these operations to *access the same variable* if they operate on the same memory location. (Future proposals may introduce additional ways to perform atomic access, including native support for atomic operations in the Swift Standard Library.)

We view the amendment above as merely formalizing pre-existing practice, rather than introducing any actual new constraint. 

> **Note:** As such, this proposal does not need to come with an associated implementation -- there is no need to change how the Swift compiler implements the Swift memory model. For example, there is no need to relax any existing compile-time or runtime checks for exclusivity violations, because unsafe pointer operations aren't currently covered by such checks. Similarly, the existing llvm-based Thread Sanitizer tool [[Tsan1], [TSan2]] already assumes a C-compatible memory model when it is run on Swift code.

Like C, we leave mixed atomic/non-atomic access to the same memory location as undefined behavior, even if these mixed accesses are guaranteed to never overlap. (This restriction does not apply to accesses during storage initialization and deinitialization; those are always nonatomic.)

## Considerations for Library Authors

While this proposal enables the use of C's atomics operations in Swift code, we don't generally recommend calling C atomics API directly. Rather, we suggest wrapping the low-level atomic invocations in more appropriate Swift abstractions. As an example of how this can be done, we've made available a [proof of concept package][package] implementing the APIs originally included in the first version of this proposal.

In this section we highlight some preexisting aspects of Swift's memory model that need to be taken into account when designing or using a C-based atomics library. 

This section doesn't propose any changes to the language or the Standard Library.

### Interaction with Non-Instantaneous Accesses

As described in [[SE-0176]], Swift allows accesses that are non-instantaneous. For example, calling a `mutating` method on a variable counts as a single write access that is active for the entire duration of the method call:

```swift
var integers: [Int] = ...
...
integers.sort() // A single, long write access
```

The Law of Exclusivity disallows overlapping read/write and write/write accesses to the same variable, so while one thread is performing `sort()`, no other thread is allowed to access `integers` at all. Note that this is independent of `sort()`'s implementation; it is merely a consequence of the fact that it is declared `mutating`.

> **Note:** One reason for this is that the compiler may decide to implement the mutating call by first copying the current value of `integers` into a temporary variable, running `sort` on that, and then copying the resulting value back to `integers`. If `integers` had a computed getter and setter, this is in fact the only reasonable way to implement the mutating call. If overlapping access wasn't disallowed, such implicit copying would lead to race conditions even if the `mutating` method did not actually mutate any data at all.

While C's atomic memory orderings do apply to Swift's variable accesses, and we can use them to reliably synchronize Swift code, they can only apply to accesses whose duration doesn't overlap with the atomic operations themselves. They inherently cannot synchronize variable accesses that are still in progress while the atomic operation is being executed. Code that relies on memory orderings must be carefully written to take this into account.

For example, it isn't possible to implement any "thread-safe" `mutating` methods, no matter how much synchronization we add to their implementation. The following attempt to implement an "atomic" increment operation on `Int` is inherently doomed to failure:

```swift
import Dispatch
import Foundation

let _mutex = NSLock()

extension Int {
  mutating func atomicIncrement() { // BROKEN, DO NOT USE
    _mutex.lock()
    self += 1
    _mutex.unlock()
  }
}

var i: Int
...
i = 0
DispatchQueue.concurrentPerform(iterations: 10) { _ in
  for _ in 0 ..< 1_000_000 {
    i.atomicIncrement()  // Exclusivity violation
  }
}
print(i)
```

Even though `NSLock` does guarantee that the `self += 1` line is always serialized, any concurrent `atomicIncrement` invocations still count as an exclusivity violation, because the write access to `i` starts when the function call begins, before the call to `_mutex.lock()`. Therefore, the code above has undefined behavior, despite all the locking. (For example, it may print any value between one and ten million, or it may trap in a runtime exclusivity check, or indeed it may do something else.)

The same argument also applies to property and subscript setters (unless they are declared `nonmutating`), and to `inout` arguments of any function call.

Methods of types with reference semantics (such as classes) can modify their instance variables without declaring themselves `mutating`, so they aren't constrained by this limitation. (Of course, the implementation of the method must still guarantee that the Law is upheld for any stored properties they themselves access -- but synchronization tools such as locks do work in this context.)

### Interaction with Implicit Pointer Conversions


To simplify interoperability with functions imported from C, Swift provides several forms of implicit conversions from Swift values to unsafe pointers. This often requires the use of Swift's special `&` syntax for passing inout values. At first glance, this use of the ampersand resembles C's address-of operator, and it seems to work in a similar way:

```swift
func test(_ address: UnsafePointer<Int>)

var value = 42

// Implicit conversion from `inout Int` to `UnsafePointer<Int>`
test(&value)
```

However, despite the superficial similarity, the `&` here isn't an address-of operator at all. Swift variables do not necessarily have a stable location in memory, and even in case they happen to get assigned one, there is generally no reliable way to retrieve the address of their storage. (The obvious exceptions are dynamic variables that we explicitly allocate ourselves.) What the `&`-to-pointer conversion actually does here is equivalent to a call to `withUnsafePointer(to:)`:

```swift
withUnsafePointer(to: &value) { pointer in
  test(pointer)
}
```

This counts as a write access to the original value, and (unlike with C) the generated pointer may address a temporary copy of the value -- so it is only considered valid for the duration of the closure call, and the addressed memory location may change every time the code is executed. Because of these two reasons, inout-to-pointer conversions must not be employed to pass "the address" of an atomic value to an atomic operation.

For example, consider the following constructs, imported from C wrappers of `_Atomic intptr_t`, `atomic_load` and `atomic_fetch_add`:

```swift
struct AtomicIntStorage { ... }
func atomicLoadInt(_ address: UnsafePointer<AtomicIntStorage>) -> Int
func atomicFetchAddInt(
  _ address: UnsafeMutablePointer<AtomicIntStorage>, 
  _ delta: Int
) -> Int
```

It is tempting to call these by simply passing an inout reference to a Swift variable of type `AtomicIntStorage`:

```swift
// BROKEN, DO NOT USE
var counter = AtomicIntStorage() // zero init
DispatchQueue.concurrentPerform(iterations: 10) { _ in
  for _ in 0 ..< 1_000_000 {
    atomicFetchAddInt(&counter, 1)  // Exclusivity violation
  }
}
print(atomicLoadInt(&counter) // ???
```

Unfortunately, this code has undefined behavior. `&counter` counts as a write access to `counter`, and as we explained in the previous section, this leads to a clear exclusivity violation. Additionally, `&counter` may result in a different pointer in each thread of execution (or, perhaps even each iteration of the loop), which would defeat atomicity.

Given that the concurrency in this example is neatly isolated to a single section of code, we could wrap it in a `withUnsafeMutablePointer(to:)` invocation that generates a single (but still temporary) pointer. This resolves the problem:

```swift
var counter = AtomicIntStorage() // zero init
withUnsafeMutablePointer(to: counter) { pointer in
  DispatchQueue.concurrentPerform(iterations: 10) { _ in
    for _ in 0 ..< 1_000_000 {
      atomicFetchAddInt(pointer, 1) // OK
    }
  }
  print(atomicLoadInt(pointer) // 10_000_000
}
```

However, it isn't always possible to do this. In cases where thread lifetime cannot be restricted to a single code block, the best way to produce a pointer that is suitable for atomic operations is either by manually allocating a dynamic variable, or by using `ManagedBuffer` APIs to retrieve stable pointers to inline storage inside a class instance.


## Source Compatibility

This proposal requires no changes to Swift's implementation; as such, it has no source compatibility impact.

## Effect on ABI Stability

None.

## Effect on API Resilience

None.

## Alternatives Considered

A previous version of this proposal included a large set of APIs implementing a native Swift atomics facility. We expect a revised version of these APIs will return in a followup proposal later (following further work on Swift's [Ownership Manifesto]); however, for now, we expect to develop them [as a standalone package][package], implemented around the operations provided by the C standard library. This C-based reimplementation of the module exports the same public interface and it has the same performance characteristics as the originally proposed native implementation.

## References

[Ownership Manifesto]: https://github.com/apple/swift/blob/master/docs/OwnershipManifesto.md
**\[Ownership Manifesto]** John McCall. "Ownership Manifesto." *Swift compiler documentation*, May 2, 2017. https://github.com/apple/swift/blob/master/docs/OwnershipManifesto.md

[SE-0176]: https://github.com/swiftlang/swift-evolution/blob/master/proposals/0176-enforce-exclusive-access-to-memory.md
**\[SE-0176]** John McCall. "Enforce Exclusive Access to Memory. *Swift Evolution Proposal,* SE-0176, May 2, 2017. https://github.com/swiftlang/swift-evolution/blob/master/proposals/0176-enforce-exclusive-access-to-memory.md

[Generics Manifesto]: https://github.com/apple/swift/blob/master/docs/GenericsManifesto.md
**\[Generics Manifesto]** Douglas Gregor. "Generics Manifesto." *Swift compiler documentation*, 2016. https://github.com/apple/swift/blob/master/docs/GenericsManifesto.md

[C++17]: https://isocpp.org/std/the-standard
**\[C++17]** ISO/IEC. *ISO International Standard ISO/IEC 14882:2017(E) – Programming Language C++.* 2017.
https://isocpp.org/std/the-standard

[C18]: https://www.iso.org/standard/74528.html
**\[C18]** *ISO International Standard ISO/IEC 9899:2018 - Information Technology -- Programming Languages -- C.*. 2018.
https://www.iso.org/standard/74528.html

**\[Williams 2019]** Anthony Williams. *C++ Concurrency in Action.* 2nd ed., Manning, 2019.

**\[Nagarajan 2020]** Vijay Nagarajan, Daniel J. Sorin, Mark D. Hill, David A. Wood. *A Primer on Memory Consistency and Cache Coherence.* 2nd ed., Morgan & Claypool, February 2020. https://doi.org/10.2200/S00962ED2V01Y201910CAC049 

**\[Herlihy 2012]** Maurice Herlihy, Nir Shavit. *The Art of Multiprocessor Programming.* Revised 1st ed., Morgan Kauffmann, May 2012.

[Boehm 2008]: https://doi.org/10.1145/1375581.1375591
**\[Boehm 2008]** Hans-J. Boehm, Sarita V. Adve. "Foundations of the C++ Concurrency Memory Model." In *PLDI '08: Proc. of the 29th ACM SIGPLAN Conf. on Programming Language Design and Implementation*, pages 68–78, June 2008.
  https://doi.org/10.1145/1375581.1375591

[Batty 2011]: https://doi.org/10.1145/1925844.1926394
**\[Batty 2011]** Mark Batty, Scott Owens, Susmit Sarkar, Peter Sewell, Tjark Weber. "Mathematizing C++ Concurrency." In *ACM SIGPlan Not.,* volume 46, issue 1, pages 55–66, January 2011. https://doi.org/10.1145/1925844.1926394

[Boehm 2012]: https://doi.org/10.1145/2247684.2247688
**\[Boehm 2012]** Hans-J. Boehm. "Can Seqlocks Get Along With Programming Language Memory Models?" In *MSPC '12: Proc. of the 2012 ACM SIGPLAN Workshop on Memory Systems Performance and Correctness*, pages 12–20, June 2012. https://doi.org/10.1145/2247684.2247688

[Nienhuis 2016]: https://doi.org/10.1145/2983990.2983997
**\[Nienhuis 2016]** Kyndylan Nienhuis, Kayvan Memarian, Peter Sewell. "An Operational Semantics for C/C++11 Concurrency." In *OOPSLA 2016: Proc. of the 2016 ACM SIGPLAN Conf. on Object Oriented Programming, Systems, Languages, and Applications,* pages 111–128, October 2016. https://doi.org/10.1145/2983990.2983997

[Mattarei 2018]: https://doi.org/10.1007/978-3-319-89963-3_4
**\[Mattarei 2018]** Christian Mattarei, Clark Barrett, Shu-yu Guo, Bradley Nelson, Ben Smith. "EMME: a formal tool for ECMAScript Memory Model Evaluation." In *TACAS 2018: Lecture Notes in Computer Science*, vol 10806, pages 55–71, Springer, 2018. https://doi.org/10.1007/978-3-319-89963-3_4

[N2153]: http://wg21.link/N2153
**\[N2153]** Raúl Silvera, Michael Wong, Paul McKenney, Bob Blainey. *A simple and efficient memory model for weakly-ordered architectures.* WG21/N2153, January 12, 2007. http://wg21.link/N2153

[N4455]: http://wg21.link/N4455
**\[N4455]** JF Bastien *No Sane Compiler Would Optimize Atomics.* WG21/N4455, April 10, 2015. http://wg21.link/N4455

[P0124]: http://wg21.link/P0124
**\[P0124]** Paul E. McKenney, Ulrich Weigand, Andrea Parri, Boqun Feng. *Linux-Kernel Memory Model.* WG21/P0124r6. September 27, 2018. http://wg21.link/P0124

[TSan1]: https://developer.apple.com/documentation/code_diagnostics/thread_sanitizer
**\[TSan1]** *Thread Sanitizer -- Audit threading issues in your code.* Apple Developer Documentation. Retrieved March 2020. https://developer.apple.com/documentation/code_diagnostics/thread_sanitizer

[TSan2]: https://clang.llvm.org/docs/ThreadSanitizer.html
**\[TSan2]** *ThreadSanitizer*. Clang 11 documentation. Retrieved March 2020. https://clang.llvm.org/docs/ThreadSanitizer.html


⚛︎︎

<!-- Local Variables: -->
<!-- mode: markdown -->
<!-- fill-column: 10000 -->
<!-- eval: (setq-local whitespace-style '(face tabs newline empty)) -->
<!-- eval: (whitespace-mode 1) -->
<!-- eval: (visual-line-mode 1) -->
<!-- End: -->
