# Allow distinguishing between public access and public overridability

* Proposal: [SE-0117](0117-non-public-subclassable-by-default.md)
* Authors: [Javier Soto](https://github.com/JaviSoto), [John McCall](https://github.com/rjmccall)
* Review Manager: [Chris Lattner](https://github.com/lattner)
* Status: **Implemented (Swift 3.0)**
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-with-revision-se-0117-allow-distinguishing-between-public-access-and-public-overridability/3578)
* Implementation: [apple/swift#3882](https://github.com/apple/swift/pull/3882)
* Previous Revision: [1](https://github.com/swiftlang/swift-evolution/blob/367086f18a5deaf8f9dfbe3f5a4846ef19addf38/proposals/0117-non-public-subclassable-by-default.md), [2](https://github.com/swiftlang/swift-evolution/blob/2989538daa1640cfa6a56f80b5c7599967af0905/proposals/0117-non-public-subclassable-by-default.md), [3](https://github.com/swiftlang/swift-evolution/blob/15c18d24adb7e701ae831b643e0803f1b6e601d9/proposals/0117-non-public-subclassable-by-default.md)

## Introduction

Since the first release of Swift, marking a class `public` has provided
two capabilities: it allows other modules to instantiate and use the
class, and it also allows other modules to define subclasses of it.
Similarly, marking a class member (a method, property, or subscript)
`public` has provided two capabilities: it allows other modules to
use the member, and it also allows those modules to override it.

This proposal suggests distinguishing these concepts.  It creates a new
access level `open` beyond `public`; for now, `open` can only be used
on classes and overridable class members.  A `public` class will only
be *usable* by other modules, but not *subclassable*.  An `open` class
will be *both usable and subclassable*.  Similarly, a `public` member will
only be *usable* by other modules, but not *overridable*.  An `open`
member will be *both usable and overridable*.

This spirit of this proposal is to allow one to distinguish these cases while
keeping them at the same level of support: it does not adversely affect code
that is `open`, nor does it dissuade one from using `open` in their APIs. In
fact, with this proposal, `open` APIs are syntactically lighter-weight than
`public` ones.

Swift-evolution thread: <https://forums.swift.org/t/proposal-sealed-classes-by-default/3164>

## Motivation

Swift is very concerned with the problem of designing and maintaining
libraries.  Library authors must take care to avoid making promises
to their clients that will turn out to prevent later improvements to
the library's implementation.  A programming language can never fully
lift this burden, of course, but it can help to limit accidental
over-promises by reducing the number of implicit guarantees made
by code.  This has been a guiding idea throughout the Swift language
design.

For example, declarations in Swift default to `internal` access
control.  This prevents other modules from relying on code that was
never meant to be exposed, which would be a common error if the
default were `public`.  It also encourages library authors to think
more carefully about the interface they're providing to their
clients.  During initial development, while (say) a method's signature
is still in flux, it can be left `internal` without limiting the
programmer's ability to use and test it.  When it comes time to
prepare a public interface, the explicit act of adding `public`
to the method serves as a gentle nudge to think twice about the
method's name and type.  In contrast, if the method had to be made
`public` much earlier in development just to be able to use or test
it, it would be much more likely to slip through the cracks with
an unsatisfactory signature.

Method overriding is a very flexible programming
technique, but it poses a number of problems for library design.
A subclass that overrides methods of its superclass is intricately
intertwined with it.  The two systems are not composed, and their
behavior cannot be understood independently.  For example, a
programmer who changes how methods on a class delegate to each
other is very likely to break subclasses that override those
methods, as such subclasses can often only be written by
observing the existing pattern of behavior.  Within a single
module, this can be tolerable, but across library boundaries
it's very problematic unless the superclass has established
firm rules from the beginning.  It has frequently been observed
that designing a class well for subclassing takes far more effort
than just designing it for ordinary use, precisely because
these rules of delegation do need to be carefully laid out
if independently-designed subclasses are going to have any
chance of working.

Moreover, while subclassing is a temptingly simple manner of
allowing customization, it is also inherently limiting.
Subclasses cannot be independently tested using a mocked
implementation of the superclass or composed to apply the
customizations of two different subclasses to a single
instance.  Again, within a single module, where the superclass
and subclasses are co-designed, these problems are more
manageable: testing both systems in conjunction is more
reasonable, and the divergent customizations can be merged
into a single subclass.  But across library boundaries, they
become major hindrances.

Swift is committed to supporting subclassing and overriding.  But it
makes sense to be conservative about the promises that a class interface
makes merely by being public, and it makes sense to give library authors
strong tools for managing the overridability of their classes, and it
makes sense to encourage programmers to think more carefully about
overridability when they lift it into the external interface of a
library, where these problems become most apparent.

Furthermore, the things that make overriding such a powerful and flexible
tool for programmers also have a noticeable, negative impact on
performance.  Swift is a statically (not JIT) compiled language.  It is also a
high-level language with a number of intrinsic features that simplify
and generalize the programming model and/or improve the safety and
security of programming in Swift.  These features have costs, but the
language has typically been carefully designed to make those costs
amenable to optimization.  The Swift core team believes that it is
important for the success of Swift as a language that programmers
not be regularly required to abandon safety or (worse) drop down to a
completely different language just to meet their performance goals.
That is most at risk when there are flat-line inefficiencies in
simply executing code, and so we believe that it is crucial to remain
vigilant against pervasive abstraction penalties.  Therefore, while
dynamic features will always have a place in Swift, the language must
always retain some ability to statically optimize them in the
default case and without explicit user intervention.

And the costs of unrestricted overriding are quite real.  The
vast majority of class methods are never actually overridden,
which means they can be trivially devirtualized.  Devirtualization
is a very valuable optimization in its own right, but it is even
more important as an enabling optimization that allows the compiler
to reason about the behavior of the caller and callee together,
permitting it to further specialize and optimize both.  Making room
for subclassing and overrides also requires a great deal more
supporting code and metadata, which hurts binary sizes, launch times,
memory usage, and just general speed of execution.

Finally, it is a goal of Swift's language and performance design
that costs be "progressively disclosed".  Simple code that needs
fewer special guarantees should perform better and require less
boilerplate.  If a programmer merely wants to make a class
available for public use, that should not force excess annotations
and performance penalties just from the sudden possibility of
public subclassing.


## Proposed design

Introduce a new access modifier, `open` (other spellings are discussed
in the Alternatives section below).  As usual, this access modifier
is exclusive with the other access modifiers; it is not permitted
to write something like `public open`.

`open` is a context-sensitive keyword; there are no restrictions on
using or creating declarations with the name `open`.

`open` is not permitted on arbitrary declarations.  Only the specific
declarations mentioned here may be `open`.

For the purposes of interpreting existing language rules, `open`
is a higher (more permissive) access level above `public`.

For example, the true access level of a type member is computed as
the minimum of the true access level of the type and the declared
access level of the member.  If the class is `public` but the member
is `open`, the true access level is `public`.  As an exception to
this rule, the true access level of an `open` class that is a member
of an `public` type is `open`.

Similarly, rules which grant access to `public` declarations should
generally be interpreted as granting access to both `public` and
`open` declarations.

### `open` classes

A class may be declared `open`.

A class is invalid if its superclass is declared outside of the
current module and that superclass's access level is not `open`.

An `open` class may not also be declared `final`.

The superclass of an `open` class must be `open`.  This is consistent
with the existing access rule for superclasses.  It may be desirable
to lift this restriction in a future proposal.

### `open` class members

An overridable class member may be declared `open`.  Overridable
class members include properties, subscripts, and methods.

A class member that overrides a member of its superclass is invalid
if the member is declared outside of the current module and that
superclass member's access level is not `open`.  (Note that
`dynamic` members should generally be declared `open` rather
than `public`, but this is not a requirement, and the compiler
will enforce what is actually declared.)

A class member that is explicitly declared `open` may not also be
explicitly declared `final`.  This restriction applies even if the
method's true access level is lower than `open` because of the
restricted access level of its class.

The existing rules specify that a class member that overrides
a member of its superclass must have an access level that is at
least the minimum of its class's access level and the overridden
member's access level.  Therefore, if the class is `open`, and the
superclass method is `open`, the override must also be declared
`open`.  As a special case, an override that would otherwise be
required to be declared `open` may instead be declared `public`
if it is `final` or a member of a `final` class.

An `open` class member that is inherited from a superclass is
still considered `open` in the subclass unless the class is
`final`.

Note that a class member may be explicitly declared `open` even
if its class is not `open` or even `public`.  This is consistent
with the resolution of SE-0025, in which it was decided that
`public` members should be allowed within types with lower access
(but with no additional effect).

The member overridden by an `open` member does not itself need to
be `open`.  This is consistent with the existing access rule for
members, which does not even require the overridden member to be
`public`.

Initializers do not participate in `open` checking; they cannot
be declared `open`, and there are no restrictions on providing
an initializer that has the same signature as an initializer
in the superclass.  This is true even of `required` initializers.
A class's initializers provide an interface for constructing
instances of that class that is logically distinct from the
interface of its superclass, even when signatures happen to
match and there are well-understood patterns of delegation.
Constructing an object of a subclass necessarily involves running
code associated with that subclass, and there is no value in
arbitrarily restricting what initializers the subclass may
declare.

### Other considerations

Objective-C classes and methods are always imported as `open`.  This means that
the synthesized header for an Objective-C class would pervasively replace
`public` with `open` in its interface.

The `@testable` design states that tests have the extra access
permissions of the modules that they import for testing.  Accordingly,
this proposal does not change the fact that tests are allowed to
subclass non-final `internal` and `public` classes and override
non-final `internal` and `public` methods from the modules that\
they `@testable import`.

## Code examples

```swift
/// ModuleA:

// This class is not subclassable outside of ModuleA.
public class NonSubclassableParentClass {
	// This method is not overridable outside of ModuleA.
	public func foo() {}

	// This method is not overridable outside of ModuleA because
	// its class restricts its access level.
	// It is not invalid to declare it as `open`.
	open func bar() {}

	// The behavior of `final` methods remains unchanged.
	public final func baz() {}
}

// This class is subclassable both inside and outside of ModuleA.
open class SubclassableParentClass {
	// This property is not overridable outside of ModuleA.
	public var size : Int

	// This method is not overridable outside of ModuleA.
	public func foo() {}

	// This method is overridable both inside and outside of ModuleA.
	open func bar() {}

	/// The behavior of a `final` method remains unchanged.
	public final func baz() {}
}

/// The behavior of `final` classes remains unchanged.
public final class FinalClass { }
```

```swift
/// ModuleB:

import ModuleA

// This is invalid because the superclass is defined outside
// of the current module but is not `open`.
class SubclassA : NonSubclassableParentClass { }

// This is allowed since the superclass is `open`.
class SubclassB : SubclassableParentClass {
	// This is invalid because it overrides a method that is
	// defined outside of the current module but is not `open'.
	override func foo() { }

	// This is allowed since the superclass's method is overridable.
	// It does not need to be marked `open` because it is defined on
	// an `internal` class.
	override func bar() { }
}

open class SubclassC : SubclassableParentClass {
	// This is invalid because it overrides an `open` method within
	// an `open` class but is not declared `open`.
	override func bar() { }	
}

open class SubclassD : SubclassableParentClass {
	// This is valid.
	open override func bar() { }	
}

open class SubclassE : SubclassableParentClass {
	// This is also valid.
	public final override func bar() { }	
}
```

## Alternatives

An earlier version of this proposal did not make `open` default to
`public`.  That is, you would have to write `public open` to get the
same effect.  This would have the benefit of not confusingly conflating
`open` with access control, but it has the very large drawback of making
`public open` significantly less syntactically privileged than `public`.
This raises questions about "defaults" and so on that aren't really our
intent to raise.  Instead, we want to promote the idea that `open` and
`public` are alternatives.  Therefore, while the current proposal is
still "opinionated" in the sense that it gently encourages the use
of `public` by making it more consistent with other language features,
it no longer makes `open` feel second-class by forcing more boilerplate.
This is consistent with how we've expressed our opinions on, say,
`let` vs. `var`: it's an extremely casual difference with only occasional
enforced use of the former.

`open` could be legal only on class members.  Classes would remain
subclassable outside of the current module unless explicitly made `final`.
This would prevent the creation of "sealed" class hierarchies because
allowing subclassing would always allow public subclassing.  It is also
inconsistent with the general principle that restrictions on future
evolution be opt-in because it would not be legal to make a class final.
(Note that it is not legal to make a `final` class non-`final`
in a future release.)  It also has grave conceptual problems with
inherited open members of the superclass.

`open` on classes could be interpreted as granting the right to
override members.  A `public` class would be subclassable, but none
of its members would be overridable, including inherited members.
That is, a `public` class could be used as a compositional superclass,
useful for adding new storage to an existing identity but not for
messing with its invariants.  This would prevent the creation of
sealed hierarchies and is inconsistent with the general principle
that restrictions on future evolution should be opt-in.  Authors would
have no ability to reserve the right to decide later whether to
allow subclasses; declaring something `final` is irrevocable.  This
could be added in a future extension, but it is not the right rule
for `public`.

`open` could be split into different modifiers for classes and methods.
An earlier version of this proposal used `subclassable` and `overridable`.
These keywords are self-explanatory but visually heavyweight.  They also
imply too much: it seems odd that a non-`subclassable` class can be
subclassed from inside a module, but we are not proposing to make classes
and methods `final` by default.

Classes and methods could be inferred as `final` by default.  This would
avoid using different default rules inside and outside of the defining
module.  However, it is analogous to Swift defaulting to `private` instead
of `internal`.  It penalizes code that's only being used inside an
application or library by forcing the developer to micromanage access.
The cost of getting something wrong within a module is very low, since
it is easy to fix all of the clients.

Inherited methods could be made non-`open` by default.  This would
arguably be more consistent with the principle of carefully considering
the overridable interface of a class, but it would create an enormous
annotation and maintenance burden by forcing the entire overridable
interface to be restated at every level in the class hierarchy.

Overrides could be made non-`open` by default.  However, it would be
very difficult to justify this given that inherited methods stay `open`.
Because `open` now implies `public`, the burden of asking the user to
explicit about `final` vs. `open` now seems completely reasonable.

Other proposals that have been considered:

- `public(open)`, which seems visually cluttered

- `public extensible`, which is somewhat heavyweight and invites confusion
  within `extension`

We may want to reconsider the need for `final` in the light of this change.

## Impact on existing code

This would be a backwards-breaking change for all classes and methods that are
public and non-final, which code outside of their module has overridden.
Those classes/methods would fail to compile. Their superclass would need to be
changed to `open`.

It is likely that we will want the migrator to convert existing code to
use `open` for classes and methods.

## Related work

The `fragile` modifier in the Swift 4 resilience design is very similar to this,
and will follow the precedent set by these keywords.
