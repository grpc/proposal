A93: xDS ExtProc Support
----
* Author(s): @markdroth
* Approver: @ejona86, @dfawley
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2025-07-10
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

We will add support for the xDS ext_proc filter in gRPC, both on the
client and server sides.

## Background

The ext_proc filter provides support for making side-channel call-outs
to perform filtering/interception.

The ext_proc filter will use the existing infrastructure for xDS HTTP
filters, described in gRFCs [A39].  We will support this filter on both
the gRPC client and server side.

Note that this filter will make use of the `allowed_grpc_services` map in
the bootstrap config, described in [A77].  It will also make use of the
`trusted_xds_server` server feature introduced in [A81].

### Related Proposals: 
* [A39: xDS HTTP Filter Support][A39]
* [A77: xDS Server-Side Rate Limiting][A77] (pending)
* [A81: xDS Authority Rewriting][A81]

[A39]: A39-xds-http-filters.md 
[A77]: https://github.com/grpc/proposal/pull/414
[A81]: A81-xds-authority-rewriting.md

## Proposal

We will support the ext_proc filter in gRPC on both the client and
server side.

### Payload Handling

The existing ext_proc protocol's handling of request payloads has a
significant impedence mismatch with gRPC.

The ext_proc protocol is designed for HTTP payloads, meaning that the
ext_proc client sends the raw contents of the HTTP/2 DATA frames to
the ext_proc server.  However, gRPC has its own framing of individual
messages inside of the HTTP/2 DATA frames.  An ext_proc server accessing
the payload for a gRPC stream is really going to be interested only in
the deframed gRPC messages, one at a time, not the raw DATA frames.

This means that any existing ext_proc server that was designed to handle
gRPC traffic is going to have to handle deframing the gRPC messages from
the HTTP/2 DATA frames.  It will also need to handle buffering while
the payload is sent to it in chunks, because a single gRPC message could
be spread across multiple HTTP/2 DATA frames.  This is a fair amount of
work for the ext_proc server to do, so it seems better for the ext_proc
client to do the work of deframing the gRPC messages, and sending only
complete deframed messages to the ext_proc server, one at a time.

Furthermore, the ext_proc filter in gRPC will not actually have access
to the raw HTTP/2 DATA frames in the first place.  In our architecture,
filters see individual gRPC messages, and the framing/deframing is handled
in the transport layer.  This means that an ext_proc filter running on the
gRPC client side will see the messages before they have been framed to be
sent by the transport, and an ext_proc filter running on the gRPC server
side will see the messages after they have been received and deframed by
the transport.

Therefore, we propose adding a new ext_proc `BodySendMode` called
`GRPC`.  In this mode, the ext_proc client would handle deframing the
gRPC messages, and it would send each gRPC message to the ext_proc server
as a separate `request_body`.  This would be the only processing mode
supported in gRPC.  We would request that Envoy implement the same mode,
so that users can switch back and forth between proxy and proxyless data
planes without breaking their ext_proc servers.

### Filter Configuration

