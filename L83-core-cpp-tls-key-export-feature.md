Title
----
* Author(s): Vignesh Babu (Vignesh2208)
* Approver: ctiller
* Status: In Review
* Implemented in: core, cpp
* Last updated: 07/29/2021
* Discussion at: https://github.com/grpc/grpc/issues/24944

## Abstract

This proposal discusses additions to the core/cpp public headers to support exporing of TLS Keys on the client and server side to aid in decryption of packet captures using tools such as Wireshark.

## Background

Currently unlike the GRPC Go implementation, GRPC C++ does not allow export of TLS keys at run time. This is needed to decrypt and analyse packet captures for debugging purposes. The keys when exported in files in a specific format, can be read by debugging tools like wireshark and they can automatically decrypt packet captures.

### Related Proposals:

N/A

## Proposal

The proposal extends the existing `grpc::experimental::TlsCredentialsOptions` class with methods to set configuration for supporting Tls Key export at run-time. The proposed changes
include:

* A `grpc_tls_key_log_format` enum is added to `include/grpc/grpc_security_constants.h`
  ```
  /** Supported TLS Keylogging formats **/
  typedef enum {
    // Default key logging formal is NSS compatible.
    TLS_KEY_LOG_FORMAT_NSS = 0,
    // More formats can be added here for future extensions if desired.
    TLS_KEY_LOG_FORMAT_NONE
  } grpc_tls_key_log_format;
  ```

* A `grpc::experimental::TlsKeyLoggerConfig` struct is defined in `include/grpcpp/security/tls_credentials_options.h` to support configuration for Key export. It currently has two members:
  * `std::string tls_key_log_file_path`: Full path where the TLS keys should be exported to.
  * `grpc_tls_key_log_formattls_key_log_format`: Format of the exported keys.

* Adding a `set_tls_key_log_config(grpc::experimental::TlsKeyLoggerConfig cfg)` method to the `grpc::experimental::TlsCredentialsOptions` class to set the desired configuration on the client/server side.

These changes allow specifying TLS Key export configuration on a per-channel level on the client side and on a per-port level on the server side. If the configuration is not explicitly set, TLS key logging is disabled. This config  struct can be extended in the future to support advanced features such as key logging based on source ip filtering etc.

## Example Usage

# Client Side (per-channel)

```
grpc::experimental::TlsChannelCredentialsOptions options;
grpc::experimental::TlsKeyLoggerConfig config;
config.set_tls_key_log_file_path(
  absl::StrCat(BASE_DIR, "/ssl-key-log-client.txt"));
config.set_tls_key_log_format(TLS_KEY_LOG_FORMAT_NSS);
options.set_tls_key_log_config(config);
auto channel = grpc::CreateChannel(server_address,
  grpc::experimental::TlsCredentials(options)));
```

# Server Side (per-port)

```
grpc::experimental::TlsServerCredentialsOptions options(certificate_provider);
grpc::experimental::TlsKeyLoggerConfig config;
config.set_tls_key_log_file_path(
  absl::StrCat(BASE_DIR, "/ssl-key-log-server.txt"));
config.set_tls_key_log_format(TLS_KEY_LOG_FORMAT_NSS);
options.set_tls_key_log_config(config);
...
ServerBuilder builder;
builder.AddListeningPort(server_address,
      grpc::experimental::TlsCredentials(options));
```


## Rationale

Alternatives using ENVIRONMENT Variables to set TLS Key export configurations were considered. These allow enabling/disabling key export without having to rebuild the binary. However the disadvantages here include:

* Safety issues related to setting environment variables and forgetting to unset them.
* Supporting per-channel or per-port key export configuration requires setting unique enviroment variables tied to the channel-id/port number which may not be known prior to run-time in all cases.

The proposed approach does not have these disadvantages but requires rebuilding the binary.


## Implementation

The full implementation of the TLS Key export feature can be found here: https://github.com/grpc/grpc/pull/26812

## Open issues (if applicable)

N/A
