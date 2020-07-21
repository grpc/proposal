# gRPC xDS circuit breaking

* Author(s): Chengyuan Zhang (voidzcy)
* Approver: markdroth
* Status: In Review
* Implemented in:
* Last updated: 2020-07-20
* Discussion at: https://groups.google.com/g/grpc-io/c/NEx70p8mcjg


## Abstract

Adopt Envoy's [circuit breaking](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/circuit_breaking#circuit-breaking) 
component that comes with the xDS protocol to
fail network operations quickly and apply back pressure downstream as soon as
possible. This allows gRPC clients to received distributed traffic limit 
configurations that prevent cascaded service overload failures. 

## Background

Envoy supports configurable circuit breaking settings on per upstream cluster 
and per priority basis to allow different components of the distributed system
to be tuned independently. Since gRPC xDS does not support [priority based
routing](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/http/http_routing#arch-overview-http-routing-priority), 
requests to a single upstream cluster conform to the single circuit breaking
configuration.

Envoy handles HTTP traffic with [connection pooling](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/connection_pooling#connection-pooling),
and many circuit breaking parameters are configured for
connection/connection pool management. Since each gRPC client creates at most
one HTTP/2 connection to a single upstream endpoint address, those
parameters do not apply to gRPC and will be ignored. See more in
[Other parameters considered](#other-parameters-considered).

With the introduction of [gRFC A28: xDS Traffic Splitting and Routing](https://github.com/grpc/proposal/blob/master/A28-xds-traffic-splitting-and-routing.md), 
it became possible for there to be more than one CDS LB policy instance for a 
given cluster (e.g., if one route pointed to a given cluster and a different 
route split traffic between that cluster and other clusters).  However, after 
implementing the changes described in [gRPC A31: xDS RouteActions Support](https://github.com/grpc/proposal/pull/192),
there will be only one CDS LB policy instance for a given cluster, which will
make the CDS LB policy the logical place to implement circuit breakers.

## Overview

The `Cluster` resource encodes the circuit breaking parameters in a list of
[Thresholds](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/cluster/circuit_breaker.proto#cluster-circuitbreakers-thresholds)
messages, where each message specifies the limits for a particular 
RoutingPriority. gRPC will look only at the first entry in the list for 
priority [`DEFAULT`](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/core/base.proto#enum-core-routingpriority).

Of the parameters in the `Thresholds` message, gRPC will support only
`max_requests`, which sets the maximum number of requests that can be in flight
to the cluster at any given time.  None of the other parameters are applicable
to gRPC; for details, see [Other parameters considered](#other-parameters-considered).
The design and implementation parts of this doc will mainly focus on the
application of the `max_requests` circuit breaking parameter in gRPC.

To match Envoy's implementation, circuit breaking parameters will always be 
enforced with certain defaults, even if the gRPC client receives no applicable 
circuit breakers. Users can effectively turn circuit breaking off by setting
thresholds to very high values, for example, to 
`std::numeric_limits<uint32_t>::max()`.

## Detailed design

### Maintaining thresholds

Each CDS LB policy maintains a threshold for the maximum number of outstanding 
requests to hosts in the cluster that this policy is load balancing for. This
threshold will be updated when receiving an update from the XdsClient cluster 
watch interface, and the updated value will be dynamically picked up by request
limiting logic. By default, this value is set to 1024.

### Limiting outstanding requests

The CDS policy's picker will enforce the configured limit. Each CDS LB policy
will maintain a counter of the number of requests currently in flight to 
the associated cluster. The counter will be incremented when a request
is started on a subchannel and decremented when the request is complete. 
When the picker is going to start a call, it will check whether the current
number of in-flight requests is lower than the configured limit. If so, the 
picker will do the pick as usual, returning a subchannel. If not, it will 
fail the call with status UNAVAILABLE. Such failed calls will not be retried,
but they will be recorded to the `total_dropped_requests` counts and reported
to the load reporting server.

## Implementation

### Java
- Maintain a counter in the CDS LB policy for counting the number of 
outstanding requests.
- In CDS LB policy, wrap the `SubchannelPicker` propagated from its child 
EDS policy with the logic of checking the counter and make pick decision.
- Counter manipulation is done in `ClientStreamTracer`/`Factory`: increment
in `newClientStreamTracer()` and decrement in `streamClosed()`.
    - Note there will be check-and-allocate race as the pick method and stream
    tracer are not run by the same thread. This will cause the threshold to
    be exceeded. Since the race window is small, the value should be exceeded 
    only by a small amount.

### C
TODO

### Go
TODO

## Other parameters considered

There is a group of other parameters in the xDS circuit breakers, but most of
them do not apply to gRPC's use case, and they will be ignored in the 
implementation. The following parameters have been considered:

- `max_connections`: the maximum number of connections can be created to an 
upstream cluster.
    - Envoy allows at least one connection to be allocated for each host 
    selected by cluster load balancing, even if this circuit breaker is
    overflowed. On the other hand, gRPC creates at most one connection to each
    host. So this parameter does not have effective meaning in gRPC's use
    case.
- `max_pending_requests`: the maximum number of requests that will be queued
while waiting for a connection to be established.
    - As a proxy, Envoy drops queued requests from downstream entities to avoid
    unbounded congestion and memory usage of the process. But as part of the
    application, it makes little sense for gRPC to drop its own requests.
- `max_retries`: retry related parameter, the retry feature has not been 
implemented in gRPC xDS.
- `max_budget`: retry related parameter, the retry feature has not been 
implemented in gRPC xDS.
- `max_connection_pools`: gRPC does not use connection pooling.
- `connect_timeout` (in `Cluster` configuration): hard to implement in gRPC's 
architecture, will reconsider after getting usage feedback.
- `max_requests_per_connection` (in `Cluster` configuration): lifetime limit 
of maximum requests for a single upstream connection. No valuable use case for
gRPC.

Some of the above parameters may be reconsidered for gRPC xDS circuit
breaking after we get more feedback and inquiries.
