A93: xDS ExtProc Support
----
* Author(s): @markdroth
* Approver: @ejona86, @dfawley
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2025-03-12
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

We will add support for the xDS ext_proc filter in gRPC, both on the
client and server sides.

## Background

The ext_proc filter provides support for making side-channel call-outs
to perform filtering.

The ext_proc filter will use the existing infrastructure for xDS
HTTP filters, which was originally introduced in gRFCs [A39].  We will
support this filter on both the gRPC client and server side.

Note that this filter will make use of the `allowed_grpc_services` map in
the bootstrap config, described in [A77].  It will also make use of the
`trusted_xds_server` server feature introduced in [A81].

### Related Proposals: 
* [A39: xDS HTTP Filter Support][A39]
* [A77: xDS Server-Side Rate Limiting][A77] (pending)
* [A81: xDS Authority Rewriting][A81]


* [A36: xDS-Enabled Servers][A36]
* [A41: xDS RBAC Support][A41]

[A39]: A39-xds-http-filters.md 
[A77]: https://github.com/grpc/proposal/pull/414
[A81]: A81-xds-authority-rewriting.md


[A36]: A36-xds-for-servers.md
[A41]: A41-xds-rbac.md

## Proposal



### Filter Configuration

We will support the following fields in the [`ExternalProcessor`
proto](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L101C9-L101C26):
- [grpc_service](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L129):
  This field must be present.  Inside of it:
  - [google_grpc](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/core/v3/grpc_service.proto#L303):
    This field must be present.  Inside of it:
    - [target_uri](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/core/v3/grpc_service.proto#L254):
      This field must be non-empty and must be a valid target URI.  The
      value specified here must be present in the `allowed_grpc_services`
      map in the bootstrap config, which will also determine the credentials
      to use, as described in [A77].
    - All other fields are ignored.
  - All other fields are ignored.
- [failure_mode_allow](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L177)
- [processing_mode](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L181C18-L181C33):
  Required.  Inside of it:
  - [request_header_mode](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/processing_mode.proto#L118C18-L118C37),
    [response_header_mode](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/processing_mode.proto#L121C18-L121C38),
    and
    [response_trailer_mode](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/processing_mode.proto#L133)
    will be supported.
  - [request_body_mode](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/processing_mode.proto#L124)
    and
    [response_body_mode](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/processing_mode.proto#L127):
    The only modes we support here are
    [`NONE`](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/processing_mode.proto#L66)
    and `GRPC` (to be added).
  - We ignore the request_trailer_mode field, since gRPC never sends
    request trailers.
- [request_attributes](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L188)
  and
  [response_attributes](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L195C19-L195C38)
- [message_timeout](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L205):
  If present, the value must obey the restrictions specified in the
  [`google.protobuf.Duration`
  documentation](https://developers.google.com/protocol-buffers/docs/reference/google.protobuf#google.protobuf.Duration),
  and it must have a positive value.
- [mutation_rules](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L225C55-L225C69):
  TODO: Figure out how these rules apply to gRPC.
- [max_message_timeout](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L230C28-L230C47):
  If present, the value must obey the restrictions specified in the
  [`google.protobuf.Duration`
  documentation](https://developers.google.com/protocol-buffers/docs/reference/google.protobuf#google.protobuf.Duration),
  and it must have a positive value.
- [forward_rules](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L237C25-L237C38):
  TODO: Which headers do we allow forwarding with or without
  `trusted_xds_server`?
- [allow_mode_override](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L249C8-L249C27)
- [disable_immediate_response](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L256C8-L256C34)
- [observability_mode](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L283C8-L283C26):
  TODO: Figure out flow control story!
- [deferred_close_timeout](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L305C28-L305C50):
  If present, the value must obey the restrictions specified in the
  [`google.protobuf.Duration`
  documentation](https://developers.google.com/protocol-buffers/docs/reference/google.protobuf#google.protobuf.Duration),
  and it must have a positive value.
- [send_body_without_waiting_for_header_response](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L321C8-L321C53)
- [allowed_override_modes](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L331C27-L331C49):
  Same validation rules as processing_mode field above.

The following fields will be ignored by gRPC:
- http_service: It doesn't make sense for gRPC to support non-gRPC
  mechanisms for contacting the ext_authz server.
- stat_prefix: This does not apply to gRPC.
- filter_metadata, metadata_options, on_processing_response: gRPC does not
  currently support dynamic metadata.
- disable_clear_route_cache, route_cache_action: We don't currently support
  recomputing the route.  We could consider adding this in the future if we
  have a use-case for it.

We will support the following fields in the per-route config:
- [overrides](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L406C22-L406C31):
  Optional.  Inside of it:
  - [processing_mode](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L414):
    Same as in top-level filter config.
  - [grpc_service](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L431C30-L431C42):
    Same as in top-level filter config.
  - [grpc_initial_metadata](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L444C39-L444C60):
    TODO: Why is this only in the per-route override config?
  - We will ignore the metadata_options field.

TODO: If we need the `disabled` flag, maybe just support that via the
`FilterConfig.disabled` field in `typed_per_filter_config` instead of
doing it in an ext_proc-specific way?  Just make sure that the semantics
are the same -- ext_proc config says "A set of overrides in a more
specific configuration will override a "disabled" flag set in a
less-specific one."

### Communication With the ext_proc Server

TODO: Fill this in!

The [`ProcessingRequest`
message](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L62)
sent to the server will be populated as follows:
- 

We will handle the [`ProcessingResponse`](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L132C9-L132C27)
as follows:
- 

### Header Rewriting

TODO: Update!

gRPC will support rewriting the `:authority` field only if the
`trusted_xds_server` server feature is present in the bootstrap config,
regardless of what settings are present in the ext_authz filter config.

If the ext_authz server attempts to overwrite the `host` header, that
change will actually apply to the `:authority` header instead.

Note that gRPC will not support rewriting the `:scheme` or `:method`
headers, regardless of the value of this field.

### Temporary environment variable protection

[Name the environment variable(s) used to enable/disable the feature(s) this proposal introduces and their default(s).  Generally, features that are enabled by I/O should include this type of control until they have passed some testing criteria, which should also be detailed here.  This section may be omitted if there are none.]

## Rationale

[A discussion of alternate approaches and the trade offs, advantages, and disadvantages of the specified approach.]


## Implementation

[A description of the steps in the implementation, who will do them, and when.  If a particular language is going to get the implementation first, this section should list the proposed order.]

## Open issues (if applicable)

[A discussion of issues relating to this proposal for which the author does not know the solution. This section may be omitted if there are none.]
