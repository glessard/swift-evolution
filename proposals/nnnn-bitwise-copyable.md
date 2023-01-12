# BitwiseCopyable Types

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Kavon Farvardin](https://github.com/kavon)
* Review Manager: TBD
* Status: **Awaiting implementation**

<!-- *During the review process, add the following fields as needed:*

* Implementation: [apple/swift#NNNNN](https://github.com/apple/swift/pull/NNNNN) or [apple/swift-evolution-staging#NNNNN](https://github.com/apple/swift-evolution-staging/pull/NNNNN)
* Decision Notes: [Rationale](https://forums.swift.org/), [Additional Commentary](https://forums.swift.org/)
* Bugs: [SR-NNNN](https://bugs.swift.org/browse/SR-NNNN), [SR-MMMM](https://bugs.swift.org/browse/SR-MMMM)
* Previous Revision: [1](https://github.com/apple/swift-evolution/blob/...commit-ID.../proposals/NNNN-filename.md)
* Previous Proposal: [SE-XXXX](XXXX-filename.md) -->

## Introduction

Swift's implementation of generics uses an abstraction mechanism for transferring values of generic type. This abstraction is needed in-part because the underlying concrete type may contain managed pointers that require extra work during a copy (e.g., incrementing a reference count). For concrete types _without_ any managed pointers, the abstraction for copying the value just performs a `memcpy`, which is a simple bit-for-bit copy of the value without any pointer management.

A new built-in protocol `BitwiseCopyable` is proposed to describe generic types that do not contain any managed pointers. One of the key use-cases for this protocol is to provide safety for systems-level code, such those dealing with foreign-function interfaces. For example, many of the improvements for `UnsafeMutablePointer` within [SE-0370](0370-pointer-family-initialization-improvements.md) rely on there being a notion of a "trivial type" in the language. Until now, there was no such notion. Henceforth, the term "trivial" will not be used.

<!-- Swift-evolution thread: [Discussion thread topic for that proposal](https://forums.swift.org/) -->

## Motivation

TODO: Do we really need motivation when we can just point to SE-370?

Use-cases include:
- Working with FFIs.
- Serialization APIs, such as those writing stuff over the wire?

## Proposed solution

A new `BitwiseCopyable` marker protocol is proposed for Swift. A nominal type can conform to `BitwiseCopyable` if it is a copyable value-type where all of its stored properties satisfy at least one of the following requirements:

- Its type is primitive.
- Its type conforms to `BitwiseCopyable`.
- Its type is a tuple where all elements are `BitwiseCopyable`.

In the context of this proposal, a _primitive type_ is a built-in type defined by the language to never be represented by any managed pointers. This set of types include `Int` and `Double`. See Detailed design for the full list of types.

TODO: put some examples throughout this, like a BitwiseCopyable Optional.

The reference types, like `class` and `actor`, may be represented as automatically managed pointers, so they cannot conform to `BitwiseCopyable`.

All function types are also reference types and thus _most_ are not `BitwiseCopyable`. There is currently only one exception: a `@convention(c)` function type in Swift is `BitwiseCopyable`, as there is no management required for C function pointers.

By their nature, types conforming to `BitwiseCopyable` always have an empty `deinit`.

## Detailed design

The `BitwiseCopyable` protocol has no visible requirements, similar to `Sendable`.

Conformance to `BitwiseCopyable` requires complete knowledge of all stored properties in the type.
In particular, this means resilient types declared in a different module cannot retroactively conform to `BitwiseCopyable`; only `@frozen` types from such modules are supported.

TODO: I think we need the same-file rule about Sendable and non-resilient types in the same module?


### The primitive types

<!-- Based on KnownStdlibTypes.def -->

The following built-in types in Swift are considered to be primitive: 

- `Bool`
- `StaticString`
- Any `@convention(c)` function.
- The fixed-precision integer types:
  - `Int8`, `Int16`, `Int32`, `Int64`, `Int`
  - `UInt8`, `UInt16`, `UInt32`, `UInt64`, `UInt`
- The fixed-precision floating-point types: 
  - `Float`, `Double`, `Float80`
- The unmanaged pointer types:
  - `UnsafeMutableRawPointer`, `UnsafeRawPointer`
  - `UnsafeMutablePointer`, `UnsafePointer`
  - `OpaquePointer`, `AutoreleasingUnsafeMutablePointer`
  - `UnsafeBufferPointer`, `UnsafeMutableBufferPointer`
  - `UnsafeRawBufferPointer`, `UnsafeMutableRawBufferPointer`
  - `Unmanaged<T>`


## Effect on ABI stability

The addition of conformances to `BitwiseCopyable` will not cause an ABI break.

## Source compatibility

This is an additive change with no impact on source compatability.

## Effect on API resilience

Adding a `BitwiseCopyable` constraint on a generic type will not cause an ABI break.
As with any protocol, the additional constraint can cause a source break for users.

## Alternatives considered

None.

<!-- TODO:
## Acknowledgments

If significant changes or improvements suggested by members of the 
community were incorporated into the proposal as it developed, take a
moment here to thank them for their contributions. Swift evolution is a 
collaborative process, and everyone's input should receive recognition!
-->
