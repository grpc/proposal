# Title
-------
* Author(s): Vignesh Babu (Vignesh2208)
* Approver: ctiller
* Status: In Review
* Implemented in: core, cpp
* Last updated: 09/28/2021
* Discussion at: https://github.com/grpc/grpc/issues/24944

# Abstract

This proposal discusses additions to the core/cpp public headers to support exporing of TLS session keys on the client and server side to aid in decryption of packet captures using tools such as Wireshark.

# Background

Currently unlike the GRPC Go implementation, GRPC C++ does not allow export of TLS session keys at run time. This is needed to decrypt and analyse packet captures for debugging purposes. The session keys when exported to files in the NSS format, and can be read by debugging tools like wireshark which can automatically decrypt packet captures.

# Related Proposals:

N/A

# Proposal

## C++API Additions

The proposal extends the existing `grpc::experimental::TlsCredentialsOptions` class with methods to set configuration for supporting TLS session key export at run-time. The proposed changes include:


* A `set_tls_session_key_log_file_path(std::string path)` method is added to the existing `grpc::experimental::TlsCredentialsOptions` class to set the desired TLS session key export file path on the client/server side.

These changes allow specifying TLS session key export configuration on a per-channel level on the client side and on a per-port level on the server side. If the path is not explicitly set, TLS session key logging is disabled.


## C-Core API Additions

The following additions are made to the gRPC C-Core API to enable TLS session key logging. The API additions are described below with inline comments.

```
/** --- TLS session key logging. ---
 * Experimental API to control tls session key logging. Tls session key logging
 * is expected to be used only for debugging purposes and never in production.
 * Tls session key logging is only enabled when:
 *  At least one grpc_tls_credentials_options object is assigned a tls session
 *  key logging file path using the API specified below.
 */


/**
 * EXPERIMENTAL API - Subject to change.
 * Associates a session key logging config with a
 * grpc_tls_credentials_options object.
 * - options is the grpc_tls_credentials_options object
 * - path is a string pointing to the location where TLS session keys would be
 *   stored.
 */
GRPCAPI void grpc_tls_credentials_options_set_tls_session_key_log_file_path(
    grpc_tls_credentials_options* options, const char* path);
```

# Example Usage


## Client Side (per-channel) C++ API usage

```
grpc::experimental::TlsChannelCredentialsOptions options;
options.set_tls_session_key_log_file_path(absl::StrCat(BASE_DIR, "/ssl-key-log-client.txt"));
auto channel = grpc::CreateChannel(server_address,
  grpc::experimental::TlsCredentials(options)));
```

## Server Side (per-port) C++ API usage

```
grpc::experimental::TlsServerCredentialsOptions options(certificate_provider);
options.set_tls_session_key_log_file_path(absl::StrCat(BASE_DIR, "/ssl-key-log-server.txt"));
...
ServerBuilder builder;
builder.AddListeningPort(server_address,
      grpc::experimental::TlsCredentials(options));
```

## C-core API Usage

```
grpc_tls_credentials_options * options = ...

// associate a tls session key logging file path with a grpc_tls_credentials_options object
grpc_tls_credentials_options_set_tls_session_key_log_file_path(options, "/tmp/test.txt");

```

# Rationale

Alternatives using ENVIRONMENT Variables to set TLS session key export configurations were considered. These allow enabling/disabling session key export without having to rebuild the binary. However the disadvantages here include:

* Safety issues related to setting environment variables and forgetting to unset them.
* Supporting per-channel or per-port key export configuration requires setting unique environment variables tied to the channel-id/port number which may not be known prior to run-time in all cases.

The proposed approach does not have these disadvantages but requires rebuilding the binary.


# Implementation

The full implementation of the TLS session key export feature can be found here: https://github.com/grpc/grpc/pull/26812

# Open issues (if applicable)

N/A
