# BitwiseCopyable and BitwiseMovable

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

All values of generic type in Swift today support basic capabilities of being copied, moved, and destroyed. A generic value is associated with routines and additional checking when performing such operations to abstract over the underlying concrete type.
These routines are needed in-part because references can appear anywhere within the underlying concrete value and those references require extra bookkeeping during a copy or destroy.
But when working with a value of concrete type, Swift can take advantage of knowledge that a value does not contain references, skipping the extra work and performing a simple bit-for-bit copy ("memcpy") or a no-operation destruction.

This proposal defines new constraints `BitwiseCopyable` and `BitwiseMovable` to describe generic values that support these simple kinds of operations.
One of the key use-cases for these constraint is to provide safety and performance for low-level code, such those dealing with foreign-function interfaces and serialization.
For example, many of the improvements for `UnsafeMutablePointer` within [SE-0370](0370-pointer-family-initialization-improvements.md) rely on there being a notion of a "trivial type" in the language to ensure the `Pointee` is safe to copy bit-for-bit (e.g., `UnsafeMutablePointer.initialize(to:)`). The term _trivial_ in that proposal corresponds to the `BitwiseCopyable` constraint discussed here.

## Motivation

<!-- For example, copying a struct involves copying each stored property. If the concrete type is a class, a copy of the reference to the object is created, which involves reference-count bookkeeping. Thus, if a struct contains a  -->

TODO: Craft a motivating example using SE-370???

TODO: two unsafe buffer pointers copy to and from instead of a loop for each element.
want to optimize the code doing this stuff with memcpy.

```swift
func copyAll<T>(from: UnsafeBufferPointer<T>, to: UnsafeMutableBufferPointer<T>) {
  // a manual implementation of `from.copyBytes(to: to)` ?

  if val is BitwiseCopyable {
    // do a memcpy with confidence.
  } else {
    // ... less optimized approach
  }
}

```

Other motvation: need this for evolution proposal to disallow casts to prevent casts from UnsafePointer to UnsafeRawPointer because the type is nontrivial.

UnsafeRawPointer.storeBytes  ?

Ask Andy and Guilluame

Andy has a WWDC talk about this.


<!-- 
Use-cases include:
- Working with FFIs.
- Serialization APIs, such as those writing stuff over the wire? 
-->

## Proposed solution

New type constraints `BitwiseCopyable` and `BitwiseMovable` are proposed for Swift that describe some characteristic of the value's representation in memory. This new kind of constraint is called a *layout constraint*. They work like usual type constraints in that you can constrain a generic parameter with `some BitwiseCopyable`, but you cannot form an existential of a layout constraint like `any BitwiseCopyable`. An existential erases type information and changes the representation of the underlying value to a universial boxed representation, so it is not the case that `any BitwiseCopyable` is itself bit-for-bit copyable.


 A nominal type satisfies either `BitwiseCopyable` if it is a copyable struct or enum where all of its stored properties or associated values satisfy at least one of the following requirements:

- Its type is implicitly `BitwiseCopyable`.
- Its type satisfies `BitwiseCopyable`.
- Its type is a tuple where all elements satisfy `BitwiseCopyable`.

In the context of this proposal, a _primitive type_ is a built-in type defined by the language to never be represented by any managed pointers.
This set of types include `Int` and `Double`. 

<!-- This proposal assumes the deinit must also not be explicitly defined. Happens to be the case today anyway since you can't define a deinit on these types. -->

Satisfying `BitwiseCopyable` requires complete knowledge of all stored properties in the type.
In particular, this means you cannot add retroactive conformances of `BitwiseCopyable` to resilient types declared in a different module; only `@frozen` types from such modules are supported. Any kind of type defined within the same module can have retroactive conformances to `BitwiseCopyable`.

See Detailed design for the full list of types and additional minutiae. 

By their nature, types satisfying `BitwiseCopyable` can support bit-for-bit copying:

TODO: example involving SE-370 that is now possible with BitwiseCopyable to answer the issues raised in the Motivation section.

```swift
func f() {
  weak let x = H()
  weak let y = x
}
```

TODO: everything but the following are BitwiseMovable:
- c++ types are movable, but not bitwise movable.
- weak references are BitwiseMovable but not BitwiseCopyable
  - Swift weak references are BitwiseMovable but not BitwiseCopyable.
  - only difference is objc and swift weak refs, we don't differentiate in the type system between them.
- pthread_mutex_t is not movable at all. x
- not bitwise movable to internal pointers / llvm::SmallString goes from stack to heap allocated pointer.


## Detailed design

The `BitwiseCopyable` protocol has no visible requirements, similar to the marker protocol `Sendable`.
There are two key differences between a layout constraint and a marker protocol. 
The first is that a layout constraint has runtime metadata associated with it to support dynamic casts:

```swift
func copyFromAnyInto(_ val: Any, _ pointer: UnsafeMutableRawPointer) {
  if val is BitwiseCopyable {
    // do a memcpy with confidence.
  } else {
    // ... less optimized approach
  }
}
```

The second difference is that whether a type satisfies a layout constraint can be automatically derived by the compiler.

## Automatic derivation of `BitwiseCopyable`

