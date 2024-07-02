# A Vision for ABI Stability on non-Apple platforms

## Introduction

ABI or _Application Binary Interface_ refers to the details of how an
application will interact with some other piece of code, including:

* Calling conventions
* Layout of data structures
* Representations of scalar types
* Availability of APIs

If a piece of code has a _stable ABI_, that means that application
programs can rely on there being no incompatible changes between the
state of the ABI when they were being compiled and the ABI provided at
runtime by the component they were linked with.

End users of consumer operating systems, including both Apple and
Microsoft platforms, tend to take ABI stability for granted; that is,
they expect that they can install software onto their machine and that
operating system updates will not arbitrarily break that software.  It
would be easy to fool yourself into thinking, therefore, that ABI
stability was the default, or that it was easy, whereas in practice
considerable time and effort goes into making sure that things stay
this way.

A related concept is that of the _ABI boundary_, which is the level at
which a given component provides a stable ABI.  For instance, on Apple
and Microsoft operating systems, the stable ABI is provided by system
libraries; while it is possible to dip below this level, for instance
by doing direct system calls, no guarantee is made that the system
call interface is ABI stable on those platforms---it is not part of
the ABI boundary.  Contrast that with Linux, where the system call
interface _is_ an ABI boundary, while some widely used libraries may
not themselves be ABI stable _at all_.

## What Does ABI Stability Buy Us?

Put simply, it allows for shared binary components that get updated
separately.  This is typically a requirement for consumer operating
systems and software developed for them, so that software purchased by
consumers can be guaranteed to operate across a wide range of
operating system versions.

These shared components might be system software or language runtimes
on the one hand, but ABI stability also enables other patterns like
binary plug-ins, which need a stable ABI from their host application.

## Swift and ABI Stability

At time of writing, Swift is officially ABI stable only on Apple
platforms, where it forms part of the operating system, and indeed
many operating system components are written in Swift.

Apple's basic approach to ABI stability on its platforms relies on
being able to tell the compiler the minimum system version on which
the code it is building should run, coupled with the ability to
annotate API with `@availability` annotations describing which system
versions support which APIs.  The availability mechanism also allows
the compiler to decide automatically whether to weak link the
symbol(s) in question.

That does not completely solve the problem, however, because the
platform needs to be able to evolve the data types it exposes to
applications, while allowing applications to subclass Swift or
Objective-C types declared in the platform API.

You may be familiar with the C++ "fragile base class" problem, which
is essentially that a subclass of a C++ class needs to know the layout
(or at least the size) of the base class, which implies that the
latter cannot change.  In C++ the traditional solution to that is the
_`pimpl` idiom_ (essentially, you make the base class have a single
pointer member that points to its actual data).

Both Swift and its predecessor language Objective-C solve this problem
in a different way, namely for ABI stable structs and classes (all
classes in Objective-C, but in Swift only `public` non-`frozen` types
compiled with _library evolution_ enabled), the _runtime_ is in charge
of data layout, and the compiler, when asked to access a specific data
member, will make runtime calls as necessary to establish where it
should read or write data.  This avoids the extra indirection of the
C++ `pimpl` idiom and in practice it's possible to generate reasonably
efficient code.

## Platforms and ABI Stability

### Darwin

In addition to the `@availability`/minimum system version and
non-fragile base class support, the Darwin linker has a number of
features intended to allow code to be moved from one framework or
dynamic library to another without an ABI break, including support for
re-exporting symbols, as well as a variety of meta symbols that allow
for manipulation of symbol visibility and indeed apparent location
based on system versions.

### Windows

Like Apple platforms, Windows is ABI stable at the API layer; SPIs are
not officially ABI stable (though in practice they may be), and
neither is the system call interface itself.

Windows has a platform-wide exception handling mechanism (Structured
Exception Handling or SEH), which means that the operating system
specifies in detail how stack unwinding must take place.  Unlike other
platforms we care about, Windows does not use DWARF for exception
unwinding, but instead relies on knowledge of the standard function
prologue and epilogue.  This creates problems for tail call
optimization, which is also needed for Swift Concurrency.  While we
have *a* solution in place, we need to be convinced that it's the
*right* solution before declaring ABI stability on Windows.

Windows DLLs also have some interesting design features that need to
be borne in mind here.  In particular, symbols exported from DLLs can
be exported by 16-bit _ordinal_ as well as by name; name exports are
actually done by mapping the name to an ordinal, and then looking up
the ordinal in the export table.  If symbols are exported by ordinal
alone, the ordinals need to be stable in order to achieve ABI
stability.  Further, because of the way linking works, people have in
the past generated import libraries directly from DLL files, which
tends to result in import-by-ordinal rather than import-by-name, which
has actually forced Microsoft to stabilise ordinals when it otherwise
would not have done so.

Windows DLLs are also, like dylibs on Darwin, able to re-export
symbols from other DLLs (Windows calls this feature _forwarding_); the
Windows feature is also able to rename the symbol at the same time,
and indeed the Win32 layer re-exports some NT SPIs directly as Win32
APIs using this mechanism.

