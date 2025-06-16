A92: xDS ExtAuthz Support
----
* Author(s): @markdroth
* Approver: @ejona86, @dfawley
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2025-03-10
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

[A36]: A36-xds-for-servers.md
[A39]: A39-xds-http-filters.md 
[A41]: A41-xds-rbac.md
[A77]: https://github.com/grpc/proposal/pull/414
[A81]: A81-xds-authority-rewriting.md

## Proposal

### Filter Configuration

We will support the following fields in the [`ExtAuthz`
proto](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_authz/v3/ext_authz.proto#L34):
- [grpc_service](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_authz/v3/ext_authz.proto#L45):
  This field must be present.  Inside of it:
  - [google_grpc](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/core/v3/grpc_service.proto#L303):
    This field must be present.  Inside of it:
    - [target_uri](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/core/v3/grpc_service.proto#L254):
      This field must be non-empty and must be a valid target URI.  The
      value specified here must be present in the `allowed_grpc_services`
      map in the bootstrap config, which will also determine the credentials
      to use, as described in [A77].
    - Note: All other fields are ignored.
  - [timeout](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/core/v3/grpc_service.proto#L308):
    Specifies the deadline for the RPCs sent to the ext_authz server.
    If unset, there is no deadline.  The value must obey the restrictions
    specified in the [`google.protobuf.Duration`
    documentation](https://developers.google.com/protocol-buffers/docs/reference/google.protobuf#google.protobuf.Duration),
    and it must have a positive value.
  - All other fields are ignored.
- [failure_mode_allow](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_authz/v3/ext_authz.proto#L68)
- [failure_mode_allow_header_add](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_authz/v3/ext_authz.proto#L74)
- [status_on_error](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_authz/v3/ext_authz.proto#L95): Note that this field specifies an HTTP status code,
  not a gRPC status code.  The gRPC status code will be determined using
  the normal [HTTP-to-gRPC status conversion
  rules](https://github.com/grpc/grpc/blob/master/doc/http-grpc-status-mapping.md).
- [allowed_headers](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_authz/v3/ext_authz.proto#L229):
  TODO: Want to not allow sending certain sensitive headers unless
  `trusted_xds_server` is present, but hard with regex matches...
  Maybe just don't allow matchers to match against sensitive headers?
- [disallowed_headers](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_authz/v3/ext_authz.proto#L233)
- [decoder_header_mutation_rules](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_authz/v3/ext_authz.proto#L282):
  Optional.  Inside of it:
  - [allow_all_routing](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/common/mutation_rules/v3/mutation_rules.proto#L52): This field will control only whether the
    ext_authz server can overwrite the `:authority` header.  If this
    field is set to true and the `trusted_xds_server` server feature is
    not present in the bootstrap config, we will reject the config.
  - [allow_envoy](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/common/mutation_rules/v3/mutation_rules.proto#L59):
    TODO: Do we need this?  Those headers aren't really special to gRPC.
  - [disallow_system](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/common/mutation_rules/v3/mutation_rules.proto#L65):
    TODO: Should we allow these headers to be modified at all?
  - [disallow_all](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/common/mutation_rules/v3/mutation_rules.proto#L70)
  - [allow_expression](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/common/mutation_rules/v3/mutation_rules.proto#L75):
    TODO: Figure out how to handle matching a restricted header.  Maybe just
    don't allow it to match against those?
  - [disallow_expression](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/common/mutation_rules/v3/mutation_rules.proto#L79)
  - [disallow_is_error](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/common/mutation_rules/v3/mutation_rules.proto#L87)
- [include_peer_certificate](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_authz/v3/ext_authz.proto#L181)
- TODO: CACHING!

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
  enable_dynamic_metadata_ingestion, filter_metadata: gRPC does not currently
  support dynamic metadata.
- bootstrap_metadata_labels_key: We have no current use-case for this.  We
  could consider adding it in the future if there is a need.
- filter_enabled, filter_enabled_metadata, deny_at_disable: We don't currently
  support the runtime system.
- include_tls_session: We do not currently have a use-case for this, but
  we could consider adding it in the future if we do encounter such a
  use-case.

We will not support per-route config overrides for this filter.  TODO:
If we need the `disabled` flag, maybe just support that via the
`FilterConfig.disabled` field in `typed_per_filter_config` instead of
doing it in an ext_authz-specific way?  Just make sure that the
semantics are the same -- ext_authz config says "If disabled is
specified in multiple per-filter-configs, the most specific one will be
used."

### Communication With the ext_authz Server

The [`AttributeContext`
message](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L43)
sent to the server will be populated as follows:
- [source](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L192): Will always be set.  Inside it:
  - [address](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L58)
    and [service](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L65)
    will be set.
  - labels will not be set.
  - [principal](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L82)
    will be set if the client provided a cert and we validates it, unset
    otherwise.
  - [certificate](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L86): This will be populated if configured.
- [destination](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L197):
  This field will not be populated, because there is no destination on a gRPC
  server.
- [request](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L200):
  Will always be set.  Inside it:
  - [time](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L95):
    Will be set to the RPC's start time.
  - [http](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L98):
    Will always be set.  Inside of it:
    - [id](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L112): TODO: how do we set this?
    - [method](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L115):
      Will always be "POST".
    - [header_map](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L143):
      Will be set based on config.
    - [path](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L148):
      Will always be set.
    - [scheme](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L154):
      TODO: set based on whether TLS is used?
    - [size](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L165):
      Will always be set to -1.
    - [protocol](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/attribute_context.proto#L171):
      Always set to "HTTP/2".
    - headers, query, fragment, body, raw_body: Will *not* be set.
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
  - [headers](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/auth/v3/external_auth.proto#L55):
    TODO: Figure out what restrictions to apply here.
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

gRPC will support rewriting the `:authority` field only if the
`trusted_xds_server` server feature is present in the bootstrap config,
regardless of what settings are present in the ext_authz filter config.

If the ext_authz server attempts to overwrite the `host` header, that
change will actually apply to the `:authority` header instead.

Note that gRPC will not support rewriting the `:scheme` or `:method`
headers, regardless of the value of this field.

### Temporary environment variable protection

Support for the `ext_authz` filter will be guarded by the
`GRPC_EXPERIMENTAL_XDS_EXT_AUTHZ` environment variable.  This guard will
be removed once the feature passes interop tests.

## Rationale

[A discussion of alternate approaches and the trade offs, advantages, and disadvantages of the specified approach.]


## Implementation

[A description of the steps in the implementation, who will do them, and when.  If a particular language is going to get the implementation first, this section should list the proposed order.]

## Open issues (if applicable)

[A discussion of issues relating to this proposal for which the author does not know the solution. This section may be omitted if there are none.]
