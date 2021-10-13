G2: gRPC over HTTP/3
----
* Author(s): [James Newton-King](https://github.com/jamesnk)
* Approver: ejona86
* Status: In review
* Implemented in: grpc-dotnet
* Last updated: 2021-08-25
* Discussion at: https://groups.google.com/g/grpc-io/c/b1dWReBGyX4

## Abstract

HTTP/3 is the third and upcoming major version of the Hypertext Transfer Protocol
used to exchange information on the World Wide Web, alongside HTTP/1.1 and HTTP/2.

This proposal is for how the gRPC protocol should run on top of the HTTP/3 protocol.

## Background

gRPC uses HTTP semantics but requires HTTP/2 for some features. HTTP/3 offers the
same capabilities as HTTP/2, enabling all gRPC scenarios, along with new benefits
offered by HTTP/3:

* Faster connection negotiation in fewer round-trips.
* Improved experience when there is connection packet loss.
* Client supports transitioning between networks.

[PROTOCOL-HTTP2.md](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md)
covers gRPC's HTTP semantics, along with some content that is specific to HTTP/2.
Either this document should be generalized to include HTTP/2 and HTTP/3 information,
or a new gRPC over HTTP/3 protocol document should be created.

### Related Proposals:

n/a

## Proposal

### HTTP semantics

The shape of gRPC HTTP requests and responses are unchanged in HTTP/3. The
content in [PROTOCOL-HTTP2.md](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md)
that covers requests and responses can be referred to directly, without any
duplication.

Notably, unlike gRPC-Web, the content-type of `application/grpc` is unchanged. Apps
are still communicating with gRPC, but over HTTP/3 instead of HTTP/2.

### Transport mapping

The gRPC over HTTP/2 specification [discusses HTTP2 transport mapping](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md#http2-transport-mapping).
The content discussed is mostly applicable to HTTP/3.

#### Stream Identification

HTTP/3 stream IDs function largely the same as HTTP/2 stream IDs.

#### Data frames

The relationship between `DATA` frames and length prefixed messages are unchanged in HTTP/3.

#### Errors

HTTP/3 [has different error codes](https://quicwg.org/base-drafts/draft-ietf-quic-http.html#section-8.1)
from HTTP/2. HTTP/3 errors are sent via QUIC frames instead of an `RST_STREAM` HTTP frame.

HTTP/3 errors are used in three situations:

* Abruptly terminating streams
* Aborting reading of streams
* Immediately closing HTTP/3 connections

HTTP3 Code|HTTP2 Code|GRPC Code
----------|----------|-----------
H3_NO_ERROR(0x0100)|NO_ERROR(0)|INTERNAL - An explicit GRPC status of OK should have been sent but this might be used to aggressively [lameduck](https://landing.google.com/sre/sre-book/chapters/load-balancing-datacenter/#identifying-bad-tasks-flow-control-and-lame-ducks-bEs0uy) in some scenarios.
H3_GENERAL_PROTOCOL_ERROR(0x0101)|PROTOCOL_ERROR(1)|INTERNAL
H3_INTERNAL_ERROR(0x0102)|INTERNAL_ERROR(2)|INTERNAL
H3_STREAM_CREATION_ERROR(0x0103)|n/a|INTERNAL
H3_CLOSED_CRITICAL_STREAM(0x0104)|n/a|INTERNAL
H3_FRAME_UNEXPECTED(0x0105)|FRAME_SIZE_ERROR|INTERNAL
H3_FRAME_ERROR(0x0106)|FRAME_SIZE_ERROR|INTERNAL
H3_EXCESSIVE_LOAD(0x0107)|ENHANCE_YOUR_CALM|RESOURCE_EXHAUSTED ...with additional error detail provided by runtime to indicate that the exhausted resource is bandwidth.
H3_ID_ERROR(0x0108)|n/a|INTERNAL
H3_SETTINGS_ERROR(0x0109)|SETTINGS_TIMEOUT(4)|INTERNAL
H3_MISSING_SETTINGS(0x010a)|SETTINGS_TIMEOUT(4)|INTERNAL
H3_REQUEST_REJECTED(0x010b)|REFUSED_STREAM|UNAVAILABLE - Indicates that no processing occurred and the request can be retried, possibly elsewhere.
H3_REQUEST_CANCELLED(0x010c)|CANCEL(8)|Mapped to call cancellation when sent by a client.Mapped to CANCELLED when sent by a server. Note that servers should only use this mechanism when they need to cancel a call but the payload byte sequence is incomplete.
H3_REQUEST_INCOMPLETE(0x010d)|n/a|INTERNAL
H3_MESSAGE_ERROR(0x010e)|n/a|INTERNAL
H3_CONNECT_ERROR(0x010f)|CONNECT_ERROR|INTERNAL
H3_VERSION_FALLBACK(0x0110)|n/a|INTERNAL
n/a|FLOW_CONTROL_ERROR(3)|INTERNAL
n/a|STREAM_CLOSED|No mapping as there is no open stream to propagate to. Implementations should log.
n/a|COMPRESSION_ERROR|INTERNAL
n/a|INADEQUATE_SECURITY| PERMISSION_DENIED … with additional detail indicating that permission was denied as protocol is not secure enough for call.

#### Connection management

`GOAWAY` and `PING` frames exist in HTTP/3 and serve the same purpose as in HTTP/2.
One notable difference is the `GOAWAY` frame in HTTP/2 reports the last
successfully processed stream ID. In HTTP/3 the `GOAWAY` frame ID value must be greater
that the last successfully processed stream ID.

### Exceeding deadlines

When an RPC has exceeded its deadline, the server will reset the stream. In HTTP/2, a stream
is reset using the `RST_STREAM` frame. The `RST_STREAM` frame doesn't exist in HTTP/3. 
Instead, this action is performed using a QUIC frame, called `RESET_STREAM`.

|                           | Frame        | Error code                   | Layer    
----------------------------|--------------|------------------------------|----------
Deadline exceeded HTTP/2    | RST_STREAM   | CANCEL(8)                    | HTTP     
Deadline exceeded HTTP/3    | RESET_STREAM | H3_REQUEST_CANCELLED(0x010c) | QUIC     

[RESET_STREAM](https://www.rfc-editor.org/rfc/rfc9000.html#name-reset_stream-frames) abruptly
terminates sending on a stream. An important difference between HTTP/2 and
HTTP/3 is frames are received out-of-order. Because of this, a `RESET_STREAM` sent after a
completed response could still result in the response being aborted.

HTTP/3 also has the QUIC `STOP_SENDING` frame. This frame is sent by the server when a
stream's response side completes before the request side. Using `STOP_SENDING` alone isn't
appropriate for a deadline exceeded, because both stream dictions should be aborted.
However, `STOP_SENDING` should be sent along with `RESET_STREAM` if the deadline is exceeded
while the request side is in-progress.

## Rationale

HTTP/3 is an upcoming Internet Protocol. There needs to be a standardized agreement for how
gRPC over HTTP/3 works to maintain interoperability in the gRPC eco-system.

## Implementation

A trial implementation in grpc-dotnet is underway. .NET 6 is adding preview support for
HTTP/3 to the .NET server and client. grpc-dotnet leverages that underlying HTTP/3 support.

## Open issues (if applicable)

n/a
