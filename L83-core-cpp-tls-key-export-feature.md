# Title
-------
* Author(s): Vignesh Babu (Vignesh2208)
* Approver: ctiller
* Status: In Review
* Implemented in: core, cpp
* Last updated: 09/15/2021
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

* A `grpc_tls_key_log_format` enum is added to `include/grpc/grpc_security_constants.h`
  ```
  /** Supported TLS Keylogging formats **/
  typedef enum {
    // Default key logging formal is NSS compatible.
    GRPC_TLS_KEY_LOG_FORMAT_NSS = 0,
    // More formats can be added here for future extensions if desired.
  } grpc_tls_key_log_format;
  ```

* A `grpc::experimental::TlsKeyLoggerConfig` class is defined in `include/grpcpp/security/tls_credentials_options.h` to support configuration for session key export. It currently has two members:
  * `std::string tls_key_log_file_path`: Full path where the TLS keys should be exported to.
  * `grpc_tls_key_log_format  tls_key_log_format`: Format of the exported keys.

  The `grpc::experimental::TlsKeyLoggerConfig` class has the following signature:
  ```
  class TlsKeyLoggerConfig {
    private:
      // The full path at which the TLS keys would be exported.
      std::string tls_key_log_file_path;

      // The format in which the TLS keys would be exported at this path.
      grpc_tls_key_log_format tls_key_log_format;
    public:
      // Constructor
      explicit TlsKeyLoggerConfig();

      // Copy ctor
      TlsKeyLoggerConfig(const TlsKeyLoggerConfig& copy)

      // Sets the tls key log file path.
      void set_tls_key_log_file_path(std::string key_log_file_path);

      // Sets the tls key logging format. Currently only NSS format is supported.
      void set_tls_key_log_format(grpc_tls_key_log_format tls_key_log_format);

      // Returns the set tls key log file path.
      std::string get_tls_key_log_file_path();

      // Returns the set tls key log file format.
      grpc_tls_key_log_format get_tls_key_log_format();
  };
  ```

* A `set_tls_key_log_config(grpc::experimental::TlsKeyLoggerConfig cfg)` method is added to the existing `grpc::experimental::TlsCredentialsOptions` class to set the desired TLS session key export configuration on the client/server side.

These changes allow specifying TLS Key export configuration on a per-channel level on the client side and on a per-port level on the server side. If the configuration is not explicitly set, TLS key logging is disabled. This config  struct can be extended in the future to support advanced features such as key logging based on source ip filtering etc.

For extra protection, these configurations only take effect when the `GRPC_TLS_KEY_LOGGING_ENABLED` environment variable is set to true.

## C-Core API Additions

The following additions are made to the gRPC C-Core API to enable TLS session key logging. The API additions are described below with inline comments.

```
/** --- TLS Key logging. ---
 * Experimental API to control tls key logging. Tls key logging is expected
 * to be used only for debugging purposes and never in production. Tls
 * key logging is only enabled when:
 * 1. At least one grpc_tls_credentials_options object is assigned a tls
 *  key logging config using the API specified below.
 * 2. The GRPC_TLS_KEY_LOGGING_ENABLED environment variable is set to true.
 */

/**
 * An opaque type pointing to a Tls Key logger helper object which provides
 * methods to log TLS session keys to files.
 */
typedef struct gprc_tls_key_logger grpc_tls_key_logger;

/**
 * An opaque type pointing to a Tls key logging configuration.
 */
typedef struct grpc_tls_key_log_config grpc_tls_key_log_config;

/**
 * Associates a key logging config with a a grpc_tls_credentials_options object.
 * - options is the grpc_tls_credentials_options object
 * - tls_key_log_file_path contains the path where tls keys need to be logged
 * - key_log_format contains the format in which the tls keys will be logged.
 */
GRPCAPI void grpc_tls_credentials_options_set_tls_key_log_config(
    grpc_tls_credentials_options* options, grpc_tls_key_log_config* config);

/**
 * Creates a Tls Key logger helper object corresponding to a
 * grpc_tls_credentials_options object. If no tls config was associated with
 * the credentials options object using the above API, then this method returns
 * nullptr.
 * The returned helper object provides methods to log session keys. Currently
 * it points to an object of type tsi::TlsKeyLogger which has the following
 * public method to support TLS session key logging:

    void LogSessionKeys(SSL_CTX* ssl_context,
                        const std::string& session_keys_info);

 * It writes session keys into the file according to the bound configuration.
 * This is called upon completion of a handshake. An optional ssl_context can
 * also provided here to support future extensions such as logging keys only
 * when connections are made by certain IPs etc.
 *
 * The grpc_tls_key_logger_create method can only be invoked after
 * grpc_tls_key_logger_registry_init() method has already been invoked.
 * Otherwise it would return nullptr.
 *
 * - options refers to the grpc_tls_credentials_options object which is parsed
 *  to read any previously set tls key log configuration. A Tls key logger
 *  object is constructed based on this configuration.
 */
GRPCAPI grpc_tls_key_logger* grpc_tls_key_logger_create(
    grpc_tls_credentials_options* options);

/**
 * Destroys/unrefs a previously created Tls Key logger helper object.
 * key_logger - the object to destroy/unref.
 */
GRPCAPI void grpc_tls_key_logger_destroy(grpc_tls_key_logger* key_logger);

/**
 * Initializes a Tls Key logger registry. Tls Key logger objects can only be
 * created after the registry is initialized.
 */
GRPCAPI void grpc_tls_key_logger_registry_init();

/**
 * Destroys the Tls Key logger registry. No Tls Key logger objects can be
 * created after the registry is destroyed.
 */
GRPCAPI void grpc_tls_key_logger_registry_destroy();

```

# Example Usage

## Client Side (per-channel) C++ API usage

```
grpc::experimental::TlsChannelCredentialsOptions options;
grpc::experimental::TlsKeyLoggerConfig config;
config.set_tls_key_log_file_path(
  absl::StrCat(BASE_DIR, "/ssl-key-log-client.txt"));
config.set_tls_key_log_format(GRPC_TLS_KEY_LOG_FORMAT_NSS);
options.set_tls_key_log_config(config);
auto channel = grpc::CreateChannel(server_address,
  grpc::experimental::TlsCredentials(options)));
```

## Server Side (per-port) C++ API usage

```
grpc::experimental::TlsServerCredentialsOptions options(certificate_provider);
grpc::experimental::TlsKeyLoggerConfig config;
config.set_tls_key_log_file_path(
  absl::StrCat(BASE_DIR, "/ssl-key-log-server.txt"));
config.set_tls_key_log_format(GRPC_TLS_KEY_LOG_FORMAT_NSS);
options.set_tls_key_log_config(config);
...
ServerBuilder builder;
builder.AddListeningPort(server_address,
      grpc::experimental::TlsCredentials(options));
```


# Rationale

Alternatives using ENVIRONMENT Variables to set TLS Key export configurations were considered. These allow enabling/disabling key export without having to rebuild the binary. However the disadvantages here include:

* Safety issues related to setting environment variables and forgetting to unset them.
* Supporting per-channel or per-port key export configuration requires setting unique enviroment variables tied to the channel-id/port number which may not be known prior to run-time in all cases.

The proposed approach does not have these disadvantages but requires rebuilding the binary.


# Implementation

The full implementation of the TLS Key export feature can be found here: https://github.com/grpc/grpc/pull/26812

# Open issues (if applicable)

N/A
