Add Optional GSSAPI call credentials implementation
---------------------------------------------------

* Author: David Li (@lihalite)
* Approver: a11r
* Status: Draft
* Implemented in: C++
* Last updated: 2018/09/19
* Discussion at: N/A

## Abstract

gRPC client libraries will include an optional call credentials that
uses GSSAPI to acquire a token, added to the metadata on each call.

## Background

gRPC client libraries provide an authentication API based on a
Credential interface, as well as a set of Credential implementations
for common authentication methods including TLS and OAuth2. Another
authentication method, currently unimplemented, is the Kerberos
protocol, generally used through the GSSAPI standard, and often found
in corporate environments.

### Related Proposals:

N/A

## Proposal

gRPC will optionally support GSSAPI authentication via a provided
Credential implementation. To avoid introducing a new hard dependency
for everyone, GSSAPI support will only be compiled/provided when
configured at build time. Since multiple GSSAPI implementations exist,
the user can specify the implementation at build time, by providing
the library to link to and any necessary include paths (for C++).

If not enabled at build time, then the implementation will be a stub
that simply logs a message at the INFO warning level.

In grpc_cli, if enabled at build time, we will create a composite
channel credentials that includes the GSSAPI call credentials, so that
any requests made are authenticated.

## Rationale

By making the dependency on GSSAPI optional, we avoid introducing new
hard dependencies for downstream users, but we also make the build
process more complex. However, by placing it in the C++ core, we can
integrate it with grpc_cli, allowing it to be used to debug and test
gRPC servers in environments requiring GSSAPI authentication.

Alternate approaches:

The Credential could be implemented in an external library, instead of
including it in gRPC core. However, we would like to use grpc_cli,
which is part of gRPC core, with GSSAPI authentication.

## Implementation

1. C++.
   1. Extend the build system to allow an optional dependency on a
      GSSAPI implementation. This must be done for Bazel, Make, and
      CMake.
   2. If enabled, build the `grpc_call_credentials` implementation
      that initializes a GSSAPI security context, retrieves a token,
      base64 encodes it, and sets the encoded token as the value of
      the “negotiate” metadata field.
      1. In C/C++, GSSAPI implementations all share a common API,
         gssapi.h (except for GNU GSS), so we do not need to
         explicitly support each individual library.
      2. In grpc_cli, create a combined credentials object so that
         requests are authenticated.
   3. If not enabled, build a stub that simply logs a warning.
      1. Do nothing in grpc_cli.
2. Other languages can be contributed as appropriate.

## Open issues (if applicable)

We may contribute implementations for Java and Python as well, but are
unlikely to do so for other languages gRPC supports. For these
languages, the user would likely provide a GSSAPI implementation at
runtime, not compile-time.
