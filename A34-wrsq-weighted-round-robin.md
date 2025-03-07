`wrsq_weighted_round_robin` lb_policy for per endpoint `load_balancing_weight` from `ClusterLoadAssignment` response
----
* Author(s): Yi-Shu Tai (echo80313@gmail.com)
* Approver: markdroth
* Status: In Review
* Implemented in: N/A
* Last updated: 2021-02-10
* Discussion at: https://groups.google.com/g/grpc-io/c/j76bnPgpHYo

## Abstract
This proposal is for carrying per endpoint weight in address attribute from [`ClusterLoadAssignment`](https://github.com/envoyproxy/envoy/blob/2dcf20f4baf5de71ba1d8afbd76b0681613e13f2/api/envoy/config/endpoint/v3/endpoint.proto#L34) and introducing `wrsq_weighted_round_robin` policy based on weighted random selection queue (WRSQ) for taking advantage of the information which per endpoint weight provides.

This proposal is based on [A27: xDS-Based Global Load Balancing](https://github.com/grpc/proposal/blob/master/A27-xds-global-load-balancing.md).

## Background
[A27: xDS-Based Global Load Balancing](https://github.com/grpc/proposal/blob/master/A27-xds-global-load-balancing.md) describes resolver/LB architecture and xDS client behavior. This proposal specifically extends the behavior of EDS section in [A27: xDS-Based Global Load Balancing](https://github.com/grpc/proposal/blob/master/A27-xds-global-load-balancing.md). We pass [`load_balancing_weight`](https://github.com/envoyproxy/envoy/blob/2dcf20f4baf5de71ba1d8afbd76b0681613e13f2/api/envoy/config/endpoint/v3/endpoint_components.proto#L108) of [`LbEndpoint`](https://github.com/envoyproxy/envoy/blob/2dcf20f4baf5de71ba1d8afbd76b0681613e13f2/api/envoy/config/endpoint/v3/endpoint_components.proto#L76) to `lb_policy` by carrying per endpoint `load_balancing_weight` in per-address attribute. `lb_policy` can make use of the information provided by per endpoint `load_balancing_weight` for better load balancing.

To best utilize the information, we also propose a new `lb_policy`, `wrsq_weighted_round_robin` which works on `LbEndpoint`s within same [`LocalityLbEndpoints`](https://github.com/envoyproxy/envoy/blob/2dcf20f4baf5de71ba1d8afbd76b0681613e13f2/api/envoy/config/endpoint/v3/endpoint_components.proto#L116).

This proposal has two parts. The first part is the new `lb_policy`, `wrsq_weighted_round_robin`. The second part discusses how we handle per endpoint `load_balancing_weight` from `ClusterLoadAssignment` response.

### Related Proposals:
* [A27: xDS-Based Global Load Balancing](https://github.com/grpc/proposal/blob/master/A27-xds-global-load-balancing.md).

## Proposal
The proposal is to carry [`load_balancing_weight`](https://github.com/envoyproxy/envoy/blob/2dcf20f4baf5de71ba1d8afbd76b0681613e13f2/api/envoy/config/endpoint/v3/endpoint_components.proto#L108) of [`LbEndpoint`](https://github.com/envoyproxy/envoy/blob/2dcf20f4baf5de71ba1d8afbd76b0681613e13f2/api/envoy/config/endpoint/v3/endpoint_components.proto#L76) from [`ClusterLoadAssignment`](https://github.com/envoyproxy/envoy/blob/2dcf20f4baf5de71ba1d8afbd76b0681613e13f2/api/envoy/config/endpoint/v3/endpoint.proto#L34) response to `lb_policy` and introduce a new `wrsq_weighted_round_robin` policy based on weighted random selection queue (WRSQ) algorithm picker. Each endpoint will get fraction equal to the [`load_balancing_weight`](https://github.com/envoyproxy/envoy/blob/2dcf20f4baf5de71ba1d8afbd76b0681613e13f2/api/envoy/config/endpoint/v3/endpoint_components.proto#L108) for the [`LbEndpoint`](https://github.com/envoyproxy/envoy/blob/2dcf20f4baf5de71ba1d8afbd76b0681613e13f2/api/envoy/config/endpoint/v3/endpoint_components.proto#L76) divided by the sum of the `load_balancing_weight` of all `LbEndpoint` within the same [`LocalityLbEndpoints`](https://github.com/envoyproxy/envoy/blob/2dcf20f4baf5de71ba1d8afbd76b0681613e13f2/api/envoy/config/endpoint/v3/endpoint_components.proto#L116) of traffic routed to the locality.

### Overview of `wrsq_weighted_round_robin` policy
`wrsq_weighted_round_robin` distribute traffic to each endpoint in a way that each endpoint will get fraction traffic equal to the weight associated with the endpoint divided by the sum of the weight of all endpoints. The core of `wrsq_weighted_round_robin` is weighted random selection queue (WRSQ) algorithm.

WRSQ scheduler keeps a FIFO queue for each unique weight among all endpoint weights. A queue weight, the endpoint weight times number of endpoints in the queue, is assigned to each of the queues.

#### Pick operation
Pick operation consists of 3 steps:
1. Select a queue with the probability based on the queue weight.
2. Pop the endpoint at the front of the selected queue, and the endpoint is returned to caller
3. Push the selected endpoint to the rear of the queue.

A queue `q_i` is selected by the probability `(queue i's weight) / (sum of all queue weight)`. `Pick` returns the endpoint at the front of the selected queue, then pushes the endpoint to the rear of the queue.

To select a queue efficiently, we can pre-compute a prefix-sum array. The 1st element of array is `w_1` * `t_1`, the 2nd is `w_1` * `t1` + `w_2` * `t2`, ... and the nth is `w_1` * `t1` + `w_2` * `t2` + ... + `w_n` * `t_n`. To select the queue with the intended probability is equivalent to generating a random nonnegative integer `x` in [0, the last element of prefix-sum array] and finding the first element of the prefix-sum array which is larger than `x`. Say the element is index `i`, the queue `i` is picked. The time complexity of picking a queue is `O(log n)` (n is the number of queues) because prefix-sum array is sorted so binary search can be used. After queue is picked, popping endpoint from the queue is `O(1)`.

#### Correctness
Assume that there are `w_1`, `w_2`,... `w_n` unique weight among endpoints and there are `t_1` endpoints with weight `w_1`, `t_2` endpoints with weight `w_2`,..., `t_n` endpoints with weight `w_n`. By the definition, WRSQ constructs `q_1` for `w_1`, `q_2` for `w_2`, ... , `q_n` for `w_n`. The weight of `q_i` is `w_i` times `t_i`. The expected time of a endpoint with weight `w_i` being picked among `m` pick operations is `m` times the probability of `q_i` being picked ( (`w_i` * `t_i`) / (`w_1` * `t1` + `w_2` * `t2` + ... + `w_n` * `t_n`) ) times (`1/t_i`) which is equal to `m`(`w_i` / (`w_1` * `t1` + `w_2` * `t2` + ... + `w_n` * `t_n`) ).

#### Building picker
There are 2 parts in building the picker.
1. Construct queues for each unique weight and push all endpoints to the corresponding queue 
2. Build prefix-sum array of queue weight

Time complexity of constructing queues is linear to the number of endpoints and building prefix-sum array is linear to number of queues.

To avoid all clients pick the same endpoint synchronously, we need to randomly shuffle the order of endpoints before pushing into the queue.

#### Weight of each endpoint
`wrsq_weighted_round_robin` needs extra information `weight` of each endpoint. We pass weight to `lb_policy` as per address attribute.


#### Subchannel connectivity management
`wrsq_weighted_round_robin` proactively monitors the connectivity of each subchannel. `wrsq_weighted_round_robin` always tries to keep one connection open to each address in the address list at all times. When `wrsq_weighted_round_robin` is first instantiated, it immediately tries to connect to all addresses, and whenever a subchannel becomes disconnected, it immediately tries to reconnect.

#### Service Config
The service config for the `wrsq_weighted_round_robin` LB policy is an empty proto message
```
{
  load_balancing_config: {wrsq_weighted_round_robin: {}}
}
```

### Handling per endpoint `load_balancing_weight` from `ClusterLoadAssignment` response
This part extends the behavior of EDS section in [A27: xDS-Based Global Load Balancing](https://github.com/grpc/proposal/blob/master/A27-xds-global-load-balancing.md). Instead of discarding the per endpoint `load_balancing_weight`, we want to add it to per-address attribute and pass it along to `lb_policy`.

#### `lb_policy` for per endpoint `load_balancing_weight` from `ClusterLoadAssignment`
When the `lb_policy` field in CDS response is `ROUND_ROBIN`, we use `wrsq_weighted_round_robin` as the `lb_policy`.

We only accept `ROUND_ROBIN` as `lb_policy` in CDS response per [A27: xDS-Based Global Load Balancing](https://github.com/grpc/proposal/blob/master/A27-xds-global-load-balancing.md). Therefore, `wrsq_weighted_round_robin` will always be used.

#### On update of `ClusterLoadAssignment`
When an EDS update is received, an update will be sent to the `lb_policy`. The `lb_policy` will create a new picker. Only positive integer will be accepted as valid endpoint weight, otherwise will be assigned lowest valid weight `1`.

#### NOTE
- `wrsq_weighted_round_robin` should always be updated to the latest `ClusterLoadAssignment`. It's xDS server's responsibility to maintain consistency.

## Rationale
The alternative algorithm for weighted round robin is earliest deadline first scheduling algorithm [EDF](https://en.wikipedia.org/wiki/Earliest_deadline_first_scheduling). The reasons we pick WRSQ are
1. Consistent with Envoy [issue 14597](https://github.com/envoyproxy/envoy/issues/14597)
2. It's easier to avoid all clients perform synchronized pick the same endpoint especially the first pick.

## Implementation

N/A

## Open issues (if applicable)
