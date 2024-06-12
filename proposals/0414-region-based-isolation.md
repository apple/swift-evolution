# Region based Isolation

* Proposal: [SE-0414](0414-region-based-isolation.md)
* Authors: [Michael Gottesman](https://github.com/gottesmm) [Joshua Turcotti](https://github.com/jturcotti)
* Review Manager: [Holly Borla](https://github.com/hborla)
* Status: **Accepted**
* Upcoming Feature Flag: `RegionBasedIsolation` (Enabled in Swift 6 language mode)
* Implementation: On `main` gated behind `-enable-experimental-feature RegionBasedIsolation`
* Review: ([first pitch](https://forums.swift.org/t/pitch-safely-sending-non-sendable-values-across-isolation-domains/66566)), ([second pitch](https://forums.swift.org/t/pitch-region-based-isolation/67888)), ([first review](https://forums.swift.org/t/se-0414-region-based-isolation/68805)), ([revision](https://forums.swift.org/t/returned-for-revision-se-0414-region-based-isolation/69123)), ([second review](https://forums.swift.org/t/se-0414-second-review-region-based-isolation/69740)), ([acceptance](https://forums.swift.org/t/accepted-with-modifications-se-0414-region-based-isolation/70051))

## Introduction

Swift Concurrency assigns values to *isolation domains* determined by actor and
task boundaries. Code running in distinct isolation domains can execute
concurrently, and `Sendable` checking defines away concurrent access to
shared mutable state by preventing non-`Sendable` values from being passed
across isolation boundaries full stop. In practice, this is a significant
semantic restriction, because it forbids natural programming patterns that are
free of data races.

In this document, we propose loosening these rules by introducing a
new control flow sensitive diagnostic that determines whether a non-`Sendable`
value can safely be transferred over an isolation boundary. This is done by
introducing the concept of *isolation regions* that allows the compiler to
reason conservatively if two values can affect each other. Through the usage of
isolation regions, the language can prove that transferring a non-`Sendable`
value over an isolation boundary cannot result in races because the value (and
any other value that might reference it) is not used in the caller after the
point of transfer.

## Motivation

[SE-0302](0302-concurrent-value-and-concurrent-closures.md) states
that non-`Sendable` values cannot be passed across isolation boundaries. The
following code demonstrates a `Sendable` violation when passing a
newly-constructed value into an actor-isolated function:

```swift
// Not Sendable
class Client {
  init(name: String, initialBalance: Double) { ... }
}

actor ClientStore {
  var clients: [Client] = []

  static let shared = ClientStore()

  func addClient(_ c: Client) {
    clients.append(c)
  }
}

func openNewAccount(name: String, initialBalance: Double) async {
  let client = Client(name: name, initialBalance: initialBalance)
  await ClientStore.shared.addClient(client) // Error! 'Client' is non-`Sendable`!
}
```

This is overly conservative; the program is safe because:

* `client` does not have access to any non-`Sendable` state from its constructor
  parameters since Strings and Doubles are `Sendable`.
* `client` just being constructed implies that `client` cannot have any uses
  outside of `openNewAccount`.
* `client` is not used within `openNewAccount` beyond `addClient`.

The simple example above shows the expressivity limitations of Swift's strict
concurrency checking. Programmers are required to use unsafe escape hatches,
such as `@unchecked Sendable` conformances, for common patterns that are already
free of data races.

## Proposed solution

We propose the introduction of a new control flow sensitive diagnostic that
enables transferring non-`Sendable` values across isolation boundaries and emits
errors at use sites of non-`Sendable` values that have already been transferred
to a different isolation domain.

This change makes the motivating example valid code, because the `client`
variable does not have any further uses after it's transferred to the
`ClientStore.shared` actor through the call to `addClient`. If we were to modify
`openNewAccount` to call a method on `client` after the call to `addClient`, the
code would be invalid since a non-`Sendable` value that had already been
transferred from a non-isolated context to an actor-isolated context could be
accessed concurrently:

```swift
func openNewAccount(name: String, initialBalance: Double) async {
  let client = Client(name: name, initialBalance: initialBalance)
  await ClientStore.shared.addClient(client)
  client.logToAuditStream() // Error! Already transferred into clientStore's isolation domain... this could race!
}
```

After the call to `addClient`, any other non-`Sendable` value that is statically
proven to be impossible to reference from `client` can still be used safely. We
can prove this property using the concept of *isolation regions*. An isolation
region is a set of values that can only ever be referenced through other values
within that set. Formally, two values $x$ and $y$ are defined to be within the
same isolation region at a program point $p$ if:

1. $x$ may alias $y$ at $p$.
2. $x$ or a property of $x$ might be referenceable from $y$ via chained access of $y$'s properties at $p$.

This definition ensures that non-`Sendable` values in different isolation
regions can be used concurrently, because any code that uses $x$ cannot affect
$y$. Lets consider a further example:

```swift
let john = Client(name: "John", initialBalance: 0)
let joanna = Client(name: "Joanna", initialBalance: 0)

await ClientStore.shared.addClient(john)
await ClientStore.shared.addClient(joanna) // (1)
```

The above code creates two new `Client` instances. It's impossible for
`john` to reference `joanna` and vice versa, so these two values belong to
different isolation regions. Values in different isolation regions can be
used concurrently, so the use of `joanna` at `(1)`, which may be executing
concurrently with some code inside `ClientStore.shared` that accesses `john`,
is safe from data races.

In contrast, if we add a `friend` property to `Client` and assign `joanna` to
`john.friend`:

```swift
let john = Client(name: "John", initialBalance: 0)
let joanna = Client(name: "Joanna", initialBalance: 0)

john.friend = joanna // (1)

await ClientStore.shared.addClient(john)
await ClientStore.shared.addClient(joanna) // (2)
```

After the assignment at point `(1)`, `joanna` can be referenced through
`john.friend`, so `john` and `joanna` must be in the same isolation region at
`(1)`. The access to `joanna` at point `(2)` can be executing concurrently with
code inside `ClientStore.shared` that accesses `john.friend`. Using `joanna` at
point `(2)` is diagnosed as a potential data race.

## Detailed Design

NOTE: While this proposal contains rigorous details that enable the compiler to
prove the absence of data races, programmers will not have to reason about
regions at this level of detail. The compiler will allow transfers of non-`Sendable` values between
isolation domains where it can prove they are safe and will emit diagnostics
when it cannot at potential concurrent access points so that programmers don't
have to reason through the data flow themselves.

### Isolation Regions

#### Definitions

An *isolation region* is a set of non-`Sendable` values that can only be aliased
or reachable from values that are within the isolation region. An isolation
region can be associated with a specific *isolation domain* associated with a
task, protected by an actor instance or a global actor, or disconnected from any
specific isolation domain. As the program executes, each isolation region can be
merged with other isolation regions as new values begin to alias or be reachable
from each other.

Isolation regions and isolation domains are not concepts that are explicitly
denoted in source code. To help explain the concepts throughout this proposal,
isolation regions and their isolation domains will be written in comments in
the following notation:

* `[(a)]`: A single disconnected region with a single value.

* `[{(a), actorInstance}]`: A single region that is isolated to actorInstance.

* `[(a), {(b), actorInstance}]`: Two values in separate isolation regions. a's
  region is disconnected but b's region is assigned to the isolation domain of
  the actor instance `actorInstance`.

* `[{(x, y), @OtherActor}, (z), (w, t)]`: Five values in three separate
  isolation regions. `x` and `y` are within one isolation region that is
  isolated to the global actor `@OtherActor`. `z` is within its own
  disconnected isolation region. `w` and `t` are within the same disconnected
  region.

* `[{(a), Task1}]`: A single region that is part of `Task1`'s
  isolation domain.

#### Rules for Merging Isolation Regions

Isolation regions are merged together when the program introduces a potential
alias or access path to another value. This can happen through function calls,
and assignments. Many expression forms are sugar for a function application,
including property accesses.

Given a function $f$ with arguments $a_{i}$ and result that is assigned to
variable $y$:

$$
y = f(a_{0}, ..., a_{n})
$$

1. All regions of non-`Sendable` arguments $a_{i}$ are merged into one larger
   region after $f$ executes.
2. If any of $a_{i}$ are non-`Sendable` and $y$ is non-`Sendable`, then $y$ is in
   the same merged region as $a_{i}$. If all of the $a_{i}$ are `Sendable`,
   then $y$ is within a new disconnected region that consists only of $y$.
3. If $y$ is not a new variable, i.e. it's mutable, then

   a) If $y$ was previously captured by reference in a closure, then the assignment
      to $y$ merges $y$'s new region into its old region.

   b) If $y$ was not captured by reference, then $y$'s old region is
      forgotten.

The above rules are conservative; without any further annotations, we must assume:
* In the implementation of $f$, any $a_{i}$ could become reachable from $a_{j}$.
* $y$ could be one of the $a_{i}$ values or alias contents of $a_{i}$.
* If $y$ was captured by reference in a closure and then assigned a new value,
  calling the closure could reference $y$'s new value.

See the future directions section for additional annotations that enable more
precise regions.

##### Examples

Now lets apply these rules to some specific examples in Swift code:

* **Initializing a let or var binding**. ``let y = x, var y = x``. Initializing
  a let or var binding `y` with `x` results in `y` being in the same region as
  `x`. This follows from rule `(2)` since formally a copy is equivalent to calling a
  function that accepts `x` and returns a copy of `x`.

  ```swift
  func bindingInitialization() {
    let x = NonSendable()
    // Regions: [(x)]
    let y = x
    // Regions: [(x, y)]
    let z = consume x
    // Regions: [(x, y, z)]
  }
  ```

  Note that whether or not `x` is in the region after `consume x` does not
  change program semantics. A valid program must still obey the no-reuse
  constraints of `consume`.

* **Assigning a var binding**. ``y = x``. Assigning a var binding `y` with `x`
  results in `y` being in the same region as `x`. If `y` is not captured by
  reference in a closure, then `y`'s previous assigned region is forgotten due
  to `(3)(b)`:

  ```swift
  func mutableBindingAssignmentSimple() {
    var x = NonSendable()
    // Regions: [(x)]
    let y = NonSendable()
    // Regions: [(x), (y)]
    x = y
    // Regions: [(x, y)]
    let z = NonSendable()
    // Regions: [(x, y), (z)]
    x = z
    // Regions: [(y), (x, z)]
  }
  ```

  In contrast if `y` was captured in a closure by reference, then `y`'s former
  region is merged with the region of `x` due to `(3)(a)`.

  ```swift
  // Since we pass x as inout in the closure, the closure has to capture x by
  // reference.
  func mutableBindingAssignmentClosure() {
    var x = NonSendable()
    // Regions: [(x)]
    let closure = { useInOut(&x) }
    // Regions: [(x, closure)]
    let y = NonSendable()
    // Regions: [(x, closure), (y)]
    x = y
    // Regions: [(x, closure, y)]
  }
  ```

* **Accessing a non-`Sendable` property of a non-`Sendable` value**.
  ``let y = x.f``. Accessing a property `f` on a non-`Sendable` value `x`
  results in a value `y` that must be in the same region as `x`. This follows
  from `(2)` since formally a property access is equivalent to calling a getter
  passing `x` as `self`. Importantly, this property forces all non-`Sendable`
  types to form one large region containing their non-`Sendable` state:

  ```swift
  func assignFieldToValue() {
    let x = NonSendableStruct()
    // Regions: [(x)]
    let y = x.field
    // Regions: [(x, y)]
  }
  ```

* **Setting a non-`Sendable` property of a non-`Sendable` value**. ``y.f = x``
  Assigning `x` into a property `y.f` results in `y` and `y.f` being in the
  same region as `x`. This again follows from `(2)`:

  ```swift
  func assignValueToField() {
    let x = NonSendableStruct()
    // Regions: [(x)]
    let y = NonSendable()
    // Regions: [(x), (y)]
    x.field = y
    // Regions: [(x, y)]
  }
  ```

* **Capturing non-`Sendable` values by reference in a closure**. ``closure = {
  useX(x); useY(y) }``. Capturing non-`Sendable` values `x` and `y` results in
  `x` and `y` being in the same region. This is a consequence of `(2)` since
  `x` and `y` are formally arguments to the closure formation. This
  also means that the closure must be part of that same region:

  ```swift
  func captureInClosure() {
    let x = NonSendable()
    // Regions: [(x)]
    let closure = { print(x) }
    // Regions: [(x, closure)]
  }
  ```

* **Function arguments in the body of a function**. Given a function `func
  transfer(x: NonSendable, y: NonSendable) async`, in the body of
  `transfer`, `x` and `y` are considered to be within the same region. Since
  `self` is a function argument to methods, this implies that when `self` is
  non-`Sendable` all method arguments must be in the same region as `self`:

  ```swift
  func transfer(x: NonSendable, y: NonSendable) {
    // Regions: [(x, y)]
    let z = NonSendable()
    // Regions: [(x, y), (z)]
    f(x, z)
    // Regions: [(x, y, z)]
  }
  ```

#### Control Flow

Isolation regions are also affected by control flow. Let $x$ and $y$
be two values that are used in a control flow statement. After the
control flow statement, the regions of $x$ and $y$ are merged if any
of the blocks within the statement merge the regions of $x$ and $y$.
For example:

```swift
// Regions: [(x), (y)]
var x: NonSendable? = NonSendable()
var y: NonSendable? = NonSendable()
if ... {
  // Regions: [(x), (y)]
  x = y
  // Regions: [(x, y)]
} else {
  // Regions: [(x), (y)]
}

// Regions: [(x, y)]
```

Because the first block of the `if` statement assigns `x` to `y`, causing
their regions to be merged within that block, `x` and `y` are in the
same region after the `if` statement.

This rule is conservative since it is always safe to consider two values
that are disconnected from each other as if they are isolated together. The
only effect would be the rejection of programs that we otherwise could accept.

The above description of regions naturally allows the definition of an
optimistic forward dataflow problem that allows us to determine at every point
of the program the isolation region that a value belongs to. We outline this
dataflow in more detail in an [appendix](#isolation-region-dataflow) to this proposal.

### Transferring Values and Isolation Regions

As defined above, all non-`Sendable` values in a Swift program belong to some
isolation region. An isolation region is isolated to an actor's isolation
domain, a task's isolation domain, or disconnected from any specific isolation
domain:

```swift
actor Actor {
  // 'field' is in an isolation region that is isolated to the actor instance.
  var field: NonSendable

  func method() {
    // 'ns' is in a disconnected isolation region.
    let ns = NonSendable()
  }
}

func nonisolatedFunction() async {
  // 'ns' is in a disconnected isolation region.
  let ns = NonSendable()
}

// 'globalVariable' is in a region that is isolated to @GlobalActor.
@GlobalActor var globalVariable: NonSendable

// 'x' is isolated to the task that calls taskIsolatedArgument.
func taskIsolatedArgument(_ x: NonSendable) async { ... }
```

As the program executes, an isolation region can be passed across isolation
boundaries, but an isolation region can never be accessed by multiple
isolation domains at once. When a region $R_{1}$ is merged into another region
$R_{2}$ that is isolated to an actor, $R_{1}$ becomes protected by
that isolation domain and cannot be passed or accessed across isolation
boundaries again.

The following code example demonstrates merging a disconnected region into a
region that is `@MainActor` isolated:

```swift
@MainActor func transferToMainActor<T>(_ t: T) async { ... }

func assigningIsolationDomainsToIsolationRegions() async {
  // Regions: []

  let x = NonSendable()
  // Regions: [(x)]

  let y = x
  // Regions: [(x, y)]

  await transferToMainActor(x)
  // Regions: [{(x, y), @MainActor}]

  print(y) // Error!
}
```

Passing `x` into `transferToMainActor` introduces a potential alias to `x`
from any `@MainActor`-isolated state, because the implementation of
`transferToMainActor` can store `x` into any state within that isolation
domain. So, the region containing `x` must be merged into the `@MainActor`'s
region. Accessing `y` after that merge is an error because `x` and `y` are now
both effectively `@MainActor` isolated, and the access occurs from outside the
`@MainActor`.

Formally, when we pass a non-`Sendable` value $v$ into a function $f$ and the
call to $f$ crosses an isolation boundary, then we say that $v$ and $v$'s
region are *transferred* into $f$. During the execution of $f$, the only way to
reference $v$ or any value in the same region as $v$ is through the parameter
bound to $v$ in the implementation of $f$. This deep structural isolation
guarantees that values in a region cannot be accessed concurrently.

In this proposal, we are defining the default convention for passing
non-`Sendable` values across isolation boundaries as being a transfer
operation. This does not apply when calling async functions from within the same
isolation domain. To do so would require an explicit transferring modifier which
is described in the [Future Directions](#transferring-parameters) section below.

### Taxonomy of Isolation Regions

There are four types of isolation regions that a non-`Sendable` value can belong
to that determine the rules for transferring value over an isolation boundary.

#### Disconnected Isolation Regions

A *disconnected isolation region* is a region that consists only of
non-`Sendable` values and is not associated with a specific isolation
domain. A value in a disconnected region can be transferred to another
isolation domain as long as the value is used uniquely by said isolation
domain and never used later outside of that isolation domain lest we introduce
races:

```swift
@MainActor func transferToMainActor<T>(_ t: T) async { ... }

actor Actor {
  func method() async {
    let x = NonSendable()
    // Regions: [(x)]

    await transferToMainActor(x)
    // Regions: [{(x), @MainActor}]

    print(x) // Error! x being used outside of @MainActor isolated code.
  }
}
```

#### Actor Isolated Regions

An *actor isolated region* is a region that is strongly bound to a specific
actor's isolation domain. Since the region is tied to an actor's isolation
domain, the values of the region can *never* be transferred into another
isolation domain since that would cause the non-`Sendable` value to be used by
code both inside and outside the actor's isolation domain allowing for races:

```swift
actor Actor {
  var nonSendable: NonSendable
}

@MainActor func actorRegionExample() async {
  let a = Actor()
  // Regions: [{(a.nonSendable), a}]

  let x = await a.nonSendable // Error!

  await transferToMainActor(a.nonSendable) // Error!
}
```

In the above code example, `x` must be in the actor `a`'s region because it
aliases actor-isolated state, making `x` effectively isolated to `a`. The
initialization is invalid, because `x` is not usable from a `@MainActor`
context. Similarly, attempting to transfer actor-isolated state into another
isolation domain is invalid.

The parameters of an actor method or a global actor isolated function are
considered to be within the actor's region. This is since a caller can pass
actor isolated state as an argument to such a method or function. This implies
that parameters of actor isolated methods and functions can not be transferred
like other values in actor isolation regions.

The objects that make up an actor region varies depending on the kind of actor:

* **Actor**. An actor region for an actor contains the actor's non-`Sendable`
  fields and any values derived from the actor's fields.

  ```swift
  class NonSendableLinkedList {
    var next: NonSendableLinkedList?
  }

  actor Actor {
    var listHead: NonSendableLinkedList

    func method() async {
      // Regions: [{(self.listHead, self.listHead.next, ...), self}]

      let x = self.listHead
      // Regions: [{(x, self.listHead, self.listHead.next, ...), self}]

      let z = self.listHead.next!
      // Regions: [{(x, z, self.listHead, self.listHead.next, ...), self}]
      ...
    }
  }
  ```

  In the above example, `x` is in `self`'s region because it aliases
  non-`Sendable` state isolated to `self`, and `z` is in `self`'s region
  because the value of `next` is reachable from `self.listHead`.

* **Global Actor**. An actor region for a global actor contains any global
  variables isolated to the global actor, all instances of nominal types
  isolated to the global actor, and all values derived from the fields of the
  isolated global variable or nominal types.

  ```swift
  @GlobalActor var firstList: NonSendableLinkedList
  @GlobalActor var secondList: NonSendableLinkedList

  @GlobalActor func useGlobalActor() async {
    // Regions: [{(firstList, secondList), @GlobalActor}]

    let x = firstList
    // Regions: [{(x, firstList, secondList), @GlobalActor}]

    let y = secondList.listHead.next!
    // Regions: [{(x, firstList, secondList, y), @GlobalActor}]
    ...
  }
  ```

  In the above code example `x` is in `@GlobalActor`'s region because it
  aliases `@GlobalActor`-isolated state, and `y` is in `@GlobalActor`'s region
  because it aliases a value that's reachable from `@GlobalActor`-isolated
  state.

An operation to disconnect a value from an actor region in order to transfer
it to another isolation domain is out of the scope of this proposal. A
potential extension to enable this is described in the [Future Directions](disconnected-fields-and-the-disconnect-operator).

#### Task Isolated Regions

A task isolated isolation region consists of values that are isolated to a
specific task. This can only occur today in the form of the parameters of
nonisolated asynchronous functions since unlike actors, tasks do not have
non-`Sendable` state that can be isolated to them. Similarly to actor isolated
regions, a task isolated region is strongly tied to the task so values within
the task isolated region cannot be transferred out of the task:

```swift
@MainActor func transferToMainActor(_ x: NonSendable) async { ... }

func nonIsolatedCallee(_ x: NonSendable) async { ... }

func nonIsolatedCaller(_ x: NonSendable) async {
  // Regions: [{(x), Task1}]
  
  // Not a transfer! Same Task!
  await nonIsolatedCallee(x)

  // Error!
  await transferToMainActor(x)
}
```

In the example above, `x` is in a task isolated region. Since
`nonIsolatedCallee` will execute on the same task as `nonIsolatedCallee`, they
are in the same isolation domain and a transfer does not occur. In contrast,
`transferToMainActor` is in a different isolation domain so passing `x` to it is
a transfer resulting in an error.

#### Invalid Isolation Regions

An invalid isolation region is a region that results from conditional control
flow causing the merging of regions that can never be merged together due to
isolation properties. It is an error to use a value that is in an invalid
isolation region since statically the specific region that the value belongs to
can not be determined:

```swift
func mergeTwoActorRegions() async {
  let a1 = Actor()
  // Regions: [{(), a1}]
  let a2 = Actor()
  // Regions: [{(), a1}, {(), a2}]
  let x = NonSendable()
  // Regions: [{(), a1}, {(), a2}, (x)]

  if await boolean {
    await a1.useNS(x)
    // Regions: [{(x), a1}, {(), a2}]
  } else {
    await a2.useNS(x)
    // Regions: [{(), a1}, {(x), a2}]
  }

  // Regions: [{(x), invalid}, {(), a1}, {(), a2}]
}
```

#### Merging Isolation Regions

The behavior of merging two isolation regions depends on the kind of each
region.

* **Disconnected and Disconnected**. Given two non-`Sendable` values in separate
  disconnected regions, merging the regions produces one large disconnected
  region.

  ```swift
  let x = NonSendable()
  // Regions: [(x)]
  let y = NonSendable()
  // Regions: [(x), (y)]
  useValue(x, y)
  // Regions: [(x, y)]
  ```

* **Disconnected and Actor Isolated**. Merging a disconnected region and an
  actor-isolated region expands the actor-isolated region with the values in
  the disconnected region. This forces all values in the disconnected region
  to be treated as if they are isolated to the actor. This can only occur when
  calling a method on an actor or assigning into an actor's field:

  ```swift
  func example1() async {
    let x = NonSendable()
    // Regions : [(x)]

    let a = Actor()
    // Regions: [(x), {(a.field), a}]

    await a.useNonSendable(x)
    // Regions: [{(x, a.field), a}]

    useValue(x) // Error! 'x' is effectively isolated to 'a'

    let y = NonSendable()
    // Regions: [{(x, a.field), a}, (y)]

    a.field = y
    // Regions: [{(x, a.field, y), a}]

    useValue(y) // Error! 'y' is effectively isolated to 'a'
  }
  ```

* **Disconnected and Task isolated**. Merging a disconnected region and a
  task-isolated region expands the task-isolated region with the values in the
  disconnected region. This forces all values in the disconnected region to be
  treated like they are isolated to the task:
  
  ```swift
  func nonIsolated(_ arg: NonSendable) async {
    // Regions: [{(arg), Task1}]
    let x = NonSendable()
    // Regions: [{(arg), Task1}, (x)]
    arg.doSomething(x)
    // Regions: [{(arg, x), Task1}]
    await transferToMainActor(x) // Error! 'x' is isolated to 'Task1'
  }
  ```

* **Actor isolated and Actor isolated**. Merging two actor-isolated regions
  results in an invalid region. This can only occur via conditional control flow
  since an actor isolated region cannot be transferred into another actor's
  isolation region:

  ```swift
  func test() async {
    let a1 = Actor()
    // Regions: [{(), a1}]
    let a2 = Actor()
    // Regions: [{(), a1}, {(), a2}]
    let x = NonSendable()
    // Regions: [{(), a1}, {(), a2}, (x)]

    if await boolean {
      await a1.useNS(x)
      // Regions: [{(x), a1}, {(), a2}]
    } else {
      await a2.useNS(x)
      // Regions: [{(), a1}, {(x), a2}]
    }

    // Regions: [{(x), invalid}, {(), a1}, {(), a2}]
  }
  ```

  In the above example, `x` cannot be accessed from `test` after the `if`
  statement since `x` is now in an invalid isolation domain.

* **Actor Isolated and Task Isolated**. Merging an actor isolated region and
  task isolated region results in an invalid isolation region. This occurs since
  an actor isolated region and a task isolated region can run concurrently from
  each other. Since values in either type of region cannot be transferred, this
  can only occur through conditional control flow:
  
  ```swift
  func nonIsolated(_ arg: NonSendable) async {
    // Regions: [{(arg), Task1}]
    let a = Actor()
    // Regions: [{(), a}, {(arg), Task1}]
    let x = NonSendable()
    // Regions: [(x), {(), a}, {(arg), Task1}]

    if await boolean {
      await a.useNS(x)
      // Regions: [{(x), a}, {(arg), Task1}]
    } else {
      arg.useNS(x)
      // Regions: [{(), a}, {(arg, x), Task1}]
    }

    // Regions: [{(arg, x), invalid}, {(), a}, {(), Task1}]
  }
  ```

* **Task Isolated and Task Isolated**. Since task isolated isolation regions are
  only introduced due to function arguments, it is impossible to have two
  separate task isolated regions that could be merged.

### Weak Transfers, `nonisolated` functions, and disconnected isolation regions

When we transfer a value over an isolation boundary, the caller according to the
ownership conventions of Swift may still own the value despite it being illegal
for the caller to use the value due to region based isolation:

```swift
class NonSendable {
  deinit { print("deinit was called") }
}

@MainActor func transferToMainActor<T>(_ t: T) async {  }

actor MyActor {
  func example() async {
    // Regions: [{(), self}]
    let x = NonSendable()
    
    // Regions: [(x), {(), self}]
    await transferToMainActor(x)
    // Regions: [{(x), @MainActor}, {(), self}]

    // Error! Since 'x' was transferred to @MainActor, we cannot use 'x'
    // directly here.
    useValue(x)                                                      // (1)
    
    print("After nonisolated callee")

    // But since example still owns 'x', the lifetime of 'x' ends here. (2)
  }
}

let a = MyActor()
await a.example()
```

In the above example, the program will first print out "After nonisolated
callee" and then "deinit was called". This is because even though
`nonIsolatedCallee` is transferred `x`'s region, `x` is still passed to
`nonIsolatedCallee` using Swift's default guaranteed ownership convention. This
implies that the caller from an ownership perspective still owns the memory of
the class implying the lifetime of `x` actually ends at `(1)` despite the caller
not being able to use `x` directly at that point.

This illustrates how the transfer convention used when passing a value over an
isolation boundary is a *weak transfer* convention. A weak transfer convention
implies that one can still reference a value within the transferred region from
the original isolation domain, but one cannot access the value through the
reference. In contrast, a *strong transfer* convention would require that the
caller isolation domain cannot maintain even references to values in the
transferred isolation region. This would require transferring to always be a +1
operation since to preserve this property we would always need to pass off
ownership from the caller to the callee to ensure that the callee cleans up the
region as shown in the example above.

Requiring our transfer convention to be a strong convention would have several
unfortunate side-effects:

* All async functions would by default take their parameters as owned. This
  would be an ABI break and would also have the unfortunate consequence that the
  bodies of asynchronous functions could never be marked as readonly or readnone
  since they may need to invoke a deinit to end ownership of a value and deinits
  may have unknown side-effects.

* This would hurt the performance of asynchronous functions by increasing the
  amount of ARC overhead required since unless we inline, there will be a cross
  function call boundary copy that can not be eliminated. This in turn would
  cause hits to code-size since to remedy this performance problem the inliner
  would need to be more aggressive about inlining code.

To achieve a *strong transfer* convention, one can use the *transferring* function
parameter annotation. Please see extensions below for more information about
*transferring*.

Since our transfer convention is weak, a disconnected isolation region that
was transferred into an isolation domain can be used again if the isolation
domain no longer maintains any references to the region. This occurs with
`nonisolated` asynchronous functions. When we transfer a disconnected value into
a `nonisolated` asynchronous functions, the value becomes part of the function's
task isolated isolation domain for the duration of the function's
execution. Once the function finishes executing, we know that the value is no
longer isolated to the function since:

* A `nonisolated` function does not have any non-temporary isolated state of its
  own that the non-`Sendable` value could escape into.

* Parameters in a task isolated isolation region cannot be transferred into a
  different isolation domain that does have persistant isolated state.

Thus the value in the caller's region again becomes disconnected once more and
thus can be used after the function returns and be transferred again:

```swift
func nonIsolatedCallee(_ x: NonSendable) async { ... }
func useValue(_ x: NonSendable) { ... }
@MainActor func transferToMainActor<T>(_ t: T) { ... }

actor MyActor {
  var state: NonSendable

  func example() async {
    // Regions: [{(), self}]

    let x = NonSendable()
    // Regions: [(x), {(), self}]

    // While nonIsolatedCallee executes the regions are:
    // Regions: [{(x), Task}, {(), self}]
    await nonIsolatedCallee(x)
    // Once it has finished executing, 'x' is disconnected again
    // Regions: [(x), {(), self}]

    // 'x' can be used since it is disconnected again.
    useValue(x) // (1)

    // 'x' can be transferred since it is disconnected again.
    await transferToMainActor(x) // (2)

    // Error! After transferring to main actor, permanently
    // in main actor, so we can't use it.
    useValue(x) // (3)
  }
}
```

In the example above, we transfer `x` into `nonIsolatedCallee` and while
`nonIsolatedCallee` is executing are not allowed to access `x` in the
caller. Since `nonIsolatedCallee`'s execution ends immediately after it is
called, we are then allowed to use `x` again.

### non-`Sendable` Closures

Currently non-`Sendable` closures like other non-`Sendable` values are not
allowed to be passed over isolation boundaries since they may have captured
state from within the isolation domain in which the closure is defined. We would
like to loosen these rules.

#### Captures

A non-`Sendable` closure's region is the merge of its non-`Sendable` captured
parameters. As such a nonisolated non-`Sendable` closure that only captures
values that are in disconnected regions must itself be in a disconnected region
and can be transferred:

```swift
let x = NonSendable()
// Regions: [(x)]
let y = NonSendable()
// Regions: [(x), (y)]
let closure = { useValues(x, y) }
// Regions: [(x, y, closure)]
await transferToMain(closure) // Ok to transfer!
// Regions: [{(x, y, closure), @MainActor}]
```

A non-`Sendable` closure that captures an actor-isolated value is considered to
be within the actor-isolated region of the value:

```swift
actor MyActor {
  var ns = NonSendable()

  func doSomething() {
    let closure = { print(self.ns) }
    // Regions: [{(closure, self.ns), self}]
    await transferToMain(closure) // Error! Cannot transfer value in actor region.
  }
}
```

When a non-`Sendable` value is captured by an actor-isolated non-`Sendable`
closure, we treat the value as being transferred into the actor isolation domain
since the value is now able to merged into actor-isolated state:

```swift
@MainActor var nonSendableGlobal = NonSendable()

func globalActorIsolatedClosureTransfersExample() {
  let x = NonSendable()
  // Regions: [(x), {(nonSendableGlobal), MainActor}]
  let closure = { @MainActor in
    nonSendableGlobal = x // Error! x is transferred into @MainActor and then accessed later.
  }
  // Regions: [{(nonSendableGlobal, x, closure), MainActor}]
  useValue(x) // Later access is here
}

actor MyActor {
  var field = NonSendable()
  
  func closureThatCapturesActorIsolatedStateTransfersExample() {
    let x = NonSendable()
    // Regions: [(x), {(nonSendableGlobal), MainActor}]
    let closure = {
      self.field.doSomething()
      x.doSomething() // Error! x is transferred into @MainActor and then accessed later.
    }
    // Regions: [{(nonSendableGlobal, x, closure), MainActor}]
    useValue(x) // Later access is here
  }
}
```

Importantly this ensures that APIs like `assumeIsolated` that take an
actor-isolated closure argument cannot introduce races by transferring function
parameters of nonisolated functions into an isolated closure:

```swift
actor ContainsNonSendable {
  var ns: NonSendableType = .init()

  nonisolated func unsafeSet(_ ns: NonSendableType) {
    self.assumeIsolated { isolatedSelf in
      isolatedSelf.ns = ns // Error! Cannot transfer a parameter!
    }
  }
}

func assumeIsolatedError(actor: ContainsNonSendable) async {
  let x = NonSendableType()
  actor.unsafeSet(x)
  useValue(x) // Race is here
}
```

Within the body of a non-`Sendable` closure, the closure and its non-`Sendable`
captures are treated as being Task isolated since just like a parameter, both
the closure and the captures may have uses in their caller:

```swift
var x = NonSendable()
var closure = {}
closure = {
  await transferToMain(x) // Error! Cannot transfer Task isolated value!
  await transferToMain(closure) // Error! Cannot transfer Task isolated value!
}
```

#### Transferring

A nonisolated non-`Sendable` synchronous or asynchronous closure that is in a
disconnected region can be transferred into another isolation domain if the
closure's region is never used again locally:

```swift
extension MyActor {
  func synchronousNonIsolatedNonSendableClosure() async {
    // This is non-Sendable and nonisolated since it does not capture MyActor or
    // any field of my actor.
    let nonSendable = NonSendable()
    let closure: () -> () = {
      print("I am in a closure: \(nonSendable.name)")
    }

    // We can safely transfer closure.
    await transferClosure(closure)

    // If we were to invoke closure again, an error diagnostic would be
    // emitted.
    closure() // Error!

    // If we were to access nonSendable, an error diagnostic would be
    // emitted.
    nonSendable.doSomething() // Error!
  }
}
```

An actor-isolated synchronous non-`Sendable` closure cannot be transferred to a
callsite that expects a synchronous closure. This is because as part of
transferring the closure, we have erased the specific isolation domain that the
closure was isolated to, so we cannot guarantee that we will invoke the value in
the actor's isolation domain:

```swift
@MainActor func transferClosure(_ f: () -> ()) async { ... }

extension Actor {
  func isolatedClosure() async {
    // This closure is isolated to actor since it captures self.
    let closure: () -> () = {
      self.doSomething()
    }

    // When we transfer the closure, we have lost the specific actor that
    // the closure belongs to so an error must be emitted!
    await transferClosure(closure) // Error!
  }
}
```

We may be able to accept this code in the future if we allowed for isolated
synchronous closures to propagate around the specific isolation domain that they
belonged to and dynamically swap to it. We discuss *dynamic isolation domains*
as an extension below.

In contrast, one can transfer an actor-isolated synchronous non-`Sendable`
closure at a call site that expects an asynchronous function argument. This is
because the closure will be wrapped into an asynchronous thunk that will hop
onto the defining isolation domain of the closure:

```swift
@MainActor func transferClosure(_ f: () async -> ()) async { ... }

extension Actor {
  func isolatedClosure() async {
    // This closure is isolated to actor since it captures self.
    let closure: () -> () = {
      self.doSomething()
    }

    // As part of transferring the closure, the closure is wrapped into an
    // asynchronous thunk that will hop onto the Actor's executor.
    await transferClosure(closure)
  }
}
```

In the example above, since the closure is wrapped in the asynchronous thunk and
that thunk hops onto the Actor's executor before calling the closure, we know
that isolation to the actor is preserved when we call the synchronous closure.

An actor-isolated asynchronous non-`Sendable` closure can be transferred since
upon the closure's invocation, we will always hop into the actor's isolation
domain:

```swift
extension Actor {
  func isolatedClosure() async {
    // This async closure is isolated to actor since it captures self.
    let closure: () async -> () = {
      self.doSomething()
    }

    // Since the closure is async, we can transfer it as much as we want
    // since we will always invoke the closure within the actor's isolation
    // domain...
    await transferClosure(closure)

    // ... so this is safe as well.
    await transferClosure(closure)
  }
}
```

#### Closures and Global Actors

If a closure uses values that are isolated from a global actor in any way, we
assume that the closure must also be isolated to that global actor:

```swift
@MainActor func mainActorUtility() {}

@MainActor func mainActorIsolatedClosure() async {
  let closure = {
    mainActorUtility()
  }
  // Regions: [{(closure), @MainActor}]
  await transferToCustomActor(closure) // Error!
}
```

If `mainActorUtility` was not called within `closure`'s body then `closure`
would be disconnected and could be transferred:

```swift
@MainActor func mainActorUtility() {}

@MainActor func mainActorIsolatedClosure() async {
  let closure = {
    ...
  }
  // Regions: [(closure)]
  await transferToCustomActor(closure) // Ok!
}
```

### KeyPath

A non-`Sendable` keypath that is not actor-isolated is considered to be
disconnected and can be transferred into an isolation domain as long as the
value's region is not reused again locally:

```swift
class Person {
  var name = "John Smith"
}

class Wrapper<Root: AnyObject> {
  var root: Root
  init(root: Root) { self.root = root }
  func setKeyPath<T>(_ keyPath: ReferenceWritableKeyPath<Root, T>, to value: T) {
    root[keyPath: keyPath] = value
  }
}

func useNonIsolatedKeyPath() async {
  let nonIsolated = Person()
  // Regions: [(nonIsolated)]
  let wrapper = Wrapper(root: nonIsolated)
  // Regions: [(nonIsolated, wrapper)]
  let keyPath = \Person.name
  // Regions: [(nonIsolated, wrapper, keyPath)]
  await transferToMain(keyPath) // Ok!
  await wrapper.setKeyPath(keyPath, to: "Jenny Smith") // Error!
}
```

A non-`Sendable` keypath that is actor-isolated is considered to be in the
actor's isolation domain and as such cannot be transferred out of the actor's
isolation domain:

```swift
@MainActor
final class MainActorIsolatedKlass {
  var name = "John Smith"
}

@MainActor
func useKeyPath() async {
  let actorIsolatedKlass = MainActorIsolatedKlass()
  // Regions: [{(actorIsolatedKlass.name), @MainActor}]
  let wrapper = Wrapper(root: actorIsolatedKlass)
  // Regions: [{(actorIsolatedKlass.name), @MainActor}]
  let keyPath = \MainActorIsolatedKlass.name
  // Regions: [{(actorIsolatedKlass.name, keyPath), @MainActor}]
  await wrapper.setKeyPath(keyPath, to: "value") // Error! Cannot pass non-`Sendable`
                                                 // keypath out of actor isolated domain.
}
```

If a KeyPath captures any values then the KeyPath's region consists of a merge
of the captured values regions combined with the actor-isolation region of the
KeyPath if the KeyPath is isolated to an actor:

```swift
class NonSendableType {
  subscript<T>(_ t: T) -> Bool { ... }
}

func keyPathInActorIsolatedRegionDueToCapture() async {
  let mainActorKlass = MainActorIsolatedKlass()
  // Regions: [{(mainActorKlass), @MainActor}]
  let keyPath = \NonSendableType.[mainActorKlass]
  // Regions: [{(mainActorKlass, keyPath), @MainActor}]
  await transferToMainActor(keyPath) // Error! Cannot transfer keypath in actor isolated region!
}

func keyPathInDisconnectedRegionDueToCapture() async {
  let ns = NonSendableType()
  // Regions: [(ns)]
  let keyPath = \NonSendableType.[ns]
  // Regions: [(ns, keyPath)]
  await transferToMainActor(ns)
  useValue(keyPath) // Error! Use of keyPath after transferring ns
}
```

### Async Let

When an async let binding is initialized with an expression that uses a
disconnected non-`Sendable` value, the value is treated as being transferred
into a `nonisolated` asynchronous callee that additionally allows for the value
to be transferred. If the value is used only by synchronous code and
`nonisolated` asynchronous functions, we allow for the value to be reused again
once the async let binding has been awaited upon:

```swift
func nonIsolatedCallee(_ x: NonSendable) async -> Int { 5 }

actor MyActor {
  func example() async {
    // Regions: [{(), self}]
    let x = NonSendable()
    // Regions: [(x), {(), self}]
    async let value = nonIsolatedCallee(x) + x.integerField
    // Regions: [{(x), Task}, {(), self}]
    useValue(x) // Error! Illegal to use x here.
    await value
    // Regions: [(x), {(), self}]
    useValue(x) // Ok! x is disconnected again so it can be used...
    await transferToMainActor(x) // and even transferred to another actor.
  }
}
```

If the disconnected value is transferred into an actor region, the value is
treated as if the value was transferred into the actor region at the point where
the async let is declared and is considered transferred even after the async let
has been awaited upon:

```swift
// Regions: []
let x = NonSendable()
// Regions: [(x)]
async let y = transferToMainActor(x) // Transferred here.
// Regions: [{(x), @MainActor}]
_ = await y
// Regions: [{(x), @MainActor}]
useValue(x) // Error! x is used after it has been transferred!
```

If a disconnected value is reused later in an async let initializer after
transferring it into an actor region, a use after transfer error diagnostic will
be emitted:

```swift
// Regions: []
let x = NonSendable()
// Regions: [(x)]
async let y =
  transferToMainActorAndReturnInt(x) +
  useValueAndReturnInt(x) // Error! Cannot use x after it has been transferred!
```

Since a disconnected value can only be transferred into one async let binding at
a time, a use after transfer diagnostic will be emitted if one initializes
multiple async let bindings in one statement with the same non-`Sendable`
disconnected value:

```swift
// Regions: []
let x = NonSendable()
// Regions: [(x)]
async let y = x,
          z = x // Error! Cannot use x after it has been transferred!
```

A non-`Sendable` value that is in an actor isolation region is never allowed to
be used to initialize an async let binding since values in an async let
binding's initializer are allowed to be transferred into further callees:

```swift
actor MyActor {
  var field = NonSendable()

  func example() async {
    // Regions: [{(self.field), self}]
    async let value = transferToMainActor(field) // Error! Cannot transfer actor
                                                 // isolated field to
                                                 // @MainActor!
    _ = await value
  }
}
```

### Using transferring to simplify `nonisolated` actor initializers and actor deinitializers

In [SE-0327](0327-actor-initializers.md), a flow sensitive diagnostic
was introduced to ensure that one can directly access stored properties of `self`
in `nonisolated` actor designated initializers and actor deinitializers despite
the methods not being isolated to self. The diagnostic set out a model where
initially `nonisolated` self is stated to have a weaker form of isolation that
relies on having exclusive access to self. While self is in that state, one is
allowed to access stored properties of self, but once self has escaped that
property is lost and self becomes nonisolated preventing one from accessing its
stored properties without using synchronization. In this proposal, we subsume
that proposal into the region based isolation model and eliminate the need for a
separate flow sensitive diagnostic.

In Swift's concurrency model, an actor is Sendable since one can only access the
actor's internal state from the actor's executor. If the actor is nonisolated to
the current function this implies one must hop on to the actor's executor to
safely access state. In the case of an initializer or deinitializer with
nonisolated self, this creates a conundrum since we explicitly want to
initialize or deinitialize self's stored fields without synchronizing by hopping
onto the actor's executor.

In order to implement these semantics, we model self as entering these methods
as a non-`Sendable` value that is strongly transferred into the method. Since
self is strongly transferred, we know that there cannot be any other references
in the program to self when the method begins executing and thus it is safe to
initially access the internal state of the actor directly. Self must initially
be a non-`Sendable` value since if self's storage can be accessed directly, then
passing self to another task could lead to a race on self's storage. To prevent
this possibility, when self escapes self becomes instantaneously
`Sendable`. Once self is `Sendable`, it is no longer safe to access self's
storage directly:

```swift
actor Actor {
  var nonSendableField: NonSendableType

  // self is passed into init using a strongly transferred convention. This means
  // that it is unique and safe to access without worrying about concurrency.
  init() {
    // At this point, self is non-Sendable and we can access its fields directly.
    self.nonSendableField = NonSendableType()

    // self is Sendable once callMethod is executed. This includes in callMethod itself.
    self.callMethod()

    // Error! Cannot directly access storage of a Sendable actor.
    self.nonSendableField.useValue()
  }
}
```

In the example above, self starts as a unique non-`Sendable` typed value. Thus
it is safe for us to initialize `self.nonSendableField`. When self is passed
into `callMethod`, self becomes `Sendable`. Since self could have been
transferred to another task by callMethod, it is no longer safe to directly
access self's memory and thus we emit an error when we access
`self.nonSendableField`.

Deinits work just like inits with one additional rule. Just like with initializers,
self is considered initially to be strongly transferred and non-`Sendable`. One
is allowed to access the `Sendable` stored properties of self while self is
non-`Sendable`. One can access the non-`Sendable` fields of self if one knows
statically that the non-`Sendable` fields are uniquely isolated to the self
instance. For the case of actors, this means that since the actor's state is
completely isolated only to that one actor instance we can touch non-`Sendable`
fields. But in the case of global actor isolated classes this is not true since
other global actor isolated class instances could also have a reference to the
same non-`Sendable` value since all global actor isolated instances are part of
the same isolation region:

```swift
actor Actor {
  var mutableNonSendableField: NonSendableType
  let immutableNonSendableField: NonSendableType
  var mutableSendableField: SendableType
  let immutableSendableField: SendableType

  deinit {
     _ = self.immutableSendableField // Ok
     _ = self.mutableSendableField // Ok
     // Safe to access since no other actor instances
     _ = self.mutableNonSendableField // Ok
     _ = self.immutableNonSendableField // Ok

     escapeSelfIntoNonIsolated(self)

     _ = self.immutableSendableField // Ok
     _ = self.mutableSendableField // Error! Must be immutable.
     _ = self.mutableNonSendableField // Error! Must be sendable
     _ = self.immutableNonSendableField // Error! Must be sendable
  }
}

@MainActor class GlobalActorIsolatedClass {
  var mutableNonSendableField: NonSendableType
  let immutableNonSendableField: NonSendableType
  var mutableSendableField: SendableType
  let immutableSendableField: SendableType

  deinit {
     _ = self.immutableSendableField // Ok
     _ = self.mutableSendableField // Ok
     _ = self.mutableNonSendableField // Error! Must be sendable!
     _ = self.immutableNonSendableField // Error! Must be sendable!

     escapeSelfIntoNonIsolated(self)

     _ = self.immutableSendableField // Ok
     _ = self.mutableSendableField // Error! Must be immutable!
     _ = self.mutableNonSendableField // Error! Must be sendable!
     _ = self.immutableNonSendableField // Error! Must be sendable!
  }
}
```

### Using transferring to pass non-Sendable values to async isolated actor initializers

In [SE-0327](0327-actor-initializers.md), all initializers with non-`Sendable`
arguments were only allowed to be called by delegating initializers:

```swift
actor MyActor {
  var x: NonSendableType

  // Can call this from anywhere.
  init(_ arg: SendableType) {
    self.init(NonSendableType(arg))
  }

  // Since this has a non-Sendable type, this designated initializer can only
  // be called by other initializers like the delegating init above.
  init(_ arg: NonSendableType) {
    x = arg
  }
}

func constructActor() {
  // Error! Cannot call init with non-`Sendable` argument from outside of
  // MyActor.
  let a = Actor(NonSendableType())
}
```

Using isolation regions we can loosen this restriction and allow for
non-`Sendable` types to be passed to asynchronous initializers since our region
isolation rules guarantee that the caller will have transferred the value into
the initializer due to the isolation boundary:

```swift
actor MyActor {
  var x: NonSendableType
  
  init(_ arg: NonSendableType) async {
    self.x = arg
  }
}

func makeActor() async -> MyActor {
  // Regions: []
  let x = NonSendableType()
  // Regions: [(x)]
  let a = await MyActor(x) // Ok!
  // Regions: [{(x), a}]
  return a
}
```

In the above example, it is safe to pass `x` into `MyActor` despite `x` being
non-`Sendable` since if we were to use `x` afterwards, the compiler would error
since we would be using `x` from multiple isolation domains:

```swift
func makeActor() async -> MyActor {
  // Regions: []
  let x = NonSendableType()
  // Regions: [(x)]
  let a = await MyActor(x) // Ok!
  // Regions: [{(x), a}]
  x.doSomething() // Error! 'x' was transferred to a's isolation domain!
  return a
}
```

Sadly synchronous initializers without additional work can still only take
`Sendable` types since there is not a guarantee that the non-`Sendable` types
that are passed to it is in its own region. In order to pass a non-`Sendable`
type to a synchronous initializer, one must mark the parameter with the
`transferring` function parameter modifier which is described below in [Future
Directions](#transferring-parameters).

### Regions Merge when assigning to Struct and Tuple type var like bindings

In this proposal, regions are not computed in a field sensitive manner. This
means that if we assign into a struct with multiple stored fields or a tuple
with multiple fields then assigning to one field affects the region of the
entire struct and requires us to merge into such types rather than assign since
otherwise we would lose the regions associated with the other fields:

```swift
struct NonSendableBox {
  var s1 = NonSendable()
  var s2 = NonSendable()
}

func mergeWhenAssignIntoMultiFieldStructField() async {
  var box = NonSendableBox()
  // Regions: [(box.s1, box.s1)]
  let x = NonSendable()
  // Regions: [(box.s1, box.s2), (x)]
  let y = NonSendable()
  // Regions: [(box.s1, box.s2), (x), (y)]
  box.s1 = x
  // Regions: [(box.s1, box.s2, x), (y)]
  // If we used an assignment operation instead of a merge operation,
  // this would cause us to lose that x was still in box.s1 and thus
  // in box's region.
  box.s2 = y
  // Regions: [(box.s1, box.s2, x, y)]
}
```

In the above example, if we were to treat ``box.s2 = y`` as an assignment
instead of merge then we would be removing ``x`` from ``box``'s region which
would be unsound since ``x`` and ``box.s1`` still point at the same
reference. Unfortunately this has the affect that when we overwrite an element
of a var like struct, the previous region assigned to that field would have to
remain in the overall struct/tuple's region:

```swift
func mergeWhenAssignIntoMultiFieldTupleField() async {
  var box = (NonSendable(), NonSendable())
  // Regions: [(box.0, box.1)]
  let x = NonSendable()
  // Regions: [(box.0, box.1), (x)]
  let y = NonSendable()
  // Regions: [(box.0, box.1), (x), (y)]
  box.0 = x
  // Regions: [(box.0, box.1, x), (y)]
  box.0 = y                               (1)
  // Regions: [(box.0, box.1, x, y)]
}
```

In the above, even though we reassign ``box.0`` from ``x`` to ``y``, since we
must perform a merge, we must have that ``x`` is still in ``box``'s region. If
one assigns over the entire box though, one can still get an assign instead of a
region:

```swift
func mergeWhenAssignIntoMultiFieldTupleField2() async {
  var box = (NonSendable(), NonSendable())
  // Regions: [(box.0, box.1)]
  let x = NonSendable()
  // Regions: [(box.0, box.1), (x)]
  let y = NonSendable()
  // Regions: [(box.0, box.1), (x), (y)]
  box.0 = x
  // Regions: [(box.0, box.1, x), (y)]
  box = (y, NonSendable())
  // Regions: [(box.0, box.1, y), (x)]
}
```

In order to mitigate this, we are able to be stricter with structs and tuples
that store a single field. In such a case, since the struct/tuple does not have
multiple fields updating the single field does not cause us to lose the region
of any other values:

```swift
func assignWhenAssignIntoSingleFieldStruct() async {
  var box = SingleFieldBox()
  // Regions: [(box.field)]
  let x = NonSendable()
  // Regions: [(box.field), (x)]
  let y = NonSendable()
  // Regions: [(box.field), (x), (y)]
  box.field = x
  // Regions: [(box.field, x), (y)]
  box.field = y
  // Regions: [(box.field, y), (x)]
}
```

### Accessing `Sendable` fields of non-`Sendable` types after weak transferring

Given a non-`Sendable` value `x` that has been weakly transferred, a `Sendable`
field `x.f` can be accessed in the caller after `x`'s transferring if the
compiler can statically prove that there cannot be any writes to `x.f` from
another concurrency domain. This is necessary since although `x.f` is
`Sendable`, if code from another concurrency domain can reference `x` in a
manner that allows for `x.f` to be written to, our initial access to `x.f` could
result in a race. Of course once the access is over, we are safe against races
due to the Sendability of `x.f`'s underlying type. The situations where this
occurs varies in between reference types and value types. We go through the
individual cases below.

#### Classes

If `x` is a reference type like a class, we only allow for `Sendable` let fields
of `x` to be accessed. This is safe since a let field can never be modified
after initialization implying that we cannot race on assignment to the field
when attempting to read from the field. We cannot allow for `Sendable` var
fields to be accessed due to the aforementioned possible race caused by another
concurrency domain writing to the `Sendable` field as we attempt to access it:

```swift
class NonSendable {
  let letSendable: SendableType
  var varSendable: SendableType
  let ns: NonSendable
}

@MainActor func modifyOnMainActor(_ x: NonSendable) async {
  x.varSendable = SendableType()
}

func example() async {
  let x = NonSendable()
  await modifyOnMainActor(x)
  _ = x.letSendable // This is safe.
  _ = x.varSendable // Error! Use after transfer of mutable field that could
                    // race with a write to x.varSendable in modifyOnMainActor.
}
```

#### Immutable Bindings to Value Types

If `x` is an immutable binding (e.x.: let) to a value type (e.x.: struct, tuple,
enum) then we allow for access to all of `x`'s `Sendable` subtypes. This is safe
because:

1. `x` will be initialized by copying its initial value. This means that even if
   `x`'s initial value is a field of a larger value, any modifications to the
   other value will not cause `x`'s fields to point to different values.

2. When `x` is transferred to a callee, `x` will be passed by value. Thus the
   callee will recieve a completely new value type albeit with copied
   fields. This means that if the callee attempts to modify the value, it will
   be modifying the new value instead of our caller value implying that we
   cannot race against any assignment when accessing the field in our
   caller.
   
   ```swift
   struct NonSendableStruct {
     let letSendableField: Sendable
     var varSendableField: Sendable
     let ns: NonSendable
   }
   
   @MainActor func modifyOnMainActor(_ y: consuming NonSendableStruct) async {
     // These assignments only affect our parameter, not x in the callee.
     y.varSendableField = Sendable()
     y = NonSendableStruct()
   }
   
   func letExample() async {
     let x = NonSendableStruct()
   
     await modifyOnMainActor(x) // Transfer x, giving useValueOnMainActor a
                                // shallow copy of x.
   
     // We do not race with the assignment in modifyOnMainActor since the
     // assignment is to y, not to x. Since the fields are sendable, once
     // we avoid the race on accessing the field, we are safe.
     print(x.letSendableField)
     print(x.varSendableField)
   }
   ```

3. If `x` is captured by reference, since `x` is a let it will be captured
   immutably implying that we cannot write to `x.f`.

#### Mutable Bindings to Value Types

If `x` is a mutable binding (e.x.: `var`), then we can follow the same logic as
with our immutable bindings except in the case where `x` is captured by
reference. If `x` is captured by reference, it is captured mutably implying that
when accessing `x.f`, we could race against an assignment to `x.f` in the
closure:

```swift
struct NonSendableStruct {
  let letSendableField: Sendable
  var varSendableField: Sendable
  let ns: NonSendable
}

@MainActor func invokeOnMain(_ f: () -> ()) async {
  f()
}

func unsafeMutableReferenceCaptureExample() async {
  var x = NonSendableStruct()
  let closure = {
    x = NonSendableStruct(otherInit: ())
  }
  await invokeOnMain(closure)

  _ = x.letSendableField // Error! Could race against write in closure!
  _ = x.varSendableField // Error! Could race against write in closure!
}
```

This also implies that one cannot access `Sendable` computed properties or
functions later since those routines could perform a read like the above
resulting in a race against a write in the closure.

## Source compatibility

This proposal strictly expands the set of acceptable Swift programs, so all
Swift code that was previously accepted by the compiler will still be accepted.

## ABI compatibility

This has no affect on ABI.

## Future directions

### Transferring Parameters

In the above, we mentioned that the transferring of non-`Sendable` values as
discussed above is a callee side property since when analyzing an async callee,
we do not know if the callee's caller is from a different isolation domain or
not. This means that we must be conservative and treat all function parameters as
being in the same region and prevent transferring of function parameters.

We could introduce a stronger form of transferring that is applied to a function
argument in the callee's signature and forces all callers to transfer the
parameter even if the caller is synchronous or is async but in the same
isolation domain.

The transferred parameter is guaranteed to be strongly transferred so we know
that once the callee is called there are no other program visible references to
the value outside of the callee's parameter. The implications of this are:

* Since the value is strongly isolated, it will be within its own disconnected
  region separate from the regions of the other parameters:

  ```swift
  actor Actor {
    func method(_ x: transferring NonSendable,
                _ y : NonSendable,
                _ z : NonSendable) async {
      // Regions: [(x), {(y, z), self}]
      // Safe to transfer x since x is marked as transferring.
      await transferToMainActor(x)
    }
  }
  ```


* Regardless of if the callee is synchronous or asynchronous, a non-`Sendable`
  value that is passed as a transferring parameter cannot be used again locally.

  ```swift
  actor Actor {
    func transfer<T>(_ t: transferring T) async {}
    func method() async {
      let a = NonSendable()

      // Pass a into transfer. Even though we are in the same
      // isolation domain as transfer...
      await transfer(a)

      // Since we transferred a, we are no longer allowed to use a here. Error!
      useValue(a)
    }
  }
  ```

* Given an asynchronous function, one can safely transfer the non-`Sendable`
  parameter to another asynchronous function with a different isolation domain:
  
  ```swift
  @MainActor func transferToMainActor<T>(_ t: T) async {}
  
  actor Actor {
    func method(_ x: transferring NonSendable) async {
      // Regions: [(x)]
      // Safe to transfer x since x is marked as transferring.
      await transferToMainActor(x)
    }
  }
  ```
  
* Given a transferring parameter of a synchronous function, the parameter's
  strongly isolated implies that we can transfer it into `Task.init` or
  `Task.detach`.
  
  ```swift
  func someSynchronousFunction(_ x: transferring NonSendable) {
    Task {
      doSomething(x)
    }
  }
  ```
  
  if we did not have the strong isolation, then `x` could still be used in the
  caller of `someSynchronousFunction`.

* Due to the isolation of a transferring parameter, it is legal to have a
  non-`Sendable` transferring parameter of a synchronous actor designated
  initializer:
  
  ```swift
  actor Actor {
    var field: NonSendable

    init(_ x: transferring NonSendable) {
      self.field = x
    }
  }
  ```

  Without the transferring argument modifier on `x`, it would not be safe to
  store `x` into `self.field` since it may be introducing a value into the
  actor's state that could be raced upon.

#### Returns Isolated

As discussed above, if a function takes non-`Sendable` parameters and has a
non-`Sendable` result, then the result is part of the merged region of the
function's parameters. This is not always the appropriate semantics since there
are APIs whose results will be in different regions than their parameters. As an
example of this, consider a function that performs control flow based off of
non-`Sendable` state and then returns a result:

```swift
func example(_ x: NonSendable) async -> NonSendable? {
  if x.boolean {
    return NonSendable()
  }
  return nil
}
```

In the above, the result of `example` is a newly constructed value that has no
data dependence on the parameter `x`, but as laid out in this proposal, we
cannot express this. We propose the addition of a new function parameter
modifier called `returnsIsolated` that causes callers to treat the result of a
function as being in a disconnected region regardless of the inputs. As part of
this annotation, we would only allow for the callee to return a value that is in
a disconnected region preventing the returning of function arguments or in the
case of an actor any state related internally to the actor:

```swift
actor Actor {
  var field: NonSendableType
  
  func getValue() -> @returnsIsolated NonSendableType {
    // Regions: [{(self.field), self}]
    let x = NonSendableType()
    // Regions: [(x), {(self.field), self}]
    
    if await booleanValue {
      // Safe to do since 'x' is in a disconnected region.
      return x
    }
    
    // Error! Cannot return a value from the actor's region!
    return field
  }
}
```

Since the value returned is always in its own disconnected region, it can be
used in the caller isolation domain without triggering races:

```swift
func getValueFromActor(_ a: Actor) async {
  // Regions: [{(a.field), a}]
  
  // This is safe since we know that 'x' is independent of the actor.
  let x = await a.getValue()
  // Regions: [(x), {(a.field), a}]
  
  // So we could transfer it to another function if we wanted to.
  await transferToMainActor(x)
}
```

> NOTE: @returnsIsolated is just a strawman syntax introduced for the purpose of
> expositing this extension. It is not an actual proposed or final syntax.

#### Disconnected Fields and the Disconnect Operator

Even though we can use `@returnsIsolated` to return a value from the Actor's
isolation domain, we have not specified a manner to safely return non-`Sendable`
values from the internal state of an Actor or GAIT. To do so, we introduce a new
type of field called a *disconnected field*. A disconnected field of an actor is
an actor isolated region that is separate from the normal actor's region. Since
it is separate from the other region of the actor, it cannot be reachable by the
other fields of the actor... but since it is an actor field, it cannot be
escaped from the actor without doing additional work. In order to escape such a
field, we introduce a new `disconnect` operation that consumes the disconnected
field and returns the field's value as a new disconnected region which is safe
to use as a `@returnsIsolated` result:

```swift
actor MyActor {
  disconnected var x: NonSendableType

  /// Reinitialize a field, returning the old value.
  func reinitField() -> @returnsIsolated NonSendableType {
    let result = disconnect x
    x = NonSendableType()
    return result
  }
}
```

In the above example, we disconnect `x`'s value into `result`, reinitialize `x`
with a fresh value, and return the result.

> NOTE: We may be able to reuse the `consume` operator for this purpose, but for
> the purposes of framing this as an extension, we introduce a new operator for
> simplicity.

If the author forgets to update the disconnected field with a new value, a
control flow sensitive error will be emitted:

```swift
actor MyActor {
  disconnected var x: NonSendableType

  func reinitField() -> @returnsIsolated NonSendableType {
    let result = disconnect x

    if booleanTest {
      x = newValue
    } else {
      ...
    }

    return result
  } // Error! Must update disconnected field 'x' along all program paths after disconnecting!
}
```

In the above example, we emit an error since along the else path we do not
provide a new value for `x`.

Since a disconnected field can only be initialized with a value from a
disconnected region implying that a field cannot be assigned to by a parameter
of an actor method unless the parameter is transferred:

```swift
actor MyActor {
  disconnected var x: NonSendableType

  /// Update the internal state to use a new value, returning the old value
  func updateValue(_ newValue: transferring NonSendableType) -> @returnsIsolated NonSendableType {
    let result = disconnect x
    x = newValue
    return result
  }
}
```

since the parameter in the above example is transferred, it has a disconnected
region and thus can be assigned into the disconnected region.

## Alternatives considered

### Require users to audit all types for sendability

We could require users to audit all of their non-`Sendable` types for
Sendability. This would create a large annotation burden on users that this
approach avoids.

### Force weak transferring to be explicitly marked

We could require transferred arguments to be explicitly marked with an operator
like consume or transfer. This is not needed since the APIs in question are
already explicitly marked as being a point of concurrency via `async`, `await`,
or `Task` implying that whether or not an API can result in transferring is
already explicitly marked. The only information that requiring an additional
explicit marker would provide the user is that the programmer can know without
reading the API surface that a transfer will occur here, information that can
also be ascertained by just reading the source.

## Acknowledgments

This proposal is based on work from the PLDI 2022 paper (A Flexible Type System for Fearless Concurrency)[https://www.cs.cornell.edu/andru/papers/gallifrey-types/].

Thanks to Doug Gregor, Kavon Farvardin for early assistance to Joshua during his
internship.

Thanks to Doug Gregor and Holly Borla for our stimulating discussions and to
Holly for her help with editing!

## Appendix

### Isolation Region Dataflow

The dataflow for computing *isolation regions* is defined as follows:

1. The lattice of the dataflow consists of graphs where each value is a node and
   each edge represents a statement that causes two values to be apart of the
   same region. We partially order our lattice by stating that given a graph
   `g1` and a graph `g2` then `g1 <= g2` only if `g1 U g2 = g1` where `U` is a
   graph union operation.

2. Control flow merges are defined by unions of graphs meaning that if there is
   an edge in between two nodes in any predecessor control flow blocks, there is
   an edge in the successor control flow block.

3. We consider the top of the dataflow to be the empty graph consisting of
   values that are all in their own independent regions and the bottom of our
   dataflow to be a completely connected graph where all values are in the same
   region.

4. Since the dataflow is a forward optimistic dataflow, we initially treat
   backedges as propagating the top graph.

5. We can prove that our dataflow always converges since our transfer function
   can be proven as monotonic since given two sets `g1`, `g2` with `g1 <= g2`,
   we know that `F(g1) <= F(g2)` since any edges that we remove from `g1` must
   also be removed from `g2` and any edges that we add will be added identically
   to `g1` and `g2` since `g1` is a subset of `g2`.
