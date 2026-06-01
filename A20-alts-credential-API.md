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

The first part of proposal adds C core APIs of creating ALTS channel/server credentials to include/grpc/grpc_security.h. This addition is necessary to allow C++ and wrapper languages (e.g., python) to make use of ALTS C client libraries.

### ALTS credentials options interface
```c++
/**
 * Main interface for ALTS credentials options. The options contain
 * information that are specified by gRPC users and passed to ALTS credentials
 * to be used during ALTS mutual authentication. ALTS channel and server credentials
 * will have their own implementation of this interface.
 */
typedef struct grpc_alts_credentials_options grpc_alts_credentials_options;
```
 
### ALTS credentials client options create C core API
```c++
GRPCAPI grpc_alts_credentials_options* grpc_alts_credentials_client_options_create();
```
 
### ALTS credentials server options create C core API
```c++
GRPCAPI grpc_alts_credentials_options* grpc_alts_credentials_server_options_create();
```
 
### ALTS credentials options API that adds target service accounts to client options
```c++
/**
 * Client may specify acceptable server identities when creating ALTS channel
 * credentials. This information will be added to ALTS credentials client
 * options and passed to ALTS TSI handshakers. If server's target service accounts
 * are provided but none of them matches the peer identity of the server,
 * the handshake will fail.
 *
 * - options: grpc ALTS credentials options instance.
 * - service_account: service account of target endpoint.
 */
GRPCAPI void grpc_alts_credentials_client_options_add_target_service_account(
    grpc_alts_credentials_options* options, const char* service_account);
```
 
### ALTS credentials options destroy C core API
```c++
GRPCAPI void grpc_alts_credentials_options_destroy(
    grpc_alts_credentials_options* options);
```
 
### ALTS channel credentials create C core API
```c++
/**
 * This method creates an ALTS channel credential object.
 *
 * - options: grpc ALTS credentials options instance for client.
 *
 * It returns the created ALTS channel credential object.
 */
GRPCAPI grpc_channel_credentials* grpc_alts_credentials_create(
    const grpc_alts_credentials_options* options);
```
 
### ALTS server credentials create C core API
```c++
/**
 * This method creates an ALTS server credential object.
 *
 * - options: grpc ALTS credentials options instance for server.
 *
 * It returns the created ALTS server credential object.
 */
GRPCAPI grpc_server_credentials* grpc_alts_server_credentials_create(
    const grpc_alts_credentials_options* options);
```

The second part of proposal adds C++ APIs of creating ALTS channel and server credentials to include/grpcpp/security/credentials.h and include/grpcpp/security/server_credentials.h, respectively. 

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

### ALTS server credentials create C++ API
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
implementation of ALTS client and server credentials. All we need to do is
moving API declarations to a different header file (first part) and invoking the corresponding C APIs.

