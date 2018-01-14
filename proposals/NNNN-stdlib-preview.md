# Standard Library Preview Package

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Ben Cohen](https://github.com/airspeedswift)
* Review Manager: TBD
* Status: **Awaiting review**

## Introduction

This proposal introduces a new "staging area" for additions to the standard
library, as a package that follows the evolution process, allowing for use
and feedback on proposed APIs in a production setting prior to adding them
to the ABI-stable standard library.

## Motivation

There are a large number of features to the Swift standard library which would
be considered standard for a language's standard library. Expansions of the
String API, the addition of more collections algorithms such as for searching
and sorting, and other collections such as sorted sets/dictionaries, are all
important for a full "batteries included" set of features.

Prior to ABI stability, the standard library needed to be bundled with an
application. This meant that the library needed to be kept to a "bare minimum"
due to app size concerns. With ABI stability, apps can use a standard library
embedded with the OS, relieving this pressure somewhat (though not entirely –
size does still matter).

But the flip side of this is that an ABI stable library needs to take extreme
care to get APIs right on the first iteration, because once published in an
ABI-stable library, there are restrictions on how the API can be evolved in
future versions. An open process of feedback for the standard library has been
highly useful in improving it's usability and functionality.

Additionally, development of algorithms on the standard library is a difficult
process:

 - You must first build the full Swift compiler (which requires building
LLVM, installing cmake etc). 
 - Working on code within the standard library is a
very slow process - the standard library takes several minutes to build so
write/build/test cycles are very slow (this is the reason for the `Prototypes`
folder).
 - Adding and running tests via the lit infrastructure has a steep learning curve.
 - Working on an ABI-stable library with pervasive inlining, such as the standard
library requires, will need to be made extremely carefully, and PRs touching
inlined code will need intensive review, even in the presence of comprehensive
unit tests.

By contrast, a stand-alone package of algorithms, built with SwiftPM, and
tested with XCTest, will be much more approachable as a platform for
experienced Swift developers who want to make proposals or fix bugs and improve
performance.

## Proposed solution

A new repo/SwiftPM package will be created. This "preview" package will
include algorithms proposed for addition to the standard library in a future
release of Swift.

Additions to this package will follow the standard evolution process and will
require a review on the evolution forums, and approval by the core team.

Given this library will be a source package, more flexible evolution of APIs will
be possible. The library should follow the semantic versioning conventions, so
major versions can make source-breaking changes. However, since the goal of
the package is to encourage production use and feedback,
each major version should continue to be maintained in a useable state
with each new release of Swift.

The goal of every addition to the library is that it will eventually become part
of the official standard library. This would be done through a evolution-reviewed revision
to the proposal that added it to the preview package. When a feature does eventually migrate
to the standard library, it would be deprecated from the preview library in the current major
version (with a `renamed` entry pointing to the Swift standard library equivalent), and
obsoleted in the subsequent major version.

A process would need to be established for tracking the migration process of additions
to ensure that they do eventually migrate. Entries in the preview library that do not after
a prolonged period of time should be removed.

## Detailed design


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

Does the proposal change the ABI of existing language features? The
ABI comprises all aspects of the code generation model and interaction
with the Swift runtime, including such things as calling conventions,
the layout of data types, and the behavior of dynamic features in the
language (reflection, dynamic dispatch, dynamic casting via `as?`,
etc.). Purely syntactic changes rarely change existing ABI. Additive
features may extend the ABI but, unless they extend some fundamental
runtime behavior (such as the aforementioned dynamic features), they
won't change the existing ABI.

Features that don't change the existing ABI are considered out of
scope for [Swift 4 stage 1](README.md). However, additive features
that would reshape the standard library in a way that changes its ABI,
such as [where clauses for associated
types](https://github.com/apple/swift-evolution/blob/master/proposals/0142-associated-types-constraints.md),
can be in scope. If this proposal could be used to improve the
standard library in ways that would affect its ABI, describe them
here.

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

