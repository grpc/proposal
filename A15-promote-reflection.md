Promote Reflection
----
* Author(s): Carl Mastrangelo (carl-mastrangelo)
* Approver: a11r
* Status: Draft
* Implemented in:
* Last updated: 2018-05-30
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/AOgFISlAgxk

## Abstract

Promote the Reflection Service from v1alpha to v1

## Background

Reflection is a means by which a server can describe what messages it
supports.  Both the protocol and the [description](
https://github.com/grpc/grpc/blob/master/doc/server-reflection.md) of the
reflection service have not changed for a long time.

## Proposal

It is proposed that the package name (and corresponding directory structure)
for [reflection.proto](
https://github.com/grpc/grpc/blob/v1.12.x/src/proto/grpc/reflection/v1alpha/reflection.proto)
be changed from `grpc.reflection.v1alpha` to `grpc.reflection.v1`.  Java and Go
implementations of the gRPC reflection service should also be updated to match.
Additionally, the canonical proto definition should be created in the
[grpc-proto](https://github.com/grpc/grpc-proto) repository to serve as a source
of truth.

To facilitate the package name change, the new location and package of the
proto will be created.  This will involve copying the existing proto file to
the new destination.  All clients will be adapted to prefer the new service
name.  All servers will dual support both services for a release.  Lastly, the
old service will be deprecated and marked for removal in the near future.

## Rationale

It is unlikely that the reflection proto service will change in
backwards-incompatible ways.  The service has been implemented in each language
implementation of gRPC and has not changed in two years.

## Implementation

1.  Copy grpc/reflection/v1alpha/reflection.proto to
    grpc/reflection/v1/reflection.proto
2.  Update existing service implementations in each repo to support the new
    location in addition to the old location.
3.  The old reflection.proto will be marked deprecated and for removal.
4.  Clients (such as grpc_cli) will be updated to dual request reflection
    information, preferring the new location first.
5.  In an upcoming release (e.g. v1.15.x), the new proto will be announced
    and the old proto will be declared for removal.
6.  In the subsequent release (e.g. 1.16.x), the old implementations will be
    removed, clients updated to not dual request, and the old proto removed.

