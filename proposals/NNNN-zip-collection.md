# Add Collection-conforming Zip

* Proposal: [SE-NNNN](NNNN-zip-collection.md)
* Authors: [Ben Cohen](https://github.com/airspeedswift)
* Review Manager: TBD
* Status: **Awaiting review**
* Implementation: [apple/swift#13941](https://github.com/apple/swift/pull/13941)

## Introduction

The standard library's `zip` function currently returns a sequence,
`Zip2Sequence`. But when both the arguments to `zip` are collections, 
a collection type could be returned. This proposal adds such a type.

## Motivation

`Collection` conformance brings many benefits (e.g. a `first` method, zero-cost
slicing operations, guarantees of multi-pass). Conformance to
`RandomAccessCollection` brings further benefits. There's no reason for zipped
collections to miss out on these benefits.

## Proposed solution

Introduce a new type, `Zip2Collection`:

```swift
public struct Zip2Collection<Collection1: Collection, Collection2> {
  public let left: Collection1
  public let right: Collection2
  
  internal init() // must use zip to construct
}

extension Zip2Collection: Collection { }

extension Zip2Collection: RandomAccessCollection
where Collection1: RandomAccessCollection, Collection2: RandomAccessCollection { }
```

Note that the new type.


Return this type from an overload of `zip` for collections:

```swift
public func zip<Collection1, Collection2>(
  _ collection1: Collection1, _ collection2: Collection2
) -> Zip2Collection<Collection1, Collection2> {
  return Zip2Collection(collection1, collection2)
}
```


## Detailed design

`Zip2Collection` will be a forward-traversal-only collection _unless_
both collections are random access. This is necessary

## Source compatibility

This is an additive change so source compatibility is minimal. The
`Zip2Collection` type will have all the same API as the `Zip2Sequence` type so
users will not be impacted by the type changing in their existing code.

The only circumstance in which existing code could be adversely affected
is where explicit typing is used, but where type context cannot force the
calling of the less-specific `zip`:

```swift
let a = zip(0..<5, 0..<5) // was Zip2Sequence, now Zip2Collection
let b = 
```

## Effect on ABI stability

None. This is an additive change.

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

