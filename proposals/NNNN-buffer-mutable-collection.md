# Implement MutableCollection in UnsafeMutable{Raw}BufferPointer with non-mutating methods and accessors

* Proposal: [SE-NNNN](NNNN-buffer-mutable-collection.md)
* Authors: [Guillaume Lessard](https://github.com/glessard)
* Review Manager: TBD
* Status: [Prototype branch](https://github.com/glessard/swift/tree/buffer-mutable-collection-gyb)

## Introduction

`UnsafeMutableBufferPointer` and `UnsafeMutableRawBufferPointer` are non-owning
interfaces to underlying memory storage, as are their underlying types,
`UnsafeMutablePointer` and `UnsafeMutableRawPointer`. They implement the
`MutableCollection` on behalf of their underlying storage as a convenience. In
keeping with Swift's emphasis on value semantics, `MutableCollection` methods
are declared `mutating`. In the case of `UnsafeMutableBufferPointer` and
`UnsafeMutableRawBufferPointer`, this means that they must be declared `var`
most of the time, even when in reality there is no mutation of the
struct itself.

Swift-evolution thread:
[tbd](https://lists.swift.org/pipermail/swift-evolution/)

## Motivation

`UnsafeMutableBufferPointer` and `UnsafeMutableRawBufferPointer` must be
declared `var` at most points of use, even when there is no actual mutation of
the variable. In contrast, `UnsafeMutablePointer` and `UnsafeMutableRawPointer`,
to which the `BufferPointer` types are utility front-ends, do not have any
mutable functions or variables.

Because of the (unnecessarily) mutable interface, methods such as
`Array.withUnsafeMutableBufferPointer()` must require closures where a
`BufferPointer` parameter must be marked `inout`. In such cases, `inout` falsely
suggests that one can substitute a new buffer in place of the original one. In
actuality, the implementation of `Array.withUnsafeMutableBufferPointer()` traps
with the message `Fatal error: Array withUnsafeMutableBufferPointer: replacing
the buffer is not allowed` when any change is made to the
`UnsafeMutableBufferPointer` struct (as opposed to changes to the actual memory
buffer).

There are a few bugs and PRs along these lines:
[SR-3782](https://bugs.swift.org/browse/SR-3782),
[SR-3513](https://bugs.swift.org/browse/SR-3513),
[PR-12504](https://github.com/apple/swift/pull/12504),
[PR-13071](https://github.com/apple/swift/pull/13071).

## Proposed solution

The author proposes adding a protocol, tentatively named
`_BufferMutableCollection`, which extends `MutableCollection` by redeclaring its
interface as non-mutating methods and accessors. No other API is otherwise
added. This new protocol would adopted by the `BufferPointer` types, and
would declare some relevant default implementations, for the use of
`UnsafeMutableBufferPointer` and `UnsafeMutableRawBufferPointer`.

`Array.withUnsafeMutableBufferPointer()`'s closure parameter's signature will
change to have a `let`, rather than `inout`, parameter.

This will remove unnecessary mutation markers from code, thereby clarifying the
semantics of the `BufferPointer` types.

## Detailed design

The new protocol derive from `MutableCollection`, duplicating its the interface
with exclusively `nonmutating` versions of its functions and subscripts.
```
public protocol _BufferMutableCollection: MutableCollection
{
  subscript(position: Index) -> Element { get nonmutating set }

  subscript(bounds: Range<Index>) -> SubSequence { get nonmutating set }

  nonmutating func partition(
    by belongsInSecondPartition: (Element) throws -> Bool
  ) rethrows -> Index

  nonmutating func swapAt(_ i: Index, _ j: Index)

  nonmutating func _withUnsafeMutableBufferPointerIfSupported<R>(
    _ body: (UnsafeMutableBufferPointer<Element>) throws -> R
  ) rethrows -> R?
}
```

Default implementations of these will be provided where appropriate,
e.g. `partition` and `_withUnsafeMutableBufferPointerIfSupported`.

Some functions are implemented as extensions on `MutableCollection`, without
being part of the protocol interface. These would be implemented on the new
protocol as well:
```
extension _BufferMutableCollection {
  public nonmutating func sort() { /*...*/ }

  public subscript<R: RangeExpression>(r: R) -> SubSequence
    where R.Bound == Index {
      get { /*...*/ }
      nonmutating set { /*...*/ }
  }
}

extension _BufferMutableCollection where Self: BidirectionalCollection {
  public nonmutating func reverse() { /*...*/ }
}
```

The `Array` function `withUnsafeMutabelBufferPointer` will be modified to take
a closure whose BufferPointer parameter is non-mutating:
```
extension Array {
  public mutating func withUnsafeMutableBufferPointer<R>(
    _ body: (UnsafeMutableBufferPointer<Element>) throws -> R
  ) rethrows -> R { /*...*/ }
}
```


## Source compatibility

The change to `Array.withUnsafeMutableBufferPointer()` will have source
compatibility implications, albeit minor: any explicitly typed closures will
have to have the `inout` keyword removed. The other changes do not have
compatibility implications, though warnings may appear where there previously
were none.

## Effect on ABI stability

This has an effect on the ABI of `UnsafeMutableBufferPointer`,
`UnsafeMutableRawBuferPointer` and `Array`.  The time to do this change
is before ABI stability is declared, if ever.

## Effect on API resilience

N/A

## Alternatives considered

The status quo could remain, despite the oddity of forcing the behaviour
of value semantics onto these reference types.
