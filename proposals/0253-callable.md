# Callable values of user-defined nominal types

* Proposal: [SE-0253](0253-callable.md)
* Authors: [Richard Wei](https://github.com/rxwei), [Dan Zheng](https://github.com/dan-zheng)
* Review Manager: [Chris Lattner](https://github.com/lattner)
* Status: **Implemented (Swift 5.2)**
* Implementation: [apple/swift#24299](https://github.com/apple/swift/pull/24299)
* Previous Revisions:
  [[1]](https://github.com/swiftlang/swift-evolution/blob/36ea8be09508db9380949954d0c7a101fdb15226/proposals/0253-callable.md),
  [[2]](https://github.com/swiftlang/swift-evolution/blob/e1dd65469e3525e5e230b877e1539bff1e1cc5e3/proposals/0253-callable.md)
* Decision Notes:
  ([rationale](https://forums.swift.org/t/accepted-with-modification-se-0253-callable-values-of-user-defined-nominal-types/24605)),
  ([final revision note](https://forums.swift.org/t/accepted-with-modification-se-0253-callable-values-of-user-defined-nominal-types/24605/166))

## Introduction

This proposal introduces "statically"
[callable](https://en.wikipedia.org/wiki/Callable_object) values to Swift.
Callable values are values that define function-like behavior and can be called
using function call syntax. In contrast to dynamically callable values
introduced in
[SE-0216](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0216-dynamic-callable.md),
this feature supports statically declared arities, argument labels, and
parameter types, and is not constrained to primary type declarations.

In a nutshell, values that have a method whose base name is `callAsFunction`
(referred to as a "`callAsFunction` method" for the rest of this proposal) can
be called like a function. The function call syntax forwards arguments to the
corresponding `callAsFunction` method.

```swift
struct Adder {
    var base: Int
    func callAsFunction(_ x: Int) -> Int {
        return base + x
    }
}

let add3 = Adder(base: 3)
add3(10) // => 13
```

## Motivation

Currently, in Swift, only a few kinds of values are syntactically callable:
- Values with function types.
- Type names (e.g. `T` can be called like `T(...)`, which is desugared to `T.init(...)`).
- Values with a `@dynamicCallable` type.

However, call-syntax can also be useful for other values, primarily those that behave like functions. This includes:
- Values that represent functions: mathematical functions, function expressions, etc.
- Values that have one main use and want to provide a simple call-syntax interface: neural network layers, parsers, efficient bound closures, etc.

Here are some concrete sources of motivation.

### Values representing functions

Values of some nominal types exactly represent functions: in the mathematical sense (a mapping from inputs to outputs), or in the context of programming languages.

Here are some examples:

```swift
/// Represents a polynomial function, e.g. `2 + 3x + 4x²`.
struct Polynomial {
    /// Represents the coefficients of the polynomial, starting from power zero.
    let coefficients: [Float]
}
```

Since these types represent functions, naturally they can be applied to inputs. However, currently in Swift, the "function application" functionality must be defined as a method.

```swift
extension Polynomial {
    func evaluated(at input: Float) -> Float {
        var result: Float = 0
        for (i, c) in coefficients.enumerated() {
            result += c * pow(input, Float(i))
        }
        return result
    }
}

let f = Polynomial(coefficients: [2, 3, 4])
print(f.evaluated(at: 2)) // => 24
```

The mathematical notation for function application is simply `output = f(input)`. Using subscript methods achieve a similar application syntax `f[x]`, but subscripts and square brackets typically connote "indexing into a collection", which is not the behavior here.

```swift
extension Polynomial {
    subscript(input: Float) -> Float {
        ...
    }
}
let f = Polynomial(coefficients: [2, 3, 4])
// Subscript syntax, may be confusing.
print(f[2]) // => 24
```

The proposed feature enables the same call syntax as the mathematical notation:
```swift
extension Polynomial {
    func callAsFunction(_ input: Float) -> Float {
        ...
    }
}
let f = Polynomial(coefficients: [2, 3, 4])
// Call syntax.
print(f(2)) // => 24
```

#### Bound closures

Variable-capturing closures can be modeled explicitly as structs that store the bound variables. This representation is more performant and avoids the type-erasure of closure contexts.

```swift
// Represents a nullary function capturing a value of type `T`.
struct BoundClosure<T> {
    var function: (T) -> Void
    var value: T

    func callAsFunction() { return function(value) }
}

let x = "Hello world!"
let closure = BoundClosure(function: { print($0) }, value: x)
closure() // prints "Hello world!"
```

A call syntax sugar would enable `BoundClosure` instances to be applied like normal functions.

### Nominal types with one primary method

Some nominal types have a "primary method" that performs their main use. For example:

* Calculators *calculate*: `calculator.calculating(query)`.
* Parsers *parse*: `parser.parsing(text)`.
* Neural network layers *apply to inputs*: `layer.applied(to: input)`.
* Types representing functions *apply to arguments*: `function.applied(to: arguments)`.

Types that have a primary method usually call that method frequently. Thus, it may be desirable to sugar applications of the main method with call syntax to reduce noise.

Let's explore neural network layers and string parsers in detail.

#### Neural network layers

[Machine learning](https://en.wikipedia.org/wiki/Machine_learning) models often represent a function that contains an internal state called "trainable parameters", and the function takes an input and predicts the output. In code, models are often represented as a data structure that stores trainable parameters, and a method that defines the transformation from an input to an output in terms of these trained parameters. Here's an example:

```swift
struct Perceptron {
    var weight: Vector<Float>
    var bias: Float

    func applied(to input: Vector<Float>) -> Float {
        return weight • input + bias
    }
}
```

Stored properties `weight` and `bias` are considered as trainable parameters, and are used to define the transformation from model inputs to model outputs. Models can be [trained ](https://developers.google.com/machine-learning/glossary/#training), during which parameters like `weight` are updated, thus changing the behavior of `applied(to:)`. When a model is used, the call site looks just like a function call.

```swift
let model: Perceptron = ...
let ŷ = model.applied(to: x)
```

Many deep learning models are composed of layers, or layers of layers. In the definition of those models, repeated calls to `applied(to:)` significantly complicate the look of the program and reduce the clarity of the resulting code.

```swift
struct Model {
    var conv = Conv2D<Float>(filterShape: (5, 5, 3, 6))
    var maxPool = MaxPool2D<Float>(poolSize: (2, 2), strides: (2, 2))
    var flatten = Flatten<Float>()
    var dense = Dense<Float>(inputSize: 36 * 6, outputSize: 10)

    func applied(to input: Tensor<Float>) -> Tensor<Float> {
        return dense.applied(to: flatten.applied(to: maxPool.applied(to: conv.applied(to: input))))
    }
}
```

These repeated calls to `applied(to:)` harm clarity and makes code less readable. If `model` could be called like a function, which it mathematically represents, the definition of `Model` becomes much shorter and more concise. The proposed feature [promotes clear usage by omitting needless words](https://swift.org/documentation/api-design-guidelines/#promote-clear-usage).

```swift
struct Model {
    var conv = Conv2D<Float>(filterShape: (5, 5, 3, 6))
    var maxPool = MaxPool2D<Float>(poolSize: (2, 2), strides: (2, 2))
    var flatten = Flatten<Float>()
    var dense = Dense<Float>(inputSize: 36 * 6, outputSize: 10)

    func callAsFunction(_ input: Tensor<Float>) -> Tensor<Float> {
        // Call syntax.
        return dense(flatten(maxPool(conv(input))))
    }
}

let model: Model = ...
let ŷ = model(x)
```

There are more ways to further simplify model definitions, but making models callable like functions is a good first step.

#### Domain specific languages

DSL constructs like string parsers represent functions from inputs to outputs. Parser combinators are often implemented as higher-order functions operating on parser values, which are themselves data structures—some implementations store closures, while some other [efficient implementations ](https://infoscience.epfl.ch/record/203076/files/p637-jonnalagedda.pdf) store an expression tree. They all have an "apply"-like method that performs an application of the parser (i.e. parsing).

```swift
struct Parser<Output> {
    // Stored state...

    func applied(to input: String) throws -> Output {
        // Using the stored state...
    }

    func many() -> Parser<[Output]> { ... }
    func many<T>(separatedBy separator: Parser<T>) -> Parser<[Output]> { ... }
}
```

When using a parser, one would need to explicitly call `applied(to:)`, but that is a bit cumbersome. Since parsers are like functions, it would be cleaner if parsers themself were callable.

```swift
func callAsFunction(_ input: String) throws -> Output {
    // Using the stored state...
}
```

```swift
let sexpParser: Parser<Expression> = ...
// Call syntax.
let sexp = sexpParser("(+ 1 2)")
```

### A static counterpart to `@dynamicCallable`

[SE-0216](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0216-dynamic-callable.md) introduced user-defined dynamically callable values. In its [alternatives considered](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0216-dynamic-callable.md#alternatives-considered) section, it was requested that we design and implement the "static callable" version of this proposal in conjunction with the dynamic version proposed. See its [pitch thread](https://forums.swift.org/t/pitch-3-introduce-user-defined-dynamically-callable-types/12232) for discussions about "static callables".

### Prior art

Many languages offer the call syntax sugar:

* Python: [ `object.__call__(self[, args...])` ](https://docs.python.org/3/reference/datamodel.html#emulating-callable-objects)
* C++: [ `operator()` (function call operator)](https://en.cppreference.com/w/cpp/language/operators)
* Scala: [ `def apply(...)` (apply methods) ](https://twitter.github.io/scala_school/basics2.html#apply)

### Unifying compound types and nominal types

A long term goal with the type system is to unify compound types (e.g. function types and tuple types) and nominal types, to allow compound types to conform to protocols and have members. When function types can have members, it will be most natural for them to have a `call` method, which can help unify the compiler's type checking rules for call expressions.

## Proposed design

We propose to introduce a syntactic sugar for values that have an instance
method whose base name is `call` (a `call` method).

```swift
struct Adder {
    var base: Int
    func callAsFunction(_ x: Int) -> Int {
        return base + x
    }
}
```

Values that have a `callAsFunction` method can be called like a function, forwarding
arguments to the `callAsFunction` method.

```swift
let add3 = Adder(base: 3)
add3(10) // => 13
```

Note: There are many alternative syntaxes for marking "call-syntax delegate
methods". These are listed and explored in the ["Alternatives
considered"](#alternative-ways-to-declare-call-syntax-delegate-methods) section.

## Detailed design

### `callAsFunction` methods

Instance methods whose base name is `callAsFunction` will be recognized as an
implementation that makes a value of the enclosing type "callable" like a
function.

When type-checking a call expression, the type checker will try to resolve the
callee. Currently, the callee can be a value with a function type, a type name,
or a value of a `@dynamicCallable` type. This proposal adds a fourth kind of a
callee: a value with a matching `callAsFunction` method.

```swift
struct Adder {
    var base: Int

    func callAsFunction(_ x: Int) -> Int {
        return base + x
    }

    func callAsFunction(_ x: Float) -> Float {
        return Float(base) + x
    }

    func callAsFunction<T>(_ x: T, bang: Bool) throws -> T where T: BinaryInteger {
        if bang {
            return T(Int(exactly: x)! + base)
        } else {
            return T(Int(truncatingIfNeeded: x) + base)
        }
    }
}
```

```swift
let add1 = Adder(base: 1)
add1(2) // => 3
try add1(4, bang: true) // => 5
```

When type-checking fails, error messages look like those for function calls.
When there is ambiguity, the compiler will show relevant `callAsFunction` method
candidates.

```swift
add1("foo")
// error: cannot invoke 'add1' with an argument list of type '(String)'
// note: overloads for functions named 'callAsFunction' exist with these partially matching parameter lists: (Float), (Int)
add1(1, 2, 3)
// error: cannot invoke 'add1' with an argument list of type '(Int, Int, Int)'
```

### Direct reference to `callAsFunction`

Since a `callAsFunction` method is a normal method, one can refer to a
`callAsFunction` method using its declaration name and get a closure where
`self` is captured. This is exactly how method references work today.

```swift
let add1 = Adder(base: 1)
let f1: (Int) -> Int = add1.callAsFunction
let f2: (Float) -> Float = add1.callAsFunction(_:)
let f3: (Int, Bool) throws -> Int = add1.callAsFunction(_:bang:)
```

### When the type is also `@dynamicCallable`

A type can both have `callAsFunction` methods and be declared with
`@dynamicCallable`. When type-checking a call expression, the type checker will
first try to resolve the call to a function or initializer call, then a
`callAsFunction` method call, and finally a dynamic call.

### Implementation

The implementation is very simple and non-invasive: [less than 200 lines of
code](https://github.com/apple/swift/pull/24299) in the type checker that
performs lookup and expression rewrite.

```swift
let add1 = Adder(base: 1)
add1(0) // Rewritten to `add1.callAsFunction(0)` after type checking.
```

## Source compatibility

This is a strictly additive proposal with no source-breaking changes.

## Effect on ABI stability

This is a strictly additive proposal with no ABI-breaking changes.

## Effect on API resilience

This has no impact on API resilience which is not already captured by other language features.

## Future directions

### Implicit conversions to function

A value cannot be implicitly converted to a function when the destination
function type matches the type of the `callAsFunction` method. Since
`callAsFunction` methods are normal methods, you can [refer to them
directly](#direct-reference-to-call) via `.callAsFunction` and get a function.

Implicit conversions impact the entire type system and require runtime support
to work with dynamic casts; thus, further exploration is necessary for a formal
proposal. This base proposal is self-contained; incremental proposals involving
conversion can come later.

```swift
let h: (Int) -> Int = add1
```

A less controversial future direction is to support explicit conversion via `as`:

```swift
let h = add1 as (Int) -> Int
```

### Function type as a constraint

On the [pitch
thread](https://forums.swift.org/t/pitch-introduce-static-callables/21732/2),
[Joe Groff](https://github.com/jckarter) brought up the possibility of allowing
function types to be used as conformance constraints. Performance-minded
programmers can define custom closure types where the closure context is not
fully type-erased.

```swift
struct BoundClosure<T, F: (T) -> ()>: () -> () {
  var function: F
  var value: T

  func callAsFunction() { return function(value) }
}

let f = BoundClosure({ print($0) }, x) // instantiates BoundClosure<(underlying type of closure), Int>
f() // invokes call on BoundClosure
```

In this design, the function type constraint behaves like a protocol that
requires a `callAsFunction` method whose parameter types are the same as the
function type's parameter types.

## Alternatives considered

### Alternative names for call-syntax delegate methods

In addition to the word `call` in `callAsFunction`, there are other words that
can be used to denote the function call syntax. The most common ones are `apply`
and `invoke` as they are used to declare call-syntax delegate methods in other
programming languages.

Both `apply` and `invoke` are good one-syllable English words that are
technically correct, but we feel there are two concerns with these names:

* They are officially completely new terminology to Swift. In [the _Functions_
chapter of _The Swift Programming Language_
book](https://docs.swift.org/swift-book/LanguageGuide/Functions.html), there is
no mention of "apply" or "invoke" anywhere. Function calls are officially
called "function calls".

* They do not work very well with Swift's API naming conventions. According to
  [Swift API Design Guidelines - Strive for Fluent
  Usage](https://swift.org/documentation/api-design-guidelines/#strive-for-fluent-usage),
  functions should be named according to their side-effects.
  
  > Those with side-effects should read as imperative verb phrases, e.g.,
  > `print(x)`, `x.sort()`, `x.append(y)`.
  
  Both `apply` and `invoke` are clearly imperative verbs. If call-syntax
  delegate methods must be named `apply` or `invoke`, their declarations and
  direct references will almost certainly read like a mutating function while
  they may not be.
  
  In contrast, `call` is both a noun and a verb. It is perfectly
  suited for describing the precise functionality while not having a strong
  implication about the function's side-effects.
  
  **call**
  - _v._ Cause (a subroutine) to be executed.
  - _n._ A command to execute a subroutine.
  
After the second round of proposal review, the core team accepted the proposal
while revising the function base name to `callFunction`. The revision invoked a
significant amount of bikeshedding on [the
thread](https://forums.swift.org/t/accepted-with-modification-se-0253-callable-values-of-user-defined-nominal-types/24605).
After considering the feedback, the core team
[decided](https://forums.swift.org/t/accepted-with-modification-se-0253-callable-values-of-user-defined-nominal-types/24605/166)
to further revise the proposal to name the call-syntax delegate method
`callAsFunction`.

### Alternative ways to declare call-syntax delegate methods

#### Create a new declaration kind like `subscript` and `init`

Declarations that are associated with special invocation syntax often have their
own declaration kind. For example, subscripts are implemented with a `subscript`
declaration, and initialization calls are implemented with an `init`
declaration. Since the function call syntax is first-class, one direction is to
make the declaration be as first-class as possible.

```swift
struct Adder {
    var base: Int
    call(_ x: Int) -> Int {
        return base + x
    }
}
```

This alternative is in fact what's proposed in [the first revision of this
proposal](https://github.com/swiftlang/swift-evolution/blob/36ea8be09508db9380949954d0c7a101fdb15226/proposals/0253-callable.md),
which got [returned for
revision](https://forums.swift.org/t/returned-for-revision-se-0253-static-callables/23290).

#### Use unnamed `func` declarations to mark call-syntax delegate methods

```swift
struct Adder {
    var base: Int
    // Option: `func` with literally no name.
    func(_ x: Int) -> Int { ... }

    // Option: `func` with an underscore at the base name position.
    func _(_ x: Int) -> Int
    
    // Option: `func` with a `self` keyword at the base name position.
    func self(_ x: Int) -> Int

    // Option: `call` method modifier on unnamed `func` declarations.
    // Makes unnamed `func` less weird and clearly states "call".
    call func(_ x: Int) -> Int { ... }
}
```

This approach represents call-syntax delegate methods as unnamed `func`
declarations instead of `func callAsFunction`.

One option is to use `func(...)` without an identifier name. Since the word
"call" does not appear, it is less clear that this denotes a call-syntax
delegate method. Additionally, it's not clear how direct references would work:
the proposed design of referencing `callAsFunction` methods via `foo.call` is
clear and consistent with the behavior of `init` declarations.

To make unnamed `func(...)` less weird, one option is to add a `call`
declaration modifier: `call func(...)`. The word `call` appears in both this
option and the proposed design, clearly conveying "call-syntax delegate method".
However, declaration modifiers are currently also treated as keywords, so with
both approaches, parser changes to ensure source compatibility are necessary.
`call func(...)` requires additional parser changes to allow `func` to sometimes
not be followed by a name. The authors lean towards `callAsFunction` methods for
simplicity and uniformity.

#### Use an attribute to mark call-syntax delegate methods

```swift
struct Adder {
    var base: Int
    @callDelegate
    func addingWithBase(_ x: Int) -> Int {
        return base + x
    }
}
```

This approach achieves a similar effect as `callAsFunction` methods, except that
it allows call-syntax delegate methods to have a custom name and be directly
referenced by that name. This is useful for types that want to make use of the
call syntax sugar, but for which the name "call" does not accurately describe
the callable functionality.

However, there are two concerns.

* First, we feel that using a `@callableMethod` method attribute is more noisy,
  as many callable values do not need a special name for its call-syntax
  delegate methods.
  
* Second, custom names often involve argument labels that form a phrase with the
  base name in order to be idiomatic. The grammaticality will be lost in the
  call syntax when the base name disappears.
  
  ```swift
  struct Layer {
      ...
      @callDelegate
      func applied(to x: Int) -> Int { ... }
  }
  
  let layer: Layer = ...
  layer.applied(to: x) // Grammatical.
  layer(to: x)         // Broken.
  ```

In contrast, standardizing on a specific name defines these problems away and
makes this feature easier to use.

For reference: Other languages with callable functionality typically require
call-syntax delegate methods to have a particular name (e.g. `def __call__` in
Python, `def apply` in Scala).

#### Use a type attribute to mark types with call-syntax delegate methods

```swift
@staticCallable // alternative name `@callable`; similar to `@dynamicCallable`
struct Adder {
    var base: Int
    // Informal rule: all methods with a particular name (e.g. `func callAsFunction`) are deemed call-syntax delegate methods.
    //
    // `StringInterpolationProtocol` has a similar informal requirement for
    // `func appendInterpolation` methods.
    // https://github.com/swiftlang/swift-evolution/blob/master/proposals/0228-fix-expressiblebystringinterpolation.md#proposed-solution
    func callAsFunction(_ x: Int) -> Int {
        return base + x
    }
}
```

We feel this approach is not ideal because a marker type attribute is not
particularly meaningful. The call-syntax delegate methods of a type are what
make values of that type callable - a type attribute means nothing by itself.
There's also an unforunate edge case that must be explicitly handled: if a
`@staticCallable` type defines no call-syntax delegate methods, an error must be
produced.

After the first round of review, the core team also did not think a type-level
attribute is necessary.

> After discussion, the core team doesn't think that a type level attribute is
> necessary, and there is no reason to limit this to primal type declarations -
> it is fine to add callable members (or overloads) in extensions, just as you
> can add subscripts to a type in extensions today.

#### Use a `Callable` protocol to represent callable types

```swift
// Compiler-known `Callable` marker protocol.
struct Adder: Callable {
    var base: Int
    // Informal rule: all methods with a particular name (e.g. `func callAsFunction`) are deemed call-syntax delegate methods.
    func callAsFunction(_ x: Int) -> Int {
        return base + x
    }
}
```

We feel this approach is not ideal for the same reasons as the marker type attribute. A marker protocol by itself is not meaningful and the name for call-syntax delegate methods is informal. Additionally, protocols should represent particular semantics, but call-*syntax* behavior has no inherent semantics.

### Also allow `static`/`class` `callAsFunction` methods

Static `callAsFunction` methods could in theory look like initializers at the
call site.

```swift
extension Adder {
    static func callAsFunction(base: Int) -> Int {
        ...
    }
    static func callAsFunction(_ x: Int) -> Int {
        ...
    }
}
Adder(base: 3) // error: ambiguous static member; do you mean `init(base:)` or `call(base:)`?
Adder(3) // okay, returns an `Int`, but it looks really like an initializer that returns an `Adder`.
```

This is an interesting direction, but parentheses followed by a type identifier
often connote initialization and it is not source-compatible. We believe this
would make call sites look very confusing.

### Unify callable functionality with `@dynamicCallable`

Both `@dynamicCallable` and the proposed `callAsFunction` methods involve
syntactic sugar related to function applications. However, the rules of the
sugar are different, making unification difficult. In particular,
`@dynamicCallable` provides a special sugar for argument labels that is crucial
for usability.

```swift
// Let `PythonObject` be a `@dynamicMemberLookup` type with callable functionality.
let np: PythonObject = ...
// `PythonObject` with `@dynamicCallable.
np.random.randint(-10, 10, dtype: np.float)
// `PythonObject` with `callAsFunction` methods. The empty strings are killer.
np.random.randint(["": -10, "": 10, "dtype": np.float])
```

[static-and-class-subscripts]: https://forums.swift.org/t/pitch-static-and-class-subscripts/21850 
