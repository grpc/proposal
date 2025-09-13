A93: xDS ExtProc Support
----
* Author(s): @markdroth
* Approver: @ejona86, @dfawley
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2025-09-12
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

We will add support for the xDS ext_proc filter in gRPC, both on the
client and server sides.

## Background

The ext_proc filter provides support for making side-channel call-outs
to perform filtering/interception.

The ext_proc filter will use the existing infrastructure for xDS HTTP
filters, described in gRFC [A39].  We will support this filter on both
the gRPC client and server side.

Note that this filter will make use of the `allowed_grpc_services` map in
the bootstrap config, described in [A102].  It will also make use of the
`trusted_xds_server` server feature introduced in [A81].

### Related Proposals:
* [A39: xDS HTTP Filter Support][A39]
* [A81: xDS Authority Rewriting][A81]
* [A102: xDS GrpcService Support][A102] (pending)

[A39]: A39-xds-http-filters.md
[A81]: A81-xds-authority-rewriting.md
[A102]: https://github.com/grpc/proposal/pull/510

## Proposal

We will support the ext_proc filter in gRPC on both the client and
server side.

### Filter Behavior

TODO: ExtProc channel retention (simple approach for now, probably
globally shared channel is fine?)

For every event in the data plane RPC (client headers, client message,
client half-close, server headers, server message, and server trailers),
the filter can be configured to send the contents of that event to the
ext_proc server.  The responses from the ext_proc server will indicate
what modifications to make to the data plane RPC before allowing it to
proceed.  The specific behavior for each event is covered below.

For each data plane RPC, the first time the filter needs to send an event
to the ext_proc server, the filter will create a stream to the ext_proc
server.  That ext_proc stream will be associated with that data plane RPC,
and all communication with the ext_proc server for that specific data
plane RPC will be done on that ext_proc stream.  The filter will pass
the trace context from the data plane RPC to the ext_proc RPC, so that
the ext_proc RPC appears as a child span on the data plane RPC's trace.

Note that the stream to the ext_proc server may be terminated at any time.
If the stream terminates with OK status, that indicates to the filter that
it no longer needs to send any more events to the ext_proc server for that
data plane RPC; all remaining events may proceed on the data plane RPC
without any further action taken by the ext_proc filter.  If the stream
terminates with a non-OK status, then by default the data plane RPC will
be failed with UNAVAILABLE status.  However, if the `failure_mode_allow`
config field is set to true, then the data plane RPC will instead be
allowed to continue, with no further action taken by the ext_proc filter.

#### Events on the ext_proc Stream

On the ext_proc stream, the events sent and received must be in the same
order as on the data plane.  For client-to-server events, the order must
be headers, followed by zero or more messages, followed by a half-close.
For server-to-client events, the order must be headers, followed by zero
or more messages, followed by trailers.  It is fine to interleave
client-to-server events with server-to-client events, since the two
directions are independent of each other.  It is also fine for some of
the events to be missing on the ext_proc stream; for example, if the
filter is not configured to send client headers but is configured to
send client messages, then it can start by sending a client message.
But if two different events are sent on the stream, they must be in the
correct order relative to each other, both to and from the ext_proc
server.

