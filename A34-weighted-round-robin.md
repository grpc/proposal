`weighted_round_robin` lb_policy for per endpoint `load_balancing_weight` from `ClusterLoadAssignment` response
----
* Author(s): Yi-Shu Tai (echo80313@gmail.com)
* Approver: a11r, markdroth
* Status: In Review
* Implemented in: N/A
* Last updated: 2020-09-20
* Discussion at: https://groups.google.com/g/grpc-io/c/j76bnPgpHYo

## Abstract
This proposal is for carrying per endpoint weight in address attribute from [`ClusterLoadAssignment`](https://github.com/envoyproxy/envoy/blob/2dcf20f4baf5de71ba1d8afbd76b0681613e13f2/api/envoy/config/endpoint/v3/endpoint.proto#L34) and introducing `weighted_round_robin` policy based on [earliest deadline first scheduling algorithm](https://en.wikipedia.org/wiki/Earliest_deadline_first_scheduling) for taking advantage of the information which per endpoint weight provides.

This proposal is based on [A27: xDS-Based Global Load Balancing](https://github.com/grpc/proposal/blob/master/A27-xds-global-load-balancing.md).

## Background
[A27: xDS-Based Global Load Balancing](https://github.com/grpc/proposal/blob/master/A27-xds-global-load-balancing.md) describes resolver/LB architecture and xDS client behavior. This proposal specifically extends the behavior of EDS section in [A27: xDS-Based Global Load Balancing](https://github.com/grpc/proposal/blob/master/A27-xds-global-load-balancing.md). We pass [`load_balancing_weight`](https://github.com/envoyproxy/envoy/blob/2dcf20f4baf5de71ba1d8afbd76b0681613e13f2/api/envoy/config/endpoint/v3/endpoint_components.proto#L108) of [`LbEndpoint`](https://github.com/envoyproxy/envoy/blob/2dcf20f4baf5de71ba1d8afbd76b0681613e13f2/api/envoy/config/endpoint/v3/endpoint_components.proto#L76) to `lb_policy` by carrying per endpoint `load_balancing_weight` in per-address attribute. `lb_policy` can make use of the information provided by per endpoint `load_balancing_weight` for better load balancing.

To best utilize the information, we also propose a new `lb_policy`, `weighted_round_robin` which works on `LbEndpoint`s within same [`LocalityLbEndpoints`](https://github.com/envoyproxy/envoy/blob/2dcf20f4baf5de71ba1d8afbd76b0681613e13f2/api/envoy/config/endpoint/v3/endpoint_components.proto#L116).

This proposal has two parts. The first part is the new `lb_policy`, `weighted_round_robin`. The second part discuss how we handle per endpoint `load_balancing_weight` from `ClusterLoadAssignment` response.

### Related Proposals:
* [A27: xDS-Based Global Load Balancing](https://github.com/grpc/proposal/blob/master/A27-xds-global-load-balancing.md).

## Proposal
The proposal is to carry [`load_balancing_weight`](https://github.com/envoyproxy/envoy/blob/2dcf20f4baf5de71ba1d8afbd76b0681613e13f2/api/envoy/config/endpoint/v3/endpoint_components.proto#L108) of [`LbEndpoint`](https://github.com/envoyproxy/envoy/blob/2dcf20f4baf5de71ba1d8afbd76b0681613e13f2/api/envoy/config/endpoint/v3/endpoint_components.proto#L76) from [`ClusterLoadAssignment`](https://github.com/envoyproxy/envoy/blob/2dcf20f4baf5de71ba1d8afbd76b0681613e13f2/api/envoy/config/endpoint/v3/endpoint.proto#L34) response to `lb_policy` and introduce a new `weighted_round_robin` policy based on Earliest deadline first scheduling algorithm picker [EDF](https://en.wikipedia.org/wiki/Earliest_deadline_first_scheduling). Each endpoint will get fraction equal to the [`load_balancing_weight`](https://github.com/envoyproxy/envoy/blob/2dcf20f4baf5de71ba1d8afbd76b0681613e13f2/api/envoy/config/endpoint/v3/endpoint_components.proto#L108) for the [`LbEndpoint`](https://github.com/envoyproxy/envoy/blob/2dcf20f4baf5de71ba1d8afbd76b0681613e13f2/api/envoy/config/endpoint/v3/endpoint_components.proto#L76) divided by the sum of the `load_balancing_weight` of all `LbEndpoint` within the same [`LocalityLbEndpoints`](https://github.com/envoyproxy/envoy/blob/2dcf20f4baf5de71ba1d8afbd76b0681613e13f2/api/envoy/config/endpoint/v3/endpoint_components.proto#L116) of traffic routed to the locality.

### Overview of `weighted_round_robin` policy
`weighted_round_robin` distribute traffic to each endpoint in a way that each endpoint will get fraction traffic equal to the weight associated with the endpoint divided by the sum of the weight of all endpoints. The core of `weighted_round_robin` is [EDF](https://en.wikipedia.org/wiki/Earliest_deadline_first_scheduling) picker.

#### Weight of each endpoint
`weighted_round_robin` needs extra information `weight` of each endpoint. We pass weight to `lb_policy` as per address attribute.

#### Overview of EDF scheduler
EDF picker maintains a priority queue of `EdfEntry`. The key of the priority queue is `(deadline, order_offset)` pair. On the top of the queue, it’s the entry with lowest `deadline`. `order_offset` is the tie breaker when two entries have same deadline to maintain FIFO order. If there is a tie on deadline of two entries, the one with smaller `order_offset` will have higher priority.

Proposed `EdfEntry`
```
struct EdfEntry {
    // primary key for the priority queue. The entry with least deadline is the top of the queue.
    double deadline;

    // secondary key for the priority queue. Used as a tiebreaker for same deadline to
    // maintain FIFO order. If there is a tie on deadline of two entries, the one with
    // smaller `order_offset` will have higher priority. `order_offset` is assigned to this
    // entry on constructing the priority queue for the first time and it's immutable.
    // Also the `order_offset` assigned to entries is strictly increasing, in other words,
    // no two entries have same `order_offset`.
    uint64 order_offset;

    // `load_balancing_weight` of this endpoint from address attribute of this endpoint.
    double weight;

    // Subchannel data structure of this endpoint.
    Subchannel subchannel;
}
```
Initialization
- At the very beginning, `deadline` of an entry `e` is equal to `1/e.weight`.
- We assign `order_offset` to each entry while constructing the priority queue. `order_offset` assigned to an entry is distinct and nonnegative interger. During the whole lifecycle of this picker, `order_offset` of an entry is unchanged.

Pick
- On each call to the `Pick`, EDF picker picks the entry `e` on the top of the queue, returns the subchannel associated with the entry. After that, picker updates the `deadline` of `e` to `e.deadline + 1/weight` and either performs a pop and push the entry back to the queue or key increase operation.

Notes
- If all endpoints have the same `load_balancing_weight`, `EDF` picker degenerates to `round_robin` picker. The order of picked subschannel is purely decided by `order_offset`. It's easier to reason and consistent with envoy.
- Endpoints do not have `load_balancing_weight` is assigned to 1 (the smallest possible weight). This is to be consistent with the [behavior of envoy on missing weight assignment](https://github.com/envoyproxy/envoy/blob/5d95032baa803f853e9120048b56c8be3dab4b0d/source/common/upstream/upstream_impl.cc#L359)

#### Service Config
The service config for `weighted_round_robin` is very similar to `round_robin`
```
{
  load_balancing_config: { weighted_round_robin: {}}
}
```

### Handling per endpoint `load_balancing_weight` from `ClusterLoadAssignment` response

This part extends the behavior of EDS section in [A27: xDS-Based Global Load Balancing](https://github.com/grpc/proposal/blob/master/A27-xds-global-load-balancing.md). Instead of discarding the per endpoint `load_balancing_weight`, we want to add it to per-address attibute and pass it along to `lb_policy`.

#### `lb_policy` for per endpoint `load_balancing_weight` from `ClusterLoadAssignment`
When the `lb_policy` field in CDS response is `ROUND_ROBIN`, we use `weighted_round_robin` as the `lb_policy`.

As of today, we only accept `ROUND_ROBIN` as `lb_policy` in CDS response per [A27: xDS-Based Global Load Balancing](https://github.com/grpc/proposal/blob/master/A27-xds-global-load-balancing.md). Therefore, `weighted_round_robin` will always be used.

#### On update of `ClusterLoadAssignment`
When an EDS update is received, an update will be sent to the `lb_policy`. The `lb_policy` will create a new picker. This is slightly Different from [Envoy](https://github.com/envoyproxy/envoy/blob/51551ae944c642e6fc61563cbea8653087e70f1f/source/common/upstream/load_balancer_impl.cc#L733-L737). We'd like to udpate EDF priority queue so that new weights applied immediately even endpoints list is not changed.

#### NOTE
- `weighted_round_robin` should always be updated to the lastest `ClusterLoadAssignment`. It's xDS server's responsibility to maintain consistency.

## Rationale

Several applications can be built upon this feature, e.g. utilization load balancing, blackhole erroring endpoints, load testing,... etc.

The reason to refresh EDF picker even there is only weight change on some endpoints which is different from envoy is because we'd like real time traffic shift for use cases like load testing, blackhole erroring endpoints.

The reasons to introduce a new algorithm instead of re-using the same algorithm of `weighted_target` policy are
- [EDF](https://en.wikipedia.org/wiki/Earliest_deadline_first_scheduling) maintains FIFO order for endpoints with same weight which is easier to reason.
- We want to be consistent with the behavior of Envoy.

## Implementation

N/A

## Open issues (if applicable)
- Do we need a way to opt out WRR even per endpoint weight is assigned by ClusterLoadAssignment?