We will support the following fields in the [`ExternalProcessor`
proto](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L101):
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
- [processing_mode](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L181):
  Required.  Inside of it:
  - [request_header_mode](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/processing_mode.proto#L118),
    [response_header_mode](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/processing_mode.proto#L121),
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
  [response_attributes](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L195)
- [message_timeout](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L205):
  If present, the value must obey the restrictions specified in the
  [`google.protobuf.Duration`
  documentation](https://developers.google.com/protocol-buffers/docs/reference/google.protobuf#google.protobuf.Duration),
  and it must have a positive value.
- [mutation_rules](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L225):
  Optional.  Inside of it:
  - [disallow_all](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/common/mutation_rules/v3/mutation_rules.proto#L70)
  - [allow_expression](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/common/mutation_rules/v3/mutation_rules.proto#L75)
  - [disallow_expression](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/common/mutation_rules/v3/mutation_rules.proto#L79)
  - [disallow_is_error](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/common/mutation_rules/v3/mutation_rules.proto#L87)
  - allow_all_routing, disallow_system, allow_envoy: These fields will be ignored.
- [max_message_timeout](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L230):
  If present, the value must obey the restrictions specified in the
  [`google.protobuf.Duration`
  documentation](https://developers.google.com/protocol-buffers/docs/reference/google.protobuf#google.protobuf.Duration),
  and it must have a positive value.
- [forward_rules](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L237)
- [allow_mode_override](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L249)
- [disable_immediate_response](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L256)
- [observability_mode](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L283):
  TODO: Figure out flow control story!
- [deferred_close_timeout](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L305):
  If present, the value must obey the restrictions specified in the
  [`google.protobuf.Duration`
  documentation](https://developers.google.com/protocol-buffers/docs/reference/google.protobuf#google.protobuf.Duration),
  and it must have a positive value.
- [send_body_without_waiting_for_header_response](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L321)
- [allowed_override_modes](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L331):
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

### Filter Configuration Overrides

We will support `typed_per_filter_config` config overrides for this
filter, as described in [A39].

We will support the following fields in the
[`ExtProcPerRoute`](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L395)
proto:
- [overrides](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L406):
  Optional.  Inside of it:
  - [processing_mode](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L414):
    Same as in top-level filter config.
  - [grpc_service](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L431):
    Same as in top-level filter config.
  - [grpc_initial_metadata](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L444)
  - [request_attributes](https://github.com/envoyproxy/envoy/blob/72833beab4fdc87f7fc53ec31ab70fd734581720/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L442)
    and
    [response_attributes](https://github.com/envoyproxy/envoy/blob/72833beab4fdc87f7fc53ec31ab70fd734581720/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L447)
  - [failure_mode_allow](https://github.com/envoyproxy/envoy/blob/72833beab4fdc87f7fc53ec31ab70fd734581720/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L468C29-L468C47)
  - We will ignore the metadata_options field.

Note that we will not use the [`disabled`
field](https://github.com/envoyproxy/envoy/blob/72833beab4fdc87f7fc53ec31ab70fd734581720/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L420)
in the `ExtProcPerRoute` proto itself; instead, we will support disabling
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

### Communication With the ext_proc Server

The [`ProcessingRequest`
message](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L62)
sent to the server will be populated as follows:
- [request_headers](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L76)
  and
  [response_headers](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L81).
  Inside of them:
  - [headers](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L216)
  - [end_of_stream](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L227)
- [request_body](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L85)
  and
  [response_body](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L89).
  Inside of them:
  - [body](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L235)
  - [end_of_stream](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L239)
- [response_trailers](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L103).
  Inside of it:
  - [trailers](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L247)
- [attributes](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L113)
- [observability_mode](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L126)
- Note: We will not populate request_trailers, because gRPC never sends
  request trailers.
- Note: We will not populate metadata_context, because gRPC does not
  support dynamic metadata.

We will handle the [`ProcessingResponse`](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L132)
as follows:
- [request_headers](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L139)
  and
  [response_headers](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L143).
  Inside of them:
  - [response](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L257).
    Inside of it:
    - [status](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L303):
      This must be `CONTINUE`.  gRPC will not support
      `CONTINUE_AND_REPLACE`, because those semantics don't work with
      gRPC.
    - [header_mutation](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L308):
      Inside of it:
      - [set_headers](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L375)
      - [remove_headers](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L379)
    - Note: We do not support body_mutation in response to headers.
    - Note: We do not support trailers, since that works only with
      `CONTINUE_AND_REPLACE`.
    - Note: We do not support clear_route_cache.
- [request_body](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L147)
  and
  [response_body](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L151):
  Inside of them:
  - [response](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L257).
    Inside of it:
    - [body_mutation](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L317)
      - [body](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L401)
      - Note: We do not support clear_body or streamed_response.
    - Note: We do not support header mutations in response to a body.
    - Note: We do not support trailers, since that works only with
      `CONTINUE_AND_REPLACE`.
    - Note: We do not support clear_route_cache.
- [response_trailers](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L159):
  - [header_mutation](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L273)
    Inside of it:
    - [set_headers](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L375)
    - [remove_headers](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L379)

### Header Rewriting

gRPC will not support rewriting the `:scheme`, `:method`, `:path`,
`:authority`, or `host` headers, regardless of what settings are present
in the ext_proc filter config. If the server specifies a rewrite for
one of these headers, that rewrite will be ignored.

### Temporary environment variable protection

Support for the ext_authz filter will be guarded by the
`GRPC_EXPERIMENTAL_XDS_EXT_PROC` environment variable. This guard will
be removed once the feature passes interop tests.

## Rationale

For payload handling, we could have considered having the gRPC ext_proc
filter artificially add the 5-byte gRPC frame header to each message
that we send to the ext_proc server.  However, this seems like a
sub-optimal approach, because (a) it would waste bandwidth sending data
that really isn't useful, (b) it would not avoid the need for the
ext_proc server to handle the gRPC framing, and (c) it would further
spread use of the gRPC framing to ext_proc servers, which will make it
awkward for gRPC to support other transports with different framing in
the future.

## Implementation

Will be implemented in C-core, Java, Go, and Node.
