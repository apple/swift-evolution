# Protocol-oriented integers

* Proposal: [SE-0104](0104-improved-integers.md)
* Authors: [Dave Abrahams](https://github.com/dabrahams), [Maxim Moiseev](https://github.com/moiseev)
* Review Manager: [Ben Cohen](https://github.com/airspeedswift)
* Status: **Implemented (Swift 4.0)**
* Bug: [SR-3196](https://bugs.swift.org/browse/SR-3196)
* Previous Revisions: 
  [1](https://github.com/swiftlang/swift-evolution/blob/0440700fc555a6c72abb4af807c8b79fb1bec592/proposals/0104-improved-integers.md), 
  [2](https://github.com/swiftlang/swift-evolution/blob/957ab545e05adb94507792e7871b38e34b56a0a5/proposals/0104-improved-integers.md),
  [3](https://github.com/swiftlang/swift-evolution/blob/80f57a6b7645126fe0220dcb91c19565e447d5d8/proposals/0104-improved-integers.md)
* Discussion on swift-evolution: [here](https://forums.swift.org/t/protocol-oriented-integers-take-2/4884).
* Decision notes: [Rationale](https://forums.swift.org/t/accepted-se-0104-protocol-oriented-integers/5346)


## Introduction

This proposal cleans up Swifts integer APIs and makes them more useful for
generic programming.

The language has evolved in ways that affect integers APIs since the time the
original proposal was approved for Swift 3. We also attempted to implement
the proposed model in the standard library and found that some essential APIs
were missing, whereas others could be safely removed.

Major changes to the APIs introduced by this revision are listed in a
[dedicated section](#whats-new-in-this-revision).

## Motivation

Swift's integer protocols don't currently provide a suitable basis for generic
programming. See [this blog post](http://blog.krzyzanowskim.com/2015/03/01/swift_madness_of_generic_integer/)
for an example of an attempt to implement a generic algorithm over integers.

The way the `IntegerArithmetic` protocol is defined, it does not generalize to
floating point numbers and also slows down compilation by requiring every
concrete type to provide an implementation of arithmetic operators, thus
polluting the overload set.

Converting from one integer type to another is performed using the concept of
the 'maximum width integer' (see `MaxInt`), which is an artificial limitation.
The very existence of `MaxInt` makes it unclear what to do should someone
implement `Int256`, for example.

Another annoying problem is the inability to use integers of different types in
comparison and bit-shift operations. For example, the following snippets won't
compile:

```Swift
var x: Int8 = 42
let y = 1
let z = 0

x <<= y   // error: binary operator '<<=' cannot be applied to operands of type 'Int8' and 'Int'
if x > z { ... }  // error: binary operator '>' cannot be applied to operands of type 'Int8' and 'Int'
```

Currently, bit-shifting a negative number of (or too many) bits causes a trap
on some platforms, which makes low-level bit manipulations needlessly dangerous
and unpredictable.

Finally, the current design predates many of the improvements that came since
Swift 1, and hasn't been revised since then.

## Proposed solution

We propose a new model that does not have above mentioned problems and is
more easily extensible.

~~~~
                +-------------+   +-------------+
        +------>+   Numeric   |   | Comparable  |
        |       |   (+,-,*)   |   | (==,<,>,...)|
        |       +------------++   +---+---------+
        |                     ^       ^
+-------+------------+        |       |
|    SignedNumeric   |      +-+-------+-----------+
|     (unary -)      |      |    BinaryInteger    |
+------+-------------+      |(words,%,bitwise,...)|
       ^                    ++---+-----+----------+
       |         +-----------^   ^     ^---------------+
       |         |               |                     |
+------+---------++    +---------+---------------+  +--+----------------+
|  SignedInteger  |    |  FixedWidthInteger      |  |  UnsignedInteger  |
|                 |    |(endianness,overflow,...)|  |                   |
+---------------+-+    +-+--------------------+--+  +-+-----------------+
                ^        ^                    ^       ^
                |        |                    |       |
                |        |                    |       |
               ++--------+-+                +-+-------+-+
               |Int family |-+              |UInt family|-+
               +-----------+ |              +-----------+ |
                 +-----------+                +-----------+
~~~~


There are several benefits provided by this model over the old one:

- It allows mixing integer types in generic functions.

  The possibility to initialize instances of any concrete integer type with
  values of any other concrete integer type enables writing functions that
  operate on more than one type conforming to `BinaryInteger`, such as
  heterogeneous comparisons or bit shifts, described later.

- It removes the overload resolution overhead.

  Arithmetic and bitwise operations can now be defined as generic operators on
  protocols. This approach significantly reduces the number of overloads for
  those operations, which used to be defined for every single concrete integer
  type.

- It enables protocol sharing between integer and floating point types.

  Note the exclusion of the `%` operation from `Numeric`. Its behavior for
  floating point numbers is sufficiently different from the one for integers
  that using it in generic context would lead to confusion. The `FloatingPoint`
  protocol introduced by [SE-0067](0067-floating-point-protocols.md) should now
  refine `SignedNumeric`.

- It makes future extensions possible.

  The proposed model eliminates the 'largest integer type' concept previously
  used to interoperate between integer types (see `toIntMax` in the current
  model) and instead provides access to machine words. It also introduces the
  `multipliedFullWidth(by:)`, `dividingFullWidth(_:)`, and
  `quotientAndRemainder` methods. Together these changes can be used to provide
  an efficient implementation of bignums that would be hard to achieve
  otherwise.

The implementation of proposed model in the standard library is available
[in the new-integer-protocols branch][impl].

### A note on bit shifts

This proposal introduces the concepts of *smart shifts* and *masking shifts*.

The semantics of shift operations are
[often undefined](http://llvm.org/docs/LangRef.html#bitwise-binary-operations)
in under- or over-shift cases. *Smart shifts*, implemented by `>>` and `<<`,
are designed to address this problem and always behave in a well defined way,
as shown in the examples below:

- `x << -2` is equivalent to `x >> 2`

- `(1 as UInt8) >> 42` will evaluate to `0`

- `(-128 as Int8) >> 42` will evaluate to `0xff` or `-1`

In most scenarios, the right hand operand is a literal constant, and branches
for handling under- and over-shift cases can be optimized away.  For other
cases, this proposal provides *masking shifts*, implemented by `&>>` and `&<<`.
A masking shift logically preprocesses the right hand operand by masking its
bits to produce a value in the range `0...(x-1)` where `x` is the number of bits
in the left hand operand. On most architectures this masking is already
performed by the CPU's shift instructions and has no cost. Both kinds of shift
avoid undefined behavior and produce uniform semantics across architectures.


## Detailed design

### What's new in this revision

* Shift operators were reorganized
  ([pull request](https://github.com/apple/swift/pull/11044))
  
    - Masking shift operators were moved from `BinaryInteger` to
      `FixedWidthInteger`.
    - Non-masking shift operators were moved from `FixedWidthInteger` to
      `BinaryInteger`.

  **Rationale:** attempts to implement masking shifts for
  arbitrary-precision integers have failed because
  the
  [semantics aren't clear](https://gist.github.com/xwu/d68baefaae9e9291d2e65bd12ad51be2#semantics-of-masking-shifts-for-arbitrary-precision-integers).
  Attempts to clarify the definition of the semantics of masking shift
  for `BinaryInteger` have failed, indicating that the operation
  doesn't actually make sense outside of `FixedWidthInteger`.
  
* `ArithmeticOverflow` was removed in favor of using a simple `Bool`.

   **Rationale:** the enum
   [proves to have poor ergonomics](https://gist.github.com/xwu/d68baefaae9e9291d2e65bd12ad51be2#arithmeticoverflow).
   It only appears as part of a result tuple, where a label already
   helps counteract the readability deficit usually caused by 
   returning an un-labeled `Bool`.
   
* `BinaryInteger`'s initializers from floating point values were
  changed from:

    ```swift
    init?<T : FloatingPoint>(exactly source: T)
    init<T : FloatingPoint>(_ source: T)
    ```
    
  to:
  
    ```swift
    init?<T : BinaryFloatingPoint>(exactly source: T)
    init<T : BinaryFloatingPoint>(_ source: T)
    ```

  **Rationale:** Attempts to implement these initializers for
  arbitrary models of `FloatingPoint` have failed
  (see
  [here](https://github.com/apple/swift/blob/826f8daf4a25657965f65cbb7343e751c76fe2e1/stdlib/public/core/DoubleWidth.swift.gyb#L100-L106) and
  [here](https://github.com/apple/swift/blob/826f8daf4a25657965f65cbb7343e751c76fe2e1/test/Prototypes/BigInt.swift#L145-L151))
  whereas they are
  clearly
  [implementable](https://github.com/apple/swift/blob/826f8daf4a25657965f65cbb7343e751c76fe2e1/stdlib/public/core/DoubleWidth.swift.gyb#L108-L159) for
  models of `BinaryFloatingPoint`, suggesting that the operation
  doesn't actually make sense outside of `BinaryFloatingPoint`.
  
* `BinaryInteger`'s `init(extendingOrTruncating:)` was renamed to
  `init(truncatingIfNeeded:)`
  
  **Rationale:** `extendingOrTruncating` emphasizes a part of the
  semantics (“extending”) that is lossless and thus doesn't warrant
  the implied warning of an argument label.  The two use cases for
  this initializer are intentional truncation and optimizing out range
  checks that are known by the programmer to be un-needed.  Both these
  cases are better served by `truncatingIfNeeded`.
  
### Protocols

#### `Numeric`

The `Numeric` protocol declares binary arithmetic operators – such as `+`,
`-`, and `*` — and their mutating counterparts.

It provides a suitable basis for arithmetic on scalars such as integers and
floating point numbers.

Both mutating and non-mutating operations are declared in the protocol, however
only the mutating ones are required, as default implementations of the
non-mutating ones are provided by a protocol extension.

The `Magnitude` associated type is able to hold the absolute value of any
possible value of `Self`. Concrete types do not have to provide a type alias for
it, as it can be inferred from the `magnitude` property. This property can
be useful in operations that are simpler to implement in terms of unsigned
values, for example, printing a value of an integer, which is just printing a
'-' character in front of an absolute value.

Please note that for ordinary work, the `magnitude` property **should not**
be preferred to the `abs(_)` function, whose return value is of the same type
as its argument.


```Swift
public protocol Numeric : Equatable, ExpressibleByIntegerLiteral {
  /// Creates a new instance from the given integer, if it can be represented
  /// exactly.
  ///
  /// If the value passed as `source` is not representable exactly, the result
  /// is `nil`. In the following example, the constant `x` is successfully
  /// created from a value of `100`, while the attempt to initialize the
  /// constant `y` from `1_000` fails because the `Int8` type can represent
  /// `127` at maximum:
  ///
  ///     let x = Int8(exactly: 100)
  ///     // x == Optional(100)
  ///     let y = Int8(exactly: 1_000)
  ///     // y == nil
  ///
  /// - Parameter source: A BinaryInteger value to convert to an integer.
  init?<T : BinaryInteger>(exactly source: T)

  /// A type that can represent the absolute value of any possible value of the
  /// conforming type.
  associatedtype Magnitude : Numeric, Comparable

  /// The magnitude of this value.
  ///
  /// For any numeric value `x`, `x.magnitude` is the absolute value of `x`.
  /// You can use the `magnitude` property in operations that are simpler to
  /// implement in terms of unsigned values, such as printing the value of an
  /// integer, which is just printing a '-' character in front of an absolute
  /// value.
  ///
  ///     let x = -200
  ///     // x.magnitude == 200
  ///
  /// The global `abs(_:)` function provides more familiar syntax when you need
  /// to find an absolute value. In addition, because `abs(_:)` always returns
  /// a value of the same type, even in a generic context, using the function
  /// instead of the `magnitude` property is encouraged.
  ///
  /// - SeeAlso: `abs(_:)`
  var magnitude: Magnitude { get }

  /// Returns the sum of the two given values.
  ///
  /// The sum of `lhs` and `rhs` must be representable in the same type. In the
  /// following example, the result of `100 + 200` is greater than the maximum
  /// representable `Int8` value:
  ///
  ///     let x: Int8 = 10 + 21
  ///     // x == 31
  ///     let y: Int8 = 100 + 121
  ///     // Overflow error
  static func +(_ lhs: Self, _ rhs: Self) -> Self

  /// Adds the given value to this value in place.
  ///
  /// For example:
  ///
  ///     var x = 15
  ///     x += 7
  ///     // x == 22
  static func +=(_ lhs: inout Self, rhs: Self)

  /// Returns the difference of the two given values.
  ///
  /// The difference of `lhs` and `rhs` must be representable in the same type.
  /// In the following example, the result of `10 - 21` is less than zero, the
  /// minimum representable `UInt` value:
  ///
  ///     let x: UInt = 21 - 10
  ///     // x == 11
  ///     let y: UInt = 10 - 21
  ///     // Overflow error
  static func -(_ lhs: Self, _ rhs: Self) -> Self

  /// Subtracts the given value from this value in place.
  ///
  /// For example:
  ///
  ///     var x = 15
  ///     x -= 7
  ///     // x == 8
  static func -=(_ lhs: inout Self, rhs: Self)

  /// Returns the product of the two given values.
  ///
  /// The product of `lhs` and `rhs` must be representable in the same type. In
  /// the following example, the result of `10 * 50` is greater than the
  /// maximum representable `Int8` value.
  ///
  ///     let x: Int8 = 10 * 5
  ///     // x == 50
  ///     let y: Int8 = 10 * 50
  ///     // Overflow error
  static func *(_ lhs: Self, _ rhs: Self) -> Self

  /// Multiples this value by the given value in place.
  ///
  /// For example:
  ///
  ///     var x = 15
  ///     x *= 7
  ///     // x == 105
  static func *=(_ lhs: inout Self, rhs: Self)
}

extension Numeric {
  public static prefix func + (x: Self) -> Self {
    return x
  }
}
```

#### `SignedNumeric`

The `SignedNumeric` protocol is for numbers that can be negated.

```Swift
public protocol SignedNumeric : Numeric {
  /// Returns the additive inverse of this value.
  ///
  ///     let x = 21
  ///     let y = -x
  ///     // y == -21
  ///
  /// - Returns: The additive inverse of this value.
  ///
  /// - SeeAlso: `negate()`
  static prefix func - (_ operand: Self) -> Self

  /// Replaces this value with its additive inverse.
  ///
  /// The following example uses the `negate()` method to negate the value of
  /// an integer `x`:
  ///
  ///     var x = 21
  ///     x.negate()
  ///     // x == -21
  ///
  /// - SeeAlso: The unary minus operator (`-`).
  mutating func negate()
}

extension SignedNumeric {
  public static prefix func - (_ operand: Self) -> Self {
    var result = operand
    result.negate()
    return result
  }

  public mutating func negate() {
    self = 0 - self
  }
}
```

#### `BinaryInteger`

The `BinaryInteger` protocol is the basis for all the integer types provided by
the standard library.

This protocol adds a few new initializers. Two of them allow to create integers
from floating point numbers, others support construction from instances of any
type conforming to `BinaryInteger`, using different strategies:

  - Initialize `Self` with the value, provided that the value is representable.
    The precondition should be satisfied by the caller.

  - Extend or truncate the value to fit into `Self`

  - Clamp the value to the representable range of `Self`

`BinaryInteger` also declares bitwise and shift operators.

```Swift
public protocol BinaryInteger :
  Comparable, Hashable, Numeric, CustomStringConvertible, Strideable {

  associatedtype Words : Collection // where Iterator.Element == UInt

  /// A Boolean value indicating whether this type is a signed integer type.
  ///
  /// *Signed* integer types can represent both positive and negative values.
  /// *Unsigned* integer types can represent only nonnegative values.
  static var isSigned: Bool { get }

  /// Creates an integer from the given floating-point value, if it can be
  /// represented exactly.
  ///
  /// If the value passed as `source` is not representable exactly, the result
  /// is `nil`. In the following example, the constant `x` is successfully
  /// created from a value of `21.0`, while the attempt to initialize the
  /// constant `y` from `21.5` fails:
  ///
  ///     let x = Int(exactly: 21.0)
  ///     // x == Optional(21)
  ///     let y = Int(exactly: 21.5)
  ///     // y == nil
  ///
  /// - Parameter source: A floating-point value to convert to an integer.
  init?<T : BinaryFloatingPoint>(exactly source: T)

  /// Creates an integer from the given floating-point value, truncating any
  /// fractional part.
  ///
  /// Truncating the fractional part of `source` is equivalent to rounding
  /// toward zero.
  ///
  ///     let x = Int(21.5)
  ///     // x == 21
  ///     let y = Int(-21.5)
  ///     // y == -21
  ///
  /// If `source` is outside the bounds of this type after truncation, a
  /// runtime error may occur.
  ///
  ///     let z = UInt(-21.5)
  ///     // Error: ...the result would be less than UInt.min
  ///
  /// - Parameter source: A floating-point value to convert to an integer.
  ///   `source` must be representable in this type after truncation.
  init<T : BinaryFloatingPoint>(_ source: T)

  /// Creates an new instance from the given integer.
  ///
  /// If the value passed as `source` is not representable in this type, a
  /// runtime error may occur.
  ///
  ///     let x = -500 as Int
  ///     let y = Int32(x)
  ///     // y == -500
  ///
  ///     // -500 is not representable as a 'UInt32' instance
  ///     let z = UInt32(x)
  ///     // Error
  ///
  /// - Parameter source: An integer to convert. `source` must be representable
  ///   in this type.
  init<T : BinaryInteger>(_ source: T)

  /// Creates a new instance from the bit pattern of the given instance by
  /// sign-extending or truncating to fit this type.
  ///
  /// When the bit width of `T` (the type of `source`) is equal to or greater
  /// than this type's bit width, the result is the truncated
  /// least-significant bits of `source`. For example, when converting a
  /// 16-bit value to an 8-bit type, only the lower 8 bits of `source` are
  /// used.
  ///
  ///     let p: Int16 = -500
  ///     // 'p' has a binary representation of 11111110_00001100
  ///     let q = Int8(truncatingIfNeeded: p)
  ///     // q == 12
  ///     // 'q' has a binary representation of 00001100
  ///
  /// When the bit width of `T` is less than this type's bit width, the result
  /// is *sign-extended* to fill the remaining bits. That is, if `source` is
  /// negative, the result is padded with ones; otherwise, the result is
  /// padded with zeros.
  ///
  ///     let u: Int8 = 21
  ///     // 'u' has a binary representation of 00010101
  ///     let v = Int16(truncatingIfNeeded: u)
  ///     // v == 21
  ///     // 'v' has a binary representation of 00000000_00010101
  ///
  ///     let w: Int8 = -21
  ///     // 'w' has a binary representation of 11101011
  ///     let x = Int16(truncatingIfNeeded: w)
  ///     // x == -21
  ///     // 'x' has a binary representation of 11111111_11101011
  ///     let y = UInt16(truncatingIfNeeded: w)
  ///     // y == 65515
  ///     // 'y' has a binary representation of 11111111_11101011
  ///
  /// - Parameter source: An integer to convert to this type.
  init<T : BinaryInteger>(truncatingIfNeeded source: T)

  /// Creates a new instance with the representable value that's closest to the
  /// given integer.
  ///
  /// If the value passed as `source` is greater than the maximum representable
  /// value in this type, the result is the type's `max` value. If `source` is
  /// less than the smallest representable value in this type, the result is
  /// the type's `min` value.
  ///
  /// In this example, `x` is initialized as an `Int8` instance by clamping
  /// `500` to the range `-128...127`, and `y` is initialized as a `UInt`
  /// instance by clamping `-500` to the range `0...UInt.max`.
  ///
  ///     let x = Int8(clamping: 500)
  ///     // x == 127
  ///     // x == Int8.max
  ///
  ///     let y = UInt(clamping: -500)
  ///     // y == 0
  ///
  /// - Parameter source: An integer to convert to this type.
  init<T : BinaryInteger>(clamping source: T)

  /// The collection of words in two's complement representation of the value,
  /// from the least significant to most.
  var words: Words { get }

  /// The number of bits in the current binary representation of this value.
  ///
  /// This property is a constant for instances of fixed-width integer
  /// types.
  var bitWidth : Int { get }

  /// The number of trailing zeros in this value's binary representation.
  ///
  /// For example, in a fixed-width integer type with a `bitWidth` value of 8,
  /// the number -8 has three trailing zeros.
  ///
  ///     let x = Int8(bitPattern: 0b1111_1000)
  ///     // x == -8
  ///     // x.trailingZeroBits == 3
  var trailingZeroBits: Int { get }


  /// Returns the quotient of dividing the first value by the second.
  ///
  /// For integer types, any remainder of the division is discarded.
  ///
  ///     let x = 21 / 5
  ///     // x == 4
  static func /(_ lhs: Self, _ rhs: Self) -> Self

  /// Divides this value by the given value in place.
  ///
  /// For example:
  ///
  ///     var x = 15
  ///     x /= 7
  ///     // x == 2
  static func /=(_ lhs: inout Self, rhs: Self)

  /// Returns the remainder of dividing the first value by the second.
  ///
  /// The result has the same sign as `lhs` and is less than `rhs.magnitude`.
  ///
  ///     let x = 22 % 5
  ///     // x == 2
  ///     let y = 22 % -5
  ///     // y == 2
  ///     let z = -22 % -5
  ///     // z == -2
  ///
  /// - Parameters:
  ///   - lhs: The value to divide.
  ///   - rhs: The value to divide `lhs` by. `rhs` must not be zero.
  static func %(_ lhs: Self, _ rhs: Self) -> Self

  /// Replaces this value with the remainder of itself divided by the given
  /// value. For example:
  ///
  ///     var x = 15
  ///     x %= 7
  ///     // x == 1
  ///
  /// - Parameter rhs: The value to divide this value by. `rhs` must not be
  ///   zero.
  ///
  /// - SeeAlso: `remainder(dividingBy:)`
  static func %=(_ lhs: inout Self, _ rhs: Self)

  /// Returns the inverse of the bits set in the argument.
  ///
  /// The bitwise NOT operator (`~`) is a prefix operator that returns a value
  /// in which all the bits of its argument are flipped: Bits that are `1` in
  /// the argument are `0` in the result, and bits that are `0` in the argument
  /// are `1` in the result. This is equivalent to the inverse of a set. For
  /// example:
  ///
  ///     let x: UInt8 = 5        // 0b00000101
  ///     let notX = ~x           // 0b11111010
  ///
  /// Performing a bitwise NOT operation on 0 returns a value with every bit
  /// set to `1`.
  ///
  ///     let allOnes = ~UInt8.min   // 0b11111111
  ///
  /// - Complexity: O(1).
  static prefix func ~ (_ x: Self) -> Self

  /// Returns the result of performing a bitwise AND operation on this value
  /// and the given value.
  ///
  /// A bitwise AND operation results in a value that has each bit set to `1`
  /// where *both* of its arguments have that bit set to `1`. For example:
  ///
  ///     let x: UInt8 = 5          // 0b00000101
  ///     let y: UInt8 = 14         // 0b00001110
  ///     let z = x & y             // 0b00000100
  static func &(_ lhs: Self, _ rhs: Self) -> Self
  static func &=(_ lhs: inout Self, _ rhs: Self)

  /// Returns the result of performing a bitwise OR operation on this value and
  /// the given value.
  ///
  /// A bitwise OR operation results in a value that has each bit set to `1`
  /// where *one or both* of its arguments have that bit set to `1`. For
  /// example:
  ///
  ///     let x: UInt8 = 5          // 0b00000101
  ///     let y: UInt8 = 14         // 0b00001110
  ///     let z = x | y             // 0b00001111
  static func |(_ lhs: Self, _ rhs: Self) -> Self
  static func |=(_ lhs: inout Self, _ rhs: Self)

  /// Returns the result of performing a bitwise XOR operation on this value
  /// and the given value.
  ///
  /// A bitwise XOR operation, also known as an exclusive OR operation, results
  /// in a value that has each bit set to `1` where *one or the other but not
  /// both* of its arguments had that bit set to `1`. For example:
  ///
  ///     let x: UInt8 = 5          // 0b00000101
  ///     let y: UInt8 = 14         // 0b00001110
  ///     let z = x ^ y             // 0b00001011
  static func ^(_ lhs: Self, _ rhs: Self) -> Self
  static func ^=(_ lhs: inout Self, _ rhs: Self)

  /// Returns the result of shifting this value's binary representation the
  /// specified number of digits to the right.
  static func >><RHS: BinaryInteger>(_ lhs: Self, _ rhs: RHS) -> Self

  /// Stores the result of shifting a value's binary representation the
  /// specified number of digits to the right in the left-hand-side variable.
  static func >>=<RHS: BinaryInteger>(_ lhs: inout Self, _ rhs: RHS)

  /// Returns the result of shifting a value's binary representation the
  /// specified number of digits to the left.
  static func << <RHS: BinaryInteger>(_ lhs: Self, _ rhs: RHS) -> Self

  /// Stores the result of shifting a value's binary representation the
  /// specified number of digits to the left in the left-hand-side variable
  static func <<= <RHS: BinaryInteger>(_ lhs: inout Self, _ rhs: RHS)

  /// Returns the quotient and remainder of this value divided by the given
  /// value.
  ///
  /// Use this method to calculate the quotient and remainder of a division at
  /// the same time.
  ///
  ///     let x = 1_000_000
  ///     let (q, r) = x.quotientAndRemainder(dividingBy: 933)
  ///     // q == 1071
  ///     // r == 757
  ///
  /// - Parameter rhs: The value to divide this value by.
  /// - Returns: A tuple containing the quotient and remainder of this value
  ///   divided by `rhs`.
  func quotientAndRemainder(dividingBy rhs: Self)
    -> (quotient: Self, remainder: Self)

  /// Returns `-1` if this value is negative and `1` if it's positive;
  /// otherwise, `0`.
  ///
  /// - Returns: The sign of this number, expressed as an integer of the same
  ///   type.
  func signum() -> Self
}

extension BinaryInteger {
  init() { self = 0 }
}
```

#### `FixedWidthInteger`

The `FixedWidthInteger` protocol adds the notion of endianness as well as static
properties for type bounds and bit width.

The `ReportingOverflow` family of methods is used in default implementations of
mutating arithmetic methods (see the `Numeric` protocol). Having these
methods allows the library to provide both bounds-checked and masking
implementations of arithmetic operations, without duplicating code.

The `multipliedFullWidth(by:)` and `dividingFullWidth(_:)` methods are
necessary building blocks to implement support for integer types of a greater
width such as arbitrary-precision integers.

```Swift

public protocol FixedWidthInteger : BinaryInteger {
  /// The number of bits used for the underlying binary representation of
  /// values of this type.
  ///
  /// An unsigned, fixed-width integer type can represent values from 0 through
  /// `(2 ** bitWidth) - 1`, where `**` is exponentiation. A signed,
  /// fixed-width integer type can represent values from
  /// `-(2 ** bitWidth - 1)` through `(2 ** bitWidth - 1) - 1`. For example,
  /// the `Int8` type has a `bitWidth` value of 8 and can store any integer in
  /// the range `-128...127`.
  static var bitWidth : Int { get }

  /// The maximum representable integer in this type.
  ///
  /// For unsigned integer types, this value is `(2 ** bitWidth) - 1`, where
  /// `**` is exponentiation. For signed integer types, this value is
  /// `(2 ** bitWidth - 1) - 1`.
  static var max: Self { get }

  /// The minimum representable value.
  ///
  /// For unsigned integer types, this value is always `0`. For signed integer
  /// types, this value is `-(2 ** bitWidth - 1)`, where `**` is
  /// exponentiation.
  static var min: Self { get }

  /// Returns the sum of this value and the given value along with a flag
  /// indicating whether overflow occurred in the operation.
  ///
  /// - Parameter other: The value to add to this value.
  /// - Returns: A tuple containing the result of the addition along with a
  ///   flag indicating whether overflow occurred. If the `overflow` component
  ///   is `.none`, the `partialValue` component contains the entire sum. If
  ///   the `overflow` component is `.overflow`, an overflow occurred and the
  ///   `partialValue` component contains the truncated sum of this value and
  ///   `other`.
  ///
  /// - SeeAlso: `+`
  func addingReportingOverflow(_ other: Self)
    -> (partialValue: Self, overflow: Bool)

  /// Returns the difference of this value and the given value along with a
  /// flag indicating whether overflow occurred in the operation.
  ///
  /// - Parameter other: The value to subtract from this value.
  /// - Returns: A tuple containing the result of the subtraction along with a
  ///   flag indicating whether overflow occurred. If the `overflow` component
  ///   is `.none`, the `partialValue` component contains the entire
  ///   difference. If the `overflow` component is `.overflow`, an overflow
  ///   occurred and the `partialValue` component contains the truncated
  ///   result of `other` subtracted from this value.
  ///
  /// - SeeAlso: `-`
  func subtractingReportingOverflow(_ other: Self)
    -> (partialValue: Self, overflow: Bool)

  /// Returns the product of this value and the given value along with a flag
  /// indicating whether overflow occurred in the operation.
  ///
  /// - Parameter other: The value to multiply by this value.
  /// - Returns: A tuple containing the result of the multiplication along with
  ///   a flag indicating whether overflow occurred. If the `overflow`
  ///   component is `.none`, the `partialValue` component contains the entire
  ///   product. If the `overflow` component is `.overflow`, an overflow
  ///   occurred and the `partialValue` component contains the truncated
  ///   product of this value and `other`.
  ///
  /// - SeeAlso: `*`, `multipliedFullWidth(by:)`
  func multipliedReportingOverflow(by other: Self)
    -> (partialValue: Self, overflow: Bool)

  /// Returns the quotient of dividing this value by the given value along with
  /// a flag indicating whether overflow occurred in the operation.
  ///
  /// For a value `x`, if zero is passed as `other`, the result is
  /// `(x, .overflow)`.
  ///
  /// - Parameter other: The value to divide this value by.
  /// - Returns: A tuple containing the result of the division along with a
  ///   flag indicating whether overflow occurred. If the `overflow` component
  ///   is `.none`, the `partialValue` component contains the entire quotient.
  ///   If the `overflow` component is `.overflow`, an overflow occurred and
  ///   the `partialValue` component contains the truncated quotient.
  ///
  /// - SeeAlso: `/`, `dividingFullWidth(_:)`
  func dividedReportingOverflow(by other: Self)
    -> (partialValue: Self, overflow: Bool)

  /// Returns a double-width value containing the high and low parts of the
  /// result of multiplying this value by an argument.
  ///
  /// Use this method to calculate the full result of a product that would
  /// otherwise overflow. Unlike traditional truncating multiplication, the
  /// `multipliedFullWidth(by:)` method returns both the high and low
  /// parts of the product of `self` and `other`. The following example uses
  /// this method to multiply two `UInt8` values that normally overflow when
  /// multiplied:
  ///
  ///     let x: UInt8 = 100
  ///     let y: UInt8 = 20
  ///     let result = x.multipliedFullWidth(by: y)
  ///     // result.high == 0b00000111
  ///     // result.low  == 0b11010000
  ///
  /// The product of `x` and `y` is 2000, which is too large to represent in a
  /// `UInt8` instance. The `high` and `low` components of the `result`
  /// represent 2000 when concatenated to form a double-width integer; that
  /// is, using `result.high` as the high byte and `result.low` as the low byte
  /// of a `UInt16` instance.
  ///
  ///     let z = UInt16(result.high) << 8 | UInt16(result.low)
  ///     // z == 2000
  ///
  /// - Parameters:
  ///   - other: A value to multiplied `self` by.
  /// - Returns: A tuple containing the high and low parts of the result of
  ///   multiplying `self` and `other`.
  ///
  /// - SeeAlso: `multipliedReportingOverflow(by:)`
  func multipliedFullWidth(by other: Self) -> DoubleWidth<Self>

  /// Returns a tuple containing the quotient and remainder of dividing the
  /// first argument by this value.
  ///
  /// The resulting quotient must be representable within the bounds of the
  /// type. If the quotient of dividing `lhs` by `self` is too large to
  /// represent in the type, a runtime error may occur.
  ///
  /// - Parameters:
  ///   - lhs: A value containing the high and low parts of a double-width
  ///     integer. The `high` component of the tuple carries the sign, if the
  ///     type is signed.
  /// - Returns: A tuple containing the quotient and remainder of `lhs` divided
  ///   by `self`.
  func dividingFullWidth(_ lhs: DoubleWidth<Self>)
    -> (quotient: Self, remainder: Self)

  /// The number of bits equal to 1 in this value's binary representation.
  ///
  /// For example, in a fixed-width integer type with a `bitWidth` value of 8,
  /// the number 31 has five bits equal to 1.
  ///
  ///     let x: Int8 = 0b0001_1111
  ///     // x == 31
  ///     // x.populationCount == 5
  var populationCount: Int { get }

  /// The number of leading zeros in this value's binary representation.
  ///
  /// For example, in a fixed-width integer type with a `bitWidth` value of 8,
  /// the number 31 has three leading zeros.
  ///
  ///     let x: Int8 = 0b0001_1111
  ///     // x == 31
  ///     // x.leadingZeroBits == 3
  /// - SeeAlso: `BinaryInteger.trailingZeroBits`
  var leadingZeroBits: Int { get }

  /// Creates an integer from its big-endian representation, changing the
  /// byte order if necessary.
  init(bigEndian value: Self)

  /// Creates an integer from its little-endian representation, changing the
  /// byte order if necessary.
  init(littleEndian value: Self)

  /// The big-endian representation of this integer.
  ///
  /// If necessary, the byte order of this value is reversed from the typical
  /// byte order of this integer type. On a big-endian platform, for any
  /// integer `x`, `x == x.bigEndian`.
  ///
  /// - SeeAlso: `littleEndian`
  var bigEndian: Self { get }

  /// The little-endian representation of this integer.
  ///
  /// If necessary, the byte order of this value is reversed from the typical
  /// byte order of this integer type. On a little-endian platform, for any
  /// integer `x`, `x == x.littleEndian`.
  ///
  /// - SeeAlso: `bigEndian`
  var littleEndian: Self { get }

  /// A representation of this integer with the byte order swapped.
  var byteSwapped: Self { get }


  /// Returns the result of shifting a value's binary representation the
  /// specified number of digits to the right, masking the shift amount to the
  /// type's bit width.
  static func &>>(_ lhs: Self, _ rhs: Self) -> Self

  /// Calculates the result of shifting a value's binary representation the
  /// specified number of digits to the right, masking the shift amount to the
  /// type's bit width, and stores the result in the left-hand-side variable.
  static func &>>=(_ lhs: inout Self, _ rhs: Self)
  
  /// Returns the result of shifting a value's binary representation the
  /// specified number of digits to the left, masking the shift amount to the
  /// type's bit width.
  static func &<<(_ lhs: Self, _ rhs: Self) -> Self

  /// Returns the result of shifting a value's binary representation the
  /// specified number of digits to the left, masking the shift amount to the
  /// type's bit width, and stores the result in the left-hand-side variable.
  static func &<<=(_ lhs: inout Self, _ rhs: Self)
}
```

#### Auxiliary protocols

```Swift
public protocol UnsignedInteger : BinaryInteger {
  associatedtype Magnitude : BinaryInteger
}
public protocol SignedInteger : BinaryInteger, SignedNumeric {
  associatedtype Magnitude : BinaryInteger
}
```

### DoubleWidth\<T\>

The `DoubleWidth<T>` type allows to create wider fixed-width integer types from
the ones available in the standard library.

Standard library currently provides fixed-width integer types of up to 64 bits.
A value of `DoubleWidth<Int64>` will double the range of the underlying type and
implement all the `FixedWidthInteger` requirements. _Please note_ though that
the implementation will not necessarily be the most efficient one, so it would
not be a good idea to use `DoubleWidth<Int32>` instead of a built-in `Int64`.

```swift
public enum DoubleWidth<T : FixedWidthInteger> {
  case .parts(high: T, low: T.Magnitude)

  public var high: T { get }
  public var low: T.Magnitude { get }
}
```

Representing it as an `enum` instead of a simple struct allows to use it both
as a single value, as well as in destructuring matches.

```swift
let high = doubleWidthValue.high
let low = doubleWidthValue.low

// or

case let (high, low) = doubleWidthValue
```


### Extra operators

In addition to the operators described in the [protocols section](#protocols),
we also provide a few extensions:

#### Non-mutating homogeneous shifts

```Swift
extension FixedWidthInteger {
  public static func &>> (lhs: Self, rhs: Self) -> Self
  public static func &<< (lhs: Self, rhs: Self) -> Self
```

#### Heterogeneous shifts

```Swift
extension BinaryInteger {
  // 'Smart' shifts
  static func >>  <Other : BinaryInteger>(lhs: Self, rhs: Other) -> Self
  static func >>= <Other : BinaryInteger>(lhs: inout Self, rhs: Other)
  static func <<  <Other : BinaryInteger>(lhs: Self, rhs: Other) -> Self
  static func <<= <Other : BinaryInteger>(lhs: inout Self, rhs: Other)
}
extension FixedWidthInteger {
  public static func &>> <Other : BinaryInteger>(lhs: Self, rhs: Other) -> Self
  public static func &>>= <Other : BinaryInteger>(lhs: inout Self, rhs: Other)
  public static func &<< <Other : BinaryInteger>(lhs: Self, rhs: Other) -> Self
  public static func &<<= <Other : BinaryInteger>(lhs: inout Self, rhs: Other)
}
```

#### Heterogeneous equality and comparison

```Swift
extension BinaryInteger {
  // Equality
  static func == <Other : BinaryInteger>(lhs: Self, rhs: Other) -> Bool
  static func != <Other : BinaryInteger>(lhs: Self, rhs: Other) -> Bool

  // Comparison
  static func <  <Other : BinaryInteger>(lhs: Self, rhs: Other) -> Bool
  static func <= <Other : BinaryInteger>(lhs: Self, rhs: Other) -> Bool
  static func >  <Other : BinaryInteger>(lhs: Self, rhs: Other) -> Bool
  static func >= <Other : BinaryInteger>(lhs: Self, rhs: Other) -> Bool
}
```

#### Masking arithmetic

```Swift
public func &* <T: FixedWidthInteger>(lhs: T, rhs: T) -> T
public func &- <T: FixedWidthInteger>(lhs: T, rhs: T) -> T
public func &+ <T: FixedWidthInteger>(lhs: T, rhs: T) -> T
```


## Non-goals

This proposal:

- *DOES NOT* solve the integer promotion problem, which would allow mixed-type
  arithmetic. However, we believe that it is an important step in the right
  direction.

- *DOES NOT* include the implementation of a `BigInt` type, but allows it
  to be implemented in the future.


## Source compatibility

The proposed change is designed to be as non-breaking as possible, and it has
been proven that it does not break code on concrete integer types. However,
there are still a few API breaking changes in the realm of generic code:

* Integer protocols in Swift up to and including version 3 were not particularly
useful for generic programming, but were rather a means of sharing
implementation between conforming types. Therefore we believe the amount of code
that relied on these protocols is relatively small. The breakage can be further
reduced by introducing proper aliases for the removed protocols with deprecation
warnings.

* Deprecation of the `BitwiseOperations` protocol. We find it hard to imagine a
type that conforms to this protocol, but is *not* a binary integer type.

* Addition of 'smart' shifts will change the behavior of existing code. It will
still compile, but will be potentially less performant due to extra logic
involved. In a case, where this becomes a problem, newly introduced masking
shift operators can be used instead. Unfortunately, performance characteristics
of the code cannot be statically checked, and thus migration cannot be provided.


[se91]: 0091-improving-operators-in-protocols.md
[impl]: https://github.com/apple/swift/tree/new-integer-protocols
