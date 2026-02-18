Rename verify_peer_options struct to grpc_ssl_verify_peer_options in grpc_security.h
----
* Author(s): yihuaz
* Approver: vjpai, markdroth
* Status:
* Implemented in:
* Last updated: May 14, 2019
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/52xBb0vMbKQ

## Abstract

Rename `verify_peer_options` struct to `grpc_ssl_verify_peer_options` in
grpc_security.h to follow the gRPC C core naming convention.

## Background

gRPC C core surface API's declared in grpc_security.h should follow the C core naming
convention that is starting with the `grpc_` prefix. However `verify_peer_options` struct
[introduced](https://github.com/grpc/grpc/pull/15274) for the purpose of server
authorization check does not conform to this convention and thus needs to be fixed. Here,
the server authorization check is used to validate if a peer is authorized to run
a job with a specific target name and is invoked only at the client-side.

## Proposal

* Rename `verify_peer_options` struct to `grpc_ssl_verify_peer_options`.
* Change the type of 3rd argument (i.e., `verify_options`) in `grpc_ssl_credentials_create()`to `grpc_ssl_verify_peer_options`.

## Rationale

It is desirable to have all gRPC C core surface API's to follow a consistent C
core naming convention.

## Implementation

## Open issues (if applicable)

N/A
