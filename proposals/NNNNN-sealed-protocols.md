# Sealed Protocols

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Karl Wagner](https://github.com/u/karwa)
* Review Manager: TBD
* Status: **Awaiting review**

*During the review process, add the following fields as needed:*

* Implementation: [apple/swift#21548](https://github.com/apple/swift/pull/21548)
* Decision Notes: [Rationale](https://forums.swift.org/), [Additional Commentary](https://forums.swift.org/)

## Introduction

Adds a modifier which allows `public` protocols to control whether other modules can introduce new conformances.
Allowing external conformances makes certain guarantees about requirements which protocol authors may wish to withhold.

Swift-evolution thread: [Pitch: Sealed Protocols](https://forums.swift.org/t/sealed-protocols/19118)

## Motivation


Protocols in Swift are a powerful tool to enable type-erasure and generic programming. 
Libraries often provide `public` protocols for clients to use for these purposes. 
`public` protocols are not only visible outside of their declaring module; any other module might introduce a retroactive conformances to the protocol.

Retroactive conformances are a very important feature, but allowing them limits the protocol's evolution and optimisability
in ways their authors might wish to reserve. For example, adding a new requirement to a `public` protocol requires
a default implementation for the benefit of external conformances, which might not be possible in all cases.

Similarly, when the compiler has knowledge about the conforming types, it can use optimised operations to handle existentials.
Currently, we advise to make protocols which are only conformed-to by classes inherit `AnyObject`, but this then becomes part of the protocol's ABI
and clients may depend on it. `sealed` protocols have the possibility to lower this to an 'informal' optimisation within the declaring module,
and support more patterns between conforming types.

To illustrate, the standard library provides `StringProtocol` for generic algorithms over both `String` and `Substring`, 
with [the following caveat](https://developer.apple.com/documentation/swift/stringprotocol):

> Do not declare new conformances to  `StringProtocol` . Only the  `String`  and  `Substring` types in the standard library
are valid conforming types.

Further inspection shows that `StringProtocol`'s optimised algorithms depend on the not-public storage type used by `String`.
There is no practical way for users to write their own conformances, yet it remains a useful abstraction.

Library code often encounters similar problems. A future follow-up proposal intends to use this ability to introduce
non-public requirements to protocols.


## Proposed solution


I propose to add a new attribute called `sealed`. A `sealed` protocol may be made public, but new conformances may only be introduced by the protocol's declaring module.


## Detailed design

Attempting to conform to a sealed protocol from another module will produce a compiler error.

```
// Module A.

sealed public protocol ASealedProtocol { /* ... */ }

// Okay.
extension String: ASealedProtocol { /* ... */ }
extension Int: ASealedProtocol { /* ... */ }

// --------------
// Module B.

// Error: cannot conform to sealed protocol 'ASealedProtocol' outside of its declaring module.
struct MyType: ASealedProtocol { /* ... */ }

// Error: cannot conform to sealed protocol 'ASealedProtocol' outside of its declaring module.
extension Array: ASealedProtocol { /* ... */ }
```

Refinements of sealed protocols are allowed, even across modules, and may be less-formally sealed than their parents, 
as long as no new conformances to the parent protocol are introduced.

```
// Module B.

public protocol SomeRefinement: ASealedProtocol {}
extension String: SomeRefinement {} // Okay. String already conforms to 'ASealedProtocol' from module A.

// Module C.

extension Int: SomeRefinement {} // Okay. Int already conforms to 'ASealedProtocol' from module A. 'SomeRefinement' is not sealed.
```

No other restrictions apply to sealed protocols. For instance, it is still possible to write protocol extensions 
for them or to use them in compositions.

```
// Module B.

typealias MyComposition = ASealedProtocol & SomeOtherProtocol // Okay.

extension ASealedProtocol { /* ... */ } // Okay.
```

`open` classes may conform to `sealed` protocols, as the conformance by subclasses is inherited from the declaring module. 
Importantly, the library author may freely add requirements without default implementations without breaking source or
binary compatibility.

## Source compatibility

This is an additive change. 

We would 'seal' StringProtocol, but that has had a no-conformance notice since it was introduced.

## Effect on ABI stability

It is a source- and binary-compatible change to 'unseal' a sealed protocol and open it up to external conformances.
This mirrors how the `open` attribute works for classes.

Adding the `sealed` attribute to a non-sealed protocol is potentially source- and binary-breaking,
as clients may have written conformances which will no longer compile,
and the declaring module may be optimised in ways which are not compatible with those external conformances.
Again, this is similar to suddenly disallowing subclasses on a public class.

## Effect on API resilience

This change would give library authors the ability to trade retroactive conformance for more flexibility with the protocol's requirements.
It is designed as a tool to help preserve API compatibility.

## Alternatives considered


- Making `sealed` the default, and adding an `open` attribute (as we do for classes).

  There are only 2 declarations in Swift which can be subtyped: classes and protocols (in the sense that new "isa" relationships can be introduced).
  Classes follow a conservative philosophy by disallowing subtyping by default, and it feels inconsistent for protocols to do the opposite.
  The author and community would prefer to use the same approach as for classes and introduce `open` (rather than `sealed`) protocols, but 
  such a change would be _massively_ source-breaking at this point.
  
- Making `sealed` imply `public`.

  There is an argument to be made that declaring a protocol as `sealed` should imply that it is also `public`. 
  Since sealed is more restrictive than public for protocols, this would make it the new minimum access level to look for when
  deciding if a type should be exposed or not. The author feels it better to keep the explicit `public`.

- Ban `open` members from witnessing `sealed` protocol requirements.

  On the one hand, allowing conformances to be partially/totally overridden might be seen as breaking the seal.
  On the other hand, allowing it does not limit the ability of library authors to evolve the protocol in question
  (which is the primary motivation behind this feature), and expands the classes which may participate to include
  classes from 3rd-party/system libraries.
  
- Banning refinements of `sealed` protocols across modules.

  Was lifted following discussion feedback.

- Bikeshedding. 

  `closed` would also be okay, and more closely mirror the `open` keyword for classes. Previous discussions seemed to settle around the name `sealed`, but it would be interesting to see which name the community prefers.

- Do nothing. 

  Always an option :) , but undesirable because it means library authors cannot publish a protocol to be used for type-erasure or generic programming without committing to a bunch of evolution restrictions.
