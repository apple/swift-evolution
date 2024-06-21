# Unavailability Condition

* Proposal: [SE-0290](0290-negative-availability.md)
* Author: [Bruno Rocha](https://github.com/rockbruno)
* Review Manager: [Ted Kremenek](https://github.com/tkremenek)
* Implementation: [apple/swift#33932](https://github.com/apple/swift/pull/33932)
* Status: **Implemented (Swift 5.6)**
* Previous revision: [1](https://github.com/swiftlang/swift-evolution/blob/066545c1cc9ff2b87ce233e0f8936f8d53724bdb/proposals/0290-negative-availability.md)
* Decision Notes: [Review #1](https://forums.swift.org/t/se-0290-unavailability-condition/41873/34), [Review #2](https://forums.swift.org/t/se-290-second-review-unavailability-condition/43544/59)

## Introduction

Swift historically supported the `#available` condition to check if a specific symbol **is** available for usage, but not the opposite. In this proposal, we'll present cases where checking for the **unavailability** of something is necessary, the ugly workaround needed to achieve it today and how a new `#unavailable` condition can fix it.

Swift-evolution thread: [Discussion thread topic for that proposal](https://forums.swift.org/t/support-negative-availability-literals/39946)

## Motivation

Checking whether a specific platform/symbol is **not** available is necessary when the changes made to an API are so extreme that you cannot represent it in a single if/else statement. The most common example is how building an iOS app's main `UIWindow` changed with the introduction of `SceneDelegates`. While a basic iOS 12 `AppDelegate` app sets up its window when the app finishes launching, apps that support `SceneDelegates` should instead do it *later* in the app's lifecycle -- more specifically, when the AppDelegate connects the app's main `UIScene`. Since this happens outside the usual `didFinishLaunching` flow, this extreme difference in behavior cannot be conveyed by a simple if/else availability check. Instead, the `AppDelegate` will require a *negative* platform check that makes it sure it only sets up the window if the user is **not** running iOS 13:

```swift
// if NOT in iOS 13, load the window.
// Post iOS 13 the window is loaded later in the lifecycle, in the SceneDelegate.
if #available(iOS 13, *) {

} else {
  loadMainWindow()
}
```

### Readability

As you might notice, the current way to achieve negative availability checks is by working around the current `#available` keyword. Because the availability condition is not parsed as an expression, it cannot be negated with regular boolean operations (`!`/`== false`). The way instead is to make use of the `else` clause, but as unavailability checks are not interested at all in the positive portion of the check, doing so will leave behind an empty `if` branch. For context, this problem does not exist in Objective-C as `if (@available(iOS 13.0, *) == NO)` is a valid expression.

With the exception of this very specific case, an empty statement is a sure sign that there's something wrong in the developer's code, and it is likely that every unavailability check like this had to include a comment to indicate that it was done on purpose due to the compiler's limitations. In some cases, it might even be necessary to add an exclusion rule to the project's linters as most would assume that this is a mistake and incorrectly suggest that it can be fixed by negating the statement.

This workaround has a clear negative impact on the readability of the unavailability check, as no one would expect an empty `if` statement to not be a coding mistake. Most developers will attempt to hide the problem by putting the positive portion in a single line:

```swift
// if NOT in iOS 13, load the window.
// Post iOS 13 the window is loaded later in the lifecycle, in the SceneDelegate.
if #available(iOS 13, *) {} else {
  loadMainWindow()
}
```

Unfortunately, this makes it easy to mistake it for a regular availability check. If the comment isn't clear that the statement is wrong on purpose, problems involving this check could easily land unnoticed in code-reviews. A less noisy way would be to put it under a `guard`:

```swift
guard #available(iOS 13, *) else {
  loadMainWindow()
  return
}
// no-op
```

However, this goes against the code style recommendations involving the usage of `guard`. The guarded part should be the happy path, which is not the case when doing unavailability checks. As shown, it's currently impossible to write an unavailability check that properly fits a developer's engineering expectations and Swift's general style guide. 

Currently, any iOS application that supports UIScenes will face this issue and have to write this workaround. To describe it in a generic way, this issue will be encountered when dealing with any API addition or change that requires more than one availability condition to be implemented.

### Usage of deprecated APIs

A negative availability condition might also be necessary in cases where an API is marked as deprecated (and documented as non-functional) in newer OS versions.

An example is the deprecation of the [`isAdvertisingTrackingEnabled`](https://developer.apple.com/documentation/adsupport/asidentifiermanager/1614148-isadvertisingtrackingenabled) property. Apps supporting iOS 14 must now use the new App Tracking Transparency framework for user tracking purposes, which involves displaying a permission alert for the user that explains why they are going to be tracked. A developer might determine that this large change in functionality might warrant a complete refactor of this feature, or simply conclude that the negative UX of displaying a new alert is not worth it and that they should remove this feature entirely. In any case however, the property still works when used with older iOS versions:

```swift
// If NOT on iOS 14, track this action.
// Post iOS 14, we must ask for permission to do this.
// The UX impact is not worth it. Let's not do this at all.
if #available(iOS 14.0) {

} else {
  oldIos13TrackingLogic(isEnabled: ASIdentifierManager.shared().isAdvertisingTrackingEnabled)
}
```

In this specific case, a company that wants to adopt the new privacy practices will require unavailability checks to prevent breaking old versions of the app. In general, this will be the case when dealing with any API that is now deprecated.

### Code Structure

Besides cases where having an unavailability check is mandatory, supporting them would give developers more options when structuring their code in cases where they are not mandatory. By not being forced to consider the availability of something as the happy path, developers would have more choices when considering how to architect and abstract certain pieces of code.

## Proposed solution

Given that Objective-C is capable of negating availability checks, we believe that this not being supported in Swift was simply an oversight. We would like to propose this feature back to Swift in the shape of a new `#unavailable` condition: 

```swift
if #unavailable(iOS 13, *) {
  loadMainWindow()
}
```

As dictated by the name, `#unavailable` is the reverse version of `#available`. Having a proper unavailability check will eliminate the need to use the current workaround and makes it clear to the reader that the statement is checking for the *lack* of a specific version, eliminating the need to provide a comment explaining what that piece of code is trying to achieve.

Like with `#available`, `#unavailable` has the capacity to increase the symbol availability of a scope. As opposed to `#available`, the availability is increased at the `else` clause.

```swift
if #unavailable(iOS 13, *) {
  // Symbol Availability: Default (deployment target)
} else {
  // Symbol Availability: iOS 13
}
```

## Detailed design

As the compiler is already able to calculate the symbol availability for both the positive and negative flows of the check, implementing `#unavailable` is simply a matter of introducing a new keyword that reverses them. This allows `#unavailable` to be internally implemented as a simple boolean that flips `#available's` functionality.

Implementation: [apple/swift#33932](https://github.com/apple/swift/pull/33932)

### Preventing impossible conditions

The ability to use several availability checks in a single statement allows positive and negative availability checks to be mixed. This will lead to an ambiguous symbol availability:

```swift
// User running something between iOS 9 and 12
if #available(iOS 9.0, *), #unavailable(iOS 13.0, *) {
  // Symbol Availability: iOS 9.0
} else {
  // Symbol Availability: ???
}
```

The availability of the else block cannot be determined because it depends on which of the two conditions is false. To prevent this from happening, the compiler will emit a diagnostic whenever this happens.

```swift
if #available(iOS 9.0, *), #unavailable(iOS 13.0, *)
// error: #available and #unavailable cannot be in the same statement
```

Technically we could support this by *not* improving the symbol availability of a scope if it's ambiguous, but as there are currently no legitimate cases where one would have to mix availability with unavailability, the author believes the work necessary to support this and its edge cases is not worth it at the time this proposal was written. However, you can still use them as separate statements.

```swift
if #available(iOS 9.0, *) {
  // Symbol Availability: iOS 9.0
  if #unavailable(iOS 13.0, *) {
    // Symbol Availability: iOS 9.0
  } else {
    // Symbol Availability: iOS 13.0
  }
} else {
  // Symbol Availability: Default (deployment target)
}
```

As they are separate statements, there's no ambiguity.

### Multiple Elses

In the case of multiple else flows, the compiler will increase the symbol availability in **all** of them.

```swift
if #unavailable(iOS 9.0, *) {
  // Symbol Availability: Default (deployment target)
} else if a == b {
  // Symbol Availability: iOS 9.0
} else if b == c {
  // Symbol Availability: iOS 9.0
} else {
  // Symbol Availability: iOS 9.0
}
```

### Semantics of `*` for unavailability

Availability statements are composed by a list of platform specs. The purpose of the list is to answer what's the platform version requirement necessary for the statement to return true in the platform where the code is being compiled.

For this to work properly, the spec list must always contain an entry that matches the platform that's being compiled. An entry can be added to the list by either writing an explicit platform requirement (like `iOS 12`) or by using the generic platform wildcard `*`. Essentially, the wildcard signals that the expression being written is unrelated to the current platform, meaning that no version specification is needed. In practice, "no version specification needed" is done by treating the wildcard as the minimum deployment target of the platform, which essentially will cause the statement to never be false.

```swift
// Example: Code is compiling in iOS

error: condition required for target platform 'iOS'
if #available(macOS 11) {
   ^

/// Examples of possible solutions:
if #available(macOS 11, *)
if #available(macOS 11, iOS 12)
if #available(iOS 12)
// In practice, the last two will not compile due to an additional requirement that is mentioned below.
```

Even if you have no plans to use your code in a different platform, your statement must still define the wildcard as a way to define what should happen in all unspecified current and potential new future platforms. The compiler uses the wildcard to ease porting to new platforms -- because the platform being compiled must always have an entry in the spec list, the wildcard allows these platforms to compile your code without requiring a modification to every availability guard in the program. Additionally, because new platforms typically branch from existing platforms, the wildcard also makes sure these ported checks will always return `true` by default when checking for availability.

It's important to note that availability spec lists **are not boolean expressions**. For example, it's not possible to add multiple versions of a platform to the statement:

```swift
if #available(iOS 12, *)
if #available(iOS 12, iOS 13, *) // Error: Version for 'iOS' already specified
```

Additionally, the specification of different platforms have no effect on the final result -- it depends *only* on the (unique) spec that matches the current platform being compiled. A check like `#available(iOS 12, watchOS 4, *)` compiled in iOS doesn't mean "return true if (iOS 12 **||** watchOS 4 **||** the current platform's minimum deployment target) is available", it simply means "return true if iOS 12 is available". The specs that refer to different platforms are ignored.

Finally, the wildcard represents *only* the platforms that were not explicitly specified in the spec list. When `#available(iOS 13, *)` is compiled for iOS, the wildcard will be ignored in favor of the explictly defined iOS 13 spec. As mentioned before, a platform can only be mentioned once.

For unavailability, the semantics mentioned above means that `#unavailable(*)` and `#unavailable(notWhatImCompiling X, *)` should do the opposite of `#available` and return `false`. Since the minimum deployment target will always be present, the statement can never be true. This behavior is exactly how the current workaround works, and it also matches how the theoretical `!#available(*)` would behave.

```swift
if #unavailable(*) {
  // Will never be executed
} else {
  // ...
}
```

As an interesting side effect, this means that having multiple unavailability checks in the same statement (`#unavailable(iOS 13, *), #unavailable(watchOS 3, *)` as opposed to `#unavailable(iOS 13, watchOS 3, *)`) would cause the statement to always be false if they are triggering the wildcard (in this case, because they cover different platforms). 

In these cases, since wildcard checks are eventually optimized to boolean literals, the compiler will already emit a warning indicating that the code will never be executed. Still, we could provide a more descriptive diagnostic that suggests using a single check that considers all platforms. 

```swift
if #unavailable(iOS 13, *), #unavailable(watchOS 3, *) {
  // ... 
  // Warning: code will never be executed
  // Warning: unavailability checks in this statement are canceling each other, use a single check that treats all platforms 
  // fix-it: #unavailable(iOS 13, watchOS 3, *)
}
```

### Result builders

As `#unavailable` behaves exactly like `#available`, [`ViewBuilder`](https://developer.apple.com/documentation/swiftui/viewbuilder) does not need to be modified to support it. Using `#unavailable` on a builder will simply instead trigger [`buildLimitedAvailability(_:)`](0289-result-builders.md#availability) in the `else` block.

## Source compatibility and ABI

This change is purely additive.

## Alternatives considered

### `!#available(...)` and `#available(...) == false`

While allowing the original condition to be reversed seems to be the obvious choice, supporting it in practice would require hardcoding all of this behavior as `#available` cannot be used as an expression. The author would rather not add tech debt to the compiler.

Refactoring `#available` to be usable as an expression would likely require refactoring the entire symbol availability system and has an extensive amount of implications and edge cases. The work to support it would be considerably beyond what is proposed here.

Supporting it by hardcoding this behavior is possible though, and could be implemented if the core team is willing and has a plan to eliminate the resulting tech debt in the future.

On the other hand, given that it's fair to consider that this is a developer's first guess when attempting to do unavailability, the compiler will provide fix-its for each of these spellings.

### `#unavailable(iOS 12)`, `#unavailable(iOS 12 && *)`, `#available(iOS < 12, *)` and other alternatives that involve reworking spec lists

One point of discussion was the importance of the wildcard in the case of unavailability. Because the wildcard achieves nothing in terms of functionality, we considered alternatives that involved omitting or removing it completely. However, the wildcard is still important from a platform migration point of view, because although we don't need to force the guarded branch to be executed like in `#available`, the presence of the wildcard still play its intended role of allowing code involving unavailability statements to be ported to different platforms without requiring every single statement to be modified.

Additionally, we had lenghty discussions about the *readability* of unavailability statements. We've noticed that even though availability in Swift has never been a boolean expression, it was clear that pretty much every developer's first instinct is to assume that `(iOS 12, *)` is equivalent to `iOS 12 || *`. The main point of discussion then was that the average developer might not understand why a call like `#unavailable(iOS 12, *)` will return `false` in non-iOS platforms, because they will assume the list means `iOS 12 || *` (`true`), while in reality (and as described in the `Semantics` section) the list means just `*` (`false`). During the pitch we tried to come up with alternatives that could eliminate this, and although some of them *did* succeed in doing that, they were doing so at the cost of making `#unavailable` "misleading", just like in the case of `!#available`. We ultimately decided that these improvements would be better suited for a *separate* proposal that focused on improving spec lists in general, which will be mentioned again at the end of this section.

In general, there was much worry that this confusion could cause developers to misuse `#unavailable` and introduce bugs in their applications. We can prove that this feeling cannot happen in practice by how `#unavailable` doesn't introduce any new behavior -- it's nothing more than a reversed `#available` with a reversed literal name, which is semantically no different than the current workaround of using the `else` branch. Any confusing `#unavailable` scenario can also be conveyed as a confusing `#available` scenario by simply swapping the branches.

Additionally, we were unable to locate concrete examples where this confusion could *actually* cause the feature to be misused. This is because if someone *does* misunderstand the branches, the project will simply fail to compile as there are symbols being used outside of an availability range. This was the case even when we tried to make the statements as vague as possible in an attempt to introduce a bug on purpose, and should hopefully make it clear for a confused developer that their code is simply upside-down.

Although it's possible for developers to feel confused by the syntax, this is something that already exists with `#available` and cannot result in the feature being misused. As `#unavailable` does not introduces any new functionality that could change this, the author personally believes that this issue could be considered harmless and orthogonal to this proposal.

However, we *do* have an unanimous agreement that spec lists can be confusing and that a new proposal should be created that re-evaluates how they are defined in code. Some members have also shared their belief that this re-evaluation should also happen *before* this proposal is introduced, which the author also agrees if there is a strong argument that `#unavailable` as is can be harmful for Swift.
