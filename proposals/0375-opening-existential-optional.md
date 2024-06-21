# Opening existential arguments to optional parameters

* Proposal: [SE-0375](0375-opening-existential-optional.md)
* Authors: [Doug Gregor](https://github.com/DougGregor)
* Review Manager: [Xiaodi Wu](https://github.com/xwu)
* Status: **Implemented (Swift 5.8)**
* Implementation: [apple/swift#61321](https://github.com/apple/swift/pull/61321)
* Review: ([pitch](https://forums.swift.org/t/mini-pitch-for-se-0352-amendment-allow-opening-an-existential-argument-to-an-optional-parameter/60501)) ([review](https://forums.swift.org/t/se-0375-opening-existential-arguments-to-optional-parameters/60802)) ([acceptance](https://forums.swift.org/t/accepted-se-0375-opening-existential-arguments-to-optional-parameters/61045))

## Introduction

[SE-0352 "Implicitly Opened Existentials"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0352-implicit-open-existentials.md) has a limitation that prevents the opening of an existential argument when the corresponding parameter is optional. This proposal changes that behavior, so that such a call will succeed when a (non-optional) existential argument is passed to a parameter of optional type:

```swift
func acceptOptional<T: P>(_ x: T?) { }
func test(p: any P, pOpt: (any P)?) {
  acceptOptional(p) // SE-0352 does not open "p"; this proposal will open "p" and bind "T" to its underlying type
  acceptOptional(pOpt) // does not open "pOpt", because there is no "T" to bind to when "pOpt" is "nil"
}
```

The rationale for not opening the existential `p` in the first call was to ensure consistent behavior with the second call, in an effort to avoid confusion. SE-0352 says:

> The case of optionals is somewhat interesting. It's clear that the call `cannotOpen6(pOpt)` cannot work because `pOpt` could be `nil`, in which case there is no type to bind `T` to. We *could* choose to allow opening a non-optional existential argument when the parameter is optional, e.g.,
>
> ```
> cannotOpen6(p1) // we *could* open here, binding T to the underlying type of p1, but choose not to 
> ```
>
> but this proposal doesn't allow this because it would be odd to allow this call but not the `cannotOpen6(pOpt)` call.

However, experience with implicitly-opened existentials has shown that opening an existential argument in the first case is important, because many functions accept optional parameters. It is possible to work around this limitation, but doing so requires a bit of boilerplate, using a generic function that takes a non-optional parameter as a trampoline to the one that takes an optional parameter:

```swift
func acceptNonOptionalThunk<T: P>(_ x: T) { 
  acceptOptional(x)
}

func test(p: any P) {
  acceptNonOptionalThunk(p) // workaround for SE-0352 to get a call to acceptOptional with opened existential
}
```

## Proposed solution

Allow an argument of (non-optional) existential type to be opened to be passed to an optional parameter:

```swift
func openOptional<T: P>(_ value: T?) { }

func testOpenToOptional(p: any P) {
  openOptional(p) // okay, opens 'p' and binds 'T' to its underlying type
}
```

## Source compatibility

Generally speaking, opening an existential argument in one more case will make code that would have been rejected by the compiler (e.g., with an error like "`P` does not conform to `P`") into code that is accepted, because the existential is opened. This can change the behavior of overload resolution, in the same manner as was [discussed in SE-0352](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0352-implicit-open-existentials.md#source-compatibility). Experience with SE-0352's integration into Swift 5.7 implies that the practical effect of these changes is quite small.

## Effect on ABI stability

This proposal changes the type system but has no ABI impact whatsoever.

## Effect on API resilience

This proposal changes the use of APIs, but not the APIs themselves, so it doesn't impact API resilience per se.
