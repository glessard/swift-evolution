# Bitwise Types (??)

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

All values of generic type in Swift today support basic capabilities of being transferred (copied or moved) and destroyed. The underlying concrete type of a generic value induces a unique routine for performing the copy or move. These routines are unique in-part because managed references can appear anywhere within the value and require extra bookkeeping during a copy. Thus, in general it is not safe to perform a simple byte-for-byte copy ("memcpy") of a `struct`, because copying a `class` reference additionally requires incrementing the object's reference count.

New kinds of type constraints `BitwiseCopyable` and `BitwiseMovable` are proposed to describe generic values that support these simple transfers. One of the key use-cases for these constraints is to provide safety and performance for low-level code, such those dealing with foreign-function interfaces. For example, many of the improvements for `UnsafeMutablePointer` within [SE-0370](0370-pointer-family-initialization-improvements.md) rely on there being a notion of a "trivial type" in the language. The term _trivial_ in that proposal refers to the byte-for-byte transferability that is proposed here.

<!-- Swift-evolution thread: [Discussion thread topic for that proposal](https://forums.swift.org/) -->

## Motivation

<!-- For example, copying a struct involves copying each stored property. If the concrete type is a class, a copy of the managed reference to the object is created, which involves reference-count bookkeeping. Thus, if a struct contains a  -->

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

TODO: put some examples throughout this, like a BitwiseCopyable Optional??

The reference types, like `class` and `actor`, may be represented as automatically managed pointers, so they cannot conform to `BitwiseCopyable`.

All function types are also reference types and thus _most_ are not `BitwiseCopyable`. There is currently only one exception: a `@convention(c)` function type in Swift is `BitwiseCopyable`, as there is no management required for C function pointers.

By their nature, types conforming to `BitwiseCopyable` always have an empty `deinit`.

## Detailed design

The `BitwiseCopyable` protocol has no visible requirements, similar to `Sendable`.

Conformance to `BitwiseCopyable` requires complete knowledge of all stored properties in the type.
In particular, this means you cannot add retroactive conformances of `BitwiseCopyable` to resilient types declared in a different module; only `@frozen` types from such modules are supported. Any kind of type defined within the same module can have retroactive conformances to `BitwiseCopyable`.

### The primitive types

<!-- Based on KnownStdlibTypes.def -->

The following built-in types in Swift are considered to be primitive: 

- `Bool`
<!-- - `StaticString` NOTE: it is, but no need to list it. -->
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

The addition of conformances to `BitwiseCopyable` in a library will not cause an ABI break for users.

## Source compatibility

This addition of a new protocol will not impact existing source code that does not use it. 

There is a wrinkle in the source compatability story for `BitwiseCopyable` types. Protocols in Swift typically do not define _negative_ requirements, i.e., that a kind of member does not exist. The `BitwiseCopyable` protocol, in essence, defines an unbounded number of negative requirements.

A consequence of allowing retroactive conformances to non-resilient types defined in other modules is that a new kind of source compatability issue can appear. Suppose an API vendor conditionally includes a stored property for a type, say, depending on the platform:

```swift
// vendor's type that varies per-platform:
@frozen public struct Tree {
  var left: RawPointer
  var right: RawPointer
  var size: UInt
#if macOS
  var nodeName: NSString = "?"
#endif
}
```

For clients using this API, `Tree` can now only be made retroactively `BitwiseCopyable` on some platforms:

```swift
// only when compiling for macOS, this conformance is invalid,
// because String is not BitwiseCopyable!
extension Tree: BitwiseCopyable {}
```

In this case, the conditions for when member is included can be mirrored on the client side with an `#if`-guard around the conformance. But, that's not always possible as the conditions can be arbitrary and unknown to clients. Thus, clients may be lulled into believing that their code is cross-platform, when it silently is not.

The only other protocol in Swift that expresses a negative requirement is `Sendable`, but retroactive conformances to `Sendable` are disallowed; only `@unchecked Sendable` is allowed retroactively.

## Effect on API resilience

Adding a `BitwiseCopyable` constraint on a generic type will not cause an ABI break.
As with any protocol, the additional constraint can cause a source break for users.

## Alternatives considered

Herein lies some modifications or additions left out of this proposal.

### Require direct conformance to `BitwiseCopyable`

One solution to the Source Compatability problem described earlier is to disallow retroactive conformances to `BitwiseCopyable` for types defined in a different module, even if they are non-resilient.

In addition, various levels of relaxed checking for retroactive conformances, like an `@unchecked BitwiseCopyable`, might be worth considering to allow clients to adopt the protocol when they are reasonably sure it is correct.


<!-- 
## Acknowledgments
This proposal benefitted from discussions with John McCall. -->