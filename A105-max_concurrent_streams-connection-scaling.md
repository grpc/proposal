A105: MAX_CONCURRENT_STREAMS Connection Scaling
----
* Author(s): @markdroth, @dfawley
* Approver: @ejona86
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2025-10-20
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
* [A6: gRPC Retry Design][A6]
* [A9: Server-side Connection Management][A9]
* [A32: xDS Circuit Breaking][A32]
* [A61: IPv4 and IPv6 Dualstack Backend Support][A61]
* [A79: Non-per-call Metrics Architecture][A79]
* [A94: OTel Metrics for Subchannels][A94]

[A6]: A6-client-retries.md
[A9]: A9-server-side-conn-mgt.md
[A32]: A32-xds-circuit-breaking.md
[A61]: A61-IPv4-IPv6-dualstack-backends.md
[A79]: A79-non-per-call-metrics-architecture.md
[A94]: A94-subchannel-otel-metrics.md
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
- Modify pick_first to handle the subchannel transitioning from READY to
  CONNECTING state.

### Configuration

Connection scaling will be configured via a new service config field,
as follows (schema shown in protobuf form, although gRPC actually accepts
the service config in JSON form):

```proto
message ServiceConfig {
  // ...existing fields...

  // Settings to control dynamic connection scaling.
  message ConnectionScaling {
    // Maximum connections gRPC will maintain for each subchannel in
    // this channel.  When no streams are available for an RPC in a
    // subchannel, gRPC will automatically create new connections up
    // to this limit.  If this value changes during the life of a
    // channel, existing subchannels will be updated to reflect
    // the change.  No connections will be closed as a result of
    // lowering this value; down-scaling will only happen as
    // connections are lost naturally.
    //
    // Values higher than the client-enforced limit (by default, 10)
    // will be clamped to that limit.
    google.protobuf.UInt32Value max_connections_per_subchannel = 1;
  }
  ConnectionScaling connection_scaling = N;
}
```

In the subchannel, connection scaling will be configured via a parameter
called max_connections_per_subchannel, which be passed into the subchannel
via a per-endpoint attribute.  Configuring this via the service config
will effectively set that per-endpoint attribute before passing the list
of endpoints into the LB policy, but the attribute can also be set or
modified by the LB policy.

As indicated in the comment above, the channel will enforce a maximum
limit for the max_connections_per_subchannel attribute.  This limit
will be 10 by default, but gRPC will provide a channel-level setting to
allow a client application to raise or lower that limit.  Whenever the
max_connections_per_subchannel attribute is larger than the channel's
limit, it will be capped to that limit.  This capping will be performed
in the subchannel itself, so that it will apply regardless of where the
attribute is set.

The max_connections_per_subchannel attribute can change with each resolver
update, regardless of whether it is set via the service config or via an
LB policy.  When this happens, we do not want to throw away the subchannel
and create a new one, since that would cause unnecessary connection churn.
This means that the max_connections_per_subchannel attribute must not
be considered part of the subchannel's unique identity that is set only
at subchannel creation time; instead, it must be an attribute that can
be changed over the life of a subchannel.  The approach for this will be
different in C-core than in Java and Go.

If the max_connections_per_subchannel attribute is unset, the subchannel
will assume a default of 1, which effectively means the same behavior
as before this gRFC.

#### C-core

In C-core, every time there is a resolver update, the LB policy
calls `CreateSubchannel()` for every address in the new address list.
The `CreateSubchannel()` call returns a subchannel wrapper that holds
a ref to the underlying subchannel.  The channel uses a subchannel
pool to store the set of currently existing subchannels: the requested
subchannel is created only if it doesn't already exist in the pool;
otherwise, the returned subchannel wrapper will hold a new ref to the
existing subchannel, so that it doesn't actually wind up creating a new
subchannel (only a new subchannel wrapper).  This means that we do not
want the max_connections_per_subchannel attribute to be part of the
subchannel's key in the subchannel pool, or else we will wind up
recreating the subchannel whenever the attribute's value changes.

