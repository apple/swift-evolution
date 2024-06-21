# Restructuring Condition Clauses

* Proposal: [SE-0099](0099-conditionclauses.md)
* Authors: [Erica Sadun](https://github.com/erica), [Chris Lattner](https://github.com/lattner)
* Review Manager: [Joe Groff](https://github.com/jckarter)
* Status: **Implemented (Swift 3.0)**
* Previous Revision: [1](https://github.com/swiftlang/swift-evolution/blob/83053c5f5395987caf2ecb3830a5cd8dc6213237/proposals/0099-conditionclauses.md)
* Review: ([pitch](https://forums.swift.org/t/pitch-making-where-and-interchangeable-in-guard-conditions/2702)), ([review](https://forums.swift.org/t/review-se-0099-restructuring-condition-clauses/2808)), ([acceptance](https://forums.swift.org/t/accepted-with-revision-se-0099-restructuring-condition-clauses/2921))

## Introduction

Swift condition clauses appear in `guard`, `if`, and `while` statements. This proposal re-architects the condition grammar to enable an arbitrary mix of Boolean expressions, `let` conditions (which test and unwrap optionals), general `case` clauses for arbitrary pattern matching, and availability tests.  It removes `where` clauses from optional binding conditions and case conditions, and eliminates gramatical ambiguity by using commas for separation between clauses instead of using them both to separate clauses and terms within each clause.  These modifications streamline Swift's syntax and alleviate the situation where many Swift developers don't know they can use arbitrary Boolean conditions after a value binding.

Swift-evolution thread:
[\[Pitch\] making where and ,	interchangeable in guard conditions](https://forums.swift.org/t/pitch-making-where-and-interchangeable-in-guard-conditions/2702)

## Motivation

Swift currently allows an arbitrary mix of binding, patterns, availability
tests, and Boolean assertions within a single compound condition. However, its grammar includes ambiguity that force subsequent Boolean assertions to be preceded by a `where` clause (with a special-case exception made for availability tests). 

```
condition-clause → expression
condition-clause → expression, condition-list
condition-clause → condition-list
condition-clause → availability-condition, expression
condition-list → condition | condition, condition-list
condition → availability-condition | case-condition | optional-binding-condition
```

The rules are complex, confusing, and imprecise. The grammar establishes that standalone Boolean tests must either precede binding or be joined with optional binding and pattern conditions. The `where` clause, which makes sense in for loops and switch statement pattern matching, adds little to optional binding in condition clauses, as in the following example. It allows for code such as:

```swift
guard 
    x == 0,
    let y = optional where z == 2 
    else { ... 
```

In this example, the Boolean `z == 2` clause has no semantic relationship to the optional condition to which it's syntactically bound. Eliminating `where` enables the subordinate condition to stand on its own and be treated as a first class test among peers. 

The root problem lies in the condition grammar: commas are used both to separate items within a clause (e.g. in `if let x = a, y = b {`) and to separate mixed kinds of clauses (e.g. `if let x = a, case y? = b {`).  This proposal resolves this problem by retaining commas as separators between clauses (as used elsewhere in Swift) and limits clauses to single items.

After adoption of these changes, the previous example would be written in any of
these styles:

```swift
guard
    x == 0,
    let y = optional,
    z == 2 
    else { ... 

guard x == 0, let y = optional, z == 2 else { ... 
```

etc.

This approach also solves ambiguity problems with the current grammar. For example, in current Swift, the comma in the following example could indicate a separator between two different pattern matches within the "case" clause (which has a 'let' pattern), or it could be an "if case" followed by an "if let" clause:

```swift
if case let x = a, let y = b {
```

With the new approach, this is unambiguously an `if case` followed by an `if let`.  To include two `if case` clauses, repeat the `case` keyword:

```swift
if case let x = a, case let y = b {
```

The advantages in accepting this proposal are:

* The "list of lists" ambiguity problems are solved. Swift uses a cleaner and simpler grammar.
* `where` clauses are no longer used to conjoin Boolean expressions with conditional binding.  This fixes user confusion issues and addresses a problem where Boolean conditions need to be attached to arbitrary bindings.


## Detailed Design

Under this proposal, condition lists are updated to accept a grammar along the following lines:

```
‌condition-list → condition | condition , condition-list
‌condition → expression | availability-condition | case-condition | optional-binding-condition
```

Where `case-condition` and `optional-binding-condition` are limited to a single
item.

This enables `guard`, `while`, and `if` to adopt grammars like:

```
guard condition-list else code-block
while condition-list code-block
if condition-list code-block (else-clause)?
```

*Note: A repeat-while statement does not use a condition list. Its grammar is `repeat code-block while expression`*

Where clauses are removed from optional binding conditions and `case-condition`s, so:

```
optional-binding-condition → optional-binding-head (optional-binding-continuation-list)? (where-clause)?
```

becomes:

```
optional-binding-condition → optional-binding-head
```

The `optional-binding-continuation-list` is removed, disallowing comma-delineated binding of multiple items:

```swift
guard let x = opt1, y = opt2, z = opt3, booleanAssertion else { }
```

This change will not affect case-item-lists in switch statements, which are distinct from case-conditions in Swift's guard, while, and if statements. All three conditions (availability conditions, case conditions, and optional binding conditions) remain otherwise unaffected.

## Impact on Existing Code

This proposal requires migration of condition lists to replace `where` with a comma and introduce `let` in a few places.  This should be straight-forward for the compiler to address using fixit hints.

## Alternatives Considered

An earlier version of this proposal considered allowing free interchange of commas with the `where` keyword. Adjusting the `where` clause in this way wouldn't introduce the ability to mix and match Boolean expressions, availability conditions, case conditions, and optional binding conditions in condition clauses, and was therefore discarded from
consideration.

Another version retained commas and where clauses but allowed arbitrary ordering of conditions and expressions.

Another version suggested separating clauses with semicolons and newlines.

## Rationale

On June 8, 2016, this proposal was **accepted with revision** for Swift 3.
There was near unanimous agreement that the Swift 2 grammar was inconsistent
and ambiguous and should be changed; most of the disagreement centered on how.
Many alternatives were discussed, including the following:

- The proposal as originally reviewed suggests using ';' or newline as a
  separator. To many people, this looked heavy, and it's also inconsistent with
  the rest of the language, which never otherwise used semicolon as an
  intra-statement separator (except in the defunct for;; loop).
- Introducing a keyword separator, such as using 'where' everywhere or
  introducing a new 'and' keyword, is also bulky and either reads poorly or
  requires stealing new keywords.
- Some commenters suggested using '&&' for consistency with simple boolean
  conditions. This isn't workable due to precedence issues.
- The ambiguities arise from the fact that there are comma-separated lists
  within comma-separated lists—within the list of conditions, each 'case' or
  'let' condition can have multiple declarations. If we eliminated this
  feature, so that every 'case' or 'let' condition had to start with 'case' or
  'let', the ambiguity is resolved, and comma can remain the condition
  separator. This does break consistency with non-conditional 'let'
  declarations and case clauses in 'switch' but is otherwise workable.

Of these alternatives, the core team found the last one to be the best choice.
'case' and 'let' conditions should each specify a single declaration, comma
should remain the condition separator, and the 'where' keyword can be retired
from its purpose as a boolean condition introducer. Some code becomes more
verbose, but in common formatting patterns, it aligns more nicely, as in:

```swift
guard
  let x = foo(),
  let y = bar(),
  let z = bas(),
  x == y || y == z else {
}
```

and though it breaks commonality between 'let' conditions and 'let'
declarations, it's more important to preserve higher-level consistency
throughout the language in how components of expressions and statements are
separated. Thanks everyone for the discussion, and thanks Erica and Chris for
the proposal! Since, aside from the approved syntax, the fundamental thrust of
the proposal remains the same, Chris has volunteered to revise it to be in line
with the approved decision.
