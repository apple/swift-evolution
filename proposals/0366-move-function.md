# `consume` operator to end the lifetime of a variable binding

* Proposal: [SE-0366](0366-move-function.md)
* Authors: [Michael Gottesman](https://github.com/gottesmm), [Andrew Trick](https://github.com/atrick), [Joe Groff](https://github.com/jckarter)
* Review Manager: [Holly Borla](https://github.com/hborla)
* Status: **Implemented (Swift 5.9)**
* Implementation: in main branch of compiler
* Review: ([pitch](https://forums.swift.org/t/pitch-move-function-use-after-move-diagnostic/53983)) ([first review](https://forums.swift.org/t/se-0366-move-function-use-after-move-diagnostic/59202)) ([returned for revision](https://forums.swift.org/t/returned-for-revision-se-0366-move-operation-use-after-move-diagnostic/59687)) ([second review](https://forums.swift.org/t/se-0366-second-review-take-operator-to-end-the-lifetime-of-a-variable-binding/61021)) ([third review](https://forums.swift.org/t/combined-se-0366-third-review-and-se-0377-second-review-rename-take-taking-to-consume-consuming/61904)), ([acceptance](https://forums.swift.org/t/accepted-se-0366-consume-operator-to-end-the-lifetime-of-a-variable-binding/62758))
* Previous Revisions:
  * [1](https://github.com/swiftlang/swift-evolution/blob/567fb1a66c784bcc5394491d24f72a3cb393674f/proposals/0366-move-function.md)
  * [2](https://github.com/swiftlang/swift-evolution/blob/43849aa9ae3e87c434866c5a5e389af67537ca26/proposals/0366-move-function.md)
  * [3](https://github.com/swiftlang/swift-evolution/blob/7af91127d693bffcb01aa87978d75d5a3170c4d1/proposals/0366-move-function.md)

## Introduction

In this document, we propose adding a new operator, marked by the
context-sensitive keyword `consume`, to the
language. `consume` ends the lifetime of a specific local `let`,
local `var`, or function parameter, and enforces this
by causing the compiler to emit a diagnostic upon any use after the
consume. This allows for code that relies on **forwarding ownership**
of values for performance or correctness to communicate that requirement to
the compiler and to human readers. As an example:

```swift
useX(x) // do some stuff with local variable x

// Ends lifetime of x, y's lifetime begins.
let y = consume x // [1]

useY(y) // do some stuff with local variable y
useX(x) // error, x's lifetime was ended at [1]

// Ends lifetime of y, destroying the current value.
_ = consume y // [2]
useX(x) // error, x's lifetime was ended at [1]
useY(y) // error, y's lifetime was ended at [2]
```

## Motivation

Swift uses reference counting and copy-on-write to allow developers to
write code with value semantics and not normally worry too much
about performance or memory management. However, in performance sensitive code,
developers want to be able to control the uniqueness of COW data structures and
reduce retain/release calls in a way that is future-proof against changes to
the language implementation or source code. Consider the following
example:

```swift
func test() {
  var x: [Int] = getArray()
  
  // x is appended to. After this point, we know that x is unique. We want to
  // preserve that property.
  x.append(5)
  
  // Pass the current value of x off to another function, that could take
  // advantage of its uniqueness to efficiently mutate it further.
  doStuffUniquely(with: x)

  // Reset x to a new value. Since we don't use the old value anymore,
  // it could've been uniquely referenced by the callee.
  x = []
  doMoreStuff(with: &x)
}

func doStuffUniquely(with value: [Int]) {
  // If we received the last remaining reference to `value`, we'd like
  // to be able to efficiently update it without incurring more copies.
  var newValue = value
  newValue.append(42)

  process(newValue)
}
```

In the example above, a value is built up in the variable `x` and then
handed off to `doStuffUniquely(with:)`, which makes further modifications to
the value it receives. `x` is then set to a new value. It should be possible for
the caller to **forward ownership** of the value of `x` to `doStuffUniquely`,
since it no longer uses the value as is, to avoid unnecessary retains or
releases of the array buffer and unnecessary copy-on-writes of the array
contents. `doStuffUniquely` should in turn be able to move its parameter into
a local mutable variable and modify the unique buffer in place. The compiler
could make these optimizations automatically, but a number of analyses have to
align to get the optimal result. The programmer may want to guarantee that this
series of optimizations occurs, and receive diagnostics if they wrote the code
in a way that would interfere with these optimizations being possible.

Swift-evolution pitch threads:

- [https://forums.swift.org/t/pitch-move-function-use-after-move-diagnostic](https://forums.swift.org/t/pitch-move-function-use-after-move-diagnostic)
- [https://forums.swift.org/t/selective-control-of-implicit-copying-behavior-take-borrow-and-copy-operators-noimplicitcopy/60168](https://forums.swift.org/t/selective-control-of-implicit-copying-behavior-take-borrow-and-copy-operators-noimplicitcopy/60168)

## Proposed solution: `consume` operator

That is where the `consume` operator comes into play. `consume` consumes
the current value of a **binding with static lifetime**, which is either
an unescaped local `let`, unescaped local `var`, or function parameter, with
no property wrappers or get/set/read/modify/etc. accessors applied. It then
 provides a compiler guarantee that the current value will
be unable to be used again locally. If such a use occurs, the compiler will
emit an error diagnostic. We can modify the previous example to use `consume` to
explicitly end the lifetime of `x`'s current value when we pass it off to
`doStuffUniquely(with:)`:

```swift
func test() {
  var x: [Int] = getArray()
  
  // x is appended to. After this point, we know that x is unique. We want to
  // preserve that property.
  x.append(5)
  
  // Pass the current value of x off to another function, that
  doStuffUniquely(with: consume x)

  // Reset x to a new value. Since we don't use the old value anymore,
  x = []
  doMoreStuff(with: &x)
}
```

The `consume x` operator syntax deliberately mirrors the
proposed [ownership modifier](https://forums.swift.org/t/borrow-and-take-parameter-ownership-modifiers/59581)
parameter syntax, `(x: consuming T)`, because the caller-side behavior of `consume`
operator is analogous to a callee’s behavior receiving a `consuming` parameter.
`doStuffUniquely(with:)` can use the `consume` operator, combined with
the `consuming` parameter modifier, to preserve the uniqueness of the parameter
as it moves it into its own local variable for mutation:

```swift
func doStuffUniquely(with value: consuming [Int]) {
  // If we received the last remaining reference to `value`, we'd like
  // to be able to efficiently update it without incurring more copies.
  var newValue = consume value
  newValue.append(42)

  process(newValue)
}
```

This takes the guesswork out of the optimizations discussed above: in the
`test` function, the final value of `x` before reassignment is explicitly
handed off to `doStuffUniquely(with:)`, ensuring that the callee receives
unique ownership of the value at that time, and that the caller can't
use the old value again. Inside `doStuffUniquely(with:)`, the lifetime of the
immutable `value` parameter is ended to initialize the local variable `newValue`,
ensuring that the assignment doesn't cause a copy.
Furthermore, if a future maintainer modifies the code in a way that breaks
this transfer of ownership chain, then the compiler will raise an
error. For instance, if a maintainer later introduces an additional use of
`x` after it's consumed, but before it's reassigned, they will see an error:

```swift
func test() {
  var x: [Int] = getArray()
  x.append(5)
  
  doStuffUniquely(with: consume x)

  // ERROR: x used after being consumed
  doStuffInvalidly(with: x)

  x = []
  doMoreStuff(with: &x)
}
```

Likewise, if the maintainer tries to access the original `value` parameter inside
of `doStuffUniquely` after being consumed to initialize `newValue`, they will
get an error:

```swift
func doStuffUniquely(with value: consuming [Int]) {
  // If we received the last remaining reference to `value`, we'd like
  // to be able to efficiently update it without incurring more copies.
  var newValue = consume value
  newValue.append(42)

  process(newValue)
}
```

`consume` can also end the lifetime of local immutable `let` bindings, which become
unavailable after their value is consumed since they cannot be reassigned.
Also note that `consume` operates on bindings, not values. If we declare a
constant `x` and another local constant `other` with the same value,
we can still use `other` after we consume the value from `x`, as in:

```swift
func useX(_ x: SomeClassType) -> () {}

func f() {
  let x = ...
  useX(x)
  let other = x   // other is a new binding used to extend the lifetime of x
  _ = consume x // x's lifetime ends
  useX(other)     // other is used here... no problem.
  useX(other) // other is used here... no problem.
}
```

We can also consume `other` independently of `x`, and get separate diagnostics for both
variables:

```swift
func useX(_ x: SomeClassType) -> () {}

func f() {
  let x = ...
  useX(x)
  let other = x
  _ = consume x
  useX(consume other)
  useX(other) // error: 'other' used after being consumed
  useX(x) // error: 'x' used after being consumed
}
```

`inout` function parameters can also be used with `consume`. Like a `var`, an
`inout` parameter can be reassigned after being consumed from and used again;
however, since the final value of an `inout` parameter is passed back to the
caller, it *must* be reassigned by the callee before it
returns. So this will raise an error because `buffer` doesn't have a value at
the point of return:

```swift
func f(_ buffer: inout Buffer) { // error: 'buffer' not reinitialized after consume!
  let b = consume buffer         // note: consume was here
  b.deinitialize()
  ... write code ...
}                                // note: return without reassigning inout argument `buffer`
```

But we can reinitialize `buffer` by writing the following code:

```swift
func f(_ buffer: inout Buffer) {
  let b = consume buffer
  b.deinitialize()
  // ... write code ...
  // We re-initialized buffer before end of function so the checker is satisfied
  buffer = getNewInstance()
}
```

`defer` can also be used to reinitialize an `inout` or `var` after a consume,
in order to ensure that reassignment happens on any exit from scope, including
thrown errors or breaks out of loops. So we can also write:

```swift
func f(_ buffer: inout Buffer) {
  let b = consume buffer
  // Ensure the buffer is reinitialized before we exit.
  defer { buffer = getNewInstance() }
  try b.deinitializeOrError()
  // ... write code ...
}
```

## Detailed design

At runtime, `consume x` evaluates to the current value bound to `x`, just like the
expression `x` does.  However, at compile time, the presence of a `consume` forces
ownership of the argument to be transferred out of the binding at the given
point so.  Any ensuing use of the binding that's reachable from the `consume`
is an error.  The operand to `consume` is required to be a reference
to a *binding with static lifetime*.  The following kinds of declarations can
currently be referenced as bindings with static lifetime:

- a local `let` constant in the immediately-enclosing function,
- a local `var` variable in the immediately-enclosing function,
- one of the immediately-enclosing function's parameters, or
- the `self` parameter in a `mutating` or `__consuming` method.

A binding with static lifetime also must satisfy the following requirements:

- it cannot be captured by an `@escaping` closure or nested function,
- it cannot have any property wrappers applied,
- it cannot have any accessors attached, such as `get`, `set`,
  `didSet`, `willSet`, `_read`, or `_modify`,
- it cannot be an `async let`.

Possible extensions to the set of operands that can be used with `consume` are
discussed under Future Directions. It is an error to use `consume` with an operand
that doesn't reference a binding with static lifetime.

Given a valid operand, `consume` enforces that there are no other
references to the binding after it is consumed. The analysis is
flow sensitive, so one is able to end the lifetime of a value conditionally:

```swift
if condition {
  let y = consume x
  // I can't use x anymore here!
  useX(x) // !! ERROR! Use after consume.
} else {
  // I can still use x here!
  useX(x) // OK
}
// But I can't use x here.
useX(x) // !! ERROR! Use after consume.
```

If the binding is a `var`, the analysis additionally allows for code to
conditionally reinitialize the var and thus use it in positions
that are dominated by the reinitialization. Consider the
following example:

```swift
if condition {
  _ = consume x
  // I can't use x anymore here!
  useX(x) // !! ERROR! Use after consume.
  x = newValue
  // But now that I have re-assigned into x a new value, I can use the var
  // again.
  useX(x) // OK
} else {
  // I can still use x here, since it wasn't consumed on this path!
  useX(x) // OK
}
// Since I reinitialized x along the `if` branch, and it was never consumed
// from on the `else` branch, I can use it here too.
useX(x) // OK
```

Notice how in the above, we are able to use `x` both in the true block AND the
code after the `if` block, since over both paths through the `if`, `x` ends up
with a valid value before proceeding.

For an `inout` parameter, the analysis behaves the same as for a `var`, except
that all exits from the function (whether by `return` or by `throw`) are
considered to be uses of the parameter. Correct code therefore *must* reassign
inout parameters after they are consumed from.

Using `consume` on a binding without using the result raises a warning,
just like a function call that returns an unused non-`Void` result.
To "drop" a value without using it, the `consume` can be assigned to
`_` explicitly.

## Source compatibility

`consume` behaves as a contextual keyword. In order to avoid interfering
with existing code that calls functions named `consume`, the operand to
`consume` must begin with another identifier, and must consist of an
identifier or postfix expression:

```swift
consume x // OK
consume [1, 2, 3] // Subscript access into property named `consume`, not a consume operation
consume (x) // Call to function `consume`, not a consume operation
consume x.y.z // Syntactically OK (although x.y.z is not currently semantically valid)
consume x[0] // Syntactically OK (although x[0] is not currently semantically valid
consume x + y // Parses as (consume x) + y
```

## Effect on ABI stability

`consume` requires no ABI additions.

## Effect on API resilience

None, this is additive.

## Alternatives considered

### Alternative spellings

The [first reviewed revision](https://github.com/swiftlang/swift-evolution/blob/567fb1a66c784bcc5394491d24f72a3cb393674f/proposals/0366-move-function.md)
of this proposal offered `move(x)` as a special
function with semantics recognized by the compiler. Based on initial feedback,
we pivoted to the contextual keyword spelling.  As a function, this operation
would be rather unusual, since it only accepts certain forms of expression as
its argument, and it doesn't really have any runtime behavior of its own,
acting more as a marker for the compiler to perform additional analysis.

The community reviewed the contextual keyword syntax, using the name `move x`,
and through further discussion the alternative name `take` arose. This name
aligns with the term used in the Swift compiler internals, and also reads well
as the analogous parameter ownership modifier, `(x: take T)`, so the authors
now favor this name. However, during review of SE-0377, reviewers found that
`take` was too generic, and could be confused with the common colloquial language
of talking about function calls as "taking their arguments". Of alternative
names reviewers offered, the language workgroup favors `consume`.

Many have suggested alternative spellings that also make `consume`'s special
nature more syntactically distinct, including:

- an expression attribute, like `useX(@consume x)`
- a compiler directive, like `useX(#consume x)`
- an operator, like `useX(<-x)`

### Use of scoping to end lifetimes

It is possible in the language today to foreshorten the lifetime of local
variables using lexical scoping:

```swift
func test() {
  var x: [Int] = getArray()
  
  // x is appended to. After this point, we know that x is unique. We want to
  // preserve that property.
  x.append(5)
  
  // We create a new variable y so we can write an algorithm where we may
  // change the value of y (causing a COW copy of the buffer shared with x).
  do {
    var y = x
    longAlgorithmUsing(&y)
    consumeFinalY(y)
  }

  // We no longer use y after this point. Ideally, x would be guaranteed
  // unique so we know we can append again without copying.
  x.append(7)
}
```

However, there are a number of reasons not to rely solely on lexical scoping
to end value lifetimes:

- Adding lexical scopes requires nesting, which can lead to "pyramid of doom"
  situations when managing the lifetimes of multiple variables.
- Value lifetimes don't necessarily need to nest, and may overlap or interleave
  with control flow. This should be valid:

    ```swift
    let x = foo()
    let y = bar()
    // end x's lifetime before y's
    consume(consume x)
    consume(consume y)
    ```

- Lexical scoping cannot be used by itself to shorten the lifetime of function
  parameters, which are in scope for the duration of the function body.
- Lexical scoping cannot be used to allow for taking from and reinitializing
  mutable variables or `inout` parameters.

Looking outside of Swift, the Rust programming language originally only had
strictly scoped value lifetimes, and this was a significant ergonomic
problem until "non-lexical lifetimes" were added later, which allowed for
value lifetimes to shrinkwrap to their actual duration of use.

## Future directions

### Dynamic enforcement of `consume` for other kinds of bindings

In the future, we may want to accommodate the ability to dynamically
consume bindings with dynamic lifetime, such as escaped local
variables, and class stored properties, although doing so in full
generality would require dynamic enforcement in addition to static
checking, similar to how we need to dynamically enforce exclusivity
when accessing globals and class stored properties.  Since this
dynamic enforcement turns misuse of `consume`s into runtime errors
rather than compile-time guarantees, we might want to make those
dynamic cases syntactically distinct, to make the possibility of
runtime errors clear.

`Optional` and other types with a canonical "no value" or "empty"
state can use the static `consume` operator to provide API that
dynamically takes ownership of the current value inside of them
while leaving them in their empty state:

```swift
extension Optional {
  mutating func take() -> Wrapped {
    switch consume self {
    case .some(let x):
      self = .none
      return x
    case .none:
      fatalError("trying to consume an empty Optional")
    }
  }
}
```

### Piecewise `consume` of frozen structs and tuples

For frozen structs and tuples, both aggregates that the compiler can statically
know the layout of, we could do finer-grained analysis and allow their
individual fields to be consumed independently:

```swift
struct TwoStrings {
  var first: String
  var second: String
}

func foo(x: TwoStrings) {
  use(consume x.first)
  // ERROR! part of x was consumed
  use(x)
  // OK, this part wasn't
  use(x.second)
}
```

### Destructuring methods for move-only types with `deinit`

Move-only types would allow for the possibility of value types with custom
`deinit` logic that runs at the end of a value of the type's lifetime.
Typically, this logic would run when the final owner of the value is finished
with it, which means that a function which `consume`s an instance, or a
`taking func` method on the type itself, would run the deinit if it does not
forward ownership anywhere else:

```swift
moveonly struct FileHandle {
  var fd: Int32

  // close the fd on deinit
  deinit { close(fd) }
}

func dumpAndClose(to fh: consume FileHandle, contents: Data) {
  write(fh.fd, contents)
  // fh implicitly deinit-ed here, closing it
}
```

However, this may not always be desirable, either because the function performs
an operation that invalidates the value some other way, making it unnecessary
or incorrect for `deinit` to run on it, or because the function wants to be able
to take ownership of parts away from the value:

```swift
extension FileHandle {
  // Return the file descriptor back to the user for manual management
  // and disable automatic management with the FileHandle value.

  taking func giveUp() -> Int32 {
    return fd
    // How do we stop the deinit from running here?
  }
}
```

Rust has the magic function `mem::forget` to suppress destruction of a value,
though `forget` in Rust still does not allow for the value to be destructured
into parts. We could come up with a mechanism in Swift that both suppresses
implicit deinitialization, and allows for piecewise taking of its components.
This doesn't require a new parameter convention (since it fits within the
ABI of a `consume T` parameter), but could be spelled as a new operator
inside of a `taking func`:

```swift
extension FileHandle {
  // Return the file descriptor back to the user for manual management
  // and disable automatic management with the FileHandle value.

  taking func giveUp() -> Int32 {
    // `deinit fd` is strawman syntax for consuming a value without running
    // its deinitializer. it is only allowed inside of `taking func` methods
    // that have visibility into the type layout.
    return (deinit self).fd
  }
}

moveonly struct SocketPair {
  var input, output: FileHandle

  deinit { /* ... */ }

  // Break the pair up into separately-managed FileHandles
  taking func split() -> (input: FileHandle, output: FileHandle) {
    // Break apart the value without running the standard deinit
    return ((deinit self).input, (deinit self).output)
  }
}
```

Suppressing the normal deinit logic for a type should be done carefully.
Although Rust allows `mem::forget` to be used anywhere, and considers it "safe"
based on the observation that destructors may not always run if a program
crashes or leaks a value anyway, that observation is less safe in the case
of values whose lifetime depends on a parent value, and where the child value's
deinit must be sequenced with operations on the parent. As such, I think
the ability should be limited to methods defined on the type itself. Also,
within the constraints of Swift's stable ABI and library evolution model,
code that imports a module may not always have access to the concrete layout
of a type, which would make partial destructuring impossible, which further
limits the contexts in which such an operator can be used.

### `consume` from computed properties, property wrappers, properties with accessors, etc.

It would potentially be useful to be able to `consume` variables
and properties with modified access behavior, such as computed
properties, properties with didSet/willSet observers, property
wrappers, and so on. Although we could do lifetime analysis on these
properties, we wouldn't be able to get the full performance benefits
from consuming a computed variable without allowing for some
additional accessors to be defined, such as a "consuming getter"
that can consume its `self` in order to produce the property value,
and an initializer to reinitialize `self` on reassignment after a
`move`.

### Additional selective controls for implicit copying behavior

Pitch: [Selective control of implicit copying behavior](https://forums.swift.org/t/selective-control-of-implicit-copying-behavior-take-borrow-and-copy-operators-noimplicitcopy/60168)

The `consume` operator is one of a number of implicit copy controls
we're considering:

- A value that isn't modified can generally be "borrowed" and
  shared in-place by multiple bindings, or between a caller and
  callee, without copying. However, the compiler will
  pass shared mutable state by copying the current value, and
  passing that copy to a callee. We do this to avoid
  potential rule-of-exclusivity violations, since it is difficult to
  know for sure whether a callee will go back and try to mutate the
  same global variable, object, or other bit of shared mutable
  state:

    ```swift
    var global = Foo()

    func useFoo(x: Foo) {
      // We would need exclusive access to `global` to do this:
      
      /*
      global = Foo()
       */
    }

    func callUseFoo() {
      // callUseFoo doesn't know whether `useFoo` accesses global,
      // so we want to avoid imposing shared access to it for longer
      // than necessary. So by default the compiler will
      // pass a copy instead, and this:
      useFoo(x: global)

      // will compile more like:

      /*
      let copyOfGlobal = copy(global)
      useFoo(x: copyOfGlobal)
      destroy(copyOfGlobal)
       */
    }
    ```

    Although the compiler is allowed to eliminate the defensive copy
    inside callUseFoo if it proves that useFoo doesn't try to write
    to the global variable, it is unlikely to do so in practice. The
    developer however knows that useFoo doesn't modify global, and
    may want to suppress this copy in the call site. An explicit
    `borrow` operator would let the developer communicate this to the
    compiler:

    ```swift
    var global = Foo()

    func useFoo(x: Foo) {
      /* global not used here */
    }

    func callUseFoo() {
      // The programmer knows that `useFoo` won't
      // touch `global`, so we'd like to pass it without copying
      useFoo(x: borrow global)
    }
    ```

- `consume` and `borrow` operators can eliminate copying in
  common localized situations, but it is also useful to be able to
  suppress implicit copying altogether for certain variables, types,
  and scopes. We could define an attribute to specify that bindings
  with static lifetime, types, or scopes should not admit implicit
  copies:

    ```swift
    // we're not allowed to implicitly copy `x`
    func foo(@noImplicitCopy x: String) {
    }

    // we're not allowed to implicitly copy values (statically) of
    // type Gigantic
    @noImplicitCopy struct Gigantic {
      var fee, fie, fo, fum: String
    }

    // we're not allowed to implicitly copy inside this hot loop
    for item in items {
      @noImplicitCopy do {
      }
    }
    ```

### `borrow` and `consume` argument modifiers

Pitch: [`borrow` and `consume` parameter ownership modifiers](https://forums.swift.org/t/borrow-and-take-parameter-ownership-modifiers/59581)

Swift currently only makes an explicit distinction between
pass-by-value and pass-by-`inout` parameters, leaving the mechanism
for pass-by-value up to the implementation. But there are two broad
conventions that the compiler uses to pass by value:

- The callee can **borrow** the parameter. The caller guarantees that
  its argument object will stay alive for the duration of the call,
  and the callee does not need to release it (except to balance any
  additional retains it performs itself).
- The callee can **consume** the parameter. The callee becomes
  responsible for either releasing the parameter or passing
  ownership of it along somewhere else. If a caller doesn't want to
  give up its own ownership of its argument, it must retain the
  argument so that the callee can consume the extra reference count.

In order to allow for manual optimization of code, and to support
move-only types where this distinction becomes semantically
significant, we plan to introduce explicit parameter modifiers to
let developers specify explicitly which convention a parameter
should use.

## Acknowledgments

Thanks to Nate Chandler, Tim Kientzle, and Holly Borla for their help with this!

## Revision history

Changes from the [third revision](https://github.com/swiftlang/swift-evolution/blob/7af91127d693bffcb01aa87978d75d5a3170c4d1/proposals/0366-move-function.md):

- `take` is renamed again to `consume`.

Changes from the [second revision](https://github.com/swiftlang/swift-evolution/blob/43849aa9ae3e87c434866c5a5e389af67537ca26/proposals/0366-move-function.md):

- `move` is renamed to `take`.
- Dropping a value without using it now requires an explicit
  `_ = take x` assignment again.
- "Movable bindings" are referred to as "bindings with static lifetime",
  since this term is useful and relevant to other language features.
- Additional "alternatives considered" raised during review
  and pitch discussions were added.
- Expansion of "related directions" section contextualizes the
  `take` operator among other planned features for selective copy
  control.
- Now that [ownership modifiers for parameters](https://forums.swift.org/t/borrow-and-take-parameter-ownership-modifiers/59581)
  are being pitched, this proposal ties into that one. Based on
  feedback during the first review, we have gone back to only allowing
  parameters to be used with the `take` operator if the parameter
  declaration is `take` or `inout`.

Changes from the [first revision](https://github.com/swiftlang/swift-evolution/blob/567fb1a66c784bcc5394491d24f72a3cb393674f/proposals/0366-move-function.md):

- `move x` is now proposed as a contextual keyword, instead of a magic function
  `move(x)`.
- The proposal no longer mentions `__owned` or `__shared` parameters, which
  are currently an experimental language feature, and leaves discussion of them
  as a future direction. `move x` is allowed to be used on all function
  parameters.
- `move x` is allowed as a statement on its own, ignoring the return value,
  to release the current value of `x` without forwarding ownership without
  explicitly assigning `_ = move x`.
