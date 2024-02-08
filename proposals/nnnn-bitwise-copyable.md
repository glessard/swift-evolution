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

We propose a new marker protocol `BitwiseCopyable` to identify trivial[^1] types.
Values of such types can be moved or copied with direct calls to `memcpy` and require no special destroy operation.
When compiling generic code with such constraints, the compiler can emit these efficient operations directly, only requiring minimal overhead to look up the size of the value at runtime.
Alternatively, developers can use this constraint to selectively provide high-performance variations of specific operations, such as bulk copying of a container.

[^1]: The term "trivial" is used in [SE-138](0138-unsaferawbufferpointer.md) and [SE-0370](0370-pointer-family-initialization-improvements.md) to refer to this property of a type's layout.

## Motivation

Support for unspecialized generic code is a fundamental feature of Swift.
And providing safety and performance for low-level code is a key goal for Swift.
Providing a means to work with generic values that consist just of bytes _as_ bytes is a step towards that goal which extends that fundamental feature.

Already, the standard library provides a number of functions of generic values that require those values to be "trivial"[^2].
And more are proposed for [`StorageView`](nnnn-safe-shared-contiguous-storage.md).
In particular, it proposes packing instances of such types into contiguous memory without padding for alignment.

[^2]: For example, many of the improvements for `UnsafeMutablePointer` within [SE-0369](0370-pointer-family-initialization-improvements.md) rely on there being a notion of a "trivial" type in the language to ensure the `Pointee` is safe to copy bit-for-bit (e.g., `UnsafeMutablePointer.initialize(to:)`).

Specifically, it proposes the following API:

```
public func loadUnaligned<T: BitwiseCopyable>(
  fromByteOffset: Int = 0, as: T.Type
) -> T

public func loadUnaligned<T: BitwiseCopyable>(
  from index: Index, as: T.Type
) -> T
```

Swift should provide a language-level abstraction to allow types to advertise this capability.

## Proposed solution

To allow types to advertise their triviality (or "bitwise copyability"), a new protocol `BitwiseCopyable` is proposed.

Why a protocol?
Protocols allow generic code to abstract over types that all provide some capability.
A typical protocol requires some associated functions and types.
When a generic value is constrained to the protocol, it enjoys the use of the capabilities the protocol requires of its conformers.
In order for a type to conform to the protocol, it must implement and specify the required functions and types.
The compiler checks that the type does in fact do so.

Some types can be copied bitwise.
To abstract over types that provide this capability, `BitwiseCopyable` is proposed as a new standard library protocol.

```swift
@_marker public protocol BitwiseCopyable {}
```

When a generic value is constrained to the protocol, it enjoys having the value operations being expressible in terms of `memcpy`.
When a type conforms to `BitwiseCopyable`, it expresses that it has this capability.
And the compiler checks that it actually does.

Many types in the standard library will gain a conformance to the protocol.
The list of standard library types that will be `BitwiseCopyable` includes
- numeric types such as the integer types, the floating-point types, and the SIMD types.
- pointer types
- optional, conditionally.

