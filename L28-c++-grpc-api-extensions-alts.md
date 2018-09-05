Title
----
* Author(s): yihuazhang
* Approver: vjpai, markdroth
* Status: Draft
* Implemented in: https://github.com/grpc/grpc/pull/14727
* Last updated: March 29, 2018
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/CI0xJgcPdIs

## Abstract
Since ALTS (Application Layer Transport Security) C client library has
been incorporated into gRPC core, it is necessary to make its APIs for
creating client (channel) and server credentials to be a part of gRPC public
APIs so that users can invoke those APIs to create ALTS credentials.

## Background
ALTS is a mutual authentication and transport security system that is
developed by Google and typically used to secure Remote Procedure Call (RPC).
By incorporating its client library into gRPC, it allows gRPC users that have
workloads running on Google Cloud Platform (GCP) to secure their communication
either to another workload running on GCP or to public Google cloud services.

ALTS whitepaper: https://cloud.google.com/security/encryption-in-transit/application-layer-transport-security/

Notice that ALTS is not ready for public consumption yet and
is still an experimental feature.

### Related Proposals:
N/A

## Proposal

The first part of proposal adds C++ APIs of creating ALTS channel credentials to
include/grpcpp/security/credentials.h.

### ALTS channel credentials create C++ API
```c++
/// ALTS credentials client options used to build AltsCredentials.
/// More options can be added later if necessary.
struct AltsCredentialsOptions {
  /// service accounts of target endpoint that will be acceptable
  /// by the client. If service accounts are provided and none of them matches
  /// that of the server, authentication will fail.
  std::vector<grpc::string> target_service_accounts;
};

/// Builds ALTS Credentials given ALTS specific options
std::shared_ptr<ChannelCredentials> AltsCredentials(
    const AltsCredentialsOptions& options);
```

The second part of proposal adds C++ APIs of creating ALTS server credentials
to include/grpcpp/security/server_credentials.h.
```c++
/// ALTS credentials server options used to build AltsServerCredentials.
struct AltsServerCredentialsOptions {
  /// Add fields if needed.
};

/// Builds ALTS ServerCredentials given ALTS specific options
std::shared_ptr<ServerCredentials> AltsServerCredentials(
    const AltsServerCredentialsOptions& options);
```

## Rationale

Regarding ALTS credentials server options, since it currently does not contain any
information, we may exclude its related APIs from the headers. However for the
purpose of API extensibility, I think it is still worthwhile to include it so that
the server may specify its own credentials options in future use cases.

## Implementation
The implementation of this proposal will be straightforward as we already have C
implementation of ALTS client and server credentials. All we need to do is to
invoke corresponding C APIs.

