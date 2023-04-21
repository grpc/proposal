A61: IPv4 and IPv6 Dualstack Backend Support
----
* Author(s): @markdroth
* Approver: @ejona86
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2023-04-20
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

gRPC clients currently support both IPv4 and IPv6.  However, most
implementations do not have support for individual backends that have both
an IPv4 and IPv6 address.  It is desirable to natively support such
backends in a way that correctly interacts with load balancing.

## Background

For background on the interaction between the resolver and LB policy in
the gRPC client channel, see [Load Balancing in
gRPC](https://github.com/grpc/grpc/blob/master/doc/load-balancing.md).

In most gRPC implementations, the resolver returns a flat list of
addresses, where each address is assumed to be a different endpoint, and
the LB policy is expected to balance the load across those endpoints.
The list of addresses can include both IPv4 and IPv6 addresses, but it
has no way to represent the case where two addresses point to the same
endpoint, so the LB policy will treat them as two different endpoints,
sending each one its own share of the load.  However, the actual desired
behavior in this case is for the LB policy to use only one of the
addresses for each endpoint at any given time.  (Note that gRPC Java
already supports this.)

Also, when connecting to an endpoint with multiple addresses,
it is desirable to use the "Happy Eyeballs" algorithm described in
[RFC-8305][RFC-8305] to minimize the time it takes to establish a working
connection by parallelizing connection attempts in a reasonable way.
Currently, all gRPC implementations perform connection attempts in a
completely serial manner in the pick_first LB policy.

This work is being doing in conjunction with an effort to add multiple
addresses per endpoint in xDS.  We will support the new xDS APIs being
added for that effort as well.  Note that this change has implications
for session affinity behavior in xDS.

### Related Proposals: 
* TODO: reference gRFC for [sticky-TF in PF][sticky-TF-in-PF]
* TODO: reference to xDS design or Envoy GH issue
* [gRFC A17: Client-Side Health Checking][A17]
* [gRFC A58: Weighted Round Robin LB Policy][A58]
* [gRFC A48: xDS Least Request LB Policy][A48]
* [gRFC A42: Ring Hash LB Policy][A42]
* [gRFC A55: xDS-Based Stateful Session Affinity][A55]

## Proposal

This proposal includes several parts:
- Allow resolvers to return multiple addresses per endpoint.
- Implement Happy Eyeballs.  This will be done in the pick_first LB policy,
  which will become the universal leaf policy.  It will also need to
  support client-side health checking.
- In xDS, we will support the new fields in EDS to indicate multiple
  addresses per endpoint, and we will extend the session affinity
  mechanisms to support such endpoints.

### Allow Resolvers to Return Multiple Addresses Per Endpoint

Instead of returning a flat list of addresses, the resolver will be able
to return a list of endpoints, each of which can have multiple addresses.

TODO: details of how attributes work

TODO: changes in resolver and LB policy APIs

Because DNS does not have a way to indicate which addresses are
associated with the same endpoint, the DNS resolver will return each
address as a separate endpoint.

### Happy Eyeballs in the pick_first LB Policy

The pick_first LB policy currently attempts to connect to each address
serially, stopping at the first one that succeeds.  We will change it to
instead use the Happy Eyeballs algorithm.  Specifically:

- As per [RFC-8305 section
  5](https://www.rfc-editor.org/rfc/rfc8305#section-5)), the default
  Connection Attempt Delay value is 250ms.  Implementations may provide
  a channel arg to control this value, although they must be between the
  recommended lower bound of 100ms and upper bound of 2s.  Any value
  lower than 100ms should be treated as 100ms; any value higher than 2s
  should be treated as 2s.
- Whenever we start a connection attempt on a given address, if it is not
  the last address in the list, we start a timer for the Connection
  Attempt Delay.
- If the timer fires before the connection attempt completes, we will
  start a connection attempt on the next address in the list.  Note that
  we do not interrupt the previous connection attempt that is still in
  flight; at this point, we will have in-flight connection attempts to
  multiple addresses at once.  Also note that, as per the previous
  bullet, we will once again start a timer if this new address is not
  the last address in the list.
- The first time any connection attempt succeeds, we choose that
  connection.  If there is a timer running, we cancel the timer.  We
  also cancel all other connection attempts that are still in flight
  (but see notes about C-core below).
- We will wait for at least one connection attempt on every address to
  fail before we report TRANSIENT_FAILURE.  As per [sticky-TF in
  PF][sticky-TF-in-PF], we will wrap back around to the start of the
  list and begin again with the first address, staying in state
  TRANSIENT_FAILURE until either (a) we become connected or (b) the LB
  policy is destroyed by the channel going IDLE.

In C-core, there are some additional details to handle due to the
existance of subchannel sharing between channels.  Any given subchannel
that pick_first is using may also be used other channel(s), and any
of those other channels may request a connection on the subchannel
at any time.  This means that pick_first needs to be prepared for the
fact that any subchannel may report any connectivity state at any time
(even at the moment that pick_first starts using it), even if it did
not previously request a connection on the subchannel itself.  This has
a number of implications:

- pick_first needs to be prepared for any subchannel to report READY at
  any time, even if it did not previously request a connection on that
  subchannel.  Currently (prior to this design), pick_first will immediately
  choose the first subchannel that reports READY.  That behavior seems
  consistent with the intent of Happy Eyeballs, so we will preserve it.
- A subchannel must be in state IDLE in order for pick_first to request
  a connection attempt on it.  However, when pick_first decides to start
  a connection attempt on a given subchannel (whether because it is the
  first subchannel in the list or because the timer fired before the
  previous address' connection attempt completed), that subchannel
  may not be in state IDLE.
  - If the subchannel is in state CONNECTING, we do not need to actually
    request a connection, but we will treat it as if we did.
    Specifically, we will start the timer and wait to see if the connection
    attempt completes within the Connection Attempt Delay before moving
    on to the next subchannel.
  - If the subchannel is in state TRANSIENT_FAILURE, then we know that
    it is in backoff due to a recent connection attempt failure, so we
    cannot start a new connection attempt on it yet.  In this case, we
    will skip this subchannel for now and immediately move on to the next
    subchannel.  However, we will not consider a full pass complete...

    --> TODO: maybe just say that we do Happy Eyeballs on the first pass
        only, and after that we automatically trigger a new connection
        attempt on each subchannel as soon as it reports IDLE?

- When we choose a subchannel, we will unref all of the other subchannels.
  For any subchannel on which we were the only channel holding a ref,
  this will cause any pending connection attempt to be cancelled, and
  the subchannel will be destroyed.  However, if some other channel was
  holding a ref to the subchannel, the connection attempt will continue,
  even if the other channel did not want it.  This is slightly
  sub-optimal, but it does not seem likely to be problematic enough to
  warrant the complexity of fixing it.

#### Use pick_first as the Universal Leaf Policy

There are two main types of LB policies in gRPC: leaf policies, which
directly interact with subchannels, and parent policies, which delegate
to other LB policies.  Happy Eyeballs support is necessary only in leaf
policies.

Because we do not want to implement Happy Eyeballs multiple times, we
will implement it only in pick_first, and we will change all other leaf
policies to delegate to pick_first instead of directly interacting with
subchannels.  This set of policies, which we will refer to as
"[petiole](https://en.wikipedia.org/wiki/Petiole_(botany))" policies,
includes the following:
- round_robin (see [gRPC Load Balancing](https://github.com/grpc/grpc/blob/master/doc/load-balancing.md#round_robin))
- weighted_round_robin (see [gRFC A58][A58])
- ring_hash (see [gRFC A42][A42])
- least_request (see [gRFC A48][A48] -- currently Java-only)

The petiole policies will receive a list of endpoints, each of which
may contain multiple addresses.  They will create a pick_first child
policy for each endpoint, to which they will pass a list containing a
single endpoint with all of its addresses.

The pick_first policy may be used either under a petiole policy or as
the top-level policy in the channel.  When used under a petiole policy,
it will expect to see a single endpoint with one or more addresses; when
used as the top-level policy in the channel, it may see more than one
endpoint, each of which has one or more addresses.  To address both
cases, pick_first will simply flatten the list it receives into a single
list, which it will use as input to the Happy Eyeballs algorithm.

Note that implementations should be careful to ensure that this
change does not make error messages less useful when a pick fails.
For example, today, when round_robin has all of its subchannels in state
TRANSIENT_FAILURE, it can return a picker that fails RPCs with the error
message reported by one of the subchannels (e.g., "failed to connect
to all addresses; last error: ipv4:127.0.0.1:443: Failed to connect to
remote host: Connection refused"), which tends to be more useful than
just saying something like "all subchannels failed".  With this change,
round_robin will be delegating to pick_first instead of directly
interacting with subchannels, and the LB policy API in many gRPC
implementations does not have a mechanism to report an error message
along with the connectivity state.  In those implementations, it may be
necessary for round_robin to return a picker that delegates to one of
the pick_first children's pickers, possibly modifying the error message
from the child picker before returning it to the channel.

#### Client-Side Health Checking in pick_first

Note that, as described in [gRFC
A17](A17-client-side-health-checking.md#pick_first), pick_first does not
support client-side health checking, but the other policies described
above do.  To support this, it will be necessary for pick_first to
optionally support client-side health checking so that it can be
enabled when pick_first is used as a child policy underneath a policy
like round_robin.

When client-side health checking is enabled in pick_first, it will be
necessary for pick_first to see both the "raw" connectivity state of
each subchannel and the state reflected by health checking.  The
connectivity state behavior will continue to use the "raw" connectivity
state, just as it does today.  Only once pick_first chooses a subchannel
will it start health checking, and the connectivity state reported by
the health checking code is the state that PF will report to its parent.

Note that, as described in gRFC A17, we still do not want users enabling
client-side health checking when they directly use pick_first as the LB
policy, because there is no sensible behavior for pick_first when the
chosen subchannel reports unhealthy.  As a result, we will implement
this functionality in a way that it can be triggered by a parent policy
such as round_robin but cannot be triggered by an external application.

### Support Multiple Addresses Per Endpoint in xDS

TODO: details

#### New EDS Fields

TODO: details (including health state per address)

#### Changes to Ring Hash LB Policy

TODO: details

#### Changes to Stateful Session Affinity

TODO: details

### Temporary environment variable protection

The code that reads the new EDS fields will be initially guarded by an
environment variable called `GRPC_EXPERIMENTAL_XDS_DUALSTACK_ENDPOINTS`.
This environment variable guard will be removed once this feature has
proven stable.

Note that we will not use this environment variable to guard the Happy
Eyeballs functionality, because that functionality will be on by
default, not something that is enabled via external input.

## Rationale

Note that we will not support all parts of "Happy Eyeballs" as described
in [RFC-8305][RFC-8305].  For example, because our resolver API does
not provide a way to return some addresses without others, we will not
start trying to connect before all of the DNS queries have returned.

In Java and Go, pick_first is currently implemented inside the subchannel
rather than at the LB policy layer.  In those implementations, it
might work to implement Happy Eyeballs inside the subchannel, which
would avoid the need to make pick_first the universal leaf policy,
and in Go, it would avoid the need to move the health-checking code
out of the subchannel.  However, that approach won't work for C-core,
and we would like to take this opportunity to move toward a more
uniform cross-language architecture.  Also, moving pick_first up
to the LB policy layer in Java and Go will have the nice effect of
making their backoff work per-address instead of across all addresses,
which is what C-core does and what the (poorly specified) [connection
backoff](https://github.com/grpc/grpc/blob/master/doc/connection-backoff.md)
spec seems to have originally envisioned.

## Implementation

### C-core

- move client-side health checking out of subchannel so that it can be
  controlled by pick_first (https://github.com/grpc/grpc/pull/32709)
- make pick_first the universal leaf policy, including client-side
  health checking support (https://github.com/grpc/grpc/pull/32692)
- change address list to support multiple addresses per endpoint and
  change LB policies to handle this (including ring_hash)
- implement happy eyeballs in pick_first
- support new xDS fields, and change stateful session affinity to handle
  multiple addresses per endpoint

### Java

TODO(ejona): Fill this in.

### Go

TODO(dfawley): Fill this in.

## Open issues (if applicable)

N/A

[A17]: A17-client-side-health-checking.md
[A42]: A42-xds-ring-hash-lb-policy.md
[A48]: A48-xds-least-request-lb-policy.md
[A55]: A55-xds-stateful-session-affinity.md
[A58]: A58-client-side-weighted-round-robin-lb-policy.md
[RFC-8305]: https://www.rfc-editor.org/rfc/rfc8305
[sticky-TF-in-PF]: (placeholder)
