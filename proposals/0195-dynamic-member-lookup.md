# Introduce User-defined "Dynamic Member Lookup" Types

* Proposal: [SE-0195](0195-dynamic-member-lookup.md)
* Author: [Chris Lattner](https://github.com/lattner)
* Review Manager: [Ted Kremenek](https://github.com/tkremenek)
* Implementation: [apple/swift#14546](https://github.com/apple/swift/pull/14546)
* [Previous Revision #1](https://github.com/swiftlang/swift-evolution/commit/59c7455170c231f3df9ab0ba923262e126afaa06#diff-b3460d13f154c3d6b1d8396e4159a1d2)
* Status: **Implemented (Swift 4.2)**
* Decision Notes: [Review extended](https://forums.swift.org/t/se-0195-introduce-user-defined-dynamic-member-lookup-types/8658/126), [Rationale](https://forums.swift.org/t/se-0195-introduce-user-defined-dynamic-member-lookup-types/8658/160)


## Introduction

This proposal introduces a new `@dynamicMemberLookup` attribute.  Types that use it provide
"dot" syntax for arbitrary names which are resolved
at runtime - in a **completely type safe** way.  This provides syntactic sugar that allows the
user to write:

```swift
  a = someValue.someMember
  someValue.someMember = a
  mutateParameter(&someValue.someMember)
````

and have it be interpreted by the compiler as:

```swift
  a = someValue[dynamicMember: "someMember"]
  someValue[dynamicMember: "someMember"] = a
  mutateParameter(&someValue[dynamicMember: "someMember"])
```

This allows the static type of `someValue` to decide how to implement these dynamic member
references.

Many other languages have analogous features e.g., the [dynamic](https://docs.microsoft.com/en-us/dotnet/framework/reflection-and-codedom/dynamic-language-runtime-overview) feature in C#, the [Dynamic trait in Scala](https://blog.scalac.io/2015/05/21/dynamic-member-lookup-in-scala.html), the composition of Objective-C's
[explicit properties](https://developer.apple.com/library/content/documentation/General/Conceptual/DevPedia-CocoaCore/DeclaredProperty.html) and underlying [messaging infrastructure](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtHowMessagingWorks.html).  This sort
of functionality is great for implementing dynamic language interoperability, dynamic
[proxy APIs](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtForwarding.html), and other APIs (e.g. for JSON processing).

The driving motivation for this feature is to improve interoperability with inherently dynamic
languages like Python, Javascript, Ruby and others.  That said, this feature is designed such
that it can be applied to other inherently dynamic domains in a modular way.  **NOTE**: if
you are generally supportive of interoperability with dynamic languages but are
concerned about the potential for abuse of this feature, please see the [Reducing Potential
Abuse](#reducing-potential-abuse) section in the Alternatives Considered section and voice
your support for one of those directions that limit the feature.

## Motivation and Context

Swift is well known for being exceptional at interworking with existing C and Objective-C
APIs, but its support for calling APIs written in scripting languages like Python, Perl, and Ruby
is quite lacking.

C and Objective-C are integrated into Swift by expending a heroic amount of effort into
integrating Clang ASTs, remapping existing APIs in an attempt to feel "Swifty", and by
providing a large number of attributes and customization points for changing the behavior
of this integration when writing an Objective-C header. The end result of this massive
investment of effort is that Swift tries to provide an (arguably) *better* experience when
programming against these legacy APIs than Objective-C itself does.

When considering the space of dynamic languages, four things are clear: 1) there are several
different languages of interest, and they each have significant communities in different areas:
for example, Python is big in data science and machine learning, Ruby is popular for building
server side apps, a few people apparently use Javascript, and even Perl is still widely used.
2) These languages have decades of library building behind them, sometimes with [significant
communities](https://pandas.pydata.org) and 3) there are one or two orders of magnitude
more users of these libraries than there are people currently using Swift. 4) APIs written in
these languages will never feel "Swifty", both because of serious differences between the
type systems of Swift and these languages, and because of runtime issues like the Python
GIL.

In the extensive discussion and development of this feature (including hundreds of emails to
swift-evolution) we considered many different implementation approaches.  These include
things like Objective-C style bridging support, F# type providers, generated wrappers, foreign
class support, and others described in the [Alternative Python Interoperability
Approaches](#alternative-python-interoperability-approaches) section.

The conclusion of these many discussions was that it is better to embrace the fact
that these languages are inherently dynamic and meet them where they are: F# type providers
and generated wrappers require this proposal to work in the first place (because, for example,
Javascript does not have classes and Python doesn't have stored property declarations) and
providing a good code completion experience for dynamic languages requires incorporation
of flow-sensitive analysis into SourceKit (something that is [fully compatible](#future-directions-python-code-completion)
with this proposal).

Given that Swift already has an intentionally incredibly syntax-extensible design, we only need
two minor enhancements to the language to support these dynamic languages in an
ergonomic way: this proposal (which introduces the `@dynamicMemberLookup` attribute) and a
related [`@dynamicCallable`](0216-dynamic-callable.md) proposal.

To show the impact of these proposals, consider this Python code:

```Python
class Dog:
    def __init__(self, name):
        self.name = name
        self.tricks = []    # creates a new empty list for each dog
    def add_trick(self, trick):
        self.tricks.append(trick)
        return self
```

we would like to be able to use this from Swift like this (the comments show the
corresponding syntax you would use in Python):

```swift
  // import DogModule
  // import DogModule.Dog as Dog    // an alternate
  let Dog = Python.import("DogModule.Dog")

  // dog = Dog("Brianna")
  let dog = Dog("Brianna")

  // dog.add_trick("Roll over")
  dog.add_trick("Roll over")

  // cuteDog = Dog("Kaylee").add_trick("snore")
  let cuteDog = Dog("Kaylee").add_trick("snore")
```

Of course, this would also apply to standard Python APIs as well.  Here is an example
working with the Python `pickle` API and the builtin Python function `open`:

```swift
  // import pickle
  let pickle = Python.import("pickle")

  // file = open(filename)
  let file = Python.open(filename)

  // blob = file.read()
  let blob = file.read()

  // result = pickle.loads(blob)
  let result = pickle.loads(blob)
```

This can all be expressed today as library functionality written in Swift, but without this
proposal, the code required is unnecessarily verbose and gross. Without it (but *with* the
related [`@dynamicCallable`](0216-dynamic-callable.md) proposal)
the code would have explicit member lookups all over the place:

```swift
  // import pickle
  let pickle = Python.get(member: "import")("pickle")

  // file = open(filename)
  let file = Python.get(member: "open")(filename)

  // blob = file.read()
  let blob = file.get(member: "read")()

  // result = pickle.loads(blob)
  let result = pickle.get(member: "loads")(blob)

  // dog2 = Dog("Kaylee").add_trick("snore")
  let dog2 = Dog("Kaylee").get(member: "add_trick")("snore")
```

If you'd like to explore what Python interoperability looks like with plain Swift 4 (i.e. without
either of these proposals) then check out the [Xcode 9 playground demonstrating Python
interoperability](https://forums.swift.org/t/python-interop-with-swift-4/7109/9)
that has been periodically posted to swift-evolution over the last few months.

While this is a syntactic sugar proposal, we believe that this expands Swift to be usable in
important new domains.  In addition to dynamic language interoperability, this sort of
functionality is useful for other APIs, e.g. when working with dynamically typed unstructured
data like JSON, which could provide an API like `jsonValue?.jsonField1?.jsonField2`
where each field is dynamically looked up.  An example of this is shown below in the
"Example Usage" section.

## Proposed solution

We propose introducing a new attribute to the Swift language, `@dynamicMemberLookup`.

Types with this attribute on their primary type declaration have the behavior that member
lookup - accessing `someval.member` will always succeed.  Failures to find normally
declared members of `member` will be turned into subscript references using the
`someval[dynamicMember: member]` member.  It is an error to put the
`@dynamicMemberLookup` attribute on a type but not have this subscript declared.

This attribute extends the language such that member lookup syntax (`x.y`) - when it
otherwise fails (because there is no member `y` defined on the type of `x`) and when applied
to a value with the `@dynamicMemberLookup` attribute - is accepted and
transformed into a subscript on `x`.  The produced value is a mutable
L-value if the type implements a mutable
subscript, or immutable otherwise.  This allows the type to perform arbitrary runtime
processing to calculate the value to return.  The dynamically computed property can be used
the same way as an explicitly declared computed property, including being passed `inout` if
mutable.

The attribute is intentionally designed to be flexible: the implementation can take the member
name through any `ExpressibleByStringLiteral` type, including `StaticString` and of
course `String`.  The result type may also be any type the implementation desires, including
an `Optional`, `ImplicitlyUnwrappedOptional` or some other type, which allows the
implementation to reflect dynamic failures in a way the user can be expected to process (e.g.,
see the JSON example below).

## Example Usage

While there are many potential uses of this sort of API, one motivating example comes from a
[prototype Python interoperability layer](https://forums.swift.org/t/python-interop-with-swift-4/7109/9).
There are many ways to implement this, and the details are not
particularly important, but it is perhaps useful to know that this is directly useful to address
the motivation section described above.   Given a currency type of `PyVal`, an implementation
may look like:

```swift
@dynamicMemberLookup
struct PyVal {
  ...
  subscript(dynamicMember member: String) -> PyVal {
    get {
      let result = PyObject_GetAttrString(borrowedPyObject, member)!
      return PyVal(owned: result)
    }
    set {
      PyObject_SetAttrString(borrowedPyObject, member,
                             newValue.borrowedPyObject)
    }
  }
}
```

Another example use are JSON libraries which represent JSON blobs as a Swift enum, e.g.:

```swift
enum JSON {
  case IntValue(Int)
  case StringValue(String)
  case ArrayValue(Array<JSON>)
  case DictionaryValue(Dictionary<String, JSON>)
}
```

Today, it is not unusual for them to implement members like this to allow drilling down into
the JSON value:

```swift
extension JSON {
  var stringValue : String? {
    if case .StringValue(let str) = self {
      return str
    }
    return nil
  }
  subscript(index: Int) -> JSON? {
    if case .ArrayValue(let arr) = self {
      return index < arr.count ? arr[index] : nil
    }
    return nil
  }
  subscript(key: String) -> JSON? {
    if case .DictionaryValue(let dict) = self {
      return dict[key]
    }
    return nil
  }
}
```

This allows someone to drill into a JSON value with code like:
`json[0]?["name"]?["first"]?.stringValue`.  On the other hand, if we add the
`@dynamicMemberLookup` attribute and an implementation like this:

```swift
@dynamicMemberLookup
enum JSON {
  ...
  subscript(dynamicMember member: String) -> JSON? {
    if case .DictionaryValue(let dict) = self {
      return dict[member]
    }
    return nil
  }
}
```

Now clients are able to write more natural code like:
`json[0]?.name?.first?.stringValue` which is close to the expressivity of Javascript...
while being fully type safe!

It is important to note that this proposal does *not* include introducing or changing an existing
JSON type to use this mechanic, we only point out that this proposal allows subsequent work
to turn this on if sagacious API authors approve of it.


### Future Directions: Python Code Completion

In the extensive discussion of alternative implementation approaches, one concern raised
was whether or not we could ever get a good code completion experience with this design.
It would be nice to get at least something along the lines of what Swift provides for
`AnyObject` lookup, where you can get a "big list" and filter down quickly as you type.

After extensive discussion at the Core Team, we concluded that the *best* way to get a good
code completion for Python APIs in Swift (if and when that becomes a priority) is to build such
functionality into SourceKit, and model it directly after the way that existing Python IDEs
provide their code completion.

The observation is that a state of the art Python code completion experience requires
incorporating simple control flow analysis and unsound heuristics into the the model in order
to take local hints into account, pre-filtering the lists.  These heuristics would be inappropriate
to include in the static type system of the Swift language (in any form), and thus we believe it
is better to build this as special support in SourceKit (e.g. special casing code completion on
`Python.PyVal` (or whatever the currency type ends up being called) to incorporate these
techniques.

In any case, while we would like to see such future developments, but they are beyond the
scope of this proposal, and are a tooling discussion for "swift-dev", not a language evolution
discussion.


## Source compatibility

This is a strictly additive proposal with no source breaking changes.

## Effect on ABI stability

This is a strictly additive proposal with no ABI breaking changes.

## Effect on API resilience

Types with this attribute will always succeed at member lookup (`x.foo` will
always be accepted by the compiler): members that are explicitly declared in the type or in
a visible extension will be found and referenced, and anything else will be handled by the
dynamic lookup feature.

That behavior could lead to a surprising behavior change if the API evolves over time: adding
a new statically declared member to the type or an extension will cause clients to resolve that
name to the static declaration instead of being dynamically dispatched.  This is inherent to
this sort of feature, and means it should not be used on types that have a large amount of
API, API that is likely to change over time, or API with names that are likely to conflict.

## Alternatives considered

A few alternatives were considered:

### Spelling: Attribute vs Declaration Modifier

This proposal argues for spelling this as a `@dynamicMemberLookup` attribute that is
applied to a type, but there are many other ways this can be spelled.  There could be other
]attribute names that are worth considering (suggestions welcome!), and it is also possible to
spell this as a declaration modifier like `dynamicMemberLookup`.

### Make this a marker protocol

We started with the approach of making this be a protocol that types conform to to get this
behavior.  It turns out that this behavior is very non-protocol like: it is not useful to define
generic algorithms over, and existential values are only useful if they define a specific
subscript that implements the requirements implicit in this attribute.

For these and other reasons, defining this as a protocol doesn't really fit into the design of
Swift.

### Model this with methods, instead of a labeled subscript

It may be surprising to some that this functionality is modeled as a subscript instead of a
get/set method pair.  This is intentional though: subscripts are the way that Swift supports
parameterized l-values like we're are trying to expose here.  Exposing this as two methods
doesn't fit into the language as cleanly, and would make the compiler implementation more
invasive.  It is better to use the existing model for l-values directly.

### Reducing Potential Abuse

In the discussion cycle, there was significant concern about abuse of this feature, particularly
when this was spelled as a protocol that would allow someone to retroactively conform a type
to the  `DynamicMemberLookupProtocol` protocol.  For this and other reasons, this proposal
has been revised to be an attribute that can be applied to a primary type declaration, not to
be a protocol.

On the other hand, the potential for abuse has never been a strong guiding principle for Swift
features (and many existing features could theoretically be abused (including operator
overloading, emoji identifiers,
`AnyObject` lookup, `ImplicitlyUnwrappedOptional`, and many more).  If we look to
other language communities, we find that Objective-C has many inherently dynamic features.
Furthermore, directly
analogous "dynamic" features were [added to C#](https://docs.microsoft.com/en-us/dotnet/framework/reflection-and-codedom/dynamic-language-runtime-overview) and Scala late in
their evolution cycle, and no one has produced evidence that they led to abuse.

Finally, despite **extensive discussion** on the mailing list and lots of concern
about how this feature could be abused, no one has been able to produce (even one)
non-malicious example where someone would use this feature inappropriately and lead to
harm for users (and of course, if you're consuming an API produced by a malicious entity, you
are already doomed. `:-)`).

Fortunately (if and only if a compelling example of harm were demonstrated) there are
different ways to assuage concerns of "abuse" of this feature, for example we could have
the compiler specifically bless individual well-known types, e.g. `Python.PyVal` (or
whatever it is eventually named) by baking in awareness of these types into the compiler.
Such an approach would require a Swift evolution proposal to add a new type that
conforms to this.

Despite such possibilities, this doesn't seem like a likely future direction.

### Increasing Visibility of Dynamic Member Lookups

People have suggested that we add some explicit syntax to accesses to make the dynamic
lookup visible, e.g.: `foo.^bar` or `foo->bar` or some other punctuation character we
haven't already used.  In my opinion, this is the wrong thing to do for several reasons:

1) Swift's type system already includes features (optionals, IUOs, runtime failure) for handling
    failability.  Keeping that orthogonal to this proposal is good because that allows API
    authors to make the right decision for expressing the needs of their use-case.
2) Swift already has a dynamic member lookup feature, "`AnyObject` dispatch" which does
    not use additional punctuation, so this would break precedent.  The syntax and behavior
    of AnyObject dispatch was carefully considered in a situation that was directly analogous
    to this - Swift 1 days - where nullability audited headers were rare.
3) Adding punctuation to the lookup itself reduces the likelihood that an API author would
    make the lookups return strong optionals, because of increased syntactic noise.
4) The point of this proposal is to make use of dynamic language APIs more elegant than
    what is already possible: making use of them ugly (this punctuation character would be
    pervasive through use of the APIs and just add visual noise, not clarity) undermines the
    entire purpose of this proposal.
5) There are already other features (including operator overloading, subscripts, forthcoming
    `@dynamicCallable`, etc) that are just as dynamic as property lookup when implemented
    on a type like `PyVal`.  Requiring additional syntax for "`a.b`" but not "`a + b`" (which can
    be just as dynamic) would be inconsistent.
6) Language syntax is not the only way to handle this.  IDEs like Xcode could color code
    dynamic member lookups differently, making their semantics visible without adversely
    affecting the code that is written.  It is true that not all developers use Xcode, but since
    many other major pieces of Swift assume a rich editor experience, it is consistent for this
    one to expect it too.

It probably helps to consider an example.  Assume we used the `^` sigil to represent a dynamic
operation (member lookup, call, dynamic operator, etc).  This would give us syntax like
`foo.^bar` for member lookup, `baz^(42)` for calls.  The API author isn't forced to pick an
operator that follows this scheme, but has the option to do so.  Such a design would
change reasonable code like this:

```swift
  let np = Python.import("numpy")
  let x = np.array([6, 7, 8])
  let y = np.arange(24).reshape(2, 3, 4)

  let a = np.ones(3, dtype: np.int32)
  let b = np.linspace(0, pi, 3)
  let c = a+b
  let d = np.exp(c)
  print(d)
```

into:

```swift
  let np = Python.import("numpy")
  let b = np^.array^([6, 7, 8])
  let y = np^.arange^(24)^.reshape^(2, 3, 4)

  let a = np^.ones^(3, dtype: np^.int32)
  let b = np^.linspace^(0, pi, 3)
  let c = a+^b
  let d = np^.exp^(c)
```

This does not improve clarity of code, it merely serves to obfuscate logic.  It is immediately
apparent from the APIs being used, the API style, and the static types (in Xcode or through
static declarations) that this is all Python stuff.  When you start mixing in use of native Swift
types like dictionaries (something we want to encourage because they are typed!) you end up
with an inconsistent mishmash where people would just try adding syntax or applying fixits
continuously until the code builds.

The example gets ever worse if the implementation chooses to return a strong optional value,
because you'd end up with something like this (just showing a couple lines):

```swift
  let y = np^.arange?^(24)?^.reshape?^(2, 3, 4)!
  let a = np^.ones?^(3, dtype: np^.int32!)!
```

This is so bad that no one would actually do this.  Making the Python operators return
optionals would be even worse, since binary operators break optional chaining.

## Alternative Python Interoperability Approaches

In addition to the alternatives above (which provide different approaches to refine a
proposal along the lines of this one), there have also been extensive discussion of different
approaches to the problem of dynamic language interoperability on the whole.  Before
we explore those alternatives, we need to understand the common sources of concern.

The biggest objection to this proposal is that it is "completely dynamic", and some
people have claimed that Swift intentionally pushes people towards static types.  Others
claim that dynamic features like this are unsafe (what if a member doesn't exist?), and claim
that Swift does not provide unsafe features.

While the aims are largely correct for pure Swift code, the only examples of language
interoperability we have so far is with C and Objective-C, and Swift's interop with those is
indeed already memory unsafe, unsound, and far more invasive than what we propose.
Observe:

1) Swift does have completely dynamic features, even ones that can be used unsafely or
    unwisely.  There was a recent talk at Swift Summit 2017 that explored some of these.
2) Bridging to dynamic languages inherently requires "fully dynamic" facilities, because the
    imported language has fully dynamic capabilities, and programmers use them.  This is
    pervasive in Python code, and is also reasonable common in Objective-C "the `id` type".
3) The Objective-C interoperability approach to handling the "inherently dynamic" parts of
   Objective-C is a feature called "AnyObject dispatch".  If you aren't familiar, it returns
   members lookup as `ImplicitlyUnwrappedOptional` types (aka, `T!` types), which are
   extremely dangerous to work with - though not "unsafe" in the Swift sense.
4) Beyond the problems with IUOs, Anyobject lookup is *also* completely non-type safe when
   the lookup is successful: it is entirely possible to access a property or method as "String"
   even if it were declared as an integer in Objective-C: the bridging logic doesn't even return
   such a lookup as `nil` in general.
5) The implementation of "AnyObject lookup" is incredibly invasive across the compiler and
    has been the source of a large number of bugs.
6) Beyond AnyObject lookup, the Clang importer itself is ~25 thousand lines of code, and
    while it is the source of great power, it is also a continuous source of bugs and problems.
    Beyond the code in the Clang importer library itself, it also has tendrils in almost
    every part of the compiler and runtime.

in contrast, the combination of the `@dynamicMemberLookup` and `@dynamicCallable`
proposals are both minimally invasive in the compiler, and fully type safe.  Implementations of
these proposals may choose to provide one of three styles of fully type safe implementation:

1) Return optional types, forcing clients to deal with dynamic failures.  We show an example
    of this in the JSON example above.
2) Return IUO types, providing the ability for clients to deal with dynamic failures, but allow
    them to ignore them otherwise.  This approach is similar to AnyObject dispatch, but is
    more type safe.
3) Return non-optional types, and enforce type safety at runtime with traps.  This is the
    approach followed in the Python interop example above, but that bridge is under
    development and may switch to another approach if it provides a better experience. The
    actual prototype already has a more holistic approach for handling failability that isn't
    describe here.

The style of design depends on the exact bridging problem being solved, e.g. the customs of
the language being interoperated with.  Also, yes, it is possible to use these APIs to provide
an unsafe API, but that is true of literally every feature in Swift - Swift provides support
for unsafe features as part of its goals to be pragmatic.

With this as background, let's explore the proposed alternatives approaches to dynamic
language interoperability:

### Direct language support for Python (and all the other languages)

We considered implementing something analogous to the Clang importer for Python, which
would add a first class Python specific type(s) to Swift language and/or standard library.  We
rejected this option because:

1) Such integration would **require that we do something like this proposal anyway**,
    because Python (like Objective-C) is a fundamentally dynamic language, and we need a
    way to express that fundamental dynamism: either this proposal or something like
    "`AnyObject` for Python".
2) Python is actually far less "inherently typed" than Objective-C is, because everything
    is typed with the equivalent of `id`, whereas the Objective-C community has used concrete
    types for many things (and with the introduction of Objective-C Generics and other
    features, this has become even more common).
3) it would be *significantly* more invasive in the compiler than this proposal.  While it is
    unlikely to be as large a scope of impact as the Clang importer in terms of bulk of code, it
    would require just as many tendrils spread throughout the compiler.
4)  Taking this approach would set the precedent for all other dynamic languages to get first
     class language support baked into the Swift compiler, leading to an even greater
     complexity spiral down the road.
5) The proposed "first class support" doesn't substantially improve the experience of working
    with Python over this proposal, so it is all pain and no gain.

Several people have suggested that a "Clang-importer" style Python interoperability approach
could use the "Type Hints" introduced in
[PEP 484](https://www.python.org/dev/peps/pep-0484/), which would invalidate that last
point above.  This approach to progressive typing for Python was prototyped in
[mypy](http://mypy-lang.org/) and first [shipped in Python
3.5](https://docs.python.org/3/library/typing.html) in September 2015.

Unfortunately, it isn't reasonable to expect Type Hints to significantly improve the experience
working with Python in Swift for several reasons, including:

1) PEP 484 states: "It should also be emphasized that **Python will remain a dynamically
    typed language, and the authors have no desire to ever make type hints mandatory,
     even by convention.**" (the emphasis is by the authors of PEP 484).  This means we
     need this proposal or something like AnyObject lookup ... forever.
2) These annotations are only currently supported on a [provisional
    basis](https://docs.python.org/3/glossary.html#term-provisional-api), which means that
    they are "deliberately excluded from .. backwards compatibility guarantees... up to and
    including removal of the interface".  Because they are subject to change and potentially
    removal, they haven't gotten wide adoption.
3) Unlike Objective-C type annotations, these type hints are inherently unsound by design.
    Further, Python APIs often creep to accepting many more types than a Swift programmer
    would expect, which means that a type annotation is often "so broad as to be unhelpful" or
    "too narrow to be correct".
4) These annotations are not dynamically enforced, and they are specifically designed to
    "influence" static analysis tools, which means that they are also frequently wrong.  In the
    context of a narrowly used static analysis tool, a wrong type annotation is merely annoying.
    In the context of bridging into Swift, being incorrect is a real problem.
5) The fact that they have only been available in Python **3** (which has a smaller community
    than Python 2) and have only been shipping for 2 years, means that they haven't been used
    by many people, and which contributes to their low adoption.
6) It is a goal to work with other dynamic language communities beyond Python, and not
    all have progressive typing facilities like this one.
7) Type annotations help the most in situations when you are "mixing and matching" Swift
    code with an existing body of some other code that you are able to modify.  While it is
    possible that some people will want to mix and match Swift and Python, by far the most
    common reason for wanting to interoperate with a dynamic language is to leverage the
    existing APIs that the community provides in a black box manner.  Being black box means
    that you want to reuse the code, but you don't want to touch and own it yourself.
8) Finally, the idea of Swift providing a "better Python than Python itself does"
    under-appreciates the effort that the Python community has spent trying to achieve the
    same goals.  Its community includes a lot of people in it that understand the
    benefits of static and progressive typing (e.g. the [mypy](http://mypy-lang.org/) community)
    and they have spent a lot of time on this problem.  Despite their efforts, the ideas have low
    adoption: Swift bridging doesn't change the fundamental reasons for this.

Finally, it is important to realize that Swift and Clang have a
special relationship because they were designed by many of the same people - so their
implementations are similar in many respects.  Further, the design of both Clang and the
Objective-C language shifted in major ways to support interoperability with Swift - including
the introduction of ARC, Modules, Objective-C generics, pervasive lazy decl resolution, and
many more minor features).

It is extremely unlikely that another established language/compiler community would accept
the scope of changes necessary to provide great importer support for them, and it is also
extremely unlikely that one would just magically work out of the box for what we need it to
do.  That said, our goals aren't to provide a better Python than Python, only to embrace
Python (and other dynamic languages) for what they are, without polluting the Swift compiler
and runtime with a ton of complexity that we'll have to carry around forever.


### Introduce F# style "Type Providers" into Swift

[Type providers](https://docs.microsoft.com/en-us/dotnet/fsharp/tutorials/type-providers/) are
a cool feature of the F# language.  They are an expansive (and quite
complex) metaprogramming system which permits a "metaprogrammed library" to synthesize
types and other language features into the program based on statically knowable type
databases.  This leads to significantly improved type safety in the case where schemas for
dynamic APIs are available (e.g. a JSON schema) but which are not apparent in the source
code.

While type providers are extremely interesting and might be considered for inclusion into a
future macro system in Swift, it is important to understand that they just provide a way for
library developers to extend the compiler along the lines of the Clang importer in Swift.  As
such, they aren't actually helpful for this proposal for all of the reasons described
in the section above, but the most important point is that:

Type providers can only "provide" a type that is expressible in the language, and dynamic
 languages do have completely dynamic features, so **something that provides the
semantics of `DynamicMemberLookup` will be required** with that approach anyway.  We'd
have to take this proposal (or something substantially similar) before type providers would be
useful for Python in the first place.

Because of this, we believe that type providers and other optional typing facilities are
a potentially interesting follow-on to this proposal which should be measured according to
their own merits.  The author of this proposal is also personally skeptical that type providers
could ever be done in a way that is acceptable within the goals of Swift (e.g. to
someday have fast compile times).

### Introduce a language independent "foreign class" feature to Swift

One suggestion was to introduce a [general "foreign class" feature to Swift](https://forums.swift.org/t/proposal-introduce-user-defined-dynamic-member-lookup-types/7129/90).  The core team met to discuss this and concluded that it was the wrong direction to go.  Among opinions held by core team members, several believed that forcing other languages models into the Swift model would violate their fundamental principles (e.g. Go and Javascript don't *have* classes), some felt it would be too invasive into the compiler, and others believed that such an approach ends up requiring a `DynamicMemberLookup` related feature anyway - because
e.g. Python doesn't require property declarations.

### Use "automatically generated wrappers" to interface with Python

This approach has [numerous problems](https://forums.swift.org/t/dynamicmemberlookup-proposal-status-update/7358).  Beyond that, wrappers also fundamentally require that we take (something like) this proposal
to work in the first place.  The primary issue is that Python (and other dynamic languages)
require no property declarations.  If you have no property declaration, there is nothing for the
wrapper generator to [w]rap or generate.

