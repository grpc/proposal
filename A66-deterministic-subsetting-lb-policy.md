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
This is the algorithm as previously described in Site Reliability Engineering: How Google Runs Production Systems, Chapter 20 but one improvement remains that can be made by balancing the leftover tasks in each group. The simplest way to achieve this is by choosing (before shuffling) which tasks will be leftovers in a round-robin fashion. For example, the first group of frontend tasks would choose {0, 1} to be leftovers and then shuffle the remaining tasks to get subsets {8, 3, 9, 2} and {4, 6, 5, 7}, and then the second group of frontend tasks would choose {2, 3} to be leftovers and shuffle the remaining tasks to get subsets {9, 7, 1, 6} and {0, 5, 4, 8}. This additional balancing ensures that all backend tasks are evenly excluded from consideration, producing a better distribution.
```

### Characteristics of the selected algorithm

* Every client connects to exactly `subset_size` backends.
* Clients are evenly distributed between backends. The most connected backend recieves at most 2 more connections than the least connected one.
* Backends and randomly shuffled between clients. If a small subset of backends looses connectivity, the algoitm maximizes chances that the backends from this subset will be evenly distributed between clients. 
* The algoritm generates some, potentially significant, connection charn during backends scale or redeployment. This might be a problem for very small subset sizes.

### LB Policy Config and Parameters

The `deterministic_subsetting` LB policy config will be as follows.

```
message LoadBalancingConfig {
  oneof policy {
    DeterministicSubsettingLbConfig deterministic_subsetting = 21 [json_name = "deterministic_subsetting"];
  }
}

message DeterministicSubsettingLbConfig {
  // client_index is an index within the 
  // interval [0..N-1], where N is the total number of clients.
  // Every client must have a unque index.
  google.protobuf.UInt32Value client_index = 1;
  
  // subset_size indicates to how many backends every client will be connected to.
  // Default is 10.
  google.protobuf.UInt32Value subset_size = 2;

  // sort_addresses indicates whether the LB should sort addresses by IP
  // before applying subsetting algorithm. This might be usefull 
  // in cases when the resolver doesn't provide a stable order or
  // when it could order addresses differently depending on the client.
  // Default is false.
  google.protobuf.BoolValue sort_addresses = 3;

  // The config for the child policy.
  repeated LoadBalancingConfig child_policy = 4;
}
```

### Handling Parent/Resolver Updates

When resolver updates the list of addresses, or the LB config changes  Deterministic subsetting LB will run the subsetting algorithm, described above, to filter the endpoint list. Then it will create new resolver state with the filtered list of the addresses and passes it to the clild LB. Attributes and service config from the old resolver state will be copied to the new one. 

### Handling Subchannel Connectivity State Notifications

Deterministic subsetting LB will simply redirect all requests to the child LB without doing any additional processing. This also applies to all other calllbacks in the LB interface, besides the one that hadles resolver and config updates (which is described in the previous section). This is possible because deterministic subsetting LB doesn't store or manage sub connections - it acts as a simple filter on the resolver state, and that's why it can redirect all actual work to the child LB. 

### xDS Integration

Deterministic subsetting LB won't depend on xDS in any way. People may chose to initialize it by directly providing service config. If they do this they will have to figure out how to correctly initialize and keep updating client_index. There are way how to do that, but desribing them is out of scope for this gRPC. The main point is that the LB itself will be xDS agnostic.

However, the main intended usage of this LB is via xDS. xDS control plane is a natural place to aggregate information about all avilable clients and backends and assign client_index.

#### Changes to xDS API

`deterministic_subsetting` will be added as a new LB policy.

```textproto
package envoy.extensions.load_balancing_policies.deterministic_subsetting.v3;

message DeterministicSubsetting {
  google.protobuf.UInt32Value client_index = 1;
  google.protobuf.UInt32Value subset_size = 2;
  google.protobuf.BoolValue sort_addresses = 3;
  repeated LoadBalancingConfig child_policy = 4;
}
```
As you can see, the fields in this posicy match exactly the fields in the deterministic subsetting LB service config.

#### Integration with xDS LB Policy Registry

As described in [gRFC A52][A52], grpc has an LB policy registry, which maintains alist of convertes. Every converter translates xDS LB policy to the corresponding service config. In order to allow using Deterministic subsetting LB policy via xDS the only thing that needs to be done is providing a corresponding converter function. The function implementation will be trivial as the fields in the xDS LB policy will match exactly the fields in the service config.

## Rationale

[A discussion of alternate approaches and the trade offs, advantages, and disadvantages of the specified approach.]


## Implementation

DataDog will provide Go and Java implementations.
