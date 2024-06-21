# Modernize libdispatch for Swift 3 naming conventions

* Proposal: [SE-0088](0088-libdispatch-for-swift3.md)
* Author: [Matt Wright](https://github.com/mwwa)
* Review Manager: [Chris Lattner](https://github.com/lattner)
* Status: **Implemented (Swift 3.0)**
* Decision Notes: [Rationale](https://forums.swift.org/t/accepted-with-revision-se-0088-modernize-libdispatch-for-swift-3-naming-conventions/2697)
* Previous Revision: [1](https://github.com/swiftlang/swift-evolution/blob/ef372026d5f7e46848eb2a64f292328028b667b9/proposals/0088-libdispatch-for-swift3.md)

## Introduction

The existing libdispatch module imports the C API almost verbatim. To move towards a more natural Swift interface and away from the C API, this proposal outlines changes to the libdispatch module and the motivation behind them.

This discussion focuses on the transformation of the existing libdispatch API.

[Review thread](https://forums.swift.org/t/review-se-0088-modernize-libdispatch-for-swift-3-naming-conventions/2552)

## Motivation

libdispatch on Darwin already presents Objective-C compatible types to allow its objects to participate in automatic reference counting. We propose extending this support to present a design that feels “object-oriented” and inline with Swift 3's API guidelines, all without adding runtime overhead.

In Swift 2.2, libdispatch is exposed as a collection of functions imported from C. While trailing closure syntax does much to improve the basic usage of libdispatch, the experience in Swift still feels very C-like. As the introduction implies, we intend to transform the entire libdispatch API surface.

Previously, typical dispatch usage in Swift would look like:

```swift
let queue = dispatch_queue_create("com.test.myqueue", nil)
dispatch_async(queue) {
	print("Hello World")
}
```

This proposal will transform that code into:


```swift
let queue = DispatchQueue(label: "com.test.myqueue")
queue.asynchronously {
	print("Hello World")
}
```

## Detailed design

#### Type Naming

All of the types used by libdispatch will be renamed to meet the Swift 3 naming guidelines.

C type | Swift type
-------|-----------
dispatch\_object\_t | DispatchObject
dispatch\_queue\_t | DispatchQueue
dispatch\_group\_t | DispatchGroup
dispatch\_data\_t | DispatchData
dispatch\_io\_t | DispatchIO
dispatch\_semaphore\_t | DispatchSemaphore
dispatch\_source\_t | DispatchSource¹
dispatch\_time\_t | DispatchTime, DispatchWalltime

Several other libdispatch C types will be removed and replaced with their underlying type. These C typedefs serve to make the C API more clear as headerdoc, however in Swift these types are clearer when presented as their original type. Several other types will be replaced with ```OptionSet``` types, where appropriate.

C type | Swift type
--- | ---
dispatch\_fd\_t | Int32
dispatch\_block\_t | () -> ()
dispatch\_queue_attr\_t | DispatchQueueAttributes

**[1]** *Additional DispatchSource sub-types have also been introduced, a more thorough explanation is included below.*

#### Queues

Accessors to the main queue and global queues will be moved onto ```DispatchQueue``` as class properties.

```swift
class DispatchQueue : DispatchObject {
	class var main: DispatchQueue
	class func global(attributes: GlobalAttributes) -> DispatchQueue
}
```

Queue and group functions that are responsible for submitting work to dispatch queues will be transformed or replaced by Swift methods on ```DispatchQueue```. These existing functions act on, or with, queues and form a more cohesive API surface when presented on ```DispatchQueue``` itself. Previously these functions took the following form:

```swift
func dispatch_sync(queue: dispatch_queue_t, block: dispatch_block_t)
func dispatch_async(queue: dispatch_queue_t, block: dispatch_block_t)
func dispatch_group_async(group: dispatch_group_t, queue: dispatch_queue_t, block: dispatch_block_t)
```

This proposal collects these APIs into two methods on ```DispatchQueue``` and also introduces default parameters that cover QoS and work item flags. In turn this allows for control over block and QoS inheritance behaviour:

```swift
class DispatchQueue : DispatchObject {
	func synchronously(execute block: @noescape () -> Void)

	func asynchronously(
		group: DispatchGroup? = nil,
		qos: DispatchQoS = .unspecified, 
		flags: DispatchWorkItemFlags = [], 
		work: @convention(block) () -> Void)
}

queue.asynchronously(group: group) {
	print("Hello World")
}

queue.synchronously {
	print("Hello World")
}
```

The existing ```dispatch_specific_*``` functions have a cumbersome interface when presented in Swift. These functions will also be refined in Swift, presenting a more typesafe API. The aim here is to provide the same underlying functionality but with additional type information, reducing the need for casting and opportunity for misuse.

Before:

```swift
func dispatch_get_specific(key: UnsafePointer<Void>) -> UnsafeMutablePointer<Void>
func dispatch_queue_get_specific(queue: dispatch_queue_t, _ key: UnsafePointer<Void>) -> UnsafeMutablePointer<Void>
func dispatch_queue_set_specific(queue: dispatch_queue_t, _ key: UnsafePointer<Void>, _ context: UnsafeMutablePointer<Void>, _ destructor: dispatch_function_t?)
```

After:

```swift
class DispatchSpecificKey<T> {
	init() {}
}

class DispatchQueue : DispatchObject {
	/// Return the Value? for a given DispatchSpecificKey object from
	/// the current dispatch queue hierarchy.
	class func getSpecific<Value>(key: DispatchSpecificKey<Value>) -> Value?

	/// Get and set the DispatchSpecificKey for this queue,
	/// providing type information via the DispatchSpecificKey
	/// specialisation.
	func getSpecific<Value>(key: DispatchSpecificKey<Value>) -> Value?
	func setSpecific<Value>(key: DispatchSpecificKey<Value>, value: Value?)
}

// Example usage, using a global key object
let akey = DispatchSpecificKey<Int>()
queue.setSpecific(key: akey, value: 42)
```

#### Work Items

The existing ```dispatch_block_*``` API group exposes functionality that produces ```dispatch_block_t``` blocks that are wrapped with additional metadata. That behaviour in C has multiple cases where this API group can be accidentally misused because the C types are ambiguously overloaded. This proposal will introduce a new explicit class to cover this functionality, ```DispatchWorkItem``` that provides more explicit, safer typing.

```swift
class DispatchWorkItem {
	init(group: DispatchGroup? = nil, 
		qos: DispatchQoS = .unspecified, 
		flags: DispatchWorkItemFlags = [],
		execute: () -> ())

	func perform()
	
	func wait(timeout: DispatchTime = .forever) -> Int
	
	func wait(timeout: DispatchWalltime) -> Int

	func notify(queue: DispatchQueue, execute: @convention(block) () -> Void)

	func cancel()
	
	var isCancelled: Bool
}
```

All dispatch methods that accept blocks also accept ```DispatchWorkItem```:

```swift
let item = DispatchWorkItem(qos: .qosUserInitiated) {
	print("Hello World")
}

queue.asynchronously(execute: item)
```

#### Time

libdispatch exposes a lightweight representation of time², focussing on deadlines and intervals for timers. However, the C interfaces for ```dispatch_time_t``` are very unfortunately imported in Swift 2.2, with type impedance problems that require unnecessary casting in order to use in Swift. This proposal will replace ```dispatch_time_t``` with two new time types and one interval type. ```DispatchTime```, ```DispatchWalltime``` and ```DispatchTimeInterval```.

```swift
struct DispatchTime {
	static func now() -> DispatchTime
	static let forever: DispatchTime
}

struct DispatchWalltime {
	static func now() -> DispatchWalltime
	static let forever: DispatchWalltime

	init(time: timespec)
}

enum DispatchTimeInterval {
	case seconds(Int)
	case milliseconds(Int)
	case microseconds(Int)
	case nanoseconds(Int)
}

func +(time: DispatchTime, interval: DispatchTimeInterval) -> DispatchTime
func -(time: DispatchTime, interval: DispatchTimeInterval) -> DispatchTime
func +(time: DispatchTime, seconds: Double) -> DispatchTime
func -(time: DispatchTime, seconds: Double) -> DispatchTime
func +(time: DispatchWalltime, interval: DispatchTimeInterval) -> DispatchWalltime
func -(time: DispatchWalltime, interval: DispatchTimeInterval) -> DispatchWalltime
func +(time: DispatchWalltime, seconds: Double) -> DispatchWalltime
func -(time: DispatchWalltime, seconds: Double) -> DispatchWalltime
```

The aim here is to continue to provide a lightweight representation of time, while distinguishing between time and interval quantities. Time, to be used to set deadlines, and intervals used to represent the period of repeating events. This model also allows for the expression of time and intervals either as natural second-addition, or with explicit sub-second quantities.

```swift
let a = DispatchTime.now() + 3.5 // 3.5 seconds in the future
let b = DispatchTime.now() + .microseconds(350)

// Modify a DispatchSourceTimer with new start time and interval
timer.setTimer(start: .now(), interval: .milliseconds(500))
```

**[2]** *Note that libdispatch is unable to use ```Date``` from Foundation due to layering restrictions*

#### Data

```dispatch_data_t``` will be transformed into a value type, ```DispatchData```. Data objects in libdispatch have always been immutable objects, so these objects are natural candidates for value semantics in Swift.

```swift
struct DispatchData : RandomAccessCollection, _ObjectiveCBridgeable {
	enum Deallocator {
		/// Use `free`
		case free

		/// Use `munmap`
		case unmap

		/// A custom deallocator
		case custom(DispatchQueue?, @convention(block) () -> Void)
	}

	/// Initialize a `Data` with copied memory content.
	init(bytes buffer: UnsafeBufferPointer<UInt8>)
	
	/// Initialize a `Data` without copying the bytes.
	init(bytesNoCopy bytes: UnsafeBufferPointer<UInt8>, deallocator: Deallocator = .free)
	
	mutating func append(_ bytes: UnsafePointer<UInt8>, count: Int)
		
	mutating func append(_ other: DispatchData)
		
	mutating func append<SourceType>(_ buffer: UnsafeBufferPointer<SourceType>)
		
	func subdata(in range: CountableRange<Index>) -> DispatchData
		
	func region(location: Int) -> (DispatchData, Int)

}
```

This proposal will introduce new accessor methods to access the bytes in a Data object. Along with becoming iteratable, several methods will be introduced that replace the ```dispatch_data_create_map``` approach used in C:

```swift
struct DispatchData : RandomAccessCollection, _ObjectiveCBridgeable {
	func withUnsafeBytes<Result, ContentType>(
		body: @noescape (UnsafePointer<ContentType>) throws -> Result) rethrows -> Result
		
	func enumerateBytes(
		block: @noescape (buffer: UnsafeBufferPointer<UInt8>, byteIndex: Int, stop: inout Bool) -> Void)
		
	func copyBytes(to pointer: UnsafeMutablePointer<UInt8>, count: Int)
	
	func copyBytes(
		to pointer: UnsafeMutablePointer<UInt8>, 
		from range: Range<Index>)
	
	func copyBytes<DestinationType>(
		to buffer: UnsafeMutableBufferPointer<DestinationType>, 
		from range: Range<Index>? = nil) -> Int
		
	subscript(index: Index) -> UInt8
}

```

#### Sources

Finally, this proposal will introduce additional type safety to dispatch sources. While adding additional DispatchSource subclasses is out of scope for this proposal, it will introduce a new constructor for each dispatch source type.

Kind of Source | Protocol
--- | ---
DISPATCH\_SOURCE\_TYPE\_DATA\_ADD | DispatchSourceUserDataAdd
DISPATCH\_SOURCE\_TYPE\_DATA\_OR | DispatchSourceUserDataOr
DISPATCH\_SOURCE\_TYPE\_MACH\_SEND | DispatchSourceMachSend
DISPATCH\_SOURCE\_TYPE\_MACH\_RECV | DispatchSourceMachReceive
DISPATCH\_SOURCE\_TYPE\_MEMORYPRESSURE | DispatchSourceMemoryPressure
DISPATCH\_SOURCE\_TYPE\_PROC | DispatchSourceProcess
DISPATCH\_SOURCE\_TYPE\_READ | DispatchSourceRead
DISPATCH\_SOURCE\_TYPE\_SIGNAL | DispatchSourceSignal
DISPATCH\_SOURCE\_TYPE\_TIMER | DispatchSourceTimer
DISPATCH\_SOURCE\_TYPE\_VNODE | DispatchSourceFileSystemObject
DISPATCH\_SOURCE\_TYPE\_WRITE | DispatchSourceWrite

Introducing protocols for each source allows DispatchSource to return protocols that are more strongly typed then their C equivalent. Furthermore, this proposal also adds stronger typing to return types of dispatch source accessors, where appropriate:

```swift
class DispatchSource {
	class func machSend(port: mach_port_t, eventMask: DispatchSource.MachSendEvent, queue: DispatchQueue? = default) -> DispatchSourceMachSend
	
	class func machReceive(port: mach_port_t, queue: DispatchQueue? = default) -> DispatchSourceMachReceive
	
	class func memoryPressure(eventMask: DispatchSource.MemoryPressureEvent, queue: DispatchQueue? = default) -> DispatchSourceMemoryPressure
	
	class func process(identifier: pid_t, eventMask: DispatchSource.ProcessEvent, queue: DispatchQueue? = default) -> DispatchSourceProcess
	
	class func read(fileDescriptor: Int32, queue: DispatchQueue? = default) -> DispatchSourceRead
	
	class func signal(signal: Int32, queue: DispatchQueue? = default) -> DispatchSourceSignal
	
	class func timer(flags: DispatchSource.TimerFlags = default, queue: DispatchQueue? = default) -> DispatchSourceTimer
	
	class func userDataAdd(queue: DispatchQueue? = default) -> DispatchSourceUserDataAdd
	
	class func userDataOr(queue: DispatchQueue? = default) -> DispatchSourceUserDataOr
	
	class func fileSystemObject(fileDescriptor: Int32, eventMask: DispatchSource.FileSystemEvent, queue: DispatchQueue? = default) -> DispatchSourceFileSystemObject
	
	class func write(fileDescriptor: Int32, queue: DispatchQueue? = default) -> DispatchSourceWrite
}
```

```swift
class DispatchSource {
	struct ProcessEvent : OptionSet, RawRepresentable {
		let rawValue: UInt
		
		init(rawValue: UInt)

		static let exit: ProcessEvent
		
		static let fork: ProcessEvent
		
		static let exec: ProcessEvent
		
		static let signal: ProcessEvent
		
		static let all: ProcessEvent = [.exit, .fork, .exec, .signal]
	}
}

extension DispatchSourceProcess {
	var handle: pid_t
	
	var data: DispatchSource.ProcessEvent
	
	var mask: DispatchSource.ProcessEvent
}

extension DispatchSourceUserDataAdd {
  func mergeData(value: UInt)
}
```

## Module Overview

The first revision of this proposal included only a brief overview of the most critical changes in the Dispatch module. For the interested, included below is a full output of the Dispatch module after the transformations proposed by this document.

```swift
class DispatchGroup : DispatchObject {

  init()

  func wait(timeout: DispatchTime = default) -> Int

  func wait(walltime timeout: DispatchWalltime) -> Int

  func notify(queue: DispatchQueue, exeute block: () -> Void)

  func enter()

  func leave()

}

class DispatchIO : DispatchObject {

  enum StreamType : UInt {

    case stream

    case random

    typealias RawValue = UInt

    var hashValue: Int { get }

    init?(rawValue: UInt)

    var rawValue: UInt { get }

  }

  struct CloseFlags : OptionSet, RawRepresentable {

    let rawValue: UInt

    init(rawValue: UInt)

    static let stop: DispatchIO.CloseFlags

    typealias Element = DispatchIO.CloseFlags

    typealias RawValue = UInt

  }

  struct IntervalFlags : OptionSet, RawRepresentable {

    let rawValue: UInt

    init(rawValue: UInt)

    static let strictInterval: DispatchIO.IntervalFlags

    typealias Element = DispatchIO.IntervalFlags

    typealias RawValue = UInt

  }

  class func read(fileDescriptor: Int32, length: Int, queue: DispatchQueue, handler: (DispatchData, Int32) -> Void)

  class func write(fileDescriptor: Int32, data: DispatchData, queue: DispatchQueue, handler: (DispatchData?, Int32) -> Void)

  convenience init(type: DispatchIO.StreamType, fileDescriptor: Int32, queue: DispatchQueue, cleanupHandler: (error: Int32) -> Void)

  convenience init(type: DispatchIO.StreamType, path: UnsafePointer<Int8>, oflag: Int32, mode: mode_t, queue: DispatchQueue, cleanupHandler: (error: Int32) -> Void)

  convenience init(type: DispatchIO.StreamType, io: DispatchIO, queue: DispatchQueue, cleanupHandler: (error: Int32) -> Void)

  func close(flags: DispatchIO.CloseFlags)

  var fileDescriptor: Int32 { get }

  func read(offset: off_t, length: Int, queue: DispatchQueue, ioHandler io_handler: (Bool, DispatchData?, Int32) -> Void)

  func setHighWater(highWater high_water: Int)

  func setInterval(interval: UInt64, flags: DispatchIO.IntervalFlags)

  func setLowWater(lowWater low_water: Int)

  func withBarrier(barrier: () -> Void)

  func write(offset: off_t, data: DispatchData, queue: DispatchQueue, ioHandler io_handler: (Bool, DispatchData?, Int32) -> Void)

}

class DispatchObject : OS_object {

  func suspend()

  func resume()

  func setTargetQueue(queue: DispatchQueue?)

}

class DispatchQueue : DispatchObject {

  struct GlobalAttributes : OptionSet {

    let rawValue: UInt64

    init(rawValue: UInt64)

    static let qosUserInteractive: DispatchQueue.GlobalAttributes

    static let qosUserInitiated: DispatchQueue.GlobalAttributes

    static let qosDefault: DispatchQueue.GlobalAttributes

    static let qosUtility: DispatchQueue.GlobalAttributes

    static let qosBackground: DispatchQueue.GlobalAttributes

    typealias Element = DispatchQueue.GlobalAttributes

    typealias RawValue = UInt64

  }

  class var main: DispatchQueue { get }

  class func global(attributes: DispatchQueue.GlobalAttributes) -> DispatchQueue

  class func getSpecific<T>(key: DispatchSpecificKey<T>) -> T?

  convenience init(label: String, attributes: DispatchQueueAttributes = default, target: DispatchQueue? = default)

  func after(when: DispatchTime, execute work: @convention(block) () -> Void)

  func after(walltime when: DispatchWalltime, execute work: @convention(block) () -> Void)

  func apply(applier iterations: Int, execute block: @noescape (Int) -> Void)

  func asynchronously(execute workItem: DispatchWorkItem)

  func asynchronously(group: DispatchGroup? = default, qos: DispatchQoS = default, flags: DispatchWorkItemFlags = default, execute work: @convention(block) () -> Void)

  var label: String { get }

  func synchronously(execute block: @noescape () -> Void)

  func synchronously(execute workItem: DispatchWorkItem)

  func synchronously<T>(execute work: @noescape () throws -> T) rethrows -> T

  func synchronously<T>(flags: DispatchWorkItemFlags, execute work: @noescape () throws -> T) rethrows -> T

  var qos: DispatchQoS { get }

  func getSpecific<T>(key: DispatchSpecificKey<T>) -> T?

  func setSpecific<T>(key: DispatchSpecificKey<T>, value: T)

}

@noreturn 
func dispatchMain()

class DispatchSemaphore : DispatchObject {

  init(value: Int)

  func wait(timeout: DispatchTime = default) -> Int

  func wait(walltime timeout: DispatchWalltime) -> Int

  func signal() -> Int

}

class DispatchSource : DispatchObject {

  struct MachSendEvent : OptionSet, RawRepresentable {

    let rawValue: UInt

    init(rawValue: UInt)

    static let dead: DispatchSource.MachSendEvent

    typealias Element = DispatchSource.MachSendEvent

    typealias RawValue = UInt

  }

  struct MemoryPressureEvent : OptionSet, RawRepresentable {

    let rawValue: UInt

    init(rawValue: UInt)

    static let normal: DispatchSource.MemoryPressureEvent

    static let warning: DispatchSource.MemoryPressureEvent

    static let critical: DispatchSource.MemoryPressureEvent

    static let all: DispatchSource.MemoryPressureEvent

    typealias Element = DispatchSource.MemoryPressureEvent

    typealias RawValue = UInt

  }

  struct ProcessEvent : OptionSet, RawRepresentable {

    let rawValue: UInt

    init(rawValue: UInt)

    static let exit: DispatchSource.ProcessEvent

    static let fork: DispatchSource.ProcessEvent

    static let exec: DispatchSource.ProcessEvent

    static let signal: DispatchSource.ProcessEvent

    static let all: DispatchSource.ProcessEvent

    typealias Element = DispatchSource.ProcessEvent

    typealias RawValue = UInt

  }

  struct TimerFlags : OptionSet, RawRepresentable {

    let rawValue: UInt

    init(rawValue: UInt)

    static let strict: DispatchSource.TimerFlags

    typealias Element = DispatchSource.TimerFlags

    typealias RawValue = UInt

  }

  struct FileSystemEvent : OptionSet, RawRepresentable {

    let rawValue: UInt

    init(rawValue: UInt)

    static let delete: DispatchSource.FileSystemEvent

    static let write: DispatchSource.FileSystemEvent

    static let extend: DispatchSource.FileSystemEvent

    static let attrib: DispatchSource.FileSystemEvent

    static let link: DispatchSource.FileSystemEvent

    static let rename: DispatchSource.FileSystemEvent

    static let revoke: DispatchSource.FileSystemEvent

    static let all: DispatchSource.FileSystemEvent

    typealias Element = DispatchSource.FileSystemEvent

    typealias RawValue = UInt

  }

  class func machSend(port: mach_port_t, eventMask: DispatchSource.MachSendEvent, queue: DispatchQueue? = default) -> DispatchSourceMachSend

  class func machReceive(port: mach_port_t, queue: DispatchQueue? = default) -> DispatchSourceMachReceive

  class func memoryPressure(eventMask: DispatchSource.MemoryPressureEvent, queue: DispatchQueue? = default) -> DispatchSourceMemoryPressure

  class func process(identifier: pid_t, eventMask: DispatchSource.ProcessEvent, queue: DispatchQueue? = default) -> DispatchSourceProcess

  class func read(fileDescriptor: Int32, queue: DispatchQueue? = default) -> DispatchSourceRead

  class func signal(signal: Int32, queue: DispatchQueue? = default) -> DispatchSourceSignal

  class func timer(flags: DispatchSource.TimerFlags = default, queue: DispatchQueue? = default) -> DispatchSourceTimer

  class func userDataAdd(queue: DispatchQueue? = default) -> DispatchSourceUserDataAdd

  class func userDataOr(queue: DispatchQueue? = default) -> DispatchSourceUserDataOr

  class func fileSystemObject(fileDescriptor: Int32, eventMask: DispatchSource.FileSystemEvent, queue: DispatchQueue? = default) -> DispatchSourceFileSystemObject

  class func write(fileDescriptor: Int32, queue: DispatchQueue? = default) -> DispatchSourceWrite

}

protocol DispatchSourceType : NSObjectProtocol {

  typealias DispatchSourceHandler = @convention(block) () -> Void

  func setEventHandler(handler: DispatchSourceHandler?)

  func setCancelHandler(handler: DispatchSourceHandler?)

  func setRegistrationHandler(handler: DispatchSourceHandler?)

  func cancel()

  func resume()

  func suspend()

  var handle: UInt { get }

  var mask: UInt { get }

  var data: UInt { get }

  var isCancelled: Bool { get }

}

extension DispatchSource : DispatchSourceType {

}

protocol DispatchSourceUserDataAdd : DispatchSourceType {

  func mergeData(value: UInt)

}

extension DispatchSource : DispatchSourceUserDataAdd {

}

protocol DispatchSourceUserDataOr : DispatchSourceType {

  func mergeData(value: UInt)

}

extension DispatchSource : DispatchSourceUserDataOr {

}

protocol DispatchSourceMachSend : DispatchSourceType {

  var handle: mach_port_t { get }

  var data: DispatchSource.MachSendEvent { get }

  var mask: DispatchSource.MachSendEvent { get }

}

extension DispatchSource : DispatchSourceMachSend {

}

protocol DispatchSourceMachReceive : DispatchSourceType {

  var handle: mach_port_t { get }

}

extension DispatchSource : DispatchSourceMachReceive {

}

protocol DispatchSourceMemoryPressure : DispatchSourceType {

  var data: DispatchSource.MemoryPressureEvent { get }

  var mask: DispatchSource.MemoryPressureEvent { get }

}

extension DispatchSource : DispatchSourceMemoryPressure {

}

protocol DispatchSourceProcess : DispatchSourceType {

  var handle: pid_t { get }

  var data: DispatchSource.ProcessEvent { get }

  var mask: DispatchSource.ProcessEvent { get }

}

extension DispatchSource : DispatchSourceProcess {

}

protocol DispatchSourceRead : DispatchSourceType {

}

extension DispatchSource : DispatchSourceRead {

}

protocol DispatchSourceSignal : DispatchSourceType {

}

extension DispatchSource : DispatchSourceSignal {

}

protocol DispatchSourceTimer : DispatchSourceType {

  func setTimer(start: DispatchTime, leeway: DispatchTimeInterval = default)

  func setTimer(walltime start: DispatchWalltime, leeway: DispatchTimeInterval = default)

  func setTimer(start: DispatchTime, interval: DispatchTimeInterval, leeway: DispatchTimeInterval = default)

  func setTimer(start: DispatchTime, interval: Double, leeway: DispatchTimeInterval = default)

  func setTimer(walltime start: DispatchWalltime, interval: DispatchTimeInterval, leeway: DispatchTimeInterval = default)

  func setTimer(walltime start: DispatchWalltime, interval: Double, leeway: DispatchTimeInterval = default)

}

extension DispatchSource : DispatchSourceTimer {

}

protocol DispatchSourceFileSystemObject : DispatchSourceType {

  var handle: Int32 { get }

  var data: DispatchSource.FileSystemEvent { get }

  var mask: DispatchSource.FileSystemEvent { get }

}

extension DispatchSource : DispatchSourceFileSystemObject {

}

protocol DispatchSourceWrite : DispatchSourceType {

}

extension DispatchSource : DispatchSourceWrite {

}

extension DispatchSourceMemoryPressure {

  var data: DispatchSource.MemoryPressureEvent { get }

  var mask: DispatchSource.MemoryPressureEvent { get }

}

extension DispatchSourceMachReceive {

  var handle: mach_port_t { get }

}

extension DispatchSourceFileSystemObject {

  var handle: Int32 { get }

  var data: DispatchSource.FileSystemEvent { get }

  var mask: DispatchSource.FileSystemEvent { get }

}

extension DispatchSourceUserDataOr {

  func mergeData(value: UInt)

}

struct DispatchData : RandomAccessCollection, _ObjectiveCBridgeable {

  typealias Iterator = DispatchDataIterator

  typealias Index = Int

  typealias Indices = DefaultRandomAccessIndices<DispatchData>

  static let empty: DispatchData

  enum Deallocator {

    case free

    case unmap

    case custom(DispatchQueue?, @convention(block) () -> Void)

  }

  init(bytes buffer: UnsafeBufferPointer<UInt8>)

  init(bytesNoCopy bytes: UnsafeBufferPointer<UInt8>, deallocator: DispatchData.Deallocator = default)

  var count: Int { get }

  func withUnsafeBytes<Result, ContentType>(body: @noescape (UnsafePointer<ContentType>) throws -> Result) rethrows -> Result

  func enumerateBytes(block: @noescape (buffer: UnsafeBufferPointer<UInt8>, byteIndex: Int, stop: inout Bool) -> Void)

  mutating func append(_ bytes: UnsafePointer<UInt8>, count: Int)

  mutating func append(_ other: DispatchData)

  mutating func append<SourceType>(_ buffer: UnsafeBufferPointer<SourceType>)

  func copyBytes(to pointer: UnsafeMutablePointer<UInt8>, count: Int)

  func copyBytes(to pointer: UnsafeMutablePointer<UInt8>, from range: CountableRange<Index>)

  func copyBytes<DestinationType>(to buffer: UnsafeMutableBufferPointer<DestinationType>, from range: CountableRange<Index>? = default) -> Int

  subscript(index: Index) -> UInt8 { get }

  subscript(bounds: Range<Int>) -> RandomAccessSlice<DispatchData> { get }

  func subdata(in range: CountableRange<Index>) -> DispatchData

  func region(location: Int) -> (data: DispatchData, offset: Int)

  var startIndex: Index { get }

  var endIndex: Index { get }

  func index(before i: Index) -> Index

  func index(after i: Index) -> Index

  func makeIterator() -> Iterator

  typealias IndexDistance = Int

  typealias _Element = UInt8

  typealias SubSequence = RandomAccessSlice<DispatchData>

  typealias _ObjectiveCType = __DispatchData

}

struct DispatchDataIterator : IteratorProtocol, Sequence {

  mutating func next() -> _Element?

  typealias Element = _Element

  typealias Iterator = DispatchDataIterator

  typealias SubSequence = AnySequence<_Element>

}

struct DispatchQoS : Equatable {

  let qosClass: DispatchQoS.QoSClass

  let relativePriority: Int

  static let background: DispatchQoS

  static let utility: DispatchQoS

  static let defaultQoS: DispatchQoS

  static let userInitiated: DispatchQoS

  static let userInteractive: DispatchQoS

  static let unspecified: DispatchQoS

  enum QoSClass {

    case background

    case utility

    case defaultQoS

    case userInitiated

    case userInteractive

    case unspecified

    var hashValue: Int { get }

  }

  init(qosClass: DispatchQoS.QoSClass, relativePriority: Int)

}

infix func ==(a: DispatchQoS.QoSClass, b: DispatchQoS.QoSClass) -> Bool

func ==(a: DispatchQoS, b: DispatchQoS) -> Bool

infix func ==(a: DispatchQoS.QoSClass, b: DispatchQoS.QoSClass) -> Bool

struct DispatchQueueAttributes : OptionSet {

  let rawValue: UInt64

  init(rawValue: UInt64)

  static let serial: DispatchQueueAttributes

  static let concurrent: DispatchQueueAttributes

  static let qosUserInteractive: DispatchQueueAttributes

  static let qosUserInitiated: DispatchQueueAttributes

  static let qosDefault: DispatchQueueAttributes

  static let qosUtility: DispatchQueueAttributes

  static let qosBackground: DispatchQueueAttributes

  static let noQoS: DispatchQueueAttributes

  typealias Element = DispatchQueueAttributes

  typealias RawValue = UInt64

}

final class DispatchSpecificKey<T> {

  init()

}

struct DispatchTime {

  let rawValue: dispatch_time_t

  static func now() -> DispatchTime

  static let distantFuture: DispatchTime

}

enum DispatchTimeInterval {

  case seconds(Int)

  case milliseconds(Int)

  case microseconds(Int)

  case nanoseconds(Int)

}

struct DispatchWalltime {

  let rawValue: dispatch_time_t

  static func now() -> DispatchWalltime

  static let distantFuture: DispatchWalltime

  init(time: timespec)

}

func +(time: DispatchTime, interval: DispatchTimeInterval) -> DispatchTime

func +(time: DispatchTime, seconds: Double) -> DispatchTime

func +(time: DispatchWalltime, interval: DispatchTimeInterval) -> DispatchWalltime

func +(time: DispatchWalltime, seconds: Double) -> DispatchWalltime

func -(time: DispatchTime, interval: DispatchTimeInterval) -> DispatchTime

func -(time: DispatchTime, seconds: Double) -> DispatchTime

func -(time: DispatchWalltime, interval: DispatchTimeInterval) -> DispatchWalltime

func -(time: DispatchWalltime, seconds: Double) -> DispatchWalltime

class DispatchWorkItem {

  init(group: DispatchGroup? = default, qos: DispatchQoS = default, flags: DispatchWorkItemFlags = default, block: () -> ())

  func perform()

  func wait(timeout: DispatchTime = default) -> Int

  func wait(timeout: DispatchWalltime) -> Int

  func notify(queue: DispatchQueue, execute: @convention(block) () -> Void)

  func cancel()

  var isCancelled: Bool { get }

}

struct DispatchWorkItemFlags : OptionSet, RawRepresentable {

  let rawValue: UInt

  init(rawValue: UInt)

  static let barrier: DispatchWorkItemFlags

  static let detached: DispatchWorkItemFlags

  static let assignCurrentContext: DispatchWorkItemFlags

  static let noQoS: DispatchWorkItemFlags

  static let inheritQoS: DispatchWorkItemFlags

  static let enforceQoS: DispatchWorkItemFlags

  typealias Element = DispatchWorkItemFlags

  typealias RawValue = UInt

}
```

## Impact on existing code

All Swift code that uses libdispatch via the current C API will be affected by this change.

## Alternatives considered

The alternative here was to leave the libdispatch API as it is currently imported in C. This proposal aims to improve the experience of using libdispatch in Swift and we did not feel that leaving the API as-is was a viable alternative.
