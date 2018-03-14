Continue to separate internal-only and public sections of the C++ API
----
* Author(s): vjpai
* Approver: abhikumar
* Status: Proposed
* Implemented in: C++
* Last updated: March 13, 2018
* Discussion at: 

## Abstract

Maintain separation of C++ classes and functions that are intended
to be directly used by application developers from those that are
supposed to be used only through codegen or for that exist for gRPC's
internal implementation (e.g., interactions with the core API).

## Background

[Previous work](https://github.com/grpc/proposal/pull/28) started to
separate internal-only from public sections of the gRPC C++ API by
moving selected compoennts to namespace `grpc::internal`. That effort
was primarily focused on those portions of the gRPC API accessed
through generated code. This gRFC intends to take this effort to its
next logical step by reviewing all public APIs, class-by-class and
function-by-function. A key goal here is to reduce or eliminate the
amount of information leakage through the C++ API of implementation
details and core surface API. This is a particular requirement because
the core surface API is inherently less stable than the C++ language API.

### Related Proposals:

- [C++ internalization](https://github.com/grpc/proposal/pull/28)

## Proposal

The C++ gRPC wrapping includes numerous classes and functions, but
they fall into five separate categories:

1. Documented for public use by end-user client/server applications
1. Intended for interfacing with serialization layers such as protobuf
1. Intended for use through a code-generation layer
1. Intended for use in the internal implementation of gRPC C++
1. Intended for interfacing with gRPC core

This gRFC aims to clarify that the first two are intended for public
use (the second one to support custom serializers), but that the last
three should not be used as such. This will be achieved in various
ways:

1. Privatize, protect, or remove features that should have never been
exposed and are not used in application code. In many cases, this
includes constructors to base classes that should only be invoked
through their derived classes. In some cases, classes have more than
one way to get the same information (e.g., `Server::server()` and
`Server::c_server()` both give the address of the wrapped gRPC core
server) and such uses should be de-duped.
1. Deprecate features that leak core surface APIs if they can be
meaningfully substituted with a purely C++ API. Express this with
detailed comments indicating the deprecation and the preferred choice
1. Co-opt items from the core surface API for which there is no other option, but try to give an alternative and more idiomatically C++ version for those when possible
1. Move classes or functions to `namespace grpc::internal` in some
cases
1. Mark pieces of the API as being meant only for
   internal/advanced/specialized use as appropriate if typical
   applications are not expected to use them. These are particularly
   relevant for convenience libraries that wrap gRPC features but
   provide their own tuning parameters.

## Rationale

An important part of API health is clearly identifying the parts of
the API that are not meant for public use so that application
developers can focus on what is actually useful. Although previous
work did this by moving entire functions or classes to `namespace
grpc::internal,` it was not comprehensive across the API, and it does
not easily handle situations where some methods of a class are meant
for public use and others are not. This effort is now much more
fine-grained and comprehensive.

## Implementation

This is implemented in C++ in pull-request
https://github.com/grpc/grpc/pull/14648. Ongoing work will be to
maintain discipline in deciding whether a class or function belongs in
`grpc` or `grpc::internal` as well as deciding whether a class truly
needs a public constructor or whether methods should be marked as
being meant for internal/advanced/specialized use.

## Open issues (if applicable)

This PR will cause problems for gRPC users that are using the
interfaces that are intended for internal use. This does not affect
any bits of code observable to the gRPC team.

