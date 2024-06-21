# Enforce Exclusive Access to Memory

* Proposal: [SE-0176](0176-enforce-exclusive-access-to-memory.md)
* Author: [John McCall](https://github.com/rjmccall)
* Review Manager: [Ben Cohen](https://github.com/airspeedswift)
* Status: **Implemented (Swift 4.0)**
* Previous Revision: [1](https://github.com/swiftlang/swift-evolution/blob/7e6816c22a29b0ba9bdf63ff92b380f9e963860a/proposals/0176-enforce-exclusive-access-to-memory.md)
* Previous Discussion: [Email Thread](https://forums.swift.org/t/review-se-0176-enforce-exclusive-access-to-memory/5836)

## Introduction

In Swift 3, it is possible to modify a variable while it's being used or
modified by another part of the program.  This can lead to unexpected and
confusing results.  It also forces a great deal of conservatism onto the
implementation of the compiler and the standard libraries, which must
generally ensure the basic soundness of the program (no crashes or
undefined behavior) even in unusual circumstances.

We propose that Swift should instead enforce a general rule that potential
modifications of variables must be exclusive with any other access to that
variable.

This proposal is a core part of the Ownership feature, which was described
in the [ownership manifesto](https://github.com/apple/swift/blob/master/docs/OwnershipManifesto.md).
That document presents the high-level objectives of Ownership, develops
a more rigorous theory of memory access in Swift, and applies it in detail
to a variety of different language features.  In that document, the rule
we're proposing here is called the Law of Exclusivity.  We will not be
going into that level of detail in this proposal.  Instead, we will
lay out the basic rule, how it will be enforced, and the implications for
programming in Swift.  It should be possible to understand this proposal
without actually having read the ownership manifesto at all.  That said,
if you are interested in the technical details, that document is probably
the right place to turn.

## Motivation

### Instantaneous and non-instantaneous accesses

On a basic level, Swift is an imperative language which allows programmers
to directly access mutable memory.

Many of the language features that access memory, like simply loading
from or assigning to a variable, are "instantaneous".  This means that,
from the perspective of the current thread, the operation completes
without any other code being able to interfere.  For example, when you
assign to a stored property, the current value is just replaced with
the new value.  Because arbitrary other code can't run during an
instantaneous access, it's never possible for two instantaneous accesses
to overlap each other (without introducing concurrency, which we'll
talk about later).  That makes them very easy to reason about.

However, not all accesses are instantaneous.  For example, when you
call a ``mutating`` method on a stored property, it's really one long
access to the property: ``self`` just becomes another way of referring
to the property's storage.  This access isn't instantaneous because
all of the code in the method executes during it, so if that code
manages to access the same property again, the accesses will overlap.
There are several language features like this already in Swift, and
Ownership will add a few more.

### Examples of problems due to overlap

Here's an example:

```swift
// These are simple global variables.
var global: Int = 0
var total: Int = 0

extension Int {
  // Mutating methods access the variable they were called on
  // for the duration of the method.
  mutating func increaseByGlobal() {
    // Any accesses they do will overlap the access to that variable.

    total += self // Might access 'total' through both 'total' and 'self'
    self += global // Might access 'global' through both 'global' and 'self'
  }
}
```

If ``self`` is ``total`` or ``global``, the low-level semantics of this
method don't change, but the programmer's high-level understanding
of it almost certainly does.  A line that superficially seems to not
change 'global' might suddenly start doubling it!  And the data dependencies
between the two lines instantly go from simple to very complex.  That's very
important information for someone maintaining this method, who might be
tempted to re-arrange the code in ways that seem equivalent.  That kind of
maintenance can get very frustrating because of overlap like this.

The same considerations apply to the language implementation.
The possibility of overlap means the language has to make
pessimistic assumptions about the loads and stores in this method.
For example, the following code avoids a seemingly-redundant load,
but it's not actually equivalent because of overlap:

```swift
    let value = self
    total += value
    self = value + global
```

Because these variables just have type ``Int``, the cost of this pessimism
is only an extra load.  If the types were more complex, like ``String``,
it might mean doing extra copies of the ``String`` value, which would
translate to extra retains and releases of the string's buffer; in a more
complex example, that could even lead to the underlying data being copied
unnecessarily.

In the above examples, we've made the potentially-overlapping accesses
obvious, but they don't have to be.  For example, here is another method
that takes a closure as an argument:

```swift
extension Array {
  mutating func modifyElements(_ closure: (inout Element) -> ()) {
    var i = startIndex
    while i != endIndex {
      closure(&self[i])
      i = index(after: i)
    }
  }
}
```

This method's implementation seems straightforwardly correct, but
unfortunately it doesn't account for overlap.  Absolutely nothing
prevents the closure from modifying ``self`` during the iteration,
which means that ``i`` can suddenly become an invalid index, which
could lead to all sorts of unwanted behavior.  Even if this never
happen in reality, the fact that it's *possible* means that the
implementation is blocked from pursuing all sorts of important
optimizations.

For example, the compiler has an optimization that "hoists" the
uniqueness check on a copy-on-write collection from the inside of
a loop (where it's run on each iteration) to the outside (so that
it's only checked once, before the loop begins).  But that optimization
can't be applied in this example because the closure might change or
copy ``self``.  The only realistic way to tell the compiler that
that can't happen is to enforce exclusivity on ``self``.

The same considerations that apply to ``self`` in a ``mutating``
method also apply to ``inout`` parameters.  For example:

```swift
open class Person {
  open var title: String
}

func collectTitles(people: [Person], into set: inout Set<String>) {
  for person in people {
    set.insert(person.title)
  }
}
```

This function mutates a set of strings, but it also repeatedly
calls a class method.  The compiler cannot know how this method
is implemented, because it is ``open`` and therefore overridable
from an arbitrary module.  Therefore, because of overlap, the
compiler must pessimistically assume that each of these method
calls might somehow find a way to modify the original variable
that ``set`` was bound to.  (And if the method did manage to do
so, the resulting strange behavior would probably be seen as a bug
by the caller of ``collectTitles``.)

### Eliminating non-instantaneous accesses?

If non-instantaneous accesses create all of these problems with
overlapping accesses, should we just eliminate non-instantaneous
accesses completely?  Well, no, and there's two big reasons why not.
In order to make something like a ``mutating`` method not access
the original storage of ``self`` for the duration of the method,
we would need to make it access a temporary copy instead, which
we would assign back to the storage after the method is complete.
That is, suppose we had the following Swift code:

```swift
var numbers = [Int]()
numbers.appendABunchOfStuff()
```

Currently, behind the scenes, this is implemented somewhat like
the following C code:

```c
struct Array numbers = _Array_init();
_Array_appendABunchOfStuff(&numbers);
```

You can see clearly how ``_Array_appendABunchOfStuff`` will be working
directly with the storage of ``numbers``, creating the abstract
possibility of overlapping accesses to that variable.  To prevent
this in general, we would need to pass a temporary copy instead:

```c
struct Array numbers = _Array_init();
struct Array temp = _Array_copy(numbers);
_Array_appendABunchOfStuff(&temp);
_Array_assign(&numbers, temp);
```

Like we said, there's two big problems with this.

The first problem is that it's awful for performance.  Even for a
normal type, doing extra copies is wasteful, but doing it with ``Array``
is even worse because it's a copy-on-write type.  The extra copy here
means that there will be multiple references to the buffer, which
means that ``_Array_appendABunchOfStuff`` will be forced to copy the
buffer instead of modifying it in place.  Removing these kinds of
copies, and making it easier to reason about when they happen, is a
large part of the goal of the Ownership feature.

The second problem is that it doesn't even eliminate the potential
confusion.  Suppose that ``_Array_appendABunchOfStuff`` somehow
reads or writes to ``numbers`` (perhaps because ``numbers`` is
captured in a closure, or it's actually a global variable or a class
property or something else that can be potentially accessed from
anywhere).  Because the method is now modifying the copy in ``temp``,
any reads it makes from ``numbers`` won't see any of the changes
it's made to ``temp``, and any changes it makes to ``numbers`` will
be silently lost when it returns and the caller unconditionally
overwrites ``numbers`` with ``temp``.

### Consequences of non-instantaneous accesses

So we have to accept that accesses can be non-instantaneous.  That
means programmers can write code that would naturally cause overlapping
accesses to the same variable.  We currently allow this to happen
and make a best effort to live with the consequences.  The costs,
in general, are a lot of complexity and lost performance.

For example, the ``Array`` type has an optimization in its ``subscript``
operator which allows callers to directly access the storage of array
elements.  This is a very important optimization which, among other
things, allows arrays to efficiently hold values of copy-on-write types.
However, because the caller can execute arbitrary code while they're
working with the array element storage, and that code might do something
like assign a new value to the original array variable and therefore drop
the last reference to the array buffer, this optimization has to create a
new strong reference to the buffer until the caller is done with the element,
which itself causes a whole raft of complexity.

Similarly, when the compiler is optimizing a ``mutating`` method, it has
to assume that an arbitrary call might completely rewrite ``self``.
This makes it very difficult to perform any meaningful optimization
at all, especially in generic code.  It also means that the compiler
must generally emit a large number of conservative copies just in case
things are modified in unexpected ways.

Furthermore, the possibility of overlapping accesses has a continued
impact on language evolution.  Many of the features laid out in the
Ownership manifesto rely on static guarantees that Swift simply cannot
make without stronger rules about when a variable can be modified.

Therefore we think it best to simply disallow overlapping accesses
as best as we can.

## Proposed solution

We should add a rule to Swift that two accesses to the same variable
are not allowed to overlap unless both accesses are reads.  By
"variable", we mean any kind of mutable memory: global variables,
local variables, class and struct properties, and so on.

This rule should be enforced as strongly as possible, depending on
what sort of variable it is:

* Local variables, inout parameters, and struct properties can
  generally enforce the rule statically.  The compiler can analyze
  all the accesses to the variable and emit an error if it sees
  any conflicts.

* Class properties and global variables will have to enforce the
  rule dynamically.  The runtime can keep track of what accesses
  are underway and report any conflicts.  Local variables will
  sometimes have to use dynamic enforcement when they are
  captured in closures.

* Unsafe pointers will not use any active enforcement; it is the
  programmer's responsibility to follow the rule.

* No enforcement is required for immutable memory, like a ``let``
  binding or property, because all accesses must be reads.

Examples:

```swift
var x = 0, y = 0

// NOT A CONFLICT.  These two accesses to 'x' are both reads.
// Each completes instantaneously, so the accesses do not overlap and
// therefore do not conflict.  Even if they were not instantaneous, they
// are both reads and therefore do no conflict.
let z = x + x

// NOT A CONFLICT.  The right-hand side of the assignment is a read of
// 'x' which completes instantaneously.  The assignment is a write to 'x'
// which completes instantaneously.  The accesses do not overlap and
// therefore do not conflict.
x = x

// NOT A CONFLICT.  The right-hand side is a read of 'x' which completes
// instantaneously.  Calling the operator involves passing 'x' as an inout
// argument; this is a write access for the duration of the call, but it does
// not begin until immediately before the call, after the right-hand side is
// fully evaluated.  Therefore the accesses do not overlap and do not conflict.
x += x

// CONFLICT.  Passing 'x' as an inout argument is a write access for the
// duration of the call.  Passing the same variable twice means performing
// two overlapping write accesses to that variable, which therefore conflict.
swap(&x, &x)

extension Int {
  mutating func assignResultOf(_ function: () -> Int) {
    self = function()  
  }
}

// CONFLICT.  Calling a mutating method on a value type is a write access
// that lasts for the duration of the method.  The read of 'x' in the closure
// is evaluated while the method is executing, which means it overlaps
// the method's formal access to 'x'.  Therefore these accesses conflict.
x.assignResultOf { x + 1 }
```

## Detailed design

### Concurrency

Swift has always considered read/write and write/write races on the same
variable to be undefined behavior.  It is the programmer's responsibility
to avoid such races in their code by using appropriate thread-safe
programming techniques.

We do not propose changing that.  Dynamic enforcement is not required to
detect concurrent conflicting accesses, and we propose that by default
it should not make any effort to do so.  This should allow the dynamic
bookkeeping to avoid synchronizing between threads; for example, it
can track accesses in a thread-local data structure instead of a global
one protected by locks.  Our hope is that this will make dynamic
access-tracking cheap enough to enable by default in all programs.

The implementation should still be *permitted* to detect concurrent
conflicting accesses, of course.  Some programmers may wish to use an
opt-in thread-safe enforcement mechanism instead, at least in some
build configurations.

Any future concurrency design in Swift will have the elimination
of such races as a primary goal.  To the extent that it succeeds,
it will also define away any specific problems for exclusivity.

### Value types

Calling a method on a value type is an access to the entire value:
a write if it's a ``mutating`` method, a read otherwise.  This is
because we have to assume that a method might read or write an
arbitrary part of the value.  Trying to formalize rules like
"this method only uses these properties" would massively complicate
the language.

For similar reasons, using a computed property or subscript on a
value type generally has to be treated as an access to the entire
value.  Whether the access is a read or write depends on how the
property/subscript is used and whether either the getter or the setter
is ``mutating``.

Accesses to different stored properties of a ``struct`` or different
elements of a tuple are allowed to overlap.  However, note that
modifying part of a value type still requires exclusive access to
the entire value, and that acquiring that access might itself prevent
overlapping accesses.  For example:

```swift
struct Pair {
  var x: Int
  var y: Int
}

class Paired {
  var pair = Pair(x: 0, y: 0)
}

let object = Paired()
swap(&object.pair.x, &object.pair.y)
```

Here, initiating the write-access to ``object.pair`` for the first
argument will prevent the write-access to ``object.pair`` for the
second argument from succeeding because of the dynamic enforcement
used for the property.  Attempting to make dynamic enforcement
aware of the fact that these accesses are modifying different
sub-components of the property would be prohibitive, both in terms
of the additional performance cost and in terms of the complexity
of the implementation.

However, this limitation can be worked around by binding
``object.pair`` to an ``inout`` parameter:

```swift
func modifying<T>(_ value: inout T, _ function: (inout T) -> ()) {
  function(&value)
}

modifying(&object.pair) { pair in swap(&pair.x, &pair.y) }
```

This works because now there is only a single access to
``object.pair`` and because, once the the ``inout`` parameter is
bound to that storage, accesses to the parameter within the
function can use purely static enforcement.

We expect that workarounds like this will only rarely be required.

Note that two different properties can only be assumed to not
conflict when they are both known to be stored.  This means that,
for example, it will not be allowed to have overlapping accesses
to different properties of a resilient value type.  This is not
expected to be a significant problem for programmers.

### Arrays

Collections do not receive any special treatment in this proposal.
For example, ``Array``'s indexed subscript is an ordinary computed
subscript on a value type.  Accordingly, mutating an element of an
array will require exclusive access to the entire array, and
therefore will disallow any other simultaneous accesses to the
array, even to different elements.  For example:

```swift
var array = [[1,2], [3,4,5]]

// NOT A CONFLICT.  These accesses to the elements of 'array' each
// complete instantaneously and do not overlap each other.  Even if they
// did overlap for some reason, they are both reads and therefore
// do not conflict.
print(array[0] + array[1])

// NOT A CONFLICT.  The access done to read 'array[1]' completes
// before the modifying access to 'array[0]' begins.  Therefore, these
// accesses do not conflict.
array[0] += array[1]

// CONFLICT.  Passing 'array[i]' as an inout argument performs a
// write access to it, and therefore to 'array', for the duration of
// the call.  This call makes two such accesses to the same array variable,
// which therefore conflict.
swap(&array[0], &array[1])

// CONFLICT.  Calling a non-mutating method on 'array[0]' performs a
// read access to it, and thus to 'array', for the duration of the method.
// Calling a mutating method on 'array[1]' performs a write access to it,
// and thus to 'array', for the duration of the method.  These accesses
// therefore conflict.
array[0].forEach { array[1].append($0) }
```

It's always been somewhat fraught to do simultaneous accesses to
an array because of copy-on-write.  The fact that you should not
create an array and then fork off a bunch of threads that assign
into different elements concurrently has been independently
rediscovered by a number of different programmers.  (Under this
proposal, we will still not be reliably detecting this problem
by default, because it is a race condition; see the section on
concurrency.)  The main new limitation here is that some idioms
which did work on a single thread are going to be forbidden.
This may just be a cost of progress, but there are things we
can do to mitigate the problem.

In the long term, the API of ``Array`` and other collections
should be extended to ensure that there are good ways of achieving
the tasks that exclusivity enforcement has made difficult.
It will take experience living with exclusivity in order to
understand the problems and propose the right API additions.
In the short term, these problems can be worked around with
``withUnsafeMutableBufferPointer``.

We do know that swapping two array elements will be problematic,
and accordingly we are (separately proposing)[https://github.com/swiftlang/swift-evolution/blob/master/proposals/0173-swap-indices.md] to add a
``swapAt`` method to ``MutableCollection`` that takes two indices
rather than two ``inout`` arguments.  The Swift 3 compatibility
mode should recognize the swap-of-elements pattern and automatically
translate it to use ``swapAt``, and the 3-to-4 migrator should
perform this rewrite automatically.

### Class properties

Unlike value types, calling a method on a class doesn't formally access
the entire class instance.  In fact, we never try to enforce exclusivity
of access on the whole object at all; we only enforce it for individual
stored properties.  Among other things, this means that an access to a
class property never conflicts with an access to a different property.

There are two major reasons for this difference between value
and reference types.

The first reason is that it's important to allow overlapping method
calls to a single class instance.  It's quite common for an object
to have methods called on it concurrently from different threads.
These methods may access different properties, or they may synchronize
their accesses to the same properties using locks, dispatch queues,
or some other thread-safe technique.  Regardless, it's a widespread
pattern.

The second reason is that there's no benefit to trying to enforce
exclusivity of access to the entire class instance.  For a value
type to be mutated, it has to be held in a variable, and it's
often possible to reason quite strongly about how that variable
is used.  That means that the exclusivity rule that we're proposing
here allows us to make some very strong guarantees for value types,
generally making them an even tighter, lower-cost abstraction.
In contrast, it's inherent to the nature of reference types that
references can be copied pretty arbitrarily throughout a program.
The assumptions we want to make about value types depend on having
unique access to the variable holding the value; there's no way
to make a similar assumption about reference types without knowing
that we have a unique reference to the object, which would
radically change the programming model of classes and make them
unacceptable for the concurrent patterns described above.

### Disabling dynamic enforcement.

We could add an attribute which allows dynamic enforcement to
be downgraded to an unsafe-pointer-style undefined-behavior rule
on a variable-by-variable basis.  This would allow programmers to
opt out of the expense of dynamic enforcement when it is known
to be unnecessary (e.g. because exclusivity is checked at some
higher level) or when the performance burden is simply too great.

There is some concern that adding this attribute might lead to
over-use and that we should only support it if we are certain
that the overheads cannot be reduced in some better way.

Since the rule still applies, and it's merely no longer being checked,
it makes sense to borrow the "checked" and "unchecked" terminology
from the optimizer settings.

```swift
class TreeNode {
  @exclusivity(unchecked) var left: TreeNode?
  @exclusivity(unchecked) var right: TreeNode?
}
```

### Closures

A closure (including both local function declarations and closure
expressions, whether explicit or autoclosure) is either "escaping" or
"non-escaping".  Currently, a closure is considered non-escaping
only if it is:

- a closure expression which is immediately called,

- a closure expression which is passed as a non-escaping function
  argument, or

- a local function which captures something that is not allowed
  to escape, like an ``inout`` parameter.

It is likely that this definition will be broadened over time.

A variable is said to be escaping if it is captured in an escaping
closure; otherwise, it is non-escaping.

Escaping variables generally require dynamic enforcement instead of
static enforcement.  This is because Swift cannot reason about when
an escaping closure will be called and thus when the variable will
be accessed.  There are some circumstances where static enforcement
may still be allowed, for example when Swift can reason about how
the variable will be used after it is escaped, but this is only
possible as a best-effort improvement for special cases, not as a
general rule.

In contrast, non-escaping variables can always use static enforcement.
(In order to achieve this, we must impose a new restriction on
recursive uses of non-escaping closures; see below.)  This guarantee
aligns a number of related semantic and performance goals.  For
example, a non-escaping variable does not need to be allocated
on the heap; by also promising to only use static enforcement for
the variable, we are essentially able to guarantee that the variable
will have C-like performance, which can be important for some kinds
of program.  This guarantee also ensures that only static enforcement
is needed for ``inout`` parameters, which cannot be captured in
escaping closures; this substantially simplifies the implementation
model for capturing ``inout`` parameters.

### Diagnosing dynamic enforcement violations statically

In general, Swift is permitted to upgrade dynamic enforcement to
static enforcement when it can prove that two accesses either
always or never conflict.  This is analogous to Swift's rules
about integer overflow.

For example, if Swift can prove that two accesses to a global
variable will always conflict, then it can report that error
statically, even though global variables use dynamic enforcement:

```swift
var global: Int
swap(&global, &global) // Two overlapping modifications to 'global'
```

Swift is not required to prove that both accesses will actually
be executed dynamically in order to report a violation statically.
It is sufficient to prove that one of the accesses cannot ever
be executed without causing a conflict.  For example, in the
following example, Swift does not need to prove that ``mutate``
actually calls its argument function:

```swift
// The inout access lasts for the duration of the call.
global.mutate { return global + 1 }
```

When a closure is passed as a non-escaping function argument
or captured in a closure that is passed as a non-escaping function
argument, Swift may assume that any accesses made by the closure
will be executed during the call, potentially conflicting with
accesses that overlap the call.

### Restrictions on recursive uses of non-escaping closures

In order to achieve the goal of guaranteeing the use of static
enforcement for variables that are captured only by non-escaping
closures, we do need to impose an additional restriction on
the use of such closures.  This rule is as follows:

> A non-escaping closure ``A`` may not be recursively invoked
> during the execution of a non-escaping closure ``B`` which
> captures the same local variable or ``inout`` parameter unless:
>
> - ``A`` is defined within ``B`` or
>
> - ``A`` is a local function declaration which is referenced
>   directly by ``B``.

For clarity, we will call this rule the Non-Escaping Recursion
Restriction, or NRR.  The NRR is sufficient to prove that
non-escaping variables captured by ``B`` will not be interfered
with unless ``B`` delegates to something which is locally known by
``B`` to have access to those variables.  This, together with the
fact that the uses of ``B`` itself can be statically analyzed by
its defining function, is sufficient to allow static enforcement
for the non-escaping variables it captures.  (It also enables some
powerful analyses of captured variables within non-escaping
closures; we do not need to get into that here.)

Because of the tight restrictions on how non-escaping closures
can be used in Swift today, it's already quite difficult to
violate the NRR.  The following user-level restrictions are
sufficient to ensure that the NRR is obeyed:

- A function may not call a non-escaping function parameter
  passing a non-escaping function parameter as an argument.

  For the purposes of this rule, a closure which captures
  a non-escaping function parameter is treated the same as
  the parameter.

  We will call this rule the Non-Escaping Parameter Call
  Restriction, or NPCR.

- Programmers using ``withoutActuallyEscaping`` should take
  care not to allow the result to be recursively invoked.

The NPCR is a conservative over-approximation: that is, there
is code which does not violate the NRR which will be considered
ill-formed under the NPCR.  This is unfortunate but inevitable.

Here is an example of the sort of code that will be disallowed
under the NPCR:

```swift
func recurse(fn: (() -> ()) -> ()) {
  // Invoke the closure, passing a closure which, if invoked,
  // will invoke the closure again.
  fn { fn { } }
}

func apply<T>(argProvider: () -> T, fn: (() -> T) -> T) {
  // Pass the first argument function to the second.
  fn(argProvider)
}
```

Note that it's quite easy to come up with ways to use these
functions that wouldn't violate the NRR.  For example, if
either argument to ``apply`` is not a closure, the call
cannot possibly violate the NRR.  Nonetheless, we feel that
the NPCR is a reasonable restriction:

- Functions like ``recurse`` that apply a function to itself are
  pretty much just of theoretical interest.  Recursion is an
  important programming tool, but nobody writes it like this
  because it's just unnecessarily more difficult to reason about.

- Functions like ``apply`` that take two closures are not uncommon,
  but they're likely to either invoke the closures sequentially,
  which would not violate the NPCR, or else be some sort of
  higher-order combinator, which would require the closures to be
  ``@escaping`` and thus also not violate the NPCR.

Note that passing two non-escaping functions as arguments to the
same call does not violate the NPCR.  This is because the NPCR
will be enforced, recursively, in the callee.  (Imported C
functions which take non-escaping block parameters can, of
course, easily violate the NPCR.  They can also easily allow
the block to escape.  We do not believe there are any existing
functions or methods on our target platforms that directly
violate the NPCR.)

In general, programmers who find the NPCR an unnecessarily
overbearing restriction can simply declare their function parameter
to be ``@escaping`` or, if they are certain that their code will
not violate the NRR, use ``withoutActuallyEscaping`` to disable
the NPCR check.

## Source compatibility

In order to gain the performance and language-design benefits of
exclusivity, we will have to enforce it in all language modes.
Therefore, exclusivity will eventually demand a source break.

We can mitigate some of the impact of this break by implicitly migrating
code matching certain patterns to use different patterns that are known
to satisfy the exclusivity rule.  For example, it would be straightforward
to automatically translate calls like ``swap(&array[i], &array[j])`` to
``array.swapAt(i, j)``.  Whether this makes sense for any particular
migration remains to be seen; for example, ``swap`` does not appear to be
used very often in practice outside of specific collection algorithms.

Overall, we do not expect that a significant amount of code will violate
exclusivity.  This has been borne out so far by our testing.  Often the
examples that do violate exclusivity can easily be rewritten to avoid
conflicts.  In some of these cases, it may make sense to do the rewrite
automatically to avoid source-compatibility problems.

## Effect on ABI stability and resilience

In order to gain the performance and language-desing benefits of
exclusivity, we must be able to assume that it is followed faithfully
in various places throughout the ABI.  Therefore, exclusivity must be
enforced before we commit to a stable ABI, or else we'll be stuck with
the current conservatism around ``inout`` and ``mutating`` methods
forever.
