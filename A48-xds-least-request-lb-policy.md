A48: xDS Least Request LB Policy
----
* Author(s): erikjoh & [fabric](https://github.com/orgs/spotify/teams/fabric)
* Approver: markdroth
* Status: Implementation in progress
* Implemented in: Java in progress
* Last updated: 2021-12-02
* Discussion at: https://groups.google.com/g/grpc-io/c/4qycdcFfMUs

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
[`choice_count`](https://github.com/envoyproxy/envoy/blob/2e0ad0beb9625ccedaec8f3093aa5e6e1dc484f4/api/envoy/config/cluster/v3/cluster.proto#L387)
field the same way that Envoy does with the exception of allowed values
(More details further down in [LB Policy Config](#lb-policy-config)).
The
[`active_request_bias`](https://github.com/envoyproxy/envoy/blob/2e0ad0beb9625ccedaec8f3093aa5e6e1dc484f4/api/envoy/config/cluster/v3/cluster.proto#L412)
field will be ignored as it is only used in the "*all weights not equal*" (weighted least request)
case which will not be supported as part of this proposal.
The
[`slow_start_config`](https://github.com/envoyproxy/envoy/blob/2e0ad0beb9625ccedaec8f3093aa5e6e1dc484f4/api/envoy/config/cluster/v3/cluster.proto#L416)
field will also be ignored as it requires the weighted least request algorithm.

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
Envoy currently supports any `choice_count` value greater than or equal to two.
In the `least_request_experimental` policy however, only an effective `choice_count` in the
range `[2, 10]` will be supported. If a `LeastRequestLoadBalancingConfig` with a `choice_count > 10`
is received, the `least_request_experimental` policy will set `choice_count = 10`.

##### Subchannel connectivity management

The subchannel connectivity management implemented in the `round_robin` policy will be re-used
for `least_request_experimental`.
Implementations may break out the connectivity management logic and have `round_robin` and `least_request_experimental`
inherit it instead. This would be similar to the "base balancer" already implemented in grpc-go
[here](https://github.com/grpc/grpc-go/blob/03268c8ed29e801944a2265a82f240f7c0e1b1c3/balancer/base/balancer.go).

Just like the `round_robin` policy, the `least_request_experimental` policy will attempt to keep a subchannel connected
to every address in the address list at all times. If a subchannel becomes disconnected, the policy will trigger
re-resolution and will attempt to reconnect to the address, subject to connection back-off.

##### Aggregated Connectivity State

The `least_request_experimental` policy will use the same heuristic for determining the aggregated
connectivity state as the `round_robin` policy.
The general rules in order of precedence are:

1. If there is at least one subchannel with the state `READY`, the aggregated connectivity state is `READY`.
2. If there is at least one subchannel with the state `CONNECTING` or `IDLE`,
   the aggregated connectivity state is `CONNECTING`.
3. If all subchannels have the state `TRANSIENT_FAILURE`, the aggregated connectivity state is `TRANSIENT_FAILURE`.

For the purpose of evaluating the aggregated connectivity state in the rules above, a subchannel that enters
`TRANSIENT_FAILURE` should be considered to stay in `TRANSIENT_FAILURE` until it reports `READY`. If a subchannel is
bouncing back and forth between `TRANSIENT_FAILURE` and `CONNECTING` while it attempts to re-establish a connection,
we will consider it to be in state `TRANSIENT_FAILURE` for the purpose of evaluating the aggregation rules above.

##### Outstanding request counters

The `least_request_experimental` policy will associate each subchannel to an outstanding request counter
(`active_requests`). Counters will be local to each instance of the `least_request_experimental` policy and
thereby not shared in cases where subchannels are used across multiple LB instances.
Implementations may either wrap subchannels or utilize custom subchannel attributes to provide counter access
in the picker.
These counters should additionally share the lifecycle with its corresponding subchannel.
The counter for a subchannel should be atomically incremented by one after it has been successfully
picked by the picker. Due to language-specifics, the increment may occur either before or during stream creation.
In the `PickResult` for each language-specific implementation (
[Java](https://github.com/grpc/grpc-java/blob/1f90e0e28d5628195cb1f861b73e45ed003f2973/api/src/main/java/io/grpc/LoadBalancer.java#L561-L571),
[C++](https://github.com/grpc/grpc/blob/c1089d2964528b52a25c18d748910658e4762d40/src/core/ext/filters/client_channel/lb_policy.h#L201-L217),
[Go](https://github.com/grpc/grpc-go/blob/01ed64857e3146000ec99cdea4f2932204f17cdd/balancer/balancer.go#L256-L269)
), the picker should add a
callback for atomically decrementing the subchannel counter once the RPC finishes (regardless of `Status` code).
This approach closely resembles the implementation for outstanding request counters already present in
the `cluster_impl_experimental` policy (
[Java](https://github.com/grpc/grpc-java/blob/0000cba665c69958355b639474c7387d98afcc79/xds/src/main/java/io/grpc/xds/ClusterImplLoadBalancer.java#L282-L379),
[C++](https://github.com/grpc/grpc/blob/79d684529d9db60aa2c5e82deb43e297a6818cfb/src/core/ext/filters/client_channel/lb_policy/xds/xds_cluster_impl.cc#L285-L354),
[Go](https://github.com/grpc/grpc-go/blob/03268c8ed29e801944a2265a82f240f7c0e1b1c3/xds/internal/balancer/clusterimpl/picker.go#L102-L191)
).
This approach entails some degree of raciness which will be discussed later
(See [Outstanding request counter raciness](#outstanding-request-counter-raciness)).

##### Picker Behavior

The picker should be given a list of all subchannels with the `READY` state.
With this list, the picker should pick a subchannel based on the following pseudo-code:

```
candidate = null;
for (int i = 0; i < choiceCount; ++i) {
    sampled = subchannels[random_integer(0, subchannels.length)];
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

##### Duplicate addresses in resolution result

In gRPC, if an address is present more than once in an address list, that is intended to indicate that the address
should be more heavily weighted. However, it should not result in more than one connection to that address.

In keeping with this, the `least_request_experimental` policy will de-duplicate the addresses in the address list
it receives so that it does not create more than one connection to a given address. However, because the policy
will not initially support the weighted least request algorithm, we will ignore the additional weight given to
the address. If at some point in the future we do add support for the weighted least request algorithm, then an
address that is present multiple times in the list will be given appropriate weight in that algorithm.

### Temporary environment variable protection

During initial development, this feature will be enabled via the
`GRPC_EXPERIMENTAL_ENABLE_LEAST_REQUEST` environment variable.  This
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

### Outstanding request counter raciness

While reading the outstanding request counters of samples in the picker, previously read values may become outdated.
For cases where a high `choice_count` relative the number of endpoints is used, and/or cases with high/bursty traffic,
there may be sub-optimal behaviors where certain endpoints may get an unfair amount of traffic.
This raciness is currently accepted in the Envoy implementation.
The fix for this raciness would most likely have to be a limitation on picker concurrency.
Since such a limitation could have a serious negative impact on more normal load pattern situations,
it may be argued that it's better to accept some degree of raciness in the picker instead.

### Maximum limit for choice_count

In gRPC, there are cases where LB config cannot be trusted. This means that config may contain an unreasonable
(for example `UINT_MAX`) value for the `choice_count` setting.
Thereby, it would be beneficial for gRPC to introduce a max value for the setting.
The default value `choice_count = 2` comes from the P2C (power of two choices) paper
([Mitzenmacher et al.](https://www.eecs.harvard.edu/~michaelm/postscripts/handbook2001.pdf)
which shows that such a configuration nearly is as good as an O(n) full scan.
With that in mind, picking a `choice_count > 2` would probably not add much benefit in many cases.
By introducing a limit as `2 <= choice_count <= 10`, we disallow potentially dangerous LB configurations
while still allowing some degree of freedom that could be beneficial in some edge-cases.

## Implementation

[Spotify](https://github.com/orgs/spotify/teams/fabric) is able to contribute
towards a Java implementation.
Implementation of remaining gRPC languages is left for respective gRPC team
or any other contributors.