For an exhaustive list, see [Detailed design](#detailed-design).
Future versions of Swift may conform additional existing types to `BitwiseCopyable`, but types that have been declared to conform to `BitwiseCopyable` will never lose that conformance.

Additionally, the compiler will automatically derive conformance to the protocol in many cases.
As a result, many types outside the standard library will begin conforming to `BitwiseCopyable` immediately[^3].

[^3]: As soon as they are built with a compiler in which this feature is enabled.

## Detailed design<a name="detailed-design"/>

When a generic value is constrained to `BitwiseCopyable`, the value operations for that value will be carried out more efficiently and can copy into and out of unaligned storage.
A large collection of types conforming to the protocol is built up, mostly automatically.
Because the collection is large, these advantages will be widely realized.

### Value operations

The effect of constraining a generic value to `BitwiseCopyable` is to change how its value operations are performed.

#### Decreased overhead

Consider the following function involving value operations on a generic value:

```swift
func passTwice<T>(_ t: consuming T) {
  take(copy t)
  see(t)
}
```

where its callees have the signatures

```swift
func take<T>(_ _: consuming T)
func see<T>(_ _: borrowing T)
```

The parameter `t` to the function is `consuming`.
Among other things, that means it must be consumed within `passTwice`.
Because there is another use of `t` after its use as a `consuming` argument to `take`, a copy is required.
Because the last use of `t` is as a `borrowing` argument to `see`, it is implicitly destroyed after that use.

All told, then, the value operations in this function are described in the following pseudocode:

```swift
func passTwice<T>(_ t: consuming T) {
  let t2 = copy_value(t) // value operation
  take(t2)
  see(t)
  destroy_value(t) // value operation
}
```

When `T` is not constrained to `BitwiseCopyable`, the value operations expand to dynamic function dispatch:

```swift
func passTwice<T>(_ t: consuming T) {
  // copy_value expands to...
  let size = T.ValueOperations[SizeIndex]
  let t2 = alloca(size)
  let init_with_copy = T.ValueOperations[InitWithCopyIndex]
  init_with_copy(&t2, &t)

  take(t2)
  see(t)

  // destroy_value expands to...
  let destroy = T.ValueOperations[DestroyIndex]
  destroy(&t)
}
```

If, however, `T` _is_ constrained to `BitwiseCopyable`, one value operation expands to a direct call to `memcpy` and the other disappears entirely:

```swift
func passTwice<T : BitwiseCopyable>(_ t: consuming T) {
  // copy_value expands to...
  let size = T.ValueOperations[SizeIndex]
  let t2 = alloca(size)
  let t2 = memcpy(&t2, &t, T.ValueOperatations)

  take(t2)
  see(t)

  // destroy_value expands to...
  // nothing!
}
```

This change decreases runtime overhead and code-size.

#### Unaligned value operations

Consider a different function involving value operations and potentially unaligned memory:

```swift
func store<T>(_ t: consuming T, toReinitialize pointer: UnsafeRawPointer) {
  pointer.withMemoryBound(to: T.self) { destination in
    destination = t
  }
}
```

where `withMemoryBound` is a fictional API with the signature

```swift
extension UnsafeRawPointer {
  func withMemoryBound<T>(to type: T.self, (inout T) -> Void)
}
```

In the closure passed to `withMemoryBound`, the original value at `destination` has to be destroyed and then the new value has to be stored[^4].
So the value operations in this function look as follows:

[^4]: In fact, these two operations are folded together into a single value operation in this case, "assign with copy".  This value operation is also implemented with `memcpy` for generic values conforming to `BitwiseCopyable`.  To simplify the example, that detail is omitted.

```swift
func store<T>(_ t: consuming T, toReinitialize pointer: UnsafeRawPointer) {
  pointer.withMemoryBound(to: T.self) { destination in
    destroy_value(destination)
    destination = copy_value(t)
  }
}
```

When `T` is not constrained to `BitwiseCopyable`, the value operations expand to dynamic function dispatch:

```swift
func store<T>(_ t: consuming T, toReinitialize pointer: UnsafeRawPointer) {
  pointer.withMemoryBound(to: T.self) { destination in
    // destroy_value expands to...
    let destroy = T.ValueOperations[DestroyIndex]
    destroy(&destination)

    // copy_value expands to...
    let init_with_copy = T.ValueOperations[InitWithCopyIndex]
    init_with_copy(&destination, &t)
  }
}
```

The `destroy` and `init_with_copy` value witness require that their arguments be aligned, however.
So this function can only be called correctly if `pointer` is aligned!

If `T` _is_ constrained to `BitwiseCopyable`, one value operation disappears and the other expands to a direct call to `memcpy`:

```swift
func store<T>(_ t: consuming T, toReinitialize pointer: UnsafeRawPointer) {
  pointer.withMemoryBound(to: T.self) { destination in
    // destroy_value expands to...
    // nothing!

    // copy_value expands to...
    let size = T.ValueOperations[SizeIndex]
    memcpy(&destination, &t, size)
  }
}
```

Once again, no dynamic dispatch to value witnesses occurs.
And because `memcpy` has no alignment requirements, calling this function is correct, regardless of whether `pointer` is aligned.

### Hierarchy of conformers

Primitive types which can be copied bitwise are conformed to `BitwiseCopyable`([Builtin module changes](#builtin-module-changes)) within the compiler.
A type may conform to `BitwiseCopyable` if it is merely an aggregate (enum or struct) of [suitable trivial values](#bitwise-copyable-values).
When a type is declared to conform to `BitwiseCopyable`, the compiler checks that it is such an aggregate (see [Explicit conformance to `BitwiseCopyable`](#explicit-conformance)).
Much of the time, the compiler will automatically generate conformances for such aggregates (see [Automatic derivation to `BitwiseCopyable`](#automatic-derivation)).

#### Explicit conformance to `BitwiseCopyable`<a name="explicit-conformance"/>

When a nominal type is declared to conform to `BitwiseCopyable`, the compiler will check that instances of the type can in fact be copied bitwise.

Any type that involves reference counting is not trivial: 
a copy requires incrementing the reference count; a destroy requires decrementing it.
And all reference types are reference counted.
So nominal reference types--classes and actors--cannot be conformed.

That leaves only structs and enums.
These both can conform to `BitwiseCopyable`--doing so requires that the values they aggregate together are [suitably trivial](#bitwise-copyable-values).

[Many standard library types](extending-existing-stdlib-types) are `BitwiseCopyable`, including `Int`.
So the struct

```
public struct Point : BitwiseCopyable {
  var x: Int
  var y: Int
}
```

can be conformed to `BitwiseCopyable`.
Why?
Because both of its fields `x` and `y` are of type `Int` which conforms to `BitwiseCopyable`.

Similarly, the enum

```
public enum Simplex : BitwiseCopyable {
case zero(Point)
case one(Point, Point)
}
```

can be conformed to `BitwiseCopyable`.
The enum has two associated values: `Point` and `(Point, Point)`.
The former was conformed to `BitwiseCopyable` above.
The latter conforms too because it's a tuple both of whose elements conform (see [Builtin module changes](builtin-module-changes)).

#### Automatic derivation of `BitwiseCopyable`<a name="automatic-derivation"/>

In many cases, the compiler will automatically derive a type's conformance to `BitwiseCopyable`.
This is done by attempting to conform unconditionally each non-resilient type defined in the module.
If the check determines that a type is trivial, it will gain a conformance to `BitwiseCopyable`.

#### Non-resilient aggregates

The fundamental automatic derivation behavior--to which there are a few exceptions--is the behavior for non-resilient aggregates described below.

#### Struct conformance

Given an non-resilient struct `S`, the compiler will automatically derive a conformance of `S` to `BitwiseCopyable` if and only if every field is [suitable](#bitwise-copyable-values).

For example, the compiler automatically derives a conformance of

```
struct Coordinate {
  var x: Int
  var y: Int
}
```

to `BitwiseCopyable` because:

- `Coordinate` is not resilient (it's internal)
- `x` is `BitwiseCopyable` (`Int` conforms to `BitwiseCopyable`, see [Adding `BitwiseCopyable` to existing standard library types](#extending-existing-stdlib-types))
- `y` is `BitwiseCopyable` (`Int` conforms)

#### Enum conformance

Similarly, given a non-resilient enum `E`, the compiler will automatically derive a conformance of `E` to `BitwiseCopyable` if and only if every associated value of `E` is a [suitable](#bitwise-copyable-values).

For example, the compiler automatically derives a conformance of

```
private enum PositionUpdate {
  case begin(Coordinate)
  case move(x_change: Int, y_change: Int)
  case end
}
```

to `BitwiseCopyable` because
- `PositionUpdate` is not resilient (it's private)
- `Coordinate` is `BitwiseCopyable`
- `(x_change: Int, y_change: Int)` is `BitwiseCopyable`

Note that the `end` case has no associated value which would have to conform.  This does not obstruct automatic conformance.

#### Generic derivation

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

such a conformance must be written manually.  Another proposal could lift that restriction (see [Automatic derivation of conditional conformance](#conditonal-conformance-derivation)).

#### Resilient non-derivation<a name="resilient-non-derivation"/>

When a module is built without library evolution, conformances are derived for `public` enums and structs as above.

Another case where conformance is not automatically derived is for resilient types.
These are public (or `@usableFromInline`) non-`@frozen` types defined in a module built for library evolution.
For example, the following type defined in such a module

```
public struct Coordinate2 {
  var x: Int
  var y: Int
}
```

would not enjoy automatic derivation, despite the fact that it is copyable bitwise.
Because as the library evolves, the type may cease to be trivial, the compiler will not derive a conformance for the type based on its current triviality.
Nevertheless, a manual conformance can be added:

```
extension Coordinate2 : BitwiseCopyable {}
```

Declaring that the type conforms is a promise that the type will remain `BitwiseCopyable` regardless of how else the library may evolve.
This is a promise the compiler must not make on the author's behalf.

Alternatively, if the type will never change, it can be marked `@frozen`

```
@frozen
public struct Coordinate3 {
  var x: Int
  var y: Int
}
```

In this case, conformance will once again be automatically derived because both fields are `BitwiseCopyable`.

#### Values aggregable by BitwiseCopyable conformers<a name="bitwise-copyable-values"/>

As described above, a struct or enum may conform to `BitwiseCopyable` when each value it aggregates is a "suitable" trivial value.
When is a trivial value "suitable"?

There are two cases:

(1) A value of a type which conforms to `BitwiseCopyable`.

This is the vast majority situation.
In fact, the intuition for the hierarchy of types conforming to `BitwiseCopyable` is that it starts from primitive conformers and builds from there via the operation of aggregation.

(2) A value which is decorated `unowned(unsafe)`.

Any sort of _managed_ reference to a reference type is a value that _cannot_ be copyable bitwise.
That includes strong, `weak`, and `unowned` references.
It does _not_ include `unowned(unsafe)` references however.

As a result, an aggregate which contains such a field can still conform:

```
public struct UnsafeWrapper : BitwiseCopyable {
  unowned(unsafe) var value: AnyObject
}
```

This conformance is legal because copying an `unowned(unsafe)` reference only involves copying its bits;
no reference counting occurs as part of the copy.

On the other hand, when is a trivial value _not_ "suitable"?
Whenever the value is trivial but its type doesn't conform to `BitwiseCopyable` (except for case (2) above).

There are two cases worth mentioning:

(1) A value of a generic type which could conditionally conform bound at generic arguments at which it would conform.

For example, given the `Box` type [above](#conditonal-conformance-derivation), a value of type `Box<Int>` is trivial.
If no conformance of `Box` to `BitwiseCopyable` is written manually, though, because conditional conformances are not derived, `Box` will not conform to `BitwiseCopyable`.

(2) A value of a resilient, currently trivial type.

For example, given the `Coordinate2` type [above](#resilient-non-derivation), a value of type `Coordinate2` used within the module is trivial.
If no conformance of `Coordinate2` to `BitwiseCopyable` is written manually, because it's resilient, it will not conform to `BitwiseCopyable`.

## Builtin module changes<a name="builtin-module-changes"/>

<!-- Based on KnownStdlibTypes.def -->

The following built-in types and kinds of values in Swift are considered to be
primitive, thus they implicitly satisfy `BitwiseCopyable`:

TODO: Enumerate the types.

The tuple type conforms to `BitwiseCopyable` conditionally.  It conforms if all of its elements conform.

Function types are also reference types and thus they are not generally `BitwiseCopyable`.
There are two exceptions: `@convention(c)` and `@convention(thin)` function types in Swift are `BitwiseCopyable` because there is no reference counted capture context associated with such functions.

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
  - `Unmanaged`
  - `Optional<T>` when `T` is `BitwiseCopyable`

TODO: Finish enumerating the types.

## Effect on ABI stability

The addition of the `BitwiseCopyable` constraint to either a type or a protocol in a library will not cause an ABI break for users.

## Source compatibility

This addition of a new protocol will not impact existing source code that does not use it.

## Effect on API resilience

Adding a `BitwiseCopyable` constraint on a generic type will not cause an ABI break.
As with any protocol, the additional constraint can cause a source break for users.

## Future Directions

### Automatic derivation of conditional conformances<a name="conditonal-conformance-derivation"/>

As proposed, only _unconditional_ conformances are derived automatically.
The same is true of derivation of conformances to `Sendable`.
As a result, some types which are conditionally trivial will not gain a conformance unless one is written explicitly.

Consider a wrapper type

```
struct Box<Value> {
  var first: Value
}
```

It is trivial whenever `Value` is `BitwiseCopyable`

As proposed, a conditional conformance can be written manually:

```
extension Box : BitwiseCopyable where Value : BitwiseCopyable {}
```

### MemoryLayout<T>.isBitwiseCopyable

In certain circumstances, it would be useful to be able to dynamically determine whether a type conforms to `BitwiseCopyable`.
In order to allow that, a new field could be added to `MemoryLayout`.

### BitwiseMovable

Another fact about a type's memory layout that could be leveraged to write safe code is whether the type can be _moved_ just by writing copying its bits into a new buffer.
This would enable having "borrowed properties" which store the bits of a value but don't perform any reference counting.

As it turns out, almost all values in Swift are bitwise movable.
There is a straightforward path to expand the implementation of BitwiseCopyable to BitwiseMovable when such features are proposed.

## Alternatives considered

### Trivial

"Trivial" is widely used within the compiler and Swift evolution discussions to refer to the property of bitwise copyability.
`BitwiseCopyable`, on the other hand, is more self-documenting.

## Acknowledgments

This proposal has benefitted from discussions with John McCall, Joe Groff, Andrew Trick, Michael Gottesman, Guillaume Lessard, and Arnold Schwaigofer.
