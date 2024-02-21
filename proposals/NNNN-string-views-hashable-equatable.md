#Add Equatable and Hashable Conformance to String views

* Proposal: SE-NNNN
* Authors: Shantini Vyas, David Smith
* Review Manager: TBD
* Status: **Implementation Blocked**
* Implementation: [#42237](https://github.com/apple/swift/pull/42237)

## Important Note

Existing apps may have already added these conformances in extensions, and Swift does not currently provide a compatible way for the stdlib to add conformances that are duplicates of existing ones from extensions. This proposal is conditional on a solution to this problem being added to the language, and is on hold until then. See the [Source Compatibility](## Source Compatibility, Effect on ABI stability, and Effect on API resilience) section for more details. 

##Glossary
* `String View`is not a Swift type per se, but is a helpful catch-all for describing a string or substring's unicode representation, for example `String.UTF8View`. It will be used here to refer to:

	* `String.UTF8View`
	* `String.UTF16View`
	* `String.UnicodeScalarView`
	* `Substring.UTF8View`
	* `Substring.UTF16View`
	* `Substring.UnicodeScalarView`

##Introduction and Motivation

String views provide useful functionality to `String` and `Substring`. We propose expanding that functionality by making them `Equatable` and `Hashable`.

Adding an `Equatable` conformance would allow one to directly compare String views and determine if they have the same value. 

```
let isEqual: Bool = "Dog".UTF8View == "Cat".UTF8View
```

Swift's `Dictionary` and `Set` enable fast, safe indexing and lookup. We propose adding  String Views as types that are available for use as `Dictionary` keys or `Set` members. 

Example usage:

```
var myDictionary: [String.UTF8View : Int]

```

```
var mySet: [String.UTF8View]
```

The original suggestion for these conformances can be found [here](https://github.com/apple/swift/issues/57837).

##Detailed design

Adding `Hashable` and `Equatable` conformance to these types is straightforward. Consider the following implementation for `String.UTF8View`:

```
extension String.UTF8View {
  @_alwaysEmitIntoClient
  public static func ==(lhs: Self, rhs: Self) -> Bool {
    lhs.elementsEqual(rhs)
  }
  
  @_alwaysEmitIntoClient
  public func hash(into hasher: inout Hasher) {
    hasher.combine(0xFF as UInt8)
    for element in self {
      hasher.combine(element)
    }
  }

  @_alwaysEmitIntoClient
  public var hashValue: Int {
    var hasher = Hasher()
    self.hash(into: &hasher)
    return hasher.finalize()
  }
}

@available(SwiftStdlib 5.7, *)
extension String.UTF8View: Equatable {}

@available(SwiftStdlib 5.7, *)
extension String.UTF8View: Hashable {}
```

The `Equatable` conformance leverages the already-optimized `elementsEqual()` function for collections. Since the String views are all collections, this will work just fine. 

The `Hashable` conformance also leverages collection functionality and uses the collection's contents and a terminator to generate a unique hash, consistent with other Standard Library collection types. 

**The implementation above does not work as of writing (Swift 5.6):**

* The above causes issues with retroactive conformances to the same. This is further discussed in the compatibility section below. 

## Source Compatibility, Effect on ABI stability, and Effect on API resilience

While this is an additive change, and as such would normally not impact ABI or API compatibility, this proposal runs into an unsolved issue in the language and cannot currently be made compatible; hence its unusual conditional status.

Specifically, there's two separate potential compatibility issues caused by "retroactive" conformances added by existing code that uses the standard library:

At compile time, the addition of `==` to types that didn't previously conform to `Equatable` can cause compilation failures due to the type checker being unable to determine which definition of `==` to use. Normally this would be resolved by the standard library's definition being shadowed by the client's definition, but because one definition may be inside an extension while the other is not, this ends up not happening in practice in some cases.

At runtime, we run into a related issue. Any dynamic dispatch of `Equatable` or `Hashable` methods must pick an implementation to call. When only a single conformance is present this is easy: use the one from the conformance. When two or more conformances are present, this is impossible to do safely in Swift as it is today.

0. Assume the existence of a standard library client that has added a conformance to `Equatable` or `Hashable` that has unusual semantics in some way (this is relatively unlikely in any individual case, but the universe of software is large).
1. If we pick the conformance from the standard library, the client loses its special semantics. Presumably it had those semantics for a reason, and will now behave incorrectly.
2. If we pick the conformance from the client, then any uses in the standard library itself (or other libraries the unusual client links) lose their *non*-special semantics, and may behave incorrectly.
3. It's unclear what "pick both" or "pick neither" would actually mean

The issues described here are very general, applicable to essentially any attempt to add new conformances to existing protocols to the standard library. Given that, we can reasonably expect that there will be a serious effort to find a solution in the future, at which point this proposal can be finalized.

##Alternatives Considered

In [this bug report](https://github.com/apple/swift/issues/57837) and linked discussions, it was raised that String views (see [Glossary](#glossary)) would additionally benefit from conformance to `Codable`. 

`Codable` conformance facilitates easy encoding and decoding between Swift objects and data sources. For a struct or enum to be `Codable`, all of its properties must also be `Codable`, hence the desire to add that conformance to String views. However, it is unclear what the expected behavior of the String views would be in this case, and we have chosen to omit `Codable` conformance from this proposal for simplicity. 

##Acknowledgements
* Karoy Lorentey 
* Holly Borla
* Steve Canon
* Alex Akers
