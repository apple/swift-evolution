# Adding `isPower(of:)` to `BinaryInteger`

* Proposal: [SE-0288](0288-binaryinteger-ispower.md)
* Author: [Ding Ye](https://github.com/dingobye)
* Review Manager: [Joe Groff](https://github.com/jckarter)
* Status: **Previewing**
* Implementation: [apple/swift#24766](https://github.com/apple/swift/pull/24766)
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-se-0288-adding-ispower-of-to-binaryinteger/40325)

## Introduction

Checking some mathematical properties of integers (e.g. parity, divisibility, etc.) is widely used in scientific and engineering applications.  Swift brings a lot of convenience when performing such checks, thanks to the relevant methods (e.g. `isMultiple(of:)`) provided by the standard library.  However there are still some other cases not yet supported.  One of those useful checks that are currently missing is to tell if an integer is power of another, of which the implementation is non-trivial.  Apart from inconvenience, user-implemented code can bring inefficiency, poor readability, and even incorrectness.  To address this problem, this proposal would like to add a public API `isPower(of:)`, as an extension method, to the `BinaryInteger` protocol.

Swift-evolution thread: [Pitch](https://forums.swift.org/t/adding-ispowerof2-to-binaryinteger/24087)


## Motivation

Checking whether an integer is power of a given base is a typical integral query in a wide range of applications, and is especially common when the base value is two.  A question about [How to check if a number is a power of 2](https://stackoverflow.com/questions/600293/how-to-check-if-a-number-is-a-power-of-2) on Stack Overflow grows for a decade with ~200K views.  Since there are public demands for such functionality, it has been or will be officially supported by the libraries of many popular languages, such as [C++20](https://en.cppreference.com/w/cpp/numeric/ispow2), [D Programming Language](https://dlang.org/library/std/math/is_power_of2.html), and [.NET Framework](https://docs.microsoft.com/en-us/dotnet/api/system.numerics.biginteger.ispoweroftwo?view=netframework-4.8).

Swift, as a general-purpose programming language, is also experiencing such demands covering a variety of domains, including numerical and mathematical libraries (e.g. [#1](https://github.com/jsbean/ArithmeticTools/blob/cb6dae327baf53cdf614d26e630833efa00eda3f/ArithmeticTools/IntegerExtensions.swift#L48), [#2](https://github.com/Orbifold/XAct/blob/9acad78e5571aa93fb52d88f539459effab1d5f7/XAct/Numbers.swift#L49), [#3](https://github.com/donald-pinckney/SwiftNum/blob/b92e3b964268ebf62d99f488fcdf438574974f0d/Sources/SignalProcessing/IntExtensions.swift#L12), [#4](https://github.com/dn-m/Math/blob/d1284e043377c0b924cba2ffa2ab0b9aa9dd246f/Sources/Math/IntegerExtensions.swift#L48)), programming language and compiler components (e.g. [#1](https://github.com/kai-language/kai/blob/41268660a01e0d6d1f0ac8de743d91700707135e/Sources/Core/Helpers/Helpers.swift#L205), [#2](https://github.com/llvm-swift/LLVMSwift/blob/162f1632e017349b17146e33c5905f88148e55f1/Sources/LLVM/Units.swift#L125)), image/video manipulation (e.g. [#1](https://github.com/schulz0r/CoreImage-HDR/blob/c7f5264929338beebfcfbf2e420594aa2952ef6c/CoreImage-HDR/Convenience/extensions.swift#L20), [#2](https://github.com/OpsLabJPL/MarsImagesIOS/blob/f2109f38b31bf1ad2e7b5aae6916da07d2d7d08e/MarsImagesIOS/Math.swift#L17), [#3](https://github.com/chingf/CarmenaCamera/blob/bfa928ca1595770c3b99bb8aa95dc5340a0f3284/VideoCapture/Common.swift#L22), [#4](https://github.com/dehancer/IMProcessing/blob/7a7d48edb7ceeb2635219c8139aa6fb8dbf1525d/IMProcessing/Classes/Common/IMPNumericTypes.swift#L132)), and other applications and utilities such as [geography kit](https://github.com/tokyovigilante/CesiumKit/blob/7983bd742a85982d9c3303cdacc039dcb44c8a42/CesiumKit/Core/Math.swift#L597), [quantum circuit simulator](https://github.com/indisoluble/SwiftQuantumComputing/blob/008e82e0f38792372df1a428884cccb74c2732b3/SwiftQuantumComputing/Extension/Int%2BIsPowerOfTwo.swift#L24), [tournament](https://github.com/eaheckert/Tournament/blob/c09c6b3634da9b2666b8b1f8990ff62bdc4fd625/Tournament/Tournament/ViewControllers/TCreateTournamentVC.swift#L215), [blockchain](https://github.com/yeeth/BeaconChain.swift/blob/954bcb6e47b51f90eff16818719320a228afe891/Sources/BeaconChain/Extensions/Int.swift#L5), [3D-engine](https://github.com/xuzhao-nick/3DEngine/blob/c3aab94f2bce5e29f7988b0d7c1e075d74076ad7/3DEngine/3DEngine/MoreMath.swift#L50), etc.

Apart from the *is-power-of-two* usage, queries on *non-2* bases may also be practical, though they are much less common. In signal processing, for example, the efficient radix-4 algorithms can be applied when the FFT size is a power of 4.

As a result, it would be beneficial if we could have such an API supported in the standard library.  To be more specific, it is an extension method in the form of `isPower(of:)` to the `BinaryInteger` protocol, checking whether the `BinaryInteger` *self* is a power of the base specified by the parameter.  Let us discuss the impacts in the following aspects:

### Readability

A classic approach to check if an integer `n` is power of two is to test whether the condition `n > 0 && n & (n - 1) == 0` holds. Although such code is efficient, its underlying functionality is not intuitive to many people. As a result, programmers would have to put additional information somewhere (e.g. in the comments nearby) to make it clear.

Below is an example when making an assertion that `bucketCount` is power of two.  The classic approach is applied, followed by an additional error message for necessary clarification purpose.  However, if we had the proposed API available, the code would become more fluent and concise.
```swift
// Example (1) - apple/swift/stdlib/public/core/HashTable.swift
internal struct _HashTable {
    internal init(words: UnsafeMutablePointer<Word>, bucketCount: Int) {
        _internalInvariant(bucketCount > 0 && bucketCount & (bucketCount - 1) == 0,
          "bucketCount must be a power of two")

//      _internalInvariant(bucketCount.isPower(of: 2)) --- proposed solution

        ...
    }
}
```

### Efficiency

The user-implemented code may be less performant (e.g. [#1](https://github.com/OpsLabJPL/MarsImagesIOS/blob/f2109f38b31bf1ad2e7b5aae6916da07d2d7d08e/MarsImagesIOS/Math.swift#L17), [#2](https://github.com/eaheckert/Tournament/blob/c09c6b3634da9b2666b8b1f8990ff62bdc4fd625/Tournament/Tournament/ViewControllers/TCreateTournamentVC.swift#L215)),  since some developers are not aware of the classic approach as described above.

The example below shows a controversial approach, which employs the `llvm.ctpop` intrinsic to count the number of 1 bits. It can be expensive when the hardware does not have relevant `popcount` instruction support.
```swift
// Example (2) - apple/swift/stdlib/public/core/Integers.swift
extension BinaryInteger {
    internal func _description(radix: Int, uppercase: Bool) -> String {
        // Bit shifting can be faster than division when `radix` is a power of two
        let isRadixPowerOfTwo = radix.nonzeroBitCount == 1
        ...
    }
}
```

### Abstraction

Some developers, especially those unfamiliar to the integer type hierarchy, may have their own implementation targeting inappropriate types.

The following example presents very similar implementation individually for `UInt` and `Int`.  Such code duplication could be avoided if it targeted `BinaryInteger`.
```swift
// Example (3) - apple/swift/stdlib/public/core/Misc.swift
func _isPowerOf2(_ x: UInt) -> Bool {
    // implementation
}

func _isPowerOf2(_ x: Int) -> Bool {
    // implementation very similar to above
}
```

### Discoverability

IDEs can aid discoverability by suggesting `isPower(of:)` as part of autocompletion on integer types.  Notably, it works as a companion to other existing integral query APIs (e.g. `isMultiple(of:)`), and they can help improve the discoverability of each other.


## Proposed solution

Our solution is to introduce a public API `isPower(of:)`, as an extension method, to the `BinaryInteger` protocol.  It provides a standard implementation, which can be adopted by any type that conforms to this protocol.  With regard to semantics, it returns `true` iff `self` is a power of the input `base`.  To be more specific, it holds when (1) `self` is one (i.e., any base to the zero power), or (2) `self` equals the product of one or more `base`s.  Note that this API sits at the `BinaryInteger` protocol level, and it is expected to properly handle negative integers when the type `Self` is signed.

```swift
// In the standard library
extension BinaryInteger {
    @inlinable public func isPower(of base: Self) -> Bool {
        // implementation described in Detailed Design section
    }
}

// In user code
let x: Int = Int.random(in: 0000..<0288)
1.isPower(of: x)      // 'true' since x^0 == 1
1000.isPower(of: 10)  // 'true' since 10^3 == 1000
(-1).isPower(of: 1)   // 'false'
(-32).isPower(of: -2) // 'true' since (-2)^5 == -32
```


## Detailed design

A reference implementation can be found in [pull request #24766](https://github.com/apple/swift/pull/24766).  

### Overall Design

To make the API efficient for most use cases, the implementation is based on a fast-/slow-path pattern. We presents a generic implementation, which is suitable for all inputs, as the slow path; and meanwhile provide some particularly optimized implementation for frequently used inputs (e.g. 2) individually as the fast paths.  The high-level solution is illustrated below.

```swift
extension BinaryInteger {
  @inlinable public func isPower(of base: Self) -> Bool {
    // Fast path when base is one of the common cases.
    if base == common_base_A { return self._isPowerOfCommonBaseA }
    if base == common_base_B { return self._isPowerOfCommonBaseB }
    if base == common_base_C { return self._isPowerOfCommonBaseC }
    ...
    // Slow path for other bases.
    return self._slowIsPower(of: base)
  }

  @inlinable internal var _isPowerOfCommonBaseA: Bool { /* optimized implementation */ }
  @inlinable internal var _isPowerOfCommonBaseB: Bool { /* optimized implementation */ }
  @inlinable internal var _isPowerOfCommonBaseC: Bool { /* optimized implementation */ }
  ...
  @usableFromInline internal func _slowIsPower(of base: Self) ->  Bool { /* generic implementation */ }
}
```
Calling the public API `isPower(of: commonBaseK)` is expected to be as performant as directly calling the optimized version `_isPowerOfCommonBaseK`, if argument `commonBaseK` is a constant and the type `Self` is obvious enough (e.g. built-in integers) to the compiler to apply **constant-folding** to the `base == commonBaseK` expression, followed by a **simplify-cfg** transformation.

### Fast path when base is two

As for this fast path, it is **not** recommended to directly apply the classic approach to any type that conforms to `BinaryInteger` like this:
```swift
extension BinaryInteger {
  @inlinable internal var _isPowerOfTwo: Bool {
    return self > 0 && self & (self - 1) == 0
  }
}
```
Because when `Self` is some complicated type, the arithmetic, bitwise and comparison operations can be expensive and thus lead to poor performance.  In this case, the `BinaryInteger.words`-based implementation below is preferred, where word is supported by the hardware and the operations are expected to be efficient.
```swift
extension BinaryInteger {
  @inlinable internal var _isPowerOfTwo: Bool {
    let words = self.words
    guard !words.isEmpty else { return false }

    // If the value is represented in a single word, perform the classic check.
    if words.count == 1 {
      return self > 0 && self & (self - 1) == 0
    }

    // Return false if it is negative.  Here we only need to check the most
    // significant word (i.e. the last element of `words`).
    if Self.isSigned && Int(bitPattern: words.last!) < 0 {
      return false
    }

    // Check if there is exactly one non-zero word and it is a power of two.
    var found = false
    for word in words {
      if word != 0 {
        if found || word & (word - 1) != 0 { return false }
        found = true
      }
    }
    return found
  }
}
```

### Fast path when base itself is a power of two

As long as we have `_isPowerOfTwo` available, it becomes easy to implement such fast path when the base itself is power of two.  It takes advantages of the existing APIs `isMultiple(of:)` and `trailingZeroBitCount`. The code can be written as below.

```swift
extension BinaryInteger {
  @inlinable internal func _isPowerOf(powerOfTwo base: Self) -> Bool {
    _precondition(base._isPowerOfTwo)
    guard self._isPowerOfTwo else { return false }
    return self.trailingZeroBitCount.isMultiple(of: base.trailingZeroBitCount)
  }
}
```

### Fast path when base is ten

Unfortunately, there isn't any known super efficient algorithm to test if an integer is power of ten.  Some optimizations are discussed in the forum to help boost the performance to some extent.  Since the optimizations are conceptually non-trivial, they are not described here. Please refer to the [pitch thread](https://forums.swift.org/t/adding-ispowerof2-to-binaryinteger/24087/38) for details.


### Slow path

The slow path is generic for any input base, and the algorithm is standard. It handles some corner cases when `base.magnitude <= 1` in the first place; and then it repeatedly multiplies `base` to see if it can be equal to `self` at some point.  Attentions are required to avoid overflow of the multiplications.

```swift
extension BinaryInteger {
  @usableFromInline internal func _slowIsPower(of base: Self) -> Bool {
    // If self is 1 (i.e. any base to the zero power), return true.
    if self == 1 { return true }

    // Here if base is 0, 1 or -1, return true iff self equals base.
    if base.magnitude <= 1 { return self == base }

    // At this point, we have base.magnitude >= 2. Repeatedly perform
    // multiplication by a factor of base, and check if it can equal self.
    let (bound, remainder) = self.quotientAndRemainder(dividingBy: base)
    guard remainder == 0 else { return false }
    var x: Self = 1
    while x.magnitude < bound.magnitude { x *= base }
    return x == bound
  }
}
```

## Source compatibility

The proposed solution is an additive change.

## Effect on ABI stability

The proposed solution is an additive change.


## Effect on API resilience

The proposed solution is an additive change.


## Alternatives considered

### `isPower(of:)` as a protocol requirement

Making the public API as a protocol requirement instead of an extension methods reserves future flexibility.  It can give users a chance to provide their own optimization on some custom types where the default implementation is inefficient.  However, it would increase the interface complexity of the heavily-used `BinaryInteger` protocol, so it may not be worthy.

###  `isPowerOfTwo` instead of `isPower(of:)`

In fact, [the pitch](https://forums.swift.org/t/adding-ispowerof2-to-binaryinteger/24087) originally intended to add an API to check if an integer is power of two only.  However, the more generic form `isPower(of:)` is favored by the community.  This case is similar to [SE-0225](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0225-binaryinteger-iseven-isodd-ismultiple.md), which initially proposed `isEven` and `isOdd` as well, but ended up with `isMultiple(of:)` accepted only.

### Choices for fast paths

In the Detailed Design section, there are three specific cases of fast paths presented. Of course, we can have other choices.  The fast path for base two is essential, since it is the most common case in real world applications and should always be kept as a fast path.  We may remove the fast path for base ten if it is considered introducing too much complexity to the standard library.

### Alternative optimizations for `_isPowerOfTwo`

Apart from the classic approach, there are other optimization schemes to check if an integer is power of two.  Two candidates are `ctpop`-based and `cttz`-based approaches, whose implementation is shown below.

```swift
extension FixedWidthInteger {
  internal var _isPowerOfTwo_ctpop: Bool {
    return self > 0 && self.nonzeroBitCount == 1
  }
}

extension BinaryInteger {
  internal var _isPowerOfTwo_cttz: Bool {
    return (self > 0) && (self == (1 as Self) << self.trailingZeroBitCount)
  }
}
```
As per [the experimental results](https://github.com/apple/swift/pull/24766#issuecomment-492237146), they are overall less performant than the proposed solution on the types and platforms tested.  In addition, the `ctpop`-based approach narrows down the scope from `BinaryInteger` to `FixedWidthInteger`.

## Acknowledgments

This proposal has been greatly improved by the community. Below are some cases of significant help.

- Steve Canon followed the pitch all the way through and provided a lot of valuable comments to keep the work on the right track.
- Jens Persson showed some inspiration in [an earlier thread](https://forums.swift.org/t/even-and-odd-integers/11774/117), and discussed some use cases as well as its extendability to floating point types.
- Nevin Brackett-Rozinsky gave many details to optimize the implementation, and discovered [SR-10657](https://bugs.swift.org/browse/SR-10657) during the discussion.
- Michel Fortin provided an efficient solution to the fast path for base 10 (i.e. checking if an integer is power of ten), together with thorough explanation.
- Jordan Rose gave prompt and continued comments on the PR, and advised the API should better be an extension method rather than a protocol requirement.
- Erik Strottmann suggested a more appropriate naming `_isPowerOfTwo` instead of `_isPowerOf2`.
- Antoine Coeur had valuable discussions.
