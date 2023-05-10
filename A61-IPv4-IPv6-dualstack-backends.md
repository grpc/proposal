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
* [Support for dual stack EDS endpoints in Envoy][envoy-design]
* [gRFC A17: Client-Side Health Checking][A17]
* [gRFC A58: Weighted Round Robin LB Policy][A58]
* [gRFC A48: xDS Least Request LB Policy][A48]
* [gRFC A42: Ring Hash LB Policy][A42]
* [gRFC A55: xDS-Based Stateful Session Affinity][A55]
* [gRFC A62: pick_first: Sticky TRANSIENT_FAILURE and address order
  randomization][A62]
* [gRFC A51: Custom Backend Metrics][A51]

## Proposal

This proposal includes several parts:
- Allow resolvers to return multiple addresses per endpoint.
- Implement Happy Eyeballs.  This will be done in the pick_first LB policy,
  which will become the universal leaf policy.  It will also need to
  support client-side health checking.  In Java and Go, the pick_first
  logic will be moved out of the subchannel and into the pick_first
  policy itself.
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
instead use the Happy Eyeballs algorithm on the initial pass through the
address list.  Specifically:

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
- The first time any connection attempt succeeds (i.e., the subchannel
  reports READY, which happens after all handshakes are complete),
  we choose that connection.  If there is a timer running, we cancel
  the timer.  We also cancel all other connection attempts that are
  still in flight (but see notes about C-core below).
- We will wait for at least one connection attempt on every address to
  fail before we consider the first pass to be complete.  As per [gRFC
  A62][A62], we will report TRANSIENT_FAILURE state and will continue
  trying to connect.  We will stay in TRANSIENT_FAILURE until either (a)
  we become connected or (b) the LB policy is destroyed by the channel
  shutting down or going IDLE.

If the first pass completes without a successful connection attempt, we
will switch to a mode where we keep trying to connect to all addresses at
all times, with no regard for the order of the addresses.  Each
individual subchannel will provide [backoff behavior][backoff-spec],
reporting TRANSIENT_FAILURE while in backoff and then IDLE when backoff
has finished.  The pick_first policy will therefore automatically
request a connection whenever a subchannel reports IDLE.

In C-core, there are some additional details to handle due to the
existance of subchannel sharing between channels.  Any given subchannel
that pick_first is using may also be used other channel(s), and any
of those other channels may request a connection on the subchannel
at any time.  This means that pick_first needs to be prepared for the
fact that any subchannel may report any connectivity state at any time
(even at the moment that pick_first starts using the subchannel), even
if it did not previously request a connection on the subchannel itself.
This has a number of implications:

- pick_first needs to be prepared for any subchannel to report READY at
  any time, even if it did not previously request a connection on that
  subchannel.  Currently (prior to this design), pick_first will immediately
  choose the first subchannel that reports READY.  That behavior seems
  consistent with the intent of Happy Eyeballs, so we will preserve it.
- A subchannel must be in state IDLE in order for pick_first to request
  a connection attempt on it.  However, during the first pass through
  the address list, when pick_first decides to start a connection attempt
  on a given subchannel (whether because it is the first subchannel in
  the list or because the timer fired before the previous address'
  connection attempt completed), that subchannel may not be in state IDLE.
  - If the subchannel is in state CONNECTING, we do not need to actually
    request a connection, but we will treat it as if we did.
    Specifically, we will start the timer and wait to see if the connection
    attempt completes within the Connection Attempt Delay before moving
    on to the next subchannel.
  - If the subchannel is in state TRANSIENT_FAILURE, then we know that
    it is in backoff due to a recent connection attempt failure, so we
    treat it as if we have already made a connection attempt on this
    subchannel, and we will immediately move on to the next subchannel.
- When we choose a subchannel that has become successfully connected,
  we will unref all of the other subchannels.  For any subchannel on
  which we were the only channel holding a ref, this will cause any
  pending connection attempt to be cancelled, and the subchannel will
  be destroyed.  However, if some other channel was holding a ref to the
  subchannel, the connection attempt will continue, even if the other
  channel did not want it.  This is slightly sub-optimal, but it does
  not seem likely to be problematic enough to warrant the complexity of
  fixing it.

#### Move pick_first Logic Out of Subchannel (Java/Go)

In Java and Go, the pick_first logic is currently implemented in the
subchannel.  We will pull this logic out of the subchannel and move it
into the pick_first policy itself.  This means that subchannels will
have only one address, and that address does not change over the
lifetime of the subchannel.  It will also mean that connection backoff
will be done on a per-address basis rather than a per-endpoint basis.
This will move us closer to having uniform architecture across all of
our implementations.

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
single endpoint with all of its addresses.  (See below for more details
on individual petiole policies.)

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

#### Address List Handling in pick_first

