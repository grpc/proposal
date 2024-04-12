Server-side Connection Management
----
* Author(s): [Eric Anderson](https://github.com/ejona86)
* Approver: a11r
* Status: Implemented
* Implemented in: C, Java, Go
* Last updated: 2017-01-27
* Discussion at:
  https://groups.google.com/d/topic/grpc-io/DIiIMiq9sV8/discussion

## Abstract

Provide options to allow configuring gRPC servers to sanely manage their
connections:
 1. GC old connections; unused connections can waste server resources
 2. GC dead connections; dead connections accumulate and waste server resources.
    Also, any outstanding RPCs on dead connections waste resources
 3. Close connections to encourage load balancing fairness

## Background

Clients keep connections open after use in case they have further RPCs to
perform. By default, clients do not close the connection after a period of
inactivity because the client may want the lower latency.

TCP does not monitor idle connections; in the absence of writes, TCP will not
notice if the remote machine has crashed, the network failed, and similar. This
means that dead connections will *not* be closed naturally and would accumulate
over time. TCP Keep-Alive can be used to address this problem, although in many
languages it is infeasible to configure and the defaults are frequently too
generous. See [Client-side Keepalive][] for similar client-side discussion.

Level 4 load balancers (L4 LBs) do not balance RPCs but instead TCP connections.
L4 LBs are common due their simplicity and being protocol agnostic.
Unfortunately, with HTTP/2 and clients keeping idle connections open, they are
not able to quickly distribute load away from hot nodes or to new nodes. While
gRPC would encourage using level 7 load balancers, it is not always possible,
practical, or necessary.

[Client-side Keepalive]: A8-client-side-keepalive.md

### Related Proposals:
* A8: [Client-side Keepalive][]

## Proposal

Provide the following configuration options during server creation:
* `MAX_CONNECTION_IDLE`, defaulting to `infinite`
* `MAX_CONNECTION_AGE`, defaulting to `infinite`
* `MAX_CONNECTION_AGE_GRACE`, defaulting to `infinite`
* `KEEPALIVE_TIME`, defaulting to `2 hours`
* `KEEPALIVE_TIMEOUT`, defaulting to `20 seconds`

`MAX_CONNECTION_IDLE` is a duration for the amount of time after which an idle
connection would be closed. To close the server should send GOAWAY with status
code NO\_ERROR and ASCII debug data `max_idle` and then continue the [graceful
connection termination][]. Idleness duration is defined since the most recent
time the number of outstanding RPCs became zero or the connection establishment.

`MAX_CONNECTION_AGE` is a duration for the maximum amount of time a connection
may exist before it will be shut down. When a connection exceeds the configured
age the server should send GOAWAY with status code NO\_ERROR and ASCII debug data
`max_age` and then continue the [graceful connection termination][].
Specifically, outstanding RPCs should be permitted to complete, and
implementations are encouraged to initially set the last stream identifier to
2<sup>31</sup>-1. This option can force the client to periodically reconnect to
a L4 LB which allows spreading load more quickly. A per-connection random jitter
of +/-10% will be added to `MAX_CONNECTION_AGE` to spread out connection storms.
Note that this option without jitter would not create connection storms by
itself, but if there happened to be a connection storm it could cause it to
repeat at a fixed period.

`MAX_CONNECTION_AGE_GRACE` is the maximum time the connection will be kept alive
for outstanding RPCs to complete, ideally relative to the second GOAWAY of the
graceful connection termination. No jitter is applied to the grace period. Note
that implementations, as general practice, should limit the maximum time between
the first and second GOAWAY of the graceful connection termination, but it
should rarely have an impact and so is treated as implementation-specific for
this specification.

The `KEEPALIVE_*` options solve the dead connection issue by monitoring
connection health. It should be handled like [Client-side Keepalive][] using
HTTP/2 PINGs, except the keepalives should take place independent of whether
there are outstanding RPCs.

[graceful connection termination]: http://httpwg.org/specs/rfc7540.html#rfc.section.6.8

## Rationale

The main benefit of these solutions is they are simple, easy to communicate, and
resolve the problems fairly strongly. The server-side keepalive is by far the
most complex, but the same general logic is also available on client-side so the
two can share code.

Since all options discussed here are server-side, they are able to be managed by
the service owner and configured as appropriate. This means that accidental DDoS
risk is reduced and would be able to be addressed by rolling out new
configuration for the service, without the the need to update the clients.

These server-side behaviors are not latency sensitive; they simply need to
happen *eventually*. Implementations are generally free to implement them quite
coarsely without applications noticing.

The main exception is implementations do need to ensure they don't encourage
stampeding herds and other second-order effects. This should not be a problem
for idle or dead connections, but could be a problem for handling L4 LBs since
clients are not idle and may be sending a high RPC load. Concretely, we should
be aware that closing a connection will cause the client to open a new one; if
many connections are closed at once, then many connections will happen all at
once. And also when using GOAWAY on a connection that has long-lived outstanding
RPCs, the system can accidentally degrade to having many old but active
connections with few RPCs on them.

## Implementation

* C. TODO get current progress; close, if not done. Being done by @y-zeng.
  Mapping from the option name in this spec to the C core option name:
  * `MAX_CONNECTION_IDLE`: `GRPC_ARG_MAX_CONNECTION_IDLE_MS`
  * `MAX_CONNECTION_AGE`: `GRPC_ARG_MAX_CONNECTION_AGE_MS`
  * `MAX_CONNECTION_AGE_GRACE`: `GRPC_ARG_MAX_CONNECTION_AGE_GRACE_MS`
  * `KEEPALIVE_TIME`: `GRPC_ARG_KEEPALIVE_TIME_MS`
  * `KEEPALIVE_TIMEOUT`: `GRPC_ARG_KEEPALIVE_TIMEOUT_MS`
* Java. Complete since grpc/grpc-java@4a96e259 in v1.4.0. Bug fix in v1.7.1.
  * The options can be specified via `NettyServerBuilder.maxConnectionIdle()`,
    `maxConnectionAge()`, `maxConnectionAgeGrace()`, `keepAliveTime()`, and
    `keepAliveTimeout()`.
* Go. TODO get current progress; close, if not done. Being done by @MakMukhi
