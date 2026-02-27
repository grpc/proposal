A103: xDS Composite Filter
----
* Author(s): markdroth
* Approver: ejona86, dfawley
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2026-02-27
* Discussion at: https://groups.google.com/g/grpc-io/c/es5taH0OZS8

## Abstract

gRPC will support the [xDS Composite filter][composite], which is a
"wrapper" filter that dynamically determines which filter to use based
on request attributes.

[composite]: https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/composite_filter

## Background

xDS support in the gRPC client and server are described in [A27] and
[A36], respectively.  xDS HTTP filter support is described in [A39].

The composite filter will make use of the Unified Matching API and CEL
support described in [A106].

### Related Proposals: 
* [A27: xDS-Based Global Load Balancing][A27]
* [A39: xDS HTTP Filter Support][A39]
* [A36: xDS-Enabled Servers][A36]
* [A83: xDS GCP Authentication Filter][A83]
* [A106: xDS Unified Matcher and CEL Integration][A106] (pending)

[A27]: A27-xds-global-load-balancing.md
[A36]: A36-xds-for-servers.md
[A39]: A39-xds-http-filters.md
[A106]: https://github.com/grpc/proposal/pull/520
[A83]: A83-xds-gcp-authn-filter.md

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
proto](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/extensions/common/matching/v3/extension_matcher.proto#L25)
message.  Within it, gRPC will look at the following fields:
- [extension_config](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/extensions/common/matching/v3/extension_matcher.proto#L34C39-L34C55):
  This must contain a
  [`envoy.extensions.filters.http.composite.v3.Composite`
  proto](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/extensions/filters/http/composite/v3/composite.proto#L33)
  message, which has no fields.
- [xds_matcher](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/extensions/common/matching/v3/extension_matcher.proto#L31):
  Specifies a unified matcher tree indicating the config of the filter
  to use, as described in [A106].  Validation will fail if `keep_matching`
  is enabled anywhere in the matcher tree.  The actions in this tree must
  be one of two types:
  - [`envoy.extensions.filters.common.matcher.action.v3.SkipFilter`](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/extensions/filters/common/matcher/action/v3/skip_action.proto#L24):
    This indicates that no filter will be executed.
  - [`envoy.extensions.filters.http.composite.v3.ExecuteFilterAction`](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/extensions/filters/http/composite/v3/composite.proto#L49):
    This indicates which filter(s) should be executed.  Within it:
    - [typed_config](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/extensions/filters/http/composite/v3/composite.proto#L54C39-L54C51):
      The filter to configure.  See [Nested Filter
      Validation](#nested-filter-validation) below for validation rules.
      This field is ignored if `filter_chain` is set.  It
      is an error if neither `typed_config` nor `filter_chain` are set.
    - [dynamic_config](https://github.com/envoyproxy/envoy/blob/0685d7bf568485eb112df2a9c73248cb8bfc1c37/api/envoy/extensions/filters/http/composite/v3/composite.proto#L59):
      This field will be ignored for now, since gRPC does not currently
      support ECDS.  Support for ECDS will be added in a subsequent gRFC.
    - filter_chain (new field added in
      https://github.com/envoyproxy/envoy/pull/40885): This specifies a
      chain of filters to call, in order.  See [Nested Filter
      Validation](#nested-filter-validation) below for validation rules.
      If set, the `typed_config` field is ignored.  It is an error if
      neither `typed_config` nor `filter_chain` are set.
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
field must be validated the same way as the corresponding field in the
top-level config.  The value of this field will replace the value of the
field in the top-level config.

The parsed representation of the composite filter's config will be
a matcher tree, using the unified matcher API described in [A106].
The actions in the matcher tree will be one of two possible values: a
list of parsed filter configs, or an indication that the filter should
be skipped (in the case of a `SkipFilter` proto).

Note that in order to avoid potential stack overflows, we will impose
a maximum recursion depth of 8 when parsing HTTP filter configs.

#### Nested Filter Validation

Filter configs within the matcher tree will be validated using the xDS
HTTP filter registry, just as the top-level filters in the HTTP connection
manager config are.

It will be considered a configuration error if a nested filter is a
terminal filter (as described in [A39]).  While it is conceivably
possible that there could be a use-case where the composite filter is
*intended* to provide a terminal filter, it will be difficult to
determine at config validation time whether this will actually happen
in all cases, so for now we will simply disallow this.

Note that, as per [A39], any given xDS HTTP filter may be supported on
only the gRPC client or server side.  When processing the filter list
in LDS, we know whether the resource is an API listener (client side)
or socket listener (server side), so we reject filters that are not
supported on the side they are being configured on.  However, because
the composite filter's per-route override config may be delivered via
RDS instead of LDS, it will in this case not be possible for gRPC to
detect that a filter is being configured on an unsupported side when
validating the RDS resource.  Instead, the composite filter will need
to handle this on a per-RPC basis: if a nested filter chain includes a
filter that is not supported on the side that it is running on, we will
fail the RPC with status UNAVAILABLE.  Note that if the problematic
filter is not the first filter in the nested filter chain, implementations
may fail the RPC without ever starting to process the RPC on that filter
chain, or they may wait to fail the RPC until it gets to the problematic
filter.

### CEL Attributes

In addition to the CEL request attributes described in [A106], we will
also add support for some additional [CEL
attributes](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/advanced/attributes.html)
that we expect to be useful for the composite filter.

On both the gRPC client and server sides, we will add support for the
`xds.cluster_metadata.filter_metadata` attribute.  This will be backed
by the same cluster metadata whose parsing was added in [A83].  Note that
we will support only `xds.cluster_metadata.filter_metadata`, not
`xds.cluster_metadata.typed_filter_metadata`, so that we do not have to
handle protobuf descriptor functionality; to that end, we will use only
those entries in the parsed metadata map that correspond to
`google.protobuf.Struct` type.  The actual CEL expression we see will be
something like `xds.cluster_metadata.filter_metadata['%s']['%s']`, where
the first `%s` indicates the key of the entry in the parsed metadata map
and the second `%s` indicates the key in the JSON object.  If the JSON
value is not an object, this will not resolve.  The type of the CEL
value will be based on the type of the underlying JSON value, either
string or integer.

We will also add support for the following attributes on the gRPC server
side only (these attributes are not relevant on the client side):
- `source.address`
- `source.port`
- `connection.requested_server_name`
- `connection.tls_version`
- `connection.sha256_peer_certificate_digest`

### Filter Behavior

When the filter sees the client's initial metadata, it will evaluate the
matcher tree for the RPC.  The result of that evaluation will be one of
the following:

- If the matcher tree does not find a match, the RPC will be failed with
  UNAVAILABLE status.
- If the matcher tree finds a `SkipFilter` match, the filter will simply
  pass the RPC through to the next filter (the one after the composite
  filter), without delegating to any nested filters.
- If the matcher tree finds an `ExecuteFilterAction` match, then the
  filter will generate a random number and check the sample_percent
  field to determine if the RPC should be sampled.  If the RPC is not
  sampled, then the filter will pass the RPC through to the next filter
  (the one after the composite filter), without delegating to any nested
  filters.  Otherwise (if the RPC *is* sampled), the RPC will be passed
  to the nested filter chain before being sent to the next filter (the
  one after the composite filter).

Note: Because all of the CEL attributes that we are currently supporting
are available when we see the client's initial metadata, that is the
point at which the filter will evaluate the matcher tree and decide which
filter chain to use.  After that point, all other filter hooks will be
delegated to the chosen filter chain.  (If we ever in the future need
to support other attributes that are not yet available at this point,
such as response attributes, we might need a more complex structure here.)

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
reasons.  (Envoy is currently attempting to add a more sane API in
https://github.com/envoyproxy/envoy/pull/43227.  In the future,
we can support that new API as well.)

## Implementation

Will be implemented in C-core, Java, Go, and Node.
