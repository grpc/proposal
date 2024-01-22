A76: Improvements to the Ring Hash LB Policy
----
* Author(s): @atollena
* Approver: ?
* Status: Draft
* Implemented in: <language, ...>
* Last updated: 2024-01-15
* Discussion at: TODO

## Abstract

This proposal describes two improvements to the `ring_hash` load balancing policy:

1. The ability to use ring hash without xDS, by extending the policy
   configuration to define the [request metadata][metadata] to use as the
   request hash key.
2. The ability to specify endpoint hash keys explicitly, instead of hashing the
   endpoint IP address.

## Background

### Terminology

* The *request hash key*, after being hashed, defines where a given request is
  to be placed on the ring in order to find the closest endpoints.
* The *endpoint hash key*, after being hashed, determines the locations of an
  endpoint on the ring.

### Using ring hash without xDS by explicitly setting the request hash key

gRPC supports the `ring_hash` load balancing policy for consistent hashing. This
policy currently requires using xDS for configuration because users have no
other way to configure the hash for a request but to use the route configuration
`hash_policy` field in the `RouteAction` route configuration. This makes the
`ring_hash` policy unusable without an xDS infrastructure in place.

This proposal extends the configuration of `ring_hash` policy to specify a
metadata key. The value associated with this metadata key will be used as the
request hash key if present. This will make it possible to use `ring_hash` by
configuring it entirely in the [service config][service-config]. If this
configuration is omitted, we will preserve the current behavior of using the xDS
hash policy.

### Using an explicit endpoint hash key

Another limitation of the current `ring_hash` load balancing policy is that it
always hashes the endpoint IP address to place the endpoints on the ring. In
some scenario, this choice is not ideal: for example, [Kubernetes
Statefulsets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
offer a way to configure workloads with sticky identity such that endpoints keep
storage and network identifier across restarts. However, the IP address may
change across restarts. After a deployment, if all IP addresses have changed,
then a completely new ring has to be constructed, even though it may have been
desirable to keep the ring unchanged based on the Statefulsets identities, so
that each instance stays at the same location on the ring.

Envoy offers a solution to control endpoint hashes independently of IP
addresses. This mechanism uses the `"envoy.lb"`
[LbEndpoint.Metadata][LbEndpoint.Metadata] field `hash_key` value available in
EDS instead of the endpoint IP address, as described in [the Envoy documentation
for ring hash][envoy-ringhash].  This proposal adds support for setting the
endpoint hash key explicitly via EDS by reusing the configuration mechanism
implemented in Envoy. To retain the advantage of being able to use `ring_hash`
without xDS, custom gRPC name resolvers will be able to set this endpoint
attribute through the language specific resolver attribute interface.

### Related Proposals:

This proposal extends the following existing gRFC:

* [gRFC A42: xDS Ring Hash LB Policy][A42]

## Proposal

### Explicitly setting the request hash key

A new string field `request_metadata_key` will be added to the `ring_hash`
policy config. The ring hash policy will be modified as follows:

Upon loading the load balancing config, if the `request_metadata_key` field
contains a value that is not a valid metadata key, then the configuration is
rejected. If the `request_metadata_key` refers to a binary metadata (suffixed
with `-bin`), the configuration is also rejected.

At pick time:
- If `request_metadata_key` is not empty, and the request metadata has a
non-empty value, then the request hash key will be set to this value. If
`request_metadata_key` contains multiple values, then values are joined with a
comma `,` character between each value before hashing.
- If `request_metadata_key` is not empty, and the request has no value or an
empty value associated with the metadata key defined, then the picker will
generate a random hash for it. The use of a random hash key will effectively
sends the request to a random endpoint.
- If `request_metadata_key` is empty, then the request hash key will be based on
the xDS hash policy in RDS, which keeps the existing LB configuration for ring
hash working as before with xDS.
- If `request_metadata_key` is empty but the xDS configuration does not provide
the hash key, then the picker will generate a random hash for it to select a
random endpoint, which matches the current behavior for xDS.


### Explicitly setting the endpoint hash key

The `ring_hash` policy will be changed such that the hash key used for
determining the locations of each endpoint on the ring will be extracted from a
pre-defined endpoint attribute called `hash_key`. If this attribute is set, then
the endpoint is placed on the ring by hashing its value. If this attribute is
not set or empty, then the endpoint IP address is hashed, matching the current
behavior. The locations of an existing endpoint on the ring is updated if its
`hash_key` endpoint attribute changes.

The cluster resolver LB policy responsible for translating EDS responses into
resolver endpoints will be changed to set the `hash_key` endpoint attribute to
the value of [LbEndpoint.Metadata][LbEndpoint.Metadata] `envoy.lb` `hash_key`
field, as described in [Envoy's documentation for the ring hash load
balancer][envoy-ring-hash].

### LB Policy Config Changes

After the addition of this field, the `ring_hash` LB policy config will be:

```proto
    message RingHashLoadBalancingConfig {
      // A client-side cap will limit these values.  If either of these values
      // are greater than the client-side cap, they will be treated as the
      // client-side cap.  The default cap is 4096.
      uint64 min_ring_size = 1;        // Optional, defaults to 1024.
      uint64 max_ring_size = 2;        // Optional, defaults to 4096, max is 8M.

      string request_metadata_key = 3; // Optional, unused if unset.
    }
```

### Temporary environment variable protection

Explicitly setting the request hash key cannot possibly lead to problem with
existing deployment because the new behavior requires setting a load balancing
policy configuration field that did not exist before. Therefore, it is not gated
behind an environment variable.

The second behavior change will be enabled by the
`GRPC_EXPERIMENTAL_XDS_RING_HASH_ENDPOINT_HASH_KEY` environment variable. This
will protect from the case where an xDS control plane is already setting the
`LbEndpoint.Metadata` `envoy.lb` `hash_key` field, in which case deploying this
new behavior would change all endpoint hash keys. This environment variable will
be removed once the feature has proven stable.

## Rationale

We originally proposed using language specific interfaces to set the request
hash key. The advantage would have been that the request hash key would not have
to be exposed through gRPC outgoing metadata. However, this would have required
defining language specific APIs, which would increase the complexity of this
change.
 
We also discussed the option of exposing all `LbEndpoint.metadata` from EDS
through name resolver attributes, instead of only extracting the specific
`hash_key` attribute, so as to make them available to custom LB policies. We
decided to keep only extract `hash_key` to limit the scope of this gRFC.

## Implementation

Will provide an implementation in Go.

[A42]: A42-xds-ring-hash-lb-policy.md
[envoy-ringhash]: https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/load_balancers#ring-hash
[metadata]: https://grpc.io/docs/what-is-grpc/core-concepts/#metadata
[service-config]: https://github.com/grpc/grpc/blob/master/doc/service_config.md
[LbEndpoint.Metadata]: https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/endpoint/v3/endpoint_components.proto#envoy-v3-api-field-config-endpoint-v3-lbendpoint-metadata
