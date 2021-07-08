# Accept Values for Custom Compilation Flags

* Proposal: [SE-NNNN](NNNN-compilation-flags-with-values.md)
* Authors: [John Holdsworth](https://github.com/johnno1962)
* Review Manager: TBD
* Status: **Awaiting review**
* Implementation: [apple/swift#19585](https://github.com/apple/swift/pull/19585)
* Toolchain: [available here](http://johnholdsworth.com/swift-LOCAL-2018-10-25-a-osx.tar.gz)
* Decision Notes: TBD
* Bugs: [SR-8818](https://bugs.swift.org/browse/SR-8818)

## Introduction

At present, the Swift compiler frontend is able to accept "custom compiler flags" on invocation using the `-DFLAG_NAME` option. These flags are currently used only in `#if` conditionals and it is not possible to supply values. This proposal puts forward a simple implementation where values can be supplied using the established `-DFLAG_NAME=VALUE` convention where VALUE can be integer, float, string or boolean literals. To support this, comparison operators `==`, `!=`, `>` etc. are added to the implementation of the conditional compilation directive and the following are now valid statements.

```swift
#if TOLERANCE > 0.9
#if THRESHOLD != 0.0
#if VERSION == "4.2" && TOLERANCE
```
When the compiler is invoked using these options, a new construct `#flagValue("FLAG_NAME"[, default: <expression>])` can be used in Swift expressions to refer to the literal value of the option as if it had been typed at that position in the source. As an example, it would be possible to invoke the compiler as follows:

```shell
$ swift -DBUILD_NUMBER=1234 -DTOLERANCE=0.9 -DBUILD_DATE='"Sun 23 Sep 2018"' file.swift
```
One can then refer to these variables from the code:

```swift
print("Build number \(#flagValue("BUILD_NUMBER")), built on \(#flagValue("BUILD_DATE"))")
```
Finally, as a refinement for embedding larger resources in executables, if the `FLAG_NAME` argument (or the option's value) starts with an `@` the value of the flag will be used as the path to a file which contains utf-8 encoded string data that will be read in during compilation and used the body of a raw string literal as if the file was read in. For example:

```swift
let sql = #flagValue(“@LONG_SQL_FILE")
```
For a binary resource you could use:

```swift
let image = Data(base64Encoded: #flagValue(“@IMAGE_BASE64_FILE"))
```
These simple changes combine to provide a powerful bridge between the build system/command line and the Swift language.

Swift-evolution thread: [Be able to supply values such as “-DBUILD_DATE=21/12/1953” to Swift compiler](https://forums.swift.org/t/be-able-to-supply-values-such-as-dbuild-date-21-12-1953-to-swift-compiler/11119)

## Motivation

When building an application, it is often useful to embed information in the binary determined at build time to aid with support such as a build number, build date or version. It is also useful to build an application with particular tuneable parameters proved from the test/production build configuration such as a tolerance for results, sample rate of a signal or a URL. Until now, this needed to be stored in the application's source files or an `Info.plist` or another resource but often it is not convenient to extract values from resources and sometimes not desirable that such parameter values be quite so public. By embedding resources using the `@` convention it is possible to create standalone binaries as one must on Linux.

## Proposed solution

If this proposal is accepted, the existing Swift compiler frontend flag `"-D"` would be adapted so that it is able to take values. Operation of the option without values would not be affected, the custom parameter defaulting to have a value `true` and it would also be `true` from the point of view of `#if` `#conditionals` if any value is provided unless it had the specific value `false`. As parameters can now have values, comparison operator are added to the existing `&&` and `||` supported in conditional compilation expressions with a higher precedence.

When referred to in Swift expressions using the new the new construct `#flagValue("FLAG_NAME")`, values would be "sub-lexed" to yield a literal token. This feature is implemented early on in parsing which allows parameters to have a type determined late according to their context in an expression. For example, a with compiler argument `-DNUMBER=10`, `#flagValue("NUMBER")` could have `Int` or `Float` or `Double` type depending on it's context in the expression or declaration. Interpolations are not processed inside string literals provided from the command line and give a warning (though this can be implemented.)

## Detailed design

The conditional compilation directive implementation has been renovated to accept comparison operators with higher precedence than either `||` or `&&`. Numeric comparisons: `>`, `>=`,  `<=` and `<` convert the value to a double before making the comparison. Equality operators `==` and `!=` compare strings unless one of the operands is a numeric literal in which case both values are converted to a double. If a FLAG name that has not been defined is used in a comparison operation a warning is given.

Literals in conditional compilation directives are currently restricted to floats and strings due to some legacy code that provides a fix-it for for legacy statements such as `#if 1`. It is recommended this code is removed and integers be allowed.

When parsing the new `#flagValue` construct, a check is made to see if it has the name argument of a custom compilation flag and if so, it's value extracted. The extracted value is sent to a new sub-instance of the Lexer and a token extracted. If this token is a string, numeric or boolean literal it is used as the AST "Expr" at that location in the source or an error reported. If a compilation flag of that name is not found, the parser looks for a `, default:` argument as a substitute or if there is no default specified, an error(warning?) is reported and a nil value returned. If the flag name or value specified starts with an `@` the file pointed to by the path in the compilation flag's value (with the `@` removed) is loaded and used as the body of an uninterpreted raw string literal and embedded into the binary.

## Source compatibility

This is an additive feature as custom compilation flags currently cannot have values or be referred to in expressions.

## Effect on ABI stability

N/A, creates literals as would a conventional source.

## Effect on API resilience

N/A.

## Alternatives considered

Initial versions of the proposal allowed users to refer to the compilation flag value as a sort of per-module global with a name such as `BUILD_NUMBER`, `#BUILD_NUMBER`, `$BUILD_NUMBER`. It was felt this didn't identify references to compilation flags sufficiently explicitly for developers.

In some ways, the obvious thing to do is for the compiler to do is generate a fileprivate variable for each flag in each source and object file of the module. This would loose the late typing feature of the functionality and be wasteful when parameters are not referred to. Better to introduce a new construct and apply it during early stages of parsing.

Whether string literals should require surrounding double quotes when supplied on the command line is open to discussion. This can be confusing as `"` also has a role in shell. It was decided to require them in order to be able to distinguish cleanly the string `"0"` from the numeric value `0`.

It was decided to repurpose the `-D` flag rather than add a new option for these parameter values as to an extent not being able to provide a value is more unexpected than expected. This also creates a single, less confusing shared namespace between conditionals and the new `#flagValue` construct.

The resource inclusion feature creates a string literal as there is currently no data literal in the Swift language and indeed `Data` is part of `Foundation` not the `stdlib` so this could not be built into the compiler.
