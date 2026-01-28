A113: pick_first: Weighted Random Shuffling
----
* Author(s): Alex Polcyn (@apolcyn)
* Approver: Mark Roth (@markdroth), Eric Anderson (@ejona86), Doug Fawley (@dfawley), Easwar Swaminathan (@easwars)
* Status: In Review
* Implemented in: <language, ...>
* Last updated: Jan 26, 2026
* Discussion at: https://groups.google.com/g/grpc-io/c/iCsweGDmUU4

## Abstract

Support weighted random shuffling in the pick first LB policy.

## Background

The pick first LB policy currently supports random shuffling. A primary intention of the feature
is for load balancing, however it does not take (possibly present) locality or endpoint weights
into account. Naturally this can lead to skewed load distribution and hotspots, when the load
balancing control plane delivers varied weights and expects them to be followed.


### Related Proposals: 
* [A62](https://github.com/grpc/proposal/blob/master/A62-pick-first.md): pick_first: sticky TRANSIENT_FAILURE and address order randomization
* [A42](https://github.com/grpc/proposal/blob/master/A42-xds-ring-hash-lb-policy.md) xDS Ring Hash LB Policy

## Proposal

### Changes within Pick First

Modify behavior of pick_first when the `shuffle_address_list` option is set, and
perform a weighted random sort *based on per-endpoint weights*. To do this, we will
use the [Weighted Random Sampling](https://utopia.duth.gr/~pefraimi/research/data/2007EncOfAlg.pdf) algorithm
proposed by Efraimidis, Spirakis:

1) Assign a key to each endpoint, `u ^ (1 / weight)`, where `u` is a uniform random number in `(0, 1)` and weight
is the weight of the endpoint (as present in a weight attribute). Default `weight` to 1 if no weight attribute is
present.

2) Sort endpoints by key in *descending* order.

Note: the paper suggests `u` be in `(0, 1)` *exclusive*. Random numbers *on* zero or one effectively
drop their weight. Also, technically zero will not transform to the exponential distribution that we are trying
to create. However, load balancing skew introduced by such edge cases is unlikely to be noticeable, and so
implementations are free to include these bounds so long as it does not cause other problems
(e.g. crashes).


### CDS LB Policy changes: Computing Endpoint Weights

In XDS, we have a notion of both locality and endpoint weights. The expectation of the load balancing
control plane is to *first* pick locality and *second* pick endpoint. The total probability distribution
reflected by per-endpoint weights must reflect this. As such, we need to normalize locality weights within
each priority and endpoint weights within locality; the final weight provided to `pick_first` should be a
product of the two normalized weights (i.e. a logical AND of the two selection events).

The CDS LB policy currently calculates per-endpoint weight attributes. It will continue to do so however
we need to fix the mechanics: an endpoint's final weight should be a product of its *normalized* locality
weight and *normalized* endpoint weight, rather than their product outright.

Note: as a side effect this will fix per-endpoint weights in Ring Hash LB, which
[currently](https://github.com/grpc/proposal/blob/master/A42-xds-ring-hash-lb-policy.md#change-child-policy-config-generation-in-xds_cluster_resolver-policy) are a product of the initial *raw* locality and endpoint weights.
This "fix" will not require any changes within Ring Hash LB itself.

We can continue to represent weights as integers if we represent their normalized values in
fixed point UQ1.31 format. Math as follows (citation due for @ejona):

```
// To normalize:
uint32_t ONE = 1 << 31;
uint32_t weight = (uint64_t) weight * ONE / weight_sum;

// To multiply the weights for an endpoint:
weight = ((uint64_t) locality_weight * weight) >> 31;
if (weight == 0) weight = 1;
```

Note: currently we round down to zero (and then up if we hit zero).
We *could* use more accurate rounding schemes. However, rounding down
is simple and should provide enough precision for load balancing
purposes. For example, we only round down to zero if the product of
two normalized weight probabilities is less than `2 ^ -31`, this kind
of error is unlikely to cause noticeable skew in load balancing.

### Temporary environment variable protection

CDS LB policy and Pick First LB policy behavior changes will be guarded by `GRPC_EXPERIMENTAL_PF_WEIGHTED_SHUFFLING`.

This should be enabled by default, after testing.

## Rationale

CDS LB policy changes are needed to generate correct weight distributions, not only for Pick First but
also for Ring Hash.

Reasons for UQ1.31 fixed point integers:

- Predictable and acceptable bounds on precision.
- Allows us to continue representing weights as integers internally.
- Avoids risk of overflow bugs by preserving the (XDS) property that the sum of all weights within
  a "grouping" does not exceed max uint32. For example note how if we used UQ32, *after*
  normalization and multiplication a subsequent summation of endpoint weights in a locality may
  result in uint32 overflow due to contributions of rounding errors.

## Implementation

TBD

