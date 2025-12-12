A68: Random subsetting with rendezvous hashing LB policy
----

* Author(s): @s-matyukevich
* Approver: @markdroth
* Status: Draft
* Implemented in: PoC in Go
* Last updated: 2024-04-15
* Discussion at: <https://groups.google.com/g/grpc-io/c/oxNJT1GgjEg>

## Abstract

Add support for the `random_subsetting_experimental` load balancing policy.

## Background

Currently, gRPC is lacking a way to select a subset of endpoints available from the resolver and load-balance requests between them. Out of the box, users have the choice between two extremes: `pick_first` which sends all requests to one random backend, and `round_robin` which sends requests to all available backends. `pick_first` has poor connection balancing when the number of client is not much higher than the number of servers. `round_robin` results in every server having as many connections open as there are clients, which is unnecessarily costly when there are many clients, and makes local load balancing decisions (such as outlier detection) less precise.

### Related Proposals

* [gRFC A52: gRPC xDS Custom Load Balancer Configuration](https://github.com/grpc/proposal/blob/master/A52-xds-custom-lb-policies.md)
* [gRFC A42: xDS Ring Hash LB Policy](https://github.com/grpc/proposal/blob/master/A42-xds-ring-hash-lb-policy.md)

## Proposal

Introduce a new LB policy, `random_subsetting_experimental`. This policy selects a subset of endpoints and passes them to the child LB policy. It maintains 2 important properties:

* The policy tries to distribute connections among servers as equally as possible. The higher `(N_clients*subset_size)/N_servers` ratio is, the closer the resulting server connection distribution is to uniform.
* The policy minimizes the amount of connection churn generated during server scale-ups by using [rendezvous hashing](https://en.wikipedia.org/wiki/Rendezvous_hashing)

### LB Policy Config and Parameters

The `random_subsetting_experimental` LB policy config will be as follows.

```proto
message LoadBalancingConfig {
    oneof policy {
        RandomSubsettingLbConfig random_subsetting_experimental = 21 [json_name = "random_subsetting_experimental"];
    }
}
message RandomSubsettingLbConfig {
    // subset_size indicates how many backends every client will be connected to.
    // The value is required and must be greater than 0.
    uint32 subset_size = 1;

    // The config for the child policy.
    // The value is required.
    repeated LoadBalancingConfig child_policy = 2;
}
```

### Subsetting algorithm

* The policy receives a single configuration parameter: `subset_size`, which must be configured by the user.
* When the lb policy is initialized it also creates a random integer `seed`.
* After every resolver update the policy picks a new subset. It does this by implementing `rendezvous hashing` algorithm:
  * For every endpoint in the list, compute a hash with previously generated seed. Any uniform hash function should work, but we suggest using `XXH64` function as deined in <https://github.com/Cyan4973/xxHash> since it is already used in [gRFC A42](https://github.com/grpc/proposal/blob/master/A42-xds-ring-hash-lb-policy.md).
  * If an endpoint defines multiple addresses - use the first one as input to the hash function.
  * Sort all endpoints by hash.
  * Pick first `subset_size` values from the list.
* Pass the resulting subset to the child LB policy.
* If the number of endpoints is less than `subset_size` always use all available endpoints.

Here is the implementation of this algorithm in pseudocode.

```
func filter_endpoints(endpoints, subset_size, seed)
    if subset_size >= len(endpoints) { 
        return endpoints 
    }
    endpoints_with_hash = []
    foreach endpoint in endpoints {
        // Use XXH64 function to compute hash for every endpoint.
        hash = XXH64(endpoint.addresses[0], seed)
        endpoints_with_hash.append({
            hash: hash, 
            endpoint: endpoint,
        })
    }

    // Sort by hash.
    endpoints_with_hash.sort(func(a, b) { return a.hash < b.hash })

    // Convert endpoints_with_hash list back to the list of endpoints.
    sorted_endpoints = endpoints_with_hash.map(func(a) { return a.endpoint })

    // Select first subset_size elements.
    return sorted_endpoints[:subset_size]
```

### Characteristics of the selected algorithm

#### Uniform connection distribution on high scale

When `(N_clients*subset_size)/N_servers` ratio is high, the resulting connection distribution between servers is close to uniform. This is because the chosen hash function is uniform and every server has equal probability to be chosen by every client.
Though it could be done, we don't provide any mathematical guaranties about the resulting connection distribution. Any such guarantees will be probabilistic and have limited value in practice. Instead, we can give you a few samples:

* N_clients = 100, N_servers = 100, subset_size = 5
![State Diagram](A68_graphics/subsetting100-100-5.png)
* N_clients = 100, N_servers = 100, subset_size = 25
![State Diagram](A68_graphics/subsetting100-100-25.png)
* N_clients = 100, N_servers = 10, subset_size = 5
![State Diagram](A68_graphics/subsetting100-10-5.png)
* N_clients = 500, N_servers = 10, subset_size = 5
![State Diagram](A68_graphics/subsetting500-10-5.png)
* N_clients = 2000, N_servers = 10, subset_size = 5
![State Diagram](A68_graphics/subsetting2000-10-5.png)

#### Low connection churn during server rollouts

The graphs provided in the previous section prove this is the case in practice (we rollout all servers in the middle of every test, and there is no visible increase in the number of connections per server) Low connection churn during server rollouts is the primary motivation why rendezvous hashing was used as the subsetting algorithm: it guaranties that if a single server is either added or removed to the endpoints list, every client will update at most 1 entry in its subset. This is because the hashes for all unaffected servers remain the same, which guarantees that the order of the servers after sorting also remains stable. The same logic applies to the situation when multiple servers got updated.

### Handling Parent/Resolver Updates

When the resolver updates the list of endpoints, or the LB config changes, Random subsetting LB will run the subsetting algorithm, described above, to filter the endpoint list. Then it will create a new resolver state with the filtered list of the endpoints and pass it to the child LB. Attributes and service config from the parent resolver state will be passed down with each resolver update.

### Handling Subchannel Connectivity State Notifications

Random subsetting LB will simply redirect all requests to the child LB without doing any additional processing. This also applies to all other callbacks in the LB interface, besides the one that handles resolver and config updates (which is described in the previous section). This is possible because random subsetting LB doesn't store or manage sub-connections - it acts as a simple filter on the resolver state, and that's why it can redirect all actual work to the child LB.

### xDS Integration

Random subsetting LB won't depend on xDS in any way. People may choose to initialize it by directly providing service config. We will only provide a corresponding xDS policy wrapper to allow configuring this LB via xDS.

#### Changes to xDS API

`random_subsetting` will be added as a new LB policy.

```proto
package envoy.extensions.load_balancing_policies.random_subsetting.v3;
message RandomSubsetting {
    // subset_size indicates how many backends every client will be connected to.
    // The value must be greater than 0.
    uint32 subset_size = 1;

    // The config for the child policy.
    // The value is required.
    config.cluster.v3.LoadBalancingPolicy child_policy = 2;
}
```

As you can see, the fields in this policy match exactly the fields in the random subsetting LB service config.

#### Integration with xDS LB Policy Registry

As described in [gRFC A52](https://github.com/grpc/proposal/blob/master/A52-xds-custom-lb-policies.md), gRPC has an LB policy registry, which maintains a list of converters. Every converter translates xDS LB policy to the corresponding service config. In order to allow using the Random subsetting LB policy via xDS, the only thing that needs to be done is providing a corresponding converter function. The function implementation will be trivial as the fields in the xDS LB policy will match exactly the fields in the service config.

## Rationale

### Alternatives Considered: Deterministic subsetting

We explored the possibility of using deterministic subsetting in <https://github.com/grpc/proposal/pull/383>  and got push-back on this for the reasons explained [here](https://github.com/grpc/proposal/pull/383#discussion_r1334587561)

## Implementation

DataDog will provide Go and Java implementations.
