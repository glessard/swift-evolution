# BitwiseCopyable

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Kavon Farvardin](https://github.com/kavon)
* Review Manager: TBD
* Implementation: On `main` gated behind `-enable-experimental-feature BitwiseCopyable`

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

To model this capability, `BitwiseCopyable` is proposed as a new marker protocol.  
When a type is declared to conform to the protocol, the compiler will check that it is in fact bitwise copyable.
Additionally, the compiler will automatically derive conformance to the protocol in many cases.

A nominal type may conform to the protocol `BitwiseCopyable` if it is an escapable, copyable struct or enum all of whose stored properties or associated values satisfy conform to `BitwiseCopyable`.

Many types in the standard library will gain a conformance to the protocol.  
The list of standard library types that will be `BitwiseCopyable` includes
- numeric types such as the integer types, the floating-point types, and the SIMD types.
- the pointer types
- optional, conditionally.
For an exhaustive list, see [Detailed design](#detailed-design).
Future versions of Swift may conform additional existing types to `BitwiseCopyable`, but types that have been conformed to `BitwiseCopyable` will never lose that conformance.

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

## Detailed design<a name="detailed-design"/>

Primitive types which can be copied bitwise are conformed to `BitwiseCopyable`--see [Builtin module changes](builtin-module-changes).
A type may conform to `BitwiseCopyable` whenever it is merely an aggregate (enum or struct) of values which are themselves `BitwiseCopyable`.
When a type is declared to conform to `BitwiseCopyable`, the compiler checks that it is such an aggregate (see [Explicit conformance to `BitwiseCopyable`](explicit-conformance)).
Much of the time, the compiler will automatically generate these conformances for such aggregates (see [Automatic derivation to `BitwiseCopyable`](#automatic-derivation)).

## Explicit conformance of `BitwiseCopyable`<a name="explicit-conformance"/>

When a nominal type is declared to conform to `BitwiseCopyable`, the compiler will check that the type is in fact "bitwise copyable".

Any type that involves reference counting is not "bitwise copyable".
All reference types are reference counted.
So nominal reference types--classes and actors--cannot be conformed.

That leaves only structs and enums, both of which can conform to `BitwiseCopyable`--so long as the types they aggregate together are themselves bitwise copyable.
By definition, all types that conform to `BitwiseCopyable` are bitwise copyable.
So types all of whose fields are of types which conform to `BitwiseCopyable` can themselves be conformed to the protocol.

And numerous standard library types (see [Existing standard library types](extending-existing-stdlib-types)) conform to the protocol.
One such conforming type from the standard library is `Int`.

Which means that the struct

```
public struct Point : BitwiseCopyable {
  var x: Int
  var y: Int
}
```

can be conformed to `BitwiseCopyable`.
Why?
Because both of its fields `x` and `y` are of type `Int` which conforms to `BitwiseCopyable` and which consequently are bitwise copyable.

Similarly, the enum

```
public enum Simplex : BitwiseCopyable {
case zero
case one(Int)
case two(Int, Int)
}
```

can be conformed to `BitwiseCopyable`.
The enum has two associated values: `Int` and `(Int, Int)`.
As discussed, the former conforms to `BitwiseCopyable`.  
The latter does too because it's a tuple both of whose elements conform (see [Builtin module changes](builtin-module-changes)).

An aggregate which contains any sort of managed reference to a reference type cannot conform to `BitwiseCopyable`.
That includes strong, `weak`, and `unowned` references.
It does _not_ include `unowned(unsafe)` references however.
As a result, an aggregate which contains such a field can still conform:

```
public struct UnsafeWrapper : BitwiseCopyable {
  unowned(unsafe) var value: AnyObject
}
```

Function types are also reference types and thus they are not generally `BitwiseCopyable`. There are two exceptions: `@convention(c)` and `@convention(thin)` function types in Swift are `BitwiseCopyable` because there is no reference counted capture context associated with such functions.

Classes 

## Automatic derivation of `BitwiseCopyable`<a name="automatic-derivation"/>

In many cases, the compiler will automatically derive a type's conformance to `BitwiseCopyable`.  
The automatic derivation to `BitwiseCopyable` is mostly analogous to the automatic derivation to `Sendable`.
Aggregates (structs and enums) that are local to a module enjoy automatic deriviation if all of their components (fields and associated values, respectively) conform.  
The same applies to `@frozen` types exported from a module.  
Finally, conformance is not derived automatically on a conditional basis.

### Unexported aggregates

The fundamental automatic derivation behavior--to which there are a few exceptions--is the behavior for aggregates which aren't exported from a module, described below.

#### Struct conformance

Given an unexported struct `S`, the compiler will automatically derive a conformance of `S` to `BitwiseCopyable` if and only if every field of `S` conforms to `BitwiseCopyable`.

For example, the compiler automatically derives a conformance of

```
struct Coordinate {
  var x: Int
  var y: Int
}
```

to `BitwiseCopyable` because:

- `Coordinate` is not exported (it's internal)
- `x` is `BitwiseCopyable` (`Int` conforms to `BitwiseCopyable`, see [Adding `BitwiseCopyable` to existing standard library types](#extending-existing-stdlib-types))
- `y` is `BitwiseCopyable` (`Int` conforms)

#### Enum conformance

Similarly, given an unexported enum `E`, the compiler will automatically derive a conformance of `E` to `BitwiseCopyable` if and only if every associated value of `E` conforms to `BitwiseCopyable`.

For example, the compiler automatically derives a conformance of 

```
private enum PositionUpdate {
  case begin(Coordinate)
  case move(x_change: Int, y_change: Int)
  case end
}
```

to `BitwiseCopyable` because
- `PositionUpdate` is not exported (it's private)
- `Coordinate` is `BitwiseCopyable`
- `Vector` is `BitwiseCopyable`

Note that the `end` case has no associated value which would have to conform.  This does not obstruct automatic conformance.

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

## Builtin module changes<a name="builtin-module-changes"/>

<!-- Based on KnownStdlibTypes.def -->

The following built-in types and kinds of values in Swift are considered to be 
primitive, thus they implicitly satisfy `BitwiseCopyable`:

The tuple type conforms to `BitwiseCopyable` conditionally.  It conforms if all of its elements conform.

## Standard library changes<a name="standard-library-changes"/>

### Existing standard library protocols<a name="extending-existing-stdlib-protocols"/>

The following protocols in the standard library will gain the `BitwiseCopyable` constraint:

- `FixedWidthInteger`
- `_Pointer`
- `SIMDStorage`, `SIMDScalar`, `SIMD`

### Existing standard library types<a name="extending-existing-stdlib-types"/>

The following types in the standard library will gain the `BitwiseCopyable` constraint:

- `Bool`
- The fixed-precision integer types:
  - `Int8`, `Int16`, `Int32`, `Int64`, `Int`
  - `UInt8`, `UInt16`, `UInt32`, `UInt64`, `UInt`
- The fixed-precision floating-point types: 
  - `Float`, `Double`, `Float80`
- the family of `SIMDx<Scalar>` types
- `StaticString`
- The family of unmanaged pointer types:
  - `OpaquePointer`
  - `UnsafeRawPointer`, `UnsafeMutableRawPointer`
  - `UnsafePointer`, `UnsafeMutablePointer`, `AutoreleasingUnsafeMutablePointer`
  - `UnsafeBufferPointer`, `UnsafeMutableBufferPointer`
  - `UnsafeRawBufferPointer`, `UnsafeMutableRawBufferPointer`
  - `Unmanaged<T>`
  - `Optional<T>` when `T` is `BitwiseCopyable`

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

## Acknowledgments
This proposal has benefitted from discussions with John McCall, Joe Groff, Andrew Trick, Michael Gottesman, and Guillaume Lessard.
