Dynamic Connection Scaling Based On Stream Availability
----
* Author(s): dfawley
* Approvers: roth, ejona
* Implemented in: none
* Last updated: 2021-11-22
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

A feature to automatically create new connections when the limit of streams on a
connection is reached.

## Background

HTTP/2 contains a connection-level setting,
[`SETTINGS_MAX_CONCURRENT_STREAMS`](https://httpwg.org/specs/rfc7540.html#SETTINGS_MAX_CONCURRENT_STREAMS),
to limit the number of streams a peer may initiate.  By default, gRPC HTTP/2
servers do not set this (allowing unlimited streams), but it can be lowered to
protect against a single client using too many resources.  In an environment
with reverse proxies or virtual IPs, it may be possible to create multiple
connections to the same IP address that lead to different physical servers, and
this can be a feasible way for a client to achieve more throughput (QPS) without
overloading any server.  In addition, TLS and HTTP/2 processing is serial, so
even for a single IP to a single server, using multiple connections can achieve
more throughput by utilizing more CPU cores.

## Proposal

### Connection Management

A feature will be added at the subchannel level to provide the ability to scale
up connections to the subchannel's address when the stream limit is reached.
The maximum number of connections per subchannel will be controlled by a new
service config field named `maxConnectionsPerSubchannel`.  This defaults to 1,
resulting in no behavior change if it is not explicitly set.  In addition, when
the value is 1, implementations should not incur any additional overhead
relative to the current behavior (e.g. no additional synchronization or memory
usage).

Note that in Java and Go, subchannels may contain multiple addresses.  Once an
address is connected, any redundant connections will be created to that same
address only.

Note also that the behavior of creating multiple connections to a single
address, with pick-first, will make the channel less likely to later use another
address.  Normally, pick-first would start back at the first address in its list
when any connection is lost, but with multiple connections to the same address,
it is less likely that all connections will be lost together, which is the only
condition under which the channel would restart at the beginning of the list.

#### Adding Connections

To determine when to create a new connection, the subchannel will monitor the
`MAX_CONCURRENT_STREAMS` and the number of active streams on each connection.  A
new connection will be created when all of the following occur:

- One or more RPCs are pending in the subchannel.
- No existing connection in the subchannel has any available streams.
- The number of current connections in the subchannel is fewer than the
  `maxConnectionsPerSubchannel` setting in the channel's service config.
- No other connections are currently being created by the subchannel.

If a connection attempt fails, the backoff period must be respected and scale
accordingly before attempting a new connection.

Note that any of the above conditions may trigger the creation of a connection.
Examples:

- A new RPC is dispatched and all connections are saturated.
- RPCs were already pending and a connection was lost or a connection attempt
  fails.
- RPCs were already pending and a new connection was just created that did not
  provide enough available streams for all pending RPCs.

#### Removing Connections

When connections are lost for any reason, they are removed from the subchannel.
If no `READY` connections remain and no connection attempts are in progress, the
subchannel will enter `IDLE`.  If no `READY` connections remain and a connection
attempt is in progress, the subchannel will enter `CONNECTING` instead.  In
either case, any pending RPCs will result in an error with status code
`UNAVAILABLE`.  These RPCs would be eligible for transparent retries since no
wire traffic was produced for them.  Note that connections are never closed
automatically by the client.  Servers should use
[`MAX_CONNECTION_IDLE`](https://github.com/grpc/proposal/blob/master/A9-server-side-conn-mgt.md)
to make sure clients do not keep more open connections than necessary.

### Connection Selection

When an RPC is started and a subchannel is picked for that RPC, the subchannel
will find the first available connection for it, in order of connection
creation.

The subchannel must ensure that races do not happen while dispatching RPCs to a
connection that will lead to one or more RPCs being queued in the connection
despite having available quota elsewhere.  For example, if two RPCs are
initiated at the same time and one stream is available in a connection, both
RPCs must not choose the same connection, or else one will queue.  This race can
be avoided with locks or atomics.

One race that may lead to RPCs being queued in a connection is if the
`MAX_CONCURRENT_STREAMS` setting of a connection is lowered by the server after
RPCs are dispatched to the connection.  This race can be avoided if the
connection is modified to not queue RPCs, but report the scenario back to the
subchannel instead, or to coordinate the `SETTINGS` frame ACK with the
subchannel.  Such changes are out of scope for this design, but may be
considered in the future.  Until this is designed and implemented, it is
acceptable to queue RPCs on a connection due to this race, which is expected to
be rare.

If no connection is available for an RPC, the RPC must be queued in the
subchannel until a connection is available for it.  This queue must be roughly
fair: RPCs must be dispatched in the order in which they are received into the
queue, acknowledging that timing between threads may lead to concurrent RPCs
being added to the queue in an arbitrary order.  See "Potential Future Work" for
a possible adjustment to this queuing strategy.

### Settings

#### Service Config Settings

The top-level service config of gRPC will be modified for this feature as follows:

```
{
	// ...
	// existing fields, e.g. "methodConfig" and "loadBalancingConfig"
	// ...

	// Settings to control dynamic connection scaling.  For more details,
    // please refer to gRFC AXX (this one; XX = TBD).
	"connectionScaling": {
		// Maximum connections gRPC will maintain for each subchannel in
        // this channel.  When no streams are available for an RPC in a
        // subchannel, gRPC will automatically create new connections up
        // to this limit.  If this value changes during the life of a
        // channel, existing subchannels will be updated to reflect
        // the change.  No connections will be closed as a result of
        // lowering this value.  Down-scaling will only happen as
        // connections are lost naturally.
        //
        // Must be a whole number greater than 1.  Values higher than
        // the configured limit (by default, 10) will be clamped.
		"maxConnectionsPerSubchannel": number
	}
}
```

#### Channel Settings

The `maxConnectionsPerSubchannel` setting from service config has a natural
limit of 10.  A channel setting should be provided to allow a client application
to raise or lower this limit.

### Temporary environment variable protection

Until sufficient testing is performed, this feature will initially be disabled
by default, and only enabled if `GRPC_EXPERIMENTAL_CONNECTION_SCALING=true`.

### Subchannel Sharing

Subchannels may be shared across channels.  The `maxConnectionsPerSubchannel`
setting will be applied to a shared subchannel and that subchannel will use the
highest maximum of all channels currently sharing it.  If channels sharing the
subchannel are closed, then the remaining channels' settings will be combined to
determine the new max.  If the max is lowered to less than the number of open
connections, the excess connections will not be closed as a result.
Down-scaling will only happen as connections are lost naturally.

## Rationale

Other approaches were considered as part of the design process.  A design
incorporating a more sophisticated -- or pluggable -- connection selection
method was rejected due to the complexity of such a mechanism.  A design to
expose the `MAX_CONCURRENT_STREAMS` to LB policies was similarly rejected,
because utilizing this properly would require rewriting any existing LB policies
and complexities related to synchronizing `MAX_CONCURRENT_STREAMS` updates.

### Potential Future Work

- Additional service config settings:
  - `minConnectionsPerSubchannel` to force clients to create a minimum number of
    connections, even if they are not necessary.
  - `minAvailableStreamsPerSubchannel` to allow clients to create new connections
    before the hard `MAX_CONCURRENT_STREAMS` setting is reached.
- Channel arg to limit streams per connection lower than
  `MAX_CONCURRENT_STREAMS`.
- Instead of queuing RPCs in the subchannel, it may be possible to improve
  aggregate performance by failing the RPC, resulting in transparent retry and a
  re-pick.  Without other systems in place, this would lead to a busy-loop if
  the same subchannel is picked repeatedly, so this is not included in this
  design.

### Limitations

This design does not handle connection affinity; there is no way to ensure
related RPCs end up on the same connection without setting
`maxConnectionsPerSubchannel` to 1.  For use cases where affinity is important,
multiple channels should be utilized instead.

This design also does not attempt to maximize throughput of connections, which
would be a far more complex problem.  To maximize throughput effectively, more
information about the nature of RPCs would need to be exposed, e.g. how much
bandwidth they may require and how long they might be expected to last.  The
goal of this design is to simply overcome the stream limits on connections,
hence the simple and greedy connection selection mechanism.

## Implementation

TBD
