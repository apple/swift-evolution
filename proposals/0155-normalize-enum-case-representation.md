# Normalize Enum Case Representation

* Proposal: [SE-0155][]
* Authors: [Daniel Duan][], [Joe Groff][]
* Review Manager: [John McCall][]
* Status: **Implemented (Swift 3.0)**
* Decision Notes: [Rationale][]
* Previous Revision: [1][Revision 1], [Originally Accepted Proposal][], [Expired Proposal][]
* Bugs: [SR-4691](https://bugs.swift.org/browse/SR-4691), [SR-12206](https://bugs.swift.org/browse/SR-12206), [SR-12229](https://bugs.swift.org/browse/SR-12229)

## Introduction

In Swift 3, associated values of an enum case are represented by a tuple. This
implementation causes inconsistencies in case declaration, construction and
pattern matching in several places.

Enums, therefore, can be made more "regular" when we replace tuple as the
representation of associated case values. This proposal aims to define the
effect of doing so on various parts of the language.

Swift-evolution thread: [Normalize Enum Case Representation (rev. 2)][]

## Motivation

When user declares a case for an enum, a function which constructs the
corresponding case value is declared. We'll refer to such functions as _case
constructors_ in this proposal.

```swift
enum Expr {
    // this case declares the case constructor `Expr.elet(_:_:)`
    indirect case elet(locals: [(String, Expr)], body: Expr)
}

// f's signature is f(_: _), type is ([(String, Expr)], Expr) -> Expr
let f = Expr.elet

// `f` is just a function
f([], someExpr) // construct a `Expr.elet`
```

There are many surprising aspects of enum constructors, however:

1. After [SE-0111][], Swift function's fully qualified name consists of its base
   name and all of its argument labels. User can use the full name of the
   function at use site. In the example above, `locals` and `body` are currently
   not part of the case constructors name, therefore the expected syntax is
   invalid.

   ```swift
   func f(x: Int, y: Int) {}
   f(x: y:)(0, 0) // Okay, this is equivalent to f(x: 0, y: 0)
   Expr.elet(locals: body:)([], someExpr) // this doesn't work in Swift 3
   ```
2. Case constructors cannot include a default value for each parameter. This
   is yet another feature available to functions.

As previous mentioned, these are symptoms of associated values being a tuple
instead of having its own distinct semantics. This problem manifests more in
Swift 3's pattern matching:

1. A pattern with a single value would match and result in a tuple:

    ```swift
    // this works for reasons most user probably don't expect!
    if case .elet(let wat) = anExpr {
        eval(wat.body)
    }
    ```

2. Labels in patterns are not enforced:

    ```swift
    // note: there's no label in the first sub-pattern
    if case .elet(let p, let body: q) = anExpr {
        // code
    }
    ```

These extra rules makes pattern matching difficult to teach and to expand to
other types.

## Proposed Solution

We'll add first class syntax (which largely resemble the syntax in Swift 3) for
declaring associated values with labels. Tuple will no longer be used to
represent the aggregate of associated values for an enum case. This means
pattern matching for enum cases needs its own syntax as well (as opposed to
piggybacking on tuple patterns, which remains in the language for tuples.).

## Detailed Design

### Compound Names For Enum Constructors

Associated values' labels should be part of the enum case's constructor name.
When constructing an enum value with the case name, label names must either be
supplied in the argument list it self, or as part of the full name.

```swift
Expr.elet(locals: [], body: anExpr) // Okay, the Swift 3 way.
Expr.elet(locals: body:)([], anExpr) // Okay, equivalent to the previous line.
Expr.elet(locals: body:)(locals: 0, body: 0) // This would be an error, however.
```

Note that since the labels aren't part of a tuple, they no longer participate in
type checking, behaving consistently with functions.

```swift
let f = Expr.elet // f has type ([(String, Expr)], Expr) -> Expr
f([], anExpr) // Okay!
f(locals: [], body: anExpr) // Won't compile.
```

Enum cases should have distinct *full* names. Therefore, shared base name will
be allowed:

```swift
enum SyntaxTree {
    case type(variables: [TypeVariable])
    case type(instantiated: [Type])
}
```

Using only the base name in pattern matching for the previous example would be
ambiguous and result in an compile error. In this case, the full name must be
supplied to disambiguate.

```swift
case .type // error: ambiguous
case .type(variables: let variables) // Okay
```

### Default Parameter Values For Enum Constructors

From a user's point view, declaring an enum case should remain the same as Swift
3 except now it's possible to add `= expression` after the type of an
associated value to convey a default value for that field.

```swift
enum Animation {
    case fadeIn(duration: TimeInterval = 0.3) // Okay!
}
let anim = Animation.fadeIn() // Great!
```

Updated syntax:

```ebnf
union-style-enum-case = enum-case-name [enum-case-associated-value-clause];
enum-case-associated-value-clause = "(" ")"
                                  | "(" enum-case-associated-value-list ")";
enum-case-associated-value-list = enum-associated-value-element
                                | enum-associated-value-element ","
                                  enum-case-associated-value-list;
enum-case-associated-value-element = element-name type-annotation
                                     [enum-case-element-default-value-clause]
                                   | type
                                     [enum-case-element-default-value-clause];
element-name = identifier;
enum-case-element-default-value-clause = "=" expression;
```

### Alternative Payload-less Case Declaration

In Swift 3, the following syntax is valid:

```swift
enum Tree {
    case leaf() // the type of this constructor is confusing!
}
```

`Tree.leaf` has a very unexpected type to most Swift users: `(()) -> Tree`

We propose this syntax become illegal. User must explicitly declare
associated value of type `Void` if needed:

```swift
enum Tree {
    case leaf(Void)
}
```

## Source compatibility

Despite a few additions, case declaration remain mostly source-compatible with
Swift 3, with the exception of the change detailed in "Alternative Payload-less
Case Declaration".

Syntax for case constructor at use site remain source-compatible.

## Effect on ABI stability and resilience

After this proposal, enum cases may have compound names. This means the standard
library will expose different symbols for enum constructors. The name mangling
rules should also change accordingly.

## Alternative Considered

Between case declaration and pattern matching, there exist many reasonable
combinations of improvement. On one hand, we can optimize for consistency,
simplicity and teachability by bringing in as much similarity between enum and
other part of the language as possible. Many decisions in the first revision
were made in favor if doing so. Through the feedbacks from swift-evolution, we
found that some of the changes impedes the ergonomics of these features too much
. In this section, we describe some of the alternatives that were raised and
rejected in hope to strike a balance between the two end of the goals.

We discussed allowing user to declare a *parameter name* ("internal names")
for each associated value. Such names may be used in various rules in pattern
matching. Some feedback suggested they maybe used as property names when we
make enum case subtypes of the enum and resembles a struct. This feature is not
included in this proposal because parameter names are not very useful *today*.
Using them in patterns actually don't improve consistency as users don't use
them outside normal function definitions at all. If enum case gains a function
body in a future proposal, it'd be better to define the semantics of parameter
names then, as opposed to locking it down now.

To maintain ergonomics/source compatibility, we could allow user to choose
arbitrary bindings for each associated value. The problem is it makes the
pattern deviate a lot from declaration and makes it hard for beginners to
understand. This also decrease readability for seasoned users.

Along the same line, a pattern that gets dropped is binding all associated
values as a labeled tuple, which tuple pattern allowed in Swift 3. As T.J.
Usiyan [pointed out][TJs comment], implementation of the equality protocol would
be simplified due to tuple's conformance to `Equatable`. This feature may still
be introduced with alternative syntax (perhaps related to splats) later without
source-breakage.  And the need to implement `Equatable` may also disappear with
auto-deriving for `Equatable` conformance.

## Revision History

The [first revision of this proposal][Revision 1] mandated that the labeled form of
sub-pattern (`case .elet(locals: let x, body: let y)`) be the only acceptable
pattern. Turns out the community considers this to be too verbose in some cases.

A drafted version of this proposal considered allowing "overloaded" declaration
of enum cases (same full-name, but with associated values with different types).
We ultimately decided that this feature is out of the scope of this proposal.

The [second revision of this proposal][Originally Accepted Proposal] was accepted with revisions. As originally written, the proposal required that pattern matching against an enum match either by the name of the bound variables, or by explicit labels on the parts of the associated values:

```
enum Foo {
  case foo(bar: Int)
}

func switchFoo(x: Foo) {
  switch x {
  case .foo(let bar): // ok
  case .foo(bar: let bar): // ok
  case .foo(bar: let bas): // ok
  case .foo(let bas): // not ok
  }
}
```

However, it was decided in review that this was still too restrictive and
source-breaking, and so the core team [accepted the proposal][Rationale] with the modification that pattern matches only had to match the case declaration in arity, and case labels could be either provided or elided in their entirety, unless there was an ambiguity. Even then, as of Swift 5.2, this part of the proposal has not been implemented, and it would be a source breaking change to do so. Therefore, the "Pattern Consistency" section of the original proposal has been removed, and replaced with a ["Disambiguating pattern matches" section](https://github.com/swiftlang/swift-evolution/blob/aecced4919ab297f343dafd7235d392d8b859839/proposals/0155-normalize-enum-case-representation.md), which provided a minimal disambiguation rule for pattern matching cases that share a
base name. This new design still had not been implemented at the time the [core team adopted a new expiration policy for unimplemented proposals](https://forums.swift.org/t/addressing-unimplemented-evolution-proposals/40322), so it has expired.

[SE-0155]: 0155-normalize-enum-case-representation.md
[SE-0111]: 0111-remove-arg-label-type-significance.md
[Daniel Duan]: https://github.com/dduan
[Joe Groff]: https://github.com/jckarter
[John McCall]: https://github.com/rjmccall
[TJs comment]: https://forums.swift.org/t/draft-compound-names-for-enum-cases/4933/33
[Revision 1]: https://github.com/swiftlang/swift-evolution/blob/43ca098355762014f53e1b54e02d2f6a01253385/proposals/0155-normalize-enum-case-representation.md
[Normalize Enum Case Representation (rev. 2)]: https://forums.swift.org/t/normalize-enum-case-representation-rev-2/5395
[Originally Accepted Proposal]: https://github.com/swiftlang/swift-evolution/blob/4cbb1f1fa836496d4bfba95c4b78a9754690956d/proposals/0155-normalize-enum-case-representation.md
[Expired Proposal]: https://github.com/swiftlang/swift-evolution/blob/aecced4919ab297f343dafd7235d392d8b859839/proposals/0155-normalize-enum-case-representation.md
[Rationale]: https://forums.swift.org/t/accepted-se-0155-normalize-enum-case-representation/5732
