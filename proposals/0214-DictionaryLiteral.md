# Renaming the `DictionaryLiteral` type to `KeyValuePairs`

* Proposal: [SE-0214](0214-DictionaryLiteral.md)
* Authors: [Erica Sadun](https://github.com/erica), [Chéyo Jiménez](https://github.com/masters3d)
* Review Manager: [Chris Lattner](https://github.com/lattner)
* Status: **Implemented (Swift 5.0)**
* Implementation: [apple/swift#16577](https://github.com/apple/swift/pull/16577)
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-with-revision-se-0214-renaming-the-dictionaryliteral-type-to-keyvaluepairs/13661)
* Previous Revision: [1](https://github.com/swiftlang/swift-evolution/blob/12315c44dd6b36fec924f4f6c30f48d8784ae4cc/proposals/0214-DictionaryLiteral.md)

## Introduction

This proposal renames the confusing and misnamed [`DictionaryLiteral`](https://github.com/apple/swift/blob/c25188bafd1c775d4ceecc4a795f614f00451bf9/stdlib/public/core/Mirror.swift#L646) type to `KeyValuePairs`. This type is neither a dictionary nor a literal. It is a list of key-value pairs.

There is no strong motivation to deprecate. The type does not produce active harm. Instead, it adds measurable (if small) utility and will be part of the ABI. A sensible renaming mitigates the most problematic issue with the type.

*This proposal was discussed in the Swift Forums on the [100% bikeshed topic: DictionaryLiteral](https://forums.swift.org/t/100-bikeshed-topic-dictionaryliteral/7385) thread.*

## Motivation

This proposal renames the standard library's `DictionaryLiteral` before the Swift Language declares ABI stability. The type is confusingly misnamed. A `DictionaryLiteral` is neither a dictionary nor a literal. 

* It offers no key-based value lookup.
* It does not represent a fixed value in source code.

It seems reasonable to give the type to a better name that fits its role and purpose.

#### Current Use:

`DictionaryLiteral` establishes the `Mirror` API's children:

```
public init<Subject>(
  _ subject: Subject, 
  children: DictionaryLiteral<String, Any>, 
  displayStyle: Mirror.DisplayStyle? = default, 
  ancestorRepresentation: Mirror.AncestorRepresentation = default
)
```

* This implementation depends on `DictionaryLiteral`'s continued existence. 
* [The `@dynamicCallable` proposal](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0216-dynamic-callable.md) will provide another use case for this type.

Even when narrowly used, a type's reach is no longer a sufficient reason to deprecate it or remove it from the language. Absent *active harm*, source stability takes precedence. In this case, the `DictionaryLiteral` type causes no measurable harm beyond API sprawl and the issues with its name. The latter is easily fixed.

#### Current Limits:

The type's doc comments note an inefficient lookup implementation. This issue can be resolved in future Swift releases if needed:

```
/// Some operations that are efficient on a dictionary are slower when using
/// `DictionaryLiteral`. In particular, to find the value matching a key, you
/// must search through every element of the collection. The call to
/// `index(where:)` in the following example must traverse the whole
/// collection to find the element that matches the predicate
```

#### Utility:

The type's support of duplicate keys could become be a feature for scanning key-value pairs:

```
/// You initialize a `DictionaryLiteral` instance using a Swift dictionary
/// literal. Besides maintaining the order of the original dictionary literal,
/// `DictionaryLiteral` also allows duplicates keys.
```

This key-value pair processing might support custom initializers. It allows duplicate keys and preserves declaration order, which are both reasonably useful features.

## Detailed Design

`DictionaryLiteral` is renamed to `KeyValuePairs`. A typealias preserves the old name for compatibility but can be deprecated as of Swift 5.0.

This name was extensively bikeshedded on the [Swift Forum thread](https://forums.swift.org/t/100-bikeshed-topic-dictionaryliteral/7385) before proposal. The runner up name was `KeyValueArray`.

## Source compatibility

The old name can live on indefinitely via a typealias (which has no ABI consequences, so could be retired at a later date once everyone has had plenty of time to address the deprecation warnings). 

Removing it as not carrying its weight (and instead using `[(Key,Value)]`, which is basically what it’s a wrapper for) is probably off the table for reasons of source stability.

## Effect on ABI stability

Should be decided before ABI Stability is declared.

## Effect on API resilience

None.

## Alternatives and Future Directions

* This proposal does not change syntax.  It processes an ordered immutable list of pairs declared using `[:]` syntax. This syntax offers better visual aesthetics than `[(,)]`.

* Swift cannot yet replace `DictionaryLiteral` with conditional conformance using `Array: ExpressibleByDictionaryLiteral where Element = (Key,Value)` because [parameterized extensions](https://github.com/apple/swift/blob/master/docs/GenericsManifesto.md#parameterized-extensions) are not yet available. Further, creating an array from a dictionary literal may be unnecessarily confusing.

* This proposal does not deprecate and remove the type, as `Mirror` relies on its existence. This proposal does not recommend the removal of the `Mirror` type as third party custom debugging features rely on this feature.

* A forum discussion considered a one-time split of the standard library, creating a "SwiftDeprecated" module that could eventually be phased out. That idea lies outside the scope of this proposal and involves a tradeoff between sunsetting APIs in the future for a slight reduction in size of today's standard library. Most applications will not use these APIs, whether such an approach is taken or not.

## Acknowledgments

Thanks, Ben Cohen, for pointing out this problem and starting the forum thread to arrive at a better name. Thanks Chris Lattner and Ted Kremenek for design direction. 