In addition, by default, C-core's subchannel pool is shared between
channels, meaning that if two channels attempt to create the same
subchannel, they will wind up sharing a single subchannel.  In this
case, each channel using a given subchannel may have a different value
for the max_connections_per_subchannel attribute.  The subchannel will
use the maximum value set for this attribute across all channels.

To support this, the implementation will be as follows:
- The subchannel will store a map from max_connections_per_subchannel
  value to the number of subchannel wrappers currently holding a ref to
  it with that value.  Entries are added to the map whenever the first
  ref for a given value of max_connections_per_subchannel is taken, and
  entries are removed from the map whenever the last ref for a given
  value of max_connections_per_subchannel is removed.  Whenever the set
  of entries changes, the subchannel will do a pass over the map to find
  the new max value to use.
- We will create a new channel arg called
  `GRPC_ARG_MAX_CONNECTIONS_PER_SUBCHANNEL`, which will be treated
  specially by the channel.  Specifically, when this attribute is passed
  to `CreateSubchannel()`, it will be excluded from the channel args
  that are used as a key in the subchannel pool.  Instead, when the
  subchannel wrapper is instantiated, it will call a new API on the
  underlying subchannel to tell it that a new ref is being held for this
  value of max_connections_per_subchannel.  When the subchannel wrapper
  is orphaned, it will call a new API on the underlying subchannel
  to tell it that the ref is going away for a particular value of
  max_connections_per_subchannel.

#### Java and Go

In Java and Go, there is no subchannel pool, and LB policies will not
call `CreateSubchannel()` for any address for which they already have a
subchannel from the previous address list.  There is also no need to
deal with the case of multiple channels sharing the same subchannel.
Therefore, a different approach is called for.  However, it seems
desirable to design this API such that it leaves open the possibility of
Java and Go introducing a subchannel pool at some point in the future.

TODO: flesh this out, maybe use some sort of injector approach?

Note that this approach assumes that Java and Go have switched to a
model where there is only one address per subchannel, as per [A61].

#### xDS Configuration

In xDS, the max_connections_per_subchannel value will be configured via
a per-host circuit breaker in the CDS resource.  This uses a similar
structure to the circuit breaker described in [A32].

