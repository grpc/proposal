A105: MAX_CONCURRENT_STREAMS Connection Scaling
----
* Author(s): @markdroth, @dfawley
* Approver: @ejona86
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2025-10-09
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

We propose allowing gRPC clients to automatically establish new
connections to the same endpoint address when it hits the HTTP/2
MAX_CONCURRENT_STREAMS limit for an existing connection.

## Background

HTTP/2 contains a connection-level setting called
[MAX_CONCURRENT_STREAMS][H2MCS] that limits the number of streams a peer
may initiate.  This is often set by servers or proxies to protect against
a single client using too many resources.  However, in an environment with
reverse proxies or virtual IPs, it may be possible to create multiple
connections to the same IP address that lead to different physical
servers, and this can be a feasible way for a client to achieve more
throughput (QPS) without overloading any server.

Today, the MAX_CONCURRENT_STREAMS limit is visible only inside the
transport, not to anything at the channel layer.  This means that the
channel may dispatch an RPC to a subchannel that is already at its
MAX_CONCURRENT_STREAMS limit, even if there are other subchannels that
are not at that limit and would be able to accept the RPC.  In this
situation, the transport normally queues the RPC until it can start
another stream (i.e., either when one of the existing streams ends or
when the server increases its MAX_CONCURRENT_STREAMS limit).

Applications today are typically forced to deal with this problem
by creating multiple gRPC channels to the same target (and disabling
subchannel sharing, for implementations that share subchannels between
channels) and doing their own load balancing across those channels.
This is an obvious flaw in the gRPC channel abstraction, because the
channel is intended to hide connection management details like this so
that the application doesn't have to know about it.  It is also not a
very flexible solution for the application in a case where the server is
dynamically tuning its MAX_CONCURRENT_STREAMS setting, because the
application cannot see that setting in order to tune the number of
channels it uses.

### Related Proposals: 
* [A9: Server-side Connection Management][A9]
* [A61: IPv4 and IPv6 Dualstack Backend Support][A61]

[A9]: A9-server-side-conn-mgt.md
[A61]: A61-IPv4-IPv6-dualstack-backends.md
[H2MCS]: https://httpwg.org/specs/rfc7540.html#SETTINGS_MAX_CONCURRENT_STREAMS

## Proposal

We will add a mechanism for the gRPC client to dynamically scale up the
number of connections to a particular endpoint address when it hits the
MAX_CONCURRENT_STREAMS limit.  This functionality will be implemented at
the subchannel layer.

There are several parts to this proposal:
- Configuration for the max number of connections per subchannel, via
  either the [service
  config](https://github.com/grpc/grpc/blob/master/doc/service_config.md)
  or via a per-endpoint attribute in the LB policy tree.
- A mechanism for the transport to report the current MAX_CONCURRENT_STREAMS
  setting to the subchannel layer.
- Connection scaling functionality in the subchannel.
- Functionality to pick a connection for each RPC in the subchannel,
  including the ability to queue RPCs while waiting for new connections
  to be established.

### Configuration

- service config
- per-endpoint attribute in the LB policy tree
  - needs to NOT affect subchannel identity
  - if subchannels are shared between channels, will use the highest
    value of any channel sharing the subchannel
  - in C-core, will just have a channel arg that the channel will treat
    specially -- this works because "updating" a subchannel is done by
    "recreating" it
  - in java/go, consider some sort of injector approach?
  - note: requires implementations to have only one address per subchannel,
    as per A61 (right?)
- xDS story (per-host circuit breaker)
- disabled by default (max connections per subchannel == 1)

### Transport Reporting Current MAX_CONCURRENT_STREAMS

- transport needs to report this to subchannel
- in C-core, maybe revamp the connectivity state API between transport
  and subchannel?  was thinking about doing this anyway for subchannel
  metrics disconnection reason -- but need to figure out a plan for
  direct channels

### Connection Scaling Within the Subchannel

- conditions for scaling up
- connections are removed when they terminate (depends on A9 configured
  on server side)
- need to document connectivity state behavior (and update client
  channel spec!)

### Picking a Connection for Each RPC

- algorithm for picking a connection for each RPC
  - show picker pseudo-code
- RPC queueing in the subchannel
- describe synchronization

### Temporary environment variable protection

Enabling this feature via either the gRPC service config or xDS will
initially be guarded via the environment variable
`GRPC_EXPERIMENTAL_MAX_CONCURRENT_STREAMS_CONNECTION_SCALING`.  The
feature will be enabled by default once it has passed interop tests.

- TODO: define interop tests?

## Rationale

[A discussion of alternate approaches and the trade offs, advantages, and disadvantages of the specified approach.]

- TODO: document reasons why not doing this in LB policy tree

## Implementation

Will be implemented in C-core, Java, and Go.

## Open issues (if applicable)

[A discussion of issues relating to this proposal for which the author does not know the solution. This section may be omitted if there are none.]
