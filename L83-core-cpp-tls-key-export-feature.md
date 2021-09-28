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

Currently unlike the GRPC Go implementation, GRPC C++ does not allow export of TLS session keys at run time. This is needed to decrypt and analyse packet captures for debugging purposes. The session keys when exported in files in a specific format, can be read by debugging tools like wireshark and they can automatically decrypt packet captures.

# Related Proposals:

N/A

# Proposal

## C++API Additions

The proposal extends the existing `grpc::experimental::TlsCredentialsOptions` class with methods to set configuration for supporting TLS session key export at run-time. The proposed changes include:

* A `grpc_tls_session_key_log_format` enum is added to `include/grpc/grpc_security_constants.h`
  ```
  /** Supported TLS session key logging formats **/
  typedef enum {
    // Default key logging formal is NSS compatible.
    GRPC_TLS_KEY_LOG_FORMAT_NSS = 0,
    // More formats can be added here for future extensions if desired.
  } grpc_tls_session_key_log_format;
  ```

* A `grpc::experimental::TlsSessionKeyLoggerConfig` class is defined in `include/grpcpp/security/tls_credentials_options.h` to support configuration for session key export. It currently has two members:
  * `std::string tls_session_key_log_file_path`: Full path where the TLS keys should be exported to.
  * `grpc_tls_session_key_log_format  tls_session_key_log_format`: Format of the exported keys.

  The `grpc::experimental::TlsSessionKeyLoggerConfig` class has the following signature:
  ```
  class TlsSessionKeyLoggerConfig {
    public:
      // Constructor
      explicit TlsSessionKeyLoggerConfig();

      // Copy ctor
      TlsSessionKeyLoggerConfig(const TlsSessionKeyLoggerConfig& copy)

      // Sets the tls key log file path.
      void set_tls_session_key_log_file_path(std::string session_key_log_file_path);

      // Sets the tls key logging format. Currently only NSS format is supported.
      void set_tls_session_key_log_format(grpc_tls_session_key_log_format tls_session_key_log_format);
    private:
      ...
  };
  ```

* A `set_tls_session_key_log_config(grpc::experimental::TlsSessionKeyLoggerConfig cfg)` method is added to the existing `grpc::experimental::TlsCredentialsOptions` class to set the desired TLS session key export configuration on the client/server side.

These changes allow specifying TLS session key export configuration on a per-channel level on the client side and on a per-port level on the server side. If the configuration is not explicitly set, TLS session key logging is disabled. This config  struct can be extended in the future to support advanced features such as key logging based on source ip filtering etc.


## C-Core API Additions

The following additions are made to the gRPC C-Core API to enable TLS session key logging. The API additions are described below with inline comments.

```
/** --- TLS session key logging. ---
 * Experimental API to control tls session key logging. Tls session key logging
 * is expected to be used only for debugging purposes and never in production.
 * Tls session key logging is only enabled when:
 *  At least one grpc_tls_credentials_options object is assigned a tls session
 *  key logging config using the API specified below.
 */

/**
 * EXPERIMENTAL API - Subject to change.
 * An opaque type pointing to a Tls session key logging configuration.
 */
typedef struct grpc_tls_session_key_log_config grpc_tls_session_key_log_config;

/**
 * EXPERIMENTAL API - Subject to change.
 * Creates and returns a grpc_tls_session_key_log_config. The caller becomes an
 * owner of the returned object and can free the allocated resources
 * appropriately using the grpc_tls_session_key_log_config_release API
 */

GRPCAPI grpc_tls_session_key_log_config*
  grpc_tls_session_key_log_config_create();

/**
 * EXPERIMENTAL API - Subject to change.
 * Releases the ownership of a previously created
 * grpc_tls_session_key_log_config object.
 */
GRPCAPI void grpc_tls_session_key_log_config_release(
    grpc_tls_session_key_log_config* config);

/**
 * EXPERIMENTAL API - Subject to change.
 * Sets the logging format for TLS session key logging.
 */
GRPCAPI void grpc_tls_session_key_log_config_set_log_format(
    grpc_tls_session_key_log_config* config,
    grpc_tls_session_key_log_format format);

/**
 * EXPERIMENTAL API - Subject to change.
 * Sets the file path for TLS session key logging.
 */

GRPCAPI void grpc_tls_session_key_log_config_set_log_path(
    grpc_tls_session_key_log_config* config, const char* path);


/**
 * EXPERIMENTAL API - Subject to change.
 * Associates a session key logging config with a
 * grpc_tls_credentials_options object.
 * - options is the grpc_tls_credentials_options object
 * - config is a grpc_tls_session_key_log_config object containing session key
 *   logging configuration.
 *
 * Each grpc_tls_credentials_options object can be associated with only
 * one grpc_tls_session_key_log_config. An existing config will not be
 * overwritten.
 */
GRPCAPI void grpc_tls_credentials_options_set_tls_session_key_log_config(
    grpc_tls_credentials_options* options,
    grpc_tls_session_key_log_config* config);
```

# Example Usage


## Client Side (per-channel) C++ API usage

```
grpc::experimental::TlsChannelCredentialsOptions options;
grpc::experimental::TlsSessionKeyLoggerConfig config;
config.set_tls_session_key_log_file_path(
  absl::StrCat(BASE_DIR, "/ssl-key-log-client.txt"));
config.set_tls_session_key_log_format(GRPC_TLS_KEY_LOG_FORMAT_NSS);
options.set_tls_session_key_log_config(config);
auto channel = grpc::CreateChannel(server_address,
  grpc::experimental::TlsCredentials(options)));
```

## Server Side (per-port) C++ API usage

```
grpc::experimental::TlsServerCredentialsOptions options(certificate_provider);
grpc::experimental::TlsSessionKeyLoggerConfig config;
config.set_tls_session_key_log_file_path(
  absl::StrCat(BASE_DIR, "/ssl-key-log-server.txt"));
config.set_tls_session_key_log_format(GRPC_TLS_KEY_LOG_FORMAT_NSS);
options.set_tls_session_key_log_config(config);
...
ServerBuilder builder;
builder.AddListeningPort(server_address,
      grpc::experimental::TlsCredentials(options));
```

## C-core API Usage

```
grpc_tls_credentials_options * options = ...
...
// create a session key log config
grpc_tls_session_key_log_config * config = grpc_tls_session_key_log_config_create();

// set config parameters
grpc_tls_session_key_log_config_set_log_path(config, "/tmp/test.txt");
grpc_tls_session_key_log_config_set_log_format(config, GRPC_TLS_KEY_LOG_FORMAT_NSS);

// associate with a grpc_tls_credentials_options object
grpc_tls_credentials_options_set_tls_session_key_log_config(options, config);

// release ownership of config
grpc_tls_session_key_log_config_release(config);
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
