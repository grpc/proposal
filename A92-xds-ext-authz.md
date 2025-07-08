A92: xDS ExtAuthz Support
----
* Author(s): @markdroth
* Approver: @ejona86, @dfawley
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2025-07-07
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

We will add support for the xDS ext_authz filter in the gRPC server.

## Background

The ext_authz filter provides support for servers making side-channel
call-outs to perform authorization decisions.

The ext_authz filter will use the existing infrastructure for xDS HTTP
servers in the gRPC server, which was originally introduced in gRFCs [A39]
and [A36].  This infrastructure has previously been used for the RBAC
filter, described in [A41], and the rate-limiting filter, described in
[A77].

Note that this filter will make use of the `allowed_grpc_services` map in
the bootstrap config, described in [A77].  It will also make use of the
`trusted_xds_server` server feature introduced in [A81].

### Related Proposals: 
* [A39: xDS HTTP Filter Support][A39]
* [A36: xDS-Enabled Servers][A36]
* [A41: xDS RBAC Support][A41]
* [A77: xDS Server-Side Rate Limiting][A77] (pending)
* [A81: xDS Authority Rewriting][A81]
* [A33: xDS Fault Injection][A33]

[A36]: A36-xds-for-servers.md
[A39]: A39-xds-http-filters.md 
[A41]: A41-xds-rbac.md
[A77]: https://github.com/grpc/proposal/pull/414
[A81]: A81-xds-authority-rewriting.md
[A33]: A33-Fault-Injection.md

## Proposal

### Filter Configuration

We will support the following fields in the [`ExtAuthz`
proto](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_authz/v3/ext_authz.proto#L34):
- [grpc_service](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_authz/v3/ext_authz.proto#L45):
  This field must be present.  Inside of it:
  - [google_grpc](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/core/v3/grpc_service.proto#L303):
    This field must be present.  Inside of it:
    - [target_uri](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/core/v3/grpc_service.proto#L254):
      This field must be non-empty and must be a valid target URI.  If
      the `trusted_xds_server` server feature (see [A81]) is *not* set in
      the bootstrap config, then the value specified here must be present
      in the `allowed_grpc_services` map in the bootstrap config, which
      will also determine the credentials to use, as described in [A77].
    - [channel_credentials](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/core/v3/grpc_service.proto#L256):
      Used only if `trusted_xds_server` server feature is present in the
      bootstrap config.
      TODO: flesh this out
    - [call_credentials](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/core/v3/grpc_service.proto#L260):
      Used only if `trusted_xds_server` server feature is present in the
      bootstrap config.
      TODO: flesh this out
    - [credentials_factory_name](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/core/v3/grpc_service.proto#L276)
      and
      [config](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/core/v3/grpc_service.proto#L280):
      These fields are used to configure channel credentials using the
      same channel credentials registry that we use in the xDS bootstrap
      file.  If `credentials_factory_name` is set, it takes precedence
      over `channel_credentials`.
    - Note: All other fields are ignored.
  - [timeout](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/core/v3/grpc_service.proto#L308):
    Specifies the deadline for the RPCs sent to the ext_authz server.
    If unset, there is no deadline.  The value must obey the restrictions
    specified in the [`google.protobuf.Duration`
    documentation](https://developers.google.com/protocol-buffers/docs/reference/google.protobuf#google.protobuf.Duration),
    and it must have a positive value.
  - All other fields are ignored.
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

### Filter Configuration Overrides

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
we will honor both the `disabled` field in both the [`HttpFilter`
message](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#L1229)
in the HCM config and in the [`FilterConfig` wrapper
message](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/route/v3/route_components.proto#L2562)
that can be used in `typed_per_filter_config` fields.

Note that the most specific `typed_per_filter_config` will be used.  For
example, if there is an override config at the virtual host level that
disables the filter but then another config at the route level that does
not disable the filter, the filter will be enabled.

### Communication With the ext_authz Server

The [`AttributeContext`
message](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L43)
sent to the server will be populated as follows:
- [source](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L192): Will always be set.  Inside it:
  - [address](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L58)
    will be set to the peer address of the connection that the request
    came in on.
  - [principal](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L82):
    If TLS is used and the client provided a valid certificate, this will be
    set to the cert's first URI SAN if set, otherwise the cert's first DNS
    SAN if set, otherwise the subject field of the certificate in RFC
    2253 format.  If TLS is not used or the client did not provide a
    cert, this field will be unset.
  - [certificate](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L86): This will be populated if configured.
  - service, labels: Will *not* be set.
- [destination](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L197):
  Will always be set.  Inside it:
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
      Will be set based on config.
    - [path](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L148):
      Will always be set.
    - [size](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L165):
      Will always be set to -1.
    - [protocol](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L171):
      Always set to "HTTP/2".
    - id, scheme, headers, query, fragment, body, raw_body: Will *not* be set.
- context_extensions, metadata_context, route_metadata_context,
  tls_session: Will *not* be set.

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
  - [headers](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/external_auth.proto#L55)
  - body: Ignored; does not apply to gRPC.
- [ok_response](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/external_auth.proto#L134):
  - [headers](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/external_auth.proto#L75),
    [headers_to_remove](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/external_auth.proto#L92),
    [response_headers_to_add](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/external_auth.proto#L104):
    See [header rewriting](#header-rewriting) below for details.
  - query_parameters_to_set, query_parameters_to_remove: Ignored; these
    do not apply to gRPC.
- dynamic_metadata: Ignored.

### Header Rewriting

gRPC will not support rewriting the `:scheme`, `:method`, `:path`,
`:authority`, or `host` headers, regardless of what settings are present
in the ext_authz filter config.  If the server specifies a rewrite for
one of these headers, that rewrite will be ignored.

### Temporary environment variable protection

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
