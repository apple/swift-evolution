# Enhanced Floating Point Protocols

* Proposal: [SE-0067](0067-floating-point-protocols.md)
* Author: [Stephen Canon](https://github.com/stephentyrone)
* Review Manager: [Chris Lattner](https://github.com/lattner)
* Status: **Implemented (Swift 3.0)**
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-se-0067-enhanced-floating-point-protocols/2420)
* Implementation: [apple/swift#2453](https://github.com/apple/swift/pull/2453)
* Previous Revision: [1](https://github.com/swiftlang/swift-evolution/blob/fb1368a6a5474f57aa8f1846b5355d18753098f3/proposals/0067-floating-point-protocols.md)

## Introduction

The current FloatingPoint protocol is quite limited, and provides only a small
subset of the features expected of an IEEE 754 conforming type.  This proposal
expands the protocol to cover most of the expected basic operations, and adds
a second protocol, BinaryFloatingPoint, that provides a number of useful tools
for generic programming with the most commonly used types.

Revision 2 reintroduces some of the details of the `Arithmetic` protocol from
earlier drafts of this proposal, but as methods in the `FloatingPoint` API,
with the goal of reducing the number of overloads for basic operations.  This
change was requested by some members of the core language team.

Revision 2 also incorporates a number of suggestions from the review list and
corrects some typos; thanks especially to Xiaodi Wu for thoughtful feedback.
Consult the changelog at the end of this document for full details.

* [Proposal draft](https://forums.swift.org/t/proposal-draft-enhanced-floating-point-protocols/2201)
* [Review #1](https://forums.swift.org/t/review-se-0067-enhanced-floating-point-protocols/2264)
* [Review #2](https://forums.swift.org/t/review-2-se-0067-enhanced-floating-point-protocols/2339)

## Motivation

Beside the high-level motivation provided by the introduction, the proposed
prototypes address a number of pain points that we've heard about from
developers:

- FloatingPoint should conform to Equatable, and Comparable
- FloatingPoint should conform to FloatLiteralConvertible
- Deprecate the `%` operator for floating-point types
- Provide basic constants (analogues of C's DBL_MAX, etc.)
- Make Float80 conform to FloatingPoint

## Detailed design

The `FloatingPoint` protocol is split into two parts; `FloatingPoint` and
`BinaryFloatingPoint`, which conforms to `FloatingPoint`.  If decimal
types were added at some future point, they would conform to
`DecimalFloatingPoint`.

`FloatingPoint` is expanded to contain most of the IEEE 754 basic
operations, as well as conformance to `Comparable` (which implies `Equatable`),
and `IntegerLiteralConvertible` (`BinaryFloatingPoint` includes conformance
to `FloatLiteralConvertible`).

```swift
/// A floating-point type that provides most of the IEEE 754 basic (clause 5)
/// operations.  The base, precision, and exponent range are not fixed in
/// any way by this protocol, but it enforces the basic requirements of
/// any IEEE 754 floating-point type.
///
/// The BinaryFloatingPoint protocol refines these requirements, adds some
/// additional operations that only make sense for a fixed radix, and also
/// provides default implementations of some of the FloatingPoint APIs.
public protocol FloatingPoint: Comparable, IntegerLiteralConvertible, SignedNumber {

  /// An integer type that can represent any written exponent.
  associatedtype Exponent: SignedInteger

  /// Initialize to zero
  init()

  /// Initialize from sign, exponent, and significand.
  ///
  /// The result is:
  ///
  /// ~~~
  /// (sign == .Minus ? -1 : 1) * significand * radix**exponent
  /// ~~~
  ///
  /// (where `**` is exponentiation) computed as if by a single correctly-
  /// rounded floating-point operation.  If this value is outside the
  /// representable range of the type, overflow or underflow occurs, and zero,
  /// a subnormal value, or infinity may result, as with any basic operation.
  /// Other edge cases:
  ///
  /// - If `significand` is zero or infinite, the result is zero or infinite,
  ///   regardless of the value of `exponent`.
  ///
  /// - If `significand` is NaN, the result is NaN.
  ///
  /// Note that for any floating-point `x` the result of
  ///
  ///   `Self(sign: x.sign,
  ///         exponent: x.exponent,
  ///         significand: x.significand)`
  ///
  /// is "the same" as `x`; it is `x` canonicalized.
  ///
  /// This initializer implements the IEEE 754 `scaleB` operation.
  init(sign: Sign, exponent: Exponent, significand: Self)

  /// A floating point value whose exponent and signficand are taken from
  /// `magnitude` and whose sign is taken from `signOf`.  Implements the
  /// IEEE 754 `copysign` operation.
  init(signOf: Self, magnitudeOf other: Self)

  //  NOTE: --------------------------------------------------------------------
  //  The next two APIs are not implementable without a revised integer
  //  protocol.  Nonetheless, I believe that it makes sense to consider them
  //  with the rest of this proposal, with the understanding that they will
  //  be implemented when it becomes possible to do so.

  /// The closest representable value to the argument.
  init<Source: Integer>(_ value: Source)

  /// Fails if the argument cannot be exactly represented.
  init?<Source: Integer>(exactly value: Source)
  //  --------------------------------------------------------------------------

  /// 2 for binary floating-point types, 10 for decimal.
  ///
  /// A conforming type may use any integer radix, but values other than
  /// 2 or 10 are extraordinarily rare in practice.
  static var radix: Int { get }

  /// A quiet NaN (not-a-number).  Compares not equal to every value,
  /// including itself.
  static var nan: Self { get }

  /// A signaling NaN (not-a-number).
  ///
  /// The default IEEE 754 behavior of operations involving a signaling NaN
  /// is to raise the Invalid flag in the floating-point environment and
  /// return a quiet NaN.  Operations on types conforming to FloatingPoint
  /// should support this behavior, but they might also support other options;
  /// for example, it would be reasonable to implement alternative operations
  /// in which operating on a signaling NaN is a `fatalError()` or results in
  /// a diagnostic for debugging purposes.
  static var signalingNaN: Self { get }

  /// Positive infinity.  Compares greater than all finite numbers.
  static var infinity: Self { get }

  /// The greatest finite number.
  ///
  /// Compares greater than or equal to all finite numbers, but less than
  /// infinity.  Corresponds to the C macros `FLT_MAX`, `DBL_MAX`, etc.
  /// The naming of those macros is slightly misleading, because infinity
  /// is greater than this value.
  static var greatestFiniteMagnitude: Self { get }

  /// The mathematical constant π = 3.14159...
  ///
  /// Extensible floating-point types might provide additional APIs to obtain
  /// this value to caller-specified precision.
  static var pi: Self { get }

  // NOTE: Rationale for "ulp" instead of "epsilon":
  // We do not use that name because it is ambiguous at best and misleading
  // at worst:
  //
  // - Historically several definitions of "machine epsilon" have commonly
  //   been used, which differ by up to a factor of two or so.  By contrast
  //   "ulp" is a term with a specific unambiguous definition.
  //
  // - Some languages have used "epsilon" to refer to wildly different values,
  //   such as `leastNonzeroMagnitude`.
  //
  // - Inexperienced users often believe that "epsilon" should be used as a
  //   tolerance for floating-point comparisons, because of the name.  It is
  //   nearly always the wrong value to use for this purpose.

  /// The unit in the last place of `self`.
  ///
  /// This is the unit of the least significant digit in the significand of
  /// `self`.  For most numbers `x`, this is the difference between `x` and
  /// the next greater (in magnitude) representable number.  There are some
  /// edge cases to be aware of:
  ///
  /// - `greatestFiniteMagnitude.ulp` is a finite number, even though
  ///   the next greater representable value is `infinity`.
  /// - `x.ulp` is `NaN` if `x` is not a finite number.
  /// - If `x` is very small in magnitude, then `x.ulp` may be a subnormal
  ///   number.  On targets that do not support subnormals, `x.ulp` may be
  ///   flushed to zero.
  ///
  /// This quantity, or a related quantity is sometimes called "epsilon" or
  /// "machine epsilon".  We avoid that name because it has different meanings
  /// in different languages, which can lead to confusion, and because it
  /// suggests that it is a good tolerance to use for comparisons,
  /// which is almost never is.
  ///
  /// (See https://en.wikipedia.org/wiki/Machine_epsilon for more detail)
  var ulp: Self { get }

  /// The unit in the last place of 1.0.
  ///
  /// The positive difference between 1.0 and the next greater representable
  /// number.  Corresponds to the C macros `FLT_EPSILON`, `DBL_EPSILON`, etc.
  static var ulpOfOne: Self { get }

  /// The least positive normal number.
  ///
  /// Compares less than or equal to all positive normal numbers.  There may
  /// be smaller positive numbers, but they are "subnormal", meaning that
  /// they are represented with less precision than normal numbers.
  /// Corresponds to the C macros `FLT_MIN`, `DBL_MIN`, etc.  The naming of
  /// those macros is slightly misleading, because subnormals, zeros, and
  /// negative numbers are smaller than this value.
  static var leastNormalMagnitude: Self { get }

  /// The least positive number.
  ///
  /// Compares less than or equal to all positive numbers, but greater than
  /// zero.  If the target supports subnormal values, this is smaller than
  /// `leastNormalMagnitude`; otherwise they are equal.
  static var leastNonzeroMagnitude: Self { get }

  /// `Minus` if the signbit of `self` is set, and `Plus` otherwise.
  /// Implements the IEEE 754 `signbit` operation.
  ///
  /// Note that the property `x.sign == .Minus` is not the same as `x < 0`.
  /// In particular, `x < 0` while `x.sign == .Minus` if `x` is -0, and while
  /// `x < 0` is always false if `x` is NaN, `x.sign` could be either `.Plus`
  /// or `.Minus`.
  var sign: Sign { get }

  /// The integer part of the base-r logarithm of the magnitude of `self`,
  /// where r is the radix (2 for binary, 10 for decimal).  Implements the
  /// IEEE 754 `logB` operation.
  ///
  /// Edge cases:
  ///
  /// - If `x` is zero, then `x.exponent` is `Int.min`.
  /// - If `x` is +/-infinity or NaN, then `x.exponent` is `Int.max`
  var exponent: Exponent { get }

  /// The significand satisfies:
  ///
  /// ~~~
  /// self = (sign == .Minus ? -1 : 1) * significand * radix**exponent
  /// ~~~
  ///
  /// (where `**` is exponentiation).  If radix is 2, then for finite non-zero
  /// numbers `1 <= significand` and `significand < 2`.  For other values of
  /// `x`, `x.significand` is defined as follows:
  ///
  /// - If `x` is zero, then `x.significand` is 0.0.
  /// - If `x` is infinity, then `x.significand` is 1.0.
  /// - If `x` is NaN, then `x.significand` is NaN.
  ///
  /// For all floating-point `x`, if we define y by:
  ///
  /// ~~~
  /// let y = Self(sign: x.sign, exponent: x.exponent,
  ///              significand: x.significand)
  /// ~~~
  ///
  /// then `y` is equivalent to `x`, meaning that `y` is `x` canonicalized.
  var significand: Self { get }

  /// Sum of `self` and `other` rounded to a representable value.  The IEEE
  /// 754 addition operation.
  ///
  /// A default implementation is provided in terms of `add()`.
  @warn_unused_result
  func adding(other: Self) -> Self

  /// Replace `self` with the sum of `self` and `other` rounded to a
  /// representable value.
  mutating func add(other: Self)

  /// Additive inverse of `self`.  Always exact.
  @warn_unused_result
  func negated() -> Self

  /// Replace `self` with its additive inverse.
  mutating func negate()

  /// Sum of `self` and the additive inverse of `other` rounded to a
  /// representable value.  The IEEE 754 subtraction operation.
  ///
  /// A default implementation is provided in terms of `subtract()`.
  @warn_unused_result
  func subtracting(other: Self) -> Self

  /// Replace `self` with the sum of `self` and the additive inverse of `other`
  /// rounded to a representable value.
  mutating func subtract(other: Self)

  /// Product of `self` and `other` rounded to a representable value.  The
  /// IEEE 754 multiply operation.
  ///
  /// A default implementation is provided in terms of `multiply(by:)`.
  @warn_unused_result
  func multiplied(by other: Self) -> Self

  /// Replace `self` with the product of `self` and `other` rounded to a
  /// representable value.
  mutating func multiply(by other: Self)

  /// Quotient of `self` and `other` rounded to a representable value.  The
  /// IEEE 754 divide operation.
  ///
  /// A default implementation is provided in terms of `divide(by:)`.
  @warn_unused_result
  func divided(by other: Self) -> Self

  /// Replace `self` with the quotient of `self` and `other` rounded to a
  /// representable value.
  mutating func divide(by other: Self)

  /// Remainder of `self` divided by `other`.  This is the IEEE 754 remainder
  /// operation.
  ///
  /// For finite `self` and `other`, the remainder `r` is defined by
  /// `r = self - other*n`, where `n` is the integer nearest to `self/other`.
  /// (Note that `n` is *not* `self/other` computed in floating-point
  /// arithmetic, and that `n` may not even be representable in any available
  /// integer type).  If `self/other` is exactly halfway between two integers,
  /// `n` is chosen to be even.
  ///
  /// It follows that if `self` and `other` are finite numbers, the remainder
  /// `r` satisfies `-|other|/2 <= r` and `r <= |other|/2`.
  ///
  /// `remainder` is always exact.
  @warn_unused_result
  func remainder(dividingBy other: Self) -> Self

  /// Mutating form of `remainder`.
  mutating func formRemainder(dividingBy other: Self)

  /// Remainder of `self` divided by `other` using truncating division.
  /// Equivalent to the C standard library function `fmod`.
  ///
  /// If `self` and `other` are finite numbers, the truncating remainder
  /// `r` has the same sign as `other` and is strictly smaller in magnitude.
  /// It satisfies `r = self - other*n`, where `n` is the integral part
  /// of `self/other`.
  ///
  /// `truncatingRemainder` is always exact.
  @warn_unused_result
  func truncatingRemainder(dividingBy other: Self) -> Self

  /// Mutating form of `truncatingRemainder`.
  mutating func formTruncatingRemainder(dividingBy other: Self)

  /// Square root of `self`.
  @warn_unused_result
  func squareRoot() -> Self

  /// Mutating form of square root.
  mutating func formSquareRoot()

  /// `self + lhs*rhs` computed without intermediate rounding.  Implements the
  /// IEEE 754 `fusedMultiplyAdd` operation.
  @warn_unused_result
  func addingProduct(lhs: Self, _ rhs: Self) -> Self

  /// Fused multiply-add, accumulating the product of `lhs` and `rhs` to `self`.
  mutating func addProduct(lhs: Self, _ rhs: Self)

  /// The minimum of `x` and `y`.  Implements the IEEE 754 `minNum` operation.
  ///
  /// Returns `x` if `x <= y`, `y` if `y < x`, and whichever of `x` or `y`
  /// is a number if the other is NaN.  The result is NaN only if both
  /// arguments are NaN.
  ///
  /// This function is an implementation hook to be used by the free function
  /// min(Self, Self) -> Self so that we get the IEEE 754 behavior with regard
  /// to NaNs.
  @warn_unused_result
  static func minimum(x: Self, _ y: Self) -> Self

  /// The maximum of `x` and `y`.  Implements the IEEE 754 `maxNum` operation.
  ///
  /// Returns `x` if `x >= y`, `y` if `y > x`, and whichever of `x` or `y`
  /// is a number if the other is NaN.  The result is NaN only if both
  /// arguments are NaN.
  ///
  /// This function is an implementation hook to be used by the free function
  /// max(Self, Self) -> Self so that we get the IEEE 754 behavior with regard
  /// to NaNs.
  @warn_unused_result
  static func maximum(x: Self, _ y: Self) -> Self

  /// Whichever of `x` or `y` has lesser magnitude.  Implements the IEEE 754
  /// `minNumMag` operation.
  ///
  /// Returns `x` if abs(x) <= abs(y), `y` if abs(y) < abs(x), and whichever of
  /// `x` or `y` is a number if the other is NaN.  The result is NaN
  /// only if both arguments are NaN.
  @warn_unused_result
  static func minimumMagnitude(x: Self, _ y: Self) -> Self

  /// Whichever of `x` or `y` has greater magnitude.  Implements the IEEE 754
  /// `maxNumMag` operation.
  ///
  /// Returns `x` if abs(x) >= abs(y), `y` if abs(y) > abs(x), and whichever of
  /// `x` or `y` is a number if the other is NaN.  The result is NaN
  /// only if both arguments are NaN.
  @warn_unused_result
  static func maximumMagnitude(x: Self, _ y: Self) -> Self

  /// The least representable value that compares greater than `self`.
  ///
  /// - If `x` is `-infinity`, then `x.nextUp` is `-greatestMagnitude`.
  /// - If `x` is `-leastNonzeroMagnitude`, then `x.nextUp` is `-0.0`.
  /// - If `x` is zero, then `x.nextUp` is `leastNonzeroMagnitude`.
  /// - If `x` is `greatestMagnitude`, then `x.nextUp` is `infinity`.
  /// - If `x` is `infinity` or `NaN`, then `x.nextUp` is `x`.
  var nextUp: Self { get }

  /// The greatest representable value that compares less than `self`.
  ///
  /// `x.nextDown` is equivalent to `-(-x).nextUp`
  var nextDown: Self { get }

  /// IEEE 754 equality predicate.
  ///
  /// -0 compares equal to +0, and NaN compares not equal to anything,
  /// including itself.
  @warn_unused_result
  func isEqual(to other: Self) -> Bool

  /// IEEE 754 less-than predicate.
  ///
  /// NaN compares not less than anything.  -infinity compares less than
  /// all values except for itself and NaN.  Everything except for NaN and
  /// +infinity compares less than +infinity.
  @warn_unused_result
  func isLess(than other: Self) -> Bool

  /// IEEE 754 less-than-or-equal predicate.
  ///
  /// NaN compares not less than or equal to anything, including itself.
  /// -infinity compares less than or equal to everything except NaN.
  /// Everything except NaN compares less than or equal to +infinity.
  ///
  /// Because of the existence of NaN in FloatingPoint types, trichotomy does
  /// not hold, which means that `x < y` and `!(y <= x)` are not equivalent.
  /// This is why `isLessThanOrEqual(to:)` is a separate implementation hook
  /// in the protocol.
  ///
  /// Note that this predicate does not impose a total order.  The
  /// `isTotallyOrdered` predicate refines this relation so that all values
  /// are totally ordered.
  @warn_unused_result
  func isLessThanOrEqualTo(_ other: Self) -> Bool

  /// True if and only if `self` precedes `other` in the IEEE 754 total order
  /// relation.
  ///
  /// This relation is a refinement of `<=` that provides a total order on all
  /// values of type `Self`, including non-canonical encodings, signed zeros,
  /// and NaNs.  Because it is used much less frequently than the usual
  /// comparisons, there is no operator form of this relation.
  @warn_unused_result
  func isTotallyOrdered(below other: Self) -> Bool

  /// True if and only if `self` is normal.
  ///
  /// A normal number uses the full precision available in the format.  Zero
  /// is not a normal number.
  var isNormal: Bool { get }

  /// True if and only if `self` is finite.
  ///
  /// If `x.isFinite` is `true`, then one of `x.isZero`, `x.isSubnormal`, or
  /// `x.isNormal` is also `true`, and `x.isInfinite` and `x.isNaN` are
  /// `false`.
  var isFinite: Bool { get }

  /// True iff `self` is zero.  Equivalent to `self == 0`.
  var isZero: Bool { get }

  /// True if and only if `self` is subnormal.
  ///
  /// A subnormal number does not use the full precision available to normal
  /// numbers of the same format.  Zero is not a subnormal number.
  ///
  /// Subnormal numbers are often called "denormal" or "denormalized".  These
  /// are simply different names for the same concept.  IEEE 754 prefers the
  /// name "subnormal", and we follow that usage.
  var isSubnormal: Bool { get }

  /// True if and only if `self` is infinite.
  ///
  /// Note that `isFinite` and `isInfinite` do not form a dichotomy, because
  /// they are not total.  If `x` is `NaN`, then both properties are `false`.
  var isInfinite: Bool { get }

  /// True if and only if `self` is NaN ("not a number"); this property is
  /// true for both quiet and signaling NaNs.
  var isNaN: Bool { get }

  /// True if and only if `self` is a signaling NaN.
  var isSignalingNaN: Bool { get }

  /// The IEEE 754 "class" of this type.
  var floatingPointClass: FloatingPointClassification { get }

  /// True if and only if `self` is canonical.
  ///
  /// Every floating-point value of type Float or Double is canonical, but
  /// non-canonical values of type Float80 exist, and non-canonical values
  /// may exist for other types that conform to FloatingPoint.
  ///
  /// The non-canonical Float80 values are known as "pseudo-denormal",
  /// "unnormal", "pseudo-infinity", and "pseudo-NaN".
  /// (https://en.wikipedia.org/wiki/Extended_precision#x86_Extended_Precision_Format)
  var isCanonical: Bool { get }
}

public enum Sign: Int {
  case Plus
  case Minus
}

public enum FloatingPointClassification {
  case signalingNaN
  case quietNaN
  case negativeInfinity
  case negativeNormal
  case negativeSubnormal
  case negativeZero
  case positiveZero
  case positiveSubnormal
  case positiveNormal
  case positiveInfinity
}
```

The usual arithmetic and comparison operators are defined in terms of the
method hooks from the protocol:

```swift
@warn_unused_result
prefix func +<T: FloatingPoint>(x: T) -> T {
  return x
}

@warn_unused_result
func +<T: FloatingPoint>(lhs: T, rhs: T) -> T {
  return lhs.adding(rhs)
}

func +=<T: FloatingPoint>(inout lhs: T, rhs: T) {
  lhs.add(rhs)
}

@warn_unused_result
prefix func -<T: FloatingPoint>(x: T) -> T {
  return x.negated()
}

@warn_unused_result
func -<T: FloatingPoint>(lhs: T, rhs: T) -> T {
  return lhs.subtracting(rhs)
}

func -=<T: FloatingPoint>(inout lhs: T, rhs: T) {
  lhs.subtract(rhs)
}

@warn_unused_result
func *<T: FloatingPoint>(lhs: T, rhs: T) -> T {
  return lhs.multiplied(by: rhs)
}

func *=<T: FloatingPoint>(inout lhs: T, rhs: T) {
  lhs.multiply(by: rhs)
}

@warn_unused_result
func /<T: FloatingPoint>(lhs: T, rhs: T) -> T {
  return lhs.divided(by: rhs)
}

func /=<T: FloatingPoint>(inout lhs: T, rhs: T) {
  lhs.divide(by: rhs)
}

func sqrt<T: FloatingPoint>(_ rhs: T) -> T {
  return rhs.squareRoot()
}

func ==<T: FloatingPoint>(lhs: T, rhs: T) -> Bool {
  return lhs.isEqual(to: rhs)
}

func < <T: FloatingPoint>(lhs: T, rhs: T) -> Bool {
  return lhs.isLess(than: rhs)
}

func <= <T: FloatingPoint>(lhs: T, rhs: T) -> Bool {
  return lhs.isLessThanOrEqualTo(rhs)
}

func > <T: FloatingPoint>(lhs: T, rhs: T) -> Bool {
  return rhs.isLess(than: lhs)
}

func >= <T: FloatingPoint>(lhs: T, rhs: T) -> Bool {
  return rhs.isLessThanOrEqualTo(lhs)
}
```

The `BinaryFloatingPoint` protocol provides a number of additional APIs
that only make sense for types with fixed radix 2:

```swift
/// A radix-2 (binary) floating-point type that follows the IEEE 754 encoding
/// conventions.
public protocol BinaryFloatingPoint: FloatingPoint, FloatLiteralConvertible {

  /// An unsigned integer type that can represent the significand of any value.
  ///
  /// The significand (http://en.wikipedia.org/wiki/Significand) is frequently
  /// also called the "mantissa", but this terminology is slightly incorrect
  /// (see the "Use of 'mantissa'" section on the linked Wikipedia page for
  /// more details).  "Significand" is the preferred terminology in IEEE 754.
  associatedtype RawSignificand: UnsignedInteger

  /// An unsigned integer type that can represent the exponent encoding of any
  /// value.
  associatedtype RawExponent: UnsignedInteger

  /// Combines `sign`, `exponent` and `significand` bit patterns to produce
  /// a floating-point value.
  init(sign: Sign,
       exponentBitPattern: RawExponent,
       significandBitPattern: RawSignificand)

  //  NOTE: --------------------------------------------------------------------
  //  The next two APIs are not implementable without a revised integer
  //  protocol.  Nonetheless, I believe that it makes sense to consider them
  //  with the rest of this proposal, with the understanding that they will
  //  be implemented when it becomes possible to do so.

  /// `value` rounded to the closest representable value.
  init<Source: BinaryFloatingPoint>(_ value: Source)

  /// Fails if `value` cannot be represented exactly as `Self`.
  init?<Source: BinaryFloatingPoint>(exactly value: Source)
  //  --------------------------------------------------------------------------

  /// The number of bits used to represent the exponent.
  ///
  /// Following IEEE 754 encoding convention, the exponent bias is:
  ///
  /// ~~~
  /// bias = 2**(exponentBitCount-1) - 1
  /// ~~~
  ///
  /// (where `**` is exponentiation).  The least normal exponent is `1-bias`
  /// and the largest finite exponent is `bias`.  The all-zeros exponent is
  /// reserved for subnormals and zeros, and the all-ones exponent is reserved
  /// for infinities and NaNs.
  static var exponentBitCount: Int { get }

  /// For fixed-width floating-point types, this is the number of fractional
  /// significand bits.
  ///
  /// For extensible floating-point types, `significandBitCount` should be
  /// the maximum allowed significand width (without counting any leading
  /// integral bit of the significand).  If there is no upper limit, then
  /// `significandBitCount` should be `Int.max`.
  ///
  /// Note that `Float80.significandBitCount` is 63, even though 64 bits
  /// are used to store the significand in the memory representation of a
  /// `Float80` (unlike other floating-point types, `Float80` explicitly
  /// stores the leading integral significand bit, but the
  /// `BinaryFloatingPoint` APIs provide an abstraction so that users don't
  /// need to be aware of this detail).
  static var significandBitCount: Int { get }

  /// The raw encoding of the exponent field of the floating-point value.
  var exponentBitPattern: RawExponent { get }

  /// The raw encoding of the significand field of the floating-point value.
  ///
  /// `significandBitPattern` does *not* include the leading integral bit of
  /// the significand, even for types like `Float80` that store it explicitly.
  var significandBitPattern: RawSignificand { get }

  /// The least-magnitude member of the binade of `self`.
  ///
  /// If `x` is `+/-significand * 2**exponent`, then `x.binade` is
  /// `+/- 2**exponent`; i.e. the floating point number with the same sign
  /// and exponent, but with a significand of 1.0.
  var binade: Self { get }

  /// The number of bits required to represent significand.
  ///
  /// If `self` is not a finite non-zero number, `significandWidth` is
  /// `-1`.  Otherwise, it is the number of fractional bits required to
  /// represent `self.significand`, which is an integer between zero and
  /// `significandBitCount`.  Some examples:
  ///
  /// - For any representable power of two, `significandWidth` is zero,
  ///   because `significand` is `1.0`.
  /// - If `x` is 10, then `x.significand` is `1.01` in binary, so
  ///   `x.significandWidth` is 2.
  /// - If `x` is Float.pi, `x.significand` is `1.10010010000111111011011`,
  ///   and `x.significandWidth` is 23.
  var significandWidth: Int { get }

  //  NOTE: --------------------------------------------------------------------
  //  These APIs are not implementable without the generic inits, which in turn
  //  depend on a revised Integer protocol.  Nonetheless, I believe that it
  //  makes sense to consider them with the rest of this proposal, with the
  //  understanding that they will be implemented when it becomes possible to
  //  do so.

  @warn_unused_result
  func isEqual<Other: BinaryFloatingPoint>(to other: Other) -> Bool

  @warn_unused_result
  func isLess<Other: BinaryFloatingPoint>(than other: Other) -> Bool

  @warn_unused_result
  func isLessThanOrEqual<Other: BinaryFloatingPoint>(to other: Other) -> Bool

  @warn_unused_result
  func isTotallyOrdered<Other: BinaryFloatingPoint>(below other: Other) -> Bool
  //  --------------------------------------------------------------------------
}
```

`Float`, `Double`, `Float80` and `CGFloat` conform to both of these protocols.

Additionally, an initializer will be added to each of those types to construct
a NaN with specified payload:

```swift
  /// NaN with specified `payload`.
  ///
  /// Compares not equal to every value, including itself.  Most operations
  /// with a NaN operand will produce a NaN result.
  init(nan payload: Self.RawSignificand, signaling: Bool)
```

This initializer is not attached to either of the protocols because it may not
actually make sense or have consistent behavior for all possible types
conforming to the protocols.

## Impact on existing code

1. The `%` operator is no longer available for FloatingPoint types.  We don't
believe that it was widely used correctly, and the operation is still available
via the `formTruncatingRemainder` method for people who need it.

2. To follow the naming guidelines, `NaN` and `isNaN` are replaced with `nan`
and `isNaN`.

3. The redundant property `quietNaN` is removed.

4. `isSignaling` is renamed `isSignalingNaN`.

## Changes from the draft proposal

1. Removed the `Arithmetic` protocol; it may be circulated again in the future
as an independent proposal, or as part of a new model for integers.

2. Removed the `add[ing]`, `subtract[ing]`, etc methods, which were hooks for
`Arithmetic`.  This proposal now includes only the existing operator forms.

3. Removed static `nan(payload: signaling:)` method from protocols.  This will
exist as an initializer on concrete types, but not be part of the protocol.

4. Added the static `signalingNaN` property to the protocol to make up for #3.

5. Added the static `pi` property in response to popular demand.

6. Renamed the static `ulp` property `ulpOfOne` to avoid ambiguity with the
member property `ulp`.

7. Renamed `totalOrder` to `isTotallyOrdered` to be consistent with the other
comparison methods.

8. Additional clarifications and comments.

## Changes from revision 1

1. Reinstated `add`, `subtract`, etc from the draft of `Arithmetic` as methods
on `FloatingPoint`.  Operators moved out of protocol, mapped to method hooks.

2. Moved the associated type `RawSignificand` from `FloatingPoint` to the
`BinaryFloatingPoint` protocol.

3. Additional documentation added to some operations.

4. `leastMagnitude` --> `leastNonzeroMagnitude`.

5. Removed `isUnordered(with:)`, renamed `isTotallyOrdered(with:)` argument
label to `below:`.

6. `Nan` --> `NaN`; `nan` stays the same, as suggested by Jordan Rose.

7. `isLessThanOrEqual(to:)` --> `isLessThanOrEqualTo(_:)`.  I'm still not
totally convinced that this is an improvement, so I'd love to hear further
suggestions on how to handle this case.

8. Added associated types `Exponent` and `RawExponent` to allow for soft-float
types with stupendously large range.

9. Fixed (some) typos.

## Implementation status

Everything that does not depend on [SE-0104](0104-improved-integers.md) is implemented.  Some related operations were added in [SE-0113](0113-rounding-functions-on-floatingpoint.md).
