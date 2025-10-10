A92: xDS ExtAuthz Support
----
* Author(s): @markdroth
* Approver: @ejona86, @dfawley
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2025-10-10
* Discussion at: https://groups.google.com/g/grpc-io/c/sPfb9NoB474

## Abstract

We will add support for the xDS ext_authz filter in both the gRPC client
and server.

## Background

The ext_authz filter provides support for making side-channel call-outs
to perform authorization decisions.

The ext_authz filter will use the existing infrastructure for xDS HTTP
filters, described in [A39]. We will support this filter on both
the gRPC client and server side.

Note that this filter will make use of the `allowed_grpc_services` map in
the bootstrap config, described in [A102].  It will also make use of the
`trusted_xds_server` server feature introduced in [A81].

### Related Proposals: 
* [A39: xDS HTTP Filter Support][A39]
* [A36: xDS-Enabled Servers][A36]
* [A81: xDS Authority Rewriting][A81]
* [A83: xDS GCP Authentication Filter][A83]
* [A33: xDS Fault Injection][A33]
* [A102: xDS GrpcService Support][A102] (pending)
* [A60: xDS-Based Stateful Session Affinity for Weighted Clusters][A60]
* [A79: Non-Per-Call Metrics Architecture][A79]
* [A66: OpenTelemetry Metrics][A66]
* [A89: Backend Service Metric Label][A89]

[A36]: A36-xds-for-servers.md
[A39]: A39-xds-http-filters.md 
[A81]: A81-xds-authority-rewriting.md
[A83]: A83-xds-gcp-authn-filter.md
[A33]: A33-Fault-Injection.md
[A102]: https://github.com/grpc/proposal/pull/510
[A60]: A60-xds-stateful-session-affinity-weighted-clusters.md
[A79]: A79-non-per-call-metrics-architecture.md
[A89]: A89-backend-service-metric-label.md
[A66]: A66-otel-stats.md

## Proposal

We will support the ext_authz filter on both the gRPC client and server
side.

### Filter Behavior

On both the gRPC client and server side, the ext_authz filter will perform
the per-RPC authorization check when it sees the client's initial metadata
(i.e., at the start of the stream).