When sending client headers, server headers, and server trailers events to
the ext_proc server, if not in [observability mode](#observability-mode),
the filter will hold on to the headers and wait for a response for that
event from the ext_proc server before allowing the event to proceed on
the data plane RPC.  The response from the ext_proc server will tell
the filter what modifications to make to the headers before allowing
them to proceed on the data plane RPC.

In contrast, when sending client or server messages to the ext_proc
server, if not in [observability mode](#observability-mode), the
filter will *not* hold on to the contents of those messages.  Instead,
the ext_proc server will be responsible for sending back the full set of
messages to be used on the data plane RPC.  That set of messages may have
absolutely no relationship to the messages originally sent to the ext_proc
server; the ext_proc server may decide to drop messages, modify messages,
pass messages back unmodified, or completely replace all of the messages
on the stream.  Note that the number of messages produced by the ext_proc
server may not match the number of messages originally sent to it.

Events on the data plane RPC should be sent on the ext_proc stream as
they occur, even if the filter has not yet received a response from the
ext_proc server for a previous event.  For example, if the filter is
waiting for a response for client headers, if it sees a client message,
it can send the client message event on the ext_proc stream immediately
(assuming it is configured to send client messages).

#### Observability Mode

If the
[`observability_mode`](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L283)
config field is set to true, the contents of the data plane RPC events
are sent to the ext_proc server, but the ext_proc server is not expected
to make any modifications to the data plane RPC, so the data plane
RPC proceeds without waiting for a response from the ext_proc server.
Note that in this mode, all messages on the ext_proc stream will have the
[`observability_mode`](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L126)
field set.

One challenge in this mode is flow control: if we are blocked sending
a message to the ext_proc server by flow control, then we need some
push-back on the data plane RPC, or else we would have to buffer messages
to be sent to the ext_proc server, which can cause OOMs.  (Envoy has
noted this problem in https://github.com/envoyproxy/envoy/issues/33319
but has not proposed a solution yet.)

For gRPC, we will address this problem by requiring the message to the
ext_proc server to pass flow control before we allow the message to
proceed on the data plane RPC.  Due to differences in flow control
semantics across languages, this will look a little different in each
one:

- In C-core, we will wait for the write to complete on the ext_proc
  stream before we allow the message to continue on the data plane RPC.
- In Java, the application has to explicitly check whether there is flow
  control push-back before doing a write.  For the purposes of that API,
  Java will not consider the flow control available until it passes flow
  control on both the ext_proc stream and on the data plane stream.
- In Go, a write is blocking and doesn't return until the write passed
  flow control.  So observability mode doesn't need to do anything
  special to handle flow control; it will simply not wait for the
  ext_proc response before sending the message on the data plane stream.

#### Payload Handling

The existing ext_proc protocol's handling of request payloads has a
significant impedence mismatch with gRPC.

The ext_proc protocol is designed for HTTP payloads, meaning that the
ext_proc client sends the raw contents of the HTTP/2 DATA frames to
the ext_proc server.  However, gRPC has its own framing of individual
messages inside of the HTTP/2 DATA frames.  An ext_proc server accessing
the payload for a gRPC stream is really going to be interested only in
the deframed gRPC messages, one at a time, not the raw DATA frames.

This means that any existing ext_proc server that wants to handle
gRPC traffic is going to have to handle deframing the gRPC messages from
the HTTP/2 DATA frames.  It will also need to handle buffering while
the payload is sent to it in chunks, because a single gRPC message could
be spread across multiple HTTP/2 DATA frames.  This is a fair amount of
work for the ext_proc server to do, so it seems better for the ext_proc
client to do the work of deframing the gRPC messages, and sending only
complete deframed messages to the ext_proc server, one at a time.

More seriously, ext_proc currently assumes that all of the contents of
the DATA frames are a single payload, and the ext_proc server is only
allowed to send a single response to modify that payload.  That model is
incompatible with gRPC streaming semantics, where the ext_proc server
may need to modify each message individually and cannot wait until the
end of the stream to do so.

Furthermore, the ext_proc filter in gRPC will not actually have access
to the raw HTTP/2 DATA frames in the first place.  In our architecture,
the framing/deframing is handled in the transport layer, and filters
see only individual gRPC messages.  This means that an ext_proc filter
running on the gRPC client side will see the messages before they have
been framed to be sent by the transport, and an ext_proc filter running on
the gRPC server side will see the messages after they have been received
and deframed by the transport.

Therefore, we propose adding a new ext_proc `BodySendMode` called `GRPC`
(see https://github.com/envoyproxy/envoy/pull/38753).  In this mode, the
ext_proc client would handle deframing the gRPC messages.  It will send
each gRPC message to the ext_proc server as a separate `request_body`,
and the ext_proc server will send a separate response for each one.
This is a streaming mode similar to the existing `FULL_DUPLEX_STREAMED`
mode.

The new `GRPC` mode will be the only processing mode supported in gRPC.
It is be desirable for Envoy to implement the same mode, so that users
can switch back and forth between proxy and proxyless data planes without
breaking their ext_proc servers.

#### Metrics

TODO: define metrics for tracking time between entering and exiting the
ext_proc filter for the following events:
- client headers
- client half-close
- server headers
- server trailers

### Filter Configuration

The filter supports both a top-level configuration and an override
config.

#### Top-Level Configuration

We will support the following fields in the [`ExternalProcessor`
proto](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L101):
- [grpc_service](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L129):
  This field must be present.  It will be validated as described in [A102].
- [failure_mode_allow](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L177):
  By default, if the RPC to the ext_proc server fails with a non-OK
  status, the data plane RPC will be failed with status UNAVAILABLE.  If
  this field is set to true, then the data plane RPC will instead be
  allowed to continue, with no further action taken by the ext_proc filter.
- [processing_mode](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L181):
  Required.  Inside of it:
  - [request_header_mode](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/processing_mode.proto#L118),
    [response_header_mode](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/processing_mode.proto#L121),
    and
    [response_trailer_mode](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/processing_mode.proto#L133)
    will be supported as described in the proto file.
  - [request_body_mode](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/processing_mode.proto#L124)
    and
    [response_body_mode](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/processing_mode.proto#L127):
    The only modes we support here are
    [`NONE`](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/processing_mode.proto#L66)
    and `GRPC` (to be added).
  - We ignore the request_trailer_mode field, since gRPC never sends
    request trailers.
- [allow_mode_override](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L249):
  If true, allows the ext_proc server to dynamically override the
  processing mode for an individual data plane RPC.
- [allowed_override_modes](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L331):
  Ignored unless `allow_mode_override` is true.  This list indicates the
  set of allowed override modes, ignoring the request header mode.  If
  this list is empty, then any override sent by the server is honored.
- [request_attributes](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L188)
  and
  [response_attributes](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L195):
  Attributes to be sent to ext_proc server along with client-to-server
  and server-to-client events, respectively.  The set of supported
  attributes is the same as what we support for any CEL expression in xDS.
  any unsupported attribute name will be ignored.  See [Attributes Sent to
  ext_proc Server](#attributes-sent-to-the-ext_proc-server) below for details.
- [mutation_rules](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L225):
  Optional.  Inside of it:
  - [disallow_all](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/common/mutation_rules/v3/mutation_rules.proto#L70):
    If true, disallows all header mutations from the ext_proc server.
  - [allow_expression](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/common/mutation_rules/v3/mutation_rules.proto#L75):
    If set, specifically allows any matching header that is not also
    matched by `disallow_expression`.  Note that regexes should be checked
    for validity as part of resource validation.
  - [disallow_expression](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/common/mutation_rules/v3/mutation_rules.proto#L79):
    If set, specifically disallows any matching header.  Note that regexes
    should be checked for validity as part of resource validation.
  - [disallow_is_error](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/common/mutation_rules/v3/mutation_rules.proto#L87):
    If true, a disallowed header will cause the filter to fail the data
    plane RPC with INTERNAL status.
  - allow_all_routing, disallow_system, allow_envoy: These fields will be ignored.
- [forward_rules](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L237):
  Configures which headers from the data plane RPC will be included in
  the message to the ext_proc server.  In this message:
  - [allowed_headers](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L386)
    and
    [disallowed_headers](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L390):
    These fields behave as documented in the proto file.
- [disable_immediate_response](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L256):
  If true, then if the ext_proc server sends a message with the
  `immediate_response` field populated, the filter will act as if the
  ext_proc stream terminated with a non-OK status.
- [observability_mode](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L283):
  See [Observability Mode](#observability-mode) below.
- [deferred_close_timeout](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L305):
  In observability mode, the data plane stream may terminate before the
  ext_proc server has finished reading all data off of the ext_proc
  stream.  To avoid that, we delay closing the ext_proc stream from the
  client side for a short time after the data plane stream is destroyed.
  If unset, the delay is 5 seconds.  If present, the value must obey the
  restrictions specified in the [`google.protobuf.Duration`
  documentation](https://developers.google.com/protocol-buffers/docs/reference/google.protobuf#google.protobuf.Duration),
  and it must have a positive value.

The following fields will be ignored by gRPC:
- message_timeout and max_message_timeout: Message timeouts do not make
  sense in GRPC body send mode.
- http_service: It doesn't make sense for gRPC to support non-gRPC
  mechanisms for contacting the ext_authz server.
- stat_prefix: This does not apply to gRPC.
- filter_metadata, metadata_options, on_processing_response: gRPC does not
  currently support dynamic metadata.
- disable_clear_route_cache, route_cache_action: We don't currently support
  recomputing the route.  We could consider adding this in the future if we
  have a use-case for it.
- send_body_without_waiting_for_header_response: This is relevant only
  in `STREAMED` body send mode, which gRPC does not support.

#### Override Configuration

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

This section describes the protocol for communication with the ext_proc
server.

#### Messages Sent To the ext_proc Server

The [`ProcessingRequest`
message](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L62)
sent to the server will be populated as follows:
- [request_headers](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L76)
  and
  [response_headers](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L81).
  Populated when sending client headers or server headers, respectively.
  Inside of them:
  - [headers](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L216):
    Contains the headers.
  - [end_of_stream](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L227):
    This will always be false for client headers.  For server headers,
    it will be true when the server sends a Trailers-Only response.
- [request_body](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L85)
  and
  [response_body](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L89).
  Populated when sending a client message or server message, respectively.
  Inside of them:
  - [body](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L235):
    Contains the serialized message.  Will be empty if the client sends
    a half-close when there is no message to send (i.e., if the client
    never sent any message on the stream, or if the half-close is sent
    after the filter has already sent the last message to the ext_proc
    server).
  - [end_of_stream](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L239):
    For client messages, may be true if the client sent a half-close at
    the same time as the last message.  For server messages, will always
    be false.
- [response_trailers](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L103).
  Populated when sending server trailers.  Inside of it:
  - [trailers](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L247):
    Contains the trailers.
- [attributes](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L113):
  See [Attributes Sent to ext_proc
  Server](#attributes-sent-to-the-ext_proc-server) below.
- [observability_mode](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L126):
  Will be set to the value of the `observability_mode` config field.
- [protocol_config](https://github.com/envoyproxy/envoy/blob/564612e32eafc10a7a7fd490cdb5cc7149e5802b/api/envoy/service/ext_proc/v3/external_processor.proto#L153):
  Populated only on the first message that the filter sends on the
  ext_proc stream.  Inside of it:
  - [request_body_mode](https://github.com/envoyproxy/envoy/blob/564612e32eafc10a7a7fd490cdb5cc7149e5802b/api/envoy/service/ext_proc/v3/external_processor.proto#L67)
    and
    [response_body_mode](https://github.com/envoyproxy/envoy/blob/564612e32eafc10a7a7fd490cdb5cc7149e5802b/api/envoy/service/ext_proc/v3/external_processor.proto#L72):
    Populated from the filter config's processing mode.
  - The send_body_without_waiting_for_header_response field will never
    be set, since that applies only in STREAMED body send mode.
- Note: We will not populate request_trailers, because gRPC never sends
  request trailers.
- Note: We will not populate metadata_context, because gRPC does not
  support dynamic metadata.

#### Attributes Sent to ext_proc Server

The
[`attributes`](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L113)
field in the ext_proc request will be populated based on the filter's
configuration.  The field will have only one entry, whose key will be
the fixed string `envoy.filters.http.ext_proc`, and the value will be a
`google.protobuf.Struct` message that contains a map from attribute name
to attribute value.

The list of attributes to include is specified by the
[`request_attributes`](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L188)
config field, for client-to-server events, or the
[`response_attributes`](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/extensions/filters/http/ext_proc/v3/ext_proc.proto#L195)
config field, for server-to-client events.  The set of supported attribute
names is the same as what we support for any CEL expression in xDS.

For example, consider the case where the `request_attributes` config
field contains the attribute `request.path` and the filter is processing
an RPC to the method `Service.Method`.  When sending client headers,
client messages, or client half-close to the ext_proc server, the
`google.protobuf.Struct` message will contain an entry with key
`request.path` and value `/Service/Method`.

#### Messages Received From the ext_proc Server

We will handle the [`ProcessingResponse`](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L132)
as follows:
- [request_headers](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L139)
  and
  [response_headers](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L143).
  Sent in response to client headers and server headers, respectively.
  Inside of them:
  - [response](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L257).
    Inside of it:
    - [status](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L303):
      This must be `CONTINUE`.  gRPC will not support `CONTINUE_AND_REPLACE`,
      because those semantics don't work with gRPC; if that value is seen,
      the filter will cancel the ext_proc stream and treat it as if it
      failed with a non-OK status (see above for details).
    - [header_mutation](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L308):
      Header mutations.  See [Header rewriting](#header-rewriting) below.
    - Note: We do not support body_mutation in response to headers.
      This field will be ignored in this context.
    - Note: We do not support trailers, since that works only with
      `CONTINUE_AND_REPLACE`.  This field will be ignored.
    - Note: We do not support clear_route_cache.
- [request_body](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L147)
  and
  [response_body](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L151):
  Sent in response to client messages and server messages, respectively.
  Inside of them:
  - [response](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L257).
    Inside of it:
    - [status](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L303):
      This must be `CONTINUE`.  gRPC will not support `CONTINUE_AND_REPLACE`,
      because those semantics don't work with gRPC; if that value is seen,
      the filter will cancel the ext_proc stream and treat it as if it
      failed with a non-OK status (see above for details).
    - [body_mutation](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L317):
      - [streamed_response](https://github.com/envoyproxy/envoy/blob/564612e32eafc10a7a7fd490cdb5cc7149e5802b/api/envoy/service/ext_proc/v3/external_processor.proto#L447):
        Replaces the original serialized message.  Within it:
        - [body](https://github.com/envoyproxy/envoy/blob/564612e32eafc10a7a7fd490cdb5cc7149e5802b/api/envoy/service/ext_proc/v3/external_processor.proto#L419):
          The serialized message body.
        - [end_of_stream](https://github.com/envoyproxy/envoy/blob/564612e32eafc10a7a7fd490cdb5cc7149e5802b/api/envoy/service/ext_proc/v3/external_processor.proto#L424C8-L424C21):
          If true, indicates that a half-close should be sent after the
          message.  Honored only on client-to-server messages.
      - Note: We do not support body or clear_body.
    - Note: We do not support header_mutation in response to a body.
      This field will be ignored in this context.
    - Note: We do not support trailers, since that works only with
      `CONTINUE_AND_REPLACE`.  This field will be ignored.
    - Note: We do not support clear_route_cache.
- [response_trailers](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L159):
  Sent in response to server trailers.  Inside of it:
  - [header_mutation](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L273):
    Header mutations.  See [Header rewriting](#header-rewriting) below.
- [immediate_response](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L168):
  May be sent in response to any message from the data plane.  If sent
  in response to a server trailers event, sets the status and optionally
  headers to be included in the trailers.  If sent in response to any
  other event, then the behavior differs depending on whether the
  ext_proc filter is running on the gRPC client or the gRPC server.  On
  the gRPC client side, it will cause the data plane RPC to immediately
  fail with the specified status as if it were an out-of-band
  cancellation.  On the gRPC server side, it will cause the server to
  immediately send trailers with the specified status.  The filter may
  cancel the ext_proc stream after seeing this.  Inside this message:
  - [grpc_status](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L353):
    The status code to send on the data plane RPC.
  - [details](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L358):
    The status message to send on the data plane RPC.
  - [headers](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L346):
    Headers to modify.  Use of these header modifications is best effort,
    depending on which event it was sent in response to and whether the
    filter is running in the gRPC client or server.
  - We will ignore the status and body fields, since these don't apply
    to gRPC.
- [mode_override](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L189C60-L189C73):
  See [Processing Mode Override](#processing-mode-override) below.
- request_drain (new field being added in
  https://github.com/envoyproxy/envoy/pull/38753): If true, the filter
  will send a half-close on the ext_proc stream.  It will then continue
  sending message bodies received from the ext_proc server until the
  ext_proc stream terminates with OK status.  After that, any subsequent
  message on the stream will be passed through as-is.
- We will ignore override_message_timeout, since GRPC body send mode
  does not support timeouts.
- We ignore the dynamic_metadata field, since it is not relevant to gRPC.

Note that the responses from the ext_proc server must come back in the
same order that the events were sent by the filter.  For example, if the
client sends a client headers event and a client message event and the
ext_proc server responds to the client message event first, that is
considered a protocol error.  The filter will treat that as if the
ext_proc stream failed with a non-OK status.

#### Header Rewriting

When responding to a client headers, server headers, or server trailers
event, the ext_proc server can return a
[`HeaderMutation`](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L369)
message that says how to mutate the headers.  That message will be
handled as follows:
- [set_headers](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L375):
  Headers to set or mutate.  Inside this message:
  - [header](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/core/v3/base.proto#L458):
    Required.  Within it:
    - [key](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/core/v3/base.proto#L404):
      The header name.  Must be non-empty and all lower-case.  Length
      must not exceed 16384.  The entry will be ignored if the key is
      `host` or starts with a `:`.
    - [raw_value](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/core/v3/base.proto#L422):
      The header value.  Length must not exceed 16384.
    - The value field will be ignored.
  - [append_action](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/core/v3/base.proto#L476):
    We honor the 4 enum values as described in the proto file.
  - [keep_empty_value](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/config/core/v3/base.proto#L480):
    By default, any header mutation that results in a header with an
    empty value will cause the header key to be removed.  If this field
    is set to true, then such empty headers will be kept.
  - We do not support the deprecated append field.
- [remove_headers](https://github.com/envoyproxy/envoy/blob/cdd19052348f7f6d85910605d957ba4fe0538aec/api/envoy/service/ext_proc/v3/external_processor.proto#L379):
  Header names to remove.  The filter will ignore `host` and any entry
  that starts with `:`.

#### Processing Mode Override

The `mode_override` field in the ext_proc response allows the ext_proc
server to tell the client to change its behavior for the duration of
the RPC.  For example, the ext_proc server may decide after seeing the
client headers event that it does not actually need to see the client
message events.

If the `allow_mode_override` config field is set to false, then the
`mode_override` field will be ignored.

If the `allowed_override_modes` config field is non-empty and
the value of the `mode_override` field does not match any entry in
`allowed_override_modes`, the mode override will be ignored.  Note that
this matching will ignore the value of the `request_header_mode` field,
since that mode cannot be overridden.  (The filter will already have
sent the request headers to the ext_proc server before the ext_proc
server can send back a response changing the processing mode.)

Otherwise, when the filter receives a response from the ext_proc server
with `mode_override` set, it will comply with the requested change for
all subsequent events on the data plane RPC.  However, note that due to
the streaming nature of the ext_proc communication, there is no
guarantee that the filter will see the message before it has already
processed events from the data plane RPC that would have been affected
by the change.

A common use-case for this is when the filter is configured to send
client messages, and in the response to the client headers event, the
ext_proc server uses `mode_override` to set `request_body_mode` to
`NONE`.  The filter may have already sent one or more client message
events by the time it sees this override.  In this case, the ext_proc
server will ignore those client message events, and the filter should (a)
stop sending any subsequent client message events and (b) stop waiting
for responses to any client message events that it has already sent and
allow those client messages to proceed on the data plane RPC.

Note that when using `mode_override` to *enable* sending an event that the
filter was configured to *disable*, the event may have already occurred
by the time the filter sees the `mode_override`.  For example, if the
filter was originally configured to not send server headers and the
ext_proc server tries to enable that in a response to a client message
event, it's possible that the filter has already allowed server headers
to proceed on the data plane RPC before it saw that `mode_override`.
Therefore, ext_proc servers should be aware that this usage is
best-effort and not guaranteed.

To implement this, the filter will store the processing mode separately
for each data plane RPC.  The processing mode for an RPC will be
initialized based on the filter's config, but it may be modified later
by subsequent overrides.

### Temporary environment variable protection

TODO: do we need separate env vars for client and server sides?

Support for the ext_proc filter will be guarded by the
`GRPC_EXPERIMENTAL_XDS_EXT_PROC` environment variable. This guard will
be removed once the feature passes interop tests.

## Rationale

For payload handling, we could have considered having the gRPC ext_proc
filter artificially add the 5-byte gRPC frame header to each message
that we send to the ext_proc server.  However, this seems like a
sub-optimal approach, for the following reasons:
- It would still not work for streaming cases, where the ext_proc server
  needs to be able to modify individual messages as they are sent,
  rather than only modifying the entire stream as a single payload.
- It would waste bandwidth sending data that really isn't useful.
- It would not avoid the need for the ext_proc server to handle the
  gRPC framing.
- It would further spread use of the gRPC framing to ext_proc servers,
  which will make it awkward for gRPC to support other transports with
  different framing in the future.

We considered supporting non-streaming body send modes, but that would
have significantly increased latency for streaming RPCs, because we would
have needed to wait a full RTT with the ext_proc server between each
message on the stream.  We also considered a more pipelined approach
where the data plane would have queued the messages and the ext_proc
server would have been required to send back a response for each message
indicating an optional replacement, but that approach would have (a)
not allowed the ext_proc server to modify the number of messages on
the stream and (b) would have required implementing some form of flow
control to impose push-back upon hitting some maximum buffer size.

## Implementation

Will be implemented in C-core, Java, Go, and Node.
