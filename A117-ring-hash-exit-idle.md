A117: Ring Hash Exit Idle Behavior Changes
----
* Author(s): dfawley@
* Approver(s): markdroth@, ejona@
* Implemented in:
* Last updated: 2026-05-15
* Discussion at: https://groups.google.com/g/grpc-io/c/ByCPdvi5lrQ

## Abstract

Update the Ring Hash LB policy to create connections when expliictly requested
by the channel to do so, instead of ignoring the request.

## Background

The Ring Hash LB policy, as defined in [gRFC A42][A42], does not create
connections upon creation.  Even though it is not specified, all implementations
of gRPC also ignored calls to the "exit idle" / "request connection" method.

It has been observed that this behavior can cause problems, however.
Specifically, when using the gRPC [connectivity API][], if an application
requests a channel using ring-hash to connect, but ring hash ignores the
incoming request from the channel, the application will never see a change to
the state of the channel.  If the connectivity state is used for something like
computing the health of a server, that can cause serious problems, as an
unhealthy server will not receive the traffic it needs to begin connecting.

### Related Proposals:

* [gRFC A42: xDS Ring Hash LB Policy][A42]
* [gRFC A76: Improvements to the Ring Hash LB Policy][A76]
* [gRFC A61: IPv4 and IPv6 Dualstack Backend Support][A61rh]

## Proposal

This proposal makes two behavior changes:

1. Ring Hash should enter the CONNECTING state when the channel requests it to
   exit idle mode, instead of ignoring the request.

   This will cause the eager-connection logic described in [gRFC A61][A61rh] to
   trigger:

   > when the aggregated connectivity state is either TRANSIENT_FAILURE or
   > CONNECTING, the ring_hash policy proactively triggers connection attempts
   > across all of the subchannels, even without seeing any picks.

2. New connection attempts not triggered by an RPC should begin from a
   randomized location (behavior change from [gRFC A61][A61rh]'s
   TRANSIENT_FAILURE / CONNECTING behavior), which currently states the choice
   of endpoints is up to the implementation:

   > It does not matter which IDLE endpoint is chosen; that is left up to the
   > implementation to determine.

   Using a randomized endpoint will ensure that load is distributed evenly
   across available backends if many individual clients are instructed to
   connect at the same time.

### Temporary environment variable protection

Even though this is a behavior change, as the behavior is triggered by the
application, this feature does not need to be protected by a standardized
environment variable.  Individual langauge implementations may wish to provide
their language-specific mechanism to revert the behavior change temporarily in
the event that issues are encountered by users.

## Rationale

Another option is to change nothing in the Ring Hash policy and require users to
consider "IDLE" as a "healthy" state in their application.  This is still
surprising / unintuitive behavior, and it would be too hard to communicate this
to all users.

## Implementation

The implementation should be done in all languages that support Ring Hash.


[A42]: A42-xds-ring-hash-lb-policy.md
[A76]: A76-ring-hash-improvements.md
[A61rh]: A61-IPv4-IPv6-dualstack-backends.md#ring-hash
[connectivity API]: https://github.com/grpc/grpc/blob/master/doc/connectivity-semantics-and-api.md