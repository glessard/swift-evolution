# BitwiseCopyable

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Kavon Farvardin](https://github.com/kavon)
* Review Manager: TBD
* Implementation: On `main` gated behind `-enable-experimental-feature BitwiseCopyable`

- if <T: BitwiseCopyable> { pointerToT.loadUnaligned(...) }
- Andy wants BWC available in the public interfaces of APIs. Look for isPOD tests. 


<!-- *During the review process, add the following fields as needed:*

* Implementation: [apple/swift#NNNNN](https://github.com/apple/swift/pull/NNNNN) or [apple/swift-evolution-staging#NNNNN](https://github.com/apple/swift-evolution-staging/pull/NNNNN)
* Decision Notes: [Rationale](https://forums.swift.org/), [Additional Commentary](https://forums.swift.org/)
* Bugs: [SR-NNNN](https://bugs.swift.org/browse/SR-NNNN), [SR-MMMM](https://bugs.swift.org/browse/SR-MMMM)
* Previous Revision: [1](https://github.com/apple/swift-evolution/blob/...commit-ID.../proposals/NNNN-filename.md)
* Previous Proposal: [SE-XXXX](XXXX-filename.md) -->

## Introduction

All values of generic type in Swift today support basic capabilities of being copied, moved, and destroyed. A generic value is associated with routines and additional checking when performing such operations to abstract over the underlying concrete type.
These routines are needed in-part because references can appear anywhere within the underlying concrete value and those references require extra bookkeeping during a copy or destroy.
But when working with a value of concrete type, Swift can take advantage of knowledge that a value does not contain references, skipping the extra work and performing a simple bit-for-bit copy ("memcpy") or a no-operation destruction.

This proposal defines a new marker protocol `BitwiseCopyable` to describe generic values that support these simple operations.
One of the key use-cases for this constraint is to provide safety and performance for low-level code, such that dealing with foreign-function interfaces and serialization.
For example, many of the improvements for `UnsafeMutablePointer` within [SE-0370](0370-pointer-family-initialization-improvements.md) rely on there being a notion of a "trivial type" in the language to ensure the `Pointee` is safe to copy bit-for-bit (e.g., `UnsafeMutablePointer.initialize(to:)`). The term _trivial_ in that proposal corresponds to the `BitwiseCopyable` constraint discussed here.

## Motivation

<!-- For example, copying a struct involves copying each stored property. If the concrete type is a class, a copy of the reference to the object is created, which involves reference-count bookkeeping. Thus, if a struct contains a  -->

TODO: Craft a motivating example using SE-370???

TODO: two unsafe buffer pointers copy to and from instead of a loop for each element.
want to optimize the code doing this stuff with memcpy.

Other motivation: need this for evolution proposal to disallow casts to prevent casts from UnsafePointer to UnsafeRawPointer because the type is nontrivial.

UnsafeRawPointer.storeBytes  ?

Ask Andy and Guilluame

Andy has a WWDC talk about this.


<!-- 
Use-cases include:
- Working with FFIs.
- Serialization APIs, such as those writing stuff over the wire? 
-->

## Proposed solution

Ordinary type constraints, like a protocol, describe capabilities in the form of required members for a type, such as methods, computed properties, and nested types.
The ability to copy a value of some type bit-for-bit is a fact about a conforming type's memory layout.
A type's layout, however, may change as a library evolves.

Thus, `BitwiseCopyable` is proposed as a new marker protocol.  
The compiler will automatically derive conformance to the protocol in many cases.
And even when conformance of a type to the protocol is declared manually, the compiler will check that its fields are all bitwise copyable.

A nominal type conforms to the protocol `BitwiseCopyable` if it is an non-resilient, escapable, copyable struct or enum all of whose stored properties or associated values satisfy conform to `BitwiseCopyable`.

Many types in the standard library will gain a conformance to the protocol.  
The list of standard library types that will be `BitwiseCopyable` include:

- `Bool`
- The fixed-precision integer types:
  - `Int8`, `Int16`, `Int32`, `Int64`, `Int`
  - `UInt8`, `UInt16`, `UInt32`, `UInt64`, `UInt`
- The fixed-precision floating-point types: 
  - `Float`, `Double`, `Float80`
- the family of `SIMDx<Scalar>` types
- Any `@convention(c)` function.
- a stored property marked `unowned(unsafe)`
- `StaticString`
- The family of unmanaged pointer types:
  - `OpaquePointer`
  - `UnsafeRawPointer`, `UnsafeMutableRawPointer`
  - `UnsafePointer`, `UnsafeMutablePointer`, `AutoreleasingUnsafeMutablePointer`
  - `UnsafeBufferPointer`, `UnsafeMutableBufferPointer`
  - `UnsafeRawBufferPointer`, `UnsafeMutableRawBufferPointer`
  - `Unmanaged<T>`
  - `Optional<T>` when `T` is `BitwiseCopyable`

This list is not exhaustive. Future versions of Swift may conform additional existing types to `BitwiseCopyable`, but types that have conformed to `BitwiseCopyable` will never lose that conformance.

Types satisfying `BitwiseCopyable` can safely support bit-for-bit copying:

```swift
func copyAll<T : BitwiseCopyable>(from: UnsafeBufferPointer<T>, 
                                  to: UnsafeMutableBufferPointer<T>) {
  guard from.count <= to.count else { fatalError("destination too small") }
  // We know it is safe to quickly memcpy this data as raw bytes!
  let fromRaw = UnsafeRawBufferPointer(from)
  let toRaw = UnsafeMutableRawBufferPointer(to)
  toRaw.copyMemory(from: fromRaw)  // effectively, a memcpy
}
```

## Detailed design

## Automatic derivation of `BitwiseCopyable`

Generally, automatic derivation of `BitwiseCopyable` occurs in the same situations as automatic derivation of `Sendable`.  Aggregates (structs and enums) that are local to a module enjoy automatic deriviation if all of their components (fields and associated values, respectively) conform.  The same applies to `@frozen` types exported from a module.  Finally, conformance is not derived automatically on a conditional basis.

In a bit more detail: a struct's conformance is automatically derived if all of its fields conforms.  For example, the struct

```
struct Coordinate {
  var x: Int
  var y: Int
}
```

conforms because both of its fields, `x` and `y` both conform (`Int` is `BitwiseCopyable`).  The same is true for an enum like `(x: Int, y: Int)`.

Similarly, an enum's conformance is automatically derived if all of its associated values conform.  For example, the enum

```
enum Change {
  case initial(Coordinate)
  case update(Vector)
}
```

because `Coordinate` and `Vector` both conform to `BitwiseCopyable`.

The same applies for generic types.  Conformance of this struct

```
struct Pair<Value : BitwiseCopyable> {
  var first: Value
  var second: Value
}
```

to `BitwiseCopyable` is derived automatically because both fields `first` and `second` are `BitwiseCopyable` because `Value` is constrained to conform to it.

On the other hand, conformance would _not_ be derived if `Value` were not constrained in this way.  _No_ conformance is automatically derived for the following struct

```
struct Pair2<Value> {
  var first: Value
  var second: Value
}
```

The reason is that neither `first` nor `second` conforms to `BitwiseCopyable` since `Value` is unconstrained.  While this type can be conformed to `BitwiseCopyable` on a conditional basis

```
extension Pair2 : BitwiseCopyable where Value : BitwiseCopyable {}
```

such a conformance must be written manually.  Another proposal could lift that restriction (see Future Directions).

Another case where conformance is not automatically derived is for non-`@frozen` public types.  For example, the following type

```
public struct Coordinate2 {
  var x: Int
  var y: Int
}
```

would not enjoy automatic derivation, despite the fact that both of its fields are `BitwiseCopyable`.  Because the type may change in the future, adding fields that are not `BitwiseCopyable`, the compiler will not conform the type automatically.  Nevertheless, you can add a conformance manually:

```
extension Coordinate2 : BitwiseCopyable {}
```

Declaring that the type conforms is a promise that the type will remain `BitwiseCopyable` regardless of the fields that are added or removed from it.

Alternatively, if the type will never change, it can be marked `@frozen`

```
@frozen
public struct Coordinate2 {
  var x: Int
  var y: Int
}
```

In this case, conformance will once again be automatically derived because both fields are `BitwiseCopyable`.

The only exception is that types with an explicitly `@unchecked` conformance to `BitwiseCopyable` are not considered to be `BitwiseCopyable` during derivation.
The family of Unsafe types in the Swift standard library have an `@unchecked BitwiseCopyable` conformance.
Excluding such types during automatic derivation helps ensure that using unsafe constructs remains explicitly opt-in rather than implicit.
Take for example a type that keeps an internal pointer into itself using one of the Unsafe types:

```swift
struct FlattenedTree {
  var data: Buffer<UInt8>
  var childrenStart: UnsafeRawPointer<UInt8>
  var metadataStart: UnsafeRawPointer<UInt8>

  func copy() { /* ... */ }
}
```

It would be incorrect to make a bit-for-bit copy of `FlattenedTree` because `childrenStart` is actually an absolute pointer into buffer stored in the same struct.  
Copying the buffer correctly requires not only making a copy of these pointers, but re-calculating their addresses based on where the copy resides in memory.
The author of `FlattenedTree` knows this, so a conformance to `BitwiseCopyable` was not written on this type.
But if automatic derivation were permitted for this type, users of the type may accidentially mis-use the type.

## Conforming to `BitwiseCopyable`

The reference types, like `class` and `actor`, may be represented as automatically managed pointers, so they cannot conform to `BitwiseCopyable`. This extends to `weak` and `unowned` storage containing such a reference type, as they still require bookkeeping during a copy. Only `unowned(unsafe)` is truly free of any safety and thus bookkeeping, but will not be considered as `BitwiseCopyable` during automatic derivation.

Function types are also reference types and thus they are not generally `BitwiseCopyable`. There are only two exception: `@convention(c)` and `@convention(thin)` function types in Swift are `BitwiseCopyable` because there is no reference counted capture context associated with such functions.

### The primitive types

<!-- Based on KnownStdlibTypes.def -->

The following built-in types and kinds of values in Swift are considered to be 
primitive, thus they implicitly satisfy `BitwiseCopyable`:



### Adding `BitwiseCopyable` to existing standard library types

The following protocol types in the standard library will gain the `BitwiseCopyable` 
constraint:

- `FixedWidthInteger`
- `_Pointer`
- `SIMDStorage`, `SIMDScalar`, `SIMD`

The following value types in the standard library will gain the `BitwiseCopyable` 
constraint:




## Effect on ABI stability

The addition of the `BitwiseCopyable` constraint to a type in a library will not
cause an ABI break for users.

## Source compatibility

This addition of a new protocol will not impact existing source code that does not use it. 

<!-- There is a wrinkle in the source compatability story for `BitwiseCopyable` types. Protocols in Swift typically do not define _negative_ requirements, i.e., that a kind of member does not exist. The `BitwiseCopyable` protocol, in essence, defines an unbounded number of negative requirements.

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

The only other protocol in Swift that expresses a negative requirement is `Sendable`, but retroactive conformances to `Sendable` are disallowed; only `@unchecked Sendable` is allowed retroactively. -->

## Effect on API resilience

Adding a `BitwiseCopyable` constraint on a generic type will not cause an ABI break.
As with any protocol, the additional constraint can cause a source break for users.

## Future Directions

* Conditional inference
* MemoryLayout<T>.isBitwiseCopyable
* BitwiseMovable

## Alternatives considered

Herein lies some modifications or additions left out of this proposal.

### Require direct conformance to `BitwiseCopyable`

One solution to the Source Compatability problem described earlier is to disallow retroactive conformances to `BitwiseCopyable` for types defined in a different module, even if they are non-resilient.

In addition, various levels of relaxed checking for retroactive conformances, like an `@unchecked BitwiseCopyable`, might be worth considering to allow clients to adopt the protocol when they are reasonably sure it is correct.


## Acknowledgments
This proposal has benefitted from discussions with John McCall, Joe Groff, Andrew Trick, Michael Gottesman, and Guillaume Lessard.
