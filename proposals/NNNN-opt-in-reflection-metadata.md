# Opt-In Reflection metadata

* Proposal: [SE-NNNN](NNNN-opt-in-reflection-metadata.md)
* Authors: [Max Ovtsin](https://github.com/maxovtsin)
* Review Manager: TBD
* Status: **Awaiting implementation**
* Implementation: [apple/swift#34199](https://github.com/apple/swift/pull/34199)


## Introduction
Reflection can be a useful thing to create convenient and concise APIs for libraries. This proposal seeks to improve the safety of such APIs and to tackle the binary size problem by introducing a mechanism of selectively keeping reflection metadata only for types that need it and dead strip it for all others. Developers will gain an opportunity to express a requirement to have reflection metadata in source-code.

Swift-evolution thread: [Discussion thread topic for that proposal](https://forums.swift.org/t/pitch-2-opt-in-reflection-metadata/41696)

## Motivation
*API safety*
Currently, it is impossible for APIs to express reflection metadata requirements which can lead to unexpected behavior. For example, SwiftUI implementation uses reflection metadata from user modules to trigger re-rendering of the view hierarchy when a state has changed. If by some reason a user module was compiled with the disabled metadata, changing the state won't trigger that behavior and will cause inconsistency between state and representation which will make such API less safe since it becomes a runtime issue rather than compile-time one.

*Dead Stripping*
The compiler can’t statically determine the uses of reflection metadata between the boundaries of the main binary and frameworks. Therefore all symbols must be no_dead_strippable just in case other frameworks uses that metadata. Hereupon the linker can't strip such symbols which can lead to an increase of binary size especially in cases when API's surface is big.


## Proposed solution
Introducing a new attribute `@reflectable` for nominal types will help to solve two issues mentioned above at once:

* Firstly, this will allow libraries to express a requirement to have reflection metadata for types on the source-code level which will make such APIs safer. If a declaration is marked with `@reflectable` or inherits it, but the reflection metadata is disabled, the compiler will emit a compile-time error suggesting to enable the metadata emission.
* Secondly, this will help to reduce the binary size overhead from the metadata that is not used at runtime and possibly lead to other optimizations. The reflection metadata for nominal declarations (classes, structs, enums, and protocols) marked with `@reflectable` will be forced to stay at the binary while the other can be dead stripped by DCE or the linker if are not used within a module.

*Study Case:*
SwiftUI Framework:
```swift
@reflectable
public protocol SwiftUI.View {}
```
User module:
```swift
import SwiftUI

struct SomeView: SwiftUI.View {
    var body: some View {        
        Text("Hello, World!")
            .frame(...)    
    }
}

struct SomeModel {}

window.contentView = NSHostingView(rootView: SomeView())
```
Reflection metadata for `SomeView` will be emitted with the no_dead_strip label because it inherits the attribute from `SwiftUI.View` protocol, while `SomeModel` metadata might be dead-stripped unless it’s referenced from somewhere else. If an API's user module gets compiled with the reflection metadata disabled, the compiler will emit a error.

## Detailed design
One more level of reflection metadata is introduced in addition to two existing ones:

1. Full Reflection metadata is enabled by default.
2. Opt-In reflection metadata is enabled explicitly with `-enable-opt-in-reflection-metadata` flag.
3. Reflection metadata is completely disabled with `-disable-reflection-metadata` flag.


A new attribute for nominal type declarations `@reflectable` marks types for which the compiler will emit reflection metadata symbols with `no_dead_strip` directive while other reflection symbols might be dead stripped by the linker in case they are not in use within a module.

The attribute should be inheritable from parents for classes and protocols:

Example 1:
```swift
@reflectable
protocol Ref {}
// Class A will inherit the attribute from the protocol conformance.
class A: Ref {}
```
Example 2:
```swift
@reflectable
class B {}
// Class C will inherit the protocol from the parent class.
class C: B {}
```

Introducing a new flag gating the feature will allow us to safely roll out it and avoid breakages of the existing code. For those modules that get compiled with fully enabled metadata, nothing will change (all symbols will stay). For modules that have the metadata disabled, but are users of reflectable API, the compiler will emit the error enforcing the guarantee. Only explicit passing the new flag will allow developers to opt-in to new behavior.


## Source compatibility
The change won’t break source compatibility due to the purely additive nature of the change.


## Effect on ABI stability
The proposal doesn’t change the existing ABI but introduces new guarantees between libraries and clients.


## Effect on API resilience
Adding/removing the `@reflectable` attribute won’t affect API resilience and break any existing guarantees. However, it’s still possible that some reflection metadata might be unavailable for cases when a library was updated with the new attribute, but a client wasn’t recompiled against a new version of the library.


## Future direction
If this proposal gets accepted, it will allow extending the Swift Standard library with a new protocol Reflectable that will be marked with the new attribute. All existing APIs that require reflection metadata might be extended to accept objects conforming to that protocol. It will help to improve the safety of such API because the compiler will ensure that for all types, conforming to the Reflectable protocol, the required reflection metadata is discoverable at runtime.

```swift
@reflectable
public protocol Reflectable {}

public struct Mirror {
    // A new constructor that takes an object conforming to Reflectable protocol.
    public init<Subject : Reflectable>(
        reflecting subject: Subject
        ) {
        // ...
    }
}
```

## Alternatives considered
An implementation with a magic protocol also was considered. However, It seems that an attribute will be a more generic approach in this case. An attribute is more flexible and doesn't require conformance for cases when it's not necessary to introduce a new protocol (Swift can be written not only in protocol-oriented style). It's also possible that in the future, an attribute will gain some additional parameters to setup IRGen. For instance, the level of emitted reflection and/or turn on/off names.



