A66: `deterministic_subsetting` LB policy.
----
* Author(s): @s-matyukevich, @joybestourous
* Approver: 
* Status: Draft
* Implemented in: Go, Java
* Last updated: [Date]
* Discussion at:

## Abstract

Add support for the `deterministic_subsetting` load balancing policy configured via xDS.

## Background

Currently, gRPC is lacking a way to select a subset of endpoints available from the resolver and load-balance requests between them. Out of the box, users have the choice between two extremes: `pick_first` which sends all requests to one random backend, and `round_robin` which sends requests to all available backends. `pick_first` has poor connection balancing when the number of client is not much higher than the number of servers because of the birthday paradox. The problem is exacerbated during rollouts because `pick_first` does not change endpoint on resolver updates if the current subchannel remains `READY`. `round_robin` results in every servers having as many connections open as there are clients, which is unnecessarily costly when there are many clients, and makes local decisions load balancing (such as outlier detection) less precise.

### Related Proposals: 

## Proposal

Introduce a new LB policy, `deterministic_subsetting`. This policy selects a subset of addresses and passes them to the child LB policy in such a way that:
* Every client is connected to exactly N backends. If there are not enough backends, the policy falls back to its child policy.
* The number of resulting connections per backend is as balanced as possible.
* For any 2 given clients the probability of them connecting on the same or very similar subset of backends is as low as possible. 

### Subsetting algorithm

