# BitwiseCopyable

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Kavon Farvardin](https://github.com/kavon), [Guillaume Lessard](https://github.com/glessard), [Nate Chandler](https://github.com/nate-chandler)
* Review Manager: TBD
* Implementation: On `main` gated behind `-enable-experimental-feature BitwiseCopyable`

<!-- *During the review process, add the following fields as needed:*

* Implementation: [apple/swift#NNNNN](https://github.com/apple/swift/pull/NNNNN) or [apple/swift-evolution-staging#NNNNN](https://github.com/apple/swift-evolution-staging/pull/NNNNN)
* Decision Notes: [Rationale](https://forums.swift.org/), [Additional Commentary](https://forums.swift.org/)
* Bugs: [SR-NNNN](https://bugs.swift.org/browse/SR-NNNN), [SR-MMMM](https://bugs.swift.org/browse/SR-MMMM)
* Previous Revision: [1](https://github.com/apple/swift-evolution/blob/...commit-ID.../proposals/NNNN-filename.md)
* Previous Proposal: [SE-XXXX](XXXX-filename.md) -->

## Introduction

We propose a new marker protocol `BitwiseCopyable` that can be used
to identify types that can be moved or copied with direct calls to
`memcpy` and which require no special destroy operation.
When compiling generic code with such constraints,
the compiler can emit these efficient operations directly,
only requiring minimal overhead to look up the size of the value at runtime.
Alternatively, developers can use this constraint to selectively provide
high-performance variations of specific operations, such as bulk copying of a container.

**Note**:  The term "trivial" is used in
[SE-138](0138-unsaferawbufferpointer.md) and
[SE-0370](0370-pointer-family-initialization-improvements.md)
to refer to types with the property above.
The discussion below will explain why certain generic or resilient types that
are trivial will not in fact be `BitwiseCopyable`.

## Motivation

Swift can compile generic code into an unspecialized form in which
the compiled function receives a value and type information about that value.
Basic operations are implemented by the compiler as calls to
a table of "value witness functions."
Similarly, code that manipulates resilient types defined in another module
must invoke functions for all basic operations on values of such types.

This approach is flexible, but can represent significant overhead.
For example, using this approach to copy a buffer with a large number of `Int` values
requires a function call for each value.

Being able to write generic functions that are constrained to
`BitwiseCopyable` types allows the compiler (and in some cases,
the developer) to instead use highly efficient direct memory operations
in such cases.

The standard library already contains many examples of functions
that could benefit from such a concept, and more are being
proposed:

The `UnsafeMutablePointer.initialize(to:count:)` function introduced in
[SE-0370](0370-pointer-family-initialization-improvements.md)
could use a bulk memory copy whenever it statically knew that its
argument was `BitwiseCopyable`.

The proposal for [`StorageView`](nnnn-safe-shared-contiguous-storage.md)
includes the ability to copy items to or from potentially-unaligned storage,
which requires that it be safe to use bulk memory operations:
```swift
public func loadUnaligned<T: BitwiseCopyable>(
  fromByteOffset: Int = 0, as: T.Type
) -> T

public func loadUnaligned<T: BitwiseCopyable>(
  from index: Index, as: T.Type
) -> T
```

## Proposed solution

We add a new protocol `BitwiseCopyable` to the standard library:
```swift
@_marker public protocol BitwiseCopyable {}
```

Many basic types in the standard library will be explicitly marked as conforming to this protocol,
including fixed-size numeric types (integer, floating-point, and SIMD types) and pointer types.

The compiler will infer this protocol for any struct, enum, or tuple
where all of the stored members are known to be `BitwiseCopyable`.
Note that this includes generic types when the stored members are themselves suitably constrained,
but excludes resilient types, since we cannot know all of their stored members.

Developers can explicitly mark types as conforming to `BitwiseCopyable`
in order to make this property accessible to other modules.
The compiler will check any such conformance and emit a diagnostic
if the type may contain elements that are not known to be `BitwiseCopyable`.

## Detailed design<a name="detailed-design"/>

Our design first marks a number of core types as being bitwise copyable,
and then extends that to aggregate types.

### Standard library changes<a name="standard-library-changes"/>

The following protocols in the standard library will gain the `BitwiseCopyable` constraint:

- `FixedWidthInteger`
- `_Pointer`
- `SIMDStorage`, `SIMDScalar`, `SIMD`

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

### Additional BitwiseCopyable types

In addition to the standard library types marked above,
the compiler will recognize several other basic types as `BitwiseCopyable`:

* An `unowned(unsafe)` reference.
  Such references can be copied without reference counting operations.
  
* `@convention(c)` and `@convention(thin)` function types do not carry a reference-counted
  capture context, unlike other Swift function types,
  and can therefore be bitwise copyable.

### Automatic Inference for Aggregates

Structs, enums, and tuples whose elements are all `BitwiseCopyable` will
be automatically considered to be `BitwiseCopyable`.

For example, the compiler automatically derives a conformance of
```swift
struct Coordinate {
  var x: Int
  var y: Int
}
```
to `BitwiseCopyable` because `Int` is `BitwiseCopyable` and `Coordinate`
is not resilient (it's internal).

Similarly, the compiler automatically derives a conformance of
```swift
private enum PositionUpdate {
  case begin(Coordinate)
  case move(x_change: Int, y_change: Int)
  case end
}
```
to `BitwiseCopyable`.

This does not apply to resilient types:
When a struct type is referenced across a resilience barrier,
we may not know all of it's stored members,
and thus cannot safely assume it is bitwise copyable.

For generic types, we will only consider a property with
generic type to be `BitwiseCopyable` if it is suitably
constrained.
For example, with the following definition, `Box<Int>`
is in fact trivial but would not be inferred to be
`BitwiseCopyable`:

```swift
struct Box<Value> {
  var first: Value
}
```

### Explicit conformance to `BitwiseCopyable`<a name="explicit-conformance"/>

Nominal types can be explicitly declared to conform to `BitwiseCopyable`.
Such a declaration is checked when the module defining it is compiled.
An error will be raised unless the declaration would have
been inferred to be `BitwiseCopyable` according to the rules above.


### Resilient types

Conformance is not normally automatically derived for resilient types in other modules.
The local module does not have full information about the resilient type,
and cannot make assumptions about its layout or operations.

Developers can add manual conformances to obtain the benefits of bitwise copyability
to their public (or `@usableFromInline`) types:
```swift
extension Coordinate2 : BitwiseCopyable {}
```

Similary, `@frozen` types can have `BitwiseCopyable` conformance inferred
even across module boundaries:
```swift
@frozen
public struct Coordinate3 {
  var x: Int
  var y: Int
}
```

## Effect on ABI stability

The addition of the `BitwiseCopyable` constraint to either a type or a protocol
in a library will not cause an ABI break for users.

## Source compatibility

This addition of a new protocol will not impact existing source code that does not use it.

Removing the `BitwiseCopyable` marker from a type is source-breaking.
As a result, future versions of Swift may conform additional existing types to `BitwiseCopyable`,
but will not remove it from any type previously declared to conform to `BitwiseCopyable`.

## Effect on API resilience

Adding a `BitwiseCopyable` constraint on a generic type will not cause an ABI break.
As with any protocol, the additional constraint can cause a source break for users.

## Future Directions

### Automatic derivation of conditional conformances<a name="conditonal-conformance-derivation"/>

Consider a wrapper type
```swift
struct Box<Value> {
  var first: Value
}
```
It is clearly trivial whenever `Value` is `BitwiseCopyable`.

Such a conditional conformance can be added manually,
but in the future we may in some cases be able to
derive it automatically.
```swift
extension Box : BitwiseCopyable where Value : BitwiseCopyable {}
```

### MemoryLayout<T>.isBitwiseCopyable

In certain circumstances, it would be useful to be able to dynamically determine
whether a type conforms to `BitwiseCopyable`.
In order to allow that, a new field could be added to `MemoryLayout`.

### BitwiseMovable

Most Swift types have the property that their representation can
be relocated in memory with direct memory operations.
This could be represented with a `BitwiseMovable` property
that was handled similarly to `BitwiseCopyable`.

## Alternatives considered

### Alternate Spellings

**Trivial** is widely used within the compiler and Swift evolution discussions to refer to the property of bitwise copyability. `BitwiseCopyable`, on the other hand, is more self-documenting.

## Acknowledgments

This proposal has benefitted from discussions with John McCall, Joe Groff, Andrew Trick, Michael Gottesman, Guillaume Lessard, and Arnold Schwaigofer.
