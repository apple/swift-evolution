# Handling Future Enum Cases

* Proposal: [SE-0192](0192-non-exhaustive-enums.md)
* Author: [Jordan Rose](https://github.com/jrose-apple)
* Review Manager: [Ted Kremenek](https://github.com/tkremenek)
* Status: **Implemented (Swift 5.0)**
* Implementation: [apple/swift#14945](https://github.com/apple/swift/pull/14945)
* Previous revision: [1](https://github.com/swiftlang/swift-evolution/blob/a773d07ff4beab8b7855adf0ac56d1e13bb7b44c/proposals/0192-non-exhaustive-enums.md), [2 (informal)](https://github.com/jrose-swiftlang/swift-evolution/blob/57dfa2408fe210ed1d5a1251f331045b988ee2f0/proposals/0192-non-exhaustive-enums.md), [3](https://github.com/swiftlang/swift-evolution/blob/af284b519443d3d985f77cc366005ea908e2af59/proposals/0192-non-exhaustive-enums.md)
* Pre-review discussion: [Enums and Source Compatibility](https://forums.swift.org/t/enums-and-source-compatibility/6460), with additional [orphaned thread](https://forums.swift.org/t/enums-and-source-compatibility/6651)
* Review discussion: [Review author summarizes some feedback from review discussion and proposes alternatives](https://forums.swift.org/t/se-0192-non-exhaustive-enums/7291/26), [full discussion thread](https://forums.swift.org/t/se-0192-non-exhaustive-enums/7291/337), plus [Handling unknown cases in enums](https://forums.swift.org/t/handling-unknown-cases-in-enums-re-se-0192/7388/)
* Decision Notes: [Rationale](https://forums.swift.org/t/se-0192-non-exhaustive-enums-review-2/11043/62)

## Introduction

Currently, adding a new case to an enum is a source-breaking change, something that's at odds with Apple's established process for evolving APIs. This proposal aims to distinguish between enums that are _frozen_ (meaning they will never get any new cases) and those that are _non-frozen,_ and to ensure that clients handle any future cases when dealing with the latter.

A key note: in this version of the proposal, *nothing changes for user-defined Swift enums.* This only affects C enums and enums in the standard library and overlays today. (This refers to libraries that Apple could hypothetically ship with its OSs, as it does with Foundation.framework and the Objective-C runtime.) The features described here may be used by third-party libraries in the future.


### Post-acceptance revision

- Since the proposal was accepted months after it was written, the rollout plan turned out to be a little too aggressive. Therefore, in Swift 5 the diagnostic for omitting `@unknown default:` or `@unknown case _:` will only be a warning, and in Swift 4 mode there will be no diagnostic at all. (The previous version of the proposal used an error and a warning, respectively.) Developers are still free to use `@unknown` in Swift 4 mode, in which case the compiler will still produce a warning if all known cases are not handled.


### Revision at acceptance

- The "new case" `unknown:` was changed to the `unknown` attribute, which can only be applied to `default:` and `case _:`.


### Differences from the first revision

- [This now only affects C enums and enums defined in the standard library and overlays](https://forums.swift.org/t/se-0192-non-exhaustive-enums/7291/337)
- The `unknown` case has been added, to preserve exhaustivity checking
- The term used to describe enums that will not change is now "frozen" rather than "exhaustive"
- The proposal now describes what will happen if you "break the contract" in a new library version
- Much more discussion of future directions and alternatives considered

Thanks to everyone who offered feedback!


## Motivation

It's well-established that many enums need to grow new cases in new versions of a library. For example, in last year's release of iOS 10, Foundation's [DateComponentsFormatter.UnitsStyle][] gained a `brief` case and UIKit's [UIKeyboardType][] gained an `asciiCapableNumberPad` case. Large error enums also often grow new cases to go with new operations supported by the library. This all implies that library authors *must* have a way to add new cases to enums without breaking binary compatibility.

At the same time, we really like that you can exhaustively switch over enums. This feature helps prevent bugs and makes it possible to enforce [definitive initialization][DI] without having `default` cases in every `switch`. So we don't want to get rid of enums where every case is known, either. This calls for a distinction between enums where every case can be known statically and enums that might grow new cases in the future.

To see how this distinction will play out in practice, I investigated the public headers of Foundation in the macOS SDK. Out of all 60 or so `NS_ENUM`s in Foundation, only 6 of them are clearly intended to be switched exhaustively:

- [ComparisonResult](https://developer.apple.com/documentation/foundation/comparisonresult)
- [NSKeyValueChange](https://developer.apple.com/documentation/foundation/nskeyvaluechange) / [NSKeyValueSetMutationKind](https://developer.apple.com/documentation/foundation/nskeyvaluesetmutationkind)
- [NSRectEdge](https://developer.apple.com/documentation/foundation/nsrectedge)
- [FileManager.URLRelationship](https://developer.apple.com/documentation/foundation/filemanager.urlrelationship)
- *maybe* [Decimal.CalculationError](https://developer.apple.com/documentation/foundation/nsdecimalnumber.calculationerror)

...with a handful more that could go either way, such as [Stream.Status](https://developer.apple.com/documentation/foundation/stream.status). This demonstrates that there is a clear default for public enums, at least in Objective-C.

  [DateComponentsFormatter.UnitsStyle]: https://developer.apple.com/documentation/foundation/datecomponentsformatter.unitsstyle
  [UIKeyboardType]: https://developer.apple.com/documentation/uikit/uikeyboardtype
  [DI]: https://developer.apple.com/swift/blog/?id=28


## Proposed solution

In Swift 4.2, enums imported from C and enums defined in the standard library and overlays are either *frozen* or *non-frozen.* (Grammatical note: they are not "unfrozen" because that implies that they were frozen at one point.)

When a client tries to switch over a non-frozen enum, they should include a "catch-all" case of some kind (`default`, `case _`, etc). In Swift 5 mode, omitting this case will result in a warning.

All enums written in Swift outside of the standard library and overlays will implicitly be considered frozen in Swift 4.2. Enums imported from C will be non-frozen by default, with a new C-side annotation to treat them as frozen.


## Detailed design

When switching over a non-frozen enum, the switch statement that matches against it must include a catch-all case (usually `default` or an "ignore" `_` pattern).

```swift
switch excuse {
case .eatenByPet:
  // …
case .thoughtItWasDueNextWeek:
  // …
}
```

Failure to do so will produce a warning in Swift 5. A program will trap at run time if an unknown enum case is actually encountered.

All other uses of enums (`if case`, creation, accessing members, etc) do not change. Only the exhaustiveness checking of switches is affected by the frozen/non-frozen distinction. Non-exhaustive switches over frozen enums (and boolean values) will continue to be invalid in all language modes.

Here's a more complicated example:

```swift
switch (excuse, notifiedTeacherBeforeDeadline) {
case (.eatenByPet, true):
  // …
case (.thoughtItWasDueNextWeek, true):
  // …
case (_, false):
  // …
}
```

This switch handles all *known* patterns, but still doesn't account for the possibility of a new enum case when the second tuple element is `true`. This should result in a warning in Swift 5, like the first example.


### `@unknown`

The downside of using a `default` case is that the compiler can no longer alert a developer that a particular enum has elements that aren't explicitly handled in the `switch`. To remedy this, switch cases will gain a new attribute, `@unknown`.

```swift
switch excuse {
case .eatenByPet:
  // …
case .thoughtItWasDueNextWeek:
  // …
@unknown default:
  // …
}
```

Like the regular `default`, `@unknown default` matches any value; it is a "catch-all" case. However, the compiler will produce a *warning* if all known elements of the enum have not already been matched. This is a warning rather than an error so that adding new elements to the enum remains a source-compatible change. (This is also why `@unknown default` matches any value rather than just those not seen at compile-time.)

`@unknown` may only be applied to `default` or a case consisting of the single pattern `_`. Even in the latter case, `@unknown` must be used with the last case in a `switch`. This restriction is discussed further in the "`unknown` patterns" section under "Future directions".

The compiler will warn if all enums in the pattern being matched by `@unknown` are explicitly annotated as frozen, or if there are no enums in the pattern at all. This is a warning rather than an error so that annotating an enum as frozen remains a source-compatible change. If the pattern contains any enums that are implicitly frozen (i.e. because it is a user-defined Swift enum), `@unknown` is permitted, in order to make it easier to adapt to newly-added cases.

`@unknown` has a downside that it is not testable, since there is no way to create an enum value that does not match any known cases, and there wouldn't be a safe way to use it if there was one. However, combining `@unknown` with other cases using `fallthrough` can get the effect of following another case's behavior while still getting compiler warnings for new cases.

```swift
switch excuse {
case .eatenByPet:
  showCutePicturesOfPet()

case .thoughtItWasDueNextWeek:
  fallthrough
@unknown default:
  askForDueDateExtension()
}
```


### C enums

Enums imported from C are tricky, because it's difficult to tell whether they're part of the current project or not. An `NS_ENUM` in Apple's SDK should probably be treated as non-frozen, but one in your own framework might be frozen. Even there, though, it's possible that there's a "private case" defined in a .m file:

```objc
// MyAppPaperSupport.h
typedef NS_ENUM(NSInteger, PaperSize) {
  PaperSizeUSLetter = 0,
  PaperSizeA4 = 1,
  PaperSizePhoto4x6 = 2
};
```
```objc
// MyAppPaperSupport.m
static const PaperSize PaperSizeStickyNote = 255;
```

(While this pattern may be unfamiliar, it is used in Apple's SDKs, though not often.)

Therefore, enums imported from C will be treated conservatively: an otherwise-unannotated `NS_ENUM` will be imported as non-frozen and treated as such in all contexts. The newly-added C attribute `enum_extensibility` can be used to override this behavior:

```objc
typedef NS_ENUM(NSInteger, GregorianMonth) {
  GregorianMonthJanuary = 1,
  GregorianMonthFebruary,
  GregorianMonthMarch,
  GregorianMonthApril,
  GregorianMonthMay,
  GregorianMonthJune,
  GregorianMonthJuly,
  GregorianMonthAugust,
  GregorianMonthSeptember,
  GregorianMonthOctober,
  GregorianMonthNovember,
  GregorianMonthDecember,
} __attribute__((enum_extensibility(closed)));
```

Apple doesn't speak about future plans for its SDKs, so having an alternate form of `NS_ENUM` that includes this attribute is out of scope for this proposal.

Apart from the effect on switches, a frozen C enum's `init(rawValue:)` will also enforce that the case is one of those known at compile time. Imported non-frozen enums will continue to perform no checking on the raw value.

> This section only applies to enums that Swift considers "true enums", rather than option sets or funny integer values. In the past, the only way to get this behavior was to use the `NS_ENUM` or `CF_ENUM` macros, but the presence of `enum_extensibility(closed)` *or* `enum_extensibility(open)` will instruct Swift to treat the enum as a "true enum". Similarly, the newly-added `flag_enum` C attribute can be used to signify an option set like `NS_OPTIONS`.


### Effect on the standard library and overlays

The majority of enums defined in the standard library do not need the flexibility afforded by being non-frozen, and so will be marked as frozen. This includes the following enums:

- ❄️ ClosedRange.Index
- ❄️ FloatingPointSign
- ❄️ FloatingPointClassification
- ❄️ Never
- ❄️ Optional
- ❄️ UnicodeDecodingResult
- ❄️ Unicode.ParseResult

The following public enums in the standard library will *not* be marked as frozen:

- DecodingError
- EncodingError
- FloatingPointRoundingRule
- Mirror.AncestorRepresentation
- Mirror.DisplayStyle
- PlaygroundQuickLook (deprecated anyway)

And while the overlays are not strictly part of the Swift Open Source project (since they are owned by framework teams at Apple), the tentative plan would be to mark these two enums as frozen:

- ❄️ ARCamera.TrackingState (a tri-state of "on", "off", and "limited(Reason)")
- ❄️ DispatchTimeoutResult ("success" and "timed out")

And the other public enums in the overlays would be non-frozen:

- ARCamera.TrackingState.Reason
- Calendar.Component
- Calendar.Identifier
- Calendar.MatchingPolicy
- Calendar.RepeatedTimePolicy
- Calendar.SearchDirection
- CGPathFillRule
- Data.Deallocator
- DispatchData.Deallocator
- DispatchIO.StreamType
- DispatchPredicate
- DispatchQoS.QoSClass
- DispatchQueue.AutoreleaseFrequency
- DispatchQueue.GlobalQueuePriority (deprecated anyway)
- DispatchTimeInterval
- JSONDecoder.DataDecodingStrategy
- JSONDecoder.DateDecodingStrategy
- JSONDecoder.KeyDecodingStrategy
- JSONDecoder.NonConformingFloatDecodingStrategy
- JSONEncoder.DataEncodingStrategy
- JSONEncoder.DateEncodingStrategy
- JSONEncoder.KeyEncodingStrategy
- JSONEncoder.NonConformingFloatEncodingStrategy
- MachErrorCode
- POSIXErrorCode


## Comparison with other languages

"Enums", "unions", "variant types", "sum types", or "algebraic data types" are present in a number of other modern languages, most of which don't seem to treat this as an important problem.


### Languages without non-frozen enums

**Haskell** and **OCaml** make heavy use of enums ("algebraic data types", or just "types") without any feature like this; adding a new "case" is always a source-breaking change. (Neither of these languages seems to care much about binary compatibility.) This is definitely a sign that you can have a successful language without a form of non-frozen enum other than "protocols". **Kotlin** also falls in this bucket, although it uses enums ("enum classes") less frequently.

The **C#** docs have a nice section on [how the language isn't very helpful][c-sharp] for distinguishing frozen and non-frozen enums. **Objective-C**, of course, is in the same bucket, though Apple could start doing things with the `enum_extensibility` Clang attribute that was recently added.

  [c-sharp]: https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/enum#robust-programming


### Languages with alternate designs

**F#** enums ("unions") [either expose all of their "cases" or none of them][f-sharp]. The Swift equivalent of this would be not allowing you to switch on such an enum at all, as if it were a struct with private fields.

Enums in **D** are like enums in C, but D distinguishes `switch` from `final switch`, and only the latter is exhaustive. That is, it's a client-side decision at the use site, rather than a decision by the definer of the enum.

**Scala** has enums, but the pattern most people seem to use is "sealed traits", which in Swift terms would be "protocols where all conforming types are known, usually singletons". A non-frozen enum would then just be a normal protocol. Some downsides of applying this to Swift are discussed below under "Use protocols instead".

  [f-sharp]: https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/signatures


### Languages with designs similar to this proposal

**Rust** has an [accepted proposal][rust] to add non-frozen enums that looks a lot like this one, but where "frozen" is still the default to not break existing Rust programs. (There are some interesting differences that come up in Rust but not Swift; in particular they need a notion of non-frozen structs because their structs can be decomposed in pattern-matching as well.)

  [rust]: https://github.com/rust-lang/rfcs/blob/master/text/2008-non-exhaustive.md



## Source compatibility

It is now a source-compatible change to add a case to a non-frozen enum (whether imported from C or defined in the standard library).

It is not a source-compatible change to add a case to a frozen enum.

It is still not a source-compatible change to remove a case from a public enum (frozen or non-frozen).

It is a source-compatible change to change a non-frozen enum into a frozen enum, but not vice versa.


### Breaking the contract

If a library author adds a case to a frozen enum, any existing switches will likely not handle this new case. The compiler will produce an error for any such switch (i.e. those without a `default` case or `_` pattern to match the enum value), noting that the case is unhandled; this is the same error that is produced for a non-exhaustive switch in Swift 4.

If a library author changes an enum previously marked frozen to make it non-frozen, the compiler will produce a warning for any switch that does not have a catch-all case.


## Effect on ABI stability

The layout of a non-frozen Swift enum must not be exposed to clients, since the library may choose to add a new case that does not fit in that layout in its next release. This results in extra indirection when that enum appears in public API. The layout of a frozen enum will continue to be made available to clients for optimization purposes.

This change does not affect the layout of `@objc` enums, whether imported from C or defined in Swift. (Note that the representation of a non-`@objc` enum's case may differ from its raw value; this improves the efficiency of `switch` statements when all cases are known at compile time.)


## Effect on Library Evolution

It is now a binary-compatible change to add a case to a non-frozen enum.

It is still not a binary-compatible change to remove a case from a public enum (frozen or non-frozen).

It is not a binary-compatible change to add `@objc` to an enum, nor to remove it.

Taking an existing non-frozen enum and making it frozen is something we'd like to support without breaking binary compatibility, but there is no design for that yet. The reverse will not be allowed.


### Breaking the contract

Because the compiler uses the set of cases in a frozen enum to determine its in-memory representation and calling convention, adding a new case or marking such an enum as non-frozen will result in "undefined behavior" from any client apps that have not been recompiled. This means a loss of memory-safety and type-safety on par with a misuse of "unsafe" types, which would most likely lead to crashes but could lead to code unexpectedly being executed or skipped. In short, things would be very bad.

Some ideas for how to prevent library authors from breaking the rules accidentally are discussed in "Compatibility checking" under "Future directions".

As a special case, switching over an unexpected value in an `@objc` enum (whether imported or defined in Swift) will always result in a trap rather than "undefined behavior", even if the enum is frozen.


## Future directions

### Non-frozen Swift enums outside the standard library

Earlier versions of this proposal included syntax that allowed *all* public Swift enums to have a frozen/non-frozen distinction, rather than just those in the standard library and overlays. This is still something we want to support, but the core team has made it clear that such a distinction is only worth it for libraries that have binary compatibility concerns (such as those installed into a standard location and used by multiple clients), at least without a more developed notion of versioning and version-locking. Exactly what it means to be a "library with binary compatibility concerns" is a large topic that deserves its own proposal.


### `unknown` patterns

As described, `@unknown` cases can only be used to match the entire switched value; it does not work when trying to match a tuple element, or another enum's associated type. In theory, we could make a new *pattern* kind that allows matching unknown cases anywhere within a larger pattern:

```swift
switch (excuse, notifiedTeacherBeforeDeadline) {
case (.eatenByPet, true):
  // …
case (.thoughtItWasDueNextWeek, true):
  // …
case (#unknown, true):
  // …
case (_, false):
  // …
}
```

(The `#unknown` spelling is chosen by analogy with `#selector` to not conflict with existing syntax; it is not intended to be a final proposal.)

However, this produces potentially surprising results when followed by a case that could also match a particular input. Because `@unknown` is only supported on catch-all cases, the input `(.thoughtItWasDueNextWeek, true)` would result in case 2 being chosen rather than case 3.

```swift
switch (excuse, notifiedTeacherBeforeDeadline) {
case (.eatenByPet, true): // 1
  // …
case (#unknown, true): // 2
  // …
case (.thoughtItWasDueNextWeek, _): // 3
  // …
case (_, false): // 4
  // …
}
```

The compiler would warn about this, at least, since there is a known value that can reach the `unknown` pattern.

`@unknown` must appear only on the last case in a switch to avoid this issue. However, it's not possible to enforce the same thing for arbitrary patterns because there may be multiple enums in the pattern whose unknown cases need to be treated differently.

A key point of this discussion is that as proposed `@unknown` merely produces a *warning* when the compiler can see that some enum cases are unhandled, rather than an error. If the compiler produced an error instead, it would make more sense to use a pattern-like syntax for `unknown` (see the naming discussions under "Alternatives considered"). However, if the compiler produced an error, then adding a new case would not be a source-compatible change.

For these reasons, generalized `unknown` patterns are not being included in this proposal.


### Using `@unknown` with other catch-all cases

At the moment, `@unknown` is only supported on cases that are written as `default:` or as `case _:`. However, there are other ways to form catch-all cases, such as `case let value:`, or `case (_, let b):` for a tuple input. Supporting `@unknown` with these cases was considered outside the scope of this proposal, which had already gone on for quite a while, but there are no known technical issues with lifting this restriction.


### Non-public cases

The work required for non-frozen enums also allows for the existence of non-public cases in a public enum. This already shows up in practice in Apple's SDKs, as described briefly in the section on "C enums" above. Like "enum inheritance", this kind of behavior can mostly be emulated by using a second enum inside the library, but that's not sufficient if the non-public values need to be vended opaquely to clients.

Were such a proposal to be written, I advise that a frozen enum not be permitted to have non-public cases. An enum in a user-defined library would then be implicitly considered frozen if and only if it had no non-public cases.


### Compatibility checking

Of course, the compiler can't stop a library author from adding a new case to a frozen enum, even though that will break source and binary compatibility. We already have two ideas on how we could catch mistakes of this nature:

- A checker that can compare APIs across library versions, using swiftmodule files or similar.

- Encoding the layout of a type in a symbol name. Clients could link against this symbol so that they'd fail to launch if it changes, but even without that an automated system could check the list of exported symbols to make sure nothing was removed.

Frozen enums remain useful even without any automated checking, and such checking should account for more than just enums, so it's not being included in this proposal.


### Efficient representation of enums with raw types

For enums with raw types, a 32-bit integer can be used as the representation rather than a fully opaque value, on the grounds that 4 billion is a reasonable upper limit for the number of distinct cases in an enum without payloads. However, this would make it an ABI-breaking change to add or remove a raw type from an enum, and would make the following definitions not equivalent:

```swift
/* non-frozen */ public enum HTTPMethod: String {
  case get = "GET"
  case put = "PUT"
  case post = "POST"
  case delete = "DELETE"
}
```

```swift
/* non-frozen */ public enum HTTPMethod: RawRepresentable {
  case get
  case put
  case post
  case delete

  public init?(rawValue: String) {
    switch rawValue {
    case "GET": return .get
    case "PUT": return .put
    case "POST": return .post
    case "DELETE": return .delete
    default: return nil
    }
  }

  public var rawValue: String {
    switch self {
    case .get: return "GET"
    case .put: return "PUT"
    case .post: return "POST"
    case .delete: return "DELETE"
    }
  }
}
```

As such, this representation change is out of scope for this proposal.


## Alternatives considered

### Terminology and syntax

#### Terminology: "closed" and "open"

The original description of the problem used "closed" and "open" to describe frozen and non-frozen enums, respectively. However, this conflicts with the use of `open` in classes and their members. In this usage, `open` is clearly a greater level of access than `public`, in that clients of an `open` class can do everything they can with a `public` class and more; it is source-compatible to turn a `public` class into an `open` one. For enums, however, it is frozen enums that are "greater": you can do everything you can with a non-frozen enum and more, and it would be source-compatible for a standard library contributor to turn a non-frozen enum into a frozen one (at the cost of a warning).


#### Terminology: other options

Several more options were suggested during initial discussions:

- complete / incomplete
- covered
- exhaustive / non-exhaustive
- non-extensible
- final / non-final
- finite / non-finite (not "infinite")
- fixed
- locked
- sealed / non-sealed
- total / partial

I didn't have a strong preference for any particular choice as long as it *isn't* "closed" / "open", for the reasons described above. In the first revision of this proposal I picked "exhaustive" because it matches the name proposed [in Rust][rust]. (Unfortunately, Clang's `enum_extensibility` attribute, recently added by us at Apple, uses `open` and `closed`.)

Note that "nonextensible" does have one problem: Apple already uses [`NS_TYPED_EXTENSIBLE_ENUM `][NS_TYPED_EXTENSIBLE_ENUM] to refer to enum-like sets of constants (usually strings) that *clients* can add "cases" to. That's not the same meaning as the exhaustiveness discussed in this proposal.

During the first review for this proposal, Becca Royal-Gordon suggested "frozen", which was met with general approval or at least no major objections.

  [NS_TYPED_EXTENSIBLE_ENUM]: https://developer.apple.com/library/content/documentation/Swift/Conceptual/BuildingCocoaApps/InteractingWithCAPIs.html#//apple_ref/doc/uid/TP40014216-CH8-ID206


#### `unknown` naming

The first version of this proposal did not include `unknown`, but did discuss it as a "considered alternative" under the name `future`. Previous discussions have also used `unexpected` or `undeclared` to describe this feature as well.

It was pointed out that neither `future` nor `unexpected` really described the feature being provided. `unknown` does not just handle cases added in the future; it also handles private cases and invalid values for C enums. Nor are such cases entirely unexpected, since the compiler is telling the developer to expect them. `undeclared` has fewer issues, but certainly private cases can be declared *somewhere;* the declarations just aren't visible.

The "intermediate" revision of this proposal where `unknown` was first added used the spelling `unknown case`, but restricted the new case to only match values that *were* enums rather than values *containing* enums. When that restriction was loosened, the reading of `unknown case` as "(enum) cases that I don't know about" no longer made as much sense.

During discussion, the name `unknown default` (or `@unknown default`) was suggested as an alternative to `unknown case`, since the semantics behave very much like `default`. However, it isn't the "default" that's "unknown". Other proposed spellings included `default unknown` (a simple attempt to avoid reading "unknown" as an adjective modifying "default") and `default(unknown)` (by analogy with `private(set)`). Nevertheless, this attribute syntax won out in the end by not tying it to `default`; the alternate spelling `@unknown case _` is also accepted.

Moving away from "unknown", `@unused default` was also suggested, but the case is *not* unused. A more accurate `@runtimeReachableOnly` (or even `@runtimeOnly`) was proposed instead, but that's starting to get overly verbose for something that will appear reasonably often. 

For standalone names, `fallback` was also suggested, but semantically that seems a little too close to "default", and in terms of actual keywords it was pointed out that this was very similar to `fallthrough` despite having no relation. `invisible` was suggested as well (though in the context of patterns rather than cases), but that doesn't exactly apply to future cases.

To summarize, the following spellings were considered for `unknown`:

- `future:`
- `unexpected:`
- `undeclared:`
- `unknown case:`
- `unknown default:`
- `@unknown default:`
- `@unused default:`
- `@runtimeReachableOnly default:`
- `default unknown:`
- `default(unknown):`
- `fallback:`
- `invisible:`

For the review of the proposal, I picked `unknown:` as the best option, admittedly as much for *not* having *unwanted* connotations as for having *good* connotations. The core team ultimately went with `@unknown default:` / `@unknown case _:` instead.

A bigger change would be to make a custom *pattern* instead of a custom *case,* even if it were subject to the same restrictions in implementation (see "`unknown` patterns" above). This usually meant using a symbol of some kind to distinguish the "unknown" from a normal label or pattern, leading to `case #unknown` or similar. This makes the new feature less special, since it's "just another pattern". However, it would be surprising to have such a pattern but keep the restrictions described in this proposal; thus, it would only make sense to do this if we were going to implement fully general pattern-matching for this feature. See "`unknown` patterns" above for more discussion.

Finally, there was the option to put an annotation on a `switch` instead of customizing the catch-all case, e.g. `@warnUnknownCases switch x {`. This is implementable but feels easier for a developer to forget to write, and the compiler can only help if the developer actually *has* implemented all of the current cases alongside their `default` case.


### `switch!`

`switch!` was an alternative to `@unknown` that would not support any action other than trapping when the enum is not one of the known cases. This avoids some of the problems with `@unknown` (such as making it much less important to test), but isn't exactly in the spirit of non-frozen enums, where you *know* there will be more cases in the future.

The following two examples would be equivalent (except perhaps in the form of the diagnostic produced).

```swift
switch! excuse {
case .eatenByPet:
  // …
case .thoughtItWasDueNextWeek:
  // …
}
```

```swift
switch excuse {
case .eatenByPet:
  // …
case .thoughtItWasDueNextWeek:
  // …
unknown:
  fatalError("unknown case in switch: \(excuse)")
}
```


### Testing invalid cases

Another issue with non-frozen enums is that clients cannot properly test what happens when a new case is introduced, almost by definition. Becca Royal-Gordon came up with the idea to have a new type annotation that would allow the creation of an invalid enum value. Since this is only something to use for testing, the initial version of the idea used `@testable` as the spelling for the annotation. The tests could then use a special expression, `#invalid`, to pass this invalid value to a function with a `@testable` enum parameter.

However, this would only work in cases where the action to be taken does not actually depend on the enum value. If it needs to be passed to the original library that owns the enum, or used with an API that is not does not have this annotation, the code still cannot be tested properly.

```swift
override func process(_ transaction: @testable Transaction) {
  switch transaction {
  case .deposit(let amount):
    // …
  case .withdrawal(let amount):
    // …
  default:
    super.process(transaction) // hmm…
  }
}
```

This is an additive feature, so we can come back and consider it in more detail even if we leave it out of the language for now. Meanwhile, the effect can be imitated using an Optional or ImplicitlyUnwrappedOptional parameter.


### Allow enums defined in source packages to be considered non-frozen

The first version of this proposal applied the frozen/non-frozen distinction to all public enums, even those in user-defined libraries. The motivation for this was to allow package authors to add cases to their enums without it being a source-breaking change, meaning it can be done in a minor version release of a library (i.e. one intended to be backwards-compatible). Like deprecations, this can produce new warnings, but not new errors, and it should not (if done carefully) break existing code.

The core team decided that this feature was not worth the disruption and long-term inconvenience it would cause for users who did not care about this capability.


### Leave out `@unknown`

The [initial version][] of this proposal did not include `@unknown`, and required people to use a normal `default` to handle cases added in the future instead. However, many people were unhappy with the loss of exhaustivity checking for `switch` statements, both for enums in libraries distributed as source and enums imported from Apple's SDKs. While this is an additive feature that does not affect ABI, it seems to be one that the community considers a necessary part of a language model that provides non-frozen enums.

  [initial version]: https://github.com/swiftlang/swift-evolution/blob/a773d07ff4beab8b7855adf0ac56d1e13bb7b44c/proposals/0192-non-exhaustive-enums.md


### Mixing `@unknown` with other catch-all cases

The proposal as written forbids having two catch-all cases in the same `switch` where only one is marked `@unknown`. Most people would expect this to have the following behavior:

```swift
switch excuse {
case .eatenByPet:
  // Specific known case
@unknown case _:
  // Any cases not recognized by the compiler
case _:
  // Any other cases the compiler *does* know about,
  // such as .thoughtItWasDueNextWeek
}
```

However, I can't think of an actual use case for this; it's not clear what action one would take in the `@unknown` case that they wouldn't take in the later default case. Furthermore, this becomes a situation where the same code behaves differently before and after recompilation:

1. A new case is added to the HomeworkExcuse enum, say, `droppedInMud`.
2. When using the new version of the library with an existing built client app, the `droppedInMud` case will end up in the `@unknown` part of the `switch`.
3. When the client app *is* recompiled, the `droppedInMud` case will end up in the `case _` case. The compiler will not (and cannot) provide any indication that the behavior has changed.

Without a resolution to these concerns, this feature does not seem worth including in the proposal. It's also additive and has no ABI impact, so if we do find use cases for it in the future we can always add it then.


### Introduce a new declaration kind instead

There have been a few suggestions to distinguish `enum` from some other kind of declaration that acts similarly but allows adding cases:

```swift
choices HomeworkExcuse {
  case eatenByPet
  case thoughtItWasDueNextWeek
}
```

My biggest concern with this is that if we ever *do* expand this beyond the standard library and overlays, it increases the possibility of a library author accidentally publishing a (frozen) `enum` when they meant to publish a (non-frozen) `choices`. As described above, the opposite mistake is one that can be corrected without breaking source compatibility, but this one cannot.

A smaller concern is that both `enum` and `choices` would behave the same when they *aren't* `public`.

Stepping back, increasing the surface area of the language in this way does not seem desirable. Exhaustive switching has been a key part of how Swift enums work, but it is not their only feature. Given how people already struggle with the decision of "struct vs. class" when defining a new type, introducing another pair of "similar but different" declaration kinds would have to come with strong benefits.

My conclusion is that it is better to think of frozen and non-frozen enums as two variants of the same declaration kind, rather than as two different declaration kinds.


### Use protocols instead

Everything you can do with non-frozen enums, you can do with protocols as well, except for:

- exhaustivity checking with `@unknown`
- forbidding others from adding their own "cases"

```swift
protocol HomeworkExcuse {}
struct EatenByPet: HomeworkExcuse {}
struct ThoughtItWasDueNextWeek: HomeworkExcuse {}

switch excuse {
case is EatenByPet:
  // …
case is ThoughtItWasDueNextWeek:
  // …
default:
  // …
}
```

(Associated values are a little harder to get out of the cases, but let's assume we could come up for syntax as well.)

This is a valid model; it's close to what Scala does (as mentioned above), and is independently useful in Swift. However, using this as the only way to get non-frozen enum semantics would lead to a world where `enum` is dangerous for library authors, because `public enum` is now a promise that no new cases will be added. Nothing else in Swift works that way. More practically, getting around this restriction would mean rewriting existing code to use the more verbose syntax of separate types conforming to a common protocol.

(If Swift were younger, perhaps we would consider using protocols for *all* non-imported enums, not just non-frozen ones. But at this point that would be *way* too big a change to the language.)


### Import non-frozen C enums as RawRepresentable structs

The Swift compiler already makes a distinction between plain C enums, enums marked with the `flag_enum` Clang attribute (`NS_OPTIONS`), and enums marked with the `enum_extensibility` Clang attribute (`NS_ENUM`). The first two categories were deemed to not be sufficiently similar to Swift enums and are imported instead as structs. Given that we're most immediately concerned about *C* enums growing new cases (specifically, those in Apple's existing Objective-C SDKs), we could sidestep the problem by importing *all* C enums as structs except for those marked `enum_extensibility(closed)`. However, this doesn't solve the problem for future Swift libraries, while still requiring changes to existing `switch` statements across many many projects, and it doesn't support the exhaustivity checking provided by `unknown`. Furthermore, it would probably be harder to implement high-quality migration support from Swift 4 to Swift 5, since the structs-formerly-enums will look like any other structs imported from C.


### Get Apple to stop adding new cases to C enums

This isn't going to happen, but I thought I'd mention it since it was brought up during discussion. While some may consider this a distasteful use of the C language, it's an established pattern for Apple frameworks and is not going to change.


### "Can there be a kind of open enum where you can add new cases in extensions?"

There is no push to allow adding new cases to an enum from *outside* a library. This use case (no pun intended) is more appropriate for a RawRepresentable struct, where the library defines some initial values as static properties. (You can already switch over struct values in Swift as long as they are Equatable.)
