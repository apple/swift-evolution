# A New Direction for Testing in Swift

## Introduction

A key requirement for the success of any developer platform is a way to use
automated testing to identify software defects. Better APIs and tools for
testing can greatly improve a platform’s quality. Below, we propose a new
direction for testing in Swift.

We start by defining our basic principles and describe specific features that
embody those principles. We then discuss several design considerations
in-depth. Finally, we present specific ideas for delivering an all-new testing
solution for Swift, and weigh them against alternatives considered.

## Principles

Testing in Swift should be **approachable** by both new programmers and
seasoned engineers. There should be few APIs to learn, and they should feel
ergonomic and modern. It should be easy to incrementally add new tests
alongside legacy ones. Testing should integrate seamlessly into the tools and
workflows that people know and use every day.

A good test should be **expressive** and automatically include actionable
information when it fails. It should have a clear name and purpose, and there
should be facilities to customize a test’s representation and metadata. Test
details should be specified on the test, in code, whenever possible.

A testing library should be **flexible** and capable of accommodating many
needs. It should allow grouping related tests when beneficial, or letting them
be standalone. There should be ways to customize test behaviors when necessary, 
while having sensible defaults. Storing data temporarily during a test should
be possible and safe.

A modern testing system should have **scalability** in mind and gracefully
handle large test suites. It should run tests in parallel by default, but allow
some tests to opt-out. It should be effortless to repeat a test with different
inputs and see granular results. The library should be lightweight and
efficient, imposing minimal overhead on the code being tested.

## Features of a great testing system

Guided by these principles, there are many specific features we believe are
important to consider when designing a new testing system.

### Approachability

