A61: IPv4 and IPv6 Dualstack Backend Support
----
* Author(s): @markdroth
* Approver: @ejona86, @dfawley
* Status: Ready for Implementation
* Implemented in: C-core
* Last updated: 2025-02-13
* Discussion at: https://groups.google.com/g/grpc-io/c/VjORlKP97cE/m/ihqyN32TAQAJ

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

This work is being done in conjunction with an effort to add multiple
addresses per endpoint in xDS.  We will support the new xDS APIs being
added for that effort as well.  Note that this change has implications
for session affinity behavior in xDS.

### Related Proposals: 
* [Support for dual stack EDS endpoints in Envoy][envoy-design]
* [gRFC A17: Client-Side Health Checking][A17]
* [gRFC A27: xDS-Based Global Load Balancing][A27]
* [gRFC A58: Weighted Round Robin LB Policy][A58]
* [gRFC A48: xDS Least Request LB Policy][A48]
* [gRFC A42: Ring Hash LB Policy][A42]
* [gRFC A56: Priority LB Policy][A56]
* [gRFC A55: xDS-Based Stateful Session Affinity][A55]
* [gRFC A60: xDS-Based Stateful Session Affinity for Weighted Clusters][A60]
* [gRFC A62: pick_first: Sticky TRANSIENT_FAILURE and address order
  randomization][A62]
* [gRFC A50: Outlier Detection Support][A50]
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
  addresses per endpoint, and we will extend the stateful session
  affinity mechanism to support such endpoints.

### Allow Resolvers to Return Multiple Addresses Per Endpoint

Instead of returning a flat list of addresses, the resolver will be able
to return a list of endpoints, each of which can have multiple addresses.

Because DNS does not have a way to indicate which addresses are
associated with the same endpoint, the DNS resolver will return each
address as a separate endpoint.

#### Attributes Returned by the Resolver

All gRPC implementations have a mechanism for the resolver to return
arbitrary attributes to be passed to the LB policies.  Attributes can
be set at the top level, which is used for things like passing the
XdsClient instance from the resolver to the LB policies (as described in
[gRFC A27][A27]), or per-address, which is used for things like passing
hierarchical address information down through the LB policy tree (as
described in [gRFC A56][A56]).

The exact semantics for these attributes currently vary across languages.
This proposal does not attempt to define unified semantics for these
attributes, although another proposal may attempt that in the future.
For now, this proposal only defines the required changes of this
interface in the wake of supporting multiple addresses per endpoint.

Specifically, the resolver API must provide a mechanism for passing
attributes on a per-endpoint basis.  Most of the attributes that are
currently per-address will now be per-endpoint instead.  Implementations
may also support per-address attributes, but this is not required.

### Happy Eyeballs in the pick_first LB Policy

The pick_first LB policy currently attempts to connect to each address
serially, stopping at the first one that succeeds.  We will change it to
instead use the Happy Eyeballs algorithm on the initial pass through the
address list.  Specifically:

