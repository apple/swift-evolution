# Improved operator declarations

* Proposal: [SE-0077](0077-operator-precedence.md)
* Author: [Anton Zhilin](https://github.com/Anton3)
* Review Manager: [Joe Groff](https://github.com/jckarter)
* Status: **Implemented (Swift 3.0)**
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-with-revision-se-0077-v2-improved-operator-declarations/3321)

**Revision history**

- **[v1](https://github.com/swiftlang/swift-evolution/blob/40c2acad241106e1cfe697d0f75e1855dc9e96d5/proposals/0077-operator-precedence.md)** Initial version
- **[v2](https://github.com/swiftlang/swift-evolution/blob/1f3ae8bfecb2ba70d30767607f0bd3279feeec90/proposals/0077-operator-precedence.md)** After the first review
- **v3** After the second review

## Introduction

Replace syntax of operator declaration, and replace numerical precedence with partial ordering of operators:

```swift
// Before
infix operator <> { precedence 100 associativity left }

// After
precedencegroup ComparisonPrecedence {
  associativity: left
  higherThan: LogicalConjunctionPrecedence
}
infix operator <> : ComparisonPrecedence
```

[Swift-evolution discussion thread](https://forums.swift.org/t/proposal-custom-operators/2046)

## Motivation

### Problems with numeric definition of precedence

In the beginning, operators had nice precedence values: 90, 100, 110, 120, 130, 140, 150, 160.

As time went, new and new operators were introduced.
Precedence could not be simply changed, as this would be a breaking change.
Ranges got precedence 135, `as` got precedence 132.
`??` had precedence greater than `<`, but less than `as`, so it had to be given precedence 131.

Now it is not possible to insert any custom operator between `<` and `??`.
It is an inevitable consequence of current design: it will be impossible to insert an operator between two existing ones at some point.

### Problems with a single precedence hierarchy

Currently, if an operator needs to define precedence by comparison to one operator, it must do so for all other operators.

In many cases, this is undesirable. For example, `a & b < c` and `a / b as Double` are common error patterns.
C++ compilers sometimes emit warnings on these, but Swift does not.

The root of the problem is that precedence is currently defined for any pair of operators.
If `&` had its precedence defined only in relation to other bitwise operators and `/` – only to arithmetic operators,
we would have to use parentheses in the preceding examples. This would avoid subtle bugs.

### Problems with current operator declaration syntax

Current operator declaration syntax is basically an unstructured bag of words.
This proposal appempts to make it at least partially consistent with other Swift declarations.

## Proposed solution

### Syntax for declaration of operators and groups

Operator declarations no longer have body:

```swift
prefix operator !
infix operator +
```

Precedence groups declare optional associativity (`left` or `right`).
Infix operators can be included in a single precedence group using inheritance-like syntax:

```swift
precedencegroup Additive {
  associativity: left
}

infix operator + : Additive
infix operator - : Additive
```

### Precedence mechanism

Concept of a single precedence hierarchy is removed.
Instead, to omit parentheses in expression with two neighbouring `infix` operators, precedence relationship *must* be defined between their precedence groups.

It is performed by adding `higherThan` clause, see example:

```swift
precedencegroup Additive {
  associativity: left
}
precedencegroup Multiplicative {
  associativity: left
  higherThan: Additive
}
precedencegroup BitwiseAnd {
  associativity: left
}
infix operator + : Additive
infix operator * : Multiplicative
infix operator & : BitwiseAnd

1 + 2 * 3  // ok, precedence of * is greater than precedence of +
1 + 2 & 3  // error, precedence between + and & is not defined
```

Only one declaration of the same operator / precedence group is allowed,
meaning that new precedence relationships between existing groups cannot be added.

### Transitive precedence propagation

Compiler will apply transitivity axiom to compare precedence of two given precedence groups. Example:

```swift
precedencegroup Exponentiative {
  associativity: left
  higherThan: Multiplicative
}
infix operator ** : Exponentiative

1 + 2 ** 3  // same as 1 + (2 ** 3)
```

Here, `Exponentiative > Multiplicative` and `Multiplicative > Additive` imply `Exponentiative > Additive`.

Multiple precedence relationships can be stated for a single precedence group.

### `DefaultPrecedence`

If `infix` operator does not state group that it belongs to, it is assigned to `DefaultPrecedence` group, which is defined as follows:

```swift
precedencegroup DefaultPrecedence {
  higherThan: Ternary
}
```

The following two statements are equivalent:

```swift
infix operator |> : DefaultPrecedence
infix operator |>
```

### `assignment`

Swift 2.2 has `assignment` modifier that works as follows: an operator marked `assignment` gets folded into an optional chain,
allowing `foo?.bar += 2` to work as `foo?(.bar += 2)` instead of failing to type-check as `(foo?.bar) += 2`.

This behavior will be passed to `assignment: true` on precedence groups.

### `lowerThan` relationship

There are times when we want to insert an operator below an existing one.
If that existing operator resides in another module, we can use `lowerThan` relationship. Example:

```swift
// module Swift
precedencegroup Additive { higherThan: Range }
precedencegroup Multiplicative { higherThan: Additive }

// module A
precedencegroup Equivalence {
  higherThan: Comparative
  lowerThan: Additive  // possible, because Additive lies in another module
}
infix operator ~ : Equivalence

1 + 2 ~ 3    // same as (1 + 2) ~ 3, because Additive > Equivalence
1 * 2 ~ 3    // same as (1 * 2) ~ 3, because Multiplicative > Additive > Equivalence
1 < 2 ~ 3    // same as 1 < (2 ~ 3), because Equivalence > Comparative
1 += 2 ~ 3   // same as 1 += (2 ~ 3), because Equivalence > Comparative > Assignment
1 ... 2 ~ 3  // error, because Range and Equivalence are unrelated
```

## Detailed design

### Precedence rules

Relationships between precedence groups form a Directed Acyclic Graph.
Fetching precedence relationship between given operators is equivalent to problem of [Reachability](https://en.wikipedia.org/wiki/Reachability).

#### Transitivity check

All precedence relationships must be transitive. If we define `A < B`, `B < C` and `A > C`, it will be a compilation error.
Two examples:

```swift
precedencegroup A { higherThan: B }
precedencegroup B { higherThan: A }
// A > B > A
```

```swift
precedencegroup A { }
precedencegroup B { higherThan: A }

precedencegroup C {
  higherThan: B
  lowerThan: A
}
// C > B > A > C
```

Checking for such situations is equivalent to checking whether DAG of precedence groups contains directed loops.

#### Joining unrelated precedence groups

Precedence relationships that, by transitivity rule, create relationship between two imported groups, is an error. Example:

```swift
// Module X
precedencegroup A { }
precedencegroup C { }

// Module Y
import X
precedencegroup B {
  higherThan: A
  lowerThan: C
}
```

This results in *compilation error* "B uses transitivity to define relationship between imported groups A and C".
The rationale behind this is that otherwise one can create relationships between standard precedence groups that are confusing for the reader.

### Special operators

Built-ins `is`, `as`, `as?`, `as!`, `=`, `?:` have stated precedence, but cannot be declared using Swift syntax.

They will be hardcoded in the compiler and assigned to appropriate precedence groups,
**as if** the following declarations took place:

```swift
// NOT valid Swift
infix operator is : CastingPrecedence
infix operator as : CastingPrecedence
infix operator as? : CastingPrecedence
infix operator as! : CastingPrecedence
infix operator ?: : TernaryPrecedence
infix operator = : AssignmentPrecedence
```

### Grammar

`assignment` and `precedence` local keywords will be removed.
`precedencegroup` keyword, and `higherThan`, `lowerThan` local keywords will be added.

*operator-declaration* → *prefix-operator-declaration* | *postfix-operator-declaration* | *infix-operator-declaration*

*prefix-operator-declaration* → `prefix` `operator` *operator*

*postfix-operator-declaration* → `postfix` `operator` *operator*

*infix-operator-declaration* → `infix` `operator` *operator* *infix-operator-group*<sub>opt</sub>

*infix-operator-group* → `:` *precedence-group-name*

*precedence-group-declaration* → `precedencegroup` *precedence-group-name* `{` *precedence-group-attributes* `}`

*precedence-group-attributes* → *precedence-group-assignment*<sub>opt</sub>
*precedence-group-associativity*<sub>opt</sub>
*precedence-group-relations*<sub>opt</sub>

*precedence-group-assignment* → `assignment` `:` *boolean-literal*

*precedence-group-associativity* → `associativity` `:` *precedence-group-associativity-option*

*precedence-group-associativity-option* → `left` | `right`

*precedence-group-relations* → *precedence-group-relation* | *precedence-group-relation* *precedence-group-relations*

*precedence-group-relation* → `higherThan` `:` *precedence-group-name*

*precedence-group-relation* → `lowerThan` `:` *precedence-group-name*

*precedence-group-name* → *identifier*

### Standard library changes

```swift
precedencegroup AssignmentPrecedence {
  assignment: true
  associativity: right
}
precedencegroup TernaryPrecedence {
  associativity: right
  higherThan: AssignmentPrecedence
}
precedencegroup DefaultPrecedence {
  higherThan: TernaryPrecedence
}
precedencegroup LogicalDisjunctionPrecedence {
  associativity: left
  higherThan: TernaryPrecedence
}
precedencegroup LogicalConjunctionPrecedence {
  associativity: left
  higherThan: LogicalDisjunctionPrecedence
}
precedencegroup ComparisonPrecedence {
  higherThan: LogicalConjunctionPrecedence
}
precedencegroup NilCoalescingPrecedence {
  associativity: right
  higherThan: ComparisonPrecedence
}
precedencegroup CastingPrecedence {
  higherThan: NilCoalescingPrecedence
}
precedencegroup RangeFormationPrecedence {
  higherThan: CastingPrecedence
}
precedencegroup AdditionPrecedence {
  associativity: left
  higherThan: RangeFormationPrecedence
}
precedencegroup MultiplicationPrecedence {
  associativity: left
  higherThan: AdditionPrecedence
}
precedencegroup BitwiseShiftPrecedence {
  higherThan: MultiplicationPrecedence
}

postfix operator ++
postfix operator --
// postfix operator !

prefix operator ++
prefix operator --
prefix operator !
prefix operator ~
prefix operator +
prefix operator -

// infix operator = : AssignmentPrecedence
infix operator *=  : AssignmentPrecedence
infix operator /=  : AssignmentPrecedence
infix operator %=  : AssignmentPrecedence
infix operator +=  : AssignmentPrecedence
infix operator -=  : AssignmentPrecedence
infix operator <<= : AssignmentPrecedence
infix operator >>= : AssignmentPrecedence
infix operator &=  : AssignmentPrecedence
infix operator ^=  : AssignmentPrecedence
infix operator |=  : AssignmentPrecedence

// infix operator ?: : TernaryPrecedence

infix operator ||  : LogicalDisjunctionPrecedence

infix operator &&  : LogicalConjunctionPrecedence

infix operator <   : ComparisonPrecedence
infix operator <=  : ComparisonPrecedence
infix operator >   : ComparisonPrecedence
infix operator >=  : ComparisonPrecedence
infix operator ==  : ComparisonPrecedence
infix operator !=  : ComparisonPrecedence
infix operator === : ComparisonPrecedence
infix operator !== : ComparisonPrecedence
infix operator ~=  : ComparisonPrecedence

infix operator ??  : NilCoalescingPrecedence

// infix operator as : CastingPrecedence
// infix operator as? : CastingPrecedence
// infix operator as! : CastingPrecedence
// infix operator is : CastingPrecedence

infix operator ..< : RangeFormationPrecedence
infix operator ... : RangeFormationPrecedence

infix operator +   : AdditionPrecedence
infix operator -   : AdditionPrecedence
infix operator &+  : AdditionPrecedence
infix operator &-  : AdditionPrecedence
infix operator |   : AdditionPrecedence
infix operator ^   : AdditionPrecedence

infix operator *   : MultiplicationPrecedence
infix operator /   : MultiplicationPrecedence
infix operator %   : MultiplicationPrecedence
infix operator &*  : MultiplicationPrecedence
infix operator &   : MultiplicationPrecedence

infix operator <<  : BitwiseShiftPrecedence
infix operator >>  : BitwiseShiftPrecedence
```

## Impact on existing code

Standard library operator declarations will be rewritten, and precedence groups will be added.

User defined operators will need to be rewritten as well.
Migration tool will remove bodies of operator declarations. `infix` operators will be implicitly added to `DefaultPrecedence` group.

Code, which relies on precedence relations of user-defined operators being implicitly defined, may be broken.
This will need to be fixed manually by adding them to desired precedence group.

## Future directions

### Change precedence of the Standard Library operators

Actually, this is one of the main reasons why this proposal was created: break single hierarchy of operators from Standard Library.
But this will be the topic of another proposal, because separate discussion is needed.

## Alternatives considered

### Declare associativity and precedence separately

```swift
associativity Multiplicative left
precedence Multiplicative > Additive
precedence Exponentiative > Multiplicative
```

Appearance of precedence group name in any of these "declarations" would mean declaration of the precedence group.
Precedence relationship declaration would only allow `>` sign for consistency.

Limitations on connecting unrelated imported groups could still hold.

### Do not use precedence groups

It would make each operator define precedence relationships.

The graph of relationships would be considerably larger and less understandable in this case.

Precedence groups concept would still be present, but it would make one operator in each group "privileged":

```swift
precedence - = +
precedence &+ = +
precedence / = *
precedence % = *
precedence * > +
```

### Use meta-circular syntax

That is, if a constant is of special type, then it will be used only at compile time:

```swift
struct PrecedenceGroup {
  enum Associativity { case left, right, none }
  let associativity: Associativity
  let higherThan: [StaticString]
  let lowerThan: [StaticString]
}
let Multiplicative = PrecedenceGroup(.left, [Associativity], [])
```

> This is already strongly library-determined. The library defines what operators exist and defines their
> precedence w.r.t. each other and a small number of built-in operators. Operator precedence has to be
> communicated to the compiler somehow in order to parse code. This proposal is just deciding the syntax of
> that communication.
> 
> I see no reason to use a more conceptually complex approach when a simple declaration will do.
> 
> <cite>-- John McCall</cite>

### Replace error with warning in "joining unrelated precedence groups"

1. Simplify language model and reduce burden on compilers
2. When a precedence hierarchy is broken by some update, developers can use "a quick hack" to join
all the groups together and get their code up-and-running immediately

### Replace `precedencegroup` with `precedence`

Pros:

- Looks shorter, less bulky
- Declarations use same naming style as protocols

Cons:

- Need to take `precedence` as a keyword
- `precedencegroup` more precisely represents what it declares

### Possible syntax variations

Instead of `higherThan` and `lowerThan`, there could be:
- `above` and `below`
- `upper` and `lower`
- `greaterThan` and `lessThan`
- `strongerThan` and `weakerThan`
- `gt` and `lt`
- `before` and `after`

Instead of `associativity`, there could be `associate`.

```swift
precedence Multiplicative {
  associativity(left)
  precedence(> Additive)
  precedence(< Exponentiative)
}
```

```swift
precedence Multiplicative {
  associativity: left
  precedence: higherThan(Additive)
  precedence: lowerThan(Exponentiative)
}
```

```swift
precedence Multiplicative {
  associativity(left)
  higherThan(Additive)
  lowerThan(Exponentiative)
}
```

```swift
precedence Multiplicative {
  associativity: left,
  higherThan: Additive,
  lowerThan: Exponentiative
}
```

```swift
precedence Multiplicative {
  associativity left
  higherThan Additive
  lowerThan Exponentiative
}
```

```swift
precedence Multiplicative {
  associativity left
  > Additive
  < Exponentiative
}
```

```swift
precedence Multiplicative : associativity(left), higherThan(Additive), lowerThan(Exponentiative)
```

```swift
precedence Multiplicative : associativity left, higherThan Additive, lowerThan Exponentiative
```

```swift
precedence Multiplicative > Additive, < Exponentiative, associativity left
```

```swift
precedence left Multiplicative > Additive, < Exponentiative
```

```swift
precedence associativity(left) Multiplicative > Additive, < Exponentiative
```

```swift
// Only `>` relationships, associativity goes last
precedence Multiplicative : Additive, left

// Full syntax for complex cases
precedence Multiplicative {
  associativity left
  higherThan Additive
  lowerThan Exponentiative
}
```

```swift
// Only `>` relationships, associativity goes last
precedence Multiplicative > Additive, left

// Full syntax for complex cases
precedence Multiplicative {
  associativity left
  > Additive
  < Exponentiative
}
```
