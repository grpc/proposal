# gRPC xDS circuit breaking

* Author(s): Chengyuan Zhang (voidzcy)
* Approver: markdroth, ejona86, dfawley
* Status: Approved
* Implemented in:
* Last updated: 2020-09-14
* Discussion at: https://groups.google.com/g/grpc-io/c/NEx70p8mcjg


## Abstract

Adopt Envoy's [circuit breaking](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/circuit_breaking#circuit-breaking) 
component that comes with the xDS protocol to
fail network operations quickly and apply back pressure downstream as soon as
possible. This allows gRPC clients to receive distributed traffic limit 
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

## Overview

The `Cluster` resource encodes the circuit breaking parameters in a list of
[Thresholds](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/cluster/circuit_breaker.proto#cluster-circuitbreakers-thresholds)
messages, where each message specifies the parameters for a particular 
RoutingPriority. gRPC will look only at the first entry in the list for 
priority [`DEFAULT`](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/core/base.proto#enum-core-routingpriority).

Of the parameters in the `Thresholds` message, gRPC will support only
`max_requests`, which sets the maximum number of requests that can be in flight
to the cluster at any given time. None of the other parameters are applicable
to gRPC; for details, see [Other parameters considered](#other-parameters-considered).
The design and implementation parts of this doc will mainly focus on the
application of the `max_requests` circuit breaking parameter in gRPC.

To match Envoy's implementation, circuit breaking parameters will always be 
enforced with certain defaults, even if the gRPC client receives no applicable 
circuit breakers. Users can effectively turn circuit breaking off by setting
limits to very high values, for example, to 
`std::numeric_limits<uint32_t>::max()`.

## Detailed design

With the introduction of [gRFC A28: xDS Traffic Splitting and Routing](https://github.com/grpc/proposal/blob/master/A28-xds-traffic-splitting-and-routing.md), 
it became possible for there to be more than one CDS LB policy instance for a 
given cluster (e.g., if one route pointed to a given cluster and a different 
route split traffic between that cluster and other clusters).  However, after 
implementing the changes described in [gRPC A31: xDS RouteActions Support](https://github.com/grpc/proposal/pull/192),
there will be only one CDS LB policy instance for a given cluster at any time
in the LB policy tree. Since the EDS LB policy is the counterpart for endpoint
discovery of the cluster, both the CDS and EDS LB policies comprise the load
balancing logic for requests sent to the specific cluster. Circuit breakers can
be implemented in either policy. Since cluster load assignment drops are 
handled in EDS LB policy, it makes sense to handle circuit breaking drops in 
the same place.

Each CDS LB policy will receive a configured limit for the maximum number of 
outstanding requests to hosts in the cluster that this policy is load balancing
for from the XdsClient cluster watch interface. A default value of 1024 is used
if no such limit is specified by the management server. This limit will be 
included in the LB config passed to its child EDS LB policy. The EDS LB config 
will become

```proto
message EdsLoadBalancingPolicyConfig {
  string cluster = 1;
  string eds_service_name = 2;
  google.protobuf.String lrs_load_reporting_server_name = 3;
  repeated LoadBalancingConfig locality_picking_policy = 4;
  repeated LoadBalancingConfig endpoint_picking_policy = 5;
  
  // Maximum number of outstanding requests can be made to the upstream cluster.
  // Default is 1024.
  google.protobuf.UInt32Value max_concurrent_requests = 6;
}
```

The EDS LB policy will use this limit as the upper bound for the maximum number
of in-flight requests it allows to send.

The EDS policy's picker will enforce the configured limit. Each EDS LB policy
will maintain a counter for the number of requests currently in flight to 
the associated cluster. The counter will be incremented when a request
is started on a subchannel and decremented when the request is complete. 
When the picker is going to start a call, it will check whether the current
number of in-flight requests is lower than the configured limit. If so, the 
picker will do the pick as usual, returning a subchannel. If not, it will 
fail the call with status UNAVAILABLE. Such failed calls will not be retried,
but they will be recorded to the `total_dropped_requests` counts in 
cluster-level load reports and reported to the load reporting server.

Note that in Envoy's implementation, cluster resources are immutable and any 
change in the cluster resource implies replacing the old cluster with a new one
with the latest configuration. The new cluster starts with new circuit breaker
tracking states. However, in gRPC this only happens when the cluster's EDS
service name changes. This could result in a surprising behavior compared to
Enovy: if the current number of RPCs in flight is 105 and we receive a CDS
update lowering the limit to 100, new RPCs will still fail even if 5 of the
currently in-flight RPCs have finished. In contrast, Envoy will allow 
another 100 new RPCs to be sent immediately.

## Implementation

### Java
- Maintain a counter in the EDS LB policy for counting the number of 
outstanding requests.
- Add the logic of enforcing `max_request` limit in wrapping the 
`SubchannelPicker` propagated from the EDS policy's child LB policies.
- Counter manipulation is done in `ClientStreamTracer`/`Factory`: increment
in `newClientStreamTracer()` and decrement in `streamClosed()`.
    - Note there will be check-and-allocate race as the pick method and stream
    tracer are not run by the same thread. This will cause the limit to
    be exceeded. Since the race window is small, the value should be exceeded 
    only by a small amount.

### C
TODO

### Go
TODO

## Other parameters considered

There are two other categories of circuit breaking parameters we have 
considered, but they are ignored at this point.

The first category of parameters are those fundamentally not applicable to gRPC
in its current form. It is possible that gRPC could evolve to make them 
relevant, in which case we would reevaluate them, but it is also possible that 
they will simply never be supported.

- `max_requests_per_connection` (in the `Cluster` proto) and `max_connections`: 
gRPC only ever creates a single connection to a given endpoint. But in the 
future, we might provide a new LB policy to allow multiple connections to 
address `MAX_CONCURRENT_STREAMS` limit, at which point it might make sense to 
support this.
- `max_connection_pools`: gRPC does not currently have a concept of connection
pools the way that Envoy does. This will likely never apply to gRPC.

The second category of parameters are those applicable to gRPC, but their 
implementations are non-trivial. We do not want to invest the resources to 
implement them until we know that they will be useful enough to make that 
investment worthwhile. We will reevaluate them as we get feedback from users.

- `max_pending_requests`: There are difficulties in integrating this with other
existing gRPC features. For example, a buffered request may be removed upon 
hitting its deadline but there is no way to have the xDS plugin know this. It 
may need a significant amount of changes for gRPC to support this.
- `connect_timeout` (in the `Cluster` proto): Implementation may require
transport layer change, and it also has particular challenges in C-core due to
the way we share subchannels between channels.
- `max_retries` and `max_budget`: These will be addressed when we eventually
add support for configuring retries via xDS.