In the CDS resource, in the
[circuit_breakers](https://github.com/envoyproxy/envoy/blob/ed76c2e81f428248f682a9a380a4eef476ea4349/api/envoy/config/cluster/v3/cluster.proto#L885)
field, we will now add support for the following field:
- [per_host_thresholds](https://github.com/envoyproxy/envoy/blob/ed76c2e81f428248f682a9a380a4eef476ea4349/api/envoy/config/cluster/v3/circuit_breaker.proto#L120):
  As in [A32], gRPC will look only at the first entry for priority
  [DEFAULT](https://github.com/envoyproxy/envoy/blob/6ab1e7afbfda48911e187c9d653a46b8bca98166/api/envoy/config/core/v3/base.proto#L39).
  If that entry is found, then within that entry:
  - [max_connections](https://github.com/envoyproxy/envoy/blob/ed76c2e81f428248f682a9a380a4eef476ea4349/api/envoy/config/cluster/v3/circuit_breaker.proto#L59):
    If this field is set, then its value will be used to set the
    max_connections_per_subchannel attribute for all endpoints for that
    xDS cluster.  If it is unset, then the
    max_connections_per_subchannel attribute will remain unset.  A value
    of 0 will be rejected at resource validation time.

A new field will be added to the parsed CDS resource representation
containing the value of this field.

### Transport Reporting Current MAX_CONCURRENT_STREAMS

In order for the subchannel to know when to create a new connection, the
transport will need to report the current value of the peer's
MAX_CONCURRENT_STREAMS setting up to the subchannel.

TODO: in C-core, maybe revamp the connectivity state API between transport
and subchannel?  was thinking about doing this anyway for subchannel
metrics disconnection reason -- but need to figure out a plan for
direct channels

### Subchannel Behavior

The connection scaling functionality in the subchannel will be used if
the max_connections_per_subchannel attribute is greater than 1.

If the value is 1 (or unset), then implementations must not impose any
additional per-RPC overhead at this layer beyond what already exists
today.  In other words, the connection scaling feature must not affect
performance unless it is actually enabled.

#### Connection Management

When max_connections_per_subchannel is greater than 1, the subchannel will
contain up to that number of established connections.  The connections
will be stored in a list ordered by the time at which the connection was
established, so that the oldest connection is at the start of the list.

Each connection will store the peer's MAX_CONCURRENT_STREAMS setting
reported by the transport.  It will also track how many RPCs are
currently in flight on the connection.

The subchannel will also track one in-flight connection attempt, which
will be unset if no connection attempt is currently in flight.

A new connection attempt will be started when all of the
following are true:
- One or more RPCs are queued in the subchannel, waiting for a
  connection to be sent on.
- No existing connection in the subchannel has any available streams --
  i.e., the number of RPCs currently in flight on each connection is
  greater than or equal to the connection's MAX_CONCURRENT_STREAMS.
- The number of existing connections in the subchannel is fewer than the
  max_connections_per_subchannel value.
- There is no connection attempt currently in flight on the subchannel.

The subchannel will never close a connection once it has been established.
However, when a connection is closed for any reason, it is removed from
the subchannel.  If the application wishes to garbage collect unused
connections, it should configure MAX_CONNECTION_IDLE on the server side,
as described in [A9].

Some examples of cases that can trigger a new connection attempt to be
started:
- A new RPC is started on the subchannel and all existing connections
  are already at their MAX_CONCURRENT_STREAMS limit.
- RPCs were already queued on the subchannel and a connection was lost
  or a connection attempt fails.
- RPCs were already queued on the subchannel and a new connection was just
  created that did not provide enough available streams for all pending RPCs.
- The value of the max_connections_per_subchannel attribute increases,
  all existing connections are already at their MAX_CONCURRENT_STREAMS
  limit, and there are queued RPCs.

#### Backoff Behavior and Connectivity State

The subchannel must have no more than one connection
attempt in progress at any given time.  The [backoff
state](https://github.com/grpc/grpc/blob/master/doc/connection-backoff.md)
will be used for all connection attempts in the subchannel, regardless
of how many established connections there are.  If a connection attempt
fails, the backoff period must be respected and scale accordingly before
starting the next attempt on any connection.  When a connection attempt
succeeds, backoff state will be reset, just as it is today.

For example, if the subchannel has both two existing connections and a
pending connection attempt for a third connection, if one of the original
connections fails, the subchannel may not start a new connection attempt
immediately, because it already has a connection attempt in progress.
Instead, it must wait for the in-flight connection attempt to finish.
If that attempt fails, then backoff must be performed before starting the
next connection attempt.  But if that attempt succeeds, backoff state
will be reset, so if there are still enough queued RPCs to warrant a
second connection, then the subchannel may immediately start another
connection attempt.

The connectivity state of the subchannel should be determined as follows
(first match wins):
- If there is at least one connection established, report READY.
- If there is at least one connection attempt in flight, report
  CONNECTING.
- If the subchannel is in backoff after a failed connection attempt,
  report TRANSIENT_FAILURE.
- Otherwise, report IDLE.

Note that subchannels may exhibit two new state transitions that have not
previously been possible.  Today, the only possible transition from state
READY is to state IDLE, but with this design, the following additional
transitions are possible:
- If the subchannel has an existing connection and has a connection
  attempt in flight for a second connection, and then the first
  connection fails before the in-flight connection attempt completes,
  then the subchannel will transition from READY to CONNECTING.
- If the subchannel has an existing connection but is in backoff after
  failing to establish a second connection attempt, and then the original
  connection fails, the subchannel will transition from READY directly
  to TRANSIENT_FAILURE.

See [PickFirst Changes](#pickfirst-changes) below for details on how
pick_first will handle these new transitions.

TODO: update client channel spec with this info

#### Picking a Connection for Each RPC

When an RPC is started and a subchannel is picked for that RPC, the
subchannel will find the first available connection for the RPC, in
order of connection creation.

The subchannel must ensure that races do not happen while dispatching
RPCs to a connection that will lead to one or more RPCs being queued in
the connection despite having available quota elsewhere.  For example,
if two RPCs are initiated at the same time and one stream is available
in a connection, both RPCs must not choose the same connection, or else
one will queue.  This race can be avoided with locks or atomics.

One race that may lead to RPCs being queued in a connection is if the
MAX_CONCURRENT_STREAMS setting of a connection is lowered by the server
after RPCs are dispatched to the connection.  This race can be avoided
if the connection is modified to not queue RPCs but instead report the
scenario back to the subchannel, or to coordinate the SETTINGS frame ACK
with the subchannel.  Such changes are out of scope for this design,
but may be considered in the future.  For the purposes of this design,
it is acceptable to queue RPCs on a connection due to this race, which
is expected to be rare.

When choosing a connection for an RPC within a subchannel, the following
algorithm will be used (first match wins):
1. If the subchannel has no working connections, then the RPC will be
   failed with status UNAVAILABLE.
2. Look through all working connections in order from the oldest to the
   newest.  For each connection, if the number of RPCs in flight on the
   connection is lower than the peer's MAX_CONCURRENT_STREAMS setting,
   then the RPC will be dispatched on that connection.
3. If the number of existing connections is equal to the
   max_connections_per_subchannel value, the RPC will be queued.
4. If the number of existing connections is less than the
   max_connections_per_subchannel value and the subchannel is in backoff
   delay due to the last connection attempt failing, the RPC will be
   queued.
5. If the number of existing connections is less than the
   max_connections_per_subchannel value and no connection attempt is
   currently in flight, a new connection attempt will be started, and
   the RPC will be queued.

When queueing an RPC, the queue must be roughly fair: RPCs must
be dispatched in the order in which they are received into the
queue, acknowledging that timing between threads may lead to
concurrent RPCs being added to the queue in an arbitrary order.
See [Rationale](#rationale) below for a possible adjustment to this
queuing strategy.

When retrying a queued RPC, the subchannel will use the same algorithm
described above that it will when it first sees the RPC.  RPCs will be
drained from the queue upon the following events:
- When a connection attempt completes, whether successfully or not.
- When the backoff timer fires.
- When an existing connection fails.
- When the transport for a connection reports a new value for
  MAX_CONCURRENT_STREAMS.
- When an RPC dispatched on one of the connections completes.

When failing an RPC due to the subchannel not having any established
connections, note that the RPC will be eligible for transparent retries
(see [A6]), because no wire traffic was produced for it.

#### Interaction Between Channel and Subchannel

Today, when the channel does an LB pick and gets back a subchannel,
it calls a method on that subchannel to get its underlying connection.
There are only two possible results:

1. The subchannel returns a ref to the underlying connection.
2. The subchannel returns null to indicate that it no longer has a working
   connection.  This case can happen due to a race between the LB policy
   picking the subchannel and the transport seeing a GOAWAY or disconnection;
   when that occurs, the channel will queue the RPC (i.e., it is treated
   the same as if the picker indicated to queue the RPC) in the expectation
   that the LB policy will soon see the subchannel report the disconnection
   and will return a new picker, at which point a new pick will be done for
   the queued RPC.

With this design, the API used for this interaction between the channel
and the subchannel will need to change.  Instead of the channel getting
a connection from the subchannel, the channel will simply send the RPC
to the subchannel, and the subchannel will be responsible for picking a
connection or queuing the RPC if there is no connection immediately
available.

One consequence of this is that when we hit the race condition between
the LB policy picking the subchannel and the transport seeing a GOWAWAY
or disconnection, we will now rely on transparent retries (see [A6])
instead of just having the channel re-queue the LB pick to be tried
again the next time the LB policy updates the picker.

#### Interaction with xDS Circuit Breaking

gRPC currently supports xDS circuit breaking as described in [A32].
Specifically, we support configuring the max number of RPCs in flight
to each cluster.  This is done by having the xds_cluster_impl LB policy
increment an atomic counter that tracks the number of RPCs currently in
flight to the cluster, which is later decremented when the RPC ends.

Some gRPC implementations don't actually increment the counter until
after a connection is chosen for the RPC, so that they won't erroneously
increment the counter when the picker returns a subchannel for which no
connection is available (the race condition described above).  However,
now that there could be a longer delay between when the picker returns
and when a connection is chosen for the RPC, implementations will need
to increment the counter in the picker.  It will then be necessary to
decrement the counter when the RPC finishes, regardless of whether it
was actually sent out or whether the subchannel failed it.

#### Subchannel Pseudo-Code

The following pseudo-code illustrates the expected functionality in the
subchannel:

```
# Returns True if the RPC has been handled, False if it needs to be queued.
def MaybeDispatchRpc(self, rpc):
  # If there are no connections, fail the RPC.
  # Note that the RPC will be eligible for transparent retries.
  if len(self.connections) == 0:
    rpc.Fail(UNAVAILABLE)
    return True
  # Use the oldest connection that can accept a new stream, if any.
  for connection in self.connections:
    if connection.rpcs_in_flight < connection.max_concurrent_streams:
      connection.rpcs_in_flight += 1
      connection.StartRpc(rpc)
      return True
  # If we aren't yet at the max number of connections, see if we can
  # create a new one.
  if (len(self.connections) < self.max_connections_per_subchannel and
      self.connection_attempt is None and
      self.pending_backoff_timer is None):
    self.StartConnectionAttempt()
  # Didn't find a connection for the RPC, so queue it.
  return False

# Starts an RPC on the subchannel.
def StartRpc(self, rpc):
  if not self.MaybeDispatchRpc(rpc):
    self.queue.append(rpc)

# Retries RPCs from the queue, in order.
def RetryRpcsFromQueue(self):
  while len(self.queue) > 0:
    # Stop at the first RPC that gets queued.
    if not self.MaybeDispatchRpc(self.queue[-1]):
      break
    self.queue.pop()

# Starts a new connection attempt.
def StartConnectionAttempt(self):
  if self.backoff_state is None:
    self.backoff_state = BackoffState()
  self.connection_attempt = ConnectionAttempt()

# Called when a connection attempt succeeds.
def OnConnectionAttemptSucceeded(self, new_connection):
  self.connections.append(new_connection)
  self.pending_backoff_timer = None
  self.backoff_state = None
  self.RetryRpcsFromQueue()

# Called when a connection attempt fails.  This puts us in backoff.
def OnConnectionAttemptFailed(self):
  self.connection_attempt = None
  self.pending_backoff_timer = Timer(self.backoff_state.NextBackoffDelay())
  self.RetryRpcsFromQueue()

# Called when the backoff timer fires.  Will trigger a new connection
# attempt if there are RPCs in the queue.
def OnBackoffTimer(self):
  self.pending_backoff_timer = None
  self.RetryRpcsFromQueue()

# Called when an established connection fails.  Will trigger a new
# connection attempt if there are RPCs in the queue.
def OnConnectionFailed(self, failed_connection):
  self.connections.remove(failed_connection)
  self.RetryRpcsFromQueue()

# Called when a connection reports a new MAX_CONCURRENT_STREAMS value.
# May send RPCs on the connection if there are any queued and the
# MAX_CONCURRENT_STREAMS value has increased.
def OnConnectionReportsNewMaxConcurrentStreams(
    self, connection, max_concurrent_streams):
  connection.max_concurrent_streams = max_concurrent_streams
  self.RetryRpcsFromQueue()

# Called when an RPC completes on one of the subchannel's connections.
def OnRpcComplete(self, connection):
  connection.rpcs_in_flight -= 1
  self.RetryRpcsFromQueue()

# Called when the max_connections_per_subchannel value changes.
def OnMaxConnectionsPerSubchannelChanged(self, max_connections_per_subchannel):
  self.max_connections_per_subchannel = max_connections_per_subchannel
  self.RetryRpcsFromQueue()
```

### PickFirst Changes

As mentioned above, the pick_first LB policy will need to handle two new
state transitions from subchannels.  Previously, a subchannel in READY
state could transition only to IDLE state; now it will be possible for
the subchannel to instead transition to CONNECTING or TRANSIENT_FAILURE
states.

If the selected (READY) subchannel transitions to CONNECTING or
TRANSIENT_FAILURE state, then pick_first will go back into CONNECTING
state.  It will start the happy eyeballs pass across all subchannels,
as described in [A61].

### Metrics

TODO: define metrics

### Temporary environment variable protection

Enabling this feature via either the gRPC service config or xDS will
initially be guarded via the environment variable
`GRPC_EXPERIMENTAL_MAX_CONCURRENT_STREAMS_CONNECTION_SCALING`.  The
feature will be enabled by default once it has passed interop tests.

TODO: define interop tests?

## Rationale

We considered several different approaches as part of this design process.

We considered putting the connection scaling functionality in an LB
policy instead of in the subchannel, but we rejected that approach for
several reasons:
- The LB policy API is designed around the idea that returning a new
  picker will cause queued RPCs to be replayed, which works fine when an
  updated picker is likely to allow most of the queued RPCs to proceed.
  However, in a case where the workload is hitting the
  MAX_CONCURRENT_STREAMS limit across all connections, we would wind up
  updating the picker as each RPC finishes, which would allow only one
  of the queued RPCs to continue, thus exhibiting O(n^2) behavior.  We
  concluded that the LB policy API is simply not well-suited to solve
  this particular problem.
- There were some concerns about possibly exposing the
  MAX_CONCURRENT_STREAMS setting from the transport to the LB policy
  API.  However, this particular concern could have been ameliorated
  by initially using an internal or experimental API.
- Putting this functionality in the LB policy would have added
  complexity due to subchannels being shared between channels in C-core,
  where each channel can see only the RPCs that that channel is sending
  to the subchannel.

Given that we decided to add this new functionality in the subchannel,
we intentionally elected to keep it simple, at least to start with.
We considered a design incorporating a more sophisticated -- or pluggable
-- connection selection method but rejected it due to the complexity
of such a mechanism and the desire to avoid creating a duplicate load
balancing ecosystem within the subchannel.

Potential future improvements that we could consider:
- Additional configuration knobs:
  - min_connections_per_subchannel to force clients to create a minimum
    number of connections, even if they are not necessary.
  - min_available_streams_per_subchannel to allow clients to create new
    connections before the hard MAX_CONCURRENT_STREAMS setting is reached.
- Channel arg to limit streams per connection lower than
  MAX_CONCURRENT_STREAMS.
- Instead of queuing RPCs in the subchannel, it may be possible to improve
  aggregate performance by failing the RPC, resulting in transparent retry
  and a re-pick.  Without other systems in place, this would lead to a
  busy-loop if the same subchannel is picked repeatedly, so this is not
  included in this design.

This design does not handle connection affinity; there is no way to
ensure related RPCs end up on the same connection without setting
max_connections_per_subchannel to 1.  For use cases where connection
affinity is important, multiple channels will continue to be necessary
for now.

This design also does not attempt to maximize throughput of connections,
which would be a far more complex problem.  To maximize throughput
effectively, more information about the nature of RPCs would need to
be exposed; e.g., how much bandwidth they may require and how long they
might be expected to last.  And unlike this connection scaling design,
that functionality might actually want to be implemented in the LB policy
layer, since it would be totally compatible with the LB policy API.
The goal of *this* design is to simply overcome the stream limits on
connections, hence the simple and greedy connection selection mechanism.

## Implementation

Will be implemented in C-core, Java, and Go.
