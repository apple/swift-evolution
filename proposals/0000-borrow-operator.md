# Borrow Operator, Pass-by-Immutable-Borrow, and Pass-by-Value

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: ...
* Review Manager: TBD
* Status: ...

<!--

*During the review process, add the following fields as needed:*

* Implementation: [apple/swift#NNNNN](https://github.com/apple/swift/pull/NNNNN) or [apple/swift-evolution-staging#NNNNN](https://github.com/apple/swift-evolution-staging/pull/NNNNN)
* Decision Notes: [Rationale](https://forums.swift.org/), [Additional Commentary](https://forums.swift.org/)
* Bugs: [SR-NNNN](https://bugs.swift.org/browse/SR-NNNN), [SR-MMMM](https://bugs.swift.org/browse/SR-MMMM)
* Previous Revision: [1](https://github.com/apple/swift-evolution/blob/...commit-ID.../proposals/NNNN-filename.md)
* Previous Proposal: [SE-XXXX](XXXX-filename.md)

-->

## Introduction

In this document, we propose adding a new operator to the language, marked by
the context sensitive keyword `borrow`, that causes the compiler to pass vars
and inout arguments to callees as arguments using a `pass-by-immutable-borrow`
convention instead of a `pass-by-value` convention.

Swift-evolution thread: [Discussion thread topic for that proposal](https://forums.swift.org/)

## Motivation

Swift normally passes vars as non-`inout` parameters to callees using a
`pass-by-value` convention. A `pass-by-value` argument convention is defined
formally by the `var` being implicitly copied at the relevant callsite so that
each argument appears to the callee as formally independent values. As an
example, consider the following code:

```
func useValueTwice(_ x: Type, _ y: inout Type) {}

var a: Type = ...
useValueTwice(a, &a)
```

Formally during evaluation of `useValueTwice`, the compiler will perform an
immutable exclusive access to `a`'s storage and copy `a` before beginning a
mutable exclusive access to `a`'s storage. The reason that function evaluation
is performed in this way is that otherwise, we could suffer from several
undesireable outcomes, for example:

1. Potentially trigger exclusivity violations. For instance, if we formally
accessed the inout parameters memory before performing the `pass-by-value`, we
would have an exclusivity violation when attempting to exclusively access the
`var` memory to perform our copy.

2. If we did not copy the parameter, we would need to either perform a `take` of
the non-`inout` parameter meaning that said memory would now be
invalidated. This would mean that we couldn't pass in var storage directly as an
`inout` parameter since the memory would be invalidated.

For this and other reasons, we pass these arguments as `pass-by-value`
formally. These copies are observable program behavior and as shown above are
desireable in our formal model of callsite evaluation. In certain cases though,
this `pass-by-value` behavior and the implicit copies it requires the compiler
to insert may be undesireable due to the author needing strict performance
guarantees that implicit copies like the above are never generated. For most
programs this guarantee is not needed since the optimizer generally does a good
enough job eliminating such copies. But in critical system code, "good enough"
is not an acceptable constraint and a true semantic language level guarantee is
needed.

## Proposed solution

We propose introducing a new context sensitive keyword called `borrow` that when
applied to a local var, computed var, or global var passed as a function
argument forces Swift to use a `pass-by-immutable-borrow` convention on a
specific argument instead of a `pass-by-value` convention. This would forbid the
compiler from inserting such copies and instead cause the compiler to pass the
argument in the same manner as an inout argument except performing an immutable
exclusive access to memory instead of a mutable exclusive access. The result of
the usage of borrow is that a performance minded programmer would be able to
sleep well at night knowing that their code doesn't have copies in this code.

## Detailed design

The `borrow` keyword can be applied in argument positions to local, computed,
and global `vars`. It will cause the `var` to use a `pass-by-immutable-borrow`
convention. Lets revisit our `useValueTwice` example from above in more detail:

```
func useValueTwice(_ x: Type, _ y: inout Type) {}

var a: Type = ...
useValueTwice(a, &a)
```

Formally the compiler evaluates `useValueTwice` by:

1. Performing the first part of the `pass-by-exclusive-borrow` convention for
   `inout` parameters by "formally evaluating" all `inout` arguments. Formally
   evaluating a parameter means performing any operations needed to materialize
   such arguments into memory that can later be accessed by the second part of
   the `pass-by-exclusive-borrow` convention. In the case of `a` this is a no-op
   since `a` is a local `var` meaning we can just use its raw memory. In
   contrast, if `a` was a computed property, we would perform a "get" to place
   `a` into temporary memory. NOTE: We at this point have not actually taken
   exclusive access to any memory that will be passed into `useValueTwice`.

2. Implementing the `pass-by-value` convention for non-`inout` parameters. For a
   `var` this means taking immutable exclusive access to the `var`'s storage and
   then performing the aforementioned implicit copy. Once the copy is performed,
   we then close the immutable exclusivity access. The copy is independent from
   the original `a`.

3. Then, we perform the second part of the `pass-by-exclusive-borrow` convention
   for `inout` parameters. This involves formally accessing the (potentially
   materialized) `inout` parameters. In this specific case, since we can just
   access `a`'s storage directly, we begin a mutable exclusive access to `a`'s
   storage.

4. Then, we call our callee passing in the relevant temporary copies and mutable
   references to our exclusive accessed inout parameters.

5. Finally we clean up by:

   a. Ending any mutable exclusive accesses to the memory of any `inout` parameters.

   b. Destroying any non-`inout` parameter temporaries

   c. Setting any computed properties that were materialized into a temporary to
      pass into our callee as an `inout` parameter.

This ensures that we can with ease pass both `a` and `&a` to the same function
without worry. We then define the `pass-by-immutable-borrow` convention by
modifying 1., 3., and 5.a. to include performing an immutable exclusive borrow
of any caller parameters marked with `borrow`. This works just like the `inout`
parameter evaluation except that we perform immutable exclusivity accesses and
do not perform a write back during clean up if we have a computed `var`.

The effect of this is that just like with `inout`, there is a guarantee that
copies will not be made to non-computed `var`s to pass them. Additionally, one
will still be able to pass the same `var` to a function as multiple `borrow`
parameters as well as any `pass-by-value` parameters. In contrast, one will have
an exclusivity violation if one passes a var as a `borrow` if one additionally
passes it as an `inout` parameter or as a capture of an escaping
closure. Example:

```
// Ok. We copy x and borrow x.
takeTwoValues(x, borrow x)

// Ok! Immutable exclusivity doesn't conflict.
takeBorrowAndBorrow(borrow x, borrow x)

// Error! Exclusivity violation!
takeInOutAndBorrow(&x, borrow x)

// Error! Exclusivity violation!
takeBorrowAndClosure(borrow x, { x = Type() })
```

The compiler will error specifically (with a fixit) if one attempts to attach
`borrow` to any of the following:

1. A let. This is similar to the way the compiler handles '&' today.
2. An inout parameter. This would be an exclusivity violation.

## Source compatibility

`borrow` will be a contextual keyword that is only allowed to be applied to
local, computed, and global `var`s thus should not create any source breaks with
function or variable names that use the phrase borrow. So there shouldn't be any
source compatibility breaks.

## Effect on ABI stability

`borrow` does not affect the ABI of the caller where it is used or the ABI of
the callee it is used to call. It only changes code in the caller used to call
the callee and to the callee is invisible.

## Effect on API resilience

As noted above, using `borrow` in a function body does not affect how users call
the caller or the callee.

## Alternatives considered

???

## Future Directions

???

## Acknowledgments

???
