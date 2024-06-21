# Fix Private Access Levels

* Proposal: [SE-0159](0159-fix-private-access-levels.md)
* Author: [David Hart](https://github.com/hartbit)
* Review Manager: [Doug Gregor](https://github.com/DougGregor)
* Status: **Rejected**
* Decision Notes: [Rationale](https://forums.swift.org/t/rejected-se-0159-fix-private-access-levels/5576)

## Introduction

This proposal presents the problems that came with the the access level modifications in [SE-0025](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0025-scoped-access-level.md) and proposes reverting to Swift 2 behaviour.

## Motivation

Since the release of Swift 3, the access level change of SE-0025 was met with dissatisfaction by a substantial proportion of the general Swift community. Those changes can be viewed as *actively harmful*, the new requirement for syntax/API changes.

The `private` keyword is a "soft default" access modifier for restricting access within a file. Scoped access is not a good behavior for a "soft default" because it is extremely common to use several extensions within a file. A "soft default" (and therefore `private`) should work well with this idiom. It is fair to say that changing the behavior of `private` such that it does not work well with extensions meets the criteria of actively harmful in the sense that it subtly encourages overuse of scoped access control and discourages the more reasonable default by giving it the awkward name `fileprivate`.

Compared to a file-based access level, the scoped-based access level adds meaningful information by hiding implementation details which do not concern other types or extensions in the same file. But is that distinction between `private` and `fileprivate` actively used by the larger community of Swift developers? And if it were used pervasively, would it be worth the cognitive load and complexity of keeping two very similar access levels in the language? This proposal argues that answer to both questions is no and therefore wish to simplify Swift's access control story by removing scoped access and leaving more design breathing space for future discussions around submodules.

## Detailed design

The `private` keyword should be reverted back to its Swift 2 file-based meaning and the `fileprivate` keyword should be deprecated.

## Source compatibility

In Swift 3 compatibility mode, the compiler will continue to treat `private` and `fileprivate` as was previously the case.

In Swift 4 mode, the compiler will deprecate the `fileprivate` keyword and revert the semantics of the `private` access level to be file based. The migrator will rename all uses of `fileprivate` to `private`.

Cases where a type had `private` declarations with the same signature in different scopes will produce a compiler error. For example, the following piece of code compiles in Swift 3 compatibility mode but generates a `Invalid redeclaration of 'bar()'` error in Swift 4 mode.

```swift
struct Foo {
    private func bar() {}
}

extension Foo {
    private func bar() {}
}
```

## Alternatives Considered

1. Deprecate `fileprivate` and modify the semantics of `private` to include same-type extension scopes in the same file.
2. Deprecate `fileprivate` and modify the semantics of `private` to include same-type extension scopes in the same module.
3. Revert `private` to be file-based and introduce the scope-based access level under a new name.

## Thanks

I'd like to extend my thanks to Xiaodi Wu and Matthew Johnson for their respective contributions.
