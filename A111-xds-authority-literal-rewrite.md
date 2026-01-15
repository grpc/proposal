Title
----
* Author(s): Zhiyan Foo, Antoine Tollenaere
* Approver: TBD
* Status: Draft
* Implemented in: Go
* Last updated: 2026-01-15
* Discussion at: TBD (filled after thread exists)

## Abstract

Implement the [`host_rewrite_literal`][envoy-host_rewrite_literal] route_action
feature in gRPC xDS client, enabling explicit authority header rewrites. This
will be conditioned on `trusted_xds_server` as described in [gRFC A81][A81].

## Background

gRPC xDS currently supports `auto_host_rewrite` as per [gRFC A81][A81] but lacks
`host_rewrite_literal` support, useful when a single cluster targets a reverse
proxy routing based on the authority header.

```
            +------------------+
            |  gRPC xDS Client |  <- (RDS configuration sets `host_rewrite_literal` to
            +------------------+      target one of Svc A/B/C)
                     |
                     |    <- (gRPC xDS client reuses the same CDS target for all connections to
                     |        the proxy)
                 +---------+
                 |  Proxy  |
                 +---------+
                      |   <- (Proxy routing logic based on Authority header)
      .---------------+--------------.
     /                |               \
+--------+       +--------+       +--------+
| Svc A  |       | Svc B  |       | Svc C  |
+--------+       +--------+       +--------+
```

### Related Proposals:
* [gRFC A29: xDS-Based Security for gRPC Clients and Servers][A29]
* [gRFC A81: xDS Authority Rewriting][A81]
* [gRFC A86: xDS-Based HTTP CONNECT][A86]

## Proposal

This proposal has the following parts:

- xDS resource unmarshalling: We will retrieve the `host_rewrite_literal` value
  from RDS and return that information to the XdsClient watchers as part of the
  parsed resources.

- xDS ConfigSelector: the xDS ConfigSelector will enable authority rewriting
  based on the chosen route.

- xds_cluster_impl LB policy: The LB policy will perform the authority
  rewriting if it was enabled by the ConfigSelector using the mechanism
  introduced in [A81] for authority rewriting.

### xDS Resource Validation

When validating an RDS resource, if the `trusted_xds_server` server option is
present for this xDS server in the bootstrap config, we will look at the
[envoy-host_rewrite_literal] field. The string value of this field will be
included in the parsed resource struct that is passed to the XdsClient watcher.
If `trusted_xds_server` is unset, the field will be set to the empty string.

### xDS ConfigSelector Changes

When the xDS ConfigSelector performs routing, it will need to pass the values of
the route's `host_rewrite_literal` field to the LB picker. This data will be
passed using the mechanism used to handle `auto_host_rewrite` in [A81].

### xds_cluster_impl LB Policy Changes

In the xds_cluster_impl picker, if the `host_rewrite_literal` field is not empty
in the route, the picker will set the `:authority` header to the value of the
field attribute, as described in [A81].

The `:authority` header is subject to the same secure naming checks defined in
[A81].

### Precedence rules

Because `host_rewrite_literal` and `auto_host_rewrite` are part of the same
`host_rewrite_specifier` oneof, those fields cannot collide and there is no
precedence rule to define. If users set a per-call `:authority` explicitly, it
will take precedence over the value in `host_rewrite_literal`. This matches the
behavior implemented for `auto_host_rewrite`.

## Temporary environment variable protection

Feature guarded by `GRPC_EXPERIMENTAL_XDS_LITERAL_AUTHORITY_REWRITE` env var,
disabled by default. The env var guard will be removed once the feature passes
interop tests.


## Rationale

Alternative approaches considered.
- Exclusively use the existing [`auto_host_rewrite`][route_action] feature. This would require a
  CDS cluster per upstream target, which means that there would be one connection per service
  behind the proxy between each gRPC clients and the proxy.
- Using the HTTP CONNECT protocol has the same drawback in that connections to the proxy that
  target different upstream services won't reuse the same connection.
   - in the case where instead of a single proxy in between the client and target upstream
     service there are two proxies (e.g. client -> egress proxy -> ingress proxy -> svc), this
     would not only prevent connection reuse between the initial client to the proxy, but also
     between the intermediary proxies (e.g. between the egress and ingress proxy).

Even if connection pooling was not a major concern, this addition will bring gRPC xDS client
functionality closer to parity with Envoy's capabilities, simplifying configurations for users
migrating to or using both systems concurrently.

## Implementation

[Go implementation](https://github.com/grpc/grpc-go/pull/8838).

[A29]: A29-xds-tls-security.md
[A81]: A81-xds-authority-rewriting.md
[A86]: https://github.com/grpc/proposal/pull/455
[envoy-host_rewrite_literal]: https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route_components.proto#envoy-v3-api-field-config-route-v3-routeaction-host-rewrite-literal
[route_action]: https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route_components.proto
