# Swift 3.0 - Released on September 13, 2016

Swift 3 focused on solidifying and maturing the Swift language and
development experience. It focused on several areas:

* **API design guidelines**: The way in which Swift is used in popular
  libraries has almost as much of an effect on the character of Swift
  code as the Swift language itself. The [API naming and design
  guidelines](https://swift.org/documentation/api-design-guidelines/) are a
  carefully crafted set of guidelines for building great Swift APIs.

* **Automatic application of naming guidelines to imported Objective-C APIs**:
  When importing Objective-C APIs, the Swift 3 compiler 
  [automatically maps](../proposals/0005-objective-c-name-translation.md) methods
  into the new Swift 3 naming guidelines, and provides a number of Objective-C
  features to control and adapt this importing.

* **Adoption of naming guidelines in key APIs**: The Swift Standard Library has
  been significantly overhauled to embrace these guidelines, and key libraries
  like [Foundation](../proposals/0069-swift-mutability-for-foundation.md) and
  [libdispatch](../proposals/0088-libdispatch-for-swift3.md) have seen major
  updates, which provide the consistent development experience we seek.

* **Swiftification of imported Objective-C APIs**: Beyond the naming guidelines,
  Swift 3 provides an improved experience for working with Objective-C APIs.
  This includes importing
  [Objective-C generic classes](../proposals/0057-importing-objc-generics.md),
  providing the ability to [import C APIs](../proposals/0044-import-as-member.md)
  into an "Object Oriented" style, much nicer
  [imported string enums](../proposals/0033-import-objc-constants.md), safer
  syntax to work with [selectors](../proposals/0022-objc-selectors.md) and
  [keypaths](../proposals/0062-objc-keypaths.md), etc.

* **Focus and refine the language**: Since Swift 3 is the last release to make
  major source breaking changes, it is also the right release to reevaluate the
  syntax and semantics of the core language.  This means that some obscure or
  problematic features will be removed, we focus on improving consistency of
  syntax in many small ways (e.g. by 
  [revising handling of parameter labels](../proposals/0046-first-label.md), and
  focus on forward looking improvements to the type system.  This serves the
  overall goal of making Swift a simpler, more predictable, and more consistent
  language over the long term.

Swift 3 is the first release to enable
broad scale adoption across multiple platforms, including significant
functionality in the [Swift core libraries](https://swift.org/core-libraries/)
(Foundation, libdispatch, XCTest, etc), portability to a number of platforms including Linux/x86, Raspberry Pi, and Android, and the [Swift package manager](https://swift.org/package-manager/) to easily manage the distribution of Swift source code.

Finally, Swift 3 also includes a mix of relatively small but important additions
to the language and standard library that make solving common problems easier and
make everything feel nicer.

## Evolution proposals included in Swift 3.0

* [SE-0002: Removing currying `func` declaration syntax](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0002-remove-currying.md)
* [SE-0003: Removing `var` from Function Parameters](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0003-remove-var-parameters.md)
* [SE-0004: Remove the `++` and `--` operators](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0004-remove-pre-post-inc-decrement.md)
* [SE-0005: Better Translation of Objective-C APIs Into Swift](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0005-objective-c-name-translation.md)
* [SE-0006: Apply API Guidelines to the Standard Library](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0006-apply-api-guidelines-to-the-standard-library.md)
* [SE-0007: Remove C-style for-loops with conditions and incrementers](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0007-remove-c-style-for-loops.md)
* [SE-0008: Add a Lazy flatMap for Sequences of Optionals](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0008-lazy-flatmap-for-optionals.md)
* [SE-0016: Adding initializers to Int and UInt to convert from UnsafePointer and UnsafeMutablePointer](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0016-initializers-for-converting-unsafe-pointers-to-ints.md)
* [SE-0017: Change `Unmanaged` to use `UnsafePointer`](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0017-convert-unmanaged-to-use-unsafepointer.md)
* [SE-0019: Swift Testing](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0019-package-manager-testing.md)
* [SE-0023: API Design Guidelines](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0023-api-guidelines.md)
* [SE-0025: Scoped Access Level](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0025-scoped-access-level.md)
* [SE-0029: Remove implicit tuple splat behavior from function applications](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0029-remove-implicit-tuple-splat.md)
* [SE-0031: Adjusting inout Declarations for Type Decoration](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0031-adjusting-inout-declarations.md)
* [SE-0032: Add `first(where:)` method to `SequenceType`](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0032-sequencetype-find.md)
* [SE-0033: Import Objective-C Constants as Swift Types](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0033-import-objc-constants.md)
* [SE-0034: Disambiguating Line Control Statements from Debugging Identifiers](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0034-disambiguating-line.md)
* [SE-0035: Limiting `inout` capture to `@noescape` contexts](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0035-limit-inout-capture.md)
* [SE-0036: Requiring Leading Dot Prefixes for Enum Instance Member Implementations](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0036-enum-dot.md)
* [SE-0037: Clarify interaction between comments & operators](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0037-clarify-comments-and-operators.md)
* [SE-0038: Package Manager C Language Target Support](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0038-swiftpm-c-language-targets.md)
* [SE-0039: Modernizing Playground Literals](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0039-playgroundliterals.md)
* [SE-0040: Replacing Equal Signs with Colons For Attribute Arguments](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0040-attributecolons.md)
* [SE-0043: Declare variables in 'case' labels with multiple patterns](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0043-declare-variables-in-case-labels-with-multiple-patterns.md)
* [SE-0044: Import as Member](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0044-import-as-member.md)
* [SE-0046: Establish consistent label behavior across all parameters including first labels](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0046-first-label.md)
* [SE-0047: Defaulting non-Void functions so they warn on unused results](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0047-nonvoid-warn.md)
* [SE-0048: Generic Type Aliases](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0048-generic-typealias.md)
* [SE-0049: Move @noescape and @autoclosure to be type attributes](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0049-noescape-autoclosure-type-attrs.md)
* [SE-0052: Change IteratorType post-nil guarantee](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0052-iterator-post-nil-guarantee.md)
* [SE-0053: Remove explicit use of `let` from Function Parameters](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0053-remove-let-from-function-parameters.md)
* [SE-0054: Abolish `ImplicitlyUnwrappedOptional` type](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0054-abolish-iuo.md)
* [SE-0055: Make unsafe pointer nullability explicit using Optional](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0055-optional-unsafe-pointers.md)
* [SE-0057: Importing Objective-C Lightweight Generics](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0057-importing-objc-generics.md)
* [SE-0059: Update API Naming Guidelines and Rewrite Set APIs Accordingly](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0059-updated-set-apis.md)
* [SE-0060: Enforcing order of defaulted parameters](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0060-defaulted-parameter-order.md)
* [SE-0061: Add Generic Result and Error Handling to autoreleasepool()](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0061-autoreleasepool-signature.md)
* [SE-0062: Referencing Objective-C key-paths](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0062-objc-keypaths.md)
* [SE-0063: SwiftPM System Module Search Paths](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0063-swiftpm-system-module-search-paths.md)
* [SE-0064: Referencing the Objective-C selector of property getters and setters](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0064-property-selectors.md)
* [SE-0065: A New Model For Collections and Indices](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0065-collections-move-indices.md)
* [SE-0066: Standardize function type argument syntax to require parentheses](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0066-standardize-function-type-syntax.md)
* [SE-0067: Enhanced Floating Point Protocols](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0067-floating-point-protocols.md)
* [SE-0069: Mutability and Foundation Value Types](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0069-swift-mutability-for-foundation.md)
* [SE-0070: Make Optional Requirements Objective-C-only](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0070-optional-requirements.md)
* [SE-0071: Allow (most) keywords in member references](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0071-member-keywords.md)
* [SE-0072: Fully eliminate implicit bridging conversions from Swift](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0072-eliminate-implicit-bridging-conversions.md)
* [SE-0076: Add overrides taking an UnsafePointer source to non-destructive copying methods on UnsafeMutablePointer](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0076-copying-to-unsafe-mutable-pointer-with-unsafe-pointer-source.md)
* [SE-0077: Improved operator declarations](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0077-operator-precedence.md)
* [SE-0081: Move `where` clause to end of declaration](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0081-move-where-expression.md)
* [SE-0085: Package Manager Command Names](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0085-package-manager-command-name.md)
* [SE-0086: Drop NS Prefix in Swift Foundation](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0086-drop-foundation-ns.md)
* [SE-0088: Modernize libdispatch for Swift 3 naming conventions](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0088-libdispatch-for-swift3.md)
* [SE-0089: Renaming `String.init<T>(_: T)`](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0089-rename-string-reflection-init.md)
* [SE-0091: Improving operator requirements in protocols](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0091-improving-operators-in-protocols.md)
* [SE-0092: Typealiases in protocols and protocol extensions](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0092-typealiases-in-protocols.md)
* [SE-0093: Adding a public `base` property to slices](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0093-slice-base.md)
* [SE-0094: Add sequence(first:next:) and sequence(state:next:) to the stdlib](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0094-sequence-function.md)
* [SE-0095: Replace `protocol<P1,P2>` syntax with `P1 & P2` syntax](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0095-any-as-existential.md)
* [SE-0096: Converting dynamicType from a property to an operator](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0096-dynamictype.md)
* [SE-0099: Restructuring Condition Clauses](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0099-conditionclauses.md)
* [SE-0101: Reconfiguring `sizeof` and related functions into a unified `MemoryLayout` struct](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0101-standardizing-sizeof-naming.md)
* [SE-0102: Remove `@noreturn` attribute and introduce an empty `Never` type](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0102-noreturn-bottom-type.md)
* [SE-0103: Make non-escaping closures the default](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0103-make-noescape-default.md)
* [SE-0106: Add a `macOS` Alias for the `OSX` Platform Configuration Test](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0106-rename-osx-to-macos.md)
* [SE-0107: UnsafeRawPointer API](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0107-unsaferawpointer.md)
* [SE-0109: Remove the `Boolean` protocol](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0109-remove-boolean.md)
* [SE-0111: Remove type system significance of function argument labels](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0111-remove-arg-label-type-significance.md)
* [SE-0112: Improved NSError Bridging](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0112-nserror-bridging.md)
* [SE-0113: Add integral rounding functions to FloatingPoint](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0113-rounding-functions-on-floatingpoint.md)
* [SE-0114: Updating Buffer &quot;Value&quot; Names to &quot;Header&quot; Names](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0114-buffer-naming.md)
* [SE-0115: Rename Literal Syntax Protocols](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0115-literal-syntax-protocols.md)
* [SE-0116: Import Objective-C `id` as Swift `Any` type](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0116-id-as-any.md)
* [SE-0117: Allow distinguishing between public access and public overridability](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0117-non-public-subclassable-by-default.md)
* [SE-0118: Closure Parameter Names and Labels](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0118-closure-parameter-names-and-labels.md)
* [SE-0120: Revise `partition` Method Signature](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0120-revise-partition-method.md)
* [SE-0121: Remove `Optional` Comparison Operators](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0121-remove-optional-comparison-operators.md)
* [SE-0124: `Int.init(ObjectIdentifier)` and `UInt.init(ObjectIdentifier)` should have a `bitPattern:` label](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0124-bitpattern-label-for-int-initializer-objectidentfier.md)
* [SE-0125: Remove `NonObjectiveCBase` and `isUniquelyReferenced`](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0125-remove-nonobjectivecbase.md)
* [SE-0127: Cleaning up stdlib Pointer and Buffer Routines](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0127-cleaning-up-stdlib-ptr-buffer.md)
* [SE-0128: Change failable UnicodeScalar initializers to failable](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0128-unicodescalar-failable-initializer.md)
* [SE-0129: Package Manager Test Naming Conventions](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0129-package-manager-test-naming-conventions.md)
* [SE-0130: Replace repeating `Character` and `UnicodeScalar` forms of String.init](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0130-string-initializers-cleanup.md)
* [SE-0131: Add `AnyHashable` to the standard library](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0131-anyhashable.md)
* [SE-0133: Rename `flatten()` to `joined()`](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0133-rename-flatten-to-joined.md)
* [SE-0134: Rename two UTF8-related properties on String](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0134-rename-string-properties.md)
* [SE-0135: Package Manager Support for Differentiating Packages by Swift version](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0135-package-manager-support-for-differentiating-packages-by-swift-version.md)
* [SE-0136: Memory Layout of Values](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0136-memory-layout-of-values.md)
* [SE-0137: Avoiding Lock-In to Legacy Protocol Designs](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0137-avoiding-lock-in.md)
