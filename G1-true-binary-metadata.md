Title
----
* Author(s): ctiller
* Approver: ejona
* Status: Draft
* Implemented in: n/a
* Last updated: 3/29/17
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

Add a HTTP2 extension to allow binary metadata (those suffixed with -bin) to be
sent without the base64 encode/decode step.

## Background

gRPC allows binary metadata to be sent by applications. These metadata elements
are keyed with a -bin suffix. When transmitted on the wire, since HTTP2 does not
allow binary headers, we base64 encode these elements and Huffman compress them.
On receipt, we transparently reverse the transformation.

This transformation is costly in terms of CPU, and there exist use cases where
this transformation can become the CPU bottleneck for gRPC.

### Related Proposals:

n/a

## Proposal

### New setting

Expose a custom setting in our HTTP2 settings exchange:
GRPC_ALLOW_TRUE_BINARY_METADATA = 0xfe03.

This setting is randomly chosen (to avoid conflicts with other extensions), and
within the experimental range of HTTP extensions.

The setting can have the values 0 (default) or 1. If the setting is 1, then
peers MAY use a 'true binary' encoding (described below), instead of the current
base64 encoding for -bin metadata.

Implementations SHOULD transmit this setting only once, and as part of the first
settings frame.

### 'True binary' encoding

When transmitting metadata on a connection where the peer has specified
GRPC_ALLOW_TRUE_BINARY_METADATA, instead of encoding using base64, an
implementation MAY instead prefix a NUL byte to the metadata and transmit the
data in binary form.

Since this is a HTTP2 extension and other extensions might alias this extension
id, it's possible that this becomes misconfigured. In that case, peers are
required to RST_STREAM with http error PROTOCOL_ERROR. If a binary encoding was
attempted and such a RST_STREAM is received without any other headers,
implementations SHOULD retry the request with base64 encoding, and disable
binary encoding for future requests. Verbosely logging this condition is
encouraged.

### Examples

Suppose we wanted to send metadata element 'foo-bin: 0x01' (ie a single byte
containing '1').

Under base64, we'd send a http header 'foo-bin: AQ'
Under binary, we'd send 'foo-bin: 0x00 0x01' (ie prefixing a NUL byte and then
sending the binary metadata value)

## Rationale

Binary metadata transmission performance is critical for a number of
applications.

Various workarounds were considered:
1. Switching to base16 - this would require a backwards incompatible protocol
   change for gRPC, and has the disadvantage of bloating wire size, which
   additionally interacts badly with hpack.
2. Adding a new suffix (-raw): this leaks further implementation details to
   application developers, and likely would still need a base64 workaround

## Implementation

A trial implementation in gRPC C core is underway and expected to be ready in
coming weeks.

## Open issues (if applicable)

n/a
