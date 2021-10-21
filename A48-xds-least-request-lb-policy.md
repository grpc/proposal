A48: xDS Least Request LB Policy
----
* Author(s): erikjoh & [fabric](https://github.com/orgs/spotify/teams/fabric)
* Approver: a11r
* Status: Implementation in progress
* Implemented in: Java in progress
* Last updated: 2021-10-21
* Discussion at: TBD

## Abstract

Add support for least request load balancing configured via xDS.

## Background

In multi-tenancy environments like for example Kubernetes, some upstream endpoints may perform much slower than others.
When using plain round-robin load balancing, this becomes a problem since all endpoints get an equal amount of RPCs
even when some endpoints perform worse. The result is increased tail latencies and in extreme cases even increased error rates.
As an alternative to round-robin, Envoy provides a "least request"
[load balancing policy](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/load_balancers#weighted-least-request)
which uses per-endpoint outstanding/active request counters as a heuristic for determining load on each endpoint.
The Envoy least request implementation uses two different algorithms depending on whether endpoints have non-equal
weights or not. In the "*all weights equal*" case, Envoy samples N random available hosts and selects the one with
fewest outstanding requests. In the "*all weights not equal*" case, Envoy switches to a weighted round robin schedule
where weights are dynamically adjusted based on the outstanding requests.

### Related Proposals: 

This proposal builds on earlier work described in the following gRFCs:
* [gRFC A27: xDS-based Global Load Balancing](https://github.com/grpc/proposal/blob/master/A27-xds-global-load-balancing.md)
* [gRFC A42: xDS Ring Hash LB Policy](https://github.com/grpc/proposal/blob/master/A42-xds-ring-hash-lb-policy.md)

## Proposal

Support `LEAST_REQUEST` LB policy and configuration of it using xDS.
Specifically, gRPC will only support the "*all weights equal*" least request algorithm as
[defined by Envoy](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/load_balancers#weighted-least-request).

### Least Request LB Policy

#### xDS API Fields

The xDS `Cluster` resource specifies the load balancing policy to use
when choosing the endpoint to which each request is sent.  The policy to
use is specified in the [`lb_policy`
field](https://github.com/envoyproxy/envoy/blob/2443032526cf6e50d63d35770df9473dd0460fc0/api/envoy/config/cluster/v3/cluster.proto#L722).
Prior to this proposal, gRPC supported two values for this field,
`ROUND_ROBIN` and `RING_HASH`.  With this proposal, we will add support for an
additional policy, `LEAST_REQUEST`.

The configuration for the Least Request LB policy is the
[`least_request_lb_config`
field](https://github.com/envoyproxy/envoy/blob/2443032526cf6e50d63d35770df9473dd0460fc0/api/envoy/config/cluster/v3/cluster.proto#L916).
The field is optional; if not present, defaults will be assumed for all
of its values.  gRPC will support the
[`choice_count`](https://github.com/envoyproxy/envoy/blob/2443032526cf6e50d63d35770df9473dd0460fc0/api/envoy/config/cluster/v3/cluster.proto#L346)
field the same way that Envoy does.
The
[`active_request_bias`](https://github.com/envoyproxy/envoy/blob/2443032526cf6e50d63d35770df9473dd0460fc0/api/envoy/config/cluster/v3/cluster.proto#L371)
field will be ignored as it is only used in the "*all weights not equal*" (weighted least request)
case which will not be supported as part of this proposal.

#### Changes to the `xds_cluster_resolver` Policy

As a result of
[gRFC A42: xDS Ring Hash LB Policy](https://github.com/grpc/proposal/blob/master/A42-xds-ring-hash-lb-policy.md),
the `xds_cluster_resolver` policy implementation will need additional logic to understand the setup for `LEAST_REQUEST`
in addition to `ROUND_ROBIN` and `RING_HASH`.
For `LEAST_REQUEST`, the policy name will be `"LEAST_REQUEST"`, and the config will be the one for the `least_request_experimental` LB Policy described below.
When the `xds_cluster_resolver` policy sees `"LEAST_REQUEST"`, it will also assume that the
locality-picking policy is `weighted_target` and the endpoint-picking policy is `least_request_experimental`.

#### `least_request_experimental` LB Policy

We will implement a `least_request_experimental` LB policy that uses the same
algorithm as Envoy's implementation.  However, this policy will be
implemented in a non-xDS-specific way, so that it can also be used without
xDS in the future.

##### LB Policy Config

The `least_request_experimental` policy will have the following config:

```proto
message LeastRequestLoadBalancingConfig {
  uint32 choice_count = 1;  // Optional, defaults to 2.
}
```

The `choice_count` determines the number of randomly sampled subchannels which should be compared
for each pick by the picker.

##### Subchannel state handling

The subchannel state handling implemented in the `round_robin` policy will be re-used for `least_request_experimental`.
Implementations may break out this subchannel state handling and have `round_robin` and `least_request_experimental`
inherit the logic instead. This would be similar to the "base balancer" already implemented in grpc-go
[here](https://github.com/grpc/grpc-go/blob/03268c8ed29e801944a2265a82f240f7c0e1b1c3/balancer/base/balancer.go).

TODO: Possibly flesh this out with more explicit logic pending proposal PR review.

##### Aggregated Connectivity State

The `least_request_experimental` policy will use the same heuristic for determining the aggregated
connectivity state as `ring_hash_experimental` defined in
[gRFC A42: xDS Ring Hash LB Policy](https://github.com/grpc/proposal/blob/master/A42-xds-ring-hash-lb-policy.md#aggregated-connectivity-state).
As explained there, this heuristic ensures that the priority policy will fail over to the
next priority quickly when there's an outage.

##### Picker Behavior

The picker should be given a list of all subchannels with the `READY` state.
With this list, the picker should pick a subchannel based on the following pseudo-code:

```
candidate = null;
for (int i = 0; i < choiceCount; ++i) {
    sampled = subchannels[random_integer(0, subchannels.length)]
    if (candidate == null) {
        candidate = sampled;
        continue;
    }

    if (sampled.active_requests < candidate.active_requests) {
        candidate = sampled;
    }
}
return candidate;
```

This pseudo-code is mirroring the
[Envoy implementation](https://github.com/envoyproxy/envoy/blob/2443032526cf6e50d63d35770df9473dd0460fc0/source/common/upstream/load_balancer_impl.cc#L844-L865)

##### Outstanding request counters

The `least_request_experimental` policy will associate one outstanding request counter for each subchannel.
These counters should additionally share the lifecycle with its corresponding subchannel.
The counter for a subchannel should be atomically incremented by one once it has been picked by the picker.
In the `PickResult` for each language-specific implementation (
[Java](https://github.com/grpc/grpc-java/blob/1f90e0e28d5628195cb1f861b73e45ed003f2973/api/src/main/java/io/grpc/LoadBalancer.java#L561-L571),
[C++](https://github.com/grpc/grpc/blob/4567af504ed53cc8398e53bf1efd23f753d43bb8/src/core/ext/filters/client_channel/lb_policy.h#L178-L201),
[Go](https://github.com/grpc/grpc-go/blob/01ed64857e3146000ec99cdea4f2932204f17cdd/balancer/balancer.go#L256-L269)
), the picker should add a
callback for atomically decrementing the subchannel counter once the RPC finishes (regardless of `Status` code).
In some cases, these callbacks may not be called.
TODO: figure out how to avoid outstanding request counters that might not fully drain.

### Temporary environment variable protection

During initial development, this feature will be enabled via the
`GRPC_XDS_EXPERIMENTAL_ENABLE_LEAST_REQUEST` environment variable.  This
environment variable protection will be removed once the feature has
proven stable.

## Rationale

### Support for Weighted Least Request

While also supporting the "*all weights not equal*" (Weighted Least Request) case
[from Envoy](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/load_balancers#weighted-least-request)
would enable more advanced use cases, there are some missing pieces in gRPC that
are required for that to be technically feasible.
For example, gRPC ignores the per-endpoint `load_balancing_weight` field as a result
of the implementation described by
[gRFC A27: xDS-based Global Load Balancing](https://github.com/grpc/proposal/blob/master/A27-xds-global-load-balancing.md).
Later in the future when [ORCA](https://docs.google.com/document/d/1NSnK3346BkBo1JUU3I9I5NYYnaJZQPt8_Z_XCBCI3uA/edit)
is supported, gRPC may be able to locally provide per-endpoint weights that could be
used in a weighted least request policy implementation.

### Allow picking same candidate multiple times

One possible downside of the Envoy implementation, is that the random selection will allow a subchannel to
be sampled multiple times and thereby be unavoidably picked with some probability.
This means that even a frozen endpoint would never fully be excluded by the picker.
There exists a
[closed PR to Envoy](https://github.com/envoyproxy/envoy/pull/11006)
that attempted to solve this.
In that PR, there were some questions around whether there would be any significant improvement for >5 endpoints.

## Implementation

[Spotify](https://github.com/orgs/spotify/teams/fabric) is able to contribute
towards a Java implementation.
Implementation of remaining gRPC languages is left for respective gRPC team.
