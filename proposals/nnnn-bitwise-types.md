# Bitwise Transferred Types

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

Swift's implementation of generics uses an abstraction mechanism for copying and moving values of generic type. This abstraction is needed because the underlying concrete type may contain reference-counted values that must be retained during a copy. For concrete types _without_ any reference-counted values, copying the value is equivalent to doing a `memcpy`, which performs a simple bit-for-bit copy of the value without any retains. A new built-in protocol `BitwiseTransfer` is proposed to describe generic types that do not contain any references, so that the compiler can eliminate the extra overhead of abstraction.

<!-- Swift-evolution thread: [Discussion thread topic for that proposal](https://forums.swift.org/) -->

## Motivation

Use cases include:

- Working with FFIs, since unsafe buffers of the data are still safe to copy.
- Performance critical code that still wants to use generics.

## Proposed solution

Through the use of the `BitwiseTransfer` marker protocol, programmers can constrain generic types to ensure they only have simplistic copy operations.

TODO: Value witness table has things like:
- size*
- alignment*
- copy init
- copy assign
- move assign
- deinit *?

but for BitwiseTransfer types, only the starred ones need to be accessed, unless
if it's move-only.

CURIOUS: How does this mix with move-only types? I assume structs previously got an
empty entry for the deinit field, but now they do have something?

## Detailed design

TODO: use the name `BitwiseCopyable`. means it has no deinit also

TODO: noncopyable types only conform to `BitwiseMovable`

Conformers of `BitwiseCopyable == BitwiseTransfer == POD == Trivial` types must either be:
  - a trivial type
  - a value type consisting of stored properties that conform to BitwiseTransfer

The trivial types that implicitly conform to `BitwiseTransfer` consist of:
- Int
- Double
- UnsafeBufferPointer ???
- **Anything not ref-counted**
- ??


NOTE: "Trivial types" are already documented in the compiler / Library Evolution spec.

NOTE: @_specialize 

QUESTION: how will a marker protocol help us in the stdlib for situations like this:

```
enum Optional<T: BitwiseTransfer> {
  case none
  case some(T)
}
```

In the stdlib, we can't constrain `T` to be `BitwiseTransfer`. Same goes for just general generic code:

```swift
func merge<T>(_ l: [T], _ r: [T]) {
  if T is BitwiseCopyable {
    
  }
}
```

Because it's a protocol conformance expressed in user code here, that means there's only one version of the protocol. Should we have some auto-specializer thing generate two versions of the function??

Unless if we generate multiple specializations, one for BitwiseTransfer conformers and one for the general case, it seems as though no on

TODO: why we didn't call it trivial: the types are not required to have a `init()`.

TODO: take angle of dynamic type test examples for now, 

TODO: look for places in stdlib where we assert(isPOD()), that's more motivation.
the dynamic check can get eliminated if the compiler specializes a function.


## Source compatibility

Relative to the Swift 3 evolution process, the source compatibility
requirements for Swift 4 are *much* more stringent: we should only
break source compatibility if the Swift 3 constructs were actively
harmful in some way, the volume of affected Swift 3 code is relatively
small, and we can provide source compatibility (in Swift 3
compatibility mode) and migration.

Will existing correct Swift 3 or Swift 4 applications stop compiling
due to this change? Will applications still compile but produce
different behavior than they used to? If "yes" to either of these, is
it possible for the Swift 4 compiler to accept the old syntax in its
Swift 3 compatibility mode? Is it possible to automatically migrate
from the old syntax to the new syntax? Can Swift applications be
written in a common subset that works both with Swift 3 and Swift 4 to
aid in migration?

## Effect on ABI stability

The ABI comprises all aspects of how code is generated for the
language, how that code interacts with other code that has been
compiled separately, and how that code interacts with the Swift
runtime library.  It includes the basic rules of the language ABI,
such as calling conventions, the layout of data types, and the
behavior of dynamic features in the language like reflection,
dynamic dispatch, and dynamic casting.  It also includes applications
of those basic rules to ABI-exposed declarations, such as the `public`
functions and types of ABI-stable libraries like the Swift standard
library.

Many language proposals have no direct impact on the ABI.  For
example, a proposal to add the `typealias` declaration to Swift
would have no effect on the ABI because type aliases are not
represented dynamically and uses of them in code can be
straightforwardly translated into uses of the aliased type.
Proposals like this can simply state in this section that they
have no impact on the ABI.  However, if *using* the feature in code
that must maintain a stable ABI can have a surprising ABI impact,
for example by changing a function signature to be different from
how it would be without using the feature, that should be discussed
in this section.

Because Swift has a stable ABI on some platforms, proposals are
generally not acceptable if they would require changes to the ABI
of existing language features or declarations.  Proposals must be
designed to avoid the need for this.

For example, Swift could not accept a proposal for a feature which,
in order to work, would require parameters of certain (existing)
types to always be passed as owned values, because parameters are
not always passed as owned values in the ABI.  This feature could
be fixed by only enabling it for parameters marked a special new way.
Adding that marking to an existing function parameter would change
the ABI of that specific function, which programmers can make good,
context-aware decisions about: adding the marking to an existing
function with a stable ABI would not be acceptable, but adding it
to a new function or to a function with no stable ABI restrictions
would be fine.

Proposals that change the ABI may be acceptable if they can be thought
of as merely *adding* to the ABI, such as by adding new kinds of
declarations, adding new modifiers or attributes, or adding new types
or methods to the Swift standard library.  The key principle is
that the ABI must not change for code that does not use the new
feature.  On platforms with stable ABIs, uses of such features will
by default require a new release of the platform in order to work,
and so their use in code that may deploy to older releases will have
to be availability-guarded.  If this limitation applies to any part
of this proposal, that should be discussed in this section.

Adding a function to the standard library does not always require
an addition to the ABI if it can be implemented using existing
functions.  Library maintainers may be able to help you with this
during the code review of your implementation.  Adding a type or
protocol currently always requires an addition to the ABI.

If a feature does require additions to the ABI, platforms with
stable ABIs may sometimes be able to back-deploy those additions
to existing releases of the platform.  This is not always possible,
and in any case, it is outside the scope of the evolution process.
Proposals should usually discuss ABI stability concerns as if
it was not possible to back-deploy the necessary ABI additions.

## Effect on API resilience

API resilience describes the changes one can make to a public API
without breaking its ABI. Does this proposal introduce features that
would become part of a public API? If so, what kinds of changes can be
made without breaking ABI? Can this feature be added/removed without
breaking ABI? For more information about the resilience model, see the
[library evolution
document](https://github.com/apple/swift/blob/master/docs/LibraryEvolution.rst)
in the Swift repository.

## Alternatives considered

Describe alternative approaches to addressing the same problem, and
why you chose this approach instead.

## Acknowledgments

If significant changes or improvements suggested by members of the 
community were incorporated into the proposal as it developed, take a
moment here to thank them for their contributions. Swift evolution is a 
collaborative process, and everyone's input should receive recognition!
