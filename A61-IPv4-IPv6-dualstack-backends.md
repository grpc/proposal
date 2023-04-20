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

Also, when connecting to an endpoint with multiple addresses, it is
desirable to use the "Happy Eyeballs" algorithm described in
[RFC-8305](https://www.rfc-editor.org/rfc/rfc8305) to minimize the time
it takes to establish a working connection by parallelizing connection
attempts in a reasonable way.  Currently, all gRPC implementations
perform connection attempts in a completely serial manner in the
pick_first LB policy.

This work is being doing in conjunction with an effort to add multiple
addresses per endpoint in xDS.  We will support the new xDS APIs being
added for that effort as well.  Note that this change has implications
for session affinity behavior in xDS.

### Related Proposals: 
* TODO: reference gRFC for sticky-TF in PF
* TODO: reference to xDS design or Envoy GH issue
* [gRFC A17: Client-Side Health Checking][A17]
* [gRFC A58: Weighted Round Robin LB Policy][A58]
* [gRFC A48: xDS Least Request LB Policy][A48]
* [gRFC A42: Ring Hash LB Policy][A42]
* [gRFC A55: xDS-Based Stateful Session Affinity][A55]

## Proposal

This proposal includes the following parts:
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

### Happy Eyeballs in the pick_first LB Policy

The pick_first LB policy currently attempts to connect to each address
serially, stopping at the first one that succeeds.  We will change it to
instead use the Happy Eyeballs algorithm.  Specifically:

- TODO: add details
- TODO: in C-core, need to handle subchannel sharing (this PF policy may
  not be the only thing triggering connection attempts, so the
  subchannel may be in an unexpected state when we are ready to start
  connecting -- what do we do then?)
- TODO: for Java/Go, implement Happy Eyeballs in PF or in subchannel?
- TODO: differences in behavior between C-core and Java/Go with respect
  to where the backoff is implemented?  (may not matter if Java/Go
  implement Happy Eyeballs in PF instead of in the subchannel)

#### Use pick_first as the Universal Leaf Policy

Because we do not want to implement Happy Eyeballs multiple times, we
will expect all other policies that need that functionality to delegate
to pick_first.  In particular, the following "leaf" policies will need
to be changed to delegate to pick_first for each endpoint instead of
directly creating subchannels:
- round_robin (see [gRPC Load Balancing](https://github.com/grpc/grpc/blob/master/doc/load-balancing.md#round_robin))
- weighted_round_robin (see [gRFC A58][A58])
- ring_hash (see [gRFC A42][A42])
- least_request (see [gRFC A48][A48] -- currently Java-only)

TODO: details about how we generate useful error messages for failed
RPCs -- may need to delegate to PF picker and then augment the returned
error message on the way up, at least in Java/Go, because their LB
policy API cannot report a status with the connectivity state
(and maybe C-core should not do that either?)

#### Client-Side Health Checking in pick_first

Note that, as described in [gRFC A17][A17], pick_first does not support
client-side health checking, but the other policies described above do.
To support this, it will be necessary for pick_first to optionally
support 

### Support Multiple Addresses Per Endpoint in xDS

TODO: details

#### New EDS Fields

TODO: details

#### Changes to Ring Hash LB Policy

TODO: details

#### Changes to Stateful Session Affinity

TODO: details

### Temporary environment variable protection

The code that reads the new EDS fields will be initially guarded by an
environment variable called `GRPC_EXPERIMENTAL_XDS_DUALSTACK_ENDPOINTS`.
This environment variable guard will be removed once this feature has
proven stable.

## Rationale

Note that we will not support all parts of "Happy Eyeballs" as described in
[RFC-8305](https://www.rfc-editor.org/rfc/rfc8305).  For example,
because our resolver API does not provide a way to return some addresses
without others, we will not start trying to connect before all of the
DNS queries have returned.

## Implementation

### C-core

TODO: details

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
