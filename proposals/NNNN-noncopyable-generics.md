# Noncopyable Generics

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Kavon Farvardin](https://github.com/kavon)
<!-- * Upcoming Feature Flag: `NoncopyableGenerics` -->
<!-- * Review Manager: TBD
* Status: **Awaiting implementation** or **Awaiting review**
* Vision: *if applicable* [Vision Name](https://github.com/apple/swift-evolution/visions/NNNNN.md)
* Roadmap: *if applicable* [Roadmap Name](https://forums.swift.org/...))
* Bug: *if applicable* [apple/swift#NNNNN](https://github.com/apple/swift/issues/NNNNN)
* Implementation: [apple/swift#NNNNN](https://github.com/apple/swift/pull/NNNNN) or [apple/swift-evolution-staging#NNNNN](https://github.com/apple/swift-evolution-staging/pull/NNNNN)
* Previous Proposal: *if applicable* [SE-XXXX](XXXX-filename.md)
* Previous Revision: *if applicable* [1](https://github.com/apple/swift-evolution/blob/...commit-ID.../proposals/NNNN-filename.md)
* Review: ([pitch](https://forums.swift.org/...)) -->

## Table of Contents
- [Noncopyable Generics](#noncopyable-generics)
  - [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
  - [Motivation](#motivation)
  - [Proposed Solution](#proposed-solution)
    - [Copying and `Copyable`](#copying-and-copyable)
    - [General type parameters](#general-type-parameters)
    - [Structs and enums](#structs-and-enums)
      - [Automatic synthesis of conditional `Copyable` conformance](#automatic-synthesis-of-conditional-copyable-conformance)
    - [Classes](#classes)
    - [Protocols](#protocols)
      - [Inheritance](#inheritance)
    - [Source Compatibility](#source-compatibility)
    - [Custom Defaults](#custom-defaults)
      - [Defaults for constrained generics](#defaults-for-constrained-generics)
  - [Detailed Design](#detailed-design)
    - [The top type](#the-top-type)
    - [`AnyObject`](#anyobject)
    - [Tuples](#tuples)
    - [Packs](#packs)
  - [Case Studies](#case-studies)
    - [Swift Standard Library](#swift-standard-library)
  - [Alternatives Considered](#alternatives-considered)
    - [Do not introduce a new type syntax `T ~ Copyable`](#do-not-introduce-a-new-type-syntax-t--copyable)
    - [Avoid having to write `~Copyable` on constrained protocols](#avoid-having-to-write-copyable-on-constrained-protocols)
    - [Avoid needing to specify the copyability of protocols entirely](#avoid-needing-to-specify-the-copyability-of-protocols-entirely)
  - [Future Directions](#future-directions)
    - [Infinite Defaults](#infinite-defaults)
      - [Bottomless Constraint Subjects](#bottomless-constraint-subjects)
  - [Acknowledgments](#acknowledgments)


## Introduction

The noncopyable types introduced in [SE-0390: Noncopyable structs and
enums](0390-noncopyable-structs-and-enums.md) come with the heavy limitation
that such values not be substituted for a generic type parameter, erased to an
existential, or even conform to a protocol. This proposal lifts those
restrictions by extending Swift's type system with a foundational syntax and
semantics for noncopyable generics.

## Motivation

Suppose an implementation of a noncopyable file descriptor type would like to
conform to `Comparable` so that it can be ordered using the infix `<` operator:

```swift
struct FileDescriptor: ~Copyable, Comparable {
  let fd: Int
  
  static func == (lhs: borrowing FileDescriptor, 
                  rhs: borrowing FileDescriptor) -> Bool {
    return lhs.fd == rhs.fd
  }

  static func < (lhs: borrowing FileDescriptor, 
                 rhs: borrowing FileDescriptor) -> Bool {
    return lhs.fd < rhs.fd
  }
}
```

While this type `FileDescriptor` has the correct witnesses, the conformance is
not valid. A type parameter constrained by a `Comparable` value is copyable, so
an invalid copy of a `FileDescriptor` could be permitted:

```swift
extension Comparable {
  func copy() -> Self { return self }  // returns a copy of self
}

public func max<T: Comparable>(_ x: borrowing T, _ y: borrowing T) -> T {
  return y >= x ? copy y : copy x
}

let fd1 = FileDescriptor(fd: 0)
let fd2 = fd1.copy()  // 'fd2' is a copy of 'fd1'!
let bigger = max(fd1, fd2)  // 'bigger' is copy of 'fd2'!
```

This baseline assumption of copying in the generics system, without a way to
opt-out, is the fundamental reason noncopyable types in
[SE-0390](0390-noncopyable-structs-and-enums.md) came with limitations around
generics. Lifting those limitations is the goal of this proposal.

## Proposed Solution

Throughout this proposal, fragments of Swift code appear as comments within
examples to make _all_ proposed implicit generic requirements of a type visible.
If there is no comment for a generic type, then there truly are no additional
requirements than what is already written.

The primary way these implicit requirements are written is within `/* ... */` 
comments, for example, 
```swift
func f<T>(_ t: T) /* where T: Constraint */ {}
```
means that the `Constraint` on `T` is implicit and is not actually written in
the source code for that example. In addition, some comments containing a
_requirement signature_ or _interface type_ may appear next to a type or
function, respectively. These generic type signatures fully spell-out all of the
requirements or type constraints for generic parameters in scope:
```swift
// signature <Self where Self: Comparable>
protocol P: Comparable {}

struct S<T: Equatable> {
  func f() {}  // signature <T where T: Equatable>
}
```
These comments are used for clarity, especially when it is not possible to
otherwise convey the requirements on a type using Swift syntax.

### Copying and `Copyable`
The type `Copyable` is a layout protocol, a kind of marker protocol, that is
understood by Swift to represent types that support copying. Naturally, the set
of noncopyable types are exactly those that do not conform to `Copyable`. See
[SE-0390](0390-noncopyable-structs-and-enums.md) for more details about copying
and working with noncopyable values.

Being a marker protocol, `Copyable` has no explicit requirements. Value types
like structs and enums conform to `Copyable` only if all of its stored properties
or associated values conform to `Copyable`. Reference types like classes and
actors can always conform to `Copyable`, because a reference to an object can
be copied regardless of what is contained in the object.

<!-- What sets apart a layout protocol is that at runtime, it is possible to query
whether a type conforms (for more details, see a later section). -->

<!-- TODO: make sure this is exhaustive about what other non-nominal types are
copyable like metatypes, tuples, element packs -->

### General type parameters

All generic type parameters default to carrying a `Copyable` constraint:

```swift
struct FileDescriptor: ~Copyable { /* ... */ }

// signature <T where T: Copyable>
func genericFn<T>(_ t: T) /* where T: Copyable */ {
  return copy t  // OK
}

genericFn(FileDescriptor())  // ERROR: FileDescriptor is not Copyable
genericFn([1, 2]) // OK: Array<Int> is Copyable
```

In [SE-0390](0390-noncopyable-structs-and-enums.md) the prefix `~` syntax was
introduced only in the inheritance clause of a nominal type to "suppress" its
Copyable conformance. This proposal completes the picture by defining the
semantics of `~Copyable` as the inverse of a constraint that cancels out the
default requirement or conformance to `Copyable`. The definition of inverses for
types other than `Copyable` is outside the scope of this proposal.

A generic parameter can have its implicit `Copyable` constraint removed by
applying its inverse `~Copyable` to the subject as a constraint:

```swift
// signature <T>
func identity<T: ~Copyable>(_ t: borrowing T) -> T  {
  return copy t  // ERROR: 't' is noncopyable
}

identity(FileDescriptor())  // OK, FileDescriptor is not Copyable
identity([1, 2, 3])  // OK, even though Array<Int> is Copyable
```

> **Key Idea:** Applying an inverse `T: ~Copyable` does _not_ prevent a
> `Copyable` type from being substituted for `T`, i.e., it does _not_ narrow the
> kinds of types permitted, like a constraint would. This is the reason why
> `~Copyable` is referred to as the _inverse_ of `Copyable` rather than a
> constraint itself. When applying an inverse `~C` to a type variable implicitly
> constrained to `C`, that implicit constraint is nullified. Removing any kind
> of constraint from a generic parameter _broadens_ the kinds of types
> permitted.

As with a concrete noncopyable type, any generic type parameter that does not
conform `Copyable` must use one of the ownership modifiers `borrowing`,
`consuming`, or `inout`, when it appears as the type of a function's parameter.
For details on these parameter ownership modifiers, see
[SE-377](0377-parameter-ownership-modifiers.md).

### Structs and enums
Struct and enum types default to implicitly conforming to `Copyable`:

```swift
struct DataSet /* : Copyable */ {
  var samples: [Double]
}
```

As with any other generic parameter, there is a `Copyable` constraint on the
`Elm` in this `List` too:

```swift
enum List<Elm> /* : Copyable where Elm: Copyable */ {
  case empty
  indirect case elm(Elm, List<Elm>)
}
```

As in SE-390, a generic struct or enum can use the inverse `~Copyable` to
remove the implicit `Copyable` conformance and constraint:

```swift
enum List<Elm: ~Copyable>: ~Copyable { /* ... */ }
```

Once `Elm` does not have a `Copyable` constraint, a noncopyable type like
`FileDescriptor` can be substituted in `List`, in addition to copyable ones.

#### Automatic synthesis of conditional `Copyable` conformance
A common reason for having a type parameter is that values of that generic type
will be stored somewhere within the parameterized struct or enum type. Because
of `Copyable`'s containment rule, that implies the parameterized type itself
is commonly noncopyable if any of its type parameters is noncopyable. This
observation is one of the guiding principles behind the rules in this section.

Using the inverse `~Copyable` on any type parameter of a struct or enum, without
specifying the copying behavior for the parameterized type, will automatically
synthesize a conditional conformance to `Copyable`:

```swift
// A conditionally-copyable pair.
struct Pair<Elm: ~Copyable> {
  // signature <Elm>
  func exchangeFirst(_ new: consuming Elm) -> Elm? { /* ... */ }
}
/* extension Pair: Copyable where Elm: Copyable {} */

// Pair<FileDescriptor> is NOT Copyable
// Pair<Int> is Copyable

// A conditionally-copyable factory.
struct Factory<Product: ~Copyable, Token: ~Copyable, Policy> {
  var seed = 0

  // signature <Product, Token, Policy where Policy : Copyable>
  mutating func produce(_ payment: consuming Token) -> Product { /* ... */ }
}
/* extension Factory: Copyable where Product: Copyable, Token: Copyable {} */
```

Notice that despite this `Factory` not storing any noncopyable data, it does not
have an implicit unconditional `Copyable` conformance because one of its type
parameters has removed its default `Copyable` constraint. Instead, a
conditional conformance was synthesized to make `Factory` copyable once those
type parameters are `Copyable`.

The synthesis procedure generates the `where` clause of the conditional
`Copyable` conformance by enumerating _all_ of the type parameters and requiring
each one to be `Copyable`. It does not matter whether the parameter is used in
the storage of the `Factory` struct.

One case where this synthesis procedure produces an invalid conformance is
when the parameterized type contains some noncopyable type other than one of its
parameters. In such a case, the `~Copyable` becomes required on the
parameterized type: 

```swift
struct TaggedFile<Tag: ~Copyable> {
  //                             ^
  //                             note: add ': ~Copyable' here
  let tag: Tag
  let fd: FileDescriptor // error: 'FileDescriptor' is noncopyable, 
                         //         so 'TaggedFile' is always noncopyable.
}
```

Control over the conformance synthesis behavior can be achieved in a few ways.
First, explicitly write a conformance to `Copyable` to make the type
unconditionally copyable, while keeping the type parameter itself noncopyable:

```swift
struct AlwaysCopyableFactory<Product: ~Copyable, Token: ~Copyable>: Copyable {
  // signature <Product>
  func produce(_ payment: consuming Token) -> Product { /* ... */ }
}
```

Next, to disable the conditional conformance synthesis _and_ remove the implicit
`Copyable`, use the inverse `~Copyable` on the type instead:

```swift
struct ExplicitFactory<Product: ~Copyable, Token: ~Copyable>: ~Copyable {
  // signature <Product>
  func produce(_ payment: consuming Token) -> Product { /* ... */ }
}
```

After applying `~Copyable` to `ExplicitFactory` itself, it is possible to 
specify an explicit conditional conformance to `Copyable` if desired:

```swift
extension ExplicitFactory: Copyable where Product: Copyable {}
```

This design is meant to strike a balance between brevity in the most common case
(implicit conditional `Copyable` conformance) and expressivity (full control
when explicit).

### Classes

Classes (including actors) and all of their generic parameters default to being
`Copyable`:

```swift
class FileHandle<File> /* : Copyable where File: Copyable */ {
  var file: File
  // ...
}
```

To allow a class to contain a generic noncopyable value, use the inverse
`~Copyable` on the type parameter:

```swift
class FileHandle<File: ~Copyable> /* : Copyable */ {
  var file: File
  // ...
}
```

Classes can contain noncopyable storage without itself becoming noncopyable,
i.e., the containment rule does not apply to classes. As a result, a class does
_not_ become conditionally copyable when one of its type parameters has the
inverse `~Copyable`. Support for noncopyable classes is left to future work.

### Protocols

Protocols and their associated types default to carrying an implicit `Copyable`
conformance requirement:

```swift
// signature <Self where Self: Copyable, Self.Event: Copyable>
protocol Foo /* : Copyable */ {
  associatedtype T /* : Copyable */

  borrowing func bar() -> Self
  func buzz(_: T) -> T
  func blarg() -> Something<Self>
}
```

Protocols can remove the default `Copyable` requirement from `Self` using
`~Copyable`:

```swift
// signature <Self where Self.Event: Copyable>
protocol EventLog: ~Copyable {
  associatedtype Event /* : Copyable */
  
  mutating func push(_ event: consuming Event)
  mutating func pop() throws -> Event
}
```

Within `EventLog`, the type `Self` has no conformance requirements at all, but
the associated type `Self.Event` is copyable. The removal of the `Copyable`
conformance requirement on `EventLog` allows copyable and noncopyable types to
conform:

```swift
// signature <Self where Self: EventLog, Self: Copyable>
struct ArrayLog<Elm>: EventLog /*, Copyable where Elm: Copyable */ {
  typealias Event = Elm
  var log: [Elm]
  // ...
}

// signature <Self where Self: EventLog>
struct UniqueLog<Elm>: EventLog, ~Copyable /* where Elm: Copyable */ {
  typealias Event = Elm
  var log: [Elm]
  // ...
}
```

Associated types can additionally use the inverse `~Copyable` to remove their
default `Copyable` requirement, meaning a noncopyable type can witness the
requirement:

```swift
protocol JobQueue<Job> /* : Copyable */ {
  associatedtype Job: ~Copyable

  func submit(_ job: consuming Job)
}
```

In an unconditional extension of `EventLog`, the type `Self` is not `Copyable`
because `~Copyable` was used to remove its implicit `Copyable` requirement:

```swift
protocol EventLog: ~Copyable {
  associatedtype Event /* : Copyable */
  // ...
}

extension EventLog {
  func duplicate() -> Self { 
    return copy self // error: copy of noncopyable value
  } 
}
```

Of course, when the conformer _does_ happen to be copyable, additional 
functionality can be made available to it using a conditional extension:

```swift
extension EventLog where Self: Copyable {
  func duplicate() -> Self { 
    return copy self // OK
  } 
}
```

The same principle applies to `JobQueue`'s associated type `Job` in extensions,
where the `Job` is not `Copyable`. 

For a noncopyable protocol with no extension default, its existentials are also
noncopyable:

```swift
protocol Pizza: ~Copyable {
  associatedtype Topping: ~Copyable
  func peelOneTopping() -> Topping
}

let t: any Token = ... // signature <Self where Self: Token>
let _: any ~Copyable = t.peelOneTopping() // signature <Self>
```
For associated types within a protocol erased to an existential preserve
conformances to a layout protocol like `Copyable`. Thus, when calling
`peelOneTopping` on an `any Token`, an `any ~Copyable` value is returned instead
of `Any`.

#### Inheritance

When inheriting a protocol that removed it's implicit Copyable constraint via
`~Copyable`, that removal does _not_ carry-over to inheritors. That means a type
must restate `~Copyable` even if it inherits only from protocols using
`~Copyable`, because the absence of a requirement is not propagated:

```swift
protocol Token: ~Copyable {}

// signature <Self where Self : Token, Self : Copyable>
protocol ArcadeCoin: Token /* , Copyable */ {}

// signature <Self where Self : Token>
protocol CasinoChip: Token, ~Copyable {}
```

A key takeaway from the example above is that `~Copyable` as a constraint is 
not viral: a protocol that has no `Copyable` constraint does _not_ mean its 
conformers cannot have the `Copyable` capability. As a corollary, a protocol 
that inherits from a noncopyable one can still be `Copyable`.

Associated type requirements that are inherited from another protocol are 
taken as-is:

```swift
// signature <Self: Copyable>
protocol JobQueue /* : Copyable */ {
  associatedtype Job: ~Copyable
  // ...
}

// signature <Self where Self: Copyable, Self: JobQueue>
protocol FIFOJobQueue<Job>: JobQueue {
  func pushBack(_ j: Job) // error: missing ownership specifier for parameter of
                          //        noncopyable type 'Job'
}
```

In the above example, `JobQueue.Job` remains noncopyable in `FIFOJobQueue`.
The exception is if an associated type requirement with the same name is 
redeclared in the inheritor. In that case, the usual rule for `associatedtype`
requirements will add an implicit `Copyable` requirement for the redeclaration:

```swift
// signature <Self where Self: JobQueue, Self.Job: Copyable>
protocol LIFOJobQueue<Event>: JobQueue {
  associatedtype Job /* : Copyable */
}
```

### Source Compatibility

When upgrading pre-existing generic types to support noncopyable types, source
compatibility issues can arise because a fundamental layout capability of the
type is being _removed_. For example, extensions of types appearing in clients
of an API may have assumed that a type does support the copying capability, when
its requirement was removed from a type via `~Copyable`:

```swift
// The `~Copyable` was newly added to `Maybe`...
public enum Maybe<Value: ~Copyable> {
  case just(Value)
  case nothing
}
/* extension Maybe: Copyable where Value: Copyable {} */
```

Any existing extensions of `Maybe` within clients no longer can copy `Value`:

```swift
extension Maybe {
  func asArray(repeating n: Int) -> [Value] {
    // ...
    elms.append(copy v) // error: copy of noncopyable type `Value`
    // ...
  }
}
```

All existing extensions of `Maybe` in clients must effectively become qualified
with `where Value: Copyable` by default to avoid the source break. A similar
situation arises with associated types:

```swift
// Assume the '~Copyable' was newly added.
protocol VendingMachine: ~Copyable {
  associatedtype Snack: ~Copyable

  func obtainSnack() -> Snack
}

// Source break via associatedtype change
func buyDinner<V: VendingMachine>(_ machine: V) /* where V: Copyable */ {
  let snak = machine.obtainSnack()
  let _ = [snak, snak] // error: copy of noncopyable value 'V.Snack'
}
```

This function `buyDinner` defined in a client is effectively a "extension" of a
library type `VendingMachine`. Values of type `V` can still be copied in
`buyDinner`, since the generic parameter defaults to `Copyable`. But `V.Snack`
does not support copying because it came from `VendingMachine` without any
defaulted `Copyable` capability.

### Custom Defaults

An _extension default_ allows the author of a type to specify a default `where`
clause to be used implicitly whenever writing an `extension` or constraining
a generic type parameter to that type. The syntax for an extension default is
```
default extension <where-clause>
```
which declares new kind of anonymous member that can appear at most once within
a type declaration. Its `where` clause can only state requirements that:

1. are valid to write on an extension of that enclosing type;
2. contain conformance requirements `T: P` where `P` is a type that has an
inverse `~P`.

A type with an extension-default member has the `where` clause appear implicitly
on _all_ extensions of that type and for each generic parameter constrained to
the type. For example, consider a non-protocol type:

```swift
public enum Perhaps<Value: ~Copyable> {
  case thing(Value)
  case empty

  // `Value` is not copyable for members here.

  default extension where Value: Copyable
}
/* extension Perhaps: Copyable where Value: Copyable {} */

extension Perhaps /* where Value: Copyable */ {
  func asArray(repeating n: Int) -> [Value] {
    guard case let .thing(v) = self else { return [] }
    // ...
    elms.append(copy v) // OK
    // ...
    _ = copy self // OK
    // ...
  }
}

let p1: Perhaps<FileDescriptor>  // noncopyable and does not support `asArray`
```

Using the extension default, a plain extension of `Perhaps` now always assumes
`Value` is copyable, even for extensions that add conformances:

```swift
extension Perhaps: Questionable /* where Value: Copyable */ {
  // ...
}
```

To nullify the extension default on a non-protocol type, the same inverse
`~Copyable` can be applied to `Self` in an extension:

```swift
extension Perhaps: Comparable where Self: ~Copyable {
  // ... an extension for all 'Perhaps' values ...
}

func f<T: Comparable & ~Copyable>(_ t: t) { /* ... */ }

f(Perhaps<Int>.empty) // OK
f(Perhaps<FileDescriptor>.empty>) // OK
```

While `Self` in an extension of a non-protocol type is not generic, the
constraint `Self: ~Copyable` serves as syntactic sugar to nullify the default
`Copyable` conformance requirements on `Perhaps`.

#### Defaults for constrained generics

For a protocol type `P`, its extension default will add its where clause to

1. all extensions of `P`
2. protocols directly inheriting from `P`
3. generic parameters constrained to `P`
4. existentials erased to `P`

For example, a type parameter constrained by two protocols with extension
defaults will have both where-clauses added to it by default:

```swift
protocol P: ~Copyable {
  associatedtype A: ~Copyable
  func getA() -> A
  default extension where Self.A: Copyable
}
protocol Q: ~Copyable {
  associatedtype B: ~Copyable
  default extension where Self.B: Copyable
}

func f<T>(_ t: T)
  where T: P, 
        /* , T.A: Copyable */     // via custom default of P
        T: Q, 
        /* , T.B: Copyable */     // via custom default of Q
        /* , T: Copyable */ {}    // via standard default for type parameter T
```

The single application of an inverse `~L` to any subject type can be thought of
as a filter removing each _default_ conformance requirement matching `L` from
the subject. For example, applying `T: ~Copyable` removes all defaults from `T`,
including those from `P` and `Q`:

```swift
func g<T>(_ t: T)
  where T: P, T: Q, T: ~Copyable {}
```

The same concept applies to an existential `any E`, which is sugar for a type
signature `<Self where Self: E>`:

```swift
// signature <Self where Self.A: Copyable>
let z1: any P

// signature <Self where Self: P & Q>
let x: any P & Q & ~Copyable = // ...

// signature <Self where Self: P & Q, Self.A: Copyable, Self.B: Copyable>
let y: any P & Q = // ...
```

To remove the extension defaults for `Copyable` from a specific type like `P`, 
apply the inverse directly to the type by writing `P ~ Copyable`:

```swift
func g<T>(_ t: T)
  where T: P ~ Copyable, 
        T: Q, 
        /* , T.B: Copyable */     // via custom default of Q
        /* , T: Copyable */ {}    // via standard default for type parameter T
```

Protocol inheritance is also similar, as it is sugar for constraining the
protocol's `Self` to some other protocol:

```swift
protocol R: P, Q /* where Self.A: Copyable, Self.B: Copyable, Self: Copyable */ {}

protocol S: P ~ Copyable, Q /* where Self.B: Copyable, Self: Copyable */ {}

protocol U: P, Q, ~Copyable{}
```

The extension default defined in type `P` is not _inherited_ by `R`. The default
is _expanded_ as additional requirements in `R`. This means that an inverse 
applied to a type constrained by `R` will not remove any of the `Copyable`
requirements:

```swift
func h<T>(_ t: T)
  where T: R, T: ~Copyable {}
  // error: 'T' is required to conform to 'Copyable' by protocol 'R'
```

Same-type constraints only propagate positive requirements, not the
nullification of defaults via inverses:

```swift
func i<T, U>(_ t: T, _ u: U)
  where T: P, T == U, U: ~Copyable // error: 'U' is required to conform to 'Copyable'
  /* T: Copyable, T.A : Copyable */ {}
```

## Detailed Design

This section spells out additional details about the proposed extensions.

### The top type

The type `Any` is no longer the "top" type in the language, which is the type
that is a supertype of all types. The new world order is:

```
              any ~Copyable
                /        \
               /          \
              /            \
    Any == any Copyable   <all noncopyable types>
        |
< all other copyable types >
```

In other words, new top type is `any ~Copyable`.

### `AnyObject`

TODO: can classes be cast to `any ~Copyable`? If so, then it seems fine to
permit an `AnyObject` to be cast to `any ~Copyable`. 

TODO: is this legal? `func f<T>(_ t: T) where T: AnyObject, T: ~Copyable {}`

### Tuples

### Packs

## Case Studies

This section provides code examples of using the proposed design to retrofit
existing APIs. The code examples contained are _not_ to be interpreted as
proposed changes for the Swift standard library.

### Swift Standard Library

TODO: convert parts of the stdlib and popular 3rd party APIs
- Equatable, Hashable, Comparable
- Optional, SetAlgebra
- ... more ...

## Alternatives Considered

A number of solution designs were considered but ultimately not chosen for
inclusion in this proposal. For posterity, they are described in this section.

### Do not introduce a new type syntax `T ~ Copyable`

The proposed syntax for removing the custom `Copyable` defaults coming from a
_specific_ type `T` is to use the `~` in an infix position, `T ~ Copyable`:

```swift
func f<V>(_ v: V)
  where V: T ~ Copyable V: R {}
```
This `where` clause can be equivalently re-written as
```
where V: T ~ Copyable & R
```
The syntax `T & ~Copyable` was considered for this purpose, but its meaning is
reserved for removing all of the `Copyable` defaults from an entire composition. 

The reason for this is that `&` is associative. This means `(P & Q) & R` is
equivalent to `P & (Q & R)`. Thus, `(T & ~Copyable) & R` does not remove the
defaults _only_ from `T`, but from `R` as well! Because that composition is the
same as `T & (~Copyable & R)`.

### Avoid having to write `~Copyable` on constrained protocols

We could have a rule that says the implicit `Copyable` conformance on `Self` for
a protocol `P` is automatically removed if `Self` has a conformance requirement
to some other protocol `Q`. Thus we would have:

```swift
protocol Q: ~Copyable {}

protocol P: Q {
  // Self here is non-copyable.
}
```

The main problem with this is that the `~Copyable` effectively becomes viral,
in the sense that once you remove `Copyable` from a protocol, any downstream
protocols inheriting from it will also lose its implicit `Copyable` constraint.
That behavior makes it difficult to modify any upstream protocols to allow
noncopyable conformers without unintended source breaks.

### Avoid needing to specify the copyability of protocols entirely

In theory, protocol declarations don't need to impose about the copyability of
the witness, since its requirements do not have default implementations.
Copyability of the type would then become an implementation detail of the
conformer. Only extensions of protocols would need to specify whether they apply
to a `Self : Copyable`. The problem with this idea is that a requirement can be
written in such a way that whether `Self` conforms to `Copyable` must be known:


```swift
struct Box<Thing: ~Copyable>: ~Copyable {
  let thing: Thing
}
extension Box: Copyable where Thing: Copyable {}

protocol P {
  func f(box: Box<Self>)
}
```

Here, `Self` is substituted for a type parameter within one of its own method
requirements. That method has a parameter that is conditionally copyable based
on whether `Self` is copyable. Whether the requirement's parameter needs to
specify an ownership annotation now depends on `Self`. 

## Future Directions

TODO: Discuss ~Escapable

### Infinite Defaults

TODO: clean-up this section

It is sometimes useful to have an infinite nesting of associated type 
requirements:

```swift
protocol View: ~Copyable {
  associatedtype Body: View, ~Copyable
}
```

This type `View` has an infinite nesting of associated types named `Body`.
Writing an extension default with a `where` clause that says `Self` and all
`Body` types reachable from it are `Copyable` is not possible:

```swift
default extension where Self: Copyable, 
                        Self.Body: Copyable,
                        Self.Body.Body: Copyable
                        Self.Body.Body.Body: Copyable
                        // ... repeat infinitely ...
```

If an extension default could express the requirement signature:
```
<Self where Self: View, Self: Copyable, Self.Body == Self> 
```
then the `Copyable` constraint would propagate down the infinite nesting. But,
the same-type requirement `Self.Body == Self` also causes the nesting of types
to all be the same, when they may not be:

```swift
struct Viewer: View {
  typealias Body = Frame  // Viewer != Viewer.Body

  struct Frame: View {
    typealias Body = Viewer
  }
}
```

More generally, any same-type requirement that can propagate a conformance
requirement through an infinite nesting of associated types will also cause some
type in that nesting to be equal to another.

#### Bottomless Constraint Subjects

A conformance constraint with a _bottomless subject_ `T.*` is written as 
`T.* : L`, where `T` names some generic type and `L` is a layout protocol.
Because of the bottomless subject, this constraint requires that
  1. `T` conforms to `L`
  2. If there exists a generic type `V` reachable from `T` via member lookup,
  then `V` conforms to `L`.

  (T.Body.Subsequence).*

  Self.Subsequence*
  Self.Subsequence.Indices*

The use of an asterisk "*" in the subject is connected to the [Kleene
star](https://en.wikipedia.org/wiki/Kleene_star) and regular expressions. In the
subject of a constraint, `T.*` enumerates the possibly infinite set of types "T
and any types whose name is prefixed by T".

With a bottomless conformance, it is possible to provide an extension default
where all associated types `Body` of a `View` are `Copyable`:

```swift
protocol View: ~Copyable {
  associatedtype Body: View, ~Copyable

  default extension where Self.*: Copyable
}

extension View /* where Self.*: Copyable */ {}

extension View where Self: ~default {}

struct FunkyView: View {
  typealias Body = SomeType // SomeType.* is required to be Copyable
}

struct UniqueView: (View & ~default) {
  typealias Body = SomeNCType // SomeNCType.* is NOT required to be Copyable
}
```

The asterisk can be used to more narrowly apply to a subset of associated types:

TODO: more on bottomless subjects

Using bottomless subjects, extension defaults can be automatically synthesized
for a type to maintain source compatibility.

- For non-protocol types, all type parameters `T` are `T.* : Copyable`.
- For protocol types, all primary associated types are `T.* : Copyable`.

TODO: reason why it's only limited to primary associated types?


## Acknowledgments

Thank you to Joe Groff and Slava Pestov for their feedback throughout the
development of this proposal.