# Limiting `@objc` inference

* Proposal: [SE-0160](https://github.com/apple/swift-evolution/blob/master/proposals/0160-objc-inference.md)
* Author: [Doug Gregor](https://github.com/DougGregor)
* Status: **Active review (March 21...28, 2017)**
* Review manager: [Chris Lattner](http://github.com/lattner)

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

Swift-evolution thread: [here](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20160509/017308.html) and [here](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20170102/029909.html)

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
programming model. 

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
[SE-0070](https://github.com/apple/swift-evolution/blob/master/proposals/0070-optional-requirements.md),
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
here](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20160104/005312.html)
suggested requiring `@nonobjc` for members of `@objc` protocol
extensions. However, limiting inference of `@objc` eliminates the
expectation itself, addressing the problem from a different angle.

## Source compatibility

The two changes that remove inference of `@objc` are both
source-breaking in different ways. The `dynamic` change mostly
straightforward:

* In Swift 4 mode, introduce an error that when a `dynamic`
  declaration does not explicitly state `@objc`, with a Fix-It to add
  the `@objc`.

* In Swift 3 compatibility mode, continue to infer `@objc` for
  `dynamic` methods. However, introduce a warning that such code will
  be ill-formed in Swift 4, along with a Fix-It to add the
  `@objc`.

* A Swift 3-to-4 migrator could employ the same logic as Swift 3
  compatibility mode to update `dynamic` declarations appropriately.



The elimination of inference of `@objc` for declarations in `NSObject`
subclasses is more complicated. Considering again the three cases:

* In Swift 4 mode, do not infer `@objc` for such
  declarations. Source-breaking changes that will be introduced include:

    * If `#selector` or `#keyPath` refers to one such declaration, an
      error will be produced on previously-valid code that the
      declaration is not `@objc`. In most cases, a Fix-It will suggest
      the addition of `@objc`.

    * The lack of `@objc` means that Objective-C code in mixed-source
      projects won't be able to call these declarations. Most problems
      caused by this will result in warnings or errors from the
      Objective-C compiler (due to unrecognized selectors), but some
      might only be detected at runtime. These latter cases will be
      hard-to-detect.

    * Other tools and frameworks that rely on the presence of
      Objective-C entrypoints but do not make use of Swift's
      facilities for referring to them will fail. This case is
      particularly hard to diagnose well, and failures of this sort
      are likely to cause runtime failures that only the developer can
      diagnose and correct.

* In Swift 3 compatibility mode, continue to infer `@objc` for these
  declarations. When `@objc` is inferred based on this rule, modify
  the generated header (i.e., the header used by Objective-C code to
  call into Swift code) so that the declaration contains a
  "deprecated" attribute indicating that the Swift declaration should
  be explicitly marked with `@objc`. For example:

  ```swift
  class MyClass : NSObject {
    func foo() { }
  }
  ```

  will produce a generated header that includes:

  ```objc
  @interface MyClass : NSObject
  -(void)foo NS_DEPRECATED("MyClass.foo() requires an explicit `@objc` in Swift 4");
  @end
  ```

  This way, any reference to that declaration from Objective-C code
  will produce a warning about the deprecation. Users can silence the
  warning by adding an explicit `@objc`.

* A Swift 3-to-4 migrator is the hardest part of the story. Ideally,
  the migrator to only add `@objc` in places where it is needed, so
  that we see some of the expected benefits of code-size
  reduction. However, there are two problems with doing so:

  1. Some of the uses that imply the need to add `@objc` come from
  Objective-C code, so a Swift 3-to-4 migrator would also need to
  compile the Objective-C code (possibly with a modified version of
  the Objective-C compiler) and match up the "deprecated" warnings
  mentioned in the Swift 3 compatibility mode bullet with Swift
  declarations.

  2. The migrator can't reason about dynamically-constructed selectors
  or the behavior of other tools that might directly use the
  Objective-C runtime, so failing to add a `@objc` will lead to
  migrated programs that compile but fail to execute correctly.

### Overriding of declarations introduced in class extensions

Swift's class model doesn't support overriding of declarations
introduced in class extensions. For example, the following code
produces an amusing error message on the override:

```swift
class MySuperclass { }

extension MySuperclass {
  func extMethod() { }
}

class MySubclass : MySuperclass { }

extension MySubclass {
  override func extMethod() { }   // error: declarations in extensions cannot override yet
}
```

However, this *does* work in Swift 3 when the method is `@objc`, e.g.,

```swift
class MySuperclass { }

extension MySuperclass {
  @objc func extMethod() { }
}

class MySubclass : MySuperclass { }

extension MySubclass {
  override func extMethod() { }   // okay! Objective-C message dispatch allows this
}
```

Removing `@objc` inference for `NSObject` subclasses will therefore
break this correct Swift 3 code:

```swift
class MySuperclass { }

extension MySuperclass : NSObject {
  func extMethod() { } // implicitly @objc in Swift 3, not @objc in Swift 4
}

class MySubclass : MySuperclass { }

extension MySubclass {
  override func extMethod() { }   // okay in Swift 3, error in Swift 4: declarations in extensions cannot override yet
}
```

There are several potential solutions to this problem, but both are
out-of-scope for this particular proposal:

1. Require that a non-`@objc` declaration in a class extension by
explicitly declared `final` so that it is clear from the source that
this declaration cannot be overridden.

2. Extend Swift's class model to permit overriding of declarations
introduced in extensions.


Additionally, a non-`final` `@objc` declaration in a class extension
is implicitly `dynamic`. This is a second-order effect of Swift's
class model not allowing declarations introduced in class extensions
to be overridable: such declarations must always be `@objc` and (more
importantly) be called via the Objective-C runtime (`objc_msgSend`),
so they are `dynamic` *in practice*. Moreover, as an implementation
convenience in the Swift compiler, Swift infers `dynamic`. If in fact
Swift gains the ability to override declarations introduced in a class
extension (without `@objc`), the inference of `dynamic` would no
longer be necessary. Removing that inference in such a future version
of Swift could break existing applications that swizzle those methods
via the Objective-C runtime. There are a few options here:

1. Require that `@objc` declarations in a class extension be explicitly
  stated to be either `final` or `dynamic`. Note that there is no
  enforcement that these methods are not swizzled via the Objective-C
  runtime, so this is merely a case of forcing the user to be explicit
  about whether this method is overridable and to maintain the current
  behavior if Swift gets the ability to add overridable declarations
  in class extensions.

2. Consider the inference of 'dynamic' to be an implementation detail,
  not a semantic contract. Specifically, it means that we reserve the
  right to break Swift applications that swizzle declarations not
  explicitly marked `explicit` in some future version of Swift where
  one can introduce non-`@objc` overridable declarations in class
  extensions. It is plausible that the Objective-C runtime could be
  extended to realize when it is being asked to swizzle an Objective-C
  entry point for a non-`dynamic` Swift declaration, which would
  provide a more graceful failure mode.


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
