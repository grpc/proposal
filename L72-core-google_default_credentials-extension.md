Allow Call Credentials to be Specified in `grpc_google_default_credentials_create`
----
* Author(s): rbellevi
* Approver: markdroth
* Status: Draft
* Implemented in: Core
* Last updated: July 8th, 2020
* Discussion at: https://groups.google.com/g/grpc-io/c/fZNm4pU8e3s/m/2Be8u1n7BQAJ

## Abstract

This document proposes that `grpc_google_default_credentials_create` be
amended to allow the user to specify their desired call credentials.

## Background

The Google default credentials created by the
`grpc_google_default_credentials_create` function in Core enable connection to
Google services via a combination of ALTS and SSL credentials, along with a special oauth2
token that must assert the same identity as the channel-level ALTS credential.

In C++, auth is handled by the gRPC library itself. In wrapped
languages such as Python, however, auth is handled by external libraries which
incur a dependency on gRPC, such as [`google-auth-library-python`](https://github.com/googleapis/google-auth-library-python).
These libraries have their own implementation of the
[Application Default Credentials](https://cloud.google.com/docs/authentication/production?_ga=2.68587985.1354052904.1594166352-2074181900.1593114348#finding_credentials_automatically)
mechanism. Thus, if an auth library were to use the current version of
`grpc_google_default_credentials_create`, the Application Default Credentials
logic would be duplicated between the auth library and gRPC Core.

## Proposal

I propose that the signature of `grpc_google_default_credentials_create` be
amended to the following:

```C
GRPCAPI grpc_channel_credentials*
grpc_google_default_credentials_create(grpc_call_credentials* call_credentials);
```

Supplying `nullptr` for `call_credentials` will result in the current behavior
of the function. That is, Core will attach a compute engine call credential
based on the Application Default Credentials mechanism.

## Rationale

A first attempt at this problem was the addition of a new API very similar to
`grpc_google_default_credentials_create`, but it was determined that too much
was duplicated by this implementation.

It is possible that the call credentials provided by the caller are not compute
engine credentials or do not assert the identity of the default service account
of the VM. Ideally, a programmatic check would verify that no such credentials
are passed in. Unfortunately, the type of credentials passed in are opaque to
both Core and the gRPC wrapped language library, making such a check impossible.
A prominent warning will be added to the documentation for the function to warn
users of such pitfalls.

## Implementation

The implementation of this proposal will be carried out in [this PR.](https://github.com/grpc/grpc/pull/23203)

## Open issues (if applicable)

N/A
