# Noncopyable structs and enums

* Proposal: [SE-0390](0390-noncopyable-structs-and-enums.md)
* Authors: [Joe Groff](https://github.com/jckarter), [Michael Gottesman](https://github.com/gottesmm), [Andrew Trick](https://github.com/atrick), [Kavon Farvardin](https://github.com/kavon)
* Review Manager: [Stephen Canon](https://github.com/stephentyrone)
* Status: **Implemented (Swift 5.9)**
* Implementation: in main branch of compiler
* Review: ([pitch](https://forums.swift.org/t/pitch-noncopyable-or-move-only-structs-and-enums/61903)) ([first review](https://forums.swift.org/t/se-0390-noncopyable-structs-and-enums/63258)) ([second review](https://forums.swift.org/t/second-review-se-0390-noncopyable-structs-and-enums/63866)) ([acceptance](https://forums.swift.org/t/accepted-se-0390-noncopyable-structs-and-enums/65157))
* Previous Revisions: [1](https://github.com/swiftlang/swift-evolution/blob/5d075b86d57e3436b223199bd314b2642e30045f/proposals/0390-noncopyable-structs-and-enums.md)

## Introduction

This proposal introduces the concept of **noncopyable** types (also known
as "move-only" types). An instance of a noncopyable type always has unique
ownership, unlike normal Swift types which can be freely copied.

## Motivation

All currently existing types in Swift are **copyable**, meaning it is possible
to create multiple identical, interchangeable representations of any value of
the type. However, copyable structs and enums are not a great model for
unique resources. Classes by contrast *can* represent a unique resource,
since an object has a unique identity once initialized, and only references to
that unique object get copied. However, because the references to the object are
still copyable, classes always demand *shared ownership* of the resource. This
imposes overhead in the form of heap allocation (since the overall lifetime of
the object is indefinite) and reference counting (to keep track of the number
of co-owners currently accessing the object), and shared access often
complicates or introduces unsafety or additional overhead into an object's
APIs. Swift does not yet have a mechanism for defining types that
represent unique resources with *unique ownership*.

## Proposed solution

We propose to allow for `struct` and `enum` types to declare themselves as
*noncopyable*, using a new syntax for suppressing implied generic constraints,
`~Copyable`. Values of noncopyable type always have unique ownership, and
can never be copied (at least, not using Swift's implicit copy mechanism).
Since values of noncopyable structs and enums have unique identities, they can
also have `deinit` declarations, like classes, which run automatically at the
end of the unique instance's lifetime.

For example, a basic file descriptor type could be defined as:

```swift
struct FileDescriptor: ~Copyable {
  private var fd: Int32

  init(fd: Int32) { self.fd = fd }

  func write(buffer: Data) {
    buffer.withUnsafeBytes { 
      write(fd, $0.baseAddress!, $0.count)
    }
  }

  deinit {
    close(fd)
  }
}
```

Like a class, instances of this type can provide managed access to a file
handle, automatically closing the handle once the value's lifetime ends. Unlike
a class, no object needs to be allocated; only a simple struct containing the
file descriptor ID needs to be stored in the stack frame or aggregate value
that uniquely owns the instance.

## Detailed design

### The `Copyable` generic constraint

Before this proposal, almost every type in Swift was automatically copyable.
The standard library provides a new generic constraint `Copyable` to make
this capability explicit. All existing first-class types (excluding nonescaping
closures) implicitly satisfy this constraint, and all generic type parameters,
existential types, protocols, and associated type requirements implicitly
require it. Types may explicitly declare that they are `Copyable`, and generic
types may explicitly require `Copyable`, but this currently has no effect.

```swift
struct Foo<T: Copyable>: Copyable {}
```

### Declaring noncopyable types

A `struct` or `enum` type can be declared as noncopyable by suppressing the
`Copyable` requirement on their declaration, by combining the new `Copyable`
constraint with the new requirement suppression syntax `~Copyable`:

```swift
struct FileDescriptor: ~Copyable {
  private var fd: Int32
}
```

If a `struct` has a stored property of noncopyable type, or an `enum` has
a case with an associated value of noncopyable type, then the containing type
must also suppress its `Copyable` capability:

```swift
struct SocketPair: ~Copyable {
  var in, out: FileDescriptor
}

enum FileOrMemory: ~Copyable {
  // write to an OS file
  case file(FileDescriptor)
  // write to an array in memory
  case memory([UInt8])
}

// ERROR: copyable value type cannot contain noncopyable members
struct FileWithPath {
  var file: FileDescriptor
  var path: String
}
```

Classes, on the other hand, may contain noncopyable stored properties without
themselves becoming noncopyable:

```swift
class SharedFile {
  var file: FileDescriptor
}
```

A class type declaration may not use `~Copyable`; all class types remain copyable
by retaining and releasing references to the object.

```swift
// ERROR: classes must be `Copyable`
class SharedFile: ~Copyable {
  var file: FileDescriptor
}
```

It is also not yet allowed to suppress the `Copyable` requirement on generic
parameters, associated type requirements in protocols, or the `Self` type
in a protocol declaration, or in extensions:

```swift
// ERROR: generic parameter types must be `Copyable`
func foo<T: ~Copyable>(x: T) {}

// ERROR: types that conform to protocols must be `Copyable`
protocol Foo where Self: ~Copyable {
  // ERROR: associated type requirements must be `Copyable`
  associatedtype Bar: ~Copyable
}

// ERROR: cannot suppress `Copyable` in extension of `FileWithPath`
extension FileWithPath: ~Copyable {}
```

`Copyable` also cannot be suppressed in existential type declarations:

```swift
// ERROR: `any` types must be `Copyable`
let foo: any ~Copyable = FileDescriptor()
```

### Restrictions on use in generics

Noncopyable types may have generic type parameters:

```swift
// A type that reads from a file descriptor consisting of binary values of type T
// in sequence.
struct TypedFile<T>: ~Copyable {
  var rawFile: FileDescriptor

  func read() -> T { ... }
}

let byteFile: TypedFile<UInt8> // OK
```

At this time, as noted above, generic types are still always required to be
`Copyable`, so noncopyable types themselves are not allowed to be used as a
generic type argument. This means a noncopyable type _cannot_:

- conform to any protocols, except for `Sendable`.
- serve as a type witness for an `associatedtype` requirement.
- be used as a type argument when instantiating generic types or calling generic functions.
- be cast to (or from) `Any` or any other existential.
- be accessed through reflection.
- appear in a tuple.

The reasons for these restrictions and ways of lifting them are discussed under
Future Directions. The key implication of these restrictions is that a
noncopyable struct or enum is only a subtype of itself, because all other types
it might be compatible with for conversion would also permit copying.

#### The `Sendable` exception

The need for preventing noncopyable types from conforming to
protocols is rooted in the fact that all existing constrained generic types 
(like `some P` types) and existentials (`any P` types) are assumed to be 
copyable. Recording any conformances to these protocols would be invalid for
noncopyable types.

But, an exception is made where noncopyable types can conform to `Sendable`.
Unlike other protocols, the `Sendable` marker protocol leaves no conformance
record in the output program. Thus, there will be no ABI impact if a future 
noncopyable version of the `Sendable` protocol is created.

The big benefit of allowing `Sendable` conformances is that noncopyable types 
are compatible with concurrency. Keep in mind that despite their ability to 
conform to `Sendable`, noncopyable structs and enums are still only a subtype 
of themselves. That means when the noncopyable type conforms to `Sendable`, you
still cannot convert it to `any Sendable`, because copying that existential 
would copy its underlying value:

```swift
extension FileDescriptor: Sendable {} // OK

struct RefHolder: ~Copyable, Sendable {
  var ref: Ref  // ERROR: stored property 'ref' of 'Sendable'-conforming struct 'RefHolder' has non-sendable type 'Ref'
}

func openAsync(_ path: String) async throws -> FileDescriptor {/* ... */}
func sendToSpace(_ s: some Sendable) {/* ... */}

@MainActor func example() async throws {
  // OK. FileDescriptor can cross actors because it is Sendable
  var fd: FileDescriptor = try await openAsync("/dev/null")

  // ERROR: noncopyable types cannot be conditionally cast
  // WARNING: cast from 'FileDescriptor' to unrelated type 'any Sendable' always fails
  if let sendy: Sendable = fd as? Sendable {

    // ERROR: noncopyable types cannot be conditionally cast
    // WARNING: cast from 'any Sendable' to unrelated type 'FileDescriptor' always fails
    fd = sendy as! FileDescriptor
  }

  // ERROR: noncopyable type 'FileDescriptor' cannot be used with generics
  sendToSpace(fd)
}
```

#### Working around the generics restrictions

Since a good portion of Swift's standard library rely on generics, there are a
a number of common types and functions that will not work with today's 
noncopyable types:

```swift
// ERROR: Cannot use noncopyable type FileDescriptor in generic type Optional
let x = Optional(FileDescriptor(open("/etc/passwd", O_RDONLY)))

// ERROR: Cannot use noncopyable type FileDescriptor in generic type Array
let fds: [FileDescriptor] = []

// ERROR: Cannot use noncopyable type FileDescriptor in generic type Any
print(FileDescriptor(-1))

// ERROR: Noncopyable struct SocketEvent cannot conform to Error
enum SocketEvent: ~Copyable, Error {
  case requestedDisconnect(SocketPair)
}
```

For example, the `print` function expects to be able to convert its argument to
`Any`, which is a copyable value. Internally, it also relies on either 
reflection or conformance to `CustomStringConvertible`. Since a noncopyable type
can't do any of those, a suggested workaround is to explicitly define a 
conversion to `String`: 

```swift
extension FileDescriptor /*: CustomStringConvertible */ {
  var description: String {
    return "file descriptor #\(fd)"
  }
}

let fd = FileDescriptor(-1)
print(fd.description)
```

A more general kind of workaround to mix generics and noncopyable types
is to wrap the value in an ordinary class instance, which itself can participate
in generics. To transfer the noncopyable value in or out of the wrapper class
instance, using `Optional<FileDescriptor>` for the class's field would be 
ideal. But until that is supported, a concrete noncopyable enum can represent
the case where the value of interest was taken out of the instance:

```swift
enum MaybeFileDescriptor: ~Copyable {
  case some(FileDescriptor)
  case none

  // Returns this MaybeFileDescriptor by consuming it
  // and leaving .none in its place.
  mutating func take() -> MaybeFileDescriptor {
    let old = self // consume self
    self = .none   // reinitialize self
    return old
  }
}

class WrappedFile {
  var file: MaybeFileDescriptor

  enum Err: Error { case noFile }

  init(_ fd: consuming FileDescriptor) {
    file = .some(fd)
  }

  func consume() throws -> FileDescriptor {
    if case let .some(fd) = file.take() {
      return fd
    }
    throw Err.noFile
  }
}

func example(_ fd1: consuming FileDescriptor, 
             _ fd2: consuming FileDescriptor) -> [WrappedFile] {
  // create an array of descriptors
  return [WrappedFile(fd1), WrappedFile(fd2)]
}
```

All of this boilerplate melts away once noncopyable types support generics.
Even before then, one major improvement would be to eliminate the need to define
types like `MaybeFileDescriptor` through a noncopyable `Optional` 
(see Future Directions).


### Using noncopyable values

As the name suggests, values of noncopyable type cannot be copied, a major break
from most other types in Swift. Many operations are currently defined as
working as pass-by-value, and use copying as an implementation technique
to give that semantics, but these operations now need to be defined more
precisely in terms of how they *borrow* or *consume* their operands in order to
define their effects on values that cannot be copied. 

We use the term **consume** to refer to an operation
that invalidates the value that it operates on. It may do this by directly
destroying the value, freeing its resources such as memory and file handles,
or forwarding ownership of the value to yet another owner who takes
responsibility for keeping it alive. Performing a consuming operation on
a noncopyable value generally requires having ownership of the value to begin
with, and invalidates the value the operation was performed on after it is
completed.

We use the term **borrow** to refer to
a shared borrow of a single instance of a value; the operation that borrows
the value allows other operations to borrow the same value simultaneously, and
it does not take ownership of the value away from its current owner. This
generally means that borrowers are not allowed to mutate the value, since doing
so would invalidate the value as seen by the owner or other simultaneous
borrowers. Borrowers also cannot *consume* the value. They can, however,
initiate arbitrarily many additional borrowing operations on all or part of
the value they borrow.

Both of these conventions stand in contrast to **mutating** (or **inout**)
operations, which take an *exclusive* borrow of their operands. The behavior
of mutating operations on noncopyable values is much the same as `inout`
parameters of copyable type today, which are already subject to the
"law of exclusivity". A mutating operation has exclusive access to its operand
for the duration of the operation, allowing it to freely mutate the value
without concern for aliasing or data races, since not even the owner may
access the value simultaneously. A mutating operation may pass its operand
to another mutating operation, but transfers exclusivity to that other operation
until it completes. A mutating operation may also pass its operand to
any number of borrowing operations, but cannot assume exclusivity while those
borrows are enacted; when the borrowing operations complete, the mutating
operation may assume exclusivity again. Unlike having true ownership of a
value, mutating operations give ownership back to the owner at the end of an
operation.  A mutating operation therefore may consume the current value of its
operand, but if it does, it must replace it with a new value before completing.

For copyable types, the distinction between borrowing and consuming operations
is largely hidden from the programmer, since Swift will implicitly insert
copies as needed to maintain the apparent value semantics of operations; a
consuming operation can be turned into a borrowing one by copying the value and
giving the operation the copy to consume, allowing the program to continue
using the original. This of course becomes impossible for values that cannot
be copied, forcing the distinction.

Many code patterns that are allowed for copyable types also become errors for
noncopyable values because they would lead to conflicting uses of the same
value, without the ability to insert copies to avoid the conflict. For example,
a copyable value can normally be passed as an argument to the same function
multiple times, even to a `borrowing` and `consuming` parameter of the same
call, and the compiler will copy as necessary to make all of the function's
parameters valid according to their ownership specifiers:

```swift
func borrow(_: borrowing Value, and _: borrowing Value) {}
func consume(_: consuming Value, butBorrow _: borrowing Value) {}
let x = Value()
borrow(x, and: x) // this is fine, multiple borrows can share
consume(x, butBorrow: x) // also fine, we'll copy x to let a copy be consumed
                         // while the other is borrowed
```

By contrast, a noncopyable value *must* be passed by borrow or consumed,
without copying. This makes the second call above impossible for a noncopyable
`x`, since attempting to consume `x` would end the binding's lifetime while
it also needs to be borrowed:

```swift
func borrow(_: borrowing FileDescriptor, and _: borrowing FileDescriptor) {}
func consume(_: consuming FileDescriptor, butBorrow _: borrowing FileDescriptor) {}
let x = FileDescriptor()
borrow(x, and: x) // still OK to borrow multiple times
consume(x, butBorrow: x) // ERROR: consuming use of `x` would end its lifetime
                         // while being borrowed
```

A similar effect happens when `inout` parameters take noncopyable arguments.
Swift will copy the value of a variable if it is passed both by value and
`inout`, so that the by-value parameter receives a copy of the current value
while leaving the original binding available for the `inout` parameter to
exclusively access:

```swift
func update(_: inout Value, butBorrow _: borrow Value) {}
func update(_: inout Value, butConsume _: consume Value) {}
var x = Value()
update(&x, butBorrow: x) // this is fine, we'll copy `x` in the second parameter
update(&x, butConsume: x) // also fine, we'll also copy
```

But again, for a noncopyable value, this implicit copy is impossible, so
these sorts of calls become exclusivity errors:

```swift
func update(_: inout FileDescriptor, butBorrow _: borrow FileDescriptor) {}
func update(_: inout FileDescriptor, butConsume _: consume FileDescriptor) {}

var y = FileDescriptor()
update(&y, butBorrow: y) // ERROR: cannot borrow `y` while exclusively accessed
update(&y, butConsume: y) // ERROR: cannot consume `y` while exclusively accessed
```

The following sections attempt to classify existing language operations
according to what ownership semantics they have when performed on noncopyable
values.

### Consuming operations

The following operations are consuming:

- assigning a value to a new `let` or `var` binding, or setting an existing
  variable or property to the binding:

    ```swift
    let x = FileDescriptor()
    let y = x
    use(x) // ERROR: x consumed by assignment to `y`
    ```

    ```swift
    var y = FileDescriptor()
    let x = FileDescriptor()
    y = x
    use(x) // ERROR: x consumed by assignment to `y`
    ```

    ```swift
    class C {
      var property = FileDescriptor()
    }
    let c = C()
    let x = FileDescriptor()
    c.property = x
    use(x) // ERROR: x consumed by assignment to `c.property`
    ```

    The one exception is assigning to the "black hole" `_ = x`, which is
    a borrowing operation, as noted below. This allows for the
    `_ = x` idiom to still be used to prevent warnings about a borrowed
    binding that is otherwise unused.

- passing an argument to a `consuming` parameter of a function:

    ```swift
    func consume(_: consuming FileDescriptor) {}
    let x1 = FileDescriptor()
    consume(x1)
    use(x1) // ERROR: x1 consumed by call to `consume`
    ```

- passing an argument to an `init` parameter that is not explicitly
  `borrowing`:

    ```swift
    struct S: ~Copyable {
      var x: FileDescriptor, y: Int
    }
    let x = FileDescriptor()
    let s = S(x: x, y: 219)
    use(x) // ERROR: x consumed by `init` of struct `S`
    ```

- invoking a `consuming` method on a value, or accessing a property of the
  value through a `consuming get` or `consuming set` accessor:

    ```swift
    extension FileDescriptor {
      consuming func consume() {}
    }
    let x = FileDescriptor()
    x.consume()
    use(x) // ERROR: x consumed by method `consume`
    ```

- explicitly consuming a value with the `consume` operator:

    ```swift
    let x = FileDescriptor()
    _ = consume x
    use(x) // ERROR: x consumed by explicit `consume`
    ```

- `return`-ing a value;

- pattern-matching a value with `switch`, `if let`, or `if case`:

    ```swift
    let x: Optional = getValue()
    if let y = consume x { ... }
    use(x) // ERROR: x consumed by `if let`

    enum FileDescriptorOrBuffer: ~Copyable {
      case file(FileDescriptor)
      case buffer(String)
    }

    let x = FileDescriptorOrBuffer.file(FileDescriptor())

    switch consume x {
    case .file(let f):
      break
    case .buffer(let b):
      break
    }

    use(x) // ERROR: x consumed by `switch`
    ```

    In order to allow for borrowing pattern matching to potentially become
    the default later, when it's supported, the operand to `switch` or
    the right-hand side of a `case` condition in an `if` or `while` must
    use the `consume` operator in order to indicate that it is consumed.
    We may want `switch x` to borrow by default in the future.

- iterating a `Sequence` with a `for` loop:

    ```swift
    let xs = [1, 2, 3]
    for x in consume xs {}
    use(xs) // ERROR: xs consumed by `for` loop
    ```

(Although noncopyable types are not currently allowed to conform to
protocols, preventing them from implementing the `Sequence` protocol,
and cannot be used as generic parameters, preventing the formation of
`Optional` noncopyable types, these last two cases are listed for completeness,
since they would affect the behavior of other language features that
suppress implicit copying when applied to copyable types.)

The `consume` operator can always transfer ownership of its operand when the
`consume` expression is itself the operand of a consuming operation.

Consuming is flow-sensitive, so if one branch of an `if` or other control flow
consumes a noncopyable value, then other branches where the value
is not consumed may continue using it:

```swift
let x = FileDescriptor()
guard let condition = getCondition() else {
  consume(x)
  return
}
// We can continue using x here, since only the exit branch of the guard
// consumed it
use(x)
```

For the purposes of the following discussion, a closure is considered nonescaping
in the following cases:

- if the closure literal appears as an argument to a function parameter of
  non-`@escaping` function type, or
- if the closure literal is assigned to a local `let` variable, that does not
  itself get captured by an escaping closure.

These cases correspond to the cases where a closure is allowed to capture an
`inout` parameter from its surrounding scope, before this proposal.

### Borrowing operations

The following operations are borrowing:

- Passing an argument to a `func` or `subscript` parameter that does not
  have an ownership modifier, or an argument to any `func`, `subscript`, or
  `init` parameter which is explicitly marked `borrow`. The
  argument is borrowed for the duration of the callee's execution.
- Borrowing a stored property of a struct or tuple borrows the struct or tuple
  for the duration of the access to the stored property. This means that one
  field of a struct cannot be borrowed while another is being mutated, as in
  `call(struc.fieldA, &struc.fieldB)`. Allowing for fine-grained subelement
  borrows in some circumstances is discussed as a Future Direction below.
- A stored property of a class may be borrowed using a dynamic exclusivity
  check, to assert that there are no aliasing mutations attempted during the
  borrow, as discussed under "Noncopyable stored properties in classes" below.
- Invoking a `borrowing` method on a value, or a method which is not annotated
  as any of `borrowing`, `consuming` or `mutating`, borrows the `self` parameter
  for the duration of the callee's execution.
- Accessing a computed property or subscript through `borrowing` or
  `nonmutating` getter or setter borrows the `self` parameter for the duration
  of the accessor's execution.
- Capturing an immutable local binding into a nonescaping closure borrows the
  binding for the duration of the callee that receives the nonescaping closure.
- Assigning into the "black hole" `_ = x` borrows the right-hand side of the
  assignment.

### Mutating operations

The following operations are mutating uses:

- Passing an argument to a `func` parameter that is `inout`. The argument is
  exclusively accessed for the duration of the call.
- Projecting a stored property of a struct for mutation is a mutating use of
  the entire struct.
- A stored property of a class may be mutated using a dynamic exclusivity
  check, to assert that there are no aliasing mutations, as happens today.
  For noncopyable properties, the assertion also enforces that no borrows
  are attempted during the mutation, as discussed under "Noncopyable stored
  properties in classes" below.
- Invoking a `mutating` method on a value is a mutating use of the `self`
  parameter for the duration of the callee's execution.
- Accessing a computed property or subscript through a `mutating` getter and/or
  setter is a mutating use of `self` for the duration of the accessor's
  execution.
- Capturing a mutable local binding into a nonescaping closure is a mutating
  use of the binding for the duration of the callee that receives the
  nonescaping closure.

### Declaring functions and methods with noncopyable parameters

When noncopyable types are used as function parameters, the ownership
convention becomes a much more important part of the API contract.
As such, when a function parameter is declared with a noncopyable type, it
**must** declare whether the parameter uses the `borrowing`, `consuming`, or
`inout` convention:

```swift
// Redirect a file descriptor
// Require exclusive access to the FileDescriptor to replace it
func redirect(_ file: inout FileDescriptor, to otherFile: borrowing FileDescriptor) {
  dup2(otherFile.fd, file.fd)
}

// Write to a file descriptor
// Only needs shared access
func write(_ data: [UInt8], to file: borrowing FileDescriptor) {
  data.withUnsafeBytes {
    write(file.fd, $0.baseAddress, $0.count)
  }
}

// Close a file descriptor
// Consumes the file descriptor
func close(file: consuming FileDescriptor) {
  close(file.fd)
}
```

Methods of the noncopyable type are considered to be `borrowing` unless
declared `mutating` or `consuming`:

```swift
extension FileDescriptor {
  mutating func replace(with otherFile: borrowing FileDescriptor) {
    dup2(otherFile.fd, self.fd)
  }

  // borrowing by default
  func write(_ data: [UInt8]) {
    data.withUnsafeBytes {
      write(file.fd, $0.baseAddress, $0.count)
    }
  }

  consuming func close() {
    close(fd)
  }
}
```

Static casts or coercions of function types that change the ownership modifier
of a noncopyable parameter are currently invalid. One reason is that it is 
impossible to convert a function with a noncopyable `consuming` parameter, into
one where that parameter is `borrowed`, without inducing a copy of the borrowed
parameter. See Future Directions for details.

### Declaring properties of noncopyable type

A class or noncopyable struct may declare stored `let` or `var` properties of
noncopyable type. A noncopyable `let` stored property may only be borrowed,
whereas a `var` stored property may be both borrowed and mutated. Stored
properties cannot generally be consumed because doing so would leave the
containing aggregate in an invalid state.

Any type may also declare computed properties of noncopyable type. The `get`
accessor returns an owned value that the caller may consume, like a function
would. The `set` accessor receives its `newValue` as a `consuming` parameter,
so the setter may consume the parameter value to update the containing
aggregate.

Accessors may use the `consuming` and `borrowing` declaration modifiers to
affect the ownership of `self` while the accessor executes. `consuming get`
is particularly useful as a way of forwarding ownership of part of an aggregate,
such as to take ownership away from a wrapper type:

```swift
struct FileDescriptorWrapper: ~Copyable {
  private var _value: FileDescriptor

  var value: FileDescriptor {
    consuming get { return _value }
  }
}
```

However, a `consuming get` cannot be paired with a setter when the containing
type is `~Copyable`, because invoking the getter consumes the aggregate,
leaving nothing to write a modified value back to.

Because getters return owned values, non-`consuming` getters generally cannot
be used to wrap noncopyable stored properties, since doing so would require
copying the value out of the aggregate:

```swift
class File {
  private var _descriptor: FileDescriptor

  var descriptor: FileDescriptor {
    return _descriptor // ERROR: attempt to copy `_descriptor`
  }
}
```

These limitations could be addressed in the future by exposing the ability for
computed properties to also provide "read" and "modify" coroutines, which would
have the ability to yield borrowing or mutating access to properties without
copying them.

### Using stored properties and enum cases of noncopyable type

When classes or noncopyable types contain members that are of noncopyable
type, then the container is the unique owner of the member value. Outside of
the type's definition, client code cannot perform consuming operations on
the value, since it would need to take away the container's ownership to do
so:

```swift
struct Inner: ~Copyable {}

struct Outer: ~Copyable {
  var inner = Inner()
}

let outer = Outer()
let i = outer.inner // ERROR: can't take `inner` away from `outer`
```

However, when code has the ability to mutate the member, it may freely modify,
reassign, or replace the value in the field:

```swift
var outer = Outer()
let newInner = Inner()
// OK, transfers ownership of `newInner` to `outer`, destroying its previous
// value
outer.inner = newInner
```

Note that, as currently defined, `switch` to pattern-match an `enum` is a
consuming operation, so it can only be performed inside `consuming` methods
on the type's original definition:

```swift
enum OuterEnum: ~Copyable {
  case inner(Inner)
  case file(FileDescriptor)
}

// Error, can't partially consume a value outside of its definition
let enum = OuterEnum.inner(Inner())
switch enum {
case .inner(let inner):
  break
default:
  break
}
```

Being able to borrow in pattern matches would address this shortcoming.

### Noncopyable stored properties in classes

Since objects may have any number of simultaneous references, Swift uses
dynamic exclusivity checking to prevent simultaneous writes of the same
stored property. This dynamic checking extends to borrows of noncopyable
stored properties; the compiler will attempt to diagnose obvious borrowing
failures, as it will for local variables and value types, but a runtime error
will occur if an uncaught exclusivity error occurs, such as an attempt to mutate
an object's stored property while it is being borrowed:

```swift
class Foo {
  var fd: FileDescriptor

  init(fd: FileDescriptor) { self.fd = fd }
}

func update(_: inout FileDescriptor, butBorrow _: borrow FileDescriptor) {}

func updateFoo(_ a: Foo, butBorrowFoo b: Foo) {
  update(&a.fd, butBorrow: b.fd)
}

let foo = Foo(fd: FileDescriptor())

// Will trap at runtime when foo.fd is borrowed and mutated at the same time
updateFoo(foo, butBorrowFoo: foo)
```

`let` properties do not allow mutating accesses, and this continues to hold for
noncopyable types. The value of a `let` property in a class therefore does not
need dynamic checking, even if the value is noncopyable; the value behaves as
if it is always borrowed, since there may potentially be a borrow through
some reference to the object at any point in the program. Such values can
thus never be consumed or mutated.

The dynamic borrow state of properties is tracked independently for every
stored property in the class, so it is safe to mutate one property while other
properties of the same object are also being mutated or borrowed:

```swift
class SocketTriple {
  var in, middle, out: FileDescriptor
}

func update(_: inout FileDescriptor, and _: inout FileDescriptor,
            whileBorrowing _: borrowing FileDescriptor) {}

// This is OK
let object = SocketTriple(...)
update(&object.in, and: &object.out, whileBorrowing: object.middle)
```

This dynamic tracking, however, cannot track accesses at finer resolution
than properties, so in circumstances where we might otherwise eventually be
able to support independent borrowing of fields in structs, tuples, and enums,
that support will not extend to fields within class properties, since the
entire property must be in the borrowing or mutating state.

Dynamic borrowing or mutating accesses require that the enclosing object be
kept alive for the duration of the assertion of the access. Normally, this 
is transparent to the developer, as the compiler will keep a copy of a
reference to the object retained while these accesses occur. However, if
we introduce noncopyable bindings to class references, such as [the `borrow`
and `inout` bindings](https://forums.swift.org/t/pitch-borrow-and-inout-declaration-keywords/62366)
currently being pitched, this would manifest as a borrow of the noncopyable
reference, preventing mutation or consumption of the reference during
dynamically-asserted accesses to its properties:

```swift
class SocketTriple {
  var in, middle, out: FileDescriptor
}

func borrow(_: borrowing FileDescriptor,
            whileReplacingObject _: inout SocketTriple) {}

var object = SocketTriple(...)

// This is OK, since ARC will keep a copy of the `object` reference retained
// while `object.in` is borrowed
borrow(object.in, whileReplacingObject: &object)

inout objectAlias = &object

// This is an error, since we aren't allowed to implicitly copy through
// an `inout` binding, and replacing `objectAlias` without keeping a copy
// retained might invalidate the object while we're accessing it.
borrow(objectAlias.in, whileReplacingObject: &objectAlias)
```

### Noncopyable variables captured by escaping closures

Nonescaping closures have scoped lifetimes, so they can borrow their captures,
as noted in the "borrowing operations" and "consuming operations" sections
above. Escaping closures, on the other hand, have indefinite lifetimes, since
they can be copied and passed around arbitrarily, and multiple escaping closures
can capture and access the same local variables alongside the local context
from which those captures were taken. Variables captured by escaping closures
thus behave like class properties; immutable captures are treated as always
borrowed both inside the closure body and in the capture's original context.

```swift
func escape(_: @escaping () -> ()) {...}

func borrow(_: borrowing FileDescriptor) {}
func consume(_: consuming FileDescriptor) {}

func foo() {
  let x = FileDescriptor()

  // ERROR: cannot consume variable before it's been captured
  consume(x)

  escape {
    borrow(x) // OK
    consume(x) // ERROR: cannot consume captured variable
  }

  // OK
  borrow(x)

  // ERROR: cannot consume variable after it's been captured by an escaping
  // closure
  consume(x)
}
```

Mutable captures are subject to dynamic exclusivity checking like class
properties are, and similarly cannot be consumed and reinitialized. When
a closure escapes, the compiler isn't able to statically know when the closure
is invoked, and it may even be invoked multiple overlapping times, or
simultaneously on different threads if the closure is `@Sendable`, so the
captures must always remain in a valid state for memory safety, and exclusivity
of mutations can only be enforced dynamically.

```swift
var escapedClosure: (@escaping (inout FileDescriptor) -> ())?

func foo() {
  var x = FileDescriptor()

  // ERROR: cannot consume variable before it's been captured.
  // (We could potentially support local consumption before the variable
  // capture occurs as a future direction.)
  consume(x)
  x = FileDescriptor()

  escapedClosure = { _ in borrow(x) }

  // Runtime error when exclusive access to `x` dynamically conflicts
  // with attempted borrow of `x` during `escapedClosure`'s execution
  escapedClosure!(&x)
}
```

### Deinitializers

A noncopyable struct or enum may declare a `deinit`, which will run
implicitly when the lifetime of the value ends (unless explicitly suppressed
with `discard` as explained below):

```swift
struct File: ~Copyable {
  var descriptor: Int32

  func write<S: Sequence>(_ values: S) { /*..*/ }

  consuming func close() {
    print("closing file")
  }

  deinit {
    print("deinitializing file")
    closeFile(rawDescriptor: descriptor)
  }
}
```

Like a class `deinit`, a struct or enum `deinit` may not propagate any
uncaught errors. Within the body of the `deinit`, `self` behaves as in
a `borrowing` method; it may not be modified or consumed inside the
`deinit`. (Allowing for mutation and partial invalidation inside a
`deinit` is explored as a future direction.)

A value's lifetime ends, and its `deinit` runs if present, in the following
circumstances:

- For a local `var` or `let` binding, or `consuming` function parameter, that is
  not itself consumed, `deinit` runs at the end of the binding's lexical
  scope. If, on the other hand, the binding is consumed, then responsibility
  for deinitialization gets forwarded to the consumer (which may in turn forward
  it somewhere else). As explained later, a `_ = consume` operator with no
  destination immediately runs the `deinit`.

    ```swift
    do {
      let file = File(descriptor: 42)
      file.close() // consuming use
      // file's deinit runs inside `close`
      print("done writing")
    }
    // Output:
    //   closing file
    //   deinitializing file
    //   done writing
   
    do {
      let file = File(descriptor: 42)
      file.write([1,2,3]) // borrowing use
      print("done writing")
      // file's deinit runs here
    }
    // Output:
    //   done writing
    //   deinitializing file
    ```

    If a noncopyable value is conditionally consumed, then the deinitializer
    runs as late as possible on any nonconsumed paths:

    ```swift
    let condition = false
    do {
      let file = File(descriptor: 42)
      file.write([1,2,3]) // borrowing use
      if condition {
        file.close()
      } else {
        print("not closed")
        // file's deinit runs here
      }
      print("done writing")
    }
    // Output:
    //   not closed
    //   deinitializing file
    //   done writing
    ```

- When a struct, enum, or class contains a member of noncopyable type, the member is destroyed, and its deinit is
run, after the container's deinit runs. For example:

```swift
struct Inner: ~Copyable {
  deinit { print("destroying inner") }
}

struct Outer: ~Copyable {
  var inner = Inner()
  deinit { print("destroying outer") }
}

do {
  _ = Outer()
}
```

will print:
```
destroying outer
destroying inner
```

### Suppressing `deinit` in a `consuming` method

It is often useful for noncopyable types to provide alternative ways to consume
the resource represented by the value besides `deinit`. However,
under normal circumstances, a `consuming` method will still invoke the type's
`deinit` after the last use of `self`, which is undesirable when the method's
own logic already invalidates the value:

```swift
struct FileDescriptor: ~Copyable {
  private var fd: Int32

  deinit {
    close(fd)
  }

  consuming func close() {
    close(fd)

    // The lifetime of `self` ends here, triggering `deinit` (and another call to `close`)!
  }
}
```

In the above example, the double-close could be avoided by having the
`close()` method do nothing on its own and just allow the `deinit` to
implicitly run. However, we may want the method to have different behavior
from the deinit; for example, it could raise an error (which a normal `deinit`
is unable to do) if the `close` system call triggers an OS error :

```swift
struct FileDescriptor: ~Copyable {
  private var fd: Int32

  consuming func close() throws {
    // POSIX close may raise an error (which leaves the file descriptor in an
    // unspecified state, so we can't really try to close it again, but the
    // error may nonetheless indicate a condition worth handling)
    if close(fd) != 0 {
      throw CloseError(errno)
    }

    // We don't want to trigger another close here!
  }
}
```

or it could be useful to take manual control of the file descriptor back from
the type, such as to pass to a C API that will take care of closing it:

```swift
struct FileDescriptor: ~Copyable {
  // Take ownership of the C file descriptor away from this type,
  // returning the file descriptor without closing it
  consuming func take() -> Int32 {
    return fd

    // We don't want to trigger close here!
  }
}
```

We propose to introduce a special operator, `discard self`, which ends the
lifetime of `self` without running its `deinit`:

```swift
struct FileDescriptor: ~Copyable {
  // Take ownership of the C file descriptor away from this type,
  // returning the file descriptor without closing it
  consuming func take() -> Int32 {
    let fd = self.fd
    discard self
    return fd
  }
}
```

`discard self` can only be applied to `self`, in a consuming method
defined in the same file as the type's original definition. (This is in
contrast to Rust's similar special function,
[`mem::forget`](https://doc.rust-lang.org/std/mem/fn.forget.html), which is a
standalone function that can be applied to any value, anywhere.  Although the
Rust documentation notes that this operation is "safe" on the principle that
destructors may not run at all, due to reference cycles, process termination,
etc., in practice the ability to forget arbitrary values creates semantic
issues for many Rust APIs, particularly when there are destructors on types
with lifetime dependence on each other like `Mutex` and `LockGuard`. As such,
we think it is safer to restrict the ability to suppress the standard `deinit`
for a value to the core API of its type. We can relax this restriction if
experience shows a need to.)

For the extent of this proposal, we also propose that `discard self` can only
be applied in types whose components include no reference-counted, generic,
or existential fields, nor do they include any types that transitively include
any fields of those types or that have `deinit`s defined of their own. (Such
a type might be called "POD" or "trivial" following C++ terminology). We explore
lifting this restriction as a future direction.


Even with the ability to `discard self`, care would still need be taken when
writing destructive operations to avoid triggering the deinit on alternative
exit paths, such as early `return`s, `throw`s, or implicit propagation of
errors from `try` operations. For instance, if we write:

```swift
struct FileDescriptor: ~Copyable {
  private var fd: Int32

  consuming func close() throws {
    // POSIX close may raise an error (which still invalidates the
    // file descriptor, but may indicate a condition worth handling)
    if close(fd) != 0 {
      throw CloseError(errno)
      // !!! Oops, we didn't suppress deinit on this path, so we'll double close!
    }

    // We don't need to deinit self anymore
    discard self
  }
}
```

then the `throw` path exits the method without `discard`, and
`deinit` will still execute if an error occurs. To avoid this mistake, we
propose that if any path through a method uses `discard self`, then
**every** path must choose either to `discard` or to explicitly `consume self`,
which triggers the standard `deinit`. This will make the above code an error,
alerting that the code should be rewritten to ensure `discard self`
always executes:

```swift
struct FileDescriptor: ~Copyable {
  private var fd: Int32

  consuming func close() throws {
    // Save the file descriptor and give up ownership of it
    let fd = self.fd
    discard self

    // We can now use `fd` below without worrying about `deinit`:

    // POSIX close may raise an error (which still invalidates the
    // file descriptor, but may indicate a condition worth handling)
    if close(fd) != 0 {
      throw CloseError(errno)
    }
  }
}
```

The [consume operator](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0377-parameter-ownership-modifiers.md)
must be used to explicitly end the value's lifetime using its `deinit` if
`discard` is used to conditionally destroy the value on other paths
through the method.

```swift
struct MemoryBuffer: ~Copyable {
  private var address: UnsafeRawPointer

  init(size: Int) throws {
    guard let address = malloc(size) else {
      throw MallocError()
    }
    self.address = address
  }

  deinit {
    free(address)
  }

  consuming func takeOwnership(if condition: Bool) -> UnsafeRawPointer? {
    if condition {
      // Save the memory buffer and give it to the caller, who
      // is promising to free it when they're done.
      let address = self.address
      discard self
      return address
    } else {
      // We still want to free the memory if we aren't giving it away.
      _ = consume self
      return nil
    }
  }
}
```

## Source compatibility

For existing Swift code, this proposal is additive.

## Effect on ABI stability

### Adding or removing `Copyable` breaks ABI

An existing copyable struct or enum cannot have its `Copyable` capability
taken away without breaking ABI, since existing clients may copy values of the
type.

Ideally, we would allow noncopyable types to become `Copyable` without breaking
ABI; however, we cannot promise this, due to existing implementation choices we
have made in the ABI that cause the copyability of a type to have unavoidable
knock-on effects. In particular, when properties are declared in classes,
protocols, or public non-`@frozen` structs, we define the property's ABI to use
accessors even if the property is stored, with the idea that it should be
possible to change a property's implementation to change it from a stored to
computed property, or vice versa, without breaking ABI.

The accessors used as ABI today are the traditional `get` and `set`
computed accessors, as well as a `_modify` coroutine which can optimize `inout`
operations and projections into stored properties. `_modify` and `set` are
not problematic for noncopyable types. However, `get` behaves like a
function, producing the property's value by returning it like a function would,
and returning requires *consuming* the return value to transfer it to the
caller. This is not possible for noncopyable stored properties, since the
value of the property cannot be copied in order to return a copy without
invalidating the entire containing struct or object.

Therefore, properties of noncopyable type need a different ABI in order to
properly abstract them. In particular, instead of exposing a `get` accessor
through abstract interfaces, they must use a `_read` coroutine, which is the
read-only analog to `_modify`, allowing the implementation to yield a borrow of
the property value in-place instead of returning by value. This allows for
noncopyable stored properties to be exposed while still being abstracted enough
that they can be replaced by a computed implementation, since a `get`-based
implementation could still work underneath the `read` coroutine by evaluating
the getter, yielding a borrow of the returned value, then disposing of the
temporary value.

As such, we cannot simply say that making a noncopyable type copyable is an
ABI-safe change, since doing so will have knock-on effects on the ABI of any
properties of the type. We could potentially provide a "born noncopyable"
attribute to indicate that a copyable type should use the noncopyable ABI
for any properties, as a way to enable the evolution into a copyable type
while preserving existing ABI. However, it also seems unlikely to us that many
types would need to evolve between being copyable or not frequently.

### Adding, removing, or changing `deinit` in a struct or enum

An noncopyable type that is not `@frozen` can add or remove its deinit without
affecting the type's ABI. If `@frozen`, a deinit cannot be added or removed,
but the deinit implementation may change (if the deinit is not additionally
`@inlinable`).

### Adding noncopyable fields to classes

A class may add fields of noncopyable type without changing ABI.

## Effect on API resilience

Introducing new APIs using noncopyable types is an additive change. APIs that
adopt noncopyable types have some notable restrictions on how they can further
evolve while maintaining source compatibility.

A noncopyable type can be made copyable while generally maintaining source
compatibility. Values in client source would acquire normal ARC lifetime
semantics instead of eager-move semantics when those clients are recompiled
with the type as copyable, and that could affect the observable order of
destruction and cleanup. Since copyable value types cannot directly define
`deinit`s, being able to observe these order differences is unlikely, but not
impossible when references to classes are involved.

A `consuming` parameter of noncopyable type can be changed into a `borrowing`
parameter without breaking source for clients (and likewise, a `consuming`
method can be made `borrowing`). Conversely, changing
a `borrowing` parameter to `consuming` may break client source. (Either direction
is an ABI breaking change.) This is because a consuming use is required to
be the final use of a noncopyable value, whereas a borrowing use may or may not
be.

Adding or removing a `deinit` to a noncopyable type does not affect source
for clients.

## Alternatives considered

### Naming the attribute "move-only"

We have frequently referred to these types as "move-only types" in various
vision documents. However, as we've evolved related proposals like the
`consume` operator and parameter modifiers, the community has drifted away
from exposing the term "move" in the language elsewhere. When explaining these
types to potential users, we've also found that the name "move-only" incorrectly
suggests that being noncopyable is a new capability of types, and that there
should be generic functions that only operate on "move-only" types, when really
the opposite is the case: all existing types in Swift today conform to
effectively an implicit "Copyable" requirement, and what this feature does is
allow types not to fulfill that requirement. When generics grow support for
move-only types, then generic functions and types that accept noncopyable
type parameters will also work with copyable types, since copyable types
are strictly more capable. This proposal prefers the term "noncopyable" to make
the relationship to an eventual `Copyable` constraint, and the fact that annotated
types lack the ability to satisfy this constraint, more explicit.

### Spelling as a generic constraint

It's a reasonable question why declaring a type as noncopyable isn't spelled
like a regular protocol constraint, instead of as the removal of an existing
constraint:

```swift
struct Foo: NonCopyable {}
```

As noted in the previous discussion, an issue with this notation is that it
implies that `NonCopyable` is a new capability or requirement, rather than
really being the lack of a `Copyable` capability. For an example of why
this might be misleading, consider what would happen if we expand
standard library collection types to support noncopyable elements. Value types
like `Array` and `Dictionary` would become copyable only when the elements they
contain are copyable. However, we cannot write this in terms of `NonCopyable`
conditional requirements, since if we write:

```swift
extension Dictionary: NonCopyable where Key: NonCopyable, Value: NonCopyable {}
```

this says that the dictionary is noncopyable only when both the key and value
are noncopyable, which is wrong because we can't copy the dictionary even if
only the keys or only the values are noncopyable. If we flip the constraint to
`Copyable`, the correct thing would fall out naturally:

```swift
extension Dictionary: Copyable where Key: Copyable, Value: Copyable {}
```

However, for progressive disclosure and source compatibility reasons, we still
want the majority of types to be `Copyable` by default without making them
explicitly declare it; noncopyable types are likely to remain the exception
rather than the rule, with automatic lifetime management via ARC by the
compiler being sufficient for most code like it is today.

### English language bikeshedding

Some dictionaries specify that "copiable" is the standard spelling for "able to
copy", although the Oxford English Dictionary and Merriam-Webster both also
list "copyable" as an accepted alternative. We prefer the more regular "copyable"
spelling.

## Future directions

### Noncopyable tuples

It should be possible for a tuple to contain noncopyable elements, rendering
the tuple noncopyable if any of its elements are. Since tuples' structure is
always known, it would be reasonable to allow for the elements within a tuple
to be independently borrowed, mutated, and consumed, as the language allows
today for the elements of a tuple to be independently mutated via `inout`
accesses. (Due to the limitations of dynamic exclusivity checking, this would
not be possible for class properties, globals, and escaping closure captures.)

### Noncopyable `Optional`

This proposal initiates support for noncopyable types without any support for
generics at all, which precludes their use in most standard library types,
including `Optional`. We expect the lack of `Optional` support in particular
to be extremely limiting, since `Optional` can be used to manage dynamic
consumption of noncopyable values in situations where the language's static
rules cannot soundly support consumption. For instance, the static rules above
state that a stored property of a class can never be consumed, because it is
not knowable if other references to an object exist that expect the property
to be inhabited. This could be avoided using `Optional` with `mutating`
operation that forwards ownership of the `Optional` value's payload, if any,
writing `nil` back. Eventually this could be written as an extension method
on `Optional`:

```swift
extension Optional where Self: ~Copyable {
  mutating func take() -> Wrapped {
    switch self {
    case .some(let wrapped):
      self = nil
      return wrapped
    case .none:
      fatalError("trying to take from an Optional that's already empty")
    }
  }
}

class Foo {
  var fd: FileDescriptor?

  func close() {
    // We normally would not be able to close `fd` except via the
    // object's `deinit` destroying the stored property. But using
    // `Optional` assignment, we can dynamically end the value's lifetime
    // here.
    fd = nil
  }

  func takeFD() -> FileDescriptor {
    // We normally would not be able to forward `fd`'s ownership to
    // anyone else. But using
    // `Optional.take`, we can dynamically end the value's lifetime
    // here.
    return fd.take()
  }
}
```

Without `Optional` support, the alternative would be for every noncopyable type
to provide its own ad-hoc `nil`-like state, which would be very unfortunate,
and go against Swift's general desire to encourage structural code correctness
by making invalid states unrepresentable. Therefore, `Optional` is likely to
be worth considering as a special case for noncopyable support, ahead of full
generics support for noncopyable types.

### Generics support for noncopyable types

This proposal comes with an admittedly severe restriction that noncopyable types
cannot conform to protocols or be used at all as type arguments to generic
functions or types, including common standard library types like `Optional`
and `Array`. All generic parameters in Swift today carry an implicit assumption
that the type is copyable, and it is another large language design project to
integrate the concept of noncopyable types into the generics system. Full
integration will very likely also involve changes to the Swift runtime and
standard library to accommodate noncopyable types in APIs that weren't
originally designed for them, and this integration might then have backward
deployment restrictions. We believe that, even with these restrictions,
noncopyable types are a useful self-contained addition to the language for
safely and efficiently modeling unique resources, and this subset of the feature
also has the benefit of being adoptable without additional runtime requirements,
so developers can begin making use of the feature without giving up backward
compatibility with existing Swift runtime deployments.

### Conditionally copyable types

This proposal states that a type, including one with generic parameters, is
currently always copyable or always noncopyable. However, some types may
eventually be generic over copyable and non-copyable types, with the ability
to be copyable for some generic arguments but not all. A simple case might be
a tuple-like `Pair` struct:

```swift
struct Pair<T: ~Copyable, U: ~Copyable>: ~Copyable {
  var first: T
  var second: U
}
```

We will need a way to express this conditional copyability, perhaps using
conditional conformance style declarations:

```swift
extension Pair: Copyable where T: Copyable, U: Copyable {}
```

### Suppressing implicitly derived conformances with `~Constraint`

There are situations where a type's conformance to a protocol is implicitly 
derived because of aspects of its declaration or usage. For instance, enums that
don't have any associated values are implicitly made `Hashable` (and,
by refinement, `Equatable`):

```swift
enum Foo {
  case a, b, c
}

// OK to compare with `==` because `Foo` is automatically Equatable,
// through an implementation of `==` synthesized by the compiler for you.
print(Foo.a == Foo.b)
```

and internal structs and enums are implicitly `Sendable` if all of their
components are `Sendable`:

```swift
struct Bar {
    var x: Int, y: Int
}

func foo() async {
    let x = Bar(x: 17, y: 38)

    // OK to use x in an async task because it's implicitly Sendable
    async let y = x
}
```

However, this isn't always desirable; an enum may want to reserve the right to
add associated values in the future that aren't `Equatable`, or a type may be
made up of `Sendable` components that represent resources that are not safe
to share across threads. There is currently no direct way to suppress these
automatically derived conformances. We propose to introduce the `~Constraint`
syntax as a way to explicitly suppress automatic derivation of a conformance 
that would otherwise be performed for a declaration:

```swift
enum Candy: ~Equatable {
    case redVimes, twisslers, smickers
}

// ERROR: `Candy` does not conform to `Equatable`
print(Candy.redVimes == Candy.twisslers)

struct ThreadUnsafeHandle: ~Sendable {
    // although this is an integer, it represents a system resource that
    // can only be accessed from a specific thread, and should not be shared
    // across threads
    var handle: Int32 
}

func foo(handle: ThreadUnsafeHandle) async {
    // ERROR: `ThreadUnsafeHandle` is not `Sendable`
    async let y = handle
}
```

It is important to note that `~Constraint` only avoids the implicit, automatic
derivation of conformance. It does **not** mean that the type strictly does
not conform to the protocol. Extensions may add the conformance back separately,
possibly conditionally:

```swift
struct ResourceHandle<T: Resource>: ~Sendable {
    // although this is an integer, it represents a system resource that
    // gives access to values of type `T`, which may not be thread safe
    // across threads
    var handle: Int32 
}

// It is safe to share the handle when the resource type is thread safe
extension ResourceHandle: Sendable where T: Sendable {}

// Suppress the implicit Equatable (and Hashable) derivation...
enum Candy: ~Equatable {
    case redVimes, twisslers, smickers
}

// ... and still add an Equatable conformance.
extension Candy: Equatable {
    static func ==(a: Candy, b: Candy) -> Bool {
        switch (a, b) {
        // RedVimes are considered equal to Twisslers
        case (.redVimes, .redVimes), (.twisslers, .twisslers), 
             (.smickers, .smickers), (.twisslers, .redVimes)
             (.redVimes, .twisslers):
            return true
        default:
            return false
        }
    }
}
```

Keep in mind that `~Constraint` is not required to suppress Swift's synthesized implementations of protocol requirements. For example, if you only want to
provide your own implementation of `==` for an enum, but are fine with Equatable
(and Hashable, etc) being derived for you, then the derivation of `Equatable` 
already will use your version of `==`.

```swift
enum Soda {
    case mxPepper, drPibb, doogh

    // This is used instead of a synthesized `==` when
    // implicitly deriving the Equatable conformance
    static func ==(a: Soda, b: Soda) -> Bool {
      switch (a, b) {
        case (.doogh, .doogh): return true
        case (_, .doogh), (.doogh, _): return false
        default: return true
      }
    } 
}
```

### Allowing `deinit` to mutate or consume `self`, while avoiding accidental recursion

During destruction, `deinit` formally has sole ownership of `self`, so it
is possible to allow `deinit` to mutate or consume `self` as part of
deinitialization. However, inside of other `mutating` or `consuming` methods,
it's easy to inadvertently trigger implicit destruction of the value and
reenter `deinit` again:

```swift
struct Foo: ~Copyable {
  init() { ... }

  consuming func consumingHelper() {
    // If a consuming method does nothing else, it will run `deinit`
  }

  mutating func mutatingHelper() {
    // A mutating method may consume and reassign self, indirectly triggering
    // an implicit deinit
    consumingHelper()
    self = .init()
  }

  deinit {
    // mutatingHelper calls consumingHelper, which calls deinit again, leading to an infinite loop
    mutatingHelper() 
  }
}
```

Since this is an easy trap to fall into, before we allow `deinit` to mutate
or consume `self`, it's worth considering whether there are any constraints we
could impose to make it less likely to get into an infinite
`deinit` loop situation when doing so. Some possibilities include:

* We could say that the value remains immutable during `deinit`. Many types
  don't need to modify their internal state for cleanup, especially if they
  only store a pointer or handle to some resource. This seems overly
  restrictive for other kinds of types that have direct ownership of resources,
  though.
* We could say that individual *fields* of the value inside of `deinit` are
  mutable and consumable, but that the value as a whole is not. This would
  allow for `deinit` to individually mutate and/or forward ownership of
  elements of the value, but not pass off the entire value to be mutated or
  consumed (and potentially re-deinited). This would allow for `deinit`s to
  implement logic that modifies or consumes part of the value, but they
  wouldn't be allowed to use any methods of the type, other than maybe
  `borrowing` methods, to share implementation logic with other members of the
  type.
* Since `deinit` must be declared as part of the original type declaration, any
  nongeneric methods that it can possibly call on the type must be defined in
  the same module as the `deinit`, so we could potentially do some local
  analysis of those methods. We could raise a warning or error if a method
  called from the deinit either visibly contains any implicit deinit calls
  itself, or cannot be analyzed because it's generic, from a protocol
  extension, etc.
* We could do nothing and leave it in developers' hands to understand why
  deinit loops happen when they do.

### Finer-grained destructuring in `consuming` methods and `deinit`

As currently specified, noncopyable types are (outside of `init` implementations)
always either fully initialized or fully destroyed, without any support
for incremental destruction even inside of `consuming` methods or deinits. A
`deinit` may modify, but not invalidate, `self`, and a `consuming` method may
`discard self`, forward ownership of all of `self`, or destroy `self`, but
cannot yet partially consume parts of `self`. This would be particularly useful
for types that contain other noncopyable types, which may want to relinquish
ownership of some or all of the resources owned by those members. In the
current proposal, this isn't possible without allowing for an intermediate
invalid state:

```swift
struct SocketPair: ~Copyable {
  let input, output: FileDescriptor

  // Gives up ownership of the output end, closing the input end
  consuming func takeOutput() -> FileDescriptor {
    // We would like to do something like this, taking ownership of
    // `self.output` while leaving `self.input` to be destroyed.
    // However, we can't do this without being able to either copy
    // `self.output` or partially invalidate `self`.
    return self.output
  }
}
```

Analogously to how `init` implementations use a "definite initialization"
pass to allow the value to initialized field-by-field, we can implement the
inverse dataflow pass to allow `deinit` implementations to partially
invalidate `self`. This analysis would also enable `consuming` methods to
partially invalidate `self` in cases where either the type has no `deinit` or,
as discussed in the following section, `discard self` is used to disable the
`deinit` in cases when the value is partially invalidated.

### Generalizing `discard self` for types with component cleanups

The current proposal limits the use of `discard self` to types that don't have
any fields that require additional cleanup, meaning that it cannot be used in
a type that has class, generic, existential, or other noncopyable type fields.
Allowing this would be an obvious generalization; however, allowing it requires
answering some design questions:

- When `self` is discarded, are its fields still destroyed?
- Is access to `self`'s fields still allowed after `discard self`? In other
  words, does `discard self` immediately consume all of `self`, running
  the cleanups for its elements at the point where the `discard` is executed,
  or does it only disable the `deinit` on `self`, allowing the fields to
  still be individually borrowed, mutated, and/or consumed, and leaving them
  to be cleaned up when their individual lifetimes end?

Although Rust's `mem::forget` completely leaks its operand, including its fields,
the authors of this proposal generally believe that is undesirable, so we expect
that `discard self` should only disable the type's own `deinit` while still
leaving the components of `self` to be cleaned up.

The choice of what effect `discard` has on the lifetime of the fields affects
the observed order in which field deinits occurs, but also affects how code
would be expressed that performs destructuring or partial invalidation:

```swift
struct SocketPair: ~Copyable {
  let input, output: FileDescriptor

  deinit { ... }

  enum End { case input, output }

  // Give up ownership of one end and closes the other end
  consuming func takeOneEnd(which: End) -> FileDescriptor {
    // If a consuming method could partially invalidate self, would it do it
    // like this...
#if discard_immediately_consumes_whats_left_of_self
    switch which {
    case .input:
        // Move out the field we want 
        let result = self.input
        // Destroy the rest of self
        discard self
        return result

    case .output:
        let result = self.output
        discard self
        return result
    }

    // ...or like this
#elseif discard_only_disables_deinit
    // Disable deinit on self, which subsequently allows individual consumption
    // of its fields
    discard self

    switch which {
    case .input:
        return self.input
    case .output:
        return self.output
    }
#endif
  }
}
```

### `read` and `modify` accessor coroutines for computed properties

The current computed property model allows for properties to provide a getter,
which returns the value of the property on read to the caller as an owned value,
and optionally a setter, which receives the `newValue` of the property as
a parameter with which to update the containing type's state. This is
sometimes inefficient for value types, since the get/set pattern requires
returning a copy, modifying the copy, then passing the copy back to the setter
in order to model an in-place update, but it also limits what computed
properties can express for noncopyable types. Because a getter has to return
by value, it cannot pass along the value of a stored noncopyable property
without also destroying the enclosing aggregate, so `get`/`set` cannot be used
to wrap logic around access to a stored noncopyable property.

The Swift stable ABI for properties internally uses **accessor coroutines**
to allow for efficient access to stored properties, while still providing
abstraction that allows library evolution to change stored properties into
computed and back. These coroutines **yield** access to a value in-place for
borrowing or mutating, instead of passing copies of values back and forth.
We can expose the ability for code to implement these coroutines directly,
which is a good optimization for copyable value types, but also allows for
more expressivity with noncopyable properties.

### Static casts of functions with ownership modifiers

The rule for casting function values via `as` or some other static, implicit 
coercion is that a noncopyable parameter's ownership modifier must remain the 
same. But there are some cases where static conversions of functions 
with noncopyable parameters are safe. It's not safe in general to do any dynamic
casts of function values, so `as?` and `as!` are excluded.

One reason behind the currently restrictive rule for static casts is a matter of
scope for this proposal. There may be a broader demand to support such casts
even for copyable types. For example, it should be safe to allow a cast to
change a `borrowing` parameter into one that is `inout`, as it only adds a
capability (mutation) that is not actually used by the underlying function:
```swift
// This could be possible, but currently is not.
{ (x: borrowing SomeType) in () } as (inout SomeType) -> ()
```
The second reason is that some casts are _only_ valid for copyable types.
In particular, a cast that changes a `consuming` parameter into one that is
`borrowing` is only valid for copyable types, because a copy of the borrowed
value is required to provide a non-borrowed value to the underlying function.
```swift
// String is copyable, so both are OK and currently permitted.
{ (x: borrowing String) in () } as (consuming String) -> ()
{ (x: consuming String) in () } as (borrowing String) -> ()
// FileDescriptor is noncopyable, so it cannot go from consuming to borrowing:
{ (x: consuming FileDescriptor) in () } as (borrowing String) -> ()
// but the reverse could be permitted in the future:
{ (x: borrowing FileDescriptor) in () } as (consuming String) -> ()
```

## Revision history

This revision makes the following changes from the [second reviewed revision](https://github.com/swiftlang/swift-evolution/blob/a9e21e3a4eb9526f998915c6554c7c72e5885a91/proposals/0390-noncopyable-structs-and-enums.md)
in response to Language Steering Group review and implementation experience:

- `_ = x` is now a borrowing operation.
- `switch` and `if/while case` require the subject of a pattern match to use
  the `consume x` operator. The fact that they are consuming operations now
  is an artifact of the implementation, and with further development, we may
  want to make the default semantics of `switch x` without explicit consumption
  to be borrowing.
- Escaped closure captures are constrained from being consumed for their
  entire lifetime, even before the closure that escapes it is formed. This
  analysis was not practical to implement using our current analysis, and the
  added expressivity is unlikely to be worth the implementation complexity.
- `self` in a deinit is currently constrained to be immutable, since there
  is [ongoing discussion](https://forums.swift.org/t/se-0390-noncopyable-type-deinit-s-mutation-and-accidental-recursion/64767)
  about how best to manage mutation or consumption during deinits while
  managing the possibility to accidentally cause recursion into `deinit`
  by implicit destruction.

The [second reviewed revision](https://github.com/swiftlang/swift-evolution/blob/a9e21e3a4eb9526f998915c6554c7c72e5885a91/proposals/0390-noncopyable-structs-and-enums.md)
of the proposal made the following changes from the
[first reviewed revision](https://github.com/swiftlang/swift-evolution/blob/5d075b86d57e3436b223199bd314b2642e30045f/proposals/0390-noncopyable-structs-and-enums.md):

- The original revision did not provide a `Copyable` generic constraint, and
  declared types as noncopyable using a `@noncopyable` attribute. The
  language workgroup believes that it is a good idea to build toward a future
  where noncopyable types are integrated with the language's generics system,
  and that the syntax for suppressing generic constraints is a good general
  notation to have for suppressing implicit conformances or assumptions about
  generic capabilities we may take away in the future, so it makes sense to
  provide a syntax that allows for growth in those directions.

- The original revision suppressed implicit `deinit` within methods using the
  spelling `forget self`. Although the term `forget` has a precedent in Rust,
  the behavior of `mem::forget` in Rust doesn't correspond to the semantics of
  the operation proposed here, and the language workgroup doesn't find the
  term clear enough on its own. This revision of the proposal chooses the
  `discard` as a starting point for further review discussion. Furthermore,
  we limit its use to types whose contents are otherwise trivial, in order to
  avoid committing to interactions with elementwise consumption of fields
  that we may want to refine later.

- The original revision allowed for a `consuming` method declared anywhere in
  the type's original module to suppress `deinit`. This revision narrows the
  capability to only methods declared in the same file as the type, for
  consistency with other language features that depend on having visibility into
  a type's entire layout, such as implicit `Sendable` inference.
