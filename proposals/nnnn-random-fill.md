# An API for bulk random bytes

* Proposal: [SE-NNNN](nnnn-random-fill.md)
* Authors: [Stephen Canon](https://github.com/stephentyrone)
* Review Manager: TBD
* Status: **Pitched**
* Implementation: [apple/swift#63511](https://github.com/apple/swift/pull/63511)
* Discussion: [pitch](https://forums.swift.org/t/an-api-for-bulk-random-bytes/63051)

## Introduction

This proposal adds new API to the RandomNumberGenerator protocol that allows
it to produce more than 64b of randomness at a time.

Swift-evolution thread: [Discussion thread topic for that proposal](https://forums.swift.org/)

## Motivation

Sometimes you need more than 64b of randomness at once:
- to seed a deterministic random number generator with a state larger than 64b
- to sample a 64b integer from a range that isn't an exact power of two
- to generate many random integers (e.g. the indices for shuffling an array)
- ...

The current RandomNumberGenerator interface forces you to get exactly 64b of
randomness at a time; this is annoying and unergonomic, but more importantly
it is inefficient for SystemRandomNumberGenerator, which has some per-call
overhead on most platforms. By generating more than 64b at a time, we can make
some use cases 16x faster on macOS by amortizing that overhead, and see similar
speedups on other platforms.

## Proposed solution

Add the following method to the `RandomNumberGenerator` protocol:
```
mutating func fill(_ buffer: UnsafeMutableRawBufferPointer)
```
This allows you to get as many random bytes as you want at once.

## Detailed design

Besides adding the protocol requirement above, we will add a back-deployed
default implementation as a protocol extension. This implementation does not
provide the performance improvements, but allows code to use the new API 
without an availability check in concrete contexts.

We also add a concrete implementation to SystemRandomNumberGenerator that
does realize the performance benefits when used in non-generic contexts.

Non-stdlib types may want to provide their own custom implementation of 
`fill(:)`. For most RNGs other than the SystemRNG, the performance benefits 
will be  small, but they can still be significant for random sources that 
naturally produce more than 64b at a time, such as high-throughput counter-
based RNGs. 

_This proposal does not add any uses of the new `fill` method_, which raises
the question of how it can possibly be beneficial to performance. I will adopt
`fill` in the implementation of existing API, but those changes do not need
to go through the evolution process; this API is the building block that's
needed to make the other performance improvements possible.

## Source compatibility

There is no effect on Source compatibility.

## Effect on ABI stability

There is no effect on ABI stability.

## Effect on API resilience

A new customization point is added to RandomNumberGenerator with availability
annotations and a default implementation provided. The default implementation
is back-deployed, and is unconditionally available in concrete contexts.

## Alternatives considered

I considered buffering the SystemRandomNumberGenerator within the standard
library, so that all existing code would automatically benefit even without
any further changes. This was undesirable for several reasons:

- It would either require locking or per-actor/per-thread buffers, which would
  have its own performance impact.
- It could have negative security implications for some uses.
- It would require an extra few kB of dirty memory allocation per process (or
  per-thread).
  
Because of these drawbacks, and because I believe that we can achieve most of
the benefits without doing this, I am not pursuing such a solution at this
time.

## Future directions

I expect to rapidly adopt this API in the implementation of several of the
APIs that consume random number generators in the standard library; these
changes will be optimizations that can be done without further evolution
review.

Once non-copyable type support reaches maturity in the language, I will create
a follow-on proposal that will allow wrapping a random number generator to 
provide opt-in explicit buffering, which will provide an escape hatch to
resolve performance issues that cannot be addressed by the implementation
changes alone.
