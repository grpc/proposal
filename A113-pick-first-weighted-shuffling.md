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
perform a weighted random sort *based on per-endpoint weights*:
* Use the [Weighted Random Sampling](https://utopia.duth.gr/~pefraimi/research/data/2007EncOfAlg.pdf) algorithm
proposed by Efraimidis, Spirakis.
* Set the weight of each endpoint to `u ^ (1 / weight)`, where `u` is a uniform random number in `(0, 1)` and weight
is the weight of the endpoint (as present in a weight attribute). Default to 1 if no weight attribute is present.

### CDS LB Policy changes: Computing Endpoint Weights

In XDS, we have a notion of both locality and endpoint weights. The expectation of the load balancing
control plane is to *first* pick locality and *second* pick endpoint. The total probability distribution
reflected by per-endpoint weights must reflect this. As such, we need to normalize locality weights within
each priority and endpoint weights within locality; the final weight provided to `pick_first` should be a
product of the two normalized weights (i.e. a logical AND of the two selection events).

The CDS LB policy currently calculates per-endpoint weight attributes. It will continue to do so however
we need to fix the mechanics: an endpoint's final weight should be a product of its *normalized* locality
weight and *normalized* endpoint weight, rather than their product outright. Note: as a side effect this
will fix per-endpoint weights in Ring Hash LB, which
[currently](https://github.com/grpc/proposal/blob/master/A42-xds-ring-hash-lb-policy.md) multiply
*raw* locality and endpoint weights.

We can continue to represent weights as integers if we represent their normalized values in
fixed point Q31 format. Math as follows (citation due for @ejona):

```
// To normalize:
uint32_t ONE = 1 << 31;
uint32_t weight = (uint64_t) weight * ONE / weight_sum;

// To multiply the weights for an endpoint:
weight = ((uint64_t) locality_weight * weight) >> 31;
if (weight == 0) weight = 1;
```

### Temporary environment variable protection

CDS LB policy and Pick First LB policy behavior changes will be guarded by `GRPC_EXPERIMENTAL_PF_WEIGHTED_SHUFFLING`.

## Rationale

* CDS LB policy changes are needed to generate correct weight distributions, not only for Pick First but
  also for Ring Hash
* Using fixed point Q31 format has predictable bounds on precision, and allows us to continue representing
  weights as integers. Note our math assumes the sum of weights within a grouping does not exceed max uint32,
  which is mandated in the XDS protocol.

## Implementation

TBD

