Allow Call Credentials to be Specified in `grpc_google_default_credentials_create`
----
* Author(s): rbellevi
* Approver: markdroth
* Status: Draft
* Implemented in: Core
* Last updated: July 6th, 2020
* Discussion at: (TODO)

## Abstract

This document proposes that `grpc_google_default_credentials_create` be
amended to allow the user to specify their desired call credential, while
also adding a `void* reserved` argument to lessen the burden of any such
changes in the future.


## Background

The so-called "Google default credentials" created by the
`grpc_google_default_credentials_create` function in Core enables connection to
GCP via a combination of ALTS and SSL credentials, along with a special oauth2
token that must assert the same identity as the channel-level ALTS credential.

This oauth2 token attached by `grpc_google_default_credentials_create` is based
on the default VM service account, but the auth libraries for wrapped languages
allow the creation of channels with non-default service accounts. In this case,
a mismatch in identity occurs that may cause an unexpected persistent failure of
RPCs at runtime.

While the default service account is the correct option is most cases, there
should be an option that enables auth libraries to supply their own call
credentials.

## Proposal

I propose that the signature of `grpc_google_default_credentials_create` be
amended to the following:

```C
GRPCAPI grpc_channel_credentials*
grpc_google_default_credentials_create(grpc_call_credentials* call_credentials,
                                       void* reserved);
```

Supplying `nullptr` for `call_credentials` will result in the current behavior
of the function -- oauth2 call credentials based on the default service account
of the VM will be used.

## Rationale

A first attempt at this problem was the addition of a new API very similar to
`grpc_google_default_credentials_create`, but it was determined that too much
was duplicated by this implementation.

## Implementation

The implementation of this proposal will be carried out in [this PR.](https://github.com/grpc/grpc/pull/23203)

## Open issues (if applicable)

N/A