The LB policy will implement the algorithm described in [Site Reliability Engineering: How Google Runs Production Systems, Chapter 20](https://sre.google/sre-book/load-balancing-datacenter/#a-subset-selection-algorithm-deterministic-subsetting-eKsdcaUm) with the additional modification mentioned in the "Deterministic subsetting" section of the [Reinventing Backend Subsetting at Google](https://queue.acm.org/detail.cfm?id=3570937) paper. Here is the relevant quote:

```
This is the algorithm as previously described in Site Reliability Engineering: How Google Runs Production Systems, Chapter 20 but one improvement remains that can be made by balancing the leftover tasks in each group. The simplest way to achieve this is by choosing (before shuffling) which tasks will be leftovers in a round-robin fashion. For example, the first group of frontend tasks would choose {0, 1} to be leftovers and then shuffle the remaining tasks to get subsets {8, 3, 9, 2} and {4, 6, 5, 7}, and then the second group of frontend tasks would choose {2, 3} to be leftovers and shuffle the remaining tasks to get subsets {9, 7, 1, 6} and {0, 5, 4, 8}. This additional balancing ensures that all backend tasks are evenly excluded from consideration, producing a better distribution.
```

Here is the implementation of this algorithm in pseudocode with detailed comments. 

```
func filter_addresses(addresses, subset_size, client_index, sort_addresses)
    backend_count = addresses.length()
    if backend_count > subset_size {
        // if we don't have enough addresses to cover the desired subset size, just return the whole list
        return addresses
    }

    if sort_addresses {
        // sort address list because the algorithm assumes that the initial
        // order of the addresses is the same for every client
        addresses.sort()
    }

    // subset_count indicates how much clients we can have so that every cleint is connected to exactly 
    // subset_size distinct backends and no 2 clients connect to the same backend.
    subset_count = backend_count / subset_size
    
    // Given subset_count we not cat divide clients by rounds. Every round have exactly subset_count clients.
    // round indicates the index of the round for the current client based on its index.
    round = client_index / subset_count

    // There might be some lefover backends withing every round in cases when backend_count % subset_size != 0
    // excluded_count indicates how mumanych leftover backends we have on every round.
    excluded_count = backend_count % subset_size

    // We want to choose what backends are excluded in a round robin fashion before shufling the backends.
    // This way we make sure that for any 2 backends the difference of how many times they are excluded is at most 1.
    // excluded_start indicates the index of the first excluded backend
    excluded_start := (round * excluded_count) % backend_count
    // excluded_start indicates the index after the last excluded backend.
    // It could wrap around the end of the addresses list.
    excluded_end := (excluded_start + excluded_count) % backend_count

    if excluded_start < excluded_end {
        // excluded_end doesn't wrap, exclude addresses from the interval [excluded_start, excluded_end)
    } else {
        // excluded_end wraps around the end of the addresses list, exclude intervals [0:excluded_start] and [excluded_end, end_of_the_array)
    }

    // randomly shuffle the addresses to increase subset diversity. Use round as seed to make sure
    // that clients within the same round shuffle addresses identically. 
    addresses.shuffle(seed: round)

    // subset_id is the index for the current client withing the round
    subset_id := client_index % subset_count

    // calculate start start and end of the resulting subset 
    start = subsetId * subset_size
    end = start + subset_size
    return addresses[start: end]
}
```


### Characteristics of the selected algorithm

* Every client connects to exactly `subset_size` backends.
* Clients are evenly distributed between backends. The most connected backend receives at most 2 more connections than the least connected one. One comes from the fact that a backend can be excluded one more time than some other backend. And another one comes from the fact that there might be not enough client to completely fill the last round. 
* Backends are randomly shuffled between clients. If a small subset of backends loses connectivity, the algorithm maximizes the chances that the backends from this subset will be evenly distributed between clients. 
* The algorithm generates some, potentially significant, connection churn during backend scaling or redeployment. This might be a problem for very small subset sizes.

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
    // Every client must have a unique index.
    google.protobuf.UInt32Value client_index = 1;

    // subset_size indicates how many backends every client will be connected to.
    // Default is 10.
    google.protobuf.UInt32Value subset_size = 2;

    // sort_addresses indicates whether the LB should sort addresses by IP
    // before applying the subsetting algorithm. This might be useful
    // in cases when the resolver doesn't provide a stable order or
    // when it could order addresses differently depending on the client.
    // Default is false.
    google.protobuf.BoolValue sort_addresses = 3;

    // The config for the child policy.
    repeated LoadBalancingConfig child_policy = 4;
}
```

### Handling Parent/Resolver Updates

When the resolver updates the list of addresses, or the LB config changes, Deterministic subsetting LB will run the subsetting algorithm, described above, to filter the endpoint list. Then it will create a new resolver state with the filtered list of the addresses and pass it to the child LB. Attributes and service config from the old resolver state will be copied to the new one. 

### Handling Subchannel Connectivity State Notifications

Deterministic subsetting LB will simply redirect all requests to the child LB without doing any additional processing. This also applies to all other callbacks in the LB interface, besides the one that handles resolver and config updates (which is described in the previous section). This is possible because deterministic subsetting LB doesn't store or manage sub-connections - it acts as a simple filter on the resolver state, and that's why it can redirect all actual work to the child LB. 

### xDS Integration

Deterministic subsetting LB won't depend on xDS in any way. People may choose to initialize it by directly providing service config. If they do this, they will have to figure out how to correctly initialize and keep updating client\_index. There are ways to do that, but describing them is out of scope for this gRFC. The main point is that the LB itself will be xDS agnostic.

However, the main intended usage of this LB is via xDS. The xDS control plane is a natural place to aggregate information about all available clients and backends and assign client\_index.

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

As you can see, the fields in this policy match exactly the fields in the deterministic subsetting LB service config.

#### Integration with xDS LB Policy Registry
As described in [gRFC A52][A52], gRPC has an LB policy registry, which maintains a list of converters. Every converter translates xDS LB policy to the corresponding service config. In order to allow using the Deterministic subsetting LB policy via xDS, the only thing that needs to be done is providing a corresponding converter function. The function implementation will be trivial as the fields in the xDS LB policy will match exactly the fields in the service config.

## Rationale
### Alternatives Considered: Deterministic Aperture
Apart from Google's Deterministic Subsetting algorithm, we also considered [Twitter's Deterministic Aperture](https://patentimages.storage.googleapis.com/1a/09/5a/14392e76e5d8c5/US11119827.pdf) algorithm to handle subsetting. A critical difference in Twitter's algorithm is that it "splits" endpoints into multiple subsets. This is well demonstrated in their [ring diagrams](https://twitter.github.io/finagle/guide/ApertureLoadBalancers.html#deterministic-subsetting), where you can see a single service split into multiple subsets proportionately (i.e. 1/3 in one subset and 2/3 in the other). This ensures that even with simple child policies like `p2c`, the proportion of traffic sent to each backend across all subsets is balanced. 

Though this balancing is valuable, we opted for Google's algorithm for several reasons:
* Both algorithms are sufficient in reducing connection count, which is the primary drawback of `round_robin`. 
* Both algorithms provide better load distribution guarantees than `pick_first`. 
* When the client to backend ratio requires overlapping subsets, deterministic subsetting provides the additional guarantee of unique subsets. Because the aperture algorithm is built around a fixed ring, when clients \* aperture exceeds the number of servers, aperture takes "laps" over the ring, producing similar subsets for several clients. Deterministic subsetting strives to reduce the risk of misbehaving endpoints by distributing them more randomly across client subsets.  
* In order to take advantage of the partial weighting of backends, Twitter's algorithm would require passing weight information from parent balancer to child balancer. The appropriate balancer to handle this sort of weighting would be weighted\_round\_robin, but weighted\_round\_robin currently calculates weights based on server load alone, and it is not designed to consider this additional weighting information. We opted for the solution that allowed us to keep this existing balancer as is. Though deterministic subsetting does not guarantee even load distribution across subsets, its diverse subset paired with weighted\_round\_robin as the child policy within subsets should be sufficient for most use cases.  

## Implementation
DataDog will provide Go and Java implementations.
