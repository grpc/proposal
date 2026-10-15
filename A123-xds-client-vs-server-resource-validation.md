A123: xDS Client vs. Server Resource Validation
----
* Author(s): @markdroth
* Approver: @ejona, @dfawley
* Implemented in: <language, ...>
* Last updated: 2026-10-15
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

For xDS Listener and RouteConfiguration resources, there are certain
fields that are valid only on clients or only on servers.  However,
currently, gRPC does not know whether a resource will be used on the
client or server side at resource-validation time, which means that
we accept resources that would work on either side.  This can result in
failing data plane RPCs when we later determine that a resource is not
valid on the side on which it's actually being used.

This design proposal changes the resource validation code to know
whether it's being used on the client or server side, so that we can
reject bad resources and avoid failing data plane RPCs.

## Background

There are certain fields in Listener and RouteConfiguration resources
that may or may not be valid, depending on whether they are used on the
client or server side.  For example, on the gRPC client side, routes
must specify a cluster ([A27]), whereas on the server side, routes must
specify a `NonForwardingAction` ([A36]).

When we originally designed xDS support for the gRPC server in [A36],
gRPC was using a global XdsClient instance that was shared across
all channels and servers in the same process.  Because the XdsClient
was shared between clients and servers, the XdsClient could not know
whether any given resource was being used on the client or server side.
Therefore, the design specified that resource validation in the XdsClient
would accept any resource that would be valid on either the client side
or server side, and a resource used on the client side that is valid only
on the server side (or vice-versa) would cause us to fail data plane RPCs.

However, as part of the xDS fallback design in [A71], we switched to
a model where we now have multiple XdsClient instances per process,
and any given XdsClient instance will never be used by both channels
and servers.  Therefore, it is now possible for each XdsClient instance
to know which side it's being used on and use that information when
validation xDS resources.  This means that when an XdsClient instance
for the client side sees a resource that is valid only on the server side
(or vice-versa), we will be able to reject the resource and stick with
the previous valid version (subject to the bootstrap settings introduced
in [A88]), which will avoid failing data plane RPCs.

### Related Proposals:
* [A27: xDS-Based Global Load Balancing][A27]
* [A36: xDS-Enabled Servers][A36]
* [A39: xDS HTTP Filter Support][A39]
* [A71: xDS Fallback][A71]
* [A88: xDS Data Error Handling][A88]

[A27]: A27-xds-global-load-balancing.md
[A36]: A36-xds-for-servers.md
[A39]: A39-xds-http-filters.md
[A71]: A71-xds-fallback.md
[A88]: A88-xds-data-error-handling.md

## Proposal

Each XdsClient instance will know whether it is being used on the client
or server side, and it will use this information during resource
validation to reject resources that are not valid on the side in which
the XdsClient is being used.

Implementations may do this in several ways.  The options include:
1. Having the XdsClient pass some context into the resource validation
   logic (implemented by individual resource types) indicating whether
   the XdsClient is being used on the client or server side.
2. Using separate resource type implementations for Listener and
   RouteConfiguration resources to plug into the XdsClient on the client
   and server sides.

Note that the distinction between client and server side is a gRPC-specific
thing and should not leak into the generic XdsClient APIs that may
eventually be made available as a separate library outside of gRPC.
This distinction must be handled in the gRPC-specific customizations of
the generic XdsClient code.

On the client side, gRPC will reject resources in the following cases:
- A Listener resource that contains a TCP listener instead of an API
  listener (see [A27] and [A36]).
- A listener or RouteConfiguration resource that contains a filter
  config for a filter that is not supported on the client side (see
  [A39]).
- A RouteConfiguration resource that contains a route specifying a
  `NonForwardingAction` (see [A36]).

On the server side, gRPC will reject resources in the following cases:
- A Listener resource that contains an API listener instead of a TCP
  listener (see [A27] and [A36]).
- A listener or RouteConfiguration resource that contains a filter
  config for a filter that is not supported on the server side (see
  [A39]).
- A RouteConfiguration resource that contains a route specifying any
  action other than `NonForwardingAction` (see [A36]).

### Temporary environment variable protection

This proposal does not include any features enabled via external I/O,
so it does not need environment variable protection.

## Rationale

Note that as per [A36], on the gRPC server side, an invalid
RouteConfiguration is handled differently based on how it is obtained:
- If it is inlined in a Listener resource, the Listener resource will be
  considered invalid, and the server will not accept incoming connections.
- If it is obtained separately via RDS, the server will accept incoming
  connections, but it will fail any data plane RPCs that need to use the
  invalid RouteConfiguration.

Because this proposal introduces new ways in which a RouteConfiguration
can be considered invalid, it may trigger this difference in behavior in
new ways.  However, since the fundamental difference itself is not new,
we consider this behavior acceptable.

## Implementation

Implemented in C-core in https://github.com/grpc/grpc/pull/43518.

Will also be implemented in Java and Go.
