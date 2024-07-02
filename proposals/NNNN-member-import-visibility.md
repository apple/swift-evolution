# Member import visibility

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Allan Shortlidge](https://github.com/tshortli)
* Review Manager: TBD
* Status: **Awaiting review**
* Bug: [apple/swift#46493](https://github.com/apple/swift/issues/46493)
* Implementation: [apple/swift#72974](https://github.com/apple/swift/pull/72974), [apple/swift#73063](https://github.com/apple/swift/pull/73063)
* Upcoming Feature Flag: `MemberImportVisibility`
* Review: TBD

## Introduction

In Swift, there are rules dictating whether the name of a declaration in another module is considered in scope. For example, if you have a program that uses the `swift-algorithms` package and you want to use the global function [chain()](https://github.com/apple/swift-algorithms/blob/33abb694280321a84aa7dc9806de284afb8ca226/Sources/Algorithms/Chain.swift#L287) then you must write `import Algorithms` in the file that references that function or the compiler will consider it out of scope:

``` swift
// Missing 'import Algorithms'
let chained = chain([1], [2]) // error: Cannot find 'chain' in scope
```

The visibility rules for a member declaration, such as a method declared inside of a struct, are different though. When resolving a name to a member declaration, the member is in scope even if the module introducing the member is only *transitively* imported. A transitively imported module could be imported directly in another source file, or it could be a dependency of some direct dependency of your program. This inconsistency may be best understood as a subtle bug rather than an intentional design decision, and in a lot of Swift code it goes unnoticed. However, the import rules for members become more surprising when you consider the members of extensions, since an extension and its nominal type can be declared in different modules. 

This proposal unifies the behavior of name lookup by changing the rules to consistently require direct module imports in order to bring both top-level declarations and members into scope.

## Motivation

Suppose you have a program depends on a library named `RecipeKit`. The library interface looks like this:

```swift
// RecipeKit interface

public struct Recipe { /*...*/ }

extension String {
  /// Returns the recipe represented by this string.
  public func parse() -> Recipe?
}
```

To start, your program contains a single source file `main.swift` that imports `RecipeKit`:

```swift
// main.swift
import RecipeKit

let recipe = "2 slices of bread, 1.5 tbs peanut butter".parse()
```

Later, you decide to integrate with a new library named `GroceryKit` which happens to also declares its own `parse()` method in an extension on `String`:

```swift
// GroceryKit interface
public struct GroceryList { /*...*/ }

extension String {
  /// Returns the grocery list represented by this string.
  public func parse() -> GroceryList?
}

```

You add a second file that imports `GroceryKit`:

```swift
// Groceries.swift
import GroceryKit

var groceries = GroceryList()
// ...
```

Surprisingly, now that `GroceryKit` is a transitive dependency of `main.swift`, there's a new compilation error:

```swift
// main.swift
import RecipeKit

let recipe = "2 slices of bread, 1.5 tbs peanut butter".parse()
// error: Ambiguous use of 'parse()'
```

Before the new file was added, `parse()` could only refer to the extension member from `RecipeKit`. Now that it might also reference the extension member in `GroceryKit` the compiler considers the use of `parse()` to be ambiguous. To resolve the ambiguity, the developer must add a type annotation to the declaration of the variable `recipe` to give the compiler the additional context it needs to disambiguate:
```swift
let recipe: Recipe = "2 slices of bread, 1.5 tbs peanut butter".parse() // OK
```

This example demonstrates why "leaky" member visibility is undesirable. Although the fix for the new error is relatively simple in this code, providing disambiguation context to the compiler is not always so straightforward. Additionally, the fact that some declarations from `GroceryKit` are now visible in `main.swift` contradicts developer expectations, since visibility rules for top level declarations do not behave this way. This idiosyncrasy in Swift's import visibility rules harms local reasoning and results in confusing errors.

## Proposed solution

In a future language version, or whenever the `MemberImportVisibility` feature is enabled, the Swift compiler should only consider members from directly imported modules to be in scope.

## Detailed design

Excluding references to members from transitively imported modules in the local scope will prevent surprising ambiguities like the one detailed earlier. However, this change of behavior will also break some existing code that used to be accepted. The Swift compiler will now emit an error for member references that used to successfully resolve to a declaration in a transitively imported module:

```swift
// In this example, RecipeKit is imported in another file

// note: add import of module 'RecipeKit'

let recipe = "1 scoop ice cream, 1 tbs chocolate syrup".parse()
// error: instance method 'parse()' is inaccessible due to missing import of defining module 'RecipeKit'
```


Rather than simply indicating that the member's name is not in-scope, though, the compiler can identify the declaration you likely meant to use and suggest importing the module that defines it. An IDE may also offer a fix-it to add the missing module import to the file.

## Source compatibility

The suggested change in behavior is source breaking because it adds stricter requirements to name lookup. There is much existing Swift code that will need to be updated to adhere to these new requirements, either by introducing additional import statements in some source files or by reorganizing code among files. This change in behavior therefore must be opt-in, which is why it should be limited to a future language mode with an upcoming feature identifier that allows opt-in with previous language modes. 

## ABI compatibility

This change does not affect ABI.

## Implications on adoption

Adopting this feature ought to be straightforward because the compiler can identify module imports that are missing under the new rules and guide the developer to add an explicit import to resolve the error. Furthermore, although enabling this feature would be source breaking for many packages, it's important to note that source code that conforms to the new import requirements will be backwards compatible with older compilers and language modes that don't enforce this requirement. This feature does not require any new syntax or introduce rules that contradict rules of previous language modes.

## Future directions

#### Add module qualification syntax for extension members

This proposal seeks to give developers explicit control over which members are visible in a source file because this control can be used to prevent and resolve ambiguities that arise when different modules declare conflicting members in extensions. With this proposal implemented, if an extension member ambiguity still arises in a source file then the developer has the option of curating the imports in that file to resolve the ambiguity. This may work in some situations, but in others it may be awkward to refactor code in order to avoid importing a module that introduces a conflict. For these cases it would be useful to have a syntax that unambiguously identifies the desired extension member at the use site. For example, here's a hypothetical syntax for explicitly calling the `parse()` method declared in the module `RecipeKit`:

```swift
let recipe = "...".RecipeKit::parse()
```

## Alternatives considered

#### Introduce module qualification syntax for extension members instead

One alternative approach to the problem would be to rely exclusively on a new syntax for disambiguation of extension members (as discussed in Future directions). The limitation of that approach is that alone it only empowers the developer to solve conflicts *reactively*. On the other hand, the solution provided by this proposal is preventative because it stops unnecessary conflicts from arising in the first place. In the fullness of time, it would be best for both solutions to be available simultaneously.

## Acknowledgments

I would like to thank Doug Gregor for providing a proof-of-concept implementation of this pitch.