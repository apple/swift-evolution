# Limiting `@objc` inference

* Proposal: [SE-0160](0160-objc-inference.md)
* Author: [Doug Gregor](https://github.com/DougGregor)
* Review Manager: [Chris Lattner](https://github.com/lattner)
* Status: **Implemented (Swift 4.0)**
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-se-0160-limiting-objc-inference/5621) 
* Previous Revisions: [1](https://github.com/swiftlang/swift-evolution/blob/0389b1f49fc55b1a898701c549ce89738307b9fc/proposals/0160-objc-inference.md)
* Implementation: [apple/swift#8379](https://github.com/apple/swift/pull/8379)
* Bug: [SR-4481](https://bugs.swift.org/browse/SR-4481)

## Introduction

One can explicitly write `@objc` on any Swift declaration that can be
expressed in Objective-C. As a convenience, Swift also *infers*
`@objc` in a number of places to improve interoperability with
Objective-C and eliminate boilerplate. This proposal scales back the
inference of `@objc` to only those cases where the declaration *must*
be available to Objective-C to maintain semantic coherence of the model,
e.g., when overriding an `@objc` method or implementing a requirement
of an `@objc` protocol. Other cases currently supported (e.g., a
method declared in a subclass of `NSObject`) would no longer infer
`@objc`, but one could continue to write it explicitly to produce
Objective-C entry points.

Swift-evolution thread: [here](https://forums.swift.org/t/pitch-align-objc-inference-with-the-semantic-model/2563) and [here](https://forums.swift.org/t/proposal-draft-limiting-objc-inference/4812)

## Motivation

There are several observations motivating this proposal. The first is
that Swift's rules for inference of `@objc` are fairly baroque, and it
is often unclear to users when `@objc` will be inferred. This proposal
seeks to make the inference rules more straightforward. The second
observation is that it is fairly easy to write Swift classes that
inadvertently cause Objective-C selector collisions due to
overloading, e.g.,

```swift
class MyNumber : NSObject {
  init(_ int: Int) { }
  init(_ double: Double) { } // error: initializer 'init' with Objective-C selector 'init:' 
      // conflicts with previous declaration with the same Objective-C selector
}
```

The example above also illustrates the third observation, which is
that code following the [Swift API Design
Guidelines](https://swift.org/documentation/api-design-guidelines/)
will use Swift names that often translate into very poor Objective-C
names that violate the [Objective-C Coding Guidelines for
Cocoa](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CodingGuidelines/CodingGuidelines.html). Specifically,
the Objective-C selectors for the initializers above should include a noun
describing the first argument, e.g., `initWithInteger:` and
`initWithDouble:`, which requires explicit `@objc` annotations anyway:

```swift
class MyNumber : NSObject {
  @objc(initWithInteger:) init(_ int: Int) { }
  @objc(initWithDouble:) init(_ double: Double) { }
}
```

The final observation is that there is a cost for each Objective-C
entry point, because the Swift compiler must create a "thunk" method
that maps from the Objective-C calling convention to the Swift calling
convention and is recorded within Objective-C metadata. This increases
the size of the binary (preliminary tests on some Cocoa[Touch] apps
found that 6-8% of binary size was in these thunks alone, some of
which are undoubtedly unused), and can have some impact on load time
(the dynamic linker has to sort through the Objective-C metadata for
these thunks).

## Proposed solution

The proposed solution is to limit the inference of `@objc` to only
those places where it is required for semantic consistency of the
programming model. Then, add some class-level and extension-level
annotations to reduce boilerplate for cases where one wants to
enable/disable `@objc` inference more widely.

### Constructs that (still) infer `@objc`

Specifically, `@objc` will continue to be inferred
for a declaration when:

* The declaration is an override of an `@objc` declaration, e.g.,

  ```swift
  class Super {
      @objc func foo() { }
  }

  class Sub : Super {
      /* inferred @objc */
      override func foo() { }
  }
  ```

  This inference is required so that Objective-C callers to the method
  `Super.foo()` will appropriately invoke the overriding method
  `Sub.foo()`.

* The declaration satisfies a requirement of an `@objc` protocol,
  e.g.,

  ```swift
  @objc protocol MyDelegate {
      func bar()
  }

  class MyClass : MyDelegate {
      /* inferred @objc */
      func bar() { }
  }
  ```

  This inference is required because anyone calling
  `MyDelegate.bar()`, whether from Objective-C or Swift, will do so
  via an Objective-C message send, so conforming to the protocol
  requires an Objective-C entry point.

* The declaration has the `@IBAction` or `@IBOutlet` attribute. This
  inference is required because the interaction with Interface Builder
  occurs entirely through the Objective-C runtime, and therefore
  depends on the existence of an Objective-C entrypoint.

* The declaration has the `@NSManaged` attribute. This inference is
  required because the interaction with CoreData occurs entirely
  through the Objective-C runtime, and therefore depends on the
  existence of an Objective-C entrypoint.

The list above describes cases where Swift 3 already performs
inference of `@objc` and will continue to do so if this proposal is
accepted.

### Additional constructs that will infer `@objc`

These are new cases that *should* infer `@objc`, but currently don't
in Swift. `@objc` should be inferred when:

* The declaration has the `@GKInspectable` attribute. This inference
  is required because the interaction with GameplayKit occurs entirely
  through the Objective-C runtime.

* The declaration has the `@IBInspectable` attribute. This inference
  is required because the interaction with Interface Builder occurs entirely
  through the Objective-C runtime.

### `dynamic` no longer infers `@objc`

A declaration that is `dynamic` will no longer infer `@objc`. For example:

```swift
class MyClass {
  dynamic func foo() { }       // error: 'dynamic' method must be '@objc'
  @objc dynamic func bar() { } // okay
}
```

This change is intended to separate current implementation
limitations from future language evolution: the current
implementation supports `dynamic` by always using the Objective-C
message send mechanism, allowing replacement of `dynamic`
implementations via the Objective-C runtime (e.g., `class_addMethod`
and `class_replaceMethod`). In the future, it is plausible that the
Swift language and runtime will evolve to support `dynamic` without
relying on the Objective-C runtime, and it's important that we leave
the door open for that language evolution.

This change therefore does two things. First, it makes it clear that
the `dynamic` behavior is tied to the Objective-C runtime. Second,
it means that well-formed Swift 4 code will continue to work in the
same way should Swift gain the ability to provide `dynamic` without
relying on Objective-C: at that point, the method `foo()` above will
become well-formed, and the method `bar()` will continue to work as
it does today through the Objective-C runtime. Indeed, this change
is the right way forward even if Swift never supports `dynamic` in
its own runtime, following the precedent of
[SE-0070](https://github.com/swiftlang/swift-evolution/blob/master/proposals/0070-optional-requirements.md),
which required the Objective-C-only protocol feature "optional
requirements" to be explicitly marked with `@objc`.

### `NSObject`-derived classes no longer infer `@objc`

A declaration within an `NSObject`-derived class will no longer infer
`@objc`. For example:

```swift
class MyClass : NSObject {
  func foo() { } // not exposed to Objective-C in Swift 4
}
```

This is the only major change of this proposal, because it means
that a large number of methods that Swift 3 would have exposed to
Objective-C (and would, therefore, be callable from Objective-C code
in a mixed project) will no longer be exposed. On the other hand,
this is the most unpredictable part of the Swift 3 model, because
such methods infer `@objc` only when the method can be expressed in
Objective-C. For example:

```swift
extension MyClass {
  func bar(param: ObjCClass) { } // exposed to Objective-C in Swift 3; not exposed by this proposal
  func baz(param: SwiftStruct) { } // not exposed to Objective-C
}
```

With this proposal, neither method specifies `@objc` nor is either
required by the semantic model to expose an Objective-C entrypoint,
so they don't infer `@objc`: there is no need to reason about the
type of the parameter's suitability in Objective-C.

### Re-enabling `@objc` inference within a class hierarchy

Some libraries and systems still depend greatly on the Objective-C
runtime's introspection facilities. For example, XCTest uses
Objective-C runtime metadata to find the test cases in `XCTestCase`
subclasses. To support such systems, introduce a new attribute for
classes in Swift, spelled `@objcMembers`, that re-enables `@objc`
inference for the class, its extensions, its subclasses, and (by
extension) all of their extensions. For example:

```swift
@objcMembers
class MyClass : NSObject {
  func foo() { }             // implicitly @objc

  func bar() -> (Int, Int)   // not @objc, because tuple returns
      // aren't representable in Objective-C
}

extension MyClass {
  func baz() { }   // implicitly @objc
}

class MySubClass : MyClass {
  func wibble() { }   // implicitly @objc
}

extension MySubClass {
  func wobble() { }   // implicitly @objc
}
```

This will be paired with an Objective-C attribute, spelled
`swift_objc_members`, that allows imported Objective-C classes to be
imported as `@objcMembers`:

```objective-c
__attribute__((swift_objc_members))
@interface XCTestCase : XCTest
/* ... */
@end
```

will be imported into Swift as:

```swift
@objcMembers
class XCTestCase : XCTest { /* ... */ }
```

### Enabling/disabling `@objc` inference within an extension

There might be certain regions of code for which all of (or none of)
the entry points should be exposed to Objective-C. Allow either
`@objc` or `@nonobjc` to be specified on an `extension`. The `@objc`
or `@nonobjc` will apply to any member of that extension that does not
have its own `@objc` or `@nonobjc` annotation. For example:

```swift
class SwiftClass { }

@objc extension SwiftClass {
  func foo() { }            // implicitly @objc
  func bar() -> (Int, Int)  // error: tuple type (Int, Int) not
      // expressible in @objc. add @nonobjc or move this method to fix the issue
}

@objcMembers
class MyClass : NSObject {
  func wibble() { }    // implicitly @objc
}

@nonobjc extension MyClass {
  func wobble() { }    // not @objc, despite @objcMembers
}
```

Note that `@objc` on an extension provides less-surprising behavior
than the implicit `@objc` inference of Swift 3, because it indicates
the intent to expose *everything* in that extension to Objective-C. If
some member within that extension cannot be exposed to Objective-C,
such as `SwiftClass.bar()`, the compiler will produce an error.

## Side benefit: more reasonable expectations for `@objc` protocol extensions

Users are often surprised to realize that extensions of `@objc`
protocols do not, in fact, produce Objective-C entrypoints:

```swift
@objc protocol P { }

extension P {
  func bar() { }
}

class C : NSObject, P { }

let c = C()
print(c.responds(to: Selector("bar"))) // prints "false"
```

The expectation that `P.bar()` has an Objective-C entry point is set
by the fact that `NSObject`-derived Swift classes do implicitly create
Objective-C entry points for declarations within class extensions when
possible, but Swift does not (and, practically speaking, cannot) do
the same for protocol extensions.

A previous mini-proposal [discussed
here](https://forums.swift.org/t/mini-proposal-require-nonobjc-on-members-of-objc-protocol-extensions/905)
suggested requiring `@nonobjc` for members of `@objc` protocol
extensions. However, limiting inference of `@objc` eliminates the
expectation itself, addressing the problem from a different angle.

## Source compatibility

The two changes that remove inference of `@objc` are both
source-breaking in different ways. The `dynamic` change mostly
straightforward:

* In Swift 4 mode, introduce an error when a `dynamic` declaration
  does not explicitly state `@objc` (or infer it based on one of the
  `@objc` inference rules that still applies in Swift 4), with a
  Fix-It to add the `@objc`.

* In Swift 3 compatibility mode, continue to infer `@objc` for
  `dynamic` methods. However, introduce a warning that such code will
  be ill-formed in Swift 4, along with a Fix-It to add the
  `@objc`.

* A Swift 3-to-4 migrator could employ the same logic as Swift 3
  compatibility mode to update `dynamic` declarations appropriately.

The elimination of inference of `@objc` for declarations in `NSObject`
subclasses is more complicated. Considering again the three cases:

* In Swift 4 mode, do not infer `@objc` for such declarations.
  Source-breaking changes that will be introduced include:

    * If `#selector` or `#keyPath` refers to one such declaration, an
      error will be produced on previously-valid code that the
      declaration is not `@objc`. In most cases, a Fix-It will suggest
      the addition of `@objc`.

    * If a message is sent to one of these declarations via
      `AnyObject`, the compiler may produce an error (if no `@objc`
      entity by that name exists anywhere) or a failure might occur at
      runtime (if another, unrelated `@objc` entity exists with that
      same name). For example:

      ```swift
      class MyClass : NSObject {
        func foo() { }
        func bar() { }
      }

      class UnrelatedClass : NSObject {
        @objc func bar() { }
      }

      func test(object: AnyObject) {
        object.foo?()  // Swift 3: can call method MyClass.foo()
                       // Swift 4: compiler error, no @objc method "foo()"
        object.bar?()  // Swift 3: can call MyClass.bar() or UnrelatedClass.bar()
                       // Swift 4: can only call UnrelatedClass.bar()
      }
      ```

    * If one of these declarations is written in a class extension and
      is overridden, the override will produce an error in Swift 4
      because Swift's class model does not support overriding
      declarations introduced in class extensions. For example:

      ```swift
      class MySuperclass : NSObject { }

      extension MySuperclass {
        func extMethod() { } // implicitly @objc in Swift 3, not in Swift 4
      }

      class MySubclass : MySuperclass {
        override func extMethod() { }   // Swift 3: okay
           // Swift 4: error "declarations in extensions cannot override yet"
      }
      ```

    * Objective-C code in mixed-source projects won't be able to call
      these declarations. Most problems caused by this will result in
      warnings or errors from the Objective-C compiler (due to
      unrecognized selectors); some may only be detected at runtime,
      similarly to the `AnyObject` case described above.

    * Other tools and frameworks that rely on the presence of
      Objective-C entrypoints (e.g., via strings) but do not make use
      of Swift's facilities for referring to them will fail. This case
      is particularly hard to diagnose well, and failures of this sort
      are likely to cause runtime failures (e.g., unrecoignized
      selectors) that only the developer can diagnose and correct.

* In Swift 3 compatibility mode, continue to infer `@objc` for these
  declarations. We can warn about uses of the `@objc` entrypoints in
  cases where the `@objc` is inferred in Swift 3 but will not be in
  Swift 4.

* A Swift 3-to-4 migrator is the hardest part of the story. The
  migrator should have a switch: a "conservative" option and a
  "minimal" option.

  * The "conservative" option (which is the best default) simply adds
  explicit `@objc` annotations to every entity that was implicitly
  `@objc` in Swift 3 but would not implicitly be `@objc` in Swift
  4. Migrated projects won't get the benefits of the more-limited
  `@objc` inference, but they will work out-of-the-box.

  * The "minimal" option attempts to only add `@objc` in places where
    it is needed to maintain the semantics of the program. It would be
    driven by the diagnostics mentioned above (for `#selector`,
    `#keyPath`, `AnyObject` messaging, and overrides), but some manual
    intervention will be involved to catch the runtime cases. More
    discussion of the migration workflow follows.


## "Minimal" migration workflow

To migrate a Swift 3 project to Swift 4 without introducing spurious
Objective-C entry points, we can apply the following workflow:

1. In Swift 4 mode, address all of the warnings about uses of
declarations for which `@objc` was inferred based on the deprecated
rule.
2. Set the environment variable `SWIFT_DEBUG_IMPLICIT_OBJC_ENTRYPOINT`
to a value between 1 and 3 (see below) and test the application. Clean
up any "deprecated `@objc` entrypoint` warnings.
3. Migrate to Swift 4 with "minimal" migration, which at this point
will only add `@objc` to explicitly `dynamic` declarations.

The following subsections describe this migration in more detail.

### Step 1: Address compiler warnings

The compiler can warn about most instances of the source-breaking
changes outlined above. Here is an example that demonstrates the
warnings in Swift code, all of which are generated by the Swift
compiler:

```swift
class MyClass : NSObject {
  func foo() { }

  var property: NSObject? = nil

  func baz() { }
}

extension MyClass {
  func bar() { }
}

class MySubClass : MyClass {
  override func foo() { }    // okay

  override func bar() { }    // warning: override of instance method
      // 'bar()' from extension of 'MyClass' depends on deprecated inference
      //  of '@objc'
}

func test(object: AnyObject, mine: MyClass) {
  _ = #selector(MyClass.foo)     // warning: argument of `#selector`
      // refers to instance method `foo()` in `MyClass` that uses deprecated
      // `@objc` inference

  _ = #keyPath(MyClass.property) // warning: argument of '#keyPath'
      // refers to property 'property' in 'MyClass' that uses deprecated
      // `@objc` inference

  _ = object.baz?()              // warning: reference to instance
      // method 'baz()' of 'MyClass' that uses deprecated `@objc` inference
}
```

For mixed-source projects, the Swift compiler will annotate the
generated Swift header with "deprecation" attributes, so that any
references to those declarations *from Objective-C code* will also
produce warnings. For example:

```objective-c
#import "MyApp-Swift.h"

void test(MyClass *mine) {
  [mine foo];   // warning: -[MyApp.MyClass foo] uses deprecated
      // '@objc' inference; add '@objc' to provide an Objective-C entrypoint
}
```

### Step 2: Address (opt-in) runtime warnings

Swift 3 compatibility mode augments each of the Objective-C
entrypoints introduced based on the deprecated `@objc` inference rules
with a call to a new runtime function
`swift_objc_swift3ImplicitObjCEntrypoint`. This entry point can be
used in two ways to find cases where an Objective-C entry point that
will be eliminated by the migration to Swift 4:

* In a debugger, one can set a breakpoint on
`swift_objc_swift3ImplicitObjCEntrypoint` to catch specific cases
where the Objective-C entry point is getting called.

* One can set the environment variable
`SWIFT_DEBUG_IMPLICIT_OBJC_ENTRYPOINT` to one of three different
values to cause the Swift runtime to log uses of these Objective-C
entry points:
  1. Log calls to these entry points with a message such as:
  ```***Swift runtime: entrypoint -[MyApp.MyClass foo] generated by implicit @objc inference is deprecated and will be removed in Swift 4```
  2. Log (as in #1) and emit a backtrace showing how that Objective-C
  entry point was invoked.
  3. Log with a backtrace (as in #2), then crash. This last stage is
  useful for automated testing leading up to the migration to Swift 4.

Testing with logging enabled should uncover uses of the Objective-C
entry points that use the deprecated rules. As explicit `@objc` is
added to each case, the runtime warnings will go away.

### Step 3: Migrate to Swift 4

At this point, one can migrate to Swift 4. Building in Swift 4 will
remove the Objective-C entry points for any remaining case where
`@objc` was inferred based on the deprecated rules.

## Effect on ABI stability

This proposal has no effect on the Swift ABI, because it only concerns the Objective-C entry points for Swift entities, which have always been governed by the already-set-in-stone Objective-C ABI. Whether a particular Swift entity is `@objc` or not does not affect its Swift ABI.

## Effect on API resilience

The [library evolution document](https://github.com/apple/swift/blob/master/docs/LibraryEvolution.rst) notes that adding or removing `@objc` is not a resilient API change. Therefore, changing the inference behavior of `@objc` doesn't really have an impact on API resilience beyond the normal concerns about errors of omission: prior to this proposal, forgetting to add `@nonobjc` meant that an API might be stuck vending an Objective-C entry point it didn't want to expose; with this proposal, forgetting to add `@objc` means that an API might fail to be usable from Objective-C. The latter problem, at least, can be addressed by exposing an additional entrypoint. Moreover, adding an Objective-C entrypoint is "less" ABI-breaking that removing an Objective-C entrypoint, because the former is only breaking for `open` or `dynamic` members.

## Alternatives considered

Aside from the obvious alternative of "do nothing", there are ways to
address some of the problems called out in the
[Motivation](#motivation) section without eliminating inference in the
cases we're talking about, or to soften the requirements on some
constructs.

### Mangling Objective-C selectors
Some of the problems with Objective-C selector collisions could be
addressed by using "mangled" selector names for Swift-defined
declarations. For example, given:

```swift
class MyClass : NSObject {
  func print(_ value: Int) { }
}
```

Instead of choosing the Objective-C selector "print:" by default,
which is likely to conflict, we could use a mangled selector name like
`__MyModule__MyClass__print__Int:` that is unlikely to conflict with
anything else in the program. However, this change would also be
source-breaking for the same reasons that restricting `@objc`
inference is: dynamic behavior that constructs Objective-C selectors
or tools outside of Swift that expect certain selectors will break at
run-time.

### Completely eliminating `@objc` inference

Another alternative to this proposal is to go further and completely
eliminate `@objc` inference. This would simplify the programming model
further---it's exposed to Objective-C only if it's marked
`@objc`---but at the cost of significantly more boilerplate for
applications that use Objective-C frameworks. For example:

```swift
class Sub : Super {
  @objc override func foo() { }  // @objc is now required
}

class MyClass : MyDelegate {
  @objc func bar() { }  // @objc is now required
}
```

I believe that this proposal strikes the right balance already, where
`@objc` is inferred when it's needed to maintain the semantic model,
and can be explicitly added to document those places where the user is
intentionally exposing an Objective-C entrypoint for some
reason. Thus, explicitly writing `@objc` indicates intent without
creating boilerplate.

# Acknowledgments

Thanks to Brian King for noting the inference of `dynamic` and its
relationship to this proposal.

# Revision history

[Version 1](https://github.com/swiftlang/swift-evolution/blob/0389b1f49fc55b1a898701c549ce89738307b9fc/proposals/0160-objc-inference.md)
of this proposal did not include the use of `@objcMembers` on classes
or the use of `@objc`/`@nonobjc` on extensions to mass-annotate.