* **Be easy to learn and use**: There should be few individual APIs to
  memorize, they should have thorough documentation, and using them to write a
  new test should be fast and seamless. Its APIs should be egonomic and adhere
  to Swift’s [design guidelines](https://www.swift.org/documentation/api-design-guidelines/).
* **Validate expected behaviors or outcomes**: The most important job of any
  testing library is checking that code meets specific expectations—for example,
  by confirming that a function returns an expected result or that two values
  are equal. There are many interesting variations on this, such as comparing
  whole collections or checking for errors. A robust testing system should cover
  all these needs, while using progressive disclosure to remain simple for
  common cases.
* **Enable incremental adoption:** It should gracefully coexist with projects
  that use XCTest or other testing libraries and allow incremental adoption so
  that users can transition at their own pace. This is especially important
  because this new system may take time to achieve feature parity.
* **Integrate with tools, IDEs, and CI systems:** A useful testing library
  requires supporting tools for functionality such as listing and selecting
  tests to run, launching runner processes, and collecting results. These
  features should integrate seamlessly with common IDEs, SwiftPM’s `swift test`
  command, and continuous integration (CI) systems.

### Expressivity

* **Include actionable failure details**: Tests provide the most value when they
  fail and catch bugs, but for a failure to be actionable it needs to be
  sufficiently detailed. When a test fails, it should collect and show as much
  relevant information as reasonably possible, especially since it may not
  reproduce reliably.
* **Offer flexible naming, comments, and metadata:** Test authors should be able
  to customize the way tests are presented by giving them an informative name,
  comments, or assigning metadata like labels to tests which have things in
  common.
* **Allow customizing behaviors:** Some tests share common set-up or tear-down
  logic, which need to be performed once for each test or group. Other times, a
  test may begin failing for an irrelevant reason and must be temporarily
  disabled. Some tests only make sense to run under certain conditions, such as
  on specific device types or when an external resource is available. A modern
  testing system should be flexible enough to satisfy all these needs, without
  complicating simpler use cases.

### Flexibility

* **Allow organizing tests into groups (or not):** Oftentimes a component will
  have several related tests that would make sense to group together. It should
  be possible to group tests into hierarchies, while allowing simpler tests to
  remain standalone.
* **Support per-test storage:** Tests often need to store data while they are
  running and local variables are not always sufficient. For example, set up
  logic for a test may create a value the test needs to access, but these are in
  different scopes. There must be a way to carefully store per-test data, to
  ensure it is isolated to a single test and initialized deterministically to
  avoid unexpected dependencies or failures.
* **Allow observing test events:** Some use cases require an ability to observe
  test events—for example, to perform custom reporting or analysis of results. A
  testing library should offer hooks for event handling.

### Scalability

* **Parallelize execution:** Many tests can be run in parallel to improve
  execution time, either using multiple threads in a single process or multiple
  runner processes. A testing library should offer flexible parallelization
  options for eligible tests, encourage parallelizing whenever possible, and
  offer granular control over this behavior. It should also leverage Swift’s
  data race safety features (such as `Sendable` enforcement) to the fullest
  extent possible to avoid concurrency bugs.
* **Repeat a test multiple times with different arguments:** Many tests consist
  of a template with minor variations—for example, invoking a function multiple
  times with different arguments each time and validating the result of each
  invocation. A testing library should make this pattern easy to apply, and
  include detailed reporting so a failure during a single argument is
  represented clearly.
* **Behave consistently across platforms:** Any new testing solution should be
  cross-platform from its inception and support every platform Swift supports. 
  Its observable behaviors should be as consistent as possible across those
  platforms, especially for core responsibilities such as discovering and
  executing tests.

## Design considerations

Several areas deserve close examination when designing a new testing API. Some,
because they may benefit from language or compiler toolchain enhancements to
deliver the ideal experience, and others because they have non-obvious
reasoning or requirements.

### Expectations

Testing libraries typically offer APIs to compare values—for example, to
confirm that a function returns an expected result—and report a test failure if
a comparison does not succeed. Depending on the library, these APIs may be
called “assertions”, “expectations”, “checks”, “requirements”, “matchers“, or
other names. In this document we refer to them as **expectations**.

For test failures to be actionable, they need to include enough details to
understand the problem, ideally without a human manually reproducing the
failure and debugging. The most important details relevant to expectation
failures are the values being compared or checked and the kind of expectation
being performed (e.g. equal, not-equal, less-than, is-not-nil, etc.). Also, if
any error was caught while evaluating an expression passed to an expectation, 
that should be included.

Beyond the values of evaluated expressions, there are other pieces of
information that may be useful to capture and include in expectations:

* The **source code location** of the expectation, typically using the format
  `#fileID:#line:#column`. This helps test authors jump quickly to the line of
  code to view context, and lets IDEs present the failure in their UI at that
  location.
* The **source code text of expression(s)** passed to the expectation. In an
  example expectation API call `myAssertEqual(subject.label == "abc")`, the
  source code text would be the string `"subject.label == \"abc\""` .
    
    Even though source code text may not be necessary when viewing failures in
    an IDE since the code is present, this can still be helpful to confirm the
    expected source code was evaluated in case it changed recently. It’s even
    more useful when the failure is shown on a CI website or anywhere without
    source, since a subexpression (such as `subject.label` in this example) may
    give helpful clues about the failure.
* **Custom user-specified comments**. Comments can be helpful to allow test
  authors to add context or information only needed if there was a failure. They
  are typically short and included in the textual log output from the test
  library.
* **Custom data or file attachments.** Some tests involve files or data
  processing and may benefit from allowing expectations to save arbitrary data
  or files in the results for later analysis.

#### Powerful, yet simple

Since the most important details to include in expectation failure messages are
the expression(s) being compared and the kind of expression, some testing
libraries offer a large number of specialized APIs for detailed reporting. Here
are some examples from other prominent testing libraries:

| | Java (JUnit) | Ruby (RSpec) | XCTest |
|----|----|----|----|
| Equal | `assertEquals(result, 3);` | `expect(result).to eq(3)` | `XCTAssertEqual(result, 3)` |
| Identical  | `assertSame(result, expected);` | `expect(result).to be(expected)` | `XCTAssertIdentical(result, expected)` |
| Less than or equal | N/A | `expect(result).to be <= 5` | `XCTAssertLessThanOrEqual(result, 5)` |
| Is null/nil | `assertNull(actual);` | `expect(actual).to be_nil` | `XCTAssertNil(actual)` |
| Throws | `assertThrows(E.class, () -> { ... });` | `expect {...}.to raise_error(E)` | `XCTAssertThrowsError(...) { XCTAssert($0 is E) }` |

Offering a large number of specialized expectation APIs is a common practice
among testing libraries: XCTest has 40+ functions in its
[`XCTAssert` family](https://developer.apple.com/documentation/xctest/boolean_assertions); 
JUnit has
[several dozen](https://junit.org/junit5/docs/5.0.1/api/org/junit/jupiter/api/Assertions.html);
RSpec has a
[large DSL](https://relishapp.com/rspec/rspec-expectations/docs/built-in-matchers) 
of test matchers.

Although this approach allows straightforward reporting, it is not scalable:

* It increases the learning curve for new users by requiring them to learn many
  new APIs and remember to use the correct one in each circumstance, or risk
  having unclear test results.
* More complex use cases may not be supported—for example, if there is no
  expectation for testing that a `Sequence` starts with some prefix using
  `starts(with:)`, the user may need a workaround such as adding a custom
  comment which includes the sequence for the results to be actionable.
* It requires testing library maintainers add bespoke APIs supporting many use
  cases which creates a maintenance burden.
* Depending on the exact function signatures, it may require additional
  overloads that complicate type checking.

We believe expectations should strive to be as simple as possible and involve
few distinct APIs, but be powerful enough to include detailed results for every
expression. Instead of offering a large number of specialized expectations,
there should only be a few basic expectations and they should rely on ordinary
expressions, built-in language operators, and the standard library to cover all
use cases.

#### Evaluation rules

Expectations have certain rules which must be followed carefully when handling
arguments:

* The primary expression(s) being checked should be evaluated exactly once. In
  particular, if the expectation failed, showing the value of any evaluated
  expression should not cause the expression to be evaluated a second time. This
  is to avoid any undesirable or unexpected side effects of multiple
  evaluations.
* Custom comments or messages should only be evaluated if the expectation
  failed, and at most once, to similarly avoid undesirable side effects and
  prevent unnecessary work.

#### Continuing after a failure

A single test may include multiple expectations, and a testing library must
decide whether to continue executing a test after one of its expectations
fails. Some tests benefit from always running to completion, even if an earlier
expectation failed, since they validate different things and early expectations
are unrelated to later ones. Other tests are structured such that later logic
depends heavily on the results of earlier expectations, so terminating the test
after any expectation fails may save time. Still other tests take a hybrid
approach, where only certain expectations are required and should terminate
test execution upon failure.

This is a policy decision, and is something a testing library could allow users
to control on a global, per-test, or per-expectation basis.

#### Rich representation of evaluated values

Often, expectation APIs do not preserve raw expression values when reporting a
failure, and instead generate a string representation of those values for
reporting purposes. Although a string representation is often sufficient, 
failure presentation could be improved if an expectation were able to keep
values of certain, known data types.

As an example, imagine a hypothetical expectation API call
`ExpectEqual(image.height, 100)`, where `image` is a value of some well-known
graphical image type `UILibrary.Image`. Since this uses a known data type, the
expectation could potentially keep `image` upon failure and include it in test
results, and then an IDE or other tool could present the image graphically for
easier diagnosis. This capability could be extensible and cross-platform by
using a protocol to describe how to convert arbitrary values into one of the
testing library’s known data types, delivering much richer expectation results
presentation for commonly-used types.

### Test traits

A recurring theme in several of the features discussed above is a need to
express additional information or options about individual tests or groups of
tests. A few examples:

* Describing test requirements or marking a test disabled.
* Assigning a tag or label to a test, to locate or run those which have
  something in common.
* Declaring argument values for a parameterized or “data-driven” test.
* Performing common logic before or after a test.

Collectively, these are referred to in this document as **traits**. The traits
for an individual test _could_ be stored in a standalone file, separate from
the test definition, but relying on a separate file has known downsides: it can
get out of sync if a test name changes, and it’s easy to overlook important
details—such as whether a test is disabled or has specific requirements—when
they’re stored separately.

We believe that the traits for a single test should preferably be declared in
code placed as close to the test they describe as possible to avoid these
problems. However, global settings may still benefit from configuring via
external files, as there may not be a canonical location in code to place them.

#### Trait inheritance

When grouping related tests together, if a test trait is specified both for an
individual test and one of its containing groups, it may be ambiguous which
option takes precedence. The testing library must establish policies for how to
resolve this.

Test traits may fall into different categories in terms of their inheritance
behavior. Some semantically represent multiple values that a user would
reasonably expect to be added together. One example is test requirements: if a
group specifies one requirement, while one of its test functions specifies
another, the test function should only run if both requirements are satisfied.
The order these requirements are evaluated are worth considering and formally
specifying, so that a user could be assured that requirements are always
evaluated “outermost-to-innermost” or vice-versa.

Another example is test tags: they are also considered multi-value, but items
with tags are typically expected to have `Set` rather than `Array` semantics
and ignore duplicates, so for this type of trait the evaluation order is
insignificant.

Other test traits semantically represent a single value and conflicts between
them may be more challenging to resolve. As a hypothetical example, imagine a
test trait spelled `.enabled(Bool)` which includes a `Bool` that determines
whether a test should run. If a group specifies `.enabled(false)` but one of
its test functions specifies `.enabled(true)`, which value should be honored?
Arguments could be made for either policy.

When possible, it may be easier to avoid ambiguity: in the previous example,
this may be solved by only offering a `.disabled` option and not the opposite.
But the inheritance semantics of each option should be considered, and when
ambiguity is unavoidable, a policy for resolving it should be established and
documented.

#### Trait extensibility

A flexible test library should allow certain behaviors to be extended by test
authors. A common example is running logic before or after a test: if every
test in a certain group requires the same steps beforehand, those steps could
be placed in a single method in that group rather than expressed as an option
on a particular test. However, if only a few tests within a group require those
steps, it may make sense to leverage a test trait to mark those tests
individually.

Test traits should provide the ability to extend behaviors to support this
workflow. For example, it should be possible to define a custom test trait, and
implement hooks that allow it to run custom code before or after a test or
group.

### Test identity

Some features require the ability to uniquely identify a test, such as
selecting individual tests to run or serializing results. It may also be useful
to access the name of a test inside its own body or for an entity observing
test events to query test names.

A testing library should include a robust mechanism to uniquely identify tests
and identifiers should be stable across test runs. If it is possible to
customize a test’s display name, the testing library should decide which name
is authoritative and included in the unique identifier. Also, function
overloading could make certain test function names ambiguous without additional
type information.

### Test discovery

A frequent challenge for testing libraries in all languages is the need to
locate tests in order to run them. Users typically expect tests to be
discovered automatically, without needing to provide a comprehensive list since
that would be a maintenance burden.

There are three types of test discovery worth considering in particular, since
they serve different purposes:

* **At runtime:** When a test runner process is launched, the testing library
  needs to locate tests so it can execute them.
* **After a build:** After compilation of all test code has completed
  successfully, but before a test runner process has been launched, it may be
  useful for a tool to introspect the test build products and print the list of
  tests or extract other metadata about them without running them.
* **While authoring:** After tests have been written or edited, but before a
  build has completed, it is common for an IDE or other tool to statically
  analyze or index the code and locate tests so it can list them in a UI and
  allow running them.

Each of these are important to support, and may require different solutions.

#### Non-runtime discovery

Two of the above test discovery types—_After a build_ and _While authoring_—
require the ability to discover tests without launching a runner process, and
thus without using the testing library’s runtime logic and models to represent
tests. In addition to the IDE use case mentioned above, another reason
discovering tests statically may be useful is so CI systems can extract
information about tests and use it to optimize test execution scheduling on
physical devices. It is common for CI systems to run a different host OS than
the platform they are targeting—for example, an Intel Mac building tests for an
iOS device—and in those situations it may be impractical or expensive for the
CI system to launch a runner process to gather this information.

Note that not _all_ test details are eligible to extract statically: those that
enable runtime test behaviors may not be, but trivial metadata (such as a
test’s name or whether it is disabled) should be extractable, especially with
further advances in Swift’s support for
[Build-Time Constant Values](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0359-build-time-constant-values.md).
While designing a new testing API, it is important to consider which test
metadata should be statically extractable to support these non-runtime discovery
use cases.

### Parameterized testing

Repeating a test multiple times with different arguments—formally referred to as
[Parameterized or Data-Driven Testing](https://en.wikipedia.org/wiki/Data-driven_testing)
—can allow expanding test coverage to cover more scenarios with minimal code
repetition. Although a user could approximate this using a simple loop such as
`for...in` in the body of a test, it’s often better to let testing libraries
handle this task. A testing library can automatically keep track of the
argument(s) for each invocation of a test and record them in the results. It can
also provide a way to selectively re-run individual argument combinations for
fine-grained debugging in case only one instance failed.

Note that recording individual parameterized tests’ arguments in results and
re-running them requires some way to uniquely represent those arguments, which
overlaps with some of the considerations discussed in
[Test identity](#test-identity).

### Parallelization and concurrency

A modern testing system should make efficient use of the machine it runs on.
Many tests can safely run in parallel, and the testing system should encourage
this by enabling per-test parallelization by default. In addition to faster
results and shorter iteration time, running tests in parallel can help identify
bugs due to hidden dependencies between tests and encourage better state
isolation.

However, some tests may need to disable parallelization and run one at a time.
It should be possible to opt-out, and this may be especially useful while
migrating from older testing systems which don't support parallelization.
Although opting-out of this behavior should be possible, it should be narrowly
scoped to not sacrifice other tests' ability to run in parallel.

In addition to running tests in parallel relative to each other, tests
themselves should seamlessly support Swift's concurrency features. In particular,
this means:

* Tests should be able to use async/await whenever necessary.
* Tests should support isolation to a global actor such as `@MainActor`, but be
  nonisolated by default. (Isolation by default would undermine the goal of
  running tests in parallel by default.)
* Values passed as arguments to parameterized tests should be `Sendable`, since
  they may cross between isolation domains within the testing system's execution
  machinery.
* Types containing tests functions and their stored properties need not be
  `Sendable`, since they are only used from a single isolation domain while each
  test function is run.

### Tools integration

A well-rounded testing library should be integrated with popular tools used by
the community. This integration should include some essential functionality such
as:

* Building tests into products which can be be executed.
* Running all built tests.
* Showing per-test results, including details of each individual failure during
  a test.
* Showing an aggregate summary of a test run, including failure statistics.

Beyond the essentials, tools may offer other useful features, such as:

* Filtering tests by name, specific traits (e.g. custom tags), or other criteria.
* Outputting results to a standard format such as JUnit XML for importing into
  other tools.
* Controlling runtime options such as whether parallel execution is enabled,
  whether failed tests should be reattempted, etc.
* Relaunching a test executable after an unexpected crash.
* Reporting "live" progress as each test finishes.

In order to deliver the functionality above, a testing system needs to track
significant events which occur during execution, such as when each test starts
and finishes or encounters a failure. There needs to be a structured,
machine-readable representation of each event, and to provide granular progress
updates, there should be an option for tools to observe a stream of such events.
Any serialization formats involved should be versioned and provide some level of
stability over time, to ensure compatibility with tools which may evolve on
differing release schedules.

Some of the features outlined above are much easier to achieve if there are at
least two processes involved: One which executes the tests themselves (which may
be unstable and terminate unexpectedly), and another which launches and monitors
the first process. In this document, we'll refer to this second process as the
**harness**. A two-process architecture involving a supervisory harness helps
enable functionality such as relaunching after a crash, live progress reporting,
or outputting results to a standard format. Some cases effectively _require_ a
harness, such as when launching tests on a remotely-connected device. A modern
testing library should provide any necessary APIs and runtime support to
facilitate integration with a harness.

## Today’s solution: XCTest

[XCTest](https://developer.apple.com/documentation/xctest) has historically
served as the de facto standard testing library in Swift. It was originally
written [in 1998](https://www.sente.ch/ocunit/?lang=en) in Objective-C, and
heavily embraced that language’s idioms throughout its APIs. It relies on
subclassing (reference semantics), dynamic message passing, NSInvocation, and
the Objective-C runtime for things like test discovery and execution. In the
2010s, it was integrated deeply into Xcode and given many more capabilities and
APIs which have helped Apple platform developers deliver industry-leading
software.

When Swift was introduced, XCTest was extended further to support the new
language while maintaining its core APIs and overall approach. This allowed
developers familiar with using XCTest in Objective-C to quickly get up to
speed, but certain aspects of its design no longer embody modern best practices
in Swift, and some have become problematic and prevented enhancements. Examples
include its dependence on the Objective-C runtime for test discovery; its
reliance on APIs like NSInvocation which are unavailable in Swift; the frequent
need for implicitly-unwrapped optional (IUO) properties in test subclasses; and
its difficulty integrating seamlessly with Swift Concurrency.

It is time to chart a new course for testing in Swift, and in proposing a new
direction, this ultimately represents a successor to XCTest. This transition
will likely span several years, and we aim to thoughtfully design and deliver a
solution that will be even more powerful while bearing in mind the many lessons
learned from maintaining it over the years.

## A new direction

> [!NOTE]
> The approach described below is not meant to include a solution for _every_
> consideration or feature discussed in this document. It describes a starting
> point for this new direction, and covers many of the topics, but leaves some
> to be pursued as part of follow-on work.

The new direction includes 3 major components exposed via a new module named
`Testing`:

1. `@Test` and `@Suite` attached macros: These declare test functions and suite
   types, respectively.
1. Traits: Values passed to `@Test` or `@Suite` which customize the behavior of
   test functions or suite types.
1. Expectations `#expect` and `#require`: expression macros which validate
   expected conditions and report failures.

### Test and Suite declaration

To declare test functions and suites (types containing tests), we will leverage
[Attached Macros (SE-0389)](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0389-attached-macros.md).
At a high level, this will consist of several attached macros
which may be placed on a test type or test function, defined in a new module
named `Testing`:

```swift
/// Declare a test function
@attached(peer)
public macro Test(
  // ...Parameters described later
)

/// Declare a test suite.
@attached(member) @attached(peer)
public macro Suite(
  // ...Parameters described later
)
```

Then, test authors may attach these macros to functions or types in a test
target. Here are some usage examples:

```swift
import Testing

// A test implemented as a global function
@Test func example1() {
  // ...
}

@Suite struct BeginnerTests {
  // A test implemented as an instance method
  @Test func example2() { ... }
}

// Implicitly treated as a suite type, due to containing @Test functions.
actor IntermediateTests {
  private var count: Int

  init() async throws {
    // Runs before every @Test instance method in this type
    self.count = try await fetchInitialCount()
  }

  deinit {
    // Runs after every @Test instance method in this type
    print("count: \(count), delta: \(delta)")
  }

  // A test implemented as an async and throws instance method
  @Test func example3() async throws {
    delta = try await computeDelta()
    count += delta
    // ...
  }
}
```

**Test functions** may be defined as global functions or as either instance or
static methods in a type. They must always be explicitly annotated as `@Test`, 
they need not follow any naming convention (such as beginning with “test”), and
they may include `async`, `throws`, or `mutating`.

**Suite types**, or simply “suites”, are types containing `@Test` functions or
other nested suite types. Suite types may include the `@Suite` attribute
explicitly, although it is optional and only required when specifying traits
(described below). A suite type must have a zero-parameter `init()` if it
contains instance `@Test` methods.

**Per-test storage:** The `IntermediateTests` example demonstrates per-test
set-up and tear-down as well as per-test storage: A unique instance of
`IntermediateTests` is created for every `@Test`-annotated instance method it
contains, which means that its `init` and `deinit` are run once before and
after each respectively, and they may contain set-up or tear-down logic. Since
`count` is an instance stored property, it acts as per-test storage, and since
`example3()` is isolated to its enclosing actor type it is allowed to mutate
`count`.

**Sendability:** Note that the test functions and suite types in these examples
are not required to be `Sendable`. At runtime, if the `@Test` function is an
instance method, the testing library creates a thunk which instantiates the
suite type and invokes the `@Test` function on that instance. The suite type
instance is only accessed from a single Task.

**Actor isolation:** `@Test` functions or types may be annotated with a global
actor (such as `@MainActor`), in accordance with standard language and type
system rules. This allows tests to match the global actor of their subject and
reduce the need for suspension points.

#### Test discovery

To facilitate test discovery, the attached macros above will eventually use a
feature such as `@linkage`, an attribute for controlling low-level symbol
linkage (see
[pitch](https://forums.swift.org/t/pitch-2-low-level-linkage-control/69752)).
It will allow placing variables and functions in a special section of the binary,
where they can be retrieved and at runtime or inspected statically at build time.

However, before that feature lands, the testing library will use a temporary
approach of iterating through types conforming to a known protocol and
gathering their tests by calling a static property. The attached macros will
emit code which generates the types to be discovered using this mechanism. Once
more permanent support lands, the attached macros will be adjusted to adopt it
instead.

Regardless of which technique the attached macros above use to facilitate test
discovery, the APIs called by their expanded code need not be considered public
or stable. In particular, code emitted by these macros may call
underscore-prefixed APIs declared in the `Testing` module, which are marked
`public` to facilitate use by these macros but are generally considered a
private implementation detail.

### Traits

As discussed earlier, it is important to support specifying traits for a test. 
[SE-0389](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0389-attached-macros.md)
allows including parameters in an attached macro declaration, and this allows
users to pass arguments to a `@Test` attribute on a test function or type.

The `Testing` module will offer an extensible mechanism for specifying per-test
traits via types conforming to protocols such as `TestTrait` and `SuiteTrait`:

```swift
/// A protocol describing traits that can be added to a test function or
/// to a test suite.
public protocol Trait: Sendable { ... }

/// A protocol describing traits that can be added to a test function.
public protocol TestTrait: Trait { ... }

/// A protocol describing traits that can be added to a test suite.
public protocol SuiteTrait: Trait { ... }
```

Using these protocols, the attached macros `@Test` and `@Suite` shown earlier
will gain parameters accepting traits:

```swift
/// Declare a test function.
///
/// - Parameter traits: Zero or more traits to apply to this test.
@attached(peer)
public macro Test(
  _ traits: any TestTrait...
)

/// Declare a test function.
///
/// - Parameters:
///   - displayName: The customized display name of this test.
///   - traits: Zero or more traits to apply to this test.
@attached(peer)
public macro Test(
  _ displayName: _const String,
  _ traits: any TestTrait...
)

/// Declare a test suite.
///
/// - Parameter traits: Zero or more traits to apply to this test suite.
@attached(member) @attached(peer)
public macro Suite(
  _ traits: any SuiteTrait...
)

/// Declare a test suite.
///
/// - Parameters:
///   - displayName: The customized display name of this test suite.
///   - traits: Zero or more traits to apply to this test suite.
@attached(member) @attached(peer)
public macro Suite(
  _ displayName: _const String,
  _ traits: any SuiteTrait...
)
```

The specifics of the `Trait` protocols and the built-in types conforming to
them will be left to subsequent proposals. But to illustrate the general
pattern they will follow, here is an example showing how a hypothetical option
for marking a test disabled could be structured:

```swift
/// A test trait which marks a test as disabled.
public struct DisabledTrait: TestTrait {
  /// An optional comment related to this option.
  public var comment: String?
}

extension TestTrait where Self == DisabledTrait {
  /// Construct a test trait which marks a test disabled,
  /// with an optional comment.
  public static func disabled(_ comment: String? = nil) -> Self
}

// Usage example:
@Test(.disabled("Currently causing a crash: see #12345"))
func example4() {
  // ...
}
```

#### Nesting / subgrouping tests

Earlier examples showed how related tests may be grouped together by placing
them within a type. This technique also allows forming sub-groups by nesting
one type containing tests inside another:

```swift
struct OuterTests {
  @Test func outerExample() { /* ... */ }

  @Suite(.tags("edge-case"))
  struct InnerTests {
    @Test func innerExample1() { /* ... */ }
    @Test func innerExample2() { /* ... */ }
  }
}
```

When using this technique, test traits may be specified on nested types and
inherited by all tests they contain. For example, the `.tags("edge-case")` 
trait shown here on `InnerTests` would have the effect of adding the tag
`edge-case` to both `innerExample1()` and `innerExample2()`, as well as to
`InnerTests`.

#### Parameterized tests

Parameterized testing is easy to support using this API: The `@Test` functions
shown earlier do not accept any parameters, making them non-parameterized, but
if a `@Test` function includes a parameter, then a different overload of the
`@Test` macro can be used which accepts a `Collection` whose associated
`Element` type matches the type of the parameter:

```swift
/// Declare a test function parameterized over a collection of values.
///
/// - Parameters:
///   - traits: Zero or more traits to apply to this test.
///   - collection: A collection of values to pass to the associated test
///     function.
///
/// During testing, the associated test function is called once for each element
/// in `collection`.
@attached(peer)
public macro Test<C>(
  _ traits: any TestTrait...,
  arguments collection: C
) where C: Collection & Sendable, C.Element: Sendable

// Usage example:
@Test(arguments: ["a", "b", "c"])
func example5(letter: String) {
  // ...
}
```

Once Swift’s support for
[Variadic Generics](https://github.com/hborla/swift-evolution/blob/variadic-generics-vision/vision-documents/variadic-generics.md)
gains more functionality, the signature of these `@Test` macros may be revised
to accept more than one collection of arguments. This will expand the feature by
allowing a test function with arity _N_ to be repeated once for each combination
of elements from _N_ collections.

### Expectations

In existing test solutions available to Swift developers, there is limited
diagnostic information available for a failed expectation such as
`assert(2 < 1)`. The expression is reduced at runtime to a simple boolean value
with no context (such as the original source code) available to include in a
test’s output.

By adopting
[Expression Macros (SE-0382)](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0382-expression-macros.md),
we can give developers _implicitly expressive_ test
expectations. The expectation shown below, upon failure, can capture not just
the boolean value `false`, but also the left-hand and right-hand operands and
the operator itself (that is, `x`, `1`, and `<` respectively) and expand any
sub-expressions to their evaluated values, such as `x → 2`:

```swift
let x = 2
#expect(x < 1)  // failed: (x → 2) < 1
```

#### Handling optionals

Some expectations _must_ pass for a test to proceed—these would be expressed
with a separate macro `#require()`. Because `#require()` _must_ pass, we can
infer additional behaviors based on its argument that we cannot do with
`#expect()`. For example, if an optional value is passed to `#require()`, we
can infer that `#require()` should return the optional value or fail if it is
`nil`:

```swift
let x: Int? = 10
let y: String? = nil
let z = try #require(x) // passes, z == 10
let w = try #require(y) // fails, test ends early with a thrown error
```

#### Handling complex expressions

We can also extract the components of an expression like `a.contains(b)` and, on
failure, report the value of `a` and `b`:

```swift
let a = [1, 2, 3]
let b = 4
#expect(a.contains(b)) // failed: (a → [1, 2, 3]).contains(b → 4)
```

#### Handling collections

We can also leverage built-in language features for yet more expressiveness.
Consider the following test logic:

```swift
let a = [1, 2, 3, 4, 5]
let b = [1, 2, 3, 3, 4, 5]
#expect(a == b)
```

This expectation will fail because of the extra element `3` in `b`. We can
leverage
[Ordered Collection Diffing (SE-0240)](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0240-ordered-collection-diffing.md)
to capture exactly how these arrays differ and present that information to the
developer as part of the test output or in the IDE.

### API and ABI stability

Once this project reaches its first major release, its API will be considered
stable and will follow the same general rules as the Swift standard library.
Source-breaking API changes will be considered a last resort and be preceded by
a generous deprecation period.

However, unlike the standard library, this project will generally not guarantee
ABI stability. Tests are typically only run during the development cycle and
aren't distributed to end users, so the runtime testing library will not be
included in OS distributions and thus its interfaces do not need to maintain
ABI stability.

### Project governance

For this testing solution to stand the test of time, it needs to be well
maintained and any new features added to it should be thoughtfully designed and
undergo community review.

The codebase for this testing system will be open source. Any significant
additions to its feature set or changes to its API or behavior will follow a
process inspired by, but separate from,
[Swift Evolution](https://www.swift.org/swift-evolution/). Changes will be
described in writing using a standard
[proposal template](https://github.com/apple/swift-testing/blob/main/Documentation/Proposals/0000-proposal-template.md) and discussed in the
[swift-testing](https://forums.swift.org/c/related-projects/swift-testing/103)
category of the Swift Forums. The process for pitching and submitting proposals
for review will be formalized separately and documented in the project repo.

A new group—tentatively named the _Swift Testing Workgroup_—will be formed to
act as the primary governing body for this project. This group will be
considered a sub-group of the planned
[Ecosystem Steering Group](https://www.swift.org/blog/evolving-swift-project-workgroups/)
once it has been formed. The group will retroactively assume responsibility for
the [swift-corelibs-xctest](https://github.com/apple/swift-corelibs-xctest)
project as well. The responsibilities of members of this workgroup will include:

* defining and approving roadmaps for the project;
* scheduling proposal reviews;
* guiding community discussion;
* making decisions about proposals; and
* working with members of related workgroups, such as the
  [Platform](https://www.swift.org/platform-steering-group/),
  [Server](https://www.swift.org/sswg/), or
  [Documentation](https://www.swift.org/documentation-workgroup/) workgroups, on
  topics which intersect with their areas of focus.

The membership of this workgroup, its charter, and more specifics about its role
in the project will be formalized separately.

### Distribution

As mentioned in [Approachability](#approachability) above, testing should be
easy to use. The easier it is to write a test, the more tests will be written,
and software quality generally improves with more automated tests.

Most open source Swift software is distributed as source code and built by
clients, using tools such as Swift Package Manager. Although this is very common,
if we took this approach, every client would need to download the source for
this project and its dependencies. A test target is included in SwiftPM's
standard New Package template, so if this project became the default testing
library used by that template, this would mean a newly-created package would
require internet access to build its tests. In addition to downloading source,
clients would also need to _build_ this project along with its dependencies.
Since this project relies on Swift Macros, it depends on
[swift-syntax](https://github.com/apple/swift-syntax) and that is a large
project known to have lengthy build times as of this writing.

Due to these practical concerns, this project will be distributed as part of the
Swift toolchain, at least initially. Being in the toolchain means clients will
not need to download or build this project or its dependencies, and this
project's macros can use the copy of swift-syntax in the toolchain. Longer-term,
if the practical concerns described above are resolved, the library could be
removed from the toolchain, and doing so could yield other benefits such as more
explicit dependency tracking and more portable toolchains.

#### Package support

Although the Swift toolchain will be the primary method of distribution, the
project will support building as a Swift package. This will facilitate
development of the package itself by its maintainers and outside contributors.

Clients will also have the option of declaring an explicit package dependency on
this project rather than relying on the built-in copy in the toolchain. If a
client does this, it will be possible for tools to integrate with the client's
specified copy of the project, as long as it is a version of the package the
tool supports.

### Platform support

Testing should be broadly available across platforms. The long-term goal is for
this project to be available on all platforms Swift itself supports.

Whenever the
[Swift Platform Steering Group](https://www.swift.org/platform-steering-group/)
declares intention to support a new platform, this project will be considered
one of the highest-priority components to get working on the new platform (after
dependencies such as the standard library) since this project will enable
qualification of many other components in the stack. The maintainers of this
project will work with other Swift workgroups or steering groups to help enable
support on new platforms.

One reason why broad platform support is important is so that this project can
eventually support testing the Swift standard library. The standard library
currently uses a custom library for testing
([StdlibUnittest](https://github.com/apple/swift/tree/main/stdlib/private/StdlibUnittest))
but many of this project's benefits would be useful in that context as well, so
eventually we would like to rebase StdlibUnitTest on this project.

While the goal is for this project to work on every platform Swift does, it
currently does not work on Embedded Swift due to its reliance on existentials.
It is possible this limitation may be overcome through project changes, but if
it cannot be, the project maintainers will work with other Swift work/steering
groups to identify a solution.

## Alternatives considered

### Declarative test definition using Result Builders

While exploring new testing directions, we considered and thoroughly prototyped
an approach relying heavily on
[Result Builders](https://docs.swift.org/swift-book/LanguageGuide/AdvancedOperators.html#ID630).
At a high level, the idea involved a few pieces:

* Types like `TestCase` and `TestSuite` representing individual tests and
  groups, respectively.
* A `@resultBuilder` type `TestBuilder` allowing declarative creation of test
  hierarchies.
* A protocol named e.g. `TestProvider` with a requirement
  `@TestBuilder static var tests: TestSuite<Self>` which suite types would
  implement in order to define their tests.
* Tests defined as closures in the `static var tests` result builder above, 
  accepting an instance of a type named `TestContext` which allowed accessing
  per-test instance storage.

This approach seemed promising at first and satisfied many of the goals
described in the beginning of this document. But we discovered several
significant drawbacks:

* **Type-checking performance:** Certain Result Builder usage patterns are
  known to lead to poor type-checking performance, especially when the
  expression is long. When describing an entire suite of tests, which may be
  nested arbitrarily, the work can become exponential and lead to a noticeable
  increase in build time or, in the extreme case, compiler timeouts.
* **Accessing test state:** Because tests are defined in a `static` context, 
  per-test state must be accessed indirectly, via a `TestContext` wrapper type.
  This made accessing per-test storage more verbose than necessary, and
  introduced the need for synchronization on the wrapper type.
* **Global actor isolation:** It is difficult, or perhaps impossible, to use a
  global actor (most often `@MainActor`) in both the test body and the type
  enclosing the test which stored its per-test state, and ensure they match. In
  practice, this means that tests whose subjects include global actors are
  challenging to write without lots of `await` suspension points.
* **Build-time test discovery**: It is difficult, or perhaps impossible, to
  discover tests comprehensively at build time since the definition of tests
  happens in result builder functions.
* **Discovery:** Using Result Builders did not fully solve the problem of
  runtime test discovery; it is still necessary to locate types conforming to
  `TestProvider` protocol, even though the tests within each conforming type are
  trivial to gather by calling its static `tests` property.

### Imperative test definition

Another approach for defining tests is using a builder pattern with an
imperative style API. A good example of this is Swift’s own
[StdlibUnittest](https://github.com/apple/swift/tree/main/stdlib/private/StdlibUnittest)
library which is used to test the standard library. To define tests, a user
first creates a `TestSuite` and then calls `.test("Some name") { /* body */ }`
one or more times to add a closure containing each test.

One problem with generalizing this approach is that it doesn’t have a way to
deterministically discover tests either after a build or while authoring
tests in an IDE (see [Test discovery](#test-discovery)). Because tests are
defined using imperative code, it may contain arbitrary control flow logic
static analysis may not be able to reason about. As a contrived example, imagine
the following:

```swift
import StdlibUnittest

var myTestSuite = TestSuite("My tests")
if Bool.random() {
    myTestSuite.test("Foo") { /* ... */ }
}
```

There may be arbitrary logic (such as `if Bool.random()` here) which influences
the test suite construction, and this makes important features like IDE
discovery impossible in the general case.
