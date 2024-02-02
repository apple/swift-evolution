# Generalization of Implicit Conversions

* Proposal: [SE-NNNN](NNNN-implicit-conversions.md)
* Authors: [John Holdsworth](https://github.com/johnno1962)
* Review Manager: TBD
* Status: **Awaiting specific design**

*During the review process, add the following fields as needed:*

* Implementation: [apple/swift#37214](https://github.com/apple/swift/pull/37214)
* Decision Notes: [Rationale](https://forums.swift.org/), [Additional Commentary](https://forums.swift.org/)
* Bugs: [SR-14511](https://bugs.swift.org/browse/SR-14511)
* Previous Revision: [1](https://github.com/apple/swift-evolution/blob/...commit-ID.../proposals/NNNN-filename.md)
* Previous Proposal: [SE-XXXX](https://github.com/apple/swift-evolution/pull/1382)

## Introduction

It's proposed that a mechanism be added by a relatively small change to the Swift compiler to express the implementations of conversions from source to destination types which can be automatically applied by the compiler as it type-checks an expression. These conversions should be expressed in the Swift language itself and have as near as possible to zero cost to users who do not make use of the feature.

Related Swift-evolution threads: [Automatic Mutable Pointer Conversion](https://forums.swift.org/t/automatic-mutable-pointer-conversion/49304/31) [Pitch: Implicit Pointer Conversion for C Interoperability](https://forums.swift.org/t/pitch-implicit-pointer-conversion-for-c-interoperability/51129) [Generalization of Implicit Conversions](https://forums.swift.org/t/generalization-of-implicit-conversions/51344)

## Motivation

While Swift is capable of perhaps unprecedented expressiveness in its expressions it is very strictly typed with limited capacity to automatically make some of the smallest conversions between distinct types even when such a conversion would be lossless. This is a source of friction when interacting with data derived from C apis and a source of frustration to new users of Swift who are used to C's less rigid rules. One encounters this when trying to compare an `Int32` with and `Int `for example or trying to pass an `Int32` to a function that expects an Int. A means needs to be made available where the compiler can convert a less precise type to a more precise type with the user having to explicitly add a "cast" to match the types using an initialiser.

It is possible to think of many such implicit conversions the presence of which Swift would benefit from but the tendency until now has been to express these inside the compiler itself disempowering the average Swift developer and progressively adding complexity to the type checking operation and time spent compiling.

## Proposed solution

In it's simplest this conversion could be expressed in the following form as a new initializer (with the attribute @implicit) on the destination type:

```Swift
extension ToType {
	@implict init(from: FromType) {
		self.init(from)
	}
}
```

## Detailed design

A concrete example of this feature in operation would be adding the following extension to the standard library:

```Swift
extension Int {
  @implict init(from: Int8) {
    self.init(from)
  }
  @implict init(from: Int16) {
    self.init(from)
  }
  @implict init(from: Int32) {
    self.init(from)
  }
}
```
In a prototype it has been found it is then possible to pass Int8 values to functions expecting an Int and make comparisons between Int8 and Int values with requiring a "cast" of the less precise type as, operators are implemented as functions in the Swift language. In using the prototype it was not possible to measure any slowdown of the compiler with these conversions in place with repeated compilations of a reasonably large open source project. This may be due to a feature of the specific implementation which will discussed in more detail later.

Another example would be a new potential implementation of the bi-directional conversion between CGFloat and Double.

```Swift
extension Double {
  @preferred_implict init(from: CGFloat) {
    self.init(from)
  }
}
extension CGFloat {
  @implict init(from: Double) {
    self.init(from)
  }
}
```
Such a bidirectional conversions can create ambiguities for the compiler however as were one to compare a CGFloat to a Double the compiler does not know whether to convert the CGFloat to a Double and compare Doubles or vice versa. In this very rare (perhaps unique) case a means has to be found to specify that one direction is the preferred conversion perhaps using an alternative attribute @preferred_implict as shown in this example. 

A final example could be a developer working extensively with C strings in Swift in a particular project might find it useful to add the following conversions:

```
extension String {
    @implict init(_ from: UnsafeMutablePointer<Int8>) {
        self.init(cString: from)
    }
    @implict init(_ from: UnsafePointer<Int8>) {
        self.init(cString: from)
    }
}
```

Note it is not possible to express the reverse conversion in this design as an initializer cannot express adequately the lifetime of the resulting pointer. Also note that the initializer is unlabelled in this case as, to reuse the current implementation of the type checker an unlabelled initializer needs to be available. More on this later.

Whether developers can be trusted with this expressivity is an open question which a decision on which is at the core of whether this proposal should proceed at all. I would counter this is a debate which often comes up in in relation to whether the presence of operator overloading enhances a language. I personally am in favour of trusting the developer and empowering them over concerns about potential overuse. Swift does include operator overloading to implement its operators and despite this I'm not aware of such a powerful feature being a frequent source of incomprehension when approaching another developer's code base due to over use. It is typically, simply not used other than to implement the Swift language itself by library developers as is the intended audience of the implicit conversion feature. This reservation might be assuaged by ensuring implicit conversions are properly scoped by applying existing access control mechanisms. A refinement of this would be to require that `@implicit public init()`'s, i.e. those that library developer is exporting only be allowed on a type defined in that module to limit concerns over "leakage" of implicit conversions.

If the feature were to be made available, is it possible it could be used to re-implement some of the implicit conversions already built into the Swift compiler? The answer seems to be "yes and no". For example, it may be possible to express the promotion of a non-optional value to an optional in Swift but it would not however be possible to express the conversion of a  named tuple to an unnamed tuple in a generalized manner. In my opinion it may not be useful to pursue this anyway as the C++ code is already there and such a refactoring many serve little purpose other than to introduce subtle regressions. It is intended the feature should be primarily additive.

## Implementation

It is worth reflecting for a moment on the specifics of the implementation of the prototype fleshed out in the PR as this will help refute the other common reservation about adding implicit conversions: that they will result in slower type checking performance. To discuss this, let's break type checking into it's two distinct phases. The first, most time consuming operation is funding solutions for a particular expression in terms of concrete functions to call implemented by specific types. In the Swift compiler, this is performed by a very abstract "constraint solver". After this, a solution may have been found but it may need to be "fixed" or "repaired" by "applying" one or other of the conversions built into the compiler.

For example, for an expression trying to call a function expecting an `Int` with an `Int32` value it seems (I'm not the expert) the constraint solver identifies that an unlabelled initializer for `Int` taking an `Int32` argument is available but doesn't view this solution as being "viable". So, it is already performing the work but giving up at that point and logging an error. The implementation of the prototype simply adds a "fix"/"repair" if an @implicit initialiser is available making the solution viable after the initializer has been "applied".

This led me to conclude that type checking is not being slowed down and indeed it was not possible to measure a slowdown with repeated builds of a fairly substantial open source project with and without the feature being available or in use. Keeping implicit conversions out of the constraint solver also prevents them from cascading leading to an exponential explosion of possibilities to check. It also makes possible bi-directional conversions. For more details on the prototype, consult the provisional PR mentioned above and [the summary notes](https://github.com/apple/swift/pull/37214#issuecomment-900934932).

## Source compatibility

This is an additive feature allowing source that would previously have failed to compile without an explicit conversion to compile in future versions of swift.

## Effect on ABI stability

N/A as this is a source level feature and is additive.

## Effect on API resilience

N/A as this is a source level feature and is additive.

## Alternatives considered

Continuing to add add-hoc conversions to the C++ source of the compiler or not making more conversions available at all. An alternative form where conversions are expressed as an overloaded `__conversion()` function on the FromType returning the ToType has been considered in the past and was in fact a feature of the early Swift betas but adding it as an extension to the ToType seems to group conversions better from the point of view of localising them and allows the implementation to re-use the type checker essentially as-is.

## Acknowledgments

Swift