- As per [RFC-8305 section
  5](https://www.rfc-editor.org/rfc/rfc8305#section-5), the default
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
  the timer.
- We will wait for at least one connection attempt on every address to
  fail before we consider the first pass to be complete.  At that point,
  we will request re-resolution.  As per [gRFC A62][A62], we will report
  TRANSIENT_FAILURE state and will continue trying to connect.  We will
  stay in TRANSIENT_FAILURE until either (a) we become connected or (b)
  the LB policy is destroyed by the channel shutting down or going IDLE.

If the first pass completes without a successful connection attempt, we
will switch to a mode where we keep trying to connect to all addresses at
all times, with no regard for the order of the addresses.  Each
individual subchannel will provide [backoff behavior][backoff-spec],
reporting TRANSIENT_FAILURE while in backoff and then IDLE when backoff
has finished.  The pick_first policy will therefore automatically
request a connection whenever a subchannel reports IDLE.  We will count
the number of connection failures, and when that number reaches the
number of subchannels, we will request re-resolution; note that because
the backoff state will differ across the subchannels, this may mean that
we have seen multiple failures of a single subchannel and no failures
from another subchannel, but this is a close enough approximation and
very simple to implement.

Note that every time the LB policy receives a new address list, it will
start an initial Happy Eyeballs pass over the new list, even if some of
the subchannels are not actually new due to their addresses having been
present on both the old and new lists.  This means that on the initial
pass through the address list for a subsequent address list update, when
pick_first decides to start a connection attempt on a given subchannel
(whether because it is the first subchannel in the list or because the
timer fired before the previous address' connection attempt completed),
that subchannel may not be in state IDLE, which is the only state in
which a connection attempt may be requested.  (Note: This same problem
may occur in C-core even on the first address list update, due to
subchannels being shared with other channels.)  Therefore, when we are
ready to start a connection attempt on a given subchannel:

- If the subchannel is in state IDLE, we request a connection attempt
  immediately.  If it is not the last subchannel in the list, we will
  start the timer; if it is the last subchannel in the list, we will
  wait for the attempt to complete.
- If the subchannel is in state CONNECTING, we do not need to actually
  request a connection, but we will treat it as if we did.  If it is not
  the last subchannel in the list, we will start the timer; if it is the
  last subchannel in the list, we will wait for the attempt to complete.
- If the subchannel is in state TRANSIENT_FAILURE, then we know that
  it is in backoff due to a recent connection attempt failure, so we
  treat it as if we have already made a connection attempt on this
  subchannel, and we will immediately move on to the next subchannel.

Note that because we do not report TRANSIENT_FAILURE until after the
Happy Eyeballs pass has completed and we start a new Happy Eyeballs pass
whenever we receive a new address list, there is a potential failure
mode where we may never report TRANSIENT_FAILURE if we are receiving new
address lists faster than we are completing Happy Eyeballs passes.  This
is a pre-existing problem, and each gRPC implementation currently deals
with it in its own way.  This design does not propose any changes to
those existing approaches, although a future gRFC may attempt to achieve
further convergence here.

Once a subchannel does become READY, pick_first will unref all other
subchannels, thus cancelling any connection attempts that were already
in flight.  Note that the [connection backoff][backoff-spec] state is
stored in the subchannel, so this means that we will lose backoff state
for those subchannels (but see note for C-core below).  In general,
this is expected to be okay, because once we see a READY subchannel,
we generally expect to maintain that connection for a while, after which
the backoff state for the other subchannels will no longer be relevant.
However, there could be pathalogical cases where a connection does not
last very long and we wind up making subsequent connection attempts
to the other addresses sooner than we ideally should.  This should be
fairly rare, so we're willing to accept this; if it becomes a problem,
we can find ways to address it at that point.

#### Implications of Subchannel Sharing in C-core

In C-core, there are some additional details to handle due to the
existance of subchannel sharing between channels.  Any given subchannel
that pick_first is using may also be used other channel(s), and any
of those other channels may request a connection on the subchannel
at any time.  This means that pick_first needs to be prepared for the
fact that any subchannel may report any connectivity state at any time
(even at the moment that pick_first starts using the subchannel), even
if it did not previously request a connection on the subchannel itself.
This has a couple of implications:

- pick_first needs to be prepared for any subchannel to report READY at
  any time, even if it did not previously request a connection on that
  subchannel.  Currently (prior to this design), pick_first immediately
  chooses the first subchannel that reports READY.  That behavior seems
  consistent with the intent of Happy Eyeballs, so we will retain it.
- When we choose a subchannel that has become successfully connected,
  we will unref all of the other subchannels.  For any subchannel on
  which we were the only channel holding a ref, this will cause any
  pending connection attempt to be cancelled, and the subchannel will
  be destroyed.  However, if some other channel was holding a ref to the
  subchannel, the connection attempt will continue, even if the other
  channel did not want it.  This is slightly sub-optimal, but it's not
  really a new problem; the same thing can occur today if there are two
  channels both using pick_first with overlapping sets of addresses.
  We can find ways to address this in the future if and when it becomes
  a problem.

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
- least_request (see [gRFC A48][A48] -- currently supported in Java and
  Go only)

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
[RFC-8305 section 4](https://www.rfc-editor.org/rfc/rfc8305#section-4)
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

[RFC-8305 section 4](https://www.rfc-editor.org/rfc/rfc8305#section-4)
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
   families, as per [RFC-8305 section
   4](https://www.rfc-editor.org/rfc/rfc8305#section-4).  Doing this on
   the flattened address list ensures the best behavior if only one of
   the two address families is working.

#### Generic Health Reporting Mechanism

gRPC currently supports two mechanisms that provide a health signal for
a connection: client-side health checking, as described in [gRFC A17][A17],
and outlier detection, as described in [gRFC A50][A50].  Currently, both
mechanisms signal unhealthiness by essentially causing the subchannel to
report TRANSIENT_FAILURE to the leaf LB policy.  However, that approach
will no longer work with this design, as explained in the
[Reaons for Generic Health Reporting](#reasons-for-generic-health-reporting)
section below.

Instead, we need to make these health signals visible to the petiole
policies without affecting the underlying connectivity management of
the pick_first policy.  However, since both of these mechanisms work on
individual subchannels rather than on endpoints with multiple subchannels,
this functionality is best implemented in pick_first itself, since
that's where we know which subchannel was actually chosen.  Therefore,
pick_first will have an option to support these health signals, and
that option will be used only when pick_first is used as a child policy
underneath a petiole policy.

Note that we do not want either of these mechanisms to actually work
when pick_first is used as an LB policy by itself, so we will implement
this functionality in a way that it can be triggered by a parent policy
such as round_robin but cannot be triggered by an external application.
(For example, in C-core, this will be triggered via an internal-only
channel arg that will be set by the petiole policies.)

When this option is enabled in pick_first, it will be necessary for
pick_first to see both the "raw" connectivity state of each subchannel
and the state reflected by health checking.  The connection management
behavior will continue to use the "raw" connectivity state, just as it
does today.  Only once pick_first chooses a subchannel will it start
the health watch, and the connectivity state reported by that watch
is the state that pick_first will report to its parent.

Although we need pick_first to be aware of the chosen subchannel's
health, we do not want it to have to be specifically aware of individual
health-reporting mechanisms like client-side health checking or outlier
detection (or any other such mechanism that we might add in the future).
As a result, we will structure this as a general-purpose health-reporting
watch that will be started by pick_fist without regard to whether any
individual health-reporting mechanism is actually configured.  If no
health-reporting mechanisms are actually configured, the watch will
report the subchannel's raw connectivity state, so it will effectively
be a no-op.

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

Except for the cases noted below (Ring Hash and Outlier Detection),
it is up to the implementation whether a given LB policy takes resolver
attributes into account when comparing endpoints from the old list and
the new list.

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

The least-request LB policy (Java and Go only, described in [gRFC
A48][A48]) will work essentially the same way as WRR.  The only difference
is that the data it is storing on a per-endpoint basis is outstanding
request counts rather than weights.

#### Ring Hash

Currently, as described in [gRFC A42][A42], each entry in the ring is a
single address, positioned based on the hash of that address.  With this
design, that will change such that each entry in the ring is an endpoint,
positioned based on the hash of the endpoint's first address.  However,
once an entry in the ring is selected, we may wind up connecting to the
endpoint on a different address than the one that we hashed to.

Note that this means that if the order of the addresses for a given
endpoint change, that will change the position of the endpoint in
the ring.  This is considered acceptable, since ring_hash is already
subject to churn in the ring whenever the address list changes.

Because ring_hash establishes connections lazily, but pick_first will
attempt to connect as soon as it receives its initial address list, the
ring_hash policy will lazily create the pick_first child when it wants
to connect.

Note that as of [gRFC A62][A62], pick_first has sticky-TF behavior in
all languages: when a connection attempt fails, it continues retrying
indefinitely with appropriate [backoff][backoff-spec], staying in
TRANSIENT_FAILURE state until either it establishes a connection or the
pick_first policy is destroyed.  This means that the ring_hash picker no
longer needs to explicitly trigger connection attempts on subchannels in
state TRANSIENT_FAILURE, which makes the logic much simpler.  The picker
pseudo-code now becomes:

```
first_index = ring.FindIndexForHash(request.hash);
for (i = 0; i < ring.size(); ++i) {
  index = (first_index + i) % ring.size();
  if (ring[index].state == READY) {
    return ring[index].picker->Pick(...);
  }
  if (ring[index].state == IDLE) {
    ring[index].endpoint.TriggerConnectionAttemptInControlPlane();
    return PICK_QUEUE;
  }
  if (ring[index].state == CONNECTING) {
    return PICK_QUEUE;
  }
}
// All children are in transient failure. Return the first failure.
return ring[first_index].picker->Pick(...);
```

As per [gRFC A42][A42], the ring_hash policy normally triggers subchannel
connection attempts from the picker.  However, if it is being used
as a child of the priority policy, it will not be getting any picks
once it reports TRANSIENT_FAILURE, and in some cases even when it
reports CONNECTING, due to the failover timer in the priority policy.
Because it reports TRANSIENT_FAILURE when only two endpoints are failing
(see aggregation rule 2 in [gRFC A42][A42]) and CONNECTING when only
one endpoint is reporting TRANSIENT_FAILURE (see aggregation rule 4 in
[gRFC A42][A42]), this means that the priority policy could fail over to
the next priority when the ring_hash policy is only attempting a small
number of subchannels.  This would effectively cause the client to assume
that all of the ring_hash subchannels are unreachable when in fact only
a small number of them are, and it would never try any of the others,
thus never recovering from that incorrect assumption.  To work around
this, when the aggregated connectivity state is either TRANSIENT_FAILURE
or CONNECTING, the ring_hash policy proactively triggers connection
attempts across all of the subchannels, even without seeing any picks.
This ensures that if any of the subchannels become reachable, ring_hash
will eventually report READY, and the priority policy will switch back
to using the ring_hash child.

The current logic for these connection attempts tries very hard to avoid
connecting to more than one subchannel at a time; when a connection
attempt on one subchannel fails, it does not try that subchannel again,
instead moving on to the next one.  This approach was an attempt
to avoid accumulating too many possibly-unnecessary connections once
connectivity is restored.  However, with the changes in this design, it
will no longer be possible to attempt to connect to only one endpoint
at a time, because of the sticky-TF behavior in pick_first ([gRFC
A62][A62]): when a given pick_first child reports TRANSIENT_FAILURE,
it will automatically try reconnecting after the backoff period without
waiting for a connection to be requested.  Therefore, the ring_hash
policy will instead simply increase the number of endpoints that are
attempting to connect as each one fails.  Note that this means that after
an extended connectivity outage, ring_hash will now often wind up with
many unnecessary connections.  However, this situation is also possible
via the picker if ring_hash is the last child under the priority policy,
so we are willing to live with this behavior for now.  If it becomes a
problem in the future, we can consider ways to ameliorate it at that time.

The specific behavior for this will be that whenever the ring_hash
policy gets a subchannel connectivity state update or a resolver update,
if the aggregated connectivity state is TRANSIENT_FAILURE or CONNECTING
and there are no endpoints in CONNECTING state, the ring_hash policy will
choose one of the endpoints in IDLE state (if any) to trigger a connection
attempt on.  It does not matter which IDLE endpoint is chosen; that is
left up to the implementation to determine.  One possible implementation
of this is shown in the following pseudo-code:

```
if (aggregated_state_is_connecting_or_transient_failure) {
  first_idle_index = -1;
  for (i = 0; i < endpoints.size(); ++i) {
    if (endpoints[i].connectivity_state() == CONNECTING) {
      first_idle_index = -1;
      break;
    }
    if (first_idle_index == -1 && endpoints[i].connectivity_state() == IDLE) {
      first_idle_index = i;
    }
  }
  if (first_idle_index != -1) {
    endpoints[first_idle_index].RequestConnection();
  }
}
```

Note that in C-core, the normal approach for handling address list
updates described [above](#address-list-updates-in-c-core) won't work,
because if we are creating the pick_first children lazily, then we will
wind up not creating the children in the new endpoint list and thus
never swapping over to it.  As a result, ring_hash in C-core will use an
approach more like that of [Java and Go](#address-list-updates-in-javago):
it will maintain a map of endpoints by the set of addresses, and it will
update that set in place when it receives an updated address list.

Because ring_hash chooses which endpoint to use via a hash function based
solely on the first address of the endpoint, it does not make sense to
have multiple endpoints with the same address that are differentiated
only by the resolver attributes.  Thus, resolver attributes are ignored
when de-duping endpoints.

#### Outlier Detection

The goal of the outlier detection policy is to temporarily stop sending
traffic to servers that are returning an unusually large error rate.
The kinds of problems that it is intended to catch are primarily things
that are independent of which address is used to connect to the server;
a problem with the reachability of a particular address is more likely to
cause connectivity problems than individual RPC failures, and problems
that cause RPC failures are generally just as likely to occur on any
address.  Therefore, this design changes the outlier detection policy
to make ejection decisions on a per-endpoint basis, instead of on a
per-address basis as it does today.  RPCs made to any address associated
with an endpoint will count as activity on that endpoint, and ejection
or unejection decisions for an endpoint will affect subchannels for all
addresses of an endpoint.

As described in [gRFC A50][A50], the outlier detection policy currently
maintains a map keyed by individual address.  The map values contain both
the set of currently existing subchannels for a given address as well
as the ejection state for that address.  This map will be split into
two maps: a map of currently existing subchannels, keyed by individual
address, and a map of ejection state, keyed by the unordered set of
addresses on the endpoint.

The entry in the subchannel map will hold a ref to the corresponding
entry in the endpoint map.  This ref will be updated when the LB policy
receives an updated address list, when the list of addresses in the
endpoint changes.  It will be used to update the successful and failed
call counts as each RPC finishes.  Note that appropriate synchronization
is required for those two different accesses.

The entry in the endpoint map may hold a pointer to the entries in the
subchannel map for the addresses associated with the endpoint, or the
implementation may simply look up each of the endpoint's addresses in
the subchannel map separately.  These accesses from the endpoint map
to the subchannel map will be performed by the LB policy when ejecting
or unejecting the endpoint, to send health state notifications to the
corresponding subchannels.  Note that if the ejection timer runs in the
same synchronization context as the rest of the activity in the LB policy,
no additional synchronization should be needed here.

The set of entries in both maps will continue to be set based on the
address list that the outlier detection policy receives from its parent.
And the map keys will continue to use only the addresses, not taking
resolver attributes into account.

Currently, the outlier detection policy wraps the subchannels and ejects
them by reporting their connectivity state as TRANSIENT_FAILURE.
As described [above](#generic-health-reporting-mechanism), we will
change the outlier detection policy to instead eject endpoints by
wrapping the subchannel's generic health reporting mechanism.

### Support Multiple Addresses Per Endpoint in xDS

The EDS resource has been updated to support multiple addresses per
endpoint in
[envoyproxy/envoy#27881](https://github.com/envoyproxy/envoy/pull/27881).
Specifically, that PR adds a new `AdditionalAddress` message, which
contains a single `address` field, and it adds a repeated
`additional_addresses` field of that type to the `Endpoint` proto.

When validating the EDS resource, when processing the `Endpoint` proto,
we validate each entry of `additional_addresses` as follows:
- If the `address` field is unset, we reject the resource.
- If the `address` field *is* set, then we validate it exactly the same
  way that we already validate the `Endpoint.address` field.

#### Changes to Stateful Session Affinity

We need to support endpoints with multiple addresses in stateful session
affinity (see gRFCs [A55][A55] and [A60][A60]).  We want to add one
additional property here, which is that we do not want affinity to break
if an endpoint has multiple addresses and then one of those addresses
is removed in an EDS update.  This will require some changes to the
original design.

First, the session cookie, which currently contains a single endpoint
address, will be changed to contain a list of endpoint addresses.  As per
gRFC A60, the cookie's format is a base64-encoded string of the form
`<address>;<cluster>`.  This design changes that format such that the
address part will be a comma-delimited list of addresses.  The
`StatefulSession` filter currently sets a call attribute that
communicates the address from the cookie to the `xds_override_host` LB
policy; that call attribute will now contain the list of addresses from
the cookie.

Next, the entries in the address map in the `xds_override_host` LB policy
need to contain the actual address list to be used in the cookie when
a given address is picked.  Note that the original design already described
how we would represent endpoints with multiple addresses in this map,
since that was already possible in Java (see the description in A55 of
handling EquivalentAddressGroups when constructing the map).  However,
the original design envisioned that we would store a list of addresses
that would be looked up as keys in the map when finding alternative
addresses to use, which we no longer need now that we will be encoding
the list of addresses in the cookie itself.  Instead, what we need from
the map entry is the information necessary to construct the list of
addresses to be encoded in the cookie when the address for that map
entry is picked.  Implementations will likely want to store this as a
single string instead of a list, since that will avoid the need to
construct the string on a per-RPC basis.

As per the original design, when returning the server's initial metadata
to the application, the `StatefulSession` filter may need to set a cookie
indicating which endpoint was chosen for the RPC.  However, now that the
cookie needs to include all of the endpoint's addresses and not just the
specific one that is used, we need to communicate that information from
the `xds_override_host` LB policy back to the `StatefulSession` filter.
This will be done via the same call attribute that the `StatefulSession`
filter creates to communicate the list of addresses from the cookie to
the `xds_override_host` policy.  That attribute will be given a new
method to allow the `xds_override_host` policy to set the list of
addresses to be encoded in the cookie, based on the address chosen by
the picker.  The `StatefulSession` filter will then update the cookie if
the address list in the cookie does not match the address list reported
by the `xds_override_host` policy.  Note that when encoding the cookie,
the address that is actually used must be the first address in the list.

In accordance with those changes, the picker logic will now look like this:

```
def Pick(pick_args):
  override_host_attribute = pick_args.call_attributes.get(attribute_key)
  if override_host_attribute is not None:
    idle_subchannel = None
    found_connecting = False
    for address in override_host_attribute.cookie_address_list:
      entry = lb_policy.address_map[address]
      if entry found:
        if (entry.subchannel is set AND
            entry.health_status is in policy_config.override_host_status):
          if entry.subchannel.connectivity_state == READY:
            override_host_attribute.set_actual_address_list(entry.address_list)
            return entry.subchannel as pick result
          elif entry.subchannel.connectivity_state == IDLE:
            if idle_subchannel is None:
              idle_subchannel = entry.subchannel
          elif entry.subchannel.connectivity_state == CONNECTING:
            found_connecting = True
    # No READY subchannel found.  If we found an IDLE subchannel,
    # trigger a connection attempt and queue the pick until that attempt
    # completes.
    if idle_subchannel is not None:
      hop into control plane to trigger connection attempt for idle_subchannel
      return queue as pick result
    # No READY or IDLE subchannels.  If we found a CONNECTING
    # subchannel, queue the pick and wait for the connection attempt
    # to complete.
    if found_connecting:
      return queue as pick result
  # pick_args.override_addresses not set or did not find a matching subchannel,
  # so delegate to the child picker.
  result = child_picker.Pick(pick_args)
  if result.type == PICK_COMPLETE:
    entry = lb_policy.address_map[result.subchannel.address()]
    if entry found:
      override_host_attribute.set_actual_address_list(entry.address_list)
  return result
```

### Temporary environment variable protection

The code that reads the new EDS fields will be initially guarded by an
environment variable called `GRPC_EXPERIMENTAL_XDS_DUALSTACK_ENDPOINTS`.
This environment variable guard will be removed once this feature has
proven stable.

Note that we will not use this environment variable to guard the Happy
Eyeballs functionality, because that functionality will be on by
default, not something that is enabled via external input.

## Rationale

### Happy Eyeballs Functionality

Note that we will not support all parts of "Happy Eyeballs" as described
in [RFC-8305][RFC-8305].  For example, because our resolver API does
not provide a way to return some addresses without others, we will not
start trying to connect before all of the DNS queries have returned.

### Java and Go Pick First Restructuring

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

### Reasons for Generic Health Reporting

Currently, client-side health checking and outlier detection
signal unhealthiness by essentially causing the subchannel to report
TRANSIENT_FAILURE to the leaf LB policy.  This existing approach works
reasonably when petiole policies directly create and manage subchannels,
but it will not work when pick_first is the universal leaf policy.
When pick_first sees its chosen subchannel transition from READY to
TRANSIENT_FAILURE, it will interpret that as the connection failing, so
it will unref the subchannel and report IDLE to its parent.  This causes
two problems.

The first problem is that we don't want unhealthiness to trigger
connection churn, but pick_first would react in this case by dropping
the existing connection unnecessarily.  Note that, as described in [gRFC
A17](A17-client-side-health-checking.md#pick_first), the client-side
health checking mechanism does not work with pick_first, for this exact
reason.  In hindsight, we should have imposed the same restriction for
outlier detection, but that was not explicitly stated in [gRFC A50][A50].
However, that gRFC does say that outlier detection will ignore subchannels
with multiple addresses, which is the case in Java and Go.  In C-core,
it should have worked with pick_first, although it turns out that there
was a bug that prevented it from working, which means that we can know
that no users were actually counting on this behavior.  This means that
we can retroactively say that outlier detection should never have worked
with pick_first with minimal risk of affecting users that might have
been counting on this use-case.  (It might affect Java/Go channels that
use pick_first and happen to have only one address, and it might have
been used in Node.)

The second problem is that this would cause pick_first to report IDLE
instead of TRANSIENT_FAILURE up to the petiole policy.  This could
affect the aggregated connectivity state that the petiole policy reports
to *its* parent.  And parent policies like the priority policy (see
[gRFC A56][A56]) may then make the wrong routing decision based on that
incorrect state.

These problems are solved via the introduction of the [Generic Health
Reporting Mechanism](#generic-health-reporting-mechanism).

## Implementation

### C-core

- move client-side health checking out of subchannel so that it can be
  controlled by pick_first (https://github.com/grpc/grpc/pull/32709)
- assume LB policies start in CONNECTING state
  (https://github.com/grpc/grpc/pull/33009)
- prep for outlier detection ejecting via health watch
  (https://github.com/grpc/grpc/pull/33340)
- move pick_first off of the subchannel_list library that it previously
  shared with petiole policies, and add generic health watch support
  (https://github.com/grpc/grpc/pull/34218)
- change petiole policies to use generic health watch, and change outlier
  detection to eject via health state instead of raw connectivity state
  (https://github.com/grpc/grpc/pull/34222)
- change ring_hash to delegate to pick_first
  (https://github.com/grpc/grpc/pull/34244)
- add endpoint_list library for petiole policies, and use it to change
  round_robin to delegate to pick_first
  (https://github.com/grpc/grpc/pull/34337)
- change WRR to delegate to pick_first
  (https://github.com/grpc/grpc/pull/34245)
- implement happy eyeballs in pick_first
  (https://github.com/grpc/grpc/pull/34426 and
  https://github.com/grpc/grpc/pull/34717)
- implement address interleaving for happy eyeballs
  (https://github.com/grpc/grpc/pull/34615 and
  https://github.com/grpc/grpc/pull/34804)
- change resolver and LB policy APIs to support multiple addresses per
  endpoint, and update most LB policies
  (https://github.com/grpc/grpc/pull/33567)
- support new xDS fields (https://github.com/grpc/grpc/pull/34506)
- change outlier detection to handle multiple addresses per endpoint
  (https://github.com/grpc/grpc/pull/34526)
- change stateful session affinity to handle multiple addresses per endpoint
  (https://github.com/grpc/grpc/pull/34472)

### Java

- move pick_first logic out of subchannel and into pick_first policy
- make pick_first the universal leaf policy, including client-side
  health checking support
- implement happy eyeballs in pick_first
- fix ring_hash to support endpoints with multiple addresses
- support new xDS fields

### Go

- change subchannel connectivity state API (maybe)
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
[A27]: A27-xds-global-load-balancing.md
[A42]: A42-xds-ring-hash-lb-policy.md
[A48]: A48-xds-least-request-lb-policy.md
[A50]: A50-xds-outlier-detection.md
[A51]: A51-custom-backend-metrics.md
[A55]: A55-xds-stateful-session-affinity.md
[A60]: A60-xds-stateful-session-affinity-weighted-clusters.md
[A56]: A56-priority-lb-policy.md
[A58]: A58-client-side-weighted-round-robin-lb-policy.md
[RFC-8305]: https://www.rfc-editor.org/rfc/rfc8305
[A62]: A62-pick-first.md
[backoff-spec]: https://github.com/grpc/grpc/blob/master/doc/connection-backoff.md