As regards data structure ABI stability, Windows uses two different
mechanisms here; the first is that data structures typically start
with a header that contains a size field (which gets used,
essentially, as a version).  The second is use of the Component Object
Model (COM), which relies on the way the Microsoft compiler lays out
C++ vtables.

Windows doesn't have anything quite like the `availability` mechanism
Apple platforms use, though there are C preprocessor macros like
`WINVER` or `_WIN32_WINNT` that control exactly which versions of data
structures and which functions are exported by the Windows SDK headers.

Windows programs that need to cope with the possibility that a function
might not be present at runtime need to use `GetProcAddress()` (the
Win32 equivalent of `dlsym()`).

### Linux

Linux is a more complicated situation.  The Linux kernel's system call
interface *is* ABI stable, though it isn't necessarily uniform across
supported platforms (there are system calls where the exact set of and
order of arguments is platform specific).  The Swift Static SDK for
Linux relies on this feature in that it generates binaries that
directly talk to the Linux kernel, albeit via a statically linked C
library.

Beyond that, though, the platform _as a whole_ is not ABI stable,
though individual dynamic libraries might be (Glibc is a case in
point), and particular distributions might guarantee some degree of
ABI stability for each major release (e.g. a program built for Ubuntu
18.04 will likely run on Ubuntu 18.10, but might not work on Ubuntu
22.04).  Even if all the libraries you are using are themselves ABI
stable, the versions of libraries installed by different distributions
may differ---distribution X may have a newer version of Glibc, but an
older version of (say) zlib, while distribution Y has an older Glibc
but a newer zlib.  And some distributions make different choices about
what "the system C library" or "the system C++ library" might be (this
isn't limited to low-level libraries either---similar choices might
happen for much higher layers of the system, such as which window
system to use, whether to use MIT Kerberos or Heimdall, and so on).

The upshot is that it is difficult to distribute binary programs for
Linux, without doing what the Swift Static SDK for Linux does and
simply not relying on external dynamic libraries.  This difficulty has
also led to the creation of systems like Snaps and Flatpak, which let
an application bundle all its dependencies in such a way that they
won't interfere with other installed applications.  It is also a
factor in the rise in popularity of OCI images, particularly for
server applications.

There are some system features intended to support ABI stability at
the library level, including versioned shared object names and symbol
versioning.  These get used by ABI stable libraries like Glibc such
that it's generally possible to run a program that depends on such a
library on the version it was built for _or any newer version_.  This
does make it possible to ship binaries that will run on a wide variety
of Linux systems by ensuring that your program is linked against the
oldest version(s) of dynamic libraries that you care about, _provided
those libraries are themselves ABI stable_.  Python's "manylinux"
package builds take this approach, but have to restrict themselves as
a result to a small subset of libraries that are known to be ABI
stable.

## Goals

* Swift should support existing platform ABI stability mechanisms,
  so that the language is a good citizen when writing native software.

  - `@availability` should be well-defined for Windows; it should be
    possible to annotate Windows APIs to say e.g. that they were
    introduced in Windows 10 version 1709.  Swift should automatically
    generate the code to weakly reference functions it imports where
    necessary.

  - `@availability` as a system-wide concept doesn't make as much
    sense on non-Apple UNIX/UNIX-like platforms where it's possible to
    install different versions of many standard packages.  This is
    especially true on Linux where there are multiple "distributions",
    so the major versions of packages in major versions of those
    distributions are fixed, but one distribution's set of packages
    may bear little relation to another's.  In this case,
    `@availability` annotations could perhaps be used for the system C
    library and also for the Swift runtime itself.

  - It would be good to be able to support Linux-style symbol
    versioning on ELF platforms. [^1]

[^1]: See Ulrich Drepper's [_How to Write Shared
    Libraries_](https://archive.org/download/dsohowto/dsohowto.pdf),
    which goes into considerable detail on the Linux ABI stability
    story, albeit mostly focusing on Glibc.

* Swift should provide an ABI stable runtime and standard library
  wherever it makes sense to do so.

  - It is already ABI stable on Apple platforms.

  - It makes a great deal of sense for it to be ABI stable on Windows.

  - On other UNIX/UNIX-like platforms, we should aim for it to be ABI
    stable when dynamically linked, even if underlying libraries (like
    the C library) are not.  This would at least mean that programs
    that just use the standard library would work as long as a
    suitable copy of the Swift runtime was present on the system.

* It should be possible to write ABI stable APIs _in_ Swift.

## Non-Goals

At this point, the following are explicitly not goals:

* Providing `@availability`-like support for third party library code.
  This is an interesting idea, but would require a method of
  specifying the minimum version, which probably means being able to
  specify version numbers when importing modules, e.g.
  e.g. `@version(12.3) import Libfoo` or `import Libfoo@12.3` or
  similar.  It would also complicate the compiler's availability
  tracking code considerably.
