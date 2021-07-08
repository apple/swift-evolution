# Conditionals in Collections

* Proposal: [SE-NNNN](NNNN-collection-conditionals.md)
* Authors: [John Holdsworth](https://github.com/johnno1962)
* Review Manager: TBD
* Status: **Awaiting decision**
* Implementation: [apple/swift#19347](https://github.com/apple/swift/pull/19347)
* Decision Notes: TBD
* Bugs: [SR-8743](https://bugs.swift.org/browse/SR-8743)

## Introduction

This is a lightning proposal to extend the existing Swift language slightly to allow `#if` conditional inclusion of elements in array and dictionary literals. For example:

```swift
let array = [
	1,
	#if os(Linux)
	2,
	#endif
	3]
let dictionary = [
	#if DEBUG
	"a": 1,
	#if swift(>=5.0)
	"b": 2,
	#endif
	#endif
	"c": 3]
```
Swift-evolution thread: [Allow conditional inclusion of elements in array/dictionary literals?](https://forums.swift.org/t/allow-conditional-inclusion-of-elements-in-array-dictionary-literals/16171)

## Motivation

The most notable use case for this is conditional inclusion of tests for the Swift version of XCTest though it is certain to have other applications in practice allowing data to be tailored to production/development environments, architecture or build configuration.

## Proposed solution

The solution proposed is to allow #if conditionals using their exiting syntax inside collection literals surrounding sublists of elements. These elements would be either included or not included in the resulting array or dictionary instance dependant on the truth of the `#if`, `#elseif` or `#else` i.e. whether they where "active". One new syntactic requirement is the trailing comma in sublists before or inside conditional clauses is not optional as it would normally be at the end of the collection.

## Detailed design

The implementation involves a slight modification to `Parser::parseList` to detect `#if` "statements" if they are present and call `Parser::parseIfConfig` recursively call `parseList` to gather the elements in the clauses of the conditionals. Only the elements in the "active" clause of the conditional are included in the elements of the final `CollectionExpr` AST instance after parsing.

As conditionals themselves and inactive elements are not included in the parser AST representation, a new data structure, the "Conditionals Map" is maintained on the `CollectionExpr` which is used to support features such as the AST dump and stripping conditionals from module interfaces. The syntax model of libSyntax also required minor modification.

## Source compatibility

N/A. This is an purely additive proposal for syntax that is not currently valid in Swift.

## Effect on ABI stability

N/A. This is a compile time alteration of a collections's elements. The resulting collection is a conventional container as it would have been without the conditional though exactly which elements are included can affect the collection's type.

## Effect on API resilience

N/A. This is not an API.

## Alternatives considered

It was decided to tackle this limited scope for the introduction of conditional syntax first, as specific use cases can be thought of and to be honest this has always seemed like a bit of an omission. Other areas where conditionals could be introduced abound but can be discussed with reference to their own particular subtitles of implementation separately at a later date.