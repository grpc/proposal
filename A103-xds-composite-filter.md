A103: xDS Composite Filter
----
* Author(s): markdroth
* Approver: ejona86, dfawley
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2025-08-28
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

gRPC will support the [xDS Composite filter][composite], which is a
"wrapper" filter that dynamically determines which filter to use based
on request attributes.

[composite]: https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/composite_filter

## Background

xDS support in the gRPC client and server are described in [A27] and
[A36], respectively.  xDS HTTP filter support is described in [A39].

The composite filter will make use of the Unified Matching API and CEL
support described in [A77].

### Related Proposals: 
* [A27: xDS-Based Global Load Balancing][A27]
* [A39: xDS HTTP Filter Support][A39]
* [A36: xDS-Enabled Servers][A36]
* [A77: xDS Server-Side Rate Limiting][A77] (pending)

[A27]: A27-xds-global-load-balancing.md
[A36]: A36-xds-for-servers.md
[A39]: A39-xds-http-filters.md
[A77]: https://github.com/grpc/proposal/pull/414

## Proposal

We will support the composite filter in both the gRPC client and gRPC
server.

### xDS Resource Validation

Today, the composite filter supports configuring only one filter as a
result of the matching tree.  However, we have use-cases where we need
to select a chain of more than one filter based on the matching tree.
As a result, we are proposing a change to the composite filter's config
to allow selecting a chain of filters
(https://github.com/envoyproxy/envoy/pull/40885).

The composite filter is configured via the
[`envoy.extensions.common.matching.v3.ExtensionWithMatcher`
proto](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/extensions/common/matching/v3/extension_matcher.proto#L25).
Within it, gRPC will look at the following fields:
- [extension_config](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/extensions/common/matching/v3/extension_matcher.proto#L34C39-L34C55):
  This must contain a
  [`envoy.extensions.filters.http.composite.v3.Composite`
  proto](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/extensions/filters/http/composite/v3/composite.proto#L33),
  which has no fields.
- [xds_matcher](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/extensions/common/matching/v3/extension_matcher.proto#L31):
  Specifies a unified matcher tree indicating the config of the filter
  to use.  The actions in this tree must be one of two types:
  - [`envoy.extensions.filters.common.matcher.action.v3.SkipFilter`](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/extensions/filters/common/matcher/action/v3/skip_action.proto#L24):
    This indicates that the no filter will be executed.
  - [`envoy.extensions.filters.http.composite.v3.ExecuteFilterAction`](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/extensions/filters/http/composite/v3/composite.proto#L49):
    This indicates which filter should be executed.  Within it:
    - [typed_config](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/extensions/filters/http/composite/v3/composite.proto#L54C39-L54C51):
      The filter to configure.  It will be validated using the xDS HTTP filter
      registry, just as the top-level filters in the HTTP connection manager
      config are.  This field is ignored if `filter_chain` is set.
    - [dynamic_config](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/extensions/filters/http/composite/v3/composite.proto#L59):
      TODO: Do we need to support ECDS here?  Also need to define
      precedence rules.
    - filter_chain (new field added in
      https://github.com/envoyproxy/envoy/pull/40885): If set, the
      `typed_config` field is ignored.  This specifies a chain of
      filters to call, in order.  Each filter in the chain will be
      validated using the xDS HTTP filterregistry, just as the top-level
      filters in the HTTP connection manager config are.
    - [sample_percent](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/extensions/filters/http/composite/v3/composite.proto#L69C43-L69C57):
      - Optional; if unset, the specified filter(s) are always executed.
        If set, for each RPC, a random number will be generated between
        0 and 100, and if that number is less than the specified
        threshold, the specified filter(s) will be executed.
        Within this field:
        - [default_value](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/core/v3/base.proto#L648):
          This field must be present. The configured value will be capped at
          100%.
        - runtime_key: This field will be ignored, since gRPC does not
          have a runtime system.
- matcher: gRPC will not support this deprecated field.

We will also support per-route overrides via the
[`envoy.extensions.common.matching.v3.ExtensionWithMatcherPerRoute`
proto](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/extensions/common/matching/v3/extension_matcher.proto#L39C9-L39C37).
In this proto, the
[xds_matcher](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/extensions/common/matching/v3/extension_matcher.proto#L41)
must be validated the same way as the corresponding field in the
top-level config.  The value of this field will replace the value of the
field in the top-level config.

### CEL Attributes

In addition to the CEL request attributes described in [A77], we will
also add support for the following [configuration
attributes](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/advanced/attributes.html#configuration-attributes):
(TODO: document these fields!)
- `xds.route_metadata`: TODO: tie to metadata registry from A83
- `source.ip`: TODO: behavior on client side?
- `source.port`: TODO: behavior on client side?
- `connection.requested_server_name`: TODO: behavior on client side?  Do
  we have access to this data on server side?
- `connection.tls_version`: TODO: behavior on client side?  Do we have
  access to this data on server side?
- `connection.sha256_peer_certificate_digest`: TODO: behavior on client
  side?  Do we have access to this data on server side?

### Temporary environment variable protection

Support for the composite filter will be guarded by the
`GRPC_EXPERIMENTAL_XDS_COMPOSITE_FILTER` environment variable. This
guard will be removed once the feature passes interop tests.

## Rationale

The composite filter API seems a little unusual.  One would naively
have expected it to be structured by having the matcher tree live
in the config for the composite filter directly, rather than using
`ExtensionWithMatcher`.  Unfortunately, that's not the way the API evolved
in Envoy, so we'll stick with what already exists for compatibility
reasons.

## Implementation

Will be implemented in C-core, Java, Go, and Node.
