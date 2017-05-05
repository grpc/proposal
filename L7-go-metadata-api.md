Go Metadata API Change
----
* Author(s): Doug Fawley
* Approver: a11r
* Status: Ready for Implementation
* Last updated: 2017/05/04
* Discussion at:

## Abstract

Remove the `FromContext` and `NewContext` functions from the `grpc/metadata`
package.

## Background

As documented in
issue [grpc/grpc-go#1148](https://github.com/grpc/grpc-go/issues/1148), metadata
was forwarded automatically by any Go server using the incoming context when
performing outgoing gRPC calls, which is the standard practice for handling
contexts.  This behavior represents a security risk, as it exposes potentially
sensitive information in the metadata, e.g. authentication certificates.

In PR [grpc/grpc-go#1157](https://github.com/grpc/grpc-go/pull/1157), this
security issue was fixed by separating the incoming and outgoing metadata in the
context.  A new API was introduced to set and retrieve these two sets of
metadata.  The old API was left in place to support backward compatibility, with
the assumption that callers of `metadata.FromContext` were intending to retrieve
the incoming metadata and callers of `metadata.NewContext` were intending to set
the outgoing metadata.  Unfortunately, this is not the case for interceptors --
client interceptors typically intend to retrieve the outgoing context (to verify
or extend it), and server interceptors typically intend to add additional
metadata to the incoming context -- and tests might reasonably intend anything.
As a result, existing interceptors were broken (see
issue [grpc/grpc-go#1219](https://github.com/grpc/grpc-go/issues/1219) for one
such example).

## Proposal

Because it is impossible to know the intentions of the caller, we propose to
break backward compatibility with the old API by removing the `FromContext` and
`NewContext` functions from the metadata package.  This will force maintainers
to determine which metadata was intended, and update their code accordingly.
The existing `FromIncomingContext`, `FromOutgoingContext` (rare),
`NewIncomingContext` (rare), and `NewOutgoingContext` should be used instead to
read and set metadata.

## Rationale

Because the original API assumed only one copy of metadata was present in the
context, no alternative was identified that could both maintain backward
compatibility and not present a security risk.

## Implementation

The implementation is straightforward: simply remove the functions.  If any
usages remain within grpc itself, they will be inspected and updated on a
case-by-case basis.
