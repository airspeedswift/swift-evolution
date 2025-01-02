# Explicit Specialization

* Proposal: [SE-NNNN](NNNN-specialize.md)
* Authors: [Ben Cohen](https://github.com/airspeedswift)
* Review Manager: TBD
* Implementation: Available in nightly toolchains using the underscored `@_specialize`
* Status: **Awaiting review**

## Introduction

The Swift compiler has the ability to "specialize" a generic function at compile time. This specialization creates a custom implementation of the function, where the generic placeholders are substituted with specific types. This can unlock optimizations of that specialized function that can be dramatically faster than the unspecialized version in some circumstances. The compiler can generate this specialized version and call it when it can see at the call site with what concrete types a function is being called, and the body of the function to specialize.

In some cases, though, this information is obscured from the compiler. This proposal introduces a new attribute, `@specialize`, which allows the author of a generic function to generate pre-specialized versions of that function for specific types. When the unspecialized version of the function is called with one of those types, the compiler will generate code that will re-dispatch to those prespecialized versions if available.

## Motivation

Consider the following generic function that sums an array of any binary integer type:

```swift
extension Sequence where Element: BinaryInteger {
  func sum() -> Double {
    reduce(0) { $0 + Double($1) }
  }
}
```

If you call this function directly on an array of a specific integer type (e.g. `Array<Int>`):

```
let arrayOfInt: [Int] = // ...
let result = arrayOfInt.sum()
```

in optimized builds, the Swift compiler will generate a specialized version of `sum` for that type. 

If you inspect the binary, you will see this specialized version under a symbol `_$sST3sumSz7ElementRpzrlEAASdyFSaySiG_Tg5`, which demangles to `generic specialization <[Swift.Int]> of (extension in example):Swift.Sequence< where A.Element: Swift.BinaryInteger>.sum() -> Swift.Double`, alongside the unspecialized version.

This specialized version of `sum` will be optimized specifically for `Array<Int>`. It can move a pointer directly over the the array's buffer, loading the elements into registers directly from memory. On modern hardware, it can make use of dedicated instructions to convert the integers to floating point. It can do this because it can see the implementation of `sum`, and can see the exact types on which it is being called. What exact assembly instructions are generated will differ significantly between e.g. `Array<Int8>.sum` and `Array<UInt64>.sum`.

Here is that `Array<Int>`-specialized code when compiled to x86-64 with `swiftc -Osize`:

```asm
_$sST3sumSz7ElementRpzrlEAASdyFSaySiG_Tg5:
        mov     rax, qword ptr [rdi + 16]
        xorpd   xmm0, xmm0
        test    rax, rax
        je      .LBB1_3
        xor     ecx, ecx
.LBB1_2:
        cvtsi2sd        xmm1, qword ptr [rdi + 8*rcx + 32]
        inc     rcx
        addsd   xmm0, xmm1
        cmp     rax, rcx
        jne     .LBB1_2
.LBB1_3:
        ret
```

Now consider some code that places an optimization barrier between the call site and the concrete type:

```swift
protocol Summable: Sequence where Element: BinaryInteger { }
extension Array: Summable where Element: BinaryInteger { }

var summable: any Summable

// later, when summable has been populated with an array of some kind of integer
let result = summable.sum()
```

The compiler now has no way of knowing what type `Summable` is at the call site – neither the sequence type, nor the element type. So its only option is to execute the fully unspecialized code. Instead of advancing a pointer over a buffer and loading values directly from memory, it must iterate the sequence by first calling `makeIterator()` and then calling `next()`, unwrapping optional values until it reaches `nil`.  And instead of using a single instruction to convert an `Int` to a `Double`, it must call the generic initializer on `Double` that uses methods on the `BinaryInteger` protocol to convert the value into floating point representation. Unsurprisingly, this code path will be about 2 orders of magnitude slower than the specialized version. This is true even if the actual type being summed ends up being `[Int]` at runtime.

A similar situation would occur when `sum` is a generic function in a binary framework which has not provided an `@inlinable` implementation. While the Swift compiler might know the types at the call site, it has no ability to generate a specialized version because it cannot see the implementation in order to generate it. So it must call the unspecialized version of `sum`.

The best way to avoid this situation is just to avoid type erasure in the first place. This is why concrete types and generics should be preferred whenever practical over existential types for performance-sensitive code. But sometimes type erasure, particularly for heterogenous storage, is necessary.

Similarly, ABI-stable binary framework authors do not always want to expose their implementations to the caller, as this tends to generate a very large and complex ABI surface that cannot be easily changed later. It also prevents upgrades of binary implementations without recompiling the caller, allowing for e.g. an operating system to fix bugs or security vulnerabilities without an app needing to be recompiled.

## Proposed solution

A new attribute, `@specialize`, will allow the author of a function to cause the compiler to generate specializations of that function. In the body of the unspecialized version, the types are first checked to see if they are of one of the specialized types. If they are, the specialized version will be called.

So in our example above:

```swift
extension Sequence where Element: BinaryInteger {
  @specialize(where Self == [Int])
  func sum() -> Double {
    reduce(0) { $0 + Double($1) }
  }
}
``` 

A specialized version of `[Int].sum` will be generated in the same way as if it had been specialized for a callsite. And inside the unspecialized generic code, the additional check-and-redispatch logic will be inserted at the start of the function.

Doing this restores the performance of the specialized version of `sum` (less a check and branch of the calling type) even when using the existential `any Summable` type.

## Detailed design

The `@specialize` attribute can be placed on any generic function. It takes an argument with the same syntax as a `where` clause for a generic signature. 

The `where` clause must fully specialize all the generic placeholders of the types in the function signature. In the case of a protocol extension, as seen above, that includes specifying `Self`.

As well as protocol extensions, it can also be used on extensions of generic types, on computed properties (note, it must be put on `get` explicitly, not on the shorthand where it is elided), and on free functions:

```swift
extension Array where Element: BinaryInteger {
  @specialize(where Element == Int)
  func sum() -> Double {
    reduce(0) { $0 + Double($1) }
  }
  
  var product: Double {
    @specialize(where Element == Int8)
    get { reduce(1) { $0 * Double($1) } }
  }
}

@specialize(where T == Int)
func sum<T: BinaryInteger>(_ numbers: T...) -> Double {
  numbers.reduce(0) { $0 + Double($1) }
}
```

All placeholders must be fully specified in the `where` clause even if they do not appear to be used in the function body:

```swift
extension Dictionary where Value: BinaryInteger {
  // error: Too few generic parameters are specified in 'specialize' attribute (got 1, but expected 2)
  // note: Missing equality constraint for 'Key' in 'specialize' attribute
  @specialize(where Value == Int)
  func sum() -> Double {
    values.reduce(0) { $0 + Double($1) }
  }
}
```

Bear in mind that even when not explicitly used in the body, they may be used implicitly. For example, in calculating the location of stored properties. Depending on how `Dictionary` is laid out, it may be important to know the size of the `Key` even if key values aren't used. But even if there is absolutely no use of a generic type in the body being specialized, the type must be explicitly specified. This requirement could be loosened in future, see "Partial Specialization" in future directions.

Where multiple placeholders need to be specified separately, they can be separated by commas, such as in the fix for the above example:

```swift
extension Dictionary where Value: BinaryInteger {
  @specialize(where Value == Int, Key == Int)
  func sum() -> Double {
    values.reduce(0) { $0 + Double($1) }
  }
}
```

## Source compatibility

The addition or removal of explicit specializations has no impact on the caller, and is opt-in. As such it has no source compatability implications.

## ABI compatibility

This proposal covers only _internal_ specializations, dispatched to from within the implementation of an unspecialized generic function. As such it has no impact on ABI. It can be applied to existing ABI-stable functions in libraries built for distribution, and can be removed later without ABI impact. Generic functions can continue to be inlinable to be specialized by the caller, and the code to dispatch to specialized versions will not appear in the inlinable code emitted into the swift interface file.

A future direction where explicit specializations are exposed as ABI appears in Future Directions.

## Implications on adoption

Explicit specializations impose no new runtime requirements on either the caller or the called function, so use of them can be back-deployed to earlier Swift runtimes.

## Future directions

### Partial Specialization

The need to fully specify every type when using `@specialize` is somewhat limiting. For example, in the `Dictionary` example above, no reference is made in the code to the `Key` type in summing the values. Having to explicitly state the concrete type for `Key` means you need separate specializations for `[String:Int]`, `[Int:Int]` and so on. 

Even in cases where dynamic dispatch through the protocol is still required for the unspecialized aspects (such as determining where to find the values in a dictionary's layout), this might not be in the hot part of the function and so partial specialization might be the better trade-off.

Closely related to partial specialization is the ability to specialize once for all particular type layouts. In the `Dictionary.sum` example, it might only matter what the size of the key type is in order to efficiently iterate the values.

Another example of this is `Array.append`. There is only one single implementation needed for appending an `AnyObject`-constrained to an array. The append operation needs to retain the object, but does not need any other properties. So two class instances could be appended using the same method irrespective of their actual type. Similar techniques could be used to provide shared specializations for `BitwiseCopyable` types, possibly with the addition of a supplied size argument to avoid having to use value witness calls.

### Making Specializations Publicly Available

This proposal keeps explicit specializations internal only. For ABI-stable frameworks, a useful future direction would be to make these symbols publicly available and listed in the `.swiftinterface` file, for callers to link to directly. 

Doing this allows ABI-stable framework authors to expose specializations without exposing the full implementation details of a function as inlinable code. While adding a public specialized symbol to a framework is ABI, it is a much more limited ABI surface compared to providing an inlinable implementation, which requires any future changes to a type to consider the previous inlinable code's behavior to ensure it remains compatible forever in older binaries. A specialized entry point could be updated in a framework to fix a bug without recompiling the caller. Specializations in binary frameworks also have the benefit of avoiding duplication of code into the caller.

### Requiring Specialization

A related feature is the ability to _require_ specialization either at the call site, or on a protocol. Specialization does not always have to happen at the call site even when it could – it remains an optimization (albeit a very aggressively applied one currently). If the specializatino does not happen, it would be useful to force the compiler to override its heuristics, similar to forcing linlining of a long but critical function.

There are also protocols that are only meant to be used in specialized form in optimized builds. Arguably `BinaryInteger` is one of them. It may be worth exploring in these cases an annotation to indicate this to the caller, either via a compile-time errror/warning, or a runtime error.

### Tooling Directions

This proposal only outlines language syntax the developer can use to instruct the compiler to emit a specialized version of a function. Complimentary to this is development of tooling to identify profitable specializations to add, for example by profiling typical usage of an app. It should be noted that specialization is only required in highly performance sensitive code. In many cases, explicit specialization will have little impact on overall performance while increasing binary size. 

The current implementation requires the attribute to be attached directly to the function being specialized. Tooling to produce specializations would benefit from an additional syntax that could be added in a separate file, or even into a separately compiled binary.

## Alternatives considered

Many alternatives to this proposal – such as whole-program analysis to determine prespecializations automatically – are complimentary to this technique. Even in the presence of better optimizer heroics, there are still benefits to having explicit control over specialization.

This proposal takes as a given Swift's current dispatch mechanisms. Some alternatives to this proposal tend to end up requiring fundamental changes to Swift's core generics implementation, and are therefore out of scope for this proposal.