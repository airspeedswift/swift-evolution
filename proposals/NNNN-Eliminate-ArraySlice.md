# Eliminate ArraySlice and make Array.SubSequence = Slice<Array>

* Proposal: [SE-NNNN](NNNN-Eliminate-ArraySlice.md)
* Authors: [Ben Cohen](https://github.com/airspeedswift)
* Review Manager: TBD
* Status: **Awaiting review**
* Implementation: [apple/swift#NNNNN](https://github.com/apple/swift/pull/NNNNN)

## Introduction

This evolution proposal recommends eliminating the custom `ArraySlice` type, and
instead using `Slice`, with some conditional extensions and conformances, as
`Array`'s subsequence.

## Motivation

Most collection types in the standard library use the default `Slice` as their  
subsequence. There are two exceptions to this:

 - when a type can trivially serve as it's own slice, such as `Range`; and
 - when a type has special requirements that can't easily be implemented
   on top of `Slice`, such as `Substring`.

Until recently, `ArraySlice` fell into the second category. For example, it
needed to conform to `ExpressibleByArrayLiteral`. But with conditional 
conformance, it is now possible to conditionally conform `Slice` to 
`ExpressibleByArrayLiteral where Base: ExpressibleByArrayLiteral`.

This allows for the elimination of the concrete `ArraySlice` type, simplifying
`Array` both in terms of internals and its external interface. 

## Proposed solution

The following additions to the standard library will allow for the elimination 
of `ArraySlice`:

 - Introduce a new protocol, `ContiguouslyStored`, and a mutable refinement,
   `MutableContiguouslyStored`, with the `.withUnsafe` methods
   currently available on `Array` and `ArraySlice`.
 - Conform `Array` and `ContiguousArray` to `MutableContiguouslyStored`.
 - Conform `Slice: {Mutable}ContiguouslyStored where Base: {Mutable}ContiguouslyStored`.
 - Conform `Slice: ExpressibleByArrayLiteral where Base: ExpressibleByArrayLiteral`.
 - Conform `Slice: Equatable where Base: Equatable, Base: ContiguouslyStored`.
 - Create a `typealias ArraySlice<T> = Slice<[T]>`

The above will allow for significant reduction in standard library code for
implementing the `ArraySlice` type. The `ArraySlice` type alias will be 
deprecated in Swift 5 and eliminated in a future release.

## Detailed design

Note that all of the above could be done today and considered an 
implementation detail of `Array`, in order to realize the internal
benefits for the standard library. But `ContiguouslyStored` would need
to be an internal protocol used only by the standard library.

This proposal is to make `ContiguouslyStored` public, and to deprecate
the `ArraySlice` type alias, simplifying user code and the overall model.

```swift
public protocol ContiguouslyStored: Collection {
  func withUnsafeBufferPointer<R>(
    _ body: (UnsafeBufferPointer<Element>) throws -> R
  ) rethrows -> R
}

extension ContiguouslyStored {
  public func withUnsafeBytes<R>(
    _ body: (UnsafeRawBufferPointer) throws -> R
  ) rethrows -> R
}

public protocol MutableContiguouslyStored: Collection {
  mutating func withUnsafeMutableBufferPointer<R>(
    _ body: (inout UnsafeMutableBufferPointer<Element>) throws -> R
  ) rethrows -> R
}

extension MutableContiguouslyStored {
  public mutating func withUnsafeMutableBytes<R>(
    _ body: (UnsafeMutableRawBufferPointer) throws -> R
  ) rethrows -> R
}
```

Other types would benefit from conforming to `ContiguouslyStored`, such 
as `Unsafe{Mutable}BufferPointer` and `Data`, so should also have this
conformance added as well. This will help enforce consistency of these methods
accross types, and allow generic implementation of methods using unsafe
storage (such as performance optimizations like the backing-buffer-memcpying 
`append(contentsOf:)`).

`Equatable` conformance will be possible when a base is `ContiguouslyStored`.
Part of the definition of `ContiguouslyStored` is that its contents are
ordered. So `Slice` will be able to use element-wise comparison to implement 
`==`.

## Source compatibility

The `ArraySlice` type alias should provide all that is needed to make this 
fully source compatible. The only question is when to obsolete the type alias.
Since type aliases do not make up part of the ABI, they can be completely
eliminated in a later release.

## Effect on ABI stability

The plan for ABI stability for `Array` is for it to be 100% inlineable and fixed 
contents. As such, any change like this must be completed before ABI stability.

## Alternatives considered

Leaving `ArraySlice` as-is.

The introduction of `ContiguouslyStored` could also be combined with 
a rethink of the signatures for the `.withUnsafe` methods. This should
be done as part of a separate proposal as needed to avoid confusion.

`MutableContiguouslyStored` is preferred to `MutablyContiguouslyStored` 
for consistency with `MutableCollection`, and because
`MutablyContiguouslyStored` sounds a bit weird.
