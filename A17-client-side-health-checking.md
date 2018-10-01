Client-Side Health Checking
----
* Author(s): Mark D. Roth (roth@google.com)
* Approver: a11r
* Status: Draft
* Implemented in: 
* Last updated: 2018-08-27
* Discussion at: https://groups.google.com/d/topic/grpc-io/rwcNF4IQLlQ/discussion

## Abstract

This document proposes a design for supporting application-level
health-checking on the client side.

## Background

gRPC has an existing [health-checking
mechanism](https://github.com/grpc/grpc/blob/master/doc/health-checking.md),
which allows server applications to signal that they are not healthy
without actually tearing down connections to clients.  This is
used when (e.g.) a server is itself up but another service that it depends
on is not available.

Currently, this mechanism is used in some [look-aside
load-balancing](https://github.com/grpc/grpc/blob/master/doc/load-balancing.md)
implementations to provide centralized health-checking of backend
servers from the balancer infrastructure.  However, there is no
support for using this health-checking mechanism from the client side,
which is needed when not using look-aside load balancing (or when
falling back from look-aside load balancing to directly contacting
backends when the balancers are unreachable).

### Related Proposals

N/A

## Proposal

The gRPC client will be able to be configured to send health-checking RPCs
to each backend that it is connected to.  Whenever a backend responds
as unhealthy, the client's LB policy will stop sending requests to that
backend until it reports healthy again.

Note that because the health-checking service requires a service name, the
client will need to be configured with a service name to use.  However,
by default, it can use the empty string, which would mean that the
health of all services on a given host/port would be controlled with a
single switch.  Semantically, the empty string is used to represent the
overall health of the server, as opposed to the health of any individual
services running on the server.

### Watch-Based Health Checking Protocol

The current health-checking protocol is a unary API, where the client is
expected to periodically poll the server.  That was sufficient for use
via centralized health checking via a look-aside load balancer, where
the health checks come from a small number of clients and where there
is existing infrastructure to periodically poll each client.  However,
if we are going to have large numbers of clients issuing health-check
requests, then we will need to convert the health-checking protocol to
a streaming Watch-based API for scalability and bandwidth-usage reasons.

Note that one down-side of this approach is that it could conceivably
be possible that the server-side health-checking code somehow fails
to send an update when becoming unhealthy.  If the problem is due to
a server that stops polling for I/O, then the problem would be caught
by keepalives, at which point the client would disconnect.  But if the
problem is caused by a bug in the health-checking service, then it's
possible that a server could still be responding but has failed to notify
the client that it is unhealthy.

#### API Changes

For the proposed API, see https://github.com/grpc/grpc-proto/pull/33.

We propose to add the following RPC method to the health checking service:

```
rpc Watch(HealthCheckRequest) returns (stream HealthCheckResponse);
```

This is exactly the same as the existing `Check()` method, except that it
is server-side streaming instead of unary.  The server will be expected
to send a message immediately upon receiving the message from the client
and will then be expected to send a new message whenever the requested
service's health changes.

We will also add a new enum value to the `ServingStatus` enum:

```
SERVICE_UNKNOWN = 3;
```

This serving status will be returned by the `Watch()` method when the
requested service name is initially unknown by the server.  Note that the
existing `Check()` unary method fails with status `NOT_FOUND` in this case,
and we are not proposing to change that behavior.

### Client Behavior

Client-side health checking will be disabled by default;
service owners will need to explicitly enable it via the [service
config](#service-config-changes) when desired.  There will be a channel
argument that can be used on the client to disable health checking even
if it is enabled in the service config.

The health checking client code will be built into the subchannel,
so that individual LB policies do not need to explicitly add support
for health checking. (However, see also "LB Policies Can Disable Health
Checking When Needed" below.)

Note that when an LB policy sees a subchannel go into state
`TRANSIENT_FAILURE`, it will not be able to tell whether the subchannel
has become disconnected or whether the backend has reported that it
is unhealthy.  The LB policy may therefore take some unnecessary but
unharmful steps in response, such as asking the subchannel to connect
(which will be a no-op if it's already connected) or asking the resolver
to re-resolve (which will probably be throttled by the cooldown timer).

#### Client Workflow

When a subchannel first establishes a connection, if health checking
is enabled, the client will immediately start the `Watch()` call to the
backend.  The subchannel will stay in state CONNECTING until it receives
the first health-check response from the backend (see "Race Conditions
and Subchannel Startup" below).

When a client receives a health-checking response from the backend,
if the health check response indicates that the backend is healthy, the
subchannel will transition to state READY; otherwise, it will transition
to state `TRANSIENT_FAILURE`.  Note that this means that when a backend
transitions from unhealthy to healthy, the subchannel's connectivity state
will transition from `TRANSIENT_FAILURE` directly to `READY`, with no stop at
CONNECTING in between.  This is a new transition that was not [previously
supported](https://github.com/grpc/grpc/blob/master/doc/connectivity-semantics-and-api.md), although it is unlikely to be a problem for applications,
because (a) it won't affect the overall state of the parent channel
unless all subchannels are in the same state at the same time and (b)
because the connectivity state API can drop states, the application won't
be able to tell that we didn't stop in `CONNECTING` in between anyway.

If the `Watch()` call fails with status `UNIMPLEMENTED`, the client will
act as if health checking is disabled.  That is, it will not retry
the health-checking call, but it will treat the channel as healthy
(connectivity state `READY`).  However, the client will record a [channel
trace](https://github.com/grpc/proposal/blob/master/A3-channel-tracing.md)
event indicating that this has happened.  It will also log a message at
priority ERROR.

If the `Watch()` call returns any other status, the subchannel will
transition to connectivity state `TRANSIENT_FAILURE` and will retry the
call.  To avoid hammering a server that may be experiencing problems,
the client will use exponential backoff between attempts.  When the client
receives a message from the server on a given call, the backoff state is
reset, so the next attempt will occur immediately, but any subsequent
attempt will be subject to exponential backoff.  When the next attempt
starts, the subchannel will transition to state `CONNECTING`.

When the client subchannel is shutting down or when the backend sends a
GOAWAY, the client will cancel the `Watch()` call.  There is no need to
wait for the final status from the server in this case.

#### Race Conditions and Subchannel Startup

Note that because of the inherently asynchronous nature of the network,
whenever a backend transitions from healthy to unhealthy, it may still
receive some number of RPCs that were already in flight from the
client before the client received the notification that the backend is
unhealthy.  This race condition lasts approximately the network round
trip time (it includes the one-way trip time for RPCs that were already
sent by the client but not yet received by the server when the server
goes unhealthy, plus the one-way trip time for RPCs sent by the client
between when the server sends the unhealthy notification and when the
client receives it).

When the connection is first established, however, the problem could
affect a larger number of RPCs, because there could be a number of RPCs
queued up waiting for the channel to become connected, which would all
be sent at once.  And unlike an already-established connection, the race
condition is avoidable for newly established connections.

To avoid this, the client will wait for the initial health-checking
response before the subchannel goes into state `READY`.  However, this
does mean that when health checking is enabled, we require an additional
network RTT before the subchannel can be used.  If this becomes a problem
for users that cannot be solved by simply disabling health-checking,
we can consider adding an option in the future to treat the subchannel
as healthy until the initial health-checking response is received.

#### Call Credentials

If there are any call credentials associated with the channel, the client
will send those credentials with the `Watch()` call.  However, we will not
provide a way for the client to explicitly add call credentials for the
`Watch()` call.

### Service Config Changes

We will add the following new message to the [service
config](https://github.com/grpc/grpc/blob/master/doc/service_config.md):

```
message HealthCheckConfig {
  // Service name to use in the health-checking request.
  google.protobuf.StringValue service_name = 1;
}
```

We will then add the following top-level field to the ServiceConfig proto:

```
HealthCheckConfig health_check_config = 4;
```

Note that we currently need only this one parameter, but we are putting
it in its own message so that we have the option of adding more parameters
later if needed.

Here is an example of how one would set the health checking service name
to "foo" in the service config in JSON form:

```
"healthCheckConfig": {"serviceName": "foo"}
```

### LB Policies Can Disable Health Checking When Needed

There are some cases where an LB policy may want to disable client-side
health-checking.  To allow this, LB policies will be able to set the
channel argument mentioned above to inhibit health checking.

This section details how each of our existing LB policies will interact
with health checking.

#### `pick_first`

We do not plan to support health checking with `pick_first`, because it
is not clear what the behavior of `pick_first` would be for unhealthy
channels.  The naive approach of treating an unhealthy channel as having
failed and disconnecting would both result in expensive reconnection
attempts and might not actually cause us to select a different backend
after re-resolving.

The `pick_first` LB policy will unconditionally set the channel arg to
inhibit health checking.

#### `round_robin`

Unhealthiness in `round_robin` will be handled in the obvious way: the
subchannel will be considered to not be in state `READY`, and picks will
not be assigned to it.

#### `grpclb`

The `grpclb` LB policy will set the channel arg to inhibit health checking
when we are using backend addresses obtained from the balancer, on the
assumption that the balancer is doing centralized health-checking.  The arg
will not be set when using fallback addresses, since we want client-side
health checking for those.

### Caveats

Note that health checking will use a stream on every connection.
This stream counts toward the HTTP/2 `MAX_CONCURRENT_STREAMS` setting.
So, for example, any gRPC client or server that sets this to 1 will not
allow any other streams on the connection.

Enabling health checks may affect the `MAX_CONNECTION_IDLE` setting.
We do not expect this to be a problem, since we are only implementing
client-side health checking in `round_robin`, which always attempts to
reconnect to every backend after an idle timeout anyway.

## Rationale

We discussed a large number of alternative approaches.  The two most
appealling are listed here, along with the reasons why they were not
chosen.

1. Use HTTP/2 SETTINGS type to indicate unhealthiness.  Would avoid proto
dependency issues in core.  However, would be transport-specific and would
not work for HTTP/2 proxies.  Also would not allow signalling different
health status for multiple services on the same port.  Would also
require server applications to use different APIs for the centralized
health checking case and the client-side health checking case.

2. Have server stop listening when unhealthy.  This would be challenging
to implement in Go due to the structure of the listener API.  It would
also not allow signalling different health status for multiple services on
the same port.  Would also require server applications to use different
APIs for the centralized health checking case and the client-side health
checking case.  Would not allow individual clients to decide whether to
do health checking.  In cases where the server's health was flapping up
and down, would cause a lot of overhead from connections being killed
and reestablished.

## Implementation

### C-core

- Implement new `Watch()` method in health checking service
  (https://github.com/grpc/grpc/pull/16351).
- Add support for health checking in subchannel code using nanopb
  (in progress).

### Java

- Implement new `Watch()` method in health checking service.
- May need to redesign `IDLE_TIMEOUT`. Currently based on number of RPCs
  on each transport (zero vs non-zero). Will need some way to exclude
  the health-check RPC from this count.
- GRPC-LB needs to be aware of the setting to disable health checking on
  its connections to backends.
- May need yet another runtime registry to find client health checking
  implementation. Alternatively may hand-roll the protobuf
  serialization.
- Will need plumbing to allow creating a trace event.

### Go

- Implement new `Watch()` method in health checking service.
- Allow RPCs on a subchannel.
- Add mechanism in subconn for the health-check implementation,
  optionally invoking it when subchannels are created, and affecting
  connectivity state accordingly.
- Implement and register health checking in an optional package (to
  enable/configure feature).
