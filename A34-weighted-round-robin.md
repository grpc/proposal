`weighted_round_robin` lb_policy for per endpoint weight from `ClusterLoadAssignment` response
----
* Author(s): Yi-Shu Tai (echo80313@gmail.com)
* Approver: a11r, Mark D. Roth (roth@google.com)
* Status: Draft
* Implemented in: N/A
* Last updated: 2020-08-16
* Discussion at: https://groups.google.com/g/grpc-io/c/j76bnPgpHYo

## Abstract

The proposal introduces `weighted_round_robin` policy based on [earliest deadline first scheduling algorithm](https://en.wikipedia.org/wiki/Earliest_deadline_first_scheduling) for per lb_endpoint weight from [`ClusterLoadAssignment`](https://github.com/envoyproxy/envoy/blob/2dcf20f4baf5de71ba1d8afbd76b0681613e13f2/api/envoy/config/endpoint/v3/endpoint.proto#L34).

This proposal is based on [A27: xDS-Based Global Load Balancing](https://github.com/grpc/proposal/blob/master/A27-xds-global-load-balancing.md).

## Background

[A27: xDS-Based Global Load Balancing](https://github.com/grpc/proposal/blob/master/A27-xds-global-load-balancing.md) describes resolver/LB architecture and xDS client behavior. This proposal specifically extends the behavior of EDS to take [`load_balancing_weight`](https://github.com/envoyproxy/envoy/blob/2dcf20f4baf5de71ba1d8afbd76b0681613e13f2/api/envoy/config/endpoint/v3/endpoint_components.proto#L108) of [`LbEndpoint`](https://github.com/envoyproxy/envoy/blob/2dcf20f4baf5de71ba1d8afbd76b0681613e13f2/api/envoy/config/endpoint/v3/endpoint_components.proto#L76) into account, and perform `weighted_round_robin` policy on `LbEndpoint`s within same [`LocalityLbEndpoints`](https://github.com/envoyproxy/envoy/blob/2dcf20f4baf5de71ba1d8afbd76b0681613e13f2/api/envoy/config/endpoint/v3/endpoint_components.proto#L116).


### Related Proposals:
* [A27: xDS-Based Global Load Balancing](https://github.com/grpc/proposal/blob/master/A27-xds-global-load-balancing.md).

## Proposal

The proposal is to carry [`load_balancing_weight`](https://github.com/envoyproxy/envoy/blob/2dcf20f4baf5de71ba1d8afbd76b0681613e13f2/api/envoy/config/endpoint/v3/endpoint_components.proto#L108) of [`LbEndpoint`](https://github.com/envoyproxy/envoy/blob/2dcf20f4baf5de71ba1d8afbd76b0681613e13f2/api/envoy/config/endpoint/v3/endpoint_components.proto#L76) from [`ClusterLoadAssignment`](https://github.com/envoyproxy/envoy/blob/2dcf20f4baf5de71ba1d8afbd76b0681613e13f2/api/envoy/config/endpoint/v3/endpoint.proto#L34) response to `lb_policy` and implement a `weighted_round_robin` policy based on Earliest deadline first scheduling algorithm picker [EDF](https://en.wikipedia.org/wiki/Earliest_deadline_first_scheduling). Each endpoint will get fraction equal to the [`load_balancing_weight`](https://github.com/envoyproxy/envoy/blob/2dcf20f4baf5de71ba1d8afbd76b0681613e13f2/api/envoy/config/endpoint/v3/endpoint_components.proto#L108) for the [`LbEndpoint`](https://github.com/envoyproxy/envoy/blob/2dcf20f4baf5de71ba1d8afbd76b0681613e13f2/api/envoy/config/endpoint/v3/endpoint_components.proto#L76) divided by the sum of the `load_balancing_weight` of all `LbEndpoint` within the same [`LocalityLbEndpoints`](https://github.com/envoyproxy/envoy/blob/2dcf20f4baf5de71ba1d8afbd76b0681613e13f2/api/envoy/config/endpoint/v3/endpoint_components.proto#L116) of traffic routed to the locality.

### Overview of `weighted_round_robin` policy

`weighted_round_robin` policy is powered by [EDF](https://en.wikipedia.org/wiki/Earliest_deadline_first_scheduling) picker. EDF picker maintains a priority queue of `EdfEntry`. On the top of the queue, it’s the entry with lowest `deadline`. `order_offset` is the tie breaker when two entries have same deadline to maintain FIFO order.

Proposed `EdfEntry`
```
struct EdfEntry {
    // primary key for the priority queue. The entry with least deadline is the top of the queue.
    double deadline;
    // secondary key for the priority queue. Used as a tiebreaker for same deadline to maintain FIFO order.
    uint64 order_offset;
    // `load_balancing_weight` of this endpoint from the latest `ClusterLoadAssignment`.
    double weight;
    // Subchannel data structure of this endpoint.
    Subchannel subchannel;
}
```

- On each call to the `Pick`, EDF picker picks the entry `e` on the top of the queue, returns the subchannel associated with the entry. After that, picker updates the `deadline` of `e` to `e.deadline + 1/weight` and either performs a pop and push the entry back to the queue or key increase operation.
- `weighted_round_robin` updates the entries in EDF priority queue on any change of subchannel `ConnnectiveState` or endpoint `load_balancing_weight`.
- If all endpoints have the same `load_balancing_weight`, `EDF` picker degenerates to `round_robin` picker. It's easier to reason and consistent with envoy.
- Endpoints do not have `load_balancing_weight` assigned are discarded.
- `weighted_round_robin` should always be updated to the lastest `ClusterLoadAssignment`. It's xDS server's responsibility to maintain consistency.

#### EDF picker interface
```
/*
Add new SubChannel to EDF picker.

weighted_round_robin sees a new SubChannel in the most recent ClusterLoadAssignment or ConnectivityState of SubChannel is READY again.
*/
Add(SubChannel, weight)

/*
Remove Subchannel from EDF picker.

Subchannel is removed from the most recent ClusterLoadAssignment comparing to last ClusterLoadAssignment or
ConnectivityState of SubChannel becomes not READY.
*/
Remove(SubChannel)

/*
Update the weight of SubChannel

There is weight change on Subchannel in the most recent ClusterLoadAssignment comparing to last ClusterLoadAssignment
*/
Update(SubChannel, newWeight)

/*
`Pick` picks the EdfEntry on top of the queue, returns the underlying SubChannel and updates deadline of the EdfEntry.
*/
PickResult Pick(PickArgs args)
```

### On new `ClusterLoadAssignment`

#### endpoints have weight change
- Update EDF priority queue by updating entries have weight change.
- Different from [Envoy](https://github.com/envoyproxy/envoy/blob/51551ae944c642e6fc61563cbea8653087e70f1f/source/common/upstream/load_balancer_impl.cc#L733-L737), we'd like to udpate EDF priority queue so that new weights applied immediately even endpoints list is not changed.

#### endpoints list change
- Udpate EDF priority queue by adding/removing entries to reflect the change immediately.

### On endpoint ConnectiveState Update
- Udpate EDF priority queue by adding/removing entries to reflect the change immediately.

## Rationale

Several applications can be built upon this feature, e.g. utilization load balancing, blackhole erroring endpoints, load testing,... etc.

The reason to refresh EDF picker even there is only weight change on some endpoints which is different from envoy is because we'd like real time traffic shift for use cases like load testing, blackhole erroring endpoints.

The reasons to introduce a new algorithm instead of using existing `weighted_target` policy are
- [EDF](https://en.wikipedia.org/wiki/Earliest_deadline_first_scheduling) maintains FIFO order for endpoints with same weight which is easier to reason.
- We want to be consistent with the behavior of Envoy.

## Implementation

N/A

## Open issues (if applicable)

- Replace `round_robin` with `weighted_round_robin`?
- Following first issue, if not, do we want to use `weighted_round_robin` as default lb_policy for eDS response?