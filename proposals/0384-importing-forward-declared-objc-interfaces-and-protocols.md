# Importing Forward Declared Objective-C Interfaces and Protocols

* Proposal: [SE-0384](0384-importing-forward-declared-objc-interfaces-and-protocols.md)
* Author: [Nuri Amari](https://github.com/NuriAmari)
* Review Manager: [Tony Allevato](https://github.com/allevato)
* Status: **Implemented (Swift 5.9)**
* Implementation:[apple/swift#61606]( https://github.com/apple/swift/pull/61606)
* Upcoming Feature Flag: `ImportObjcForwardDeclarations` (Enabled in Swift 6 language mode)
* Review: ([pitch](https://forums.swift.org/t/pitch-importing-forward-declared-objective-c-classes-and-protocols/61926)) ([review](https://forums.swift.org/t/se-0384-importing-forward-declared-objective-c-interfaces-and-protocols/62392)) ([acceptance](https://forums.swift.org/t/accepted-se-0384-importing-forward-declared-objective-c-interfaces-and-protocols/62670))

## Introduction

This proposal seeks to improve the usability of existing Objective-C libraries from Swift by reducing the negative
impact forward declarations have on API visibility from Swift. We wish to start synthesizing placeholder types to
represent forward declared Objective-C interfaces and protocols in Swift.

## Motivation

Forward declarations are very common in many existing Objective-C code bases, used often to break cyclic dependencies or to improve build performance.
Unfortunately, when it comes to "importability" into Swift they are quite detrimental.

As it stands, the ClangImporter will fail to import any declaration that references a forward declared type in many common cases. This means a single
forward declared type can render larger portions of an Objective-C API unusable from Swift. For example, the following Objective-C API from the implementation PR
is empty from the Swift perspective (Swift textual interface generated via swift-ide-test):

_Objective-C_
```objective-c
#import <Foundation/Foundation.h>

@class ForwardDeclaredInterface;
@protocol ForwardDeclaredProtocol;

@interface IncompleteTypeConsumer1 : NSObject
@property id<ForwardDeclaredProtocol> propertyUsingAForwardDeclaredProtocol1;
@property ForwardDeclaredInterface *propertyUsingAForwardDeclaredInterface1;
- (id)init;
- (NSObject<ForwardDeclaredProtocol> *)methodReturningForwardDeclaredProtocol1;
- (ForwardDeclaredInterface *)methodReturningForwardDeclaredInterface1;
- (void)methodTakingAForwardDeclaredProtocol1:
    (id<ForwardDeclaredProtocol>)param;
- (void)methodTakingAForwardDeclaredInterface1:
            (ForwardDeclaredInterface *)param;
@end

ForwardDeclaredInterface *CFunctionReturningAForwardDeclaredInterface1();
void CFunctionTakingAForwardDeclaredInterface1(
    ForwardDeclaredInterface *param);

NSObject<ForwardDeclaredProtocol> *CFunctionReturningAForwardDeclaredProtocol1();
void CFunctionTakingAForwardDeclaredProtocol1(
    id<ForwardDeclaredProtocol> param);
```

_Swift_
```swift
class IncompleteTypeConsumer1 : NSObject {
  init!()
}
```
It is possible to fix such issues by importing the definition of the type, either in the Objective-C header or in the consuming Swift file. However, such manual changes make consuming existing libraries more difficult and needlessly increase build times. We wish to make the experience of consuming an Objective-C API with forward declarations from Swift more consistent with the experience of consuming the API from Objective-C.

We have done some work in the past to help diagnose such failures in the ClangImporter:

https://forums.swift.org/t/pitch-improved-clangimporter-diagnostics/52687/17

https://forums.swift.org/t/pitch-lazy-clangimporter-diagnostics-enabled-by-default/54651

We want to build on this work, specifically with respect to forward declarations, this time
fixing a shortcoming of the ClangImporter, not just diagnosing it.

## Proposed solution

We propose the following representation for forward declared Objective-C interfaces and protocols in Swift:

```swift
// @class Foo turns into
@available(*, unavailable, message: “This Objective-C class has only been forward declared; import its owning module to use it”)
class Foo : NSObject {}

// @protocol Bar turns into
@available(*, unavailable, message: “This Objective-C protocol has only been forward declared; import its owning module to use it”)
protocol Bar : NSObjectProtocol {}
```

The idea is to introduce the minimal change that will make Objective-C APIs usable in a predictable safe manner.

The aforementioned Objective-C API with this change looks like this from Swift:

```swift
@available(*, unavailable, message: "This Objective-C class has only been forward-declared; import its owning module to use it")
class ForwardDeclaredInterface {
}
@available(*, unavailable, message: "This Objective-C protocol has only been forward-declared; import its owning module to use it")
protocol ForwardDeclaredProtocol : NSObjectProtocol {
}
class IncompleteTypeConsumer1 : NSObject {
  var propertyUsingAForwardDeclaredProtocol1: ForwardDeclaredProtocol!
  var propertyUsingAForwardDeclaredInterface1: ForwardDeclaredInterface!
  init!()
  func methodReturningForwardDeclaredProtocol1() -> ForwardDeclaredProtocol!
  func methodReturningForwardDeclaredInterface1() -> ForwardDeclaredInterface!
  func methodTakingAForwardDeclaredProtocol1(_ param: ForwardDeclaredProtocol!)
  func methodTakingAForwardDeclaredInterface1(_ param: ForwardDeclaredInterface!)
}
func CFunctionReturningAForwardDeclaredInterface1() -> ForwardDeclaredInterface!
func CFunctionTakingAForwardDeclaredInterface1(_ param: ForwardDeclaredInterface!)
func CFunctionReturningAForwardDeclaredProtocol1() -> ForwardDeclaredProtocol!
func CFunctionTakingAForwardDeclaredProtocol1(_ param: ForwardDeclaredProtocol!)
```

More usage examples can be found in these tests introduced here: https://github.com/apple/swift/pull/61606

## Detailed design

Modifications lie almost exclusively in `ClangImporter` -- specifically in `SwiftDeclConverter`. If asked to convert a
`clang::ObjCInterfaceDecl` or `clang::ObjCProtocolDecl` with no definition `SwiftDeclConverter` will now return a placeholder type instead of
bailing. These placeholder types are as described:

```swift
// @class Foo turns into
@available(*, unavailable, message: “This Objective-C class has only been forward declared; import its owning module to use it”)
class Foo : NSObject {}

// @protocol Bar turns into
@available(*, unavailable, message: “This Objective-C protocol has only been forward declared; import its owning module to use it”)
protocol Bar : NSObjectProtocol {}
```

Permitted usages of these types are intentionally limited. You will be able to use Objective-C and C declarations that refer to these types without issue.
You will be able to pass around instances of these incomplete types from Swift to Objective-C and vice versa.

You’ll also be able to call the set of methods provided by `NSObject` or `NSObjectProtocol` on instances. Notably this assumes
that the backing Objective-C implementation does indeed inherit from / conform to `NSObject`. We are under the impression that
inheriting from `NSObject` is a requirement for Swift - Objective-C interop, but are not sure about protocols conforming to `NSObject`.
We could use some input on these assumptions, and will remove this feature if they turn out to be unsound.

Using the type itself directly in Swift is forbidden by the attached unavailable attribute. This is to keep the impact of the change small and to prevent unsound declarations,
such as declaring a new Swift class that inherits from or conforms to such a type. You will also not be able to create new instances of these types in Swift.

To make these limitations more intuitive to users, a new diagnostic has been added. This diagnostic helps guide users when they attempt to access a member on the incomplete synthesized
type, expecting the complete type. The new diagnostics are shown in the last 4 lines of this example:

```swift
swift-client-fwd-declared.swift:6:7: error: value of type 'Foo' has no member 'sayHello'
myFoo.sayHello()
~~~~~ ^~~~~~~~
<unknown>:0: note: class 'Foo' is a placeholder for a forward declared Objective-C interface and may be missing members; import the definition to access the complete interface
foo-bar-consumer.h:3:1: note: interface 'Foo' forward declared here
@class Foo;
^
```

In Swift 5.x, the feature is gated behind the upcoming feature flag `ImportObjcForwardDeclarations`. This flag is on by default for Swift version 6 and onwards.

The flag is always disabled in the REPL, as allowing it currently leads to confusing behavior. In the REPL, the environment in terms of whether a complete definition of some type Foo is available can change during execution. These examples show some of the confusing behavior: 

```swift
import IncompleteFoo
let foo = FunctionReturningFoo()
FunctingTakingFoo(foo)
import CompleteFoo // This has no effect as far as Swift's view of type 'Foo' is concerned
let completeFoo = Foo() // error type 'IncompleteFoo.Foo' has no initializer
foo.methodOnCompleteFoo() // error no member 'methodOnCompleteFoo' on type 'IncompleteFoo.Foo'
```

```swift
import IncompleteFoo
import CompleteFoo
let completeFoo = Foo() // This time it works because the cache was not populated until after CompleteFoo was imported
foo.methodOnCompleteFoo()  // Works as well
```

This happens because `ClangImporter` is lazy and caches imports, once the placeholder is synthesized (when only the incomplete definition is
available), the complete definition will never be imported. Without an understanding of `ClangImporter`, it is impossible to understand why
the statements between the two imports have an effect on those after. We have made attempts at solving these issues, but no satisfactory solution
was found. The details of these attempts are listed in the alternatives considered section. 

The [existing ClangImporter diagnostics](https://forums.swift.org/t/pitch-improved-clangimporter-diagnostics/52687) will still properly warn users about forward declaration limitations, with or with out the feature enabled.

## Source compatibility

This change does have the potential to break source compatibility, since
`ClangImporter` will now import declarations that were previously dropped.
That said, the impact should rare. The current implementation already passes
the source compatibility suite and the source kit stress test.

As mentioned, to address source compatibility issues, the feature is gated behind
a flag for Swift 5 and before, and enabled by default in Swift 6.

## Effect on ABI stability

This change has no affect on ABI.

## Effect on API resilience

This change does not influence the ABI characteristics
of public APIs.

## Alternatives considered

The unavailable attribute attached to synthesized declarations might be more restrictive than necessary.
We considered the introduction of a new attribute to denote an incomplete type. This attribute would be used to prevent
operations that are not suitable for an incomplete type (ie. declaring a Swift type that conforms to an incomplete protocol).

Allowing more use of the type however increases the complexity of the problem, and the likelihood of source compatibility issues.
For example, if the synthesized type is not file private, issues could arise with different files having different representations
of the type depending on which modules they import.

We also attempted to make this implementation usable from the REPL. As mentioned, we had issues with `ClangImporter` caching
imports. We must invalidate the cache, at the very least to allow the new definition of type 'Foo' to be imported. This creates issues
where two incompatible representations of the type (`IncompleteFoo.Foo` and `CompleteFoo.Foo`) are in context at the same time.
To solve these issues we attempted the following:

1. Invalidate all direct or indirect references to `IncompleteFoo.Foo` and replace them with `CompleteFoo.Foo`:

Issues:

- We do not know how to safely patch the type of existing instances at runtime (ex: local variable `foo` in the snippet above)
- Any imported declarations that depended upon a type, whose definition was incomplete at the time of import but is now complete, need reimporting
    - We need new machinery in `ClangImporter` to support smart cache invalidation like this
        - For all cached declarations, we need to keep track of the "completeness" of all types we depend on at the time of caching
    - A simple cache invalidation heuristic such as "don't cache anything that depends upon an incomplete type" won't work either
        - Besides the significant performance cost, the typechecker compares type declarations using pointer equality, expansion of the typechecker to recognize two imports of the same Clang declaration as equivalent would be required

2. Convince the typechecker that `IncompleteFoo.Foo` and `CompleteFoo.Foo` are the same, *in all usages*:

Issues:

- It seems achievable to teach the typechecker that `IncompleteFoo.Foo` and `CompleteFoo.Foo` can be implicitly converted between one another, but that is not enough
    - Any typechecking contraint that `CompleteFoo.Foo` satisfies must also be satisfied by `IncompleteFoo.Foo`.
    - It might be possible to "inject" an extension into the REPL's context that adds any required methods, computed properties and protocol conformances to `IncompleteFoo.Foo`. However, it is not possible to add new inheritances. 

We believe that given these issues, it is better to disable the feature in the REPL entirely rather than provide a confusing experience. We believe that this
proposal should advance in some form regardless. Forward declarations are a huge problem for consumers of large Objective-C codebases looking to transition to Swift.
The LLDB / REPL experience already diverges from the compiled experience, is relatively niche, and we are not making the REPL experience any worse. Furthermore,
unlike in large scale projects, the existing solution of "import the definition" is perfectly reasonable for small REPL sessions. If this divergence between the
REPL and compiled experience is unacceptable, we think it could be reasonable to gate the feature behind a flag. This flag could potentially also be enabled in
the REPL, but with appropriate warnings and "at your own risk".

That said, it would be great if this feature could come to the REPL eventually, but the REPL should not stand in the way of progress in the compiled experience.

## Acknowledgments

Thank you to @drodriguez and [@rmaz](https://github.com/rmaz) for helping develop the idea.
