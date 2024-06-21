# Improved NSError Bridging

* Proposal: [SE-0112](0112-nserror-bridging.md)
* Authors: [Doug Gregor](https://github.com/DougGregor), [Charles Srstka](https://github.com/CharlesJS)
* Review Manager: [Chris Lattner](https://github.com/lattner)
* Status: **Implemented (Swift 3.0)**
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-se-0112-improved-nserror-bridging/3362)

## Introduction

Swift's error handling model interoperates directly with Cocoa's
NSError conventions. For example, an Objective-C method with an
``NSError**`` parameter, e.g.,

```objc
- (nullable NSURL *)replaceItemAtURL:(NSURL *)url options:(NSFileVersionReplacingOptions)options error:(NSError **)error;
```

will be imported as a throwing method:

```swift
func replaceItem(at url: URL, options: ReplacingOptions = []) throws -> URL
```

Swift bridges between ``ErrorProtocol``-conforming types and
``NSError`` so, for example, a Swift ``enum`` that conforms to
``ErrorProtocol`` can be thrown and will be reflected as an
``NSError`` with a suitable domain and code. Moreover, an ``NSError``
produced with that domain and code can be caught as the Swift ``enum``
type, providing round-tripping so that Swift can deal in
``ErrorProtocol`` values while Objective-C deals in ``NSError``
objects.

However, the interoperability is incomplete in a number of ways, which
results in Swift programs having to walk a careful line between the
``ErrorProtocol``-based Swift way and the ``NSError``-based way. This
proposal attempts to bridge those gaps.

Swift-evolution thread: [Charles Srstka's pitch for Consistent
bridging for NSErrors at the language
boundary](https://forums.swift.org/t/pitch-consistent-bridging-for-nserrors-at-the-language-boundary/2482),
which discussed Charles' [original
proposal](https://github.com/swiftlang/swift-evolution/pull/331) that
addressed these issues by providing ``NSError`` to ``ErrorProtocol``
bridging and exposing the domain, code, and user-info dictionary for
all errors. This proposal expands upon that work, but without directly
exposing the domain, code, and user-info.

## Motivation

There are a number of weaknesses in Swift's interoperability with
Cocoa's error model, including:

1. There is no good way to provide important error information when
  throwing an error from Swift. For example, let's consider a simple
  application-defined error in Swift:

  ```swift
  enum HomeworkError : Int, ErrorProtocol {
    case forgotten
    case lost
    case dogAteIt
  }
  ```

  One can throw ``HomeworkError.dogAteIt`` and it can be interpreted
  as an ``NSError`` by Objective-C with an appropriate error domain
  (effectively, the mangled name of the ``HomeworkError`` type) and
  code (effectively, the case discriminator). However, one cannot
  provide a localized description, help anchor, recovery attempter, or
  any other information commonly placed into the ``userInfo``
  dictionary of an ``NSError``. To provide these values, one must
  specifically construct an ``NSError`` in Swift, e.g.,

  ```swift
  throw NSError(code: HomeworkError.dogAteIt.rawValue,
                domain: HomeworkError._domain,
                userInfo: [ NSLocalizedDescriptionKey : "the dog ate it" ])
  ```

2. There is no good way to get information typically associated with
  ``NSError``'s ``userInfo`` in Swift. For example, the Swift-natural
  way to catch a specific error in the ``AVError`` error domain doesn't
  give one access to the ``userInfo`` dictionary, e.g.,:

  ```swift
  catch let error as AVError where error == .diskFull {
    // AVError is an enum, so one only gets the equivalent of the code.
    // There is no way to access the localized description (for example) or
    // any other information typically stored in the ``userInfo`` dictionary.
  }
  ```

  The workaround is to catch as an ``NSError``, which is quite a bit
  more ugly:

  ```swift
  catch let error as NSError where error._domain == AVFoundationErrorDomain && error._code == AVFoundationErrorDomain.diskFull.rawValue {
    // okay: userInfo is finally accessible, but still weakly typed
  }
  ```

  This makes it extremely hard to access common information, such as
  the localized description. Moreover, the ``userInfo`` dictionary is
  effectively untyped so, for example, one has to know a priori that
  the value associated with the known ``AVErrorTimeKey`` will be typed
  as ``CMTime``:

  ```swift
  catch let error as NSError where error._domain = AVFoundationErrorDomain {
    if let time = error.userInfo[AVErrorTimeKey] as? CMTime {
      // ...
    }
  }
  ```

  It would be far better if one could catch an ``AVError`` directly
  and query the time in a type-safe manner:

  ```swift
  catch let error as AVError {
    if let time = error.time {
      // ...
    }
  }
  ```

3. ``NSError`` is inconsistently bridged with ``ErrorProtocol``. Swift
  interoperates by translating between ``NSError`` and ``ErrorProtocol``
  when mapping between a throwing Swift method/initializer and
  an Objective-C method with an ``NSError**`` parameter. However, an
  Objective-C method that takes an ``NSError*`` parameter (e.g., to
  render it) does not bridge to ``ErrorProtocol``, meaning that
  ``NSError`` is part of the API in Swift in some places (but not
  others). For example, ``NSError`` leaks through when the following
  ``UIDocument`` API in Objective-C:

  ```objc
  - (void)handleError:(NSError *)error userInteractionPermitted:(BOOL)userInteractionPermitted;
  ```

  is imported into Swift as follows:

  ```swift
  func handleError(_ error: NSError, userInteractionPermitted: Bool)
  ```

  One would expect the first parameter to be imported as ``ErrorProtocol``.

## Proposed solution

This proposal involves directly addressing (1)-(3) with new protocols
and a different way of bridging Objective-C error code types into
Swift, along with some conveniences for working with Cocoa errors:

1. Introduce three new protocols for describing more information about
  errors: ``LocalizedError``, ``RecoverableError``, and
  ``CustomNSError``. For example, an error type can provide a
  localized description by conforming to ``LocalizedError``:

  ```swift
  extension HomeworkError : LocalizedError {
    var errorDescription: String? {
      switch self {
      case .forgotten: return NSLocalizedString("I forgot it")
      case .lost: return NSLocalizedString("I lost it")
      case .dogAteIt: return NSLocalizedString("The dog ate it")
      }
    }
  }
  ```

2. Imported Objective-C error types should be mapped to struct types
  that store an ``NSError`` so that no information is lost when
  bridging from an ``NSError`` to the Swift error types. We propose to
  introduce a new macro, ``NS_ERROR_ENUM``, that one can use to both
  declare an enumeration type used to describe the error codes as well
  as tying that type to a specific domain constant, e.g.,

  ```objc
  typedef NS_ERROR_ENUM(NSInteger, AVError, AVFoundationErrorDomain) {
    AVErrorUnknown                                      = -11800,
    AVErrorOutOfMemory                                  = -11801,
    AVErrorSessionNotRunning                            = -11803,
    AVErrorDeviceAlreadyUsedByAnotherSession            = -11804,
    // ...
  }
  ```

   The imported ``AVError`` will have a struct that allows one to
   access the ``userInfo`` dictionary directly. This retains the
   ability to catch via a specific code, e.g.,

   ```swift
   catch AVError.outOfMemory {
     // ...
   }
   ```

   However, catching a specific error as a value doesn't lose information:

   ```swift
   catch let error as AVError where error.code == .sessionNotRunning {
     // able to access userInfo here!
   }
   ```

   This also gives the ability for one to add typed accessors for known
   keys within the ``userInfo`` dictionary:

   ```swift
   extension AVError {
     var time: CMTime? {
       get {
         return userInfo[AVErrorTimeKey] as? CMTime?
       }

       set {
         userInfo[AVErrorTimeKey] = newValue.map { $0 as CMTime }
       }
     }
   }
   ```

3. Bridge ``NSError`` to ``ErrorProtocol``, so that all ``NSError``
uses are bridged consistently. For example, this means that the
Objective-C API:

  ```objc
  - (void)handleError:(NSError *)error userInteractionPermitted:(BOOL)userInteractionPermitted;
  ```

  is imported into Swift as:

  ```swift
  func handleError(_ error: ErrorProtocol, userInteractionPermitted: Bool)
  ```

  This will use the same bridging logic in the Clang importer that we
  use for other value types (``Array``, ``String``, ``URL``, etc.),
  but with the runtime translation we've already been doing for
  catching/throwing errors.

  When we introduce this bridging, we will need to remove
  ``NSError``'s conformance to ``ErrorProtocol`` to avoid creating
  cyclic implicit conversions. However, one can still explicitly turn
  an ``NSError`` into ``ErrorProtocol`` via a bridging cast, e.g.,
  ``nsError as ErrorProtocol``.

4. In Foundation, add an extension to ``ErrorProtocol`` that provides
access to the localized description, which is available for all error
types.

  ```swift
  extension ErrorProtocol {
    var localizedDescription: String {
      return (self as! NSError).localizedDescription
    }
  }
  ```

  For the Cocoa error domain, which is encapsulated by the
  ``NSCocoaError`` type, add typed access for common user-info
  keys. Note that we focus only on those user-info keys that are read
  by user code (vs. only accessed by frameworks):

  ```swift
  extension NSCocoaError {
    // Note: for exposition only. Not actual API.
    private var userInfo: [NSObject : AnyObject] {
      return (self as! NSError).userInfo
    }

    var filePath: String? {
      return userInfo[NSFilePathErrorKey] as? String
    }

    var stringEncoding: String.Encoding? {
      return (userInfo[NSStringEncodingErrorKey] as? NSNumber)
               .map { String.Encoding(rawValue: $0.uintValue) }
    }

    var underlying: ErrorProtocol? {
      return (userInfo[NSUnderlyingErrorKey] as? NSError)?.asError
    }

    var url: URL? {
      return userInfo[NSURLErrorKey] as? URL
    }
  }
  ```

5. Rename ``ErrorProtocol`` to ``Error``: once we've completed the
  bridging story, ``Error`` becomes the primary way to work
  with error types in Swift, and the value type to which ``NSError``
  is bridged:

  ```swift
  func handleError(_ error: Error, userInteractionPermitted: Bool)
  ```

## Detailed design

This section details both the design (including the various new
protocols, mapping from Objective-C error code enumeration types into
Swift types, etc.) and the efficient implementation of this design to
interoperate with ``NSError``. Throughout the detailed design, we
already assume the name change from ``ErrorProtocol`` to ``Error``.

### New protocols

This proposal introduces several new protocols that allow error types
to expose more information about error types.

The ``LocalizedError`` protocol describes an error that provides
localized messages for display to the end user, all of which provide
default implementations. The conforming type can provide
implementations for any subset of these requirements:

```swift
protocol LocalizedError : Error {
  /// A localized message describing what error occurred.
  var errorDescription: String? { get }

  /// A localized message describing the reason for the failure.
  var failureReason: String? { get }

  /// A localized message describing how one might recover from the failure.
  var recoverySuggestion: String? { get }

  /// A localized message providing "help" text if the user requests help.
  var helpAnchor: String? { get }
}

extension LocalizedError {
  var errorDescription: String? { return nil }
  var failureReason: String? { return nil }
  var recoverySuggestion: String? { return nil }
  var helpAnchor: String? { return nil }
}
```

The ``RecoverableError`` protocol describes an error that might be recoverable:

```swift
protocol RecoverableError : Error {
  /// Provides a set of possible recovery options to present to the user.
  var recoveryOptions: [String] { get }

  /// Attempt to recover from this error when the user selected the
  /// option at the given index. This routine must call handler and
  /// indicate whether recovery was successful (or not).
  ///
  /// This entry point is used for recovery of errors handled at a
  /// "document" granularity, that do not affect the entire
  /// application.
  func attemptRecovery(optionIndex recoveryOptionIndex: Int,
                       resultHandler handler: (recovered: Bool) -> Void)

  /// Attempt to recover from this error when the user selected the
  /// option at the given index. Returns true to indicate
  /// successful recovery, and false otherwise.
  ///
  /// This entry point is used for recovery of errors handled at
  /// the "application" granularity, where nothing else in the
  /// application can proceed until the attmpted error recovery
  /// completes.
  func attemptRecovery(optionIndex recoveryOptionIndex: Int) -> Bool
}

extension RecoverableError {
  /// By default, implements document-modal recovery via application-model
  /// recovery.
  func attemptRecovery(optionIndex recoveryOptionIndex: Int,
                       resultHandler handler: (recovered: Bool) -> Void) {
    handler(recovered: attemptRecovery(optionIndex: recoveryOptionIndex))
  }
}
```

Error types that conform to ``RecoverableError`` may be given an
opportunity to recover from the error. The user can be presented with
some number of (localized) recovery options, described by
``recoveryOptions``, and the selected option will be passed
to the appropriate ``attemptRecovery`` method.

The ``CustomNSError`` protocol describes an error that wants to
provide custom ``NSError`` information. This can be used, e.g., to
provide a specific domain/code or to populate ``NSError``'s
``userInfo`` dictionary with values for custom keys that can be
accessed from Objective-C code but are not covered by the other
protocols.

```swift
/// Describes an error type that fills in the userInfo directly.
protocol CustomNSError : Error {
  var errorDomain: String { get }
  var errorCode: Int { get }
  var errorUserInfo: [String : AnyObject] { get }
}
```

Note that, unlike with ``NSError``, the provided ``errorUserInfo`` requires
``String`` keys. This is in line with common practice for ``NSError``
and is important for the implementation (see below). All of these
properties are defaulted, so one can provide any subset:

```swift
extension CustomNSError {
  var errorDomain: String { ... }
  var errorCode: Int { ... }
  var errorUserInfo: [String : AnyObject] { ... }
}
```

### Mapping error types to ``NSError``

Every type that conforms to the ``Error`` protocol is implicitly
bridged to ``NSError``. This has been the case since Swift 2, where
the compiler provides a domain (i.e., the mangled name of the type)
and code (based on the discriminator of the enumeration type). This
proposal also allows for the ``userInfo`` dictionary to be populated
by the runtime, which will check for conformance to the various
protocols (``LocalizedError``, ``RecoverableError``, or
``CustomNSError``) to retrieve information.

Conceptually, this could be implemented by eagerly creating a
``userInfo`` dictionary for a given instance of ``Error``:

```swift
func createUserInfo(error: Error) -> [NSObject : AnyObject] {
  var userInfo: [NSObject : AnyObject] = [:]

  // Retrieve custom userInfo information.
  if let customUserInfoError = error as? CustomNSError {
    userInfo = customUserInfoError.userInfo
  }

  if let localizedError = error as? LocalizedError {
    if let description = localizedError.errorDescription {
      userInfo[NSLocalizedDescriptionKey] = description
    }

    if let reason = localizedError.failureReason {
      userInfo[NSLocalizedFailureReasonErrorKey] = reason
    }

    if let suggestion = localizedError.recoverySuggestion {   
      userInfo[NSLocalizedRecoverySuggestionErrorKey] = suggestion
    }

    if let helpAnchor = localizedError.helpAnchor {   
      userInfo[NSHelpAnchorErrorKey] = helpAnchor
    }
  }

  if let recoverableError = error as? RecoverableError {
    userInfo[NSLocalizedRecoveryOptionsErrorKey] = recoverableError.recoveryOptions
    userInfo[NSRecoveryAttempterErrorKey] = RecoveryAttempter()
  }
}
```

The ``RecoveryAttempter`` class is an implementation detail. It will
implement the informal protocol [``NSErrorRecoveryAttempting``](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/Foundation/Protocols/NSErrorRecoveryAttempting_Protocol/) for the given error:

```swift
class RecoveryAttempter : NSObject {
  @objc(attemptRecoveryFromError:optionIndex:delegate:didRecoverSelector:contextInfo:)
  func attemptRecovery(fromError nsError: NSError,
                       optionIndex recoveryOptionIndex: Int,
                       delegate: AnyObject?,
                       didRecoverSelector: Selector,
                       contextInfo: UnsafeMutablePointer<Void>) {
    let error = nsError as! RecoverableError
    error.attemptRecovery(optionIndex: recoveryOptionIndex) { success in
      // Exposition only: this part will actually have to be
      // implemented in Objective-C to pass the BOOL and void* through.
      delegate?.perform(didRecoverSelector, with: success, with: contextInfo)
    }
  }

  @objc(attemptRecoveryFromError:optionIndex:)
  func attemptRecovery(fromError nsError: NSError,
                       optionIndex recoveryOptionIndex: Int) -> Bool {
    let error = nsError as! RecoverableError
    return error.attemptRecovery(optionIndex: recoveryOptionIndex)
  }
}
```

The actual the population of the ``userInfo`` dictionary should not be
eager. ``NSError`` provides the notion of global "user info value
providers" that it uses to lazily request the values for certain keys,
via `setUserInfoValueProvider(forDomain:provider:)`, which is declared
as:

```swift
extension NSError {
  @available(OSX 10.11, iOS 9.0, tvOS 9.0, watchOS 2.0, *)
  class func setUserInfoValueProvider(forDomain errorDomain: String,
                                      provider: ((NSError, String) -> AnyObject?)? = nil)
}
```

The runtime would need to register a user info value provider for each
error type the first time it is bridged into ``NSError``, supplying
the domain and the following user info value provider function:

```swift
func userInfoValueProvider(nsError: NSError, key: String) -> AnyObject? {
  let error = nsError as! Error
  switch key {
  case NSLocalizedDescriptionKey:
    return (error as? LocalizedError)?.errorDescription

  case NSLocalizedFailureReasonErrorKey:
    return (error as? LocalizedError)?.failureReason

  case NSLocalizedRecoverySuggestionErrorKey:
    return (error as? LocalizedError)?.recoverySuggestion

  case NSHelpAnchorErrorKey:
    return (error as? LocalizedError)?.helpAnchor

  case NSLocalizedRecoveryOptionsErrorKey:
    return (error as? RecoverableError)?.recoveryOptions

  case NSRecoveryAttempterErrorKey:
    return error is RecoverableError ? RecoveryAttempter() : nil

  default:
    guard let customUserInfoError = error as? CustomNSError else { return nil }
    return customUserInfoError.userInfo[key]
  }
}
```

On platforms that predate the introduction of user info value
providers, there are alternate implementation strategies, including
introducing a custom ``NSDictionary`` subclass to use as the
``userInfo`` in the ``NSError`` that lazily populates the dictionary
by, effectively, calling the ``userInfoValueProvider`` function
above for each requested key. Or, one could eagerly populate
``userInfo`` on older platforms.

### Importing error types from Objective-C

In Objective-C, error domains are typically constructed using an
enumeration describing the error codes and a constant describing the
error domain, e.g,

```objc
extern NSString *const AVFoundationErrorDomain;

typedef NS_ENUM(NSInteger, AVError) {
  AVErrorUnknown                                      = -11800,
  AVErrorOutOfMemory                                  = -11801,
  AVErrorSessionNotRunning                            = -11803,
  AVErrorDeviceAlreadyUsedByAnotherSession            = -11804,
  // ...
}
```

This is currently imported as an ``enum`` that conforms to ``Error``:

```swift
enum AVError : Int {
  case unknown                                      = -11800
  case outOfMemory                                  = -11801
  case sessionNotRunning                            = -11803
  case deviceAlreadyUsedByAnotherSession            = -11804

  static var _domain: String { return AVFoundationErrorDomain }
}
```

and Swift code introduces an extension that makes it an ``Error``,
along with some implementation magic to allow bridging from an
``NSError`` (losing ``userInfo`` in the process):

```swift
extension AVError : Error {
  static var _domain: String { return AVFoundationErrorDomain }
}
```

Instead, error enums should be expressed with a new macro,
``NS_ERROR_ENUM``, that ties together the code and domain in the
Objective-C header:

```objc
extern NSString *const AVFoundationErrorDomain;

typedef NS_ERROR_ENUM(NSInteger, AVError, AVFoundationErrorDomain) {
  AVErrorUnknown                                      = -11800,
  AVErrorOutOfMemory                                  = -11801,
  AVErrorSessionNotRunning                            = -11803,
  AVErrorDeviceAlreadyUsedByAnotherSession            = -11804,
  // ...
}
```

This will import as a new struct ``AVError`` that contains an
``NSError``, so there is no information loss. The actual enum will
become a nested type ``Code``, so that it is still accessible. The
resulting struct will be as follows:

```swift
struct AVError {
  /// Stored NSError. Note that error.domain == AVFoundationErrorDomain is an invariant.
  private var error: NSError

  /// Describes the error codes; directly imported from AVError
  enum Code : Int, ErrorCodeProtocol {
    typealias ErrorType = AVError

    case unknown                                      = -11800
    case outOfMemory                                  = -11801
    case sessionNotRunning                            = -11803
    case deviceAlreadyUsedByAnotherSession            = -11804

    func errorMatchesCode(_ error: AVError) -> Bool {
      return error.code == self
    }
  }

  /// Allow one to create an error (optionally) with a userInfo dictionary.
  init(_ code: Code, userInfo: [NSObject: AnyObject] = [:]) {
    error = NSError(code: code.rawValue, domain: _domain, userInfo: userInfo)
  }

  /// Retrieve the code.
  var code: Code { return Code(rawValue: error.code)! }

  /// Allow direct access to the userInfo dictionary.
  var userInfo: [NSObject: AnyObject] { return error.userInfo }

  /// Make it easy to refer to constants without context.
  static let unknown: Code = .unknown
  static let outOfMemory: Code = .outOfMemory
  static let sessionNotRunning: Code = .sessionNotRunning
  static let deviceAlreadyUsedByAnotherSession: Code = .deviceAlreadyUsedByAnotherSession
}

// Implementation detail: makes AVError conform to Error
extension AVError : Error {
  static var _domain: String { return AVFoundationErrorDomain }

  var _code: Int { return error.code }
}
```

This syntax allows one to throw specific errors fairly easily, with
or without ``userInfo`` dictionaries:

```swift
throw AVError(.sessionNotRunning)
throw AVError(.sessionNotRunning, userInfo: [ ... ])
```

The ``ImportedErrorCode`` protocol is a helper so that we can define
a general ``~=`` operator, which is used by both ``switch`` case
matching and ``catch`` blocks:

```swift
protocol ErrorCodeProtocol {
  typealias ErrorType : Error

  func errorMatchesCode(_ error: ErrorType) -> Bool
}

func ~= <EC: ErrorCodeProtocol> (error: Error, code: EC) -> Bool {
  guard let myError = error as? EC.ErrorType else { return false }
  return code.errorMatchesCode(myError)
}
```

### Mapping ``NSError`` types back into Swift

When an ``NSError`` object bridged to an ``Error`` instance, it may be
immediately mapped back to a Swift error type (e.g., if the error was
created as a ``HomeworkError`` instance in Swift and then passed
through ``NSError`` unmodified) or it might be leave as an instance of
``NSError``. The error might then be catch as a particular Swift error
type, e.g.,

```swift
catch let error as AVError where error.code == .sessionNotRunning {
  // able to access userInfo here!
}
```

In this case, the mapping from an ``NSError`` instance to ``AVError``
goes through an implementation-detail protocol
``_ObjectiveCBridgeableError``:

```swift
protocol _ObjectiveCBridgeableError : Error {
  /// Produce a value of the error type corresponding to the given NSError,
  /// or return nil if it cannot be bridged.
  init?(_bridgedNSError error: NSError)
}
```

The initializer is responsible for checking the domain and
(optionally) the code of the incoming ``NSError`` to map it to an instance
of the Swift error type. For example, ``AVError`` would adopt this
protocol as follows:

```swift
// Implementation detail: makes AVError conform to _ObjectiveCBridgeableError
extension AVError : _ObjectiveCBridgeableError {
  init?(_bridgedNSError error: NSError) {
    // Check whether the error comes from the AVFoundation error domain
    if error.domain != AVFoundationErrorDomain { return nil }

    // Save the error
    self.error = error
  }
}
```

We do not propose that ``_ObjectiveCBridgeableError`` become a public
protocol, because the core team has already deferred a similar
proposal
([SE-0058](0058-objectivecbridgeable.md))
to make the related protocol ``_ObjectiveCBridgeable`` public.

## Other Issues

``NSError`` codes and domains are important for localization of error
messages. This is barely supported today by ``genstrings``, but
becomes considerably harder when the domain and code are hidden (as
they are in Swift). We would need to consider tooling to make it
easier to localize error descriptions, recovery options, etc. in a
sensible way. Although this is out of the scope of the Swift language
per se, it's an important part of the developer story.

## Impact on existing code

This is a major source-breaking change for Objective-C APIs that
operate on ``NSError`` values, because those parameter/return/property
types will change from ``NSError`` to ``Error``. There are ~400 such
APIs in the macOS SDK, and closer to 500 in the iOS SDK, which is a
sizable number. Fortunately, this is similar in scope to the
[Foundation value types
proposal](0069-swift-mutability-for-foundation.md),
and can use the same code migration mechanism. That said, the scale of
this change means that it should either happen in Swift 3 or not at
all.

## Future directions

### Better tooling for describing errors

When adopting one of the new protocols (e.g., ``LocalizedError``) in
an enum, one will inevitably end up with a number of ``switch``
statements that have to enumerate all of the cases, leading to a lot
of boilerplate. Better tooling could improve the situation
considerably: for example, one could use something like [Cocoa's
stringsdict
files](https://developer.apple.com/library/prerelease/content/documentation/MacOSX/Conceptual/BPInternational/StringsdictFileFormat/StringsdictFileFormat.html)
to provide localized strings identified by the enum name, case name,
and property. That would eliminate the need for the
switch-on-all-cases implementations of each property.

### Round-tripping errors through ``userInfo``

The ``CustomNSError`` protocol allows one to place arbitrary
key/value pairs into ``NSError``'s ``userInfo`` dictionary. The
implementation-detail ``_ObjectiveCBridgeableError`` protocol
allows one to control how a raw ``NSError`` is mapped to a particular
error type. One could effectively serialize the entire state of a
particular error type into the ``userInfo`` dictionary via
``CustomNSError``, then restore it via
``_ObjectiveCBridgeableError``, allowing one to form a
complete ``NSError`` in Objective-C that can reconstitute itself as a
particular Swift error type, which can be useful both for mixed-source
projects and (possibly) as a weak form of serialization for
``NSError``s.

## Alternatives considered

### Exposing the domain, code, and user-info dictionary directly

This proposal does not directly expose the domain, code, or user-info
dictionary on ``ErrorProtocol``, because these notions are superseded
by Swift's strong typing of errors. The domain is effectively subsumed
by the type of the error (e.g., a Swift-defined error type uses its
mangled name as the domain); the code is some type-specific value
(e.g., the discriminator of the enum); and the user-info dictionary is
an untyped set of key-value pairs that are better expressed in Swift
as data on the specific error type.

### Bridging ``NSError`` to a new value type ``Error``

One could introduce a new value type, ``Error``, that stores a domain,
code, and user-info dictionary but provides them with value
semantics. Doing so would make it easier to create "generic" errors
that carry some information. However, we feel that introducing new
error types in Swift is already easier than establishing a new domain
and a set of codes, because a new enum type provides this information
naturally in Swift.