If the `filter_enabled` config field is set to a value less than
100%, the filter will generate a random number in the range [0,
100] for the RPC, and if that random number is greater than or
equal to the configured `filter_enabled` value, then the ext_authz
filter is not considered enabled for that RPC.  In that case, if
the `deny_at_disable` config field is set to true, then the RPC
will be failed with the status derived from the `status_on_error`
config field, using the normal [HTTP-to-gRPC status conversion
rules](https://github.com/grpc/grpc/blob/master/doc/http-grpc-status-mapping.md).
Otherwise, the RPC will be passed to the next filter without modification.

If the filter is enabled for the RPC, it will then send an RPC to the
ext_authz service to determine if the RPC should be allowed.  The filter
will pass the trace context from the data plane RPC to the ext_authz
RPC, so that the ext_authz RPC appears as a child span on the data plane
RPC's trace.

If the RPC to the ext_authz service fails, then if the
`failure_mode_allow` config field is set to false, the data plane RPC
will be failed with the status derived from the `status_on_error`
config field, using the normal [HTTP-to-gRPC status conversion
rules](https://github.com/grpc/grpc/blob/master/doc/http-grpc-status-mapping.md).
Otherwise, the data plane RPC will be allowed.  If the
`failure_mode_allow_header_add` config field is true, then the filter
will add a `x-envoy-auth-failure-mode-allowed: true` header to the data
plane RPC.

#### ExtAuthz Side Channel

The ext_authz filter will create a gRPC channel to the ext_authz server.
It will use the mechanism described in [A102] to determine the server to
talk to and the channel credentials to use.

We do not want to recreate this channel on LDS updates, unless the
target URI or channel credentials changes.  The ext_authz filter will
use the filter state retention mechanism described in [A83] to retain
the channel across updates.

TODO: stats plugin propagation?

### Filter Configuration

The filter supports both a top-level configuration and an override config.

#### Top-Level Configuration

We will support the following fields in the [`ExtAuthz`
proto](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_authz/v3/ext_authz.proto#L34):
- [grpc_service](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_authz/v3/ext_authz.proto#L45):
  This field must be present.  It will be handled as described in [A102].
- [filter_enabled](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_authz/v3/ext_authz.proto#L158):
  Optional; if unset, the filter is enabled.  This field will be validated
  the same way as in the fault injection filter (see [A33]).  Within it:
  - [default_value](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/core/v3/base.proto#L648):
    This field must be present.  The configured value will be capped at 100%.
  - The `runtime_key` field will be ignored, since gRPC does not have a
    runtime system.
- [deny_at_disable](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_authz/v3/ext_authz.proto#L175C18-L175C36):
  Optional; if unset, requests are allowed when the filter is disabled.
  If set, then within this field:
  - [default_value](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/core/v3/base.proto#L298):
    Must be present.  If true, then when the filter is disabled, the
    request will be failed with a status based on the `status_on_error`
    field (see below).
  - The `runtime_key` field will be ignored, since gRPC does not have a
    runtime system.
- [failure_mode_allow](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_authz/v3/ext_authz.proto#L68)
- [failure_mode_allow_header_add](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_authz/v3/ext_authz.proto#L74)
- [status_on_error](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_authz/v3/ext_authz.proto#L95): Note that this field specifies an HTTP status code,
  not a gRPC status code.  The gRPC status code will be determined using
  the normal [HTTP-to-gRPC status conversion
  rules](https://github.com/grpc/grpc/blob/master/doc/http-grpc-status-mapping.md).
- [allowed_headers](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_authz/v3/ext_authz.proto#L229)
- [disallowed_headers](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_authz/v3/ext_authz.proto#L233)
- [decoder_header_mutation_rules](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_authz/v3/ext_authz.proto#L282):
  Optional.  Inside of it:
  - [disallow_all](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/common/mutation_rules/v3/mutation_rules.proto#L70)
  - [allow_expression](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/common/mutation_rules/v3/mutation_rules.proto#L75)
  - [disallow_expression](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/common/mutation_rules/v3/mutation_rules.proto#L79)
  - [disallow_is_error](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/common/mutation_rules/v3/mutation_rules.proto#L87)
  - allow_all_routing, disallow_system, allow_envoy: These fields will
    be ignored.
- [include_peer_certificate](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_authz/v3/ext_authz.proto#L181)

The following fields will be ignored by gRPC:
- http_service: It doesn't make sense for gRPC to support non-gRPC
  mechanisms for contacting the ext_authz server.
- stat_prefix, charge_cluster_response_stats, emit_filter_state_stats:
  These do not apply to gRPC.
- transport_api_version: Not relevant, since gRPC supports only xDS v3.
- with_request_body: This feature is structured around an HTTP request
  and doesn't really make sense for gRPC, both because it's unclear how
  it would work for streaming RPCs and because it would include only the
  first message on the stream (after gRPC deframing), not the raw HTTP
  DATA frame content.
- validate_mutations: We will unconditionally reject invalid mutations.
- clear_route_cache: We don't currently support recomputing the route.
  We could consider adding this in the future if we have a use-case for
  it.
- encode_raw_headers: We will unconditionally encode headers in raw form.
- metadata_context_namespaces, typed_metadata_context_namespaces,
  route_metadata_context_namespaces, route_typed_metadata_context_namespaces,
  enable_dynamic_metadata_ingestion, filter_metadata, filter_enabled_metadata:
  gRPC does not currently support dynamic metadata.
- bootstrap_metadata_labels_key: We have no current use-case for this.  We
  could consider adding it in the future if there is a need.
- include_tls_session: We do not currently have a use-case for this, but
  we could consider adding it in the future if we do encounter such a
  use-case.

#### Override Configuration

We will support `typed_per_filter_config` config overrides for this
filter, as described in [A39].

The override config for this filter is encoded as an [`ExtAuthzPerRoute`
proto](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_authz/v3/ext_authz.proto#L460).
However, we will ignore all of the fields in the proto; the only reason
for supporting it is so that we can disable the filter for individual
virtual hosts, routes, or cluster weights.

Note that we will not use the [`disabled`
field](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_authz/v3/ext_authz.proto#L469)
in the `ExtAuthzPerRoute` proto itself; instead, we will support disabling
via a more generic mechanism that can apply to any filter.  Specifically,
we will honor the `disabled` field in both the [`HttpFilter`
message](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#L1229)
in the HCM config and in the [`FilterConfig` wrapper
message](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/route/v3/route_components.proto#L2562)
that can be used in `typed_per_filter_config` fields.

Note that the most specific `typed_per_filter_config` will be used.  For
example, if there is an override config at the virtual host level that
disables the filter but then another config at the route level that does
not disable the filter, the filter will be enabled.

### The ext_authz Protocol

This section describes the ext_authz protocol.

#### Constructing the ext_authz Request

The [`AttributeContext`
message](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L43)
sent to the server will be populated as follows:
- [source](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L192):
  Will be set only on gRPC server side.  Inside it:
  - [address](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L58)
    will be set to the peer address of the connection that the request
    came in on.
  - [principal](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L82):
    If TLS is used and the client provided a valid certificate, this will be
    set to the cert's first URI SAN if set, otherwise the cert's first DNS
    SAN if set, otherwise the subject field of the certificate in RFC
    2253 format.  If TLS is not used or the client did not provide a
    cert, this field will be unset.
  - [certificate](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L86):
    This will be populated if the `include_peer_certificate` config
    field is set to true.
  - service, labels: Will *not* be set.
- [destination](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L197):
  Will be set only on gRPC server side.  Inside it:
  - [address](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L58)
    will be set to the local address of the connection that the request
    came in on.
  - [principal](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L82):
    If TLS is used, this will be set to the server's cert's first URI SAN
    if set, otherwise the cert's first DNS SAN if set, otherwise the
    subject field of the certificate in RFC 2253 format.  If TLS is not
    used, this field will be unset.
  - certificate, service, labels: Will *not* be set.
- [request](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L200):
  Will always be set.  Inside it:
  - [time](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L95):
    Will be set to the RPC's start time.
  - [http](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L98):
    Will always be set.  Inside of it:
    - [method](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L115):
      Will always be "POST".
    - [header_map](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L143):
      The filter will iterate through each header on the data plane RPC.
      For each header, if the header is matched by the `disallowed_headers`
      config field, it will not be added to this map.  Otherwise,
      if the `allowed_headers` config field is unset or matches the header,
      the header will be added to this map.  Otherwise, the header will
      be excluded from this map.
    - [path](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L148):
      Will always be set.
    - [size](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L165):
      Will always be set to -1.
    - [protocol](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L171):
      Always set to "HTTP/2".
    - id, scheme, headers, query, fragment, body, raw_body: Will *not* be set.
- context_extensions, metadata_context, route_metadata_context,
  tls_session: Will *not* be set.

#### Handling the ext_authz Response

We will handle the [`CheckResponse`](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/external_auth.proto#L117)
as follows:
- [status](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/external_auth.proto#L124):
  Supported.
- [denied_response](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/external_auth.proto#L131):
  Supported.  Inside of it:
  - [status](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/external_auth.proto#L50):
    The HTTP status to fail the RPC with.  We apply the normal
    [HTTP-to-gRPC status conversion
    rules](https://github.com/grpc/grpc/blob/master/doc/http-grpc-status-mapping.md).
  - [headers](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/external_auth.proto#L55):
    In gRPC, failing an RPC involves sending a Trailers-Only response,
    so this field will be used to modify trailers on the data plane RPC
    rather than response headers.  See [Header Rewriting](#header-rewriting)
    below.
  - body: Ignored; does not apply to gRPC.
- [ok_response](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/external_auth.proto#L134):
  - [headers](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/external_auth.proto#L75)
    and
    [headers_to_remove](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/external_auth.proto#L92):
    Modifications to the data plane RPC's request headers.  See
    [Header Rewriting](#header-rewriting) below.
  - [response_headers_to_add](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/external_auth.proto#L104):
    Modifications to the data plane RPC's response headers.  See
    [Header Rewriting](#header-rewriting) below.
  - query_parameters_to_set, query_parameters_to_remove, dynamic_metadata:
    Ignored; these do not apply to gRPC.
- dynamic_metadata: Ignored.

##### Header Rewriting

The response from the ext_authz server may indicate header modifications
to make on the data plane RPC.  When the data plane RPC is allowed,
modifications may be made to the data plane RPC's request headers or
response headers.  And when the data plane RPC is denied, modifications
may be made to the response trailers as part of a Trailers-Only response.

gRPC will not support modifying or removing any header name starting with
`:` or the `host` header, regardless of what settings are present in
the filter's config.  If the server specifies a rewrite for one of these
headers, that rewrite will be ignored.  Otherwise, header rewriting will
be allowed based on the `decoder_header_mutation_rules` config field.

Header additions and modifications are expressed via an
[`envoy.config.core.v3.HeaderValueOption`](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/core/v3/base.proto#L429)
message, which is not specific to the ext_authz filter and will be used
in other places in the future.  We will validate this message as follows:
- [header](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/core/v3/base.proto#L458):
  Required.  Within it:
  - [key](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/core/v3/base.proto#L404):
    The header name.  Must be non-empty and all lower-case.  Length
    must not exceed 16384.  The entry will be ignored if the key is
    `host` or starts with a `:`.
  - [value](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/core/v3/base.proto#L415):
    The header value, used when the header name does not end with `-bin`.
    Length must not exceed 16384.
  - [raw_value](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/core/v3/base.proto#L422):
    The header value, used when the header name ends with `-bin`.  Length
    must not exceed 16384.
- [append_action](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/core/v3/base.proto#L476):
  We honor the 4 enum values as described in the proto file.
- [keep_empty_value](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/core/v3/base.proto#L480):
  By default, any header mutation that results in a header with an
  empty value will cause the header key to be removed.  If this field
  is set to true, then such empty headers will be kept.
- We do not support the deprecated append field.

### Metrics

The ext_authz filter will export metrics using the non-per-call metrics
architecture defined in [A79].  There will be a separate set of metrics
on client side and server side, because (a) there are additional labels
that are relevant on the client but not on the server, and (b) it may be
useful to differentiate between authorization behavior on the client vs.
the server.

#### Client-Side Metrics

The client-side metrics will have the following labels:

| Name        | Disposition | Description |
| ----------- | ----------- | ----------- |
| grpc.target | required | The target of the gRPC channel in which ext_authz is used, as the defined in [A66]. |
| grpc.lb.backend_service | optional | The backend service to which the traffic is being sent, as defined in [A89].  This will be populated from the xDS cluster name, which will be passed to the ext_authz filter as described in [A60]. |

The following client-side metrics will be exported:

| Name          | Type  | Unit  | Labels  | Description |
| ------------- | ----- | ----- | ------- | ----------- |
| grpc.client_ext_authz.allowed_rpcs | Counter | {RPCs} | grpc.target, grpc.lb.backend_service | Number of RPCs that were allowed by the ext_authz server. |
| grpc.client_ext_authz.denied_rpcs | Counter | {RPCs} | grpc.target, grpc.lb.backend_service | Number of RPCs that were denied by the ext_authz server. |
| grpc.client_ext_authz.filter_disabled_rpcs | Counter | {RPCs} | grpc.target, grpc.lb.backend_service | Number of RPCs for which the filter was disabled. |
| grpc.client_ext_authz.failed_rpcs | Counter | {RPCs} | grpc.target, grpc.lb.backend_service | Number of RPCs for which the ext_authz call-out failed. |

#### Server-Side Metrics

The following server-side metrics will be exported:

| Name          | Type  | Unit  | Labels  | Description |
| ------------- | ----- | ----- | ------- | ----------- |
| grpc.server_ext_authz.allowed_rpcs | Counter | {RPCs} | | Number of RPCs that were allowed by the ext_authz server. |
| grpc.server_ext_authz.denied_rpcs | Counter | {RPCs} | | Number of RPCs that were denied by the ext_authz server. |
| grpc.server_ext_authz.filter_disabled_rpcs | Counter | {RPCs} | | Number of RPCs for which the filter was disabled. |
| grpc.server_ext_authz.failed_rpcs | Counter | {RPCs} | | Number of RPCs for which the ext_authz call-out failed. |

### Temporary environment variable protection

TODO: do we need a separate env var for client and server sides?

Support for the `ext_authz` filter will be guarded by the
`GRPC_EXPERIMENTAL_XDS_EXT_AUTHZ` environment variable.  This guard will
be removed once the feature passes interop tests.

## Rationale

We will ultimately need caching support for performance reasons, since
it will not scale to make an ext_authz RPC for every data plane RPC.
However, the design discussions around caching are still ongoing, and
we have use-cases that require use of ext_authz today.  Therefore, we
are decoupling the caching discussion to a separate gRFC to be published
later.

## Implementation

Will be implemented in C-core, Java, Go, and Node.