A type does not always have to explicitly declare that it satisfies the layout constraint `BitwiseCopyable` in some cases:

- The type is marked `@frozen`
- The type is defined in the same module where a value of the type requires `BitwiseCopyable`

For example:

```swift
// Defined in module A
struct Coordinate {
  var x: Int
  var y: Int
}

func copyInto<T: BitwiseCopyable>(_ t: T, _ pointer: UnsafeMutableRawPointer) { /* ... */ }

// Call appears in module A
copyInto(Coordinate(), pointer) // OK
```
A limited form of this kind of derviation occurs for determining whether a type is `Sendable`, but `BitwiseCopyable` goes further to handle generic types like `Optional<T>`:

```swift
let x: Optional<AnyObject> = nil
let y: Optional<Coordinate> = nil

copyInto(x, pointer) // error: Optional<AnyObject> does not satisfy BitwiseCopyable
copyInto(y, pointer) // OK
```

Specifically, for a generic struct or enum, the decision procedure is performed ad-hoc only when all generic parameters are bound to a type:

```swift
struct MyGenericType<T> {
  var prop: T
}

func copyGeneric<Elm>(_ z: MyGenericType<Elm>) 
  where Elm: BitwiseCopyable {
  copyInto(z, pointer) // error: function 'copyInto' requires that 'MyGenericType<T>' conform to 'BitwiseCopyable'
}

let fullyBound: MyGenericType<Int> =  // ...
copyInto(fullyBound, pointer) // OK
```

In this example, `Elm` is an unbound generic type within the body of `copyGeneric`, so a call to `copyInto` passed a value of type `MyGenericType<Elm>` will not trigger a derivation of `BitwiseCopyable`, despite `Elm` being constrained to `BitwiseCopyable`.

Automatic derivation proceeds using a very similar recursive decision procedure as for proving a type satisfies `BitwiseCopyable`.
The only difference is that types with an explicitly `@unchecked BitwiseCopyable` conformance are not considered to be `BitwiseCopyable` during derivation.
The family of Unsafe types in the Swift standard library have an `@unchecked BitwiseCopyable` conformance.
Excluding such types during automatic derivation helps ensure that using unsafe constructs remains explicitly opt-in rather than implicit.
Take for example a type that keeps an internal pointer into itself using one of the Unsafe types:

```swift
struct FlattenedTree {
  var data: [UInt8]  // TODO: is this the right kind of buffer?
  var childrenStart: UnsafeRawPointer
  var metadataStart: UnsafeRawPointer

  func copy() { /* ... */ }
}
```

It would be incorrect to make a bit-for-bit copy of `FlattenedTree` because `childrenStart` is actually an absolute pointer into buffer stored in the same struct. Thus, copying the buffer correctly requires not only making a copy of these pointers, but re-calculating their addresses based on where the copy resides in memory.
The author of `FlattenedTree` knows this so a conformance to `BitwiseCopyable` was not written on this type.
But if automatic derivation were permitted for this type, users of the type may accidentially mis-use the type.

## Conforming to `BitwiseCopyable`

The reference types, like `class` and `actor`, may be represented as automatically managed pointers, so they cannot conform to `BitwiseCopyable`. This extends to `weak` and `unowned` storage containing such a reference type, as they still require bookkeeping during a copy. Only `unowned(unsafe)` is truly free of any safety and thus bookkeeping, but will not be considered as `BitwiseCopyable` during automatic derivation.

All function types are also reference types and thus they are not generally `BitwiseCopyable`. There is only one exception: a `@convention(c)` function type in Swift is `BitwiseCopyable`, as there is no management required for C function pointers.

### The primitive types

<!-- Based on KnownStdlibTypes.def -->

The following built-in types and kinds of values in Swift are considered to be 
primitive, thus they implicitly satisfy `BitwiseCopyable`:

- `Bool`
- Any `@convention(c)` function.
- storage marked `unowned(unsafe)`
- The fixed-precision integer types:
  - `Int8`, `Int16`, `Int32`, `Int64`, `Int`
  - `UInt8`, `UInt16`, `UInt32`, `UInt64`, `UInt`
- The fixed-precision floating-point types: 
  - `Float`, `Double`, `Float80`

### Adding `BitwiseCopyable` to existing standard library types

The following protocol types in the standard library will gain the `BitwiseCopyable` 
constraint:

- `FixedWidthInteger`
- `_Pointer`
- `SIMDStorage`, `SIMDScalar`, `SIMD`

The following value types in the standard library will gain the `BitwiseCopyable` 
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

## Alternatives considered

Herein lies some modifications or additions left out of this proposal.

### Require direct conformance to `BitwiseCopyable`

One solution to the Source Compatability problem described earlier is to disallow retroactive conformances to `BitwiseCopyable` for types defined in a different module, even if they are non-resilient.

In addition, various levels of relaxed checking for retroactive conformances, like an `@unchecked BitwiseCopyable`, might be worth considering to allow clients to adopt the protocol when they are reasonably sure it is correct.


## Acknowledgments
This proposal has benefitted from discussions with John McCall, Joe Groff, Andrew Trick, Michael Gottesman, and Guillaume Lessard.