# BitwiseCopyable and TriviallyDestroyable

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

All values of generic type in Swift today support basic capabilities of being transferred (copied or moved) and destroyed. A generic value is associated with routines and additional checking when performing such transfers to abstract over the underlying concrete type.
These routines are needed in-part because managed references can appear anywhere within the underlying concrete value and those references require extra bookkeeping during a copy or destroy.
But when working with a value of concrete type, Swift can take advantage of knowledge that a value does not contain references, skipping the extra work and performing a simple bit-for-bit copy ("memcpy") or a no-operation destruction.

This proposal introduces a layout constraint `TriviallyDestroyable` to describe generic values that support these simple kinds of operations.
Its name refers only to trivial destruction because not all types support copying, but all types support destruction.
Having trivial destruction implies support for support bit-for-bit copying and are analogous to "plain-old data" in other languages.

One of the key use-cases for this constraint is to provide safety and performance for low-level code, such those dealing with foreign-function interfaces. For example, many of the improvements for `UnsafeMutablePointer` within [SE-0370](0370-pointer-family-initialization-improvements.md) rely on there being a notion of a "trivial type" in the language. The term _trivial_ in that proposal refers to the trivial destruction that is discussed here.

<!-- Swift-evolution thread: [Discussion thread topic for that proposal](https://forums.swift.org/) -->

## Motivation

<!-- For example, copying a struct involves copying each stored property. If the concrete type is a class, a copy of the managed reference to the object is created, which involves reference-count bookkeeping. Thus, if a struct contains a  -->

TODO: Do we really need motivation when we can just point to SE-370?

Use-cases include:
- Working with FFIs.
- Serialization APIs, such as those writing stuff over the wire?

## Proposed solution

A new `TriviallyDestroyable` layout constraint is proposed for Swift. You can use a layout constraint much in the same way as a marker protocol, such as constraining a generic parameter with `some TriviallyDestroyable` or forming an existential with `any TriviallyDestroyable`.

A nominal type satisfies `TriviallyDestroyable` if it is a struct or enum with an undefined deinit where all of its stored properties or associated values satisfy at least one of the following requirements:

- Its type is primitive.
- Its type satisfies `TriviallyDestroyable`.
- Its type is a tuple where all elements satisfy `TriviallyDestroyable`.

In the context of this proposal, a _primitive type_ is a built-in type defined by the language to never be represented by any managed pointers.
This set of types include `Int` and `Double`. See Detailed design for the full list of types and additional minutiae. 

By their nature, types satisfying `TriviallyDestroyable` can support bit-for-bit copying if the type is copyable. 
But `TriviallyDestroyable` alone does not assume copyability of its conformers. 
Since copyable types are quite common, a typealias `BitwiseCopyable` is provided for convenience in the standard library:

```swift
public typealias BitwiseCopyable = TriviallyDestroyable & Copyable
```

TODO: examples involving SE-370 that are now possible with TriviallyDestroyable to answer the issues raised in the Motivation section.


## Detailed design

There are two key differences between a layout constraint and a marker protocol. 
The first is that a layout constraint has runtime metadata associated with it to support dynamic casts:

```swift
func copyInto(_ val: Any, _ pointer: UnsafeMutableRawPointer) {
  if val is BitwiseCopyable {
    // do a memcpy with confidence.
  }
}
```

The second difference is that whether a type satisfies a layout constraints can
be automatically derived by the compiler.

TODO: more details on the auto-derivation procedure. 

TODO: then integrate the below into the above

The `TriviallyDestroyable` protocol has no visible requirements, similar to the marker protocol `Sendable`.

Satisfying `TriviallyDestroyable` requires complete knowledge of all stored properties in the type.
In particular, this means you cannot add retroactive conformances of `TriviallyDestroyable` to resilient types declared in a different module; only `@frozen` types from such modules are supported. Any kind of type defined within the same module can have retroactive conformances to `TriviallyDestroyable`.

The reference types, like `class` and `actor`, may be represented as automatically managed pointers, so they cannot conform to `TriviallyDestroyable`. This extends to `weak` and `unowned` storage containing such a reference type, as they still require bookkeeping during a copy. Only `unowned(unsafe)` is truly free of any safety and thus bookkeeping.

All function types are also reference types and thus they are not generally `TriviallyDestroyable`. There is only one exception: a `@convention(c)` function type in Swift is `TriviallyDestroyable`, as there is no management required for C function pointers.

### The primitive types

<!-- Based on KnownStdlibTypes.def -->

The following built-in types and kinds of values in Swift are considered to be 
primitive, thus they implicitly satisfy `TriviallyDestroyable`:

- `Bool`
- Any `@convention(c)` function.
- storage marked `unowned(unsafe)`
- The fixed-precision integer types:
  - `Int8`, `Int16`, `Int32`, `Int64`, `Int`
  - `UInt8`, `UInt16`, `UInt32`, `UInt64`, `UInt`
- The fixed-precision floating-point types: 
  - `Float`, `Double`, `Float80`

### Adding `TriviallyDestroyable` to existing standard library types

The following protocol types in the standard library will gain the `TriviallyDestroyable` 
constraint:

- `FixedWidthInteger`
- `_Pointer`
- `SIMDStorage`, `SIMDScalar`, `SIMD`

The following value types in the standard library will gain the `TriviallyDestroyable` 
constraint:

- `StaticString`
- The family of unmanaged pointer types:
  - `OpaquePointer`
  - `UnsafeRawPointer`, `UnsafeMutableRawPointer`
  - `UnsafePointer`, `UnsafeMutablePointer`, `AutoreleasingUnsafeMutablePointer`
  - `UnsafeBufferPointer`, `UnsafeMutableBufferPointer`
  - `UnsafeRawBufferPointer`, `UnsafeMutableRawBufferPointer`
  - `Unmanaged<T>`
- the family of `SIMDx<Scalar>` types


## Effect on ABI stability

The addition of the `TriviallyDestroyable` constraint to a type in a library will not
cause an ABI break for users.

## Source compatibility

This addition of a new protocol will not impact existing source code that does not use it. 

<!-- There is a wrinkle in the source compatability story for `TriviallyDestroyable` types. Protocols in Swift typically do not define _negative_ requirements, i.e., that a kind of member does not exist. The `TriviallyDestroyable` protocol, in essence, defines an unbounded number of negative requirements.

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

For clients using this API, `Tree` can now only be made retroactively `TriviallyDestroyable` on some platforms:

```swift
// only when compiling for macOS, this conformance is invalid,
// because String is not TriviallyDestroyable!
extension Tree: TriviallyDestroyable {}
```

In this case, the conditions for when member is included can be mirrored on the client side with an `#if`-guard around the conformance. But, that's not always possible as the conditions can be arbitrary and unknown to clients. Thus, clients may be lulled into believing that their code is cross-platform, when it silently is not.

The only other protocol in Swift that expresses a negative requirement is `Sendable`, but retroactive conformances to `Sendable` are disallowed; only `@unchecked Sendable` is allowed retroactively. -->

## Effect on API resilience

Adding a `TriviallyDestroyable` constraint on a generic type will not cause an ABI break.
As with any protocol, the additional constraint can cause a source break for users.

## Alternatives considered

Herein lies some modifications or additions left out of this proposal.

### Require direct conformance to `TriviallyDestroyable`

One solution to the Source Compatability problem described earlier is to disallow retroactive conformances to `TriviallyDestroyable` for types defined in a different module, even if they are non-resilient.

In addition, various levels of relaxed checking for retroactive conformances, like an `@unchecked TriviallyDestroyable`, might be worth considering to allow clients to adopt the protocol when they are reasonably sure it is correct.


<!-- 
## Acknowledgments
This proposal benefitted from discussions with John McCall, Joe Groff, etc. -->