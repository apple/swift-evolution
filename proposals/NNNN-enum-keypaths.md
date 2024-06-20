# Enum Case KeyPaths

* Proposal: [SE-NNNN](NNNN-enum-keypaths.md)
* Author: [Alejandro Alonso](https://github.com/Azoy)
* Review Manager: TBD
* Status: **Awaiting review**
* Implementation: [apple/swift#58940](https://github.com/apple/swift/pull/58940)
* Bugs: [SR-5822](https://github.com/apple/swift/issues/48392)

## Introduction

I propose to allow keypaths to reference enum cases as components. The result
of accessing this keypath is either an optional single element type, an optional
tuple of the associated value, or, in the case of an empty case, an optional
void.

## Motivation

Currently in Swift, keypaths can refer to stored properties, computed
properties, and applied subscripts (among other components), but one cannot
use keypaths to reference an enum case. Say you want to get the associated
value out of an enum case and perform some operations on it. You would need to
manually write out the switch statement or perhaps a guard like the following:

```swift
enum Color {
  case blue
  case generic(String)
}

func genericFirstLetter(of color: Color) -> Character? {
  guard case .generic(let str) = color else {
    return nil
  }

  return str.first
}
```

This makes working with enums somewhat awkward. You can make helpers to make
this kind of operations easier to work with, but if you have a lot of cases with
associated values that you care about, this can easily become a lot of
boilerplate.

```swift
extension Color {
  var genericValue: String? {
    guard case .generic(let str) = self else {
      return nil
    }

    return str
  }
}

func genericFirstLetter(of color: Color) -> Character? {
  color.genericValue?.first
}
```

## Proposed solution

Allow enum cases to now be referenced by keypath components.

```swift
enum Color {
  case generic(String)
}

func genericFirstLetter(of color: Color) -> Character? {
  color[keyPath: \.generic?.first]
}

let pink = Color.generic("Pink")

print(genericFirstLetter(of: pink)) // Optional("P")
```

## Detailed design

What's being proposed is a read-only keypath to an enum case's payload (if it
has one). If referencing an enum case with no payload, the result is `Void?`.
If the case does have a payload, its result is `Payload?` where `Payload` is
either a single type (for single element payloads due to no 1 element tuples) or
a tuple of the payload elements. Indirect cases are also supported.

Referring to an enum case with a payload and not specifying the argument list is
ok when there's only a single case with that name.

```swift
enum Color {
  case generic(String)
}

let _: KeyPath<Color, String?> = \Color.generic // ok
```

However, you cannot do this when there are multiple cases with the same name but
a different number of arguments, labels, etc.

```swift
enum Color {
  case generic(hue: Int)
  case generic(String)
  case generic(String, Int)
}

// error: ambiguous
let _ = \Color.generic

// Refers to Color.generic(String)
let _: KeyPath<Color, String?> = \Color.generic(_:)

// Refers to Color.generic(String, Int)
let _: KeyPath<Color, (String, Int)?> = \Color.generic(_:_:)

// Refers to Color.generic(hue: Int)
let _: KeyPath<Color, Int?> = \Color.generic(hue:)
```

Enum cases cannot currently have the same name and share the same number of
arguments and argument labels.

```swift
enum Type {
  case void
  case void(Void) // error: redeclaration of void

  case string(String)
  case string(Substring) // error: redeclaration of string
  case string(slice: Substring) // ok
}
```

So it's impossible to have a scenario where `\Enum.case(_:)` refers to
potentially 2 enum cases. It will always refer to a single case.

One can also refer to a specific tuple element by using the argument label
instead if they desire.

```swift
enum Color {
  case generic(name: String, hue: Int)
}

let _ = \Color.generic?.name
let _ = \Color.generic?.hue
```

However, if a case has a named single element, you cannot refer to it by name.

```swift
enum Flower {
  case unknown(name: String)
}

let _ = \Flower.unknown?.name // error
```

because the value returned is the named argument itself.

When referring to the enum case `Optional.some`, we'll return the already
existing optional chain component, so keypaths like `\String?.some?.first` will
be transformed to `\String?.?.first`.

## Source compatibility

This has no effect source compatibility because one cannot reference an enum
case as a keypath component today in source.

## Effect on ABI stability

This requires an ABI addition to keypath patterns, but the internal
representation of keypath is not affected because it is not ABI.

We can back deploy this feature by emitting a less efficient representation for
these on older OSes treating them as if they were computed properties, but on
newer versions we can benefit from the simplified representation.

## Effect on API resilience

None.

## Alternatives considered

### Use `Bool` for empty cases

Another representation we could use for empty enum cases is `Bool`. This would
certainly be easier to use when say branching on the value of this keypath like
the following:

```swift
enum Color {
  case red
  case green
  case blue
}

func paint(with color: Color) {
  if color[keyPath: \.blue] {
    // ...
  }
}
```

however it'd probably be easier to just compare the enum case directly by
`color == .blue` instead of using the keypath's result.

I believe that going with the `Void?` approach makes all enum cases more
consistent in that the result is an optional value vs. some being optional some
being a `Bool`.

## Future Directions

### Mutable keypaths to enum cases

As discussed in the detailed design, this only covers read-only keypaths and it
might be useful to actually set specific values within an enum's payload.

```swift
enum Color {
  case generic(String)
}

// Something like the following type:
// OptionalWritableKeyPath<Color, String>
let genericKP = \Color.generic

var pink = Color.generic("Pink")

pink[keyPath: genericKP] = "Purple"

print(pink) // Color.generic("Purple")
```

The issue with this is that you can provide this keypath to a non
`Color.generic` and the set essentially fizzles out and does nothing. It would
be useful to introduce a new KeyPath subclass that gives us the specific
semantics for this operation allowing us to read as `Value?` but write into as
`Value`.

This is not being proposed due to needing a bit more designing and
implemenation to fully grasp what this new keypath means. Starting with
read-only keypaths to enum cases is a simple step forward and simple usability
win.

### More keypaths to other things

This proposal is just a simple addition to our current keypath capabilities,
but it's been pointed out before to add the ability to reference static members,
functions (applied and unapplied), initializers, etc. It makes a lot of sense to
add those features to the language, but it doesn't all need to be at once. We
can gradually work towards each and every one of them.
