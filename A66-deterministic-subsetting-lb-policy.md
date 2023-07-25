A58: `deterministic_subsetting` LB policy.
----
* Author(s): @s-matyukevich
* Approver: 
* Status: Draft
* Implemented in: 
* Last updated: [Date]
* Discussion at:

## Abstract

Add support for the `deterministic_subsetting` load balancing configured via xDS.

## Background

Currently, gRPC is lacking a way to select a subset of backend endpoints and load-balance requests between them. Mostly people are using 2 extremes: `pick_first` and `round_robin`. With`pick_first` the resulting load distribution on the backend servers ends up being very uneven, especially during server rollouts. With `round_rogin` gRPC generates a lot of unnecessary connections, that eat up resource on both clients and servers. The proposed LB policy provides a middlegroud between those 2 extremes.

### Related Proposals: 

## Proposal

Introduce a new LB policy `deterministic_subsetting`. This policy selects a subset of addresses and passes them to the clild LB policy in such a way that:
* Every client is connected to at most N backends. If there are not enough backends the policy falls back to its child policy.
* The number of resulting connections per backend is as balanced as possible.
* Every client is connected to a different subset of backends. 

### Subsetting algorithm

The LB policy will implement the algorithm desribed in [Site Reliability Engineering: How Google Runs Production Systems, Chapter 20](https://sre.google/sre-book/load-balancing-datacenter/#a-subset-selection-algorithm-deterministic-subsetting-eKsdcaUm) with the additional modification mentioned in the "Deterministic subsetting" section of the [Reinventing Backend Subsetting at Google](https://queue.acm.org/detail.cfm?id=3570937) paper. Here is the relevant quote:

```
This is the algorithm as previously described in Site Reliability Engineering: How Google Runs Production Systems, Chapter 20) but one improvement remains that can be made by balancing the leftover tasks in each group. The simplest way to achieve this is by choosing (before shuffling) which tasks will be leftovers in a round-robin fashion. For example, the first group of frontend tasks would choose {0, 1} to be leftovers and then shuffle the remaining tasks to get subsets {8, 3, 9, 2} and {4, 6, 5, 7}, and then the second group of frontend tasks would choose {2, 3} to be leftovers and shuffle the remaining tasks to get subsets {9, 7, 1, 6} and {0, 5, 4, 8}. This additional balancing ensures that all backend tasks are evenly excluded from consideration, producing a better distribution.
```

### LB Policy Config and Parameters

The `deterministic_subsetting` LB policy config will be as follows.

```
```

## Rationale

[A discussion of alternate approaches and the trade offs, advantages, and disadvantages of the specified approach.]


## Implementation

[A description of the steps in the implementation, who will do them, and when.  If a particular language is going to get the implementation first, this section should list the proposed order.]

## Open issues (if applicable)

[A discussion of issues relating to this proposal for which the author does not know the solution. This section may be omitted if there are none.]