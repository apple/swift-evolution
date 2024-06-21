# Introduce user-defined dynamically "callable" types

* Proposal: [SE-0216](0216-dynamic-callable.md)
* Authors: [Chris Lattner](https://github.com/lattner), [Dan Zheng](https://github.com/dan-zheng)
* Review Manager: [John McCall](https://github.com/rjmccall)
* Implementation: [apple/swift#20305](https://github.com/apple/swift/pull/20305)
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-se-216-user-defined-dynamically-callable-types/14110)
* Status: **Implemented (Swift 5.0)**

## Introduction

This proposal is a follow-up to [SE-0195 - Introduce User-defined "Dynamic Member
Lookup" Types](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0195-dynamic-member-lookup.md),
which shipped in Swift 4.2. It introduces a new `@dynamicCallable` attribute, which marks
a type as being "callable" with normal syntax. It is simple syntactic sugar
which allows the user to write:

```swift
a = someValue(keyword1: 42, "foo", keyword2: 19)
````

and have it be rewritten by the compiler as:

```swift
a = someValue.dynamicallyCall(withKeywordArguments: [
    "keyword1": 42, "": "foo", "keyword2": 19
])
```

Many other languages have analogous features (e.g. Python "callables", C++ `operator()`, and
[functors in many other languages](https://en.wikipedia.org/wiki/Function_object)), but the
primary motivation of this proposal is to allow elegant and natural interoperation with
dynamic languages in Swift.

Swift-evolution threads:
 - [Pitch: Introduce user-defined dynamically "callable"
    types](https://forums.swift.org/t/pitch-introduce-user-defined-dynamically-callable-types/7038).
 - [Pitch #2: Introduce user-defined dynamically “callable”
    types](https://forums.swift.org/t/pitch-2-introduce-user-defined-dynamically-callable-types/7112).
 - Current pitch thread: [Pitch #3: Introduce user-defined dynamically “callable”
   types](https://forums.swift.org/t/pitch-3-introduce-user-defined-dynamically-callable-types/12232)

## Motivation and context

Swift is exceptional at interworking with existing C and Objective-C APIs and
we would like to extend this interoperability to dynamic languages like Python,
JavaScript, Perl, and Ruby. We explored this overall goal in a long design
process wherein the Swift evolution community evaluated multiple different
implementation approaches. The conclusion was that the best approach was to put
most of the complexity into dynamic language specific bindings written as
pure-Swift libraries, but add small hooks in Swift to allow these bindings to
provide a natural experience to their clients.
[SE-0195](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0195-dynamic-member-lookup.md)
was the first step in this process, which introduced a binding to naturally
express member lookup rules in dynamic languages.

What does interoperability with Python mean? Let's explain this by looking at
an example. Here's some simple Python code:

```Python
class Dog:
    def __init__(self, name):
        self.name = name
        self.tricks = []  # creates a new empty list for each `Dog`
        
    def add_trick(self, trick):
        self.tricks.append(trick)
```

With the [SE-0195 `@dynamicMemberLookup` feature](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0195-dynamic-member-lookup.md)
introduced in Swift 4.2, it is possible to implement a [Python interoperability
layer](https://github.com/apple/swift/tree/tensorflow/stdlib/public/Python)
written in Swift. It interoperates with the Python runtime, and project all
Python values into a single `PythonObject` type. It allows us to call into the
`Dog` class like this:

```swift
// import DogModule.Dog as Dog
let Dog = Python.import.call(with: "DogModule.Dog")

// dog = Dog("Brianna")
let dog = Dog.call(with: "Brianna")

// dog.add_trick("Roll over")
dog.add_trick.call(with: "Roll over")

// dog2 = Dog("Kaylee").add_trick("snore")
let dog2 = Dog.call(with: "Kaylee").add_trick.call(with: "snore")
```

This also works with arbitrary other APIs as well. Here is an example working
with the Python `pickle` API and the builtin Python function `open`. Note that
we choose to put builtin Python functions like `import` and `open` into a
`Python` namespace to avoid polluting the global namespace, but other designs
are possible:

```swift
// import pickle
let pickle = Python.import.call(with: "pickle")

// file = open(filename)
let file = Python.open.call(with: filename)

// blob = file.read()
let blob = file.read.call()

// result = pickle.loads(blob)
let result = pickle.loads.call(with: blob)
```

This capability works well, but the syntactic burden of having to use
`foo.call(with: bar, baz)` instead of `foo(bar, baz)` is significant. Beyond
the syntactic weight, it directly harms code clarity by making code hard to
read and understand, cutting against a core value of Swift.

The proposed `@dynamicCallable` attribute directly solves this problem.
With it, these examples become more natural and clear, effectively matching the
original Python code in expressiveness:

```swift
// import DogModule.Dog as Dog
let Dog = Python.import("DogModule.Dog")

// dog = Dog("Brianna")
let dog = Dog("Brianna")

// dog.add_trick("Roll over")
dog.add_trick("Roll over")

// dog2 = Dog("Kaylee").add_trick("snore")
let dog2 = Dog("Kaylee").add_trick("snore")
```

Python builtins:

```swift
// import pickle
let pickle = Python.import("pickle")

// file = open(filename)
let file = Python.open(filename)

// blob = file.read()
let blob = file.read()

// result = pickle.loads(blob)
let result = pickle.loads(blob)
```

This proposal merely introduces a syntactic sugar - it does not add any new
semantic model to Swift. We believe that interoperability with scripting
languages is an important and rising need in the Swift community, particularly
as Swift makes inroads into the server development and machine learning
communities. This feature is also precedented in other languages (e.g. Scala's
[`Dynamic`](https://www.scala-lang.org/api/current/scala/Dynamic.html) trait), and
can be used for other purposes besides language interoperability (e.g.
implementing dynamic proxy objects).

## Proposed solution

We propose introducing a new `@dynamicCallable` attribute to the Swift language
which may be applied to structs, classes, enums, and protocols. This follows
the precedent of
[SE-0195](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0195-dynamic-member-lookup.md).

Before this proposal, values of these types are not valid in a call expression:
the only existing callable values in Swift are those with function types
(functions, methods, closures, etc) and metatypes (which are initializer
expressions like `String(42)`). Thus, it is always an error to "call" an
instance of a nominal type (like a struct, for instance).

With this proposal, types with the `@dynamicCallable` attribute on their
primary type declaration become "callable". They are required to implement at
least one of the following two methods for handling the call behavior:

```swift
func dynamicallyCall(withArguments: <#Arguments#>) -> <#R1#>
// `<#Arguments#>` can be any type that conforms to `ExpressibleByArrayLiteral`.
// `<#Arguments#>.ArrayLiteralElement` and the result type `<#R1#>` can be arbitrary.

func dynamicallyCall(withKeywordArguments: <#KeywordArguments#>) -> <#R2#>
// `<#KeywordArguments#>` can be any type that conforms to `ExpressibleByDictionaryLiteral`.
// `<#KeywordArguments#>.Key` must be a type that conforms to `ExpressibleByStringLiteral`.
// `<#KeywordArguments#>.Value` and the result type `<#R2#>` can be arbitrary.

// Note: in these type signatures, bracketed types like <#Arguments#> and <#KeywordArguments#>
// are not actual types, but rather any actual type that meets the specified conditions.
```

As stated above, `<#Arguments#>` and `<#KeywordArguments#>` can be any types
that conform to the
[`ExpressibleByArrayLiteral`](https://developer.apple.com/documentation/swift/expressiblebyarrayliteral)
and
[`ExpressibleByDictionaryLiteral`](https://developer.apple.com/documentation/swift/expressiblebydictionaryliteral)
protocols, respectively. The latter is inclusive of
[`KeyValuePairs`](https://developer.apple.com/documentation/swift/keyvaluepairs),
which supports duplicate keys, unlike [`Dictionary`](https://developer.apple.com/documentation/swift/dictionary).
Thus, using `KeyValuePairs` is recommended to support duplicate keywords and
positional arguments (because positional arguments are desugared as keyword
arguments with the empty string `""` as the key).

If a type implements the `withKeywordArguments:` method, it may be dynamically
called with both positional and keyword arguments: positional arguments have
the empty string `""` as the key. If a type only implements the
`withArguments:` method but is called with keyword arguments, a compile-time
error is emitted.

Since dynamic calls are syntactic sugar for direct calls to `dynamicallyCall`
methods, additional behavior of the `dynamicallyCall` methods is directly
forwarded.  For example, if a `dynamicallyCall` method is marked with `throws`
or `@discardableResult`, then the corresponding sugared dynamic call will
forward that behavior.

### Ambiguity resolution: most specific match

Since there are two `@dynamicCallable` methods, there may be multiple ways to
handle some dynamic calls. What happens if a type specifies both the
`withArguments:` and `withKeywordArguments:` methods?

We propose that the type checker resolve this ambiguity towards the tightest
match based on syntactic form of the expression. The exact rules are:

- If a `@dynamicCallable` type implements the `withArguments:` method and it is
  called with no keyword arguments, use the `withArguments:` method.
- In all other cases, attempt to use the `withKeywordArguments:` method.
  - This includes the case where a `@dynamicCallable` type implements the
    `withKeywordArguments:` method and it is called with at least one keyword
     argument.
  - This also includes the case where a `@dynamicCallable` type implements only
    the `withKeywordArguments:` method (not the `withArguments:` method) and
    it is called with no keyword arguments.
  - If `@dynamicCallable` type does not implement the `withKeywordArguments:`
    method but the call site has keyword arguments, an error is emitted.

Here are some toy illustrative examples:

```swift
@dynamicCallable
struct Callable {
  func dynamicallyCall(withArguments args: [Int]) -> Int { return args.count }
}
let c1 = Callable()
c1() // desugars to `c1.dynamicallyCall(withArguments: [])`
c1(1, 2) // desugars to `c1.dynamicallyCall(withArguments: [1, 2])`
c1(a: 1, 2) // error: `Callable` does not define the 'withKeywordArguments:' method

@dynamicCallable
struct KeywordCallable {
  func dynamicallyCall(withKeywordArguments args: KeyValuePairs<String, Int>) -> Int {
    return args.count
  }
}
let c2 = KeywordCallable()
c2() // desugars to `c2.dynamicallyCall(withKeywordArguments: [:])`
c2(1, 2) // desugars to `c2.dynamicallyCall(withKeywordArguments: ["": 1, "": 2])`
c2(a: 1, 2) // desugars to `c2.dynamicallyCall(withKeywordArguments: ["a": 1, "": 2])`

@dynamicCallable
struct BothCallable {
  func dynamicallyCall(withArguments args: [Int]) -> Int { return args.count }
  func dynamicallyCall(withKeywordArguments args: KeyValuePairs<String, Int>) -> Int {
    return args.count
  }
}
let c3 = BothCallable()
c3() // desugars to `c3.dynamicallyCall(withArguments: [])`
c3(1, 2) // desugars to `c3.dynamicallyCall(withArguments: [1, 2])`
c3(a: 1, 2) // desugars to `c3.dynamicallyCall(withKeywordArguments: ["a": 1, "": 2])`
```

This ambiguity resolution rule works out naturally given the behavior of the
Swift type checker, because it only resolves call expressions when the type
of the base expression is known. At that point, it knows whether the base is a
function type, metatype, or a valid `@dynamicCallable` type, and it knows the
syntactic form of the call.

This proposal does not require massive or invasive changes to the constraint
solver. Please look at the implementation for more details.

## Example usage

Here, we sketch some example bindings to show how this could be used in
practice. Note that there are lots of design decisions that are orthogonal to
this proposal (e.g. how to handle exceptions) that we aren't going into here.
This is just to show how this feature provides an underlying facility that
language bindings authors can use to achieve their desired result. These
examples also show `@dynamicMemberLookup` to illustrate how they work together,
but elides other implementation details.

JavaScript supports callable objects but does not have keyword arguments.

Here is a sample JavaScript binding:

```swift
@dynamicCallable @dynamicMemberLookup
struct JSValue {
  // JavaScript doesn't have keyword arguments.
  @discardableResult
  func dynamicallyCall(withArguments: [JSValue]) -> JSValue { ... }

  // This is a `@dynamicMemberLookup` requirement.
  subscript(dynamicMember member: JSValue) -> JSValue {...}
  
  // ... other stuff ...
}
```

On the other hand, a common JavaScript pattern is to take a dictionary of
values as a stand-in for argument labels (called like
`example({first: 1, second: 2, third: 3})` in JavaScript). A JavaScript bridge
in Swift could choose to implement keyword argument support to allow this to be
called as `example(first: 1, second: 2, third: 3)` from Swift code (kudos to
Ben Rimmington for [this
observation](https://forums.swift.org/t/pitch-3-introduce-user-defined-dynamically-callable-types/12232/45)).

Python does support keyword arguments. While a Python binding could implement
only the `withKeywordArguments:` method, it is be better to implement both the
non-keyword and keyword forms to make the non-keyword case slightly more
efficient (avoid allocating temporary storage) and to make direct calls with
positional arguments nicer (`x.dynamicallyCall(withArguments: 1, 2)` instead of
`x.dynamicallyCall(withKeywordArguments: ["": 1, "": 2])`).

Here is a sample Python binding:

```swift
@dynamicCallable @dynamicMemberLookup
struct PythonObject {
  // Python supports arbitrary mixes of keyword arguments and non-keyword
  // arguments.
  @discardableResult
  func dynamicallyCall(
    withKeywordArguments: KeyValuePairs<String, PythonObject>
  ) -> PythonObject { ... }

  // An implementation of a Python binding could choose to implement this
  // method as well, avoiding allocation of a temporary array.
  @discardableResult
  func dynamicallyCall(withArguments: [PythonObject]) -> PythonObject { ... }

  // This is a `@dynamicMemberLookup` requirement.
  subscript(dynamicMember member: String) -> PythonObject {...}
  
  // ... other stuff ...
}
```

## Limitations

Following the precedent of SE-0195, this attribute must be placed on the
primary definition of a type, not on an extension.

This proposal does not introduce the ability to provide dynamically callable
`static`/`class` members. We don't believe this is important given the goal of
supporting dynamic languages like Python, but it could be explored if a use
case is discovered in the future. Such future work should keep in mind that
call syntax on metatypes is already meaningful, and that ambiguity would have
to be resolved somehow (e.g. through the most specific rule).

This proposal supports direct calls of values and methods, but subsets out
support for currying methods in Smalltalk family languages. This is just an
implementation limitation given the current state of currying in the Swift
compiler. Support can be added in the future if there is a specific need.

## Source compatibility

This is a strictly additive proposal with no source breaking changes.

## Effect on ABI stability

This is a strictly additive proposal with no ABI breaking changes.

## Effect on API resilience

This has no impact on API resilience which is not already captured by other
language features.

## Future directions

### Dynamic member calling (for Smalltalk family languages)

In addition to supporting languages like Python and JavaScript, we would also
like to grow to support Smalltalk derived languages like Ruby and Squeak. These
languages resolve *method* calls using both the base name as well as the
keyword arguments at the same time. For example, consider this Ruby code:

```Ruby
time = Time.zone.parse(user_time)
```

The `Time.zone` reference is a member lookup, but `zone.parse(user_time)` is a
method call, and needs to be handled differently than a lookup of `zone.parse`
followed by a direct function call.

This can be handled by adding a new `@dynamicMemberCallable` attribute, which
acts similarly to `@dynamicCallable` but enables dynamic member calls (instead
of dynamic calls of `self`).

`@dynamicMemberCallable` would have the following requirements:

```swift
func dynamicallyCallMethod(named: S1, withArguments: [T5]) -> T6
func dynamicallyCallMethod(named: S2, withKeywordArguments: [S3 : T7]) -> T8
```

Here is a sample Ruby binding:

```swift
@dynamicMemberCallable @dynamicMemberLookup
struct RubyObject {
  @discardableResult
  func dynamicallyCallMethod(
    named: String, withKeywordArguments: KeyValuePairs<String, RubyObject>
  ) -> RubyObject { ... }

  // This is a `@dynamicMemberLookup` requirement.
  subscript(dynamicMember member: String) -> RubyObject {...}
  
  // ... other stuff ...
}
```

### General callable behavior

This proposal is mainly directed at dynamic language interoperability. For this
use case, it makes sense for the `dynamicallyCall` method to take a
variable-sized list of arguments where each argument has the same type.
However, it may be useful to support general callable behavior (akin to
`operator()` in C++) where the desugared "callable" method can have a fixed
number of arguments and arguments of different types.

For example, consider something like:

```swift
struct BinaryFunction<T1, T2, U> {
  func call(_ argument1: T1, _ argument1: T2) -> U { ... }
}
```

It is not unreasonable to look ahead to a day where sugaring such things is
supported, particularly when/if Swift gets [variadic
generics](https://github.com/apple/swift/blob/master/docs/GenericsManifesto.md#variadic-generics).
This could allow typesafe n-ary smart function pointer types.

We feel that the approach outlined in this proposal supports this direction.
When/if a motivating use case for general callable behavior comes up, we can
simply add a new form to represent it and enhance the type checker to prefer
that during ambiguity resolution. If this is a likely direction, then it may be
better to name the attribute `@callable` instead of `@dynamicCallable` in
anticipation of that future growth.

We believe that general callable behavior and `@dynamicCallable` are orthogonal
features and should be evaluated separately.

## Alternatives considered

Many alternatives were considered and discussed. Most of them are captured in
the ["Alternatives Considered" section of
SE-0195](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0195-dynamic-member-lookup.md#alternatives-considered).

Here are a few points raised in the discussion:

- It was suggested that we use subscripts to represent the call
  implementations instead of a function call, aligning with
  `@dynamicMemberLookup`. We think that functions are a better fit here: the
  reason `@dynamicMemberLookup` uses subscripts is to allow the members to be
  l-values, but call results are not l-values.

- It was requested that we design and implement the 'static callable' version
  of this proposal in conjunction with the dynamic version proposed here. In
  the author's opinion, it is important to consider static callable support as
  a likely future direction to make sure that the two features sit well next
  to each other and have a consistent design (something we believe this
  proposal has done) but it doesn't make sense to join the two proposals. So
  far, there have been no strong motivating use case presented for the static
  callable version, and Swift lacks certain generics features (e.g. variadics)
  that would be necessary to make static callables general. We feel that static
  callable should stand alone on its own merits.
