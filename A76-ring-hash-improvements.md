A76: Improvements to the Ring Hash LB Policy
----
* Author(s): atollena
* Approver: markdroth
* Status: Draft
* Implemented in: Go
* Last updated: 2024-04-26
* Discussion at: https://groups.google.com/g/grpc-io/c/ZKI1RIF0e_s/m/oBXqOFb0AQAJ

## Abstract

This proposal describes two improvements to the `ring_hash` load balancing
policy:

1. The ability to use ring hash without xDS, by extending the policy
   configuration to define the [request header][header] to use as the request
   hash key.
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
header to hash. This will make it possible to use `ring_hash` by configuring it
entirely in the [service config][service-config]. If this configuration is
omitted, we will preserve the current behavior of using the xDS hash policy.

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

This proposal extends the following existing gRFCs:

* [gRFC A42: xDS Ring Hash LB Policy][A42]
* [gRFC A61: IPv4 and IPv6 Dualstack Backend Support][A61]
* [gRFC A74: xDS Config Tears][A74]

## Proposal

### Explicitly setting the request hash key

A new field `request_hash_header` will be added to the `ring_hash` policy
config:

```proto
    message RingHashLoadBalancingConfig {
      // (existing fields omitted)
      string request_hash_header = 3;
    }
```

Upon loading the load balancing config, if the `request_hash_header` field
contains a value that is not a valid header name, then the configuration is
rejected. If the `request_hash_header` refers to a binary header (suffixed with
`-bin`), the configuration is also rejected.

At pick time:
- If `request_hash_header` is empty, then the request hash key will be based on
the xDS hash policy in RDS, which keeps the existing LB configuration for ring
hash working as before with xDS.
- If `request_hash_header` is not empty, and the header has a non-empty value,
then the request hash key will be set to this value. If the header contains
multiple values, then values are joined with a comma `,` character before
hashing.
- If `request_hash_header` is not empty, and the request has no value associated
with the header or its value is empty, then the picker will generate a random
hash for the request. If the use of this random hash triggers a connection
attempt (according to the rules described in [A42: Picker
Behavior][A42-picker-behavior] and updated in [A61: Ring Hash][A61-ring-hash]),
then before queuing the pick, the picker will scan forward searching for a
subchannel in `READY` state. If it finds a subchannel in `READY` state, the
picker returns it.

The following pseudo code describes the updated picker logic:

```
// Determine request hash.
using_random_hash = false;
if (config.request_hash_header.empty()) {
  request_hash = call_attributes.hash;
} else {
  header = headers.find(config.request_hash_header);
  if (header == null) {
    using_random_hash = true;
    request_hash = ComputeRandomHash();
  } else {
    request_hash = ComputeHash(header);
  }
}

// Do pick based on hash.
first_index = ring.FindIndexForHash(request_hash);
requested_connection = false;
for (i = 0; i < ring.size(); ++i) {
  index = (first_index + i) % ring.size();
  if (ring[index].state == READY) {
    return ring[index].picker->Pick(...);
  }
  if (requested_connection) continue;
  if (ring[index].state == IDLE) {
    ring[index].endpoint.TriggerConnectionAttemptInControlPlane();
    if (using_random_hash) {
      requested_connection = true;
      continue;
    }
    return PICK_QUEUE;
  }
  if (ring[index].state == CONNECTING) {
    return PICK_QUEUE;
  }
}
if (requested_connection) return PICK_QUEUE;
return PICK_FAIL;
```

This behavior ensures that a single request creates at most one new connection,
and that a request missing the header does not add extra latency in the common
case where there is already at least one subchannel in `READY` state. It
converges to picking a random host, since each request may create a new
connection to a random endpoint.

### Explicitly setting the endpoint hash key

The `ring_hash` policy will be changed such that the hash key used for
determining the locations of each endpoint on the ring will be extracted from a
pre-defined endpoint attribute called `hash_key`. If this attribute is set, then
the endpoint is placed on the ring by hashing its value. If this attribute is
not set or empty, then the endpoint's first address is hashed, matching the
current behavior. The locations of an existing endpoint on the ring is updated
if its `hash_key` endpoint attribute changes.

The xDS resolver, described in [A74][A74], will be changed to set the `hash_key`
endpoint attribute to the value of [LbEndpoint.Metadata][LbEndpoint.Metadata]
`envoy.lb` `hash_key` field, as described in [Envoy's documentation for the ring
hash load balancer][envoy-ring-hash].

### Temporary environment variable protection

Explicitly setting the request hash key cannot possibly lead to problem with
existing deployment because the new behavior requires setting a load balancing
policy configuration field that did not exist before. Therefore, it is not gated
behind an environment variable.

Adding support for the hash_key in xDS endpoint metadata could potentially break
existing clients whose control plane is setting this key, because upgrading the
client to a new version of gRPC would automatically cause the key to start being
used. We expect that this change will not cause problems for most users, but
just in case there is a problem, we will provide a migration story by supporting
a temporary mechanism to tell gRPC to ignore the `hash_key` endpoint
metadata. This will be enabled by setting the
`GRPC_XDS_ENDPOINT_HASH_KEY_BACKWARD_COMPAT` environment variable to true. This
mechanism will be supported for a couple of gRPC releases but will be removed in
the long run.

## Rationale

We originally proposed using language specific interfaces to set the request
hash key. The advantage would have been that the request hash key would not have
to be exposed through gRPC outgoing headers. However, this would have required
defining language specific APIs, which would increase the complexity of this
change.

We also discussed the option of exposing all `LbEndpoint.metadata` from EDS
through name resolver attributes, instead of only extracting the specific
`hash_key` attribute, so as to make them available to custom LB policies. We
decided to keep only extract `hash_key` to limit the scope of this gRFC.

We discussed various option to handle requests that are missing a hash key in
the non-xDS case. When using ring hash with xDS, the hash is assigned a random
value in the xDS config selector, which ensure all picks for this request can
trigger a connection to at most one endpoint. However, without xDS, there is no
place in the code to assign the hash such that it retains this property. We
considered the following alternative solutions:
1. Add a config selector or filter to pick the hash. There currently is no
   public API to do so from the service config, so we would have had to define
   one.
2. Using an internal request attribute to set the hash. Again, there is no
   cross-language public API for this.
3. Failing the pick. We generally prefer the lack of a header to affect load
   balancing but not correctness, so this option was not ideal.
4. Treating a missing header as being present but having the empty string for
   value. All instances of the channel would end up picking the same endpoint to
   send requests with a missing header, which could overload this endpoint if a
   lot of requests do not have a request hash key.

## Implementation

Implemented in Go:
- Allow setting the request hash key: https://github.com/grpc/grpc-go/pull/7170
- Make endpoint hash key configurable, and set it from EDS: https://github.com/grpc/grpc-go/pull/7161

[A42]: A42-xds-ring-hash-lb-policy.md
[A61]: A61-IPv4-IPv6-dualstack-backends.md
[A74]: A74-xds-config-tears.md
[envoy-ringhash]: https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/load_balancers#ring-hash
[header]: https://grpc.io/docs/guides/metadata/#headers
[service-config]: https://github.com/grpc/grpc/blob/master/doc/service_config.md
[LbEndpoint.Metadata]: https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/endpoint/v3/endpoint_components.proto#envoy-v3-api-field-config-endpoint-v3-lbendpoint-metadata
[A42-picker-behavior]: A42-xds-ring-hash-lb-policy.md#picker-behavior
[A61-ring-hash]: A61-IPv4-IPv6-dualstack-backends.md#ring-hash
