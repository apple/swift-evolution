# Value and Type Parameter Packs

* Proposal: [SE-0393](0393-parameter-packs.md)
* Authors: [Holly Borla](https://github.com/hborla), [John McCall](https://github.com/rjmccall), [Slava Pestov](https://github.com/slavapestov)
* Review Manager: [Xiaodi Wu](https://github.com/xwu)
* Status: **Implemented (Swift 5.9)**
* Review: ([pitch 1](https://forums.swift.org/t/pitch-parameter-packs/60543)) ([pitch 2](https://forums.swift.org/t/pitch-2-value-and-type-parameter-packs/60830)) ([review](https://forums.swift.org/t/se-0393-value-and-type-parameter-packs/63859)) ([acceptance](https://forums.swift.org/t/accepted-with-modifications-se-0393-value-and-type-parameter-packs/64382))

## Introduction

Many modern Swift libraries include ad-hoc variadic APIs with an arbitrary upper bound, typically achieved with overloads that each have a different fixed number of type parameters and corresponding arguments. Without variadic generic programming support in the language, these ad-hoc variadic APIs have a significant cost on library maintenance and the developer experience of using these APIs.

This proposal adds _type parameter packs_ and _value parameter packs_ to enable abstracting over the number of types and values with distinct type. This is the first step toward variadic generics in Swift.

## Contents

- [Value and Type Parameter Packs](#value-and-type-parameter-packs)
  - [Introduction](#introduction)
  - [Contents](#contents)
  - [Motivation](#motivation)
  - [Proposed solution](#proposed-solution)
  - [Detailed design](#detailed-design)
    - [Type parameter packs](#type-parameter-packs)
    - [Pack expansion type](#pack-expansion-type)
    - [Type substitution](#type-substitution)
      - [Single-element pack substitution](#single-element-pack-substitution)
    - [Type matching](#type-matching)
      - [Label matching](#label-matching)
      - [Trailing closure matching](#trailing-closure-matching)
      - [Type list matching](#type-list-matching)
    - [Member type parameter packs](#member-type-parameter-packs)
    - [Generic requirements](#generic-requirements)
      - [Same-shape requirements](#same-shape-requirements)
      - [Restrictions on same-shape requirements](#restrictions-on-same-shape-requirements)
    - [Value parameter packs](#value-parameter-packs)
    - [Overload resolution](#overload-resolution)
  - [Effect on ABI stability](#effect-on-abi-stability)
  - [Alternatives considered](#alternatives-considered)
    - [Modeling packs as tuples with abstract elements](#modeling-packs-as-tuples-with-abstract-elements)
    - [Syntax alternatives to `repeat each`](#syntax-alternatives-to-repeat-each)
      - [The `...` operator](#the--operator)
      - [Another operator](#another-operator)
      - [Magic builtin `map` method](#magic-builtin-map-method)
  - [Future directions](#future-directions)
    - [Variadic generic types](#variadic-generic-types)
    - [Local value packs](#local-value-packs)
    - [Explicit type pack syntax](#explicit-type-pack-syntax)
    - [Pack iteration](#pack-iteration)
    - [Pack element projection](#pack-element-projection)
      - [Dynamic pack indexing with `Int`](#dynamic-pack-indexing-with-int)
      - [Typed pack element projection using key-paths](#typed-pack-element-projection-using-key-paths)
    - [Value expansion operator](#value-expansion-operator)
    - [Pack destructuring operations](#pack-destructuring-operations)
    - [Tuple conformances](#tuple-conformances)
  - [Revision history](#revision-history)
  - [Acknowledgments](#acknowledgments)

## Motivation

Generic functions currently require a fixed number of type parameters. It is not possible to write a generic function that accepts an arbitrary number of arguments with distinct types, instead requiring one of the following workarounds:

* Erasing all of the types involved, e.g. using `Any...`
* Using a single tuple type argument instead of separate type arguments
* Overloading for each argument length with an artificial limit

One example in the Swift Standard Library is the 6 overloads for each tuple comparison operator:

```swift
func < (lhs: (), rhs: ()) -> Bool

func < <A, B>(lhs: (A, B), rhs: (A, B)) -> Bool where A: Comparable, B: Comparable

func < <A, B, C>(lhs: (A, B, C), rhs: (A, B, C)) -> Bool where A: Comparable, B: Comparable, C: Comparable

// and so on, up to 6-element tuples
```

With language support for a variable number of type parameters, this API could be expressed more naturally and concisely as a single function declaration:

```swift
func < <each Element: Comparable>(lhs: (repeat each Element), rhs: (repeat each Element)) -> Bool
```

## Proposed solution

This proposal adds support for generic functions which abstract over a variable number of type parameters. While this proposal is useful on its own, there are many future directions that build upon this concept. This is the first step toward equipping Swift programmers with a set of tools that enable variadic generic programming.

Parameter packs are the core concept that facilitates abstracting over a variable number of parameters. A pack is a new kind of type-level and value-level entity that represents a list of types or values, and it has an abstract length. A type parameter pack stores a list of zero or more type parameters, and a value parameter pack stores a list of zero or more value parameters. A type parameter pack is declared in angle brackets using the `each` contextual keyword:

```swift
// 'S' is a type parameter pack where each pack element conforms to 'Sequence'.
func zip<each S: Sequence>(...)
```

A parameter pack itself is not a first-class value or type, but the elements of a parameter pack can be used anywhere that naturally accepts a list of values or types using _pack expansions_, including top-level expressions.

A pack expansion consists of the `repeat` keyword followed by a type or an expression. The type or expression that `repeat` is applied to is called the _repetition pattern_. The repetition pattern must contain at least one pack reference, spelled with the `each` keyword. At runtime, the pattern is repeated for each element in the substituted pack, and the resulting types or values are _expanded_ into the list provided by the surrounding context.

Similarly, pack references can only appear inside repetition patterns and generic requirements:

```swift
func zip<each S>(_ sequence: repeat each S) where repeat each S: Sequence
```

Given a concrete pack substitution, the pattern is repeated for each element in the substituted pack. If `S` is substituted with `Array<Int>, Set<String>`, then `repeat Optional<each S>` will repeat the pattern `Optional<each S>` for each element in the substitution to produce `Optional<Array<Int>>, Optional<Set<String>>`.

Here are the key concepts introduced by this proposal:

- Under the new model, all existing types and values in the language become _scalar types_ and _scalar values_.
- A _type pack_ is a new kind of type-level entity which represents a list of scalar types. Type packs do not have syntax in the surface language, but we will write them as `{T1, ..., Tn}` where each `Ti` is a scalar type. Type packs cannot be nested; type substitution is defined to always flatten type packs.
- A _type parameter pack_ is a list of zero or more scalar type parameters. These are declared in a generic parameter list using the syntax `each T`, and referenced with `each T`.
- A _value pack_ is a list of scalar values. The type of a value pack is a type pack, where each element of the type pack is the scalar type of the corresponding scalar value. Value packs do not have syntax in the surface language, but we will write them as `{x1, ..., xn}` where each `xi` is a scalar value. Value packs cannot be nested; evaluation is always defined to flatten value packs.
- A _value parameter pack_ is a list of zero or more scalar function or macro parameters.
- A _pack expansion_ is a new kind of type-level and value-level construct that expands a type or value pack into a list of types or values, respectively. Written as `repeat P`, where `P` is the _repetition pattern_ that captures at least one type parameter pack (spelled with the `each` keyword). At runtime, the pattern is repeated for each element in the substituted pack.

The following example demonstrates these concepts together:

```swift
struct Pair<First, Second> {
  init(_ first: First, _ second: Second)
}

func makePairs<each First, each Second>(
  firsts first: repeat each First,
  seconds second: repeat each Second
) -> (repeat Pair<each First, each Second>) {
  return (repeat Pair(each first, each second))
}

let pairs = makePairs(firsts: 1, "hello", seconds: true, 1.0)
// 'pairs' is '(Pair(1, true), Pair("hello", 2.0))'
```

The `makePairs` function declares two type parameter packs, `First` and `Second`. The value parameter packs `first` and `second` have the pack expansion types `repeat each First` and `repeat each Second`, respectively. The return type `(repeat Pair<each First, each Second>)` is a tuple type where each element is a `Pair` of elements from the `First` and `Second` parameter packs at the given tuple position.

Inside the body of `makePairs()`, `repeat Pair(each first, each second)` is a pack expansion expression referencing the value parameter packs `first` and `second`.

The call to `makePairs()` substitutes the type pack `{Int, Bool}` for `First`, and the type pack `{String, Double}` for `Second`. These substitutions are deduced by the _type matching rules_, described below. The function is called with four arguments; `first` is the value pack `{1, "hello"}`, and `second` is the value pack `{true, 2.0}`.

The substituted return type is the tuple type with two elements `(Pair<Int, Bool>, Pair<String, Double>)`, and the returned value is the tuple value with two elements `(Pair(1, true), Pair("hello", 2.0))`.

## Detailed design

**Note:** While this proposal talks about "generic functions", everything also applies to initializers and subscripts nested inside types. With closure expressions, the situation is slightly more limited. Closure expressions support value parameter packs, however since closure expressions do not have polymorphic types in Swift, they're limited to referencing type parameter packs from outer scopes and cannot declare type parameter packs of their own. Also, the value parameter packs of closures cannot have argument labels, because as usual only named declarations have argument labels in Swift.

### Type parameter packs

The generic parameter list of a generic function can contain one or more _type parameter pack declarations_, written as an identifier preceded by `each`:

```swift
func variadic<each T, each U>() {}
```

When referenced from type context, this identifier resolves to a _type parameter pack_. References to type parameter packs can only appear in the following positions:

* The base type of a member type parameter pack, which is again subject to these rules
* The pattern type of a pack expansion type, where it stands for the corresponding scalar element type
* The pattern expression of a pack expansion expression, where it stands for the metatype of the corresponding scalar element type and can be used like any other scalar metatype, e.g. to call a static method, call an initializer, or reify the metatype value
* The subject type of a conformance, superclass, layout, or same-type requirement
* The constraint type of a same-type requirement

### Pack expansion type

A pack expansion type, written as `repeat P`, has a *pattern type* `P` and a non-empty set of _captured_ type parameter packs spelled with the `each` keyword. For example, the pack expansion type `repeat Array<each T>` has a pattern type `Array<each T>` that captures the type parameter pack `T`.

**Syntactic validity:** Pack expansion types can appear in the following positions:

* The type of a parameter in a function declaration, e.g. `func foo<each T>(values: repeat each T) -> Bool`
* The type of a parameter in a function type, e.g. `(repeat each T) -> Bool`
* The type of an unlabeled element in a tuple type, e.g. `(repeat each T)`

Because pack expansions can only appear in positions that accept a list of types or values, pack expansion patterns are naturally delimited by a comma, the next statement in top-level code, or an end-of-list delimiter, e.g. `)` for call argument lists or `>` for generic argument lists.

The restriction where only unlabeled elements of a tuple type may have a pack expansion type is motivated by ergonomics. If you could write `(t: repeat each T)`, then after a substitution `T := {Int, String}`, the substituted type would be `(t: Int, String)`. This would be strange, because projecting the member `t` would only produce the first element. When an unlabeled element has a pack expansion type, like `(repeat each T)`, then after the above substitution you would get `(Int, String)`. You can still write `0` to project the first element, but this is less surprising to the Swift programmer.

**Capture:** A type _captures_ a type parameter pack if the type parameter pack appears inside the pattern type, without any intervening pack expansion type. For example, if `T` and `U` are type parameter packs, then `repeat Array<(each T) -> each U>` captures both `T` and `U`. However, `repeat Array<(each T) -> (repeat each U)>` captures `T`, but *not* `U`. Only the inner pack expansion type `repeat each U` captures `U`. (Indeed, in a valid program, every reference to a type parameter pack is captured by exactly one pack expansion type.)

The captures of the pattern type are a subset of the captures of the pack expansion type itself. In some situations (described in the next section), the pack expansion type might capture a type parameter pack that does not appear in the pattern type.

**Typing rules:** A pack expansion type is _well-typed_ if replacing the captured type parameter packs in the pattern type with scalar type parameters of the same constraints produces a well-typed scalar type.

For example, if `each T` is a type parameter pack subject to the conformance requirement `each T: Hashable`, then `repeat Set<each T>` is well-typed, because `Set<T>` is well-typed given `T: Hashable`.

However, if `each T` were not subject to this conformance requirement, then `repeat Set<each T>` would not be well-typed; the user might substitute `T` with a type pack containing types that do not conform to `Hashable`, like `T := {AnyObject, Int}`, and the expanded substitution `Set<AnyObject>, Set<Int>` is not well-typed because `Set<AnyObject>` is not well-typed.

### Type substitution

Recall that a reference to a generic function from expression context always provides an implicit list of *generic arguments* which map each of the function's type parameters to a *replacement type*. The type of the expression referencing a generic declaration is derived by substituting each type parameter in the declaration's type with the corresponding replacement type.

The replacement type of a type parameter pack is always a type pack. Since type parameter packs always occur inside the pattern type of a pack expansion type, we need to define what it means to perform a substitution on a type that contains pack expansion types.

Recall that pack expansion types appear in function parameter types and tuple types. Substitution replaces each pack expansion type with an expanded type list, which is flattened into the outer type list.

**Intuition:** The substituted type list is formed by replacing the captured type parameter pack references with the corresponding elements of each replacement type pack.

For example, consider the declaration:

```swift
func variadic<each T, each U>(
  t: repeat each T, 
  u: repeat each U
) -> (Int, repeat ((each T) -> each U))
```

Suppose we reference it with the following substitutions:

```swift
T := {String, repeat each V, Float}
U := {NSObject, repeat Array<each W>, NSString}
```

The substituted return type of `variadic` becomes a tuple type with 4 elements:

```swift
(Int, (String) -> NSObject, repeat ((each V) -> Array<each W>), (Float) -> NSString)
```

**Formal algorithm:** Suppose `repeat P` is a pack expansion type with pattern type `P`, that captures a list of type parameter packs `Ti`, and let `S[Ti]` be the replacement type pack for `Ti`. We require that each `S[Ti]` has the same length; call this length `N`. If the lengths do not match, the substitution is malformed. Let `S[Ti][j]` be the `j`th element of `S[Ti]`, where `0 ≤ j < N`.

The `j`th element of the replacement type list is derived as follows:

1. If each `S[Ti][j]` is a scalar type, the element type is obtained by substituting each `Ti` with `S[Ti][j]` in the pattern type `P`.
2. If each `S[Ti][j]` is a pack expansion type, then `S[Ti][j]` = `repeat Pij` for some pattern type `Pij`. The element type is the pack expansion type `repeat Qij`, where `Qij` is obtained by substituting each `Ti` with `Pij` in the pattern type `P`.
3. Any other combination means the substitution is malformed.

When the lengths or structure of the replacement type packs do not match, the substitution is malformed. This situation is diagnosed with an error by checking generic requirements, as discussed below.

For example, the following substitutions are malformed because the lengths do not match:

```swift
T := {String, Float}
U := {NSObject}
```

The following substitutions are malformed because the replacement type packs have incompatible structure, hitting Case 3 above:

```swift
T := {repeat each V, Float}
U := {NSObject, repeat each W}
```

To clarify what it means for a type to capture a type parameter pack, consider the following:

```swift
func variadic<each T, each U>(t: repeat each T, u: repeat each U) -> (repeat (each T) -> (repeat each U))
```

The pack expansion type `repeat (each T) -> (repeat each U)` captures `T`, but not `U`. If we apply the following substitutions:

```swift
T := {Int, String}
U := {Float, Double, Character}
```

Then the substituted return type becomes a pair of function types:

```swift
((Int) -> (Float, Double, Character), (String) -> (Float, Double, Character))
```

Note that the entire replacement type pack for `U` was flattened in each repetition of the pattern type; we did not expand "across" `U`.

**Concrete pattern type:**  It is possible to construct an expression with a pack expansion type whose pattern type does not capture any type parameter packs. This is called a pack expansion type with a _concrete_ pattern type. For example, consider this declaration:

```swift
func counts<each T: Collection>(_ t: repeat each T) {
  let x = (repeat (each t).count)
}
```

The `count` property on the `Collection` protocol returns `Int`, so the type of the expression `(repeat (each t).count)` is written as the one-element tuple type `(repeat Int)` whose element is the pack expansion type `repeat Int`. While the pattern type `Int` does not capture any type parameter packs, the pack expansion type must still capture `T` to represent the fact that after expansion, the resulting tuple type has the same length as `T`. This kind of pack expansion type can arise during type inference, but it cannot be written in source.

#### Single-element pack substitution

If a parameter pack `each T` is substituted with a single element, the parenthesis around `(repeat each T)` are unwrapped to produce the element type as a scalar instead of a one-element tuple type.

For example, the following substitutions both produce the element type `Int`:
- Substituting `each T := {Int}` into `(repeat each T)`.
- Substituting `each T := {}` into `(Int, repeat each T)`.

Though unwrapping single-element tuples complicates type matching, surfacing single-element tuples in the programming model would increase the surface area of the language. One-element tuples would need to be manually unwrapped with `.0` or pattern matching in order to make use of their contents. This unwrapping would clutter up code.


### Type matching

Recall that the substitutions for a reference to a generic function are derived from the types of call argument expressions together with the contextual return type of the call, and are not explicitly written in source. This necessitates introducing new rules for _matching_ types containing pack expansions.

There are two separate rules:

- For call expressions where the callee is a named function declaration, _label matching_ is performed.
- For everything else, _type list matching_ is performed.

#### Label matching

Here, we use the same rule as the "legacy" variadic parameters that exist today. If a function declaration parameter has a pack expansion type, the parameter must either be the last parameter, or followed by a parameter with a label. A diagnostic is produced if the function declaration violates this rule.

Given a function declaration that is well-formed under this rule, type matching then uses the labels to delimit type packs. For example, the following is valid:

  ```swift
  func concat<each T, each U>(t: repeat each T, u: repeat each U) -> (repeat each T, repeat each U)

  // T := {Int, Double}
  // U := {String, Array<Int>}
  concat(t: 1, 2.0, u: "hi", [3])
  
  // substituted return type is (Int, Double, String, Array<Int>)
  ```

  while the following is not:

  ```swift
  func bad<each T, each U>(t: repeat each T, repeat each U) -> (repeat each T, repeat each U)
  // error: 'repeat each T' followed by an unlabeled parameter

  bad(1, 2.0, "hi", [3])  // ambiguous; where does 'each T' end and 'each U' start?
  ```

#### Trailing closure matching

Argument-to-parameter matching for parameter pack always uses a forward-scan for trailing closures. For example, the following code is valid:

```swift
func trailing<each T, each U>(t: repeat each T, u: repeat each U) {}

// T := {() -> Int}
// U := {}
trailing { 0 }
```

while the following produces an error:

```swift
func trailing<each T: Sequence, each U>(t: repeat each T, u: repeat each U) {}

// error: type '() -> Int' cannot conform to 'Sequence'
trailing { 0 }
```

#### Type list matching

In all other cases, we're matching two comma-separated lists of types. If either list contains two or more pack expansion types, the match remains _unsolved_, and the type checker attempts to derive substitutions by matching other types before giving up. (This allows a call to `concat()` as defined above to succeed, for example; the match between the contextual return type and `(repeat each T, repeat each U)` remains unsolved, but we are able to derive the substitutions for `T` and `U` from the call argument expressions.)

Otherwise, we match the common prefix and suffix as long as no pack expansion types appear on either side. After this has been done, there are three possibilities:

1. Left hand side contains a single pack expansion type, right hand size contains zero or more types.
2. Left hand side contains zero or more types, right hand side contains a single pack expansion type.
3. Any other combination, in which case the match fails.

For example:

```swift
func variadic<each T>(_: repeat each T) -> (Int, repeat each T, String) {}

let fn = { x, y in variadic(x, y) as (Int, Double, Float, String) }
```

Case 3 covers the case where one of the lists has a pack expansion, but the other one is too short; for example, matching `(Int, repeat each T, String, Float)` against `(Int, Float)` leaves you with `(repeat each T, String)` vs `()`, which is invalid.

If neither side contains a pack expansion type, Case 3 also subsumes the current behavior as implemented without this proposal, where type list matching always requires the two type lists to have the same length. For example, when matching `(Int, String)` against `(Int, Float, String)`, we end up with `()` vs `(Float)`, which is invalid.

The type checker derives the replacement type for `T` in the call to `variadic()` by matching the contextual return type `(Int, Double, Float, String)` against the declared return type `(Int, repeat each T, String)`. The common prefix `Int` and common suffix `String` successfully match. What remains is the pack expansion type `repeat each T` and the type list `Double, Float`. This successfully matches, deriving the substitution `T := {Double, Float}`.

While type list matching is positional, the type lists may still contain labels if we're matching two tuple types. We require the labels to match exactly when dropping the common prefix and suffix, and then we only allow Case 1 and 2 to succeed if the remaining type lists do not contain any labels.

For example, matching `(x: Int, repeat each T, z: String)` against `(x: Int, Double, y: Float, z: String)` drops the common prefix and suffix, and leaves you with the pack expansion type `repeat each T` vs the type list `Double, y: Float`, which fails because `Double: y: Float` contains a label.

However, matching `(x: Int, repeat each T, z: String)` against `(x: Int, Double, Float, z: String)` leaves you with `repeat each T` vs `Double, Float`, which succeeds with `T := {Double, Float}`, because the labels match exactly in the common prefix and suffix, and no labels remain once we get to Case 1 above.

### Member type parameter packs

If a type parameter pack `each T` is subject to a protocol conformance requirement `P`, and `P` declares an associated type `A`, then `(each T).A` is a valid pattern type for a pack expansion type, called a _member type parameter pack_.

Under substitution, a member type parameter pack projects the associated type from each element of the replacement type pack.

For example:

```swift
func variadic<each T: Sequence>(_: repeat each T) -> (repeat (each T).Element)
```

After the substitution `T := {Array<Int>, Set<String>}`, the substituted return type of this function becomes the tuple type `(Int, String)`.

We will refer to `each T` as the _root type parameter pack_ of the member type parameter packs `(each T).A` and `(each T).A.B`.

### Generic requirements

All existing kinds of generic requirements can be used inside _requirement expansions_, which represent a list of zero or more requirements. Requirement expansions are spelled with the `repeat` keyword followed by a generic requirement pattern that captures at least one type parameter pack reference spelled with the `each` keyword. Same-type requirements generalize in multiple different ways, depending on whether one or both sides involve a type parameter pack.

1. Conformance, superclass, and layout requirements where the subject type is a type parameter pack are interpreted as constraining each element of the replacement type pack:

  ```swift
  func variadic<each S>(_: repeat each S) where repeat each S: Sequence { ... }
  ```

  A valid substitution for the above might replace `S` with `{Array<Int>, Set<String>}`. Expanding the substitution into the requirement `each S: Sequence` conceptually produces the following conformance requirements: `Array<Int>: Sequence, Set<String>: Sequence`.

1. A same-type requirement where one side is a type parameter pack and the other type is a scalar type that does not capture any type parameter packs is interpreted as constraining each element of the replacement type pack to _the same_ scalar type:

  ```swift
  func variadic<each S: Sequence, T>(_: repeat each S) where repeat (each S).Element == T {}
  ```

  This is called a _same-element requirement_.

  A valid substitution for the above might replace `S` with `{Array<Int>, Set<Int>}`, and `T` with `Int`.


3. A same-type requirement where each side is a pattern type that captures at least one type parameter pack is interpreted as expanding the type packs on each side of the requirement, equating each element pair-wise.

  ```swift
  func variadic<each S: Sequence, each T>(_: repeat each S) where repeat (each S).Element == Array<each T> {}
  ```
  
  This is called a _same-type-pack requirement_.

  A valid substitution for the above might replace `S` with `{Array<Array<Int>>, Set<Array<String>>}`, and `T` with `{Int, String}`. Expanding `(each S).Element == Array<each T>` will produce the following list of same-type requirements: `Array<Array<Int>.Element == Array<Int>, Set<Array<String>>.Element == String`.

There is an additional kind of requirement called a _same-shape requirement_. There is no surface syntax for spelling a same-shape requirement; they are always inferred, as described in the next section.

**Symmetry:** Recall that same-type requirements are symmetrical, so `T == U` is equivalent to `U == T`. Therefore some of the possible cases above are not listed, but the behavior can be understood by first transposing the same-type requirement.

**Constrained protocol types:** A conformance requirement where the right hand side is a constrained protocol type `P<T0, ..., Tn>` may reference type parameter packs from the generic arguments `Ti` of the constrained protocol type. In this case, the semantics are defined in terms of the standard desugaring. Independent of the presence of type parameter packs, a conformance requirement to a constrained protocol type is equivalent to a conformance requirement to `P` together with one or more same-type requirements that constrain the primary associated types of `P` to the corresponding generic arguments `Ti`. After this desugaring step, the induced same-type requirements can then be understood by Case 2, 3, 4 or 5 above.

#### Same-shape requirements

A same-shape requirement states that two type parameter packs have the same number of elements, with pack expansion types occurring at identical positions.

This proposal does not include a spelling for same-shape requirements in the surface language; same-shape requirements are always inferred, and an explicit same-shape requirement syntax is a future direction. However, we will use the notation `shape(T) == shape(U)` to denote same-shape requirements in this proposal.

A same-shape requirement always relates two root type parameter packs. Member types always have the same shape as the root type parameter pack, so `shape(T.A) == shape(U.B)` reduces to `shape(T) == shape(U)`.

**Inference:** Same-shape requirements are inferred in the following ways:

1. A same-type-pack requirement implies a same-shape requirement between all type parameter packs captured by the pattern types on each side of the requirement.

   For example, given the parameter packs `<each First, each Second, each S: Sequence>`, the same-type-pack requirement `Pair<each First, each Second> == (each S).Element` implies `shape(First) == shape(Second), shape(First) == shape(S), shape(Second) == shape(S)`.

2. A same-shape requirement is inferred between each pair of type parameter packs captured by a pack expansion type appearing in the following positions
  * all types appearing in the requirements of a trailing `where` clause of a generic function
  * the parameter types and the return type of a generic function

Recall that if the pattern of a pack expansion type contains more than one type parameter pack, all type parameter packs must be known to have the same shape, as outlined in the [Type substitution](#type-substitution) section. Same-shape requirement inference ensures that these invariants are satisfied when the pack expansion type occurs in one of the two above positions.

If a pack expansion type appears in any other position, all type parameter packs captured by the pattern type must already be known to have the same shape, otherwise an error is diagnosed.

For example, `zip` is a generic function, and the return type `(repeat (each T, each U))` is a pack expansion type, therefore the same-shape requirement `shape(T) == shape(U)` is automatically inferred:

```swift
// Return type infers 'where length(T) == length(U)'
func zip<each T, each U>(firsts: repeat each T, seconds: repeat each U) -> (repeat (each T, each U)) {
  return (repeat (each firsts, each seconds))
}

zip(firsts: 1, 2, seconds: "hi", "bye") // okay
zip(firsts: 1, 2, seconds: "hi") // error; length requirement is unsatisfied
```

Here is an example where the same-shape requirement is not inferred:

```swift
func foo<each T, each U>(t: repeat each T, u: repeat each U) {
  let tup: (repeat (each T, each U)) = /* whatever */
}
```

The type annotation of `tup` contains a pack expansion type `repeat (each T, each U)`, which is malformed because the requirement `shape(T) == shape(U)` is unsatisfied. This pack expansion type is not subject to requirement inference because it does not occur in one of the above positions.

#### Restrictions on same-shape requirements

Type packs cannot be written directly, but requirements involving pack expansions where both sides are concrete types are desugared using the type matching algorithm. This means it is possible to write down a requirement that constrains a type parameter pack to a concrete type pack, unless some kind of restriction is imposed:

```swift
func constrain<each S: Sequence>(_: repeat each S) where (repeat (each S).Element) == (Int, String) {}
```

Furthermore, since the same-type requirement implies a same-shape requirement, we've implicitly constrained `S` to having a length of 2 elements, without knowing what those elements are.

This introduces theoretical complications. In the general case, same-type requirements on type parameter packs allows encoding arbitrary systems of integer linear equations:

```swift
// shape(Q) = 2 * shape(R) + 1
// shape(Q) = shape(S) + 2
func solve<each Q, each R, each S>(q: repeat each Q, r: repeat each R, s: repeat each S)
    where (repeat each Q) == (Int, repeat each R, repeat each R), 
          (repeat each Q) == (repeat each S, String, Bool) { }
```

While type-level linear algebra is interesting, we may not ever want to allow this in the language to avoid significant implementation complexity, and we definitely want to disallow this expressivity in this proposal.

To impose restrictions on same-shape and same-type requirements, we will formalize the concept of the “shape” of a pack, where a shape is one of:

* A single scalar type element; all scalar types have a singleton ``scalar shape''
* An abstract shape that is specific to a pack parameter
* A concrete shape that is composed of the scalar shape and abstract shapes

For example, the pack `{Int, repeat each T, U}` has a concrete shape that consists of two single elements and one abstract shape.

This proposal only enables abstract shapes. Each type parameter pack has an abstract shape, and same-shape requirements merge equivalence classes of abstract shapes. Any same-type requirement that imposes a concrete shape on a type parameter pack will be diagnosed as a *conflict*, much like other conflicting requirements such as `where T == Int, T == String` today.

This aspect of the language can evolve in a forward-compatible manner. Over time, some restrictions can be lifted, while others remain, as different use-cases for type parameter packs are revealed.

### Value parameter packs

A _value parameter pack_ represents zero or more function or macro parameters, and it is declared with a function parameter that has a pack expansion type. In the following declaration, the function parameter `value` is a value parameter pack that receives a _value pack_ consisting of zero or more argument values from the call site:

```swift
func tuplify<each T>(_ value: repeat each T) -> (repeat each T)

_ = tuplify() // T := {}, value := {}
_ = tuplify(1) // T := {Int}, value := {1}
_ = tuplify(1, "hello", [Foo()]) // T := {Int, String, [Foo]}, value := {1, "hello", [Foo()]}
```

**Syntactic validity:** A value parameter pack can only be referenced from a pack expansion expression. A pack expansion expression is written as `repeat expr`, where `expr` is an expression containing one or more value parameter packs or type parameter packs spelled with the `each` keyword. Pack expansion expressions can appear in any position that naturally accepts a list of expressions, including comma-separated lists and top-level expressions. This includes the following:

* Call arguments, e.g. `generic(repeat each value)`
* Subscript arguments, e.g. `subscriptable[repeat each index]`
* The elements of a tuple value, e.g. `(repeat each value)`
* The elements of an array literal, e.g. `[repeat each value]`

Pack expansion expressions can also appear in an expression statement at the top level of a brace statement. In this case, the semantics are the same as scalar expression statements; the expression is evaluated for its side effect and the results discarded.

**Capture:** A pack expansion expression _captures_ a value (or type) pack parameter the value (or type) pack parameter appears as a sub-expression without any intervening pack expansion expression.

Furthermore, a pack expansion expression also captures all type parameter packs captured by the types of its captured value parameter packs.

For example, say that `x` and `y` are both value parameter packs and `T` is a type parameter pack, and consider the pack expansion expression `repeat foo(each x, (each T).self, (repeat each y))`. This expression captures both the value parameter pack `x` and type parameter pack `T`, but it does not capture `y`, because `y` is captured by the inner pack expansion expression `repeat each y`. Additionally, if `x` has the type `Foo<U, (repeat each V)>`, then our expression captures `U`, but not `V`, because again, `V` is captured by the inner pack expansion type `repeat each V`.

**Typing rules:** For a pack expansion expression to be well-typed, two conditions must hold:

1. The types of all value parameter packs captured by a pack expansion expression must be related via same-shape requirements.

2. After replacing all value parameter packs with non-pack parameters that have equivalent types, the pattern expression must be well-typed.

Assuming the above hold, the type of a pack expansion expression is defined to be the pack expansion type whose pattern type is the type of the pattern expression.

**Evaluation semantics:** At runtime, each value (or type) parameter pack receives a value (or type) pack, which is a concrete list of values (or types). The same-shape requirements guarantee that all value (and type) packs have the same length, call it `N`. The evaluation semantics are that for each successive `i` such that `0 ≤ i < N`, the pattern expression is evaluated after substituting each occurrence of a value (or type) parameter pack with the `i`th element of the value (or type) pack. The evaluation proceeds from left to right according to the usual evaluation order, and the list of results from each evaluation forms the argument list for the parent expression.

For example, pack expansion expressions can be used to forward value parameter packs to other functions:

```swift
func tuplify<each T>(_ t: repeat each T) -> (repeat each T) {
  return (repeat each t)
}

func forward<each U>(u: repeat each U) {
  let _ = tuplify(repeat each u) //  T := {repeat each U}
  let _ = tuplify(repeat each u, 10) // T := {repeat each U, Int}
  let _ = tuplify(repeat each u, repeat each u) // T := {repeat each U, repeat each U}
  let _ = tuplify(repeat [each u]) // T := {repeat Array<each U>}
}
```

### Overload resolution

Generic functions can be overloaded by the "pack-ness" of their type parameters. For example, a function can have two overloads where one accepts a scalar type parameter and the other accepts a type parameter pack:

```swift
func overload<T>(_: T) {}
func overload<each T>(_: repeat each T) {}
```

If the parameters of the scalar overload have the same or refined requirements as the parameter pack overload, the scalar overload is considered a subtype of the parameter pack overload, because the parameters of the scalar overload can be forwarded to the parameter pack overload. Currently, if a function call successfully type checks with two different overloads, the subtype is preferred. This overload ranking rule generalizes to overloads with parameter packs, which effectively means that scalar overloads are preferred over parameter pack overloads when the scalar requirements meet the requirements of the parameter pack:

```swift
func overload() {}
func overload<T>(_: T) {}
func overload<each T>(_: repeat each T) {}

overload() // calls the no-parameter overload

overload(1) // calls the scalar overload

overload(1, "") // calls the parameter pack overload
```

The general overload subtype ranking rule applies after localized ranking, such as implicit conversions and optional promotions. That remains unchanged with this proposal. For example:

```swift
func overload<T>(_: T, _: Any) {}
func overload<each T>(_: repeat each T) {}

overload(1, "") // prefers the parameter pack overload because the scalar overload would require an existential conversion
```

More complex scenarios can still result in ambiguities. For example, if multiple overloads match a function call, but each parameter list can be forwarded to the other, the call is ambiguous:

```swift
func overload<each T>(_: repeat each T) {}
func overload<each T>(vals: repeat each T) {}

overload() // error: ambiguous
```

Similarly, if neither overload can forward their parameter lists to the other, the call is ambiguous:

```swift
func overload<each T: BinaryInteger>(str: String, _: repeat each T) {}
func overload<each U: StringProtocol>(str: repeat each U) {}

func test<Z: BinaryInteger & StringProtocol>(_ z: Z) {
  overload(str: "Hello, world!", z, z) // error: ambiguous
}
```

Generalizing the existing overload resolution ranking rules to parameter packs enables library authors to introduce new function overloads using parameter packs that generalize existing fixed-arity overloads while preserving the overload resolution behavior of existing code.

## Effect on ABI stability

This is still an area of open discussion, but we anticipate that generic functions with type parameter packs will not require runtime support, and thus will backward deploy. As work proceeds on the implementation, the above is subject to change.

## Alternatives considered

### Modeling packs as tuples with abstract elements

Under this alternative design, packs are just tuples with abstract elements. This model is attractive because it adds expressivity to all tuple types, but there are some significant disadvantages that make packs hard to work with:

* There is a fundamental ambiguity between forwarding a tuple with its elements flattened and passing a tuple as a single tuple value. This could be resolved by requiring a splat operator to forward the flattened elements, but it would still be valid in many cases to pass the tuple without flattening the elements. This may become a footgun, because you can easily forget to splat a tuple, which will become more problematic when tuples can conform to protocols.
* Because of the above issue, there is no clear way to zip tuples. This could be solved by having an explicit builtin to treat a tuple as a pack, which leads us back to needing a distinction between packs and tuples.

The pack parameter design where packs are distinct from tuples also does not preclude adding flexibility to all tuple types. Converting tuples to packs and expanding tuple values are both useful features and are detailed in the future directions.

### Syntax alternatives to `repeat each`

The `repeat each` syntax produces fairly verbose variadic generic code. However, the `repeat` keyword is explicit signal that the pattern is repeated under substitution, and requiring the `each` keyword for pack references indicates which types or values will be substituted in the expansion. This syntax design helps enforce the mental model that pack expansions result in iteration over each element in the parameter pack at runtime.

The following syntax alternatives were also considered.

#### The `...` operator

A previous version of this proposal used `...` as the pack expansion operator with no explicit syntax for pack elements in pattern types. This syntax choice follows precedent from C++ variadic templates and non-pack variadic parameters in Swift. However, there are some serious downsides of this choice, because ... is already a postfix unary operator in the Swift standard library that is commonly used across existing Swift code bases, which lead to the following ambiguities:

1. **Pack expansion vs non-pack variadic parameter.** Using `...` for pack expansions in parameter lists introduces an ambiguity with the use of `...` to indicate a non-pack variadic parameter. This ambiguity can arise when expanding a type parameter pack into the parameter list of a function type. For example:

```swift
struct X<U...> { }

struct Ambiguity<T...> {
  struct Inner<U...> {
    typealias A = X<((T...) -> U)...>
  }
}
```

Here, the `...` within the function type `(T...) -> U` could mean one of two things:

* The `...` defines a (non-pack) variadic parameter, so for each element `Ti` in the parameter pack, the function type has a single (non-pack) variadic parameter of type `Ti`, i.e., `(Ti...) -> Ui`. So, `Ambiguity<String, Character>.Inner<Float, Double>.A` would be equivalent to `X<(String...) -> Float, (Character...) -> Double>`.
* The `...` expands the parameter pack `T` into individual parameters for the function type, and no pack parameters remain after expansion. Only `U` is expanded by the outer `...`. So, `Ambiguity<String, Character>.Inner<Float, Double>.A` would be equivalent to `X<(String, Character) -> Float, (String, Character) -> Double>`.

2. **Pack expansion vs postfix closed-range operator.** Using `...` as the value expansion operator introduces an ambiguity with the postfix closed-range operator. This ambiguity can arise when `...` is applied to a value pack in the pattern of a value pack expansion, and the values in the pack are known to have a postfix closed-range operator, such as in the following code which passes a list of tuple arguments to `acceptAnything`:

```swift
func acceptAnything<T...>(_: T...) {}

func ranges<T..., U...>(values: T..., otherValues: U...) where T: Comparable, shape(T...) == shape(U...) {
  acceptAnything((values..., otherValues)...) 
}
```

In the above code, `values...` in the expansion pattern could mean either:

* The postfix `...` operator is called on each element in `values`, and the result is expanded pairwise with `otherValues` such that each argument has type `(PartialRangeFrom<T>, U)`
* `values` is expanded into each tuple passed to `acceptAnything`, with each element of `otherValues` appended onto the end of the tuple, and each argument has type `(T... U)`


3. **Pack expansion vs operator `...>`.** Another ambiguity arises when a pack expansion type `T...` appears as the final generic argument in the generic argument list of a generic type in expression context:

```swift
let foo = Foo<T...>()
```

Here, the ambiguous parse is with the token `...>`, which would necessitate changing the grammar so that `...>` is no longer considered as a single token, and instead parses as the token `...` followed by the token `>`.


#### Another operator

One alternative is to use a different operator, such as `*`, instead of `...`

```swift
func zip<T*, U*>(firsts: T*, seconds: U*) -> ((T, U)*) {
  return ((firsts, seconds)*)
}
```

The downsides to postfix `*` include:

* `*` is extremely subtle
* `*` evokes pointer types / a dereferencing operator to programmers familiar with other languages including C/C++, Go, Rust, etc.
* Choosing another operator does not alleviate the ambiguities in expressions, because values could also have a postfix `*` operator or any other operator symbol, leading to the same ambiguity.

#### Magic builtin `map` method

The prevalence of `map` and `zip` in Swift makes this syntax an attractive option for variadic generics:

```swift
func wrap<each T>(_ values: repeat each T) -> (repeat Wrapped<each T>) {
  return values.map { Wrapped($0) }
}
```

The downsides of a magic `map` method are:

* `.map` isn't only used for mapping elements to a new element, it's also used for direct forwarding, which is very different to the way `.map` is used today. An old design exploration for variadic generics used a map-style builtin, but allowed exact forwarding to omit the `.map { $0 }`. Privileging exact forwarding would be pretty frustrating, because you would need to add `.map { $0 }` as soon as you want to append other elements to either side of the pack expansion, and it wouldn't work for other values that you might want to turn into packs such as tuples or arrays.
* There are two very different models for working with packs; the same conceptual expansion has very different spellings at the type and value level, `repeat Wrapped<each T>` vs `values.map { Wrapped($0) }`.
* Magic map can only be applied to one pack at a time, leaving no clear way to zip packs without adding other builtins. A `zip` builtin would also be misleading, because expanding two packs in parallel does not need to iterate over the packs twice, but using `zip(t, u).map { ... }` looks that way.
* The closure-like syntax is misleading because it’s not a normal closure that you can write in the language. This operation is also very complex over packs with any structure, including concrete types, because the compiler either needs to infer a common generic signature for the closure that works for all elements, or it needs to separately type check the closure once for each element type.
* `map` would still need to be resolved via overload resolution amongst the existing overloads, so this approach doesn't help much with the type checking issues that `...` has.

## Future directions

### Variadic generic types

This proposal only supports type parameter packs on functions. A complementary proposal will describe type parameter packs on generic structs, enums and classes.

### Local value packs

This proposal only supports value packs for function parameters. The notion of a value parameter pack readily generalizes to a local variable of pack expansion type, for example:

```swift
func variadic<each T>(t: repeat each T) {
  let tt: repeat each T = repeat each t
}
```

References to `tt` have the same semantics as references to `t`, and must only appear inside other pack expansion expressions.

### Explicit type pack syntax

In this proposal, type packs do not have an explicit syntax, and a type pack is always inferred through the type matching rules. However, we could explore adding an explicit pack syntax in the future:

```swift
struct Variadic<each T> {}

extension Variadic where T == {Int, String} {} // {Int, String} is a concrete pack
```

### Pack iteration

All list operations can be expressed using pack expansion expressions by factoring code involving statements into a function or closure. However, this approach does not allow for short-circuiting, because the pattern expression will always be evaluated once for every element in the pack. Further, requiring a function or closure for code involving statements is unnatural. Allowing `for-in` loops to iterate over packs solves both of these problems.

Value packs could be expanded into the source of a `for-in` loop, allowing you to iterate over each element in the pack and bind each value to a local variable:

```swift
func allEmpty<each T>(_ array: repeat [each T]) -> Bool {
  for a in repeat each array {
    guard a.isEmpty else { return false }
  }

  return true
}
```

The type of the local variable `a` in the above example is an `Array` of an opaque element type with the requirements that are written on `each T`. For the *i*th iteration, the element type is the *i*th type parameter in the type parameter pack `T`.

### Pack element projection

Use cases for variadic generics that break up pack iteration across function calls, require random access, or operate over concrete packs can be supported in the future by projecting individual elements out from a parameter pack. Because elements of the pack have different types, there are two approaches to pack element projection; using an `Int` index which will return the dynamic type of the element, and using a statically typed index which is parameterized over the requested pack element type.

#### Dynamic pack indexing with `Int`

Dynamic pack indexing is useful when the specific type of the element is not known, or when all indices must have the same type, such as for index manipulation or storing an index value. Packs could support subscript calls with an `Int` index, which would return the dynamic type of the pack element directly as the opened underlying type that can be assigned to a local variable with opaque type. Values of this type need to be erased or cast to another type to return an element value from the function:

```swift
func element<each T: P>(at index: Int, in t: repeat each T) -> any P {
  // The subscript returns 'some P', which is erased to 'any P'
  // based on the function return type.
  let value: some P = t[index]
  return value
}
```

#### Typed pack element projection using key-paths

Some use cases for pack element projection know upfront which type within the pack will be projected, and can use a statically typed pack index. A statically typed pack index could be represented with `KeyPath` or a new `PackIndex` type, which is parameterized over the base type for access (i.e. the pack), and the resulting value type (i.e. the element within the pack to project). Pack element projection via key-paths falls out of 1) positional tuple key-paths, and 2) expanding packs into tuple values:

```swift
struct Tuple<each Elements> {
  var elements: (repeat each Elements)

  subscript<Value>(keyPath: KeyPath<(repeat each Elements), Value>) -> Value {
    return elements[keyPath: keyPath]
  }
}
```

The same positional key-path application could be supported directly on value packs.

### Value expansion operator

This proposal only supports the expansion operator on type parameter packs and value parameter packs, but there are other values that represent a list of zero or more values that the expansion operator would be useful for, including tuples and arrays. It would be desirable to introduce a new kind of expression that receives a scalar value and produces a value of pack expansion type.

Here, we use the straw-man syntax `.element` for accessing tuple elements as a pack:

```swift
func foo<each T, each U>(
  _ t: repeat each T,
  _ u: repeat each U
) {}

func bar1<each T, each U>(
  t: (repeat each T),
  u: (repeat each U)
) {
  repeat foo(each t.element, each u.element)
}

func bar2<each T, each U>(
  t: (repeat each T),
  u: (repeat each U)
) {
  repeat foo(each t.element, repeat each u.element)
}
```

Here, `bar1(t: (1, 2), u: ("a", "b"))` will evaluate:

```swift
foo(1, "a")
foo(2, "b")
```

While `bar2(t: (1, 2), u: ("a", "b"))` will evaluate:

```swift
foo(1, "a", "b")
foo(2, "a", "b")
```

The distinction can be understood in terms of our notion of _captures_ in pack expansion expressions.

### Pack destructuring operations

The statically-known shape of a pack can enable destructing packs with concrete shape into the component elements:

```swift
struct List<each Element> {
  let element: repeat each Element
}

extension List {
  func firstRemoved<First, each Rest>() -> List<repeat each Rest> where (repeat each Element) == (First, repeat each Rest) {
    let (first, rest) = (repeat each element)
    return List(repeat each rest)
  }
}

let list = List(1, "Hello", true)
let firstRemoved = list.firstRemoved() // 'List("Hello", true)'
```

The body of `firstRemoved` decomposes `Element` into the components of its shape -- one value of type `First` and a value pack of type `repeat each Rest` -- effectively removing the first element from the list.

### Tuple conformances

Parameter packs, the above future directions, and a syntax for declaring tuple conformances based on [parameterized extensions](https://github.com/apple/swift/blob/main/docs/GenericsManifesto.md#parameterized-extensions) over non-nominal types enable implementing custom tuple conformances:

```swift
extension<each T: Equatable> (repeat each T): Equatable {
  public static func ==(lhs: Self, rhs: Self) -> Bool {
    for (l, r) in repeat (each lhs.element, each rhs.element) {
      guard l == r else { return false }
    }
    return true
  }
}
```

## Revision history

Changes to the [first reviewed revision](https://github.com/swiftlang/swift-evolution/blob/b6ca38b9eee79650dce925e7aa8443a6a9e5e6ea/proposals/0393-parameter-packs.md):

* The `repeat` keyword is required for generic requirement expansions to distinguish requirement expansions from single requirements on an individual pack element nested inside of a pack expansion expression.
* Overload resolution prefers scalar overloads when the scalar overload is considered a subtype of a parameter pack overload.


## Acknowledgments

Thank you to Robert Widmann for exploring the design space of modeling packs as tuples, and to everyone who participated in earlier design discussions about variadic generics in Swift. Thank you to the many engineers who contributed to the implementation, including Sophia Poirier, Pavel Yaskevich, Nate Chandler, Hamish Knight, and Adrian Prantl.
