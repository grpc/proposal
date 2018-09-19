True Binary Metadata
----
* Author(s): ncteisen
* Approver: a11r
* Status: Draft
* Implemented in: POC in https://github.com/grpc/grpc/pull/16618
* Last updated: 9/19/18
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/Ww8waYz_Nes

## Abstract

Add a HTTP2 extension that signals that an implementation may breach the HTTP/2
spec for the purposes of performance.

## Background

gRPC is built on top of HTTP/2, which means it must remain spec compliant with 
[RFC 7540](https://httpwg.org/specs/rfc7540.html). That spec lays out an 
"elastic clause" in section 5.5,
[Extending HTTP/2](https://httpwg.org/specs/rfc7540.html), which allows
extensible protocol modifications as long as both peers agree to the custom
extensions.

Many aspects of the HTTP/2 spec are built upon fundamental requirements for
serving traffic over a public internet, as well as certain assumptions about
the network conditions. Some of these requirements and assumptions can become
inverted when trying to build a high performance RPC system for intra-cluster
traffic in which the client and sever are completely trusted.

### Related Proposals:
* This relates to [G1: True Binary Metadata](G1-true-binary-metadata.md)

## Proposal

### HTTP/2 Setting

We already expose custom setting in our HTTP2 settings exchange:
GRPC_ALLOW_TRUE_BINARY_METADATA = 0xfe03. This setting's id will remain the
same, but its name will be changed to GRPC_ALLOW_HFAST.

The value of the setting is an integer, which defaults to 0, signaling that
HFAST is disallowed.

If the setting is non-zero, then peers MAY use a 'true binary' encoding as 
described in [G1](G1-true-binary-metadata.md).

If the setting is higher than 1, it signals that an implementation can use
HFAST as long as client and server send the same number. If there is a mismatch,
then implementations must fall back to HFAST disabled. This gives HFAST a sense
of logical versioning.

This means that if a client/server pair was talking HFAST, but one of the two is
update before the other, they will stop using HFAST altogether (until the
lagging party also updates). This is agreeable, since the main use case for
HFAST are projects that control both client and server completely, and are able
to "live at HEAD".

Implementations SHOULD transmit this setting only once, and as part of the first
settings frame.

### HFAST 2: Assumed Header Values

If the value of the GRPC_ALLOW_HFAST setting is 2, then implementations may
assume the values of certain HTTP/2 headers ONLY IF they are not present in the
incoming header block.


| Key | Value |
|---|---|
| :method | POST |
| :status | 200 |
| te | trailers | 
| content-type | application/grpc |


