# Support ALTS Hard Bound Call Credentials in Google Default Credentials

--------------------------------------------------------------------------------

*   Author(s): anniefrchz
*   Approver: a11r
*   Status: Draft
*   Implemented in: C++
*   Last updated: 2025/06/30
*   Discussion at: https://groups.google.com/g/grpc-io/c/7rRWghiS95E/m/nENwU3BtCgAJ

## Abstract

This proposal outlines a change to the gRPC Core C-API to support
alts-credentials configurations within `grpc_google_default_credentials`. This
enhancement specifically allows a secondary set of call credentials for ALTS to
be provided alongside the default credentials.

### Background

The existing `grpc_google_default_credentials_create` function allows the
configuration of a single set of call credentials. In some scenarios, a client
would want to communicate with services that support hard bound credentials over
ALTS. This proposal addresses the possibility to support this use case on a
channel initialized with Google's default credentials.

### Proposal

To maintain backward compatibility with the existing C-API, this proposal
modifies the function `grpc_google_default_credentials_create` to add a second
set of call credentials.

```c
GRPCAPI grpc_channel_credentials* grpc_google_default_credentials_create(
    grpc_call_credentials* tls_credentials,
    grpc_call_credentials* alts_credentials);
```

This new function accepts two arguments:

1.  `tls_credentials`: The primary call credentials, consistent with the
    existing API. Usually. default back to the TLS connection.
2.  `alts_credentials`: A secondary set of call credentials to be used
    specifically for ALTS connections.

After a secure connection is established, the gRPC runtime identifies the
channel's transport security type, which indicates whether the underlying
channel is using a protocol like ALTS or TLS. The runtime then selects the
appropriate call credentials for that connection. If the determined transport
security type is ALTS, the provided alts_credentials will be used. For all other
transport types, the primary call_credentials are used, maintaining the default
behavior.

This approach was decided upon after initial feedback suggested modifying the
existing API.

## Rationale

The primary motivation for this change is to enable seamless support for hybrid
security environments on a single gRPC channel.

The advantages of this approach are:

*   Consolidated API: It avoids introducing a new function for a closely related
    feature, keeping the API surface clean and concise. An initial review of the
    pull request favored this path to avoid an unnecessary new API.
*   Improved Discoverability: Developers only need to be aware of a single
    function for creating Google default credentials. The optional nature of the
    second parameter would make the basic use case simple while allowing for the
    more advanced dual-credential scenario when needed.
*   Logical Cohesion: Since the new functionality is an extension of the
    existing credential creation process, incorporating it into the original
    function maintains logical cohesion. The function's responsibility is
    expanded rather than duplicated across multiple functions.
*   Cross-language support: Since wrapper languages like Python use this API,
    supporting ALTS bound credentials will be straightforward by passing them as
    an argument to the existing API.

## Implementation

The implementation for this proposal has been completed and merged into the main
gRPC repository.

*   Pull Request: grpc/grpc#39770
*   Key Commit: The changes were integrated via commit ca2e8c9.
