# A Vision for WebAssembly Support in Swift

## Introduction

WebAssembly (abbreviated [Wasm](https://webassembly.github.io/spec/core/intro/introduction.html#wasm)) is a virtual 
machine instruction set focused on portability, security, and high performance. It is vendor-neutral, designed and
developed by [W3C](https://w3.org). An implementation of a WebAssembly virtual machine is usually called a
*WebAssembly runtime*.

One prominent spec-compliant implementation of a Wasm runtime in Swift is [WasmKit](https://github.com/swiftwasm/WasmKit). It is available as a Swift package, supports multiple
host platforms, and has a simple API for interaction with guest Wasm modules.

An application compiled to a Wasm module can run on any platform that has a Wasm runtime available. Despite its origins
in the browser, it is a general-purpose technology that has use cases in client-side and
server-side applications and services. WebAssembly support in Swift makes the language more appealing in those settings,
and also brings it to the browser where it previously wasn't available at all. It facilitates a broader adoption of
Swift in more environments and contexts.

We can't anticipate every possible application Swift developers are going to create with Wasm, but we can provide a few
examples of its possible adoption in the Swift toolchain itself. To quote
[a GSoC 2024 idea](https://www.swift.org/gsoc2024/#building-swift-macros-with-webassembly):

> WebAssembly could provide a way to build Swift macros into binaries that can be distributed and run anywhere,
> eliminating the need to rebuild them continually.

This can be applicable not only to Swift macros, but also for the evaluation of SwiftPM manifests and plugins.

The WebAssembly instruction set has useful properties from a security perspective, as it has
no interrupts or peripheral access instructions. Access to the underlying system is always done by calling
explicitly imported functions, implementations for which are provided by an imported WebAssembly module or a WebAssembly
runtime itself. The runtime has full control over interactions of the virtual machine with the outside world.

WebAssembly code and data live in completely separate address spaces, with all executable code in a given module loaded
and validated by the runtime upfront. Combined with the lack of "jump to address" and a limited set of control flow
instructions that require explicit labels in the same function body, this makes a certain class of attacks impossible to
execute in a correctly implemented spec-compliant WebAssembly runtime.

In the context of Swift developer tools, arbitrary code execution during build time can be virtualized with Wasm.
While Swift macros, SwiftPM manifests, and plugins are sandboxed on Darwin platforms, with Wasm we can provide stronger
security guarantees on other platforms that have a compatible Wasm runtime available.

The WebAssembly instruction set is designed with performance in mind. A WebAssembly module can be JIT-compiled or
compiled on a client machine to an optimized native binary ahead of time. With recently accepted proposals to the Wasm
specification it now supports features such as SIMD, atomics, multi-threading, and more. A WebAssembly runtime can
generate a restricted subset of native binary code that implements these features with little performance overhead.

This means that adoption of Wasm in developer tools does not imply unavoidable performance overhead. In fact, with
security guarantees that virtualization brings there's no longer a need to spawn a separate process for each
Swift compiler and SwiftPM plugin/manifest invocation. Virtualized Wasm binaries can run in the host process of a
Wasm runtime, removing the overhead of new process setup and IPC infrastructure.

### WebAssembly System Interface and the Component Model

The WebAssembly virtual machine has no in-built support for I/O; instead, a Wasm module's access to I/O is dependent entirely upon the runtime that executes it.

A standardized set of APIs implemented by a Wasm runtime for interaction with the host operating system is called
[WebAssembly System Interface (WASI)](https://wasi.dev). A layer on top of WASI that Swift apps compiled to Wasm can already
use thanks to C interop is [WASI libc](https://github.com/WebAssembly/wasi-libc). In fact, the current implementation of
Swift stdlib and runtime for `wasm32-unknown-wasi` triple is based on this C library. It is important for WASI support in
Swift to be as complete as possible to ensure portability of Swift code in the broader Wasm ecosystem.

In the last few years, the W3C WebAssembly Working Group considered multiple proposals for improving the WebAssembly
[type system](https://github.com/webassembly/interface-types) and
[module linking](https://github.com/webassembly/module-linking). These were later subsumed into a combined
[Component Model](https://component-model.bytecodealliance.org) proposal thanks to the ongoing work on
[WASI Preview 2](https://github.com/WebAssembly/WASI/blob/main/preview2/README.md), which served as playground for
the new design.

The Component Model defines these core concepts:

- A *component* is a composable container for one or more WebAssembly modules that have a predefined interface;
- *WebAssembly Interface Types (WIT) language* allows defining contracts between components;
- *Canonical ABI* is an ABI for types defined by WIT and used by component interfaces in the Component Model.

Preliminary support for WIT has been implemented in
[the `wit-tool` subcommand](https://github.com/swiftwasm/WasmKit/blob/0.0.3/Sources/WITTool/WITTool.swift) of the WasmKit
CLI. Users of this tool can generate `.wit` files from Swift declarations, and vice versa: Swift bindings from `.wit`
files.

## Goals

As of March 2024 all patches necessary for basic Wasm and WASI support have been merged to the Swift toolchain and
core libraries. Based on this, we propose a high-level roadmap for WebAssembly support and adoption in the Swift
ecosystem:

1. Ensure that Swift toolchain and core libraries have a regularly running test suite for supported Wasm and WASI
triples.

2. Allow Swift developers to easily install and build with a Swift SDK for WASI. This requires an implementation
of corresponding build scripts and CI jobs to generate and publish such SDK. Some parts of the Swift toolchain and core
libraries need a Swift SDK for running tests for WASI, so this will benefit the previous point in stabilizing support
for this platform.

3. Make it easier to evaluate and adopt Wasm with increased API coverage for this platform in the Swift core libraries. As a
virtualized embeddable platform, not all system APIs are always available or easy to port to WASI. For example,
multi-threading, file system access, and localization need special support in Wasm runtimes and a certain amount of
consideration from a developer adopting these APIs.

4. Improve support for cross-compilation in Swift and SwiftPM. We can simplify versioning, installation, and overall
management of Swift SDKs for cross-compilation in general, which is beneficial not only for WebAssembly, but for all
platforms.

5. Continue work on Wasm Component Model support in Swift as the Component Model proposal is stabilized. Ensure
that future versions of WASI are available to Swift developers targeting Wasm. A more ambitious long-term goal to
consider is making interoperability with Wasm components as smooth as C and C++ interop already is for Swift. With
a formal specification for Canonical ABI progressing, this goal will become more achievable with time.

### Proposed Language Features

In our work on Wasm support in Swift we experimented with a few function attributes that could be considered
as pitches and eventually Swift Evolution proposals, if the community is interested in their wider adoption.
These attributes allow easier interoperation between Swift code and other Wasm modules linked with it by a Wasm
runtime.

* [`@_extern(wasm)` attribute](https://github.com/apple/swift/pull/69107) that corresponds to Clang's
[`__attribute__((import_name("declaration_name")))`](https://clang.llvm.org/docs/AttributeReference.html#export-name)
and [`__attribute__((import_module("module_name")))`](https://clang.llvm.org/docs/AttributeReference.html#import-module).
This indicates that function's implementation is not provided together with its declaration in Swift and is
available externally, imported by a Wasm runtime during the linking phase. The declaration is added to current Wasm
module's imports section. Without `@_extern(wasm)` developers need to rely on C interop to create a declaration in a C
header using the Clang version of attributes.

* [`@_expose(wasm)` attribute](https://github.com/apple/swift/pull/68524) that corresponds to Clang's
[`__attribute__((export_name("name")))`](https://clang.llvm.org/docs/AttributeReference.html#export-name), which
is the counterpart of `@_extern(wasm)` working in the opposite direction. This explicitly makes a Swift declaration
available outside of the current module, added to its exports section under a given name. Again, the lack of
this attribute in Swift requires use of C headers as a workaround.
