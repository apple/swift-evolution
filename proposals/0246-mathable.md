# Generic Math(s) Functions

* Proposal: [SE-0246](0246-mathable.md)
* Author: [Stephen Canon](https://github.com/stephentyrone)
* Review Manager: [John McCall](https://github.com/rjmccall)
* Status: **Accepted with modifications (2019-03-28)**
* Implementation: [apple/swift#23140](https://github.com/apple/swift/pull/23140)
* Previous Revision: [1](https://github.com/swiftlang/swift-evolution/blob/b5bbc5ae1f53189641951acfd50870f5b886859e/proposals/0246-mathable.md) [2](https://github.com/swiftlang/swift-evolution/blob/3afc4c68a4062ff045415f5eafb9d4956b30551b/proposals/0246-mathable.md)
* Decision Notes: ([review](https://forums.swift.org/t/se-0246-generic-math-s-functions/21479)) ([acceptance](https://forums.swift.org/t/accepted-with-modifications-se-0246-generic-math-functions/22244/26))

**This proposal is accepted, but currently not implemented in Swift due to source breaking consequences relating
to typechecker performance and shadowing rules. These are expected to be resolved in a future release
of Swift, at which point this proposal can be implemented.**

In the mean-time, users can make use of the [Swift Numerics](https://github.com/apple/swift-numerics) package, 
which also provides this functionality.


## Introduction

This proposal introduces two new protocols to the standard library: `ElementaryFunctions`
and `Real`. These protocols combine to provide "basic math functions" in generic contexts
for floating-point and SIMD types, and provide a path to extend that functionality to
planned complex types in the future.

[Swift Evolution Pitch thread](https://forums.swift.org/t/generic-math-functions/21059)

## Motivation

`BinaryFloatingPoint` (and the protocols it refines) provides a powerful set of
abstractions for writing numerical code, but it does not include the transcendental
operations defined by the C math library, which are instead imported by the platform
overlay as a set of overloaded concrete free functions.

There are two deficiencies with the current approach. First, *what* you need to import
to get this functions varies depending on platform, forcing the familiar but awkward
`#if` dance:
```swift
#if canImport(Darwin)
import Darwin
#elseif canImport(GlibC)
...
```
This shouldn't be required for functionality that is intended to be available on all
platforms.

Second, these bindings are overloaded for the various concrete `BinaryFloatingPoint`
types, but there's no way to use them generically. Suppose we want to implement the
"sigmoid" function generically:
```swift
func sigmoid<T>(_ x: T) -> T where T: FloatingPoint {
  return 1/(1 + exp(-x))
}
```
This doesn't work, because `exp` is not available on the `FloatingPoint` protocol.
Currently, you might work around this limitation by doing something like:
```swift
func sigmoid<T>(_ x: T) -> T where T: FloatingPoint {
  return 1/(1 + T(exp(-Double(x))))
}
```
but that's messy, inefficient if T is less precise than Double, and inaccurate if
T is more precise than Double. We can and should do better in Swift.

With the changes in this proposal, the full implementation would become:
```swift
import Math

func sigmoid<T>(_ x: T) -> T where T: Real {
  return 1/(1 + exp(-x))
}
```

## Proposed solution

There are four pieces of this proposal: first, we introduce the protocol
`ElementaryFunctions`:
```swift
public protocol ElementaryFunctions {

  /// The cosine of `x`.
  static func cos(_ x: Self) -> Self

  /// The sine of `x`.
  static func sin(_ x: Self) -> Self

  /// The tangent of `x`.
  static func tan(_ x: Self) -> Self
  
  ...
}
```
Conformance to this protocol means that the elementary functions are available as 
static functions:
```swift
(swift) Float.exp(1)
// r0 : Float = 2.7182817
```
(For the full set of functions provided, see Detailed Design below). All of the standard
library `FloatingPoint` types conform to `ElementaryFunctions`; a future `Complex`
type would also conform. `SIMD` types do not conform themselves, but the operations
are defined on them when their scalar type conforms to the protocol.

The second piece of the proposal is the protocol `Real`:
```swift
public protocol Real: FloatingPoint, ElementaryFunctions {

  /// `atan(y/x)` with quadrant fixup.
  ///
  /// There is an infinite family of angles whose tangent is `y/x`.
  /// `atan2` selects the representative that is the angle between 
  /// the vector `(x, y)` and the real axis in the range [-π, π].
  static func atan2(y: Self, x: Self) -> Self

  /// The error function evaluated at `x`.
  static func erf(_ x: Self) -> Self

  /// The complimentary error function evaluated at `x`.
  static func erfc(_ x: Self) -> Self

  /// sqrt(x*x + y*y) computed without undue overflow or underflow.
  ///
  /// Returns a numerically precise result even if one or both of x*x or
  /// y*y overflow or underflow.
  static func hypot(_ x: Self, _ y: Self) -> Self
  
  ...
}
```
This protocol does not add much API surface, but it is what most users will
write generic code against. The benefit of this protocol is that it allows 
us to avoid multiple constraints for most simple generic functions, and to 
adopt a clearer naming scheme for the protocol that most users see, while
also giving `ElementaryFunctions` a more precise name, suitable for
sophisticated uses.

The third piece of the proposal is the introduction of generic free function 
implementations of math operations:

```swift
(swift) exp(1.0)
// r0 : Float = 2.7182817
```

Finally, we will update the platform imports to obsolete existing functions
covered by the new free functions, and also remove the
imports of the suffixed <math.h> functions (which were actually never
intended to be available in Swift). Updates will only be necessary with functions like `atan2(y: x:)`
where we are adding argument labels or `logGamma( )` where we have new
function names. In these cases we will deprecate the old functions instead 
of obsoleting them to allow users time to migrate.

## Detailed design

The full API provided by `ElementaryFunctions` is as follows:
```swift
/// The square root of `x`.
///
/// For real types, if `x` is negative the result is `.nan`. For complex
/// types there is a branch cut on the negative real axis.
static func sqrt(_ x: Self) -> Self

/// The cosine of `x`, interpreted as an angle in radians.
static func cos(_ x: Self) -> Self

/// The sine of `x`, interpreted as an angle in radians.
static func sin(_ x: Self) -> Self

/// The tangent of `x`, interpreted as an angle in radians.
static func tan(_ x: Self) -> Self

/// The inverse cosine of `x` in radians.
static func acos(_ x: Self) -> Self

/// The inverse sine of `x` in radians.
static func asin(_ x: Self) -> Self

/// The inverse tangent of `x` in radians.
static func atan(_ x: Self) -> Self

/// The hyperbolic cosine of `x`.
static func cosh(_ x: Self) -> Self

/// The hyperbolic sine of `x`.
static func sinh(_ x: Self) -> Self

/// The hyperbolic tangent of `x`.
static func tanh(_ x: Self) -> Self

/// The inverse hyperbolic cosine of `x`.
static func acosh(_ x: Self) -> Self

/// The inverse hyperbolic sine of `x`.
static func asinh(_ x: Self) -> Self

/// The inverse hyperbolic tangent of `x`.
static func atanh(_ x: Self) -> Self

/// The exponential function applied to `x`, or `e**x`.
static func exp(_ x: Self) -> Self

/// Two raised to to power `x`.
static func exp2(_ x: Self) -> Self

/// Ten raised to to power `x`.
static func exp10(_ x: Self) -> Self

/// `exp(x) - 1` evaluated so as to preserve accuracy close to zero.
static func expm1(_ x: Self) -> Self

/// The natural logarithm of `x`.
static func log(_ x: Self) -> Self

/// The base-two logarithm of `x`.
static func log2(_ x: Self) -> Self

/// The base-ten logarithm of `x`.
static func log10(_ x: Self) -> Self

/// `log(1 + x)` evaluated so as to preserve accuracy close to zero.
static func log1p(_ x: Self) -> Self

/// `x**y` interpreted as `exp(y * log(x))`
///
/// For real types, if `x` is negative the result is NaN, even if `y` has
/// an integral value. For complex types, there is a branch cut on the
/// negative real axis.
static func pow(_ x: Self, _ y: Self) -> Self

/// `x` raised to the `n`th power.
///
/// The product of `n` copies of `x`.
static func pow(_ x: Self, _ n: Int) -> Self

/// The `n`th root of `x`.
///
/// For real types, if `x` is negative and `n` is even, the result is NaN.
/// For complex types, there is a branch cut along the negative real axis.
static func root(_ x: Self, _ n: Int) -> Self
```
`Real` builds on this set by adding the following additional operations that are either
difficult to implement for complex types or only make sense for real types:
```swift
/// `atan(y/x)` with quadrant fixup.
///
/// There is an infinite family of angles whose tangent is `y/x`. `atan2`
/// selects the representative that is the angle between the vector `(x, y)`
/// and the real axis in the range [-π, π].
static func atan2(y: Self, x: Self) -> Self

/// The error function evaluated at `x`.
static func erf(_ x: Self) -> Self

/// The complimentary error function evaluated at `x`.
static func erfc(_ x: Self) -> Self

/// sqrt(x*x + y*y) computed without undue overflow or underflow.
///
/// Returns a numerically precise result even if one or both of x*x or
/// y*y overflow or underflow.
static func hypot(_ x: Self, _ y: Self) -> Self

/// The gamma function evaluated at `x`.
///
/// For integral `x`, `gamma(x)` is `(x-1)` factorial.
static func gamma(_ x: Self) -> Self

/// `log(gamma(x))` computed without undue overflow.
///
/// `log(abs(gamma(x)))` is returned. To recover the sign of `gamma(x)`,
/// use `signGamma(x)`.
static func logGamma(_ x: Self) -> Self

/// The sign of `gamma(x)`.
///
/// This function is typically used in conjunction with `logGamma(x)`, which
/// computes `log(abs(gamma(x)))`, to recover the sign information that is
/// lost to the absolute value.
///
/// `gamma(x)` has a simple pole at each non-positive integer and an
/// essential singularity at infinity; we arbitrarily choose to return
/// `.plus` for the sign in those cases. For all other values, `signGamma(x)`
/// is `.plus` if `x >= 0` or `trunc(x)` is odd, and `.minus` otherwise.
static func signGamma(_ x: Self) -> FloatingPointSign
```
These functions directly follow the math library names used in most other
languages, as there is not a good reason to break with existing precedent.
The changes worth noting are as follows:
- `exp10` does not exist in most C math libraries. It is a generally useful
function, corresponding to `log10`. We'll fall back on implementing it as
`pow(10, x)` on platforms that don't have it in the system library.
- There are *two* functions named `pow` with different signatures. One
implements the IEEE 754 `powr` function (nan if `x` is negative), the other
restricts the exponent to have type `Int`, and implements the IEEE 754 `pown`
function.
- The function `root` does not exist in most math libraries; it computes the
nth root of x. For now this is implemented in terms of `pow`, but we may
adopt other implementations for better speed or accuracy in the future.
- Argument labels have been added to `atan2(y:x:)`. This is the only math.h
function whose argument order regularly causes bugs, so it would be good
to clarify here.
- `logGamma` is introduced instead of the existing `lgamma`, and returns a
single value instead of a tuple. The sign is still available via a new `signGamma`
function, but requires a separate function call. The
motivation for this approach is two-fold: first, the more common use case is
to want only the first value, so returning a tuple creates noise:

       let (result, _) = lgamma(x)
       
   Second, there's an outstanding bug that results from the C interfaces being
re-exported in Swift where `lgamma` is ambiguous; it can be either the
platform shim returning `(T, Int)`, or the C library function returning
`Double`; we want to deprecate the first and make the second unavailable.
Simulataneously introducing yet another function with the same name would
create a bit of a mess.

### Future expansion
The following functions recommended by IEEE 754 are not provided at this point
(because implementations are not widely available), but are planned for future expansion,
possibly with implementation directly in Swift: `cospi`, `sinpi`, `tanpi`, `acospi`, `asinpi`,
`atanpi`, `exp2m1`, `exp10m1`, `log2p1`, `log10p1`, `compound` (these are the names
used by IEEE 754; Swift can use different names if we like).

### Functions not defined on ElementaryFunctions
The following functions are exported by <math.h>, but will not be defined on ElementaryFunctions:
`frexp`, `ilogb`, `ldexp`, `logb`, `modf`, `scalbn`, `scalbln`, `fabs`, `ceil`,
`floor`, `nearbyint`, `rint`, `lrint`, `llrint`, `round`, `lround`, `llround`, `trunc`, `fmod`,
`remainder`, `remquo`, `copysign`, `nan`, `nextafter`, `nexttoward`, `fdim`, `fmin`, `fmax`,
`fma`.

Most of these are not defined on ElementaryFunctions because they are inherently bound to the
semantics of `FloatingPoint` or `BinaryFloatingPoint`, and so cannot be defined for
types such as Complex or Decimal. Equivalents to many of them are already defined on
`[Binary]FloatingPoint` anyway--in those cases free functions are defined as generic over `FloatingPoint` or `BinaryFloatingPoint`:
```swift
@available(swift, introduced: 5.1)
@_alwaysEmitIntoClient
public func ceil<T>(_ x: T) -> T where T: FloatingPoint {
  return x.rounded(.up)
}

@available(swift, introduced: 5.1)
@_alwaysEmitIntoClient
public func floor<T>(_ x: T) -> T where T: FloatingPoint {
  return x.rounded(.down)
}

@available(swift, introduced: 5.1)
@_alwaysEmitIntoClient
public func round<T>(_ x: T) -> T where T: FloatingPoint {
  return x.rounded()
}

@available(swift, introduced: 5.1)
@_alwaysEmitIntoClient
public func trunc<T>(_ x: T) -> T where T: FloatingPoint {
  return x.rounded(.towardZero)
}

@available(swift, introduced: 5.1)
@_alwaysEmitIntoClient
public func fma<T>(_ x: T, _ y: T, _ z: T) -> T where T: FloatingPoint {
  return z.addingProduct(x, y)
}

@available(swift, introduced: 5.1)
@_alwaysEmitIntoClient
public func remainder<T>(_ x: T, _ y: T) -> T where T: FloatingPoint {
  return x.remainder(dividingBy: y)
}

@available(swift, introduced: 5.1)
@_alwaysEmitIntoClient
public func fmod<T>(_ x: T, _ y: T) -> T where T: FloatingPoint {
  return x.truncatingRemainder(dividingBy: y)
}
```
These definitions replace the definitions existing in the platform module.

A few of the other functions (`nearbyint`, `rint`) are fundamentally tied to the
C language notion of dynamic floating-point rounding-modes, which is not modeled
by Swift (and which we do not have plans to support--even if Swift adds rounding-mode
control, we should avoid the C fenv model). These are deprecated.

The remainder will not be moved into the Math module at this point, as they can
be written more naturally in terms of the `FloatingPoint` API. We intend to
deprecate them.

## Source compatibility
This is an additive change, but it entails some changes for platform modules; the existing
platform implementations provided by the Darwin or GLibc should be deprecated and
made to redirect people to the new operations.

## Effect on ABI stability
For the standard library, this is an additive change. We'll need to continue to support the
old platform hooks to provide binary stability, but will mark them deprecated or obsoleted.

## Effect on API resilience
This is an additive change.

## Alternatives considered
1. The name `ElementaryFunctions` is a marked improvement on the earlier `Mathable`,
but is still imperfect. As discussed above, the introduction of the `Real` protocol mostly 
renders this issue moot; most code will be constrained to that instead.

2. The names of these functions are strongly conserved across languages, but they are
not universal; we *could* consider more verbose names inline with Swift usual patterns.
`sine`, `cosine`, `inverseTangent`, etc. This holds some appeal especially for the more
niche functions (like `expm1`), but the weight of common practice is pretty strong here;
almost all languages use essentially the same names for these operations. Another
alternative would be to break these up into `TrigonometricFunctions`, 
`HyperbolicFunctions`, `ExponentialFunctions`, etc, but I don't think that actually
buys us very much.

3. We may also want to add `log(_ base: T, _ x: T) -> T` at some future point as a
supplement to the existing `log`, `log2`, and `log10` functions. Python and Julia both
provide a similar interface. Doing this correctly requires a building block that the C math
library doesn't provide (an extra-precise `log` or `log2` that returns a head-tail
representation of the result); without this building block rounding errors creep in even for
exact cases:

       >>> from math import log
       >>> log(3**20, 3)
       19.999999999999996

   Julia includes a warning about this in their documentation that basically says "Use log2
or log10 instead if base is 2 or 10". We could take that approach, but base 2 and 10
cover 99% of uses, so I would rather wait to provide this function until we have time to
do it correctly.

4. We could spell `log` (the natural logarithm) `ln`. This would resolve some ambiguity for
users with a background in certain fields, at the cost of diverging from the vast majority
of programming languages. Rust and Kotlin do spell it this way, so we wouldn't be completely
alone. It would also avoid using a function name that potentially conflicts (visually or
syntactically) with an obvious name for logging facilities. However, depending on font,
`ln` is easily confused with `in`, and it breaks the similarity with the other `log` functions.
As an assistance, we will add `ln` in the `Math` module but mark it unavailable, referring
users to `log`.

5. We could put the free functions into a separate module instead of the standard library.
Having them in a separate module helps avoid adding stuff to the global namespace
unless you're actually using it, which is generally nice, and the precedent from other
languages is pretty strong here: `#include <cmath>`, `import math`, etc. However, 
the general utility of having these functions available in the global namespace is felt
to outweigh this.

6. We could define an operator like `^` or `**` for one or both definitions of `pow`. I
have opted to keep new operators out of this proposal, in the interests of focusing on
the functions and their implementation hooks. I would consider such an operator to be an
additive change to be considered in a separate proposal.

7. Add the constants `pi` and `e` to `T.Math`. There's a bit of a question about how to
handle these with hypothetical arbitrary-precision types, but that's not a great reason
to keep them out for the concrete types we already have. Plus we already have `pi` on
FloatingPoint, so someone implementing such a type already needs to make a decision about
how to handle it. There's a second question of how to handle these with `Complex` or
`SIMD` types; one solution would be to only define them for types conforming to `Real`.

## Changes from previous revisions
1. A number of functions (`atan2`, `erf`, `erfc`, `gamma`, `logGamma`) have been moved
from `ElementaryFunctions` onto `Real`. The rationale for this is threefold: `atan2` never
made much sense for non-real arguments. Implementations of `erf`, `erfc`, `gamma` and
`logGamma` are not available on all platforms. Finally, those four functions are not actually
"elementary", so the naming is more accurate following this change.

2. `hypot` has been added to `Real`. We would like to have a more general solution for
efficiently rescaling computations in the future, but `hypot` is a tool that people can use
today.

3. I have dropped the `.Math` pseudo-namespace associatedtype from the protocols.
In earlier versions of the proposal, the static functions were spelled `Float.Math.sin(x)`.
This was motivated by a desire to avoid "magic" underscore-prefixed implementation
trampolines, while still grouping these functions together under a single `.Math` in
autocompletion lists. Whatever stylistic benefits this might have were judged to be not
worth the extra layer of machinery that would be fixed into the ABI even if we get a "real"
namespace mechanism at some future point. These functions now appear simply as
`Float.sin(x)`.