As mentioned above, we are changing the LB policy API to take an address
list that contains a list of endpoints, each of which can contain one
or more addresses.  However, the Happy Eyeballs algorithm assumes a flat
list of addresses, not this two-dimensional list.  To address that, we
need to define how pick_first will flatten the list.  We also need to
define how that flattening interacts with both the sorting described in
[RFC-8304 section 4](https://www.rfc-editor.org/rfc/rfc8305#section-4)
and with the optional shuffling described in [gRFC A62][A62].

There are three cases to consider here:

A. If pick_first is used under a petiole policy, it will see a single
   endpoint with one or more addresses.

B. If pick_first is used as the top-level policy in the channel with the
   DNS resolver, it will see one or more endpoints, each of which have
   exactly one address.  It should be noted that the DNS resolver does
   not actually know which addresses might or might not be associated
   with the same endpoint, so it assumes that each address is a separate
   endpoint.

C. If pick_first is used as the top-level policy in the channel with a
   custom resolver implementation, it may see more than one endpoint,
   each of which has one or more addresses.

[RFC-8304 section 4](https://www.rfc-editor.org/rfc/rfc8305#section-4)
says to perform RFC-6724 sorting first.  In gRPC, that sorting happens
in the DNS resolver before the address list is passed to the LB policy,
so it will already be done by the time pick_first sees the address list.

When the pick_first policy sees an address list, it will perform these
steps in the following order:

1. Perform the optional shuffling described in [gRFC A62][A62].  The
   shuffling will change the order of the endpoints but will not touch
   the order of the addresses within each endpoint.  This means that the
   shuffling will work for cases B and C above, but it will not work for
   case A; this is expected to be the right behavior, because we do not
   have or anticipate any use cases where a petiole policy will need to
   enable shuffling.

2. Flatten the list by concatenating the ordered list of addresses for
   each of the endpoints, in order.

3. In the flattened list, interleave addresses from the two address
   families, as per [RFC-8304 section
   4](https://www.rfc-editor.org/rfc/rfc8305#section-4).  Doing this on
   the flattened address list ensures the best behavior if only one of
   the two address families is working.

#### Client-Side Health Checking in pick_first

Note that, as described in [gRFC
A17](A17-client-side-health-checking.md#pick_first), pick_first does not
support client-side health checking, but the petiole policies do.  To
support this, it will be necessary for pick_first to optionally support
client-side health checking so that it can be enabled when pick_first is
used as a child policy underneath a petiole policy.

When client-side health checking is enabled in pick_first, it will be
necessary for pick_first to see both the "raw" connectivity state of
each subchannel and the state reflected by health checking.  The
connection management behavior will continue to use the "raw" connectivity
state, just as it does today.  Only once pick_first chooses a subchannel
will it start health checking, and the connectivity state reported by
the health checking code is the state that pick_first will report to its
parent.

Note that, as described in gRFC A17, we still do not want users enabling
client-side health checking when they directly use pick_first as the LB
policy, because there is no sensible behavior for pick_first when the
chosen subchannel reports unhealthy.  As a result, we will implement
this functionality in a way that it can be triggered by a parent policy
such as round_robin but cannot be triggered by an external application.
(For example, in C-core, this will be triggered via an internal-only
channel arg that will be set by the petiole policies.)

#### Address List Updates in Petiole Policies

The algorithm used by petiole policies to handle address list updates
will need to be updated to reflect the new two-level nature of address
lists.

Currently, there are differences between C-core and Java/Go in terms of
how address list works are handled, so we need to specify how each
approach works and how it is going to be changed.

##### Address List Updates in C-core

In C-core, the channel provides a subchannel pool, which means that if
an LB policy creates multiple subchannels with the same address and
channel args, both of the returned subchannel objects will actually be
refs to the same underlying real subchannel.

As a result, the normal way to handle an address list update today is to
create a whole new list of subchannels, ignoring the fact that some of
them may be duplicates of subchannels in the previous list; for those
duplicates, the new list will just wind up getting a new ref to the
existing subchannel, so there will not be any connection churn.  Also, to
avoid adding unnecessary latency to RPCs being sent on the channel, we
wait to actually start using the new list until we have seen the initial
connectivity state update on all of those subchannels and they have been
given the chance to get connected, if necessary.

With the changes described in this proposal, we will continue to take
the same basic approach, except that for each endpoint, we will create a
pick_first child policy instead of creating a subchannel.  Note that the
subchannel pool will still be used by all pick_first child policies, so
creating a new pick_first child in the new list for the same address that
is already in use by a pick_first child in the old list will wind up
reusing the existing connection.

##### Address List Updates in Java/Go

In Java and Go, there is no subchannel pool, so when an LB policy gets
an updated address list, it needs to explicitly check whether any of
those addresses were already present on its previous list.  It
effectively does a set comparison: for any address on the new list that
is not on the old list, it will create a new subchannel; for any address
that was on the old list but is not on the new list, it will remove the
subchannel; and for any address on both lists, it will retain the
existing subchannel.

This algorithm will continue to be used, with the difference that each
entry in the list will now be a set of one or more addresses rather than
a single address.  Note that the order of the addresses will not matter
when determining whether an endpoint is present on the list; if the old
list had an endpoint with address list `[A, B]` and the new list has an
endpoint with address list `[B, A]`, that endpoint will be considered to
be present on both lists.  However, because the order of the addresses
will matter to the pick_first child when establishing a new connection,
the petiole policy will need to send an updated address list to the
pick_first child to ensure that it has the updated order.

Note that in this algorithm, the unordered set of addresses must be the
same on both the old and new list for an endpoint to be considered the
same.  This means that if an address is added or removed from an
existing endpoint, it will be considered a completely new endpoint,
which may cause some unnecessary connection churn.  For this design, we
are accepting this limitation, but we may consider optimizing this in
the future if it becomes a problem.

#### Weighted Round Robin

In the `weighted_round_robin` policy described in [gRFC A58][A58], some
additional state is needed to track the weight of each endpoint.

##### WRR in C-core

In C-core, WRR currently has a map of address weights, keyed by the
associated address.  The weight objects are ref-counted and remove
themselves from the map when their ref-count reaches zero.  When a
subchannel is created for a given address, it takes a new ref to the
weight object for its address.  This structure allows the weight
information to be retained when we create a new subchannel list in
response to an updated address list.

With the changes in this proposal, this map will instead be keyed by the
unordered set of addresses for each endpoint.  This will use the same
semantics as address list updates in Java/Go, described above: an
endpoint on the old list with addresses `[A, B]` will be considered
identical to an endpoint on the new list with addresses `[B, A]`.

Note that in order to start the ORCA OOB watcher for backend metrics
on the subchannel (see [gRFC A51][A51]), WRR will need to intercept
subchannel creation via the helper that it passes down into the pick_first
policy.  It will unconditionally start the watch for each subchannel
as it is created, all of which will update the same subchannel weight.
However, once pick_first chooses a subchannel, it will unref the other
subchannels, so only one OOB watcher will remain in steady state.

##### WRR in Java/Go

In Java and Go, WRR stores the subchannel weight in the individual
subchannel.  We will continue to use this same structure, except that
instead of using a map from a single address to a subchannel, we will
store a map from an unordered set of addresses to a pick_first child,
and the endpoint weight will be stored alongside that pick_first child.

Just like in C-core, in order to start the ORCA OOB watcher for backend
metrics on the subchannel, WRR will need to intercept subchannel creation
via the helper that it passes down into the pick_first policy.  However,
unlike C-core, Java and Go will need to wrap the subchannels and store
them, so that they can start or stop the ORCA OOB watcher as needed by a
subsequent config change.

#### Least Request

The least-request LB policy will (Java-only) will work essentially the
same way as WRR.  The only difference is that the data it is storing on
a per-endpoint basis is outstanding request counts rather than weights.

#### Outlier Detection

TODO: details (probably continue to set map based on endpoints in the
address list on the way down)

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
backoff spec][backoff-spec] seems to have originally envisioned.

## Implementation

### C-core

- move client-side health checking out of subchannel so that it can be
  controlled by pick_first (https://github.com/grpc/grpc/pull/32709)
- assume LB policies start in CONNECTING state
  (https://github.com/grpc/grpc/pull/33009)
- make pick_first the universal leaf policy, including client-side
  health checking support (https://github.com/grpc/grpc/pull/32692)
- change address list to support multiple addresses per endpoint and
  change LB policies to handle this (including ring_hash)
- change stateful session affinity to handle multiple addresses per endpoint
- implement happy eyeballs in pick_first
- support new xDS fields

### Java

- change grpclb to delegate to RR or PF
- move pick_first logic out of subchannel and into pick_first policy
- make pick_first the universal leaf policy, including client-side
  health checking support
- implement happy eyeballs in pick_first
- fix ring_hash to support endpoints with multiple addresses
- support new xDS fields

### Go

- change subchannel connectivity state API (maybe)
- change grpclb to delegate to RR or PF
- move pick_first logic out of subchannel and into pick_first policy
- make pick_first the universal leaf policy, including client-side
  health checking support (includes moving health checking logic out of
  the subchannel)
- change address list to support multiple addresses per endpoint and
  change LB policies to handle this (including ring_hash)
- implement happy eyeballs in pick_first
- support new xDS fields

## Open issues (if applicable)

N/A

[envoy-design]: https://docs.google.com/document/d/1AjmTcMWwb7nia4rAgqE-iqIbSbfiXCI4h1vk-FONFdM/edit
[A17]: A17-client-side-health-checking.md
[A42]: A42-xds-ring-hash-lb-policy.md
[A48]: A48-xds-least-request-lb-policy.md
[A51]: A51-custom-backend-metrics.md
[A55]: A55-xds-stateful-session-affinity.md
[A58]: A58-client-side-weighted-round-robin-lb-policy.md
[RFC-8305]: https://www.rfc-editor.org/rfc/rfc8305
[A62]: https://github.com/grpc/proposal/pull/357
[backoff-spec]: https://github.com/grpc/grpc/blob/master/doc/connection-backoff.md
