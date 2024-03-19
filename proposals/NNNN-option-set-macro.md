# Option Set Declaration Macro

* Proposal: [SE-NNNN](NNNN-option-set-macro.md)
* Authors: [Doug Gregor](https://github.com/DougGregor), [Ben Cohen](https://github.com/airspeedswift)
* Review Manager: TBD
* Status: **Awaiting review**
* Implementation: [DougGregor/swift-macro-examples#17](https://github.com/DougGregor/swift-macro-examples/pull/17/)

## Introduction

This proposal introduces an attached macro `@OptionSet` which takes over the boilerplate required when declaring a bitfield-based option set.

## Motivation

Implementing the standard library's `OptionSet` protocol, while not overly burdensome, does involve some fiddly boilerplate:

```swift
struct ShippingOptions: OptionSet {
  let rawValue: Int
  
  static let nextDay    = ShippingOptions(rawValue: 1 << 0)
  static let secondDay  = ShippingOptions(rawValue: 1 << 1)
  static let priority   = ShippingOptions(rawValue: 1 << 2)
  static let standard   = ShippingOptions(rawValue: 1 << 3)
}
```

Aside from the repetition, there is also a risk that a bit location be accidentally repeated:

```swift
  // oops, this should have been 1 << 4
  static let expedited   = ShippingOptions(rawValue: 1 << 3)
```

While this looks unlikely from the above code, in real code options often have long comment blocks above them describing their meaning, separating each case. This proposal was inspired by a bug reported to the authors caused by exactly this.

Here is a technique for preventing such errors:

```swift
struct ShippingOptions: OptionSet {
  let rawValue: Int
  
  private enum Options: Int {
    case nextDay, secondDay, priority, standard
  }
  
  static let nextDay    = ShippingOptions(rawValue: 1 << Options.nextDay.rawValue)
  static let secondDay  = ShippingOptions(rawValue: 1 << Options.secondDay.rawValue)
  static let priority   = ShippingOptions(rawValue: 1 << Options.priority.rawValue)
  static let standard   = ShippingOptions(rawValue: 1 << Options.standard.rawValue)
}
```

By using the compiler to generate the source of the raw values, the possibility of accidentally repeating a field number is eliminated. But it comes at the cost of even more repetitive boilerplate – which is also still a potential source of errors i.e.

```swift
// copy paste error – forgot to update the second occurrence
static let expedited = ShippingOptions(rawValue: 1 << Options.standard.rawValue))
```

## Proposed solution

We propose the addition of an `@OptionSet` attached macro to the standard library:

```swift
@OptionSet
struct ShippingOptions {
    private enum Options: Int {
        case nextDay, secondDay, priority, standard
    }
}
```

The macro would then generate the remaining option set implementation, similar to the code seen in the example.

## Detailed design

The above declaration would expand out to the following code:

```swift
struct ShippingOptions {
  typealias RawValue = Int

  var rawValue: RawValue

  init() { self.rawValue = 0 }

  init(rawValue: RawValue) { self.rawValue = rawValue }

  static let nextDay: Self =
    Self(rawValue: 1 << Options.nextDay.rawValue)

  static let secondDay: Self =
    Self(rawValue: 1 << Options.secondDay.rawValue)

  static let priority: Self =
    Self(rawValue: 1 << Options.priority.rawValue)

  static let standard: Self =
    Self(rawValue: 1 << Options.standard.rawValue)

  private enum Options: Int {
    case nextDay, secondDay, priority, standard
  }
}

extension ShippingOptions: OptionSet { }
```

This will require the addition of the `@OptionSet` macro declaration to the standard library:

```swift
@attached(member)
@attached(conformance)
public macro OptionSet<RawType>() = #externalMacro(module: "MacroExamplesPlugin", type: "OptionSetMacro")
```

If control over the raw value type is desired, the user can explicitly declare it:

```swift
  typealias RawValue = Int8
```

If the user wishes to explicitly choose the fields used, the enum's raw value can be specified explicitly:

```swift
  private enum Options: Int {
    case nextDay = 0
    case secondDay = 1
    // this bit is cursed, don't use it
    case priority = 3
    case standard = 4
  }
```

Existing language functionality prevents field numbers from being re-used in this case.

Users of the macro are free to include additional computed properties such as:

```
  static let express: ShippingOptions = [.nextDay, .secondDay]
```

If the user fails to declare an inner enum named `Options`, or fails to give it a raw type of `Int`, they will receive an error from the macro asking for it to be declared. An argument can also be supplied to the macro if the enum has a different name i.e. `@OptionSet("MyOptions")`.

The static option properties will have the same access modifier as the option set struct.

This macro may only be applied to structs, and must be applied to the struct's declaration.

## Source compatibility

None. 

## ABI compatibility

None. Authors of ABI-stable libraries should take care to ensure, if simplifying existing option sets, that the resulting type has the same ABI as before.

## Implications on adoption

This macro will only be available when compiling with a new enough toolchain. Source code that needs to compile with earlier toolchains must continue to conform explicitly.

## Alternatives considered

Other than doing nothing, an alternative to this proposal could be to add true native support for option sets to the language:

```swift
optionset ShippingOptions {
  case nextDay, secondDay, priority, standard
  
  static let express: ShippingOptions = [.nextDay, .secondDay]
}
```

Other than a slight further decrease in verbosity, there is no meaningful benefit to this approach over the use of a macro.
