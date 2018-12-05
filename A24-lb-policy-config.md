Load Balancing Policy Configuration
----
* Author(s): Mark D. Roth (roth@google.com)
* Approver: a11r
* Status: Draft
* Implemented in: C-core
* Last updated: 2018-12-04
* Discussion at: 

## Abstract

This document describes a new mechanism for specifying [load balancing
policy](https://github.com/grpc/grpc/blob/master/doc/load-balancing.md)
configuration via the [gRPC service
config](https://github.com/grpc/grpc/blob/master/doc/service_config.md).
It also moves away from the special case currently used for selecting
the `grpclb` LB policy and toward a more general-purpose mechanism for
selecting LB policies.

## Background

There are cases where a service owner may need to provide configuration
in the gRPC service config to be used by the load balancing policy on
the client.  Currently, the service config provides a single
`loadBalancingPolicy` field that indicates the name of the LB policy to
use but does not provide a way to encode any configuration for that
policy.  This proposal replaces the existing field with a new field that
specifies both the LB policy name and some associated configuration.

In addition, we are currently in the process of moving away from our
custom grpclb load balancing protocol and to a new design using the Envoy
xDS API (a separate gRFC for that will be published in the future).
As part of this, we want to move away from the special case that we
currently use to select the `grpclb` LB policy, which is that it gets
triggered whenever the resolver returns at least one balancer address.
Instead, we would like to use data in the service config to tell the
new `xds` policy which balancers to contact.

### Related Proposals: 

* [A5: gRPC-LB in DNS](A5-grpclb-in-dns.md)
* [A21: Service Config Error Handling](https://github.com/grpc/proposal/pull/100) (in progress)

There will be a subsequent gRFC describing the new `xds` LB policy.

## Proposal

We propose to deprecate the existing `loadBalancingPolicy` service
config field.  In its place, we will add a new field called
`loadBalancingConfig` that will encode both the policy name and a
corresponding configuration for the policy.

Here is how the service config definition will now look.  First, in
proto form:

```proto
// Configuration for round_robin LB policy.
message RoundRobinConfig {
}

// Configuration for xds LB policy.
message XdsConfig {
  // Required.  Name of balancer to connect to.
  string balancer_name = 1;
  // Optional.  What LB policy to use for intra-locality routing.
  // If unset, will use whatever algorithm is specified by the balancer.
  LoadBalancingConfig child_policy = 2;
  // Optional.  What LB policy to use in fallback mode.  If not
  // specified, defaults to round_robin.
  LoadBalancingConfig fallback_policy = 3;
}

// Configuration for grpclb LB policy.
message GrpcLbConfig {
}

// Selects LB policy and provides corresponding configuration.
message LoadBalancingConfig {
  // Exactly one LB policy may be configured.
  // If no LB policy is configured, then the default behavior is to pick
  // the first available backend.
  oneof policy {
    // For each new LB policy supported by gRPC, a new field must be added
    // here.  The field's name must be the LB policy name and its type is a
    // message that provides whatever configuration parameters are needed
    // by the LB policy.  The configuration message will be passed to the
    // LB policy when it is instantiated on the client.
    //
    // If the LB policy does not require any configuration parameters, the
    // message for that LB policy may be empty.
    //
    // Note that if an LB policy contains another nested LB policy
    // (e.g., a top-level policy picks the locality and then delegates to
    // another policy to pick the endpoint within that locality), its
    // configuration message may include a nested instance of the
    // LoadBalancingConfig message to configure the nested LB policy.
    RoundRobinConfig round_robin = 1 [json_name = "round_robin"];
    XdsConfig xds = 2;
    GrpcLbConfig grpclb = 3;
  }
}

message ServiceConfig {
  // Load balancing policy.
  //
  // Note that load_balancing_policy is deprecated in favor of
  // load_balancing_config; the former will be used only if the latter
  // is unset.
  //
  // If no LB policy is configured here, then the default behavior is to
  // pick the first available backend.
  //
  // If the deprecated load_balancing_policy field is used, note that if the
  // resolver returns at least one balancer address (as opposed to backend
  // addresses), gRPC will use grpclb (see
  // https://github.com/grpc/grpc/blob/master/doc/load-balancing.md),
  // regardless of what policy is configured here.  However, if the resolver
  // returns at least one backend address in addition to the balancer
  // address(es), the client may fall back to the requested policy if it
  // is unable to reach any of the grpclb load balancers.
  enum LoadBalancingPolicy {
    UNSPECIFIED = 0;
    ROUND_ROBIN = 1;
  }
  LoadBalancingPolicy load_balancing_policy = 1 [deprecated=true];
  // Multiple LB policies can be specified; clients will iterate through
  // the list in order and stop at the first policy that they support.
  // This allows the service config to specify custom policies that may not
  // be known to all clients.
  repeated LoadBalancingConfig load_balancing_config = 4;

  // ...other existing fields...
}
```

For example, to use the `xds` policy with a child policy of `round_robin`,
something like this could be used:

```proto
load_balacing_config: {
  xds: {
    balancer_name: "balancer.example.com:8080"
    child_policy: {
      round_robin: {}
    }
  }
}
```

In JSON form (which is what is actually used in gRPC core), it would
look like this:

```json
{"loadBalancingConfig":
  [
    {"xds": {
      "balancerName": "balancer.example.com:8080",
      "childPolicy": {
        "round_robin": {}
      }
    }}
  ]
}
```

## Rationale

When the new `xds` LB policy is ready, we will need a migration path to
migrate clients from `grpclb` to `xds`.  The new service config field
described in this document has been explicitly designed to aid that
migration.

There are several types of clients that we'll need to support for this
migration:
  * Old clients that do not understand the new LoadBalancingConfig service
    config field.
  * New clients that understand the new LoadBalancingConfig service config
    field and support the xds LB policy.
  * In-between clients that understand the new LoadBalancingConfig service
    config field but do not yet support the xds LB policy.

To ease migration from `grpclb` to `xds`, a resolver can return both
the balancer addresses used by `grpclb` and a service config with the
new `loadBalancingConfig` field (a repeated field with semantics of
choosing the first LB policy supported by the client) set indicating to
use `xds` if available and if not to fall back to `grpclb`.  The new
`loadBalancingConfig` service config field will override the special
case that selects the `grpclb` policy whenever the resolver returns at
least one balancer address.  (The old `loadBalancingPolicy` field will
continue to be overridden by the presence of balancer addresses.)

This gives us what we want in all cases:
  * Old clients will ignore the `loadBalancingConfig field and will use
    `grpclb` based on the presence of the balancer addresses.
  * New clients will use `xds` based on the `loadBalancingConfig` field.
  * In-between clients will see the `loadBalancingConfig` field, but they
    will not support the `xds` policy, so they will use the second entry
    (for `grpclb`).

Note: This assumes that the old clients do not yet implement gRPC Service
Config Error Handling.  This means that we need to add support for the
new `loadBalancingConfig` service config field (already done in C-core)
before we implement the new error handling.

Note: This also assumes that there are no clients that understand
the `loadBalancingConfig` field and have an incomplete implementation
of the `xds` policy.  To avoid this, we will temporarily use the name
`xds_experimental` while the policy is in development.

## Implementation

Already implemneted in C-core.

Java and Go TBD.

## Open issues (if applicable)

N/A
