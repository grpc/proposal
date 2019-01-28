Title
----
* Author(s): yihuazhang
* Approver: vjpai, markdroth
* Status: Draft
* Implemented in:
* Last updated: September 3, 2018
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/gMrROsizrtc

## Abstract
This proposal aims to incorporate SPIFFE-based mutual TLS into gRPC C core auth stack. Towards the goal, it proposes a new credential API model that 1) allows flexible addition of new TLS-related features (e.g., credential reload) without breaking the API and 2) supports both synchronous and asynchronous implementations of those features. It then proposes SPIFFE and HTTPS-based TLS credential API's that comply with the new model.

## Background

The current TLS implementation in gRPC C core has the following problems when it comes to support SPIFFE-based mutual TLS.

1: It is originally designed to comply with HTTPS semantics which makes it difficult to be configured to support SPIFFE-based mutual TLS. For instance, the server authorization check in the current implementation is enforced locally by leveraging whitelisted endpoint information embedded in peer certificates while in SPIFFE use case, it may be enforced via an external server authorization check service. Here, the server authorization check is used to validate if a peer is authorized to run a job with a specific target name, and is invoked only at the client-side.

2: It realizes TLS-related features such as credential reload and server authorization check in a synchronous manner. It is undesirable in SPIFFE use case as those operations could involve expensive RPC calls to other services which may block gRPC core threads.

3: The current TLS credential API is inflexible in incorporating new TLS-related features in future as not only does it break the API, it also requires all wrapped languages to update their implementations in a lockstep manner.

This proposal introduces a new credential API model that allows flexible addition of TLS-related features that can be implemented in either a synchronous or asynchronous manner. It then proposes new credential API's for both HTTPS and SPIFFE-based TLS that complies with the API model.

### Related Proposals:
N/A

## Proposal

The first part of proposal introduces credential options and credential create APIs that will be added to include/grpc/grpc_security.h. Note that although the credentials options provided to HTTPS and SPIFFE-based TLS credential API have the same format (i.e., grpc_tls_credentials_options), some of its members will not get populated and hence used in some API's, for instance, grpc_tls_server_authorization_check_config will be ignored in server-side API's.

```c++

/** Config for TLS key materials. */
typedef struct grpc_tls_key_materials_config grpc_tls_key_materials_config;

/** Config for TLS credential reload. */
typedef struct grpc_tls_credential_reload_config
    grpc_tls_credential_reload_config;

/** Config for TLS server authorization check. */
typedef struct grpc_tls_server_authorization_check_config
    grpc_tls_server_authorization_check_config;

/** TLS credentials options. */
typedef struct grpc_tls_credentials_options grpc_tls_credentials_options;

/** Create an empty TLS credentials options. */
GRPCAPI grpc_tls_credentials_options* grpc_tls_credentials_options_create();

/** Set grpc_ssl_client_certificate_request_type field in credentials options
    with the provided type. options should not be NULL.
    It returns 1 on success and 0 on failure. */
GRPCAPI int grpc_tls_credentials_options_set_cert_request_type(
    grpc_tls_credentials_options* options,
    grpc_ssl_client_certificate_request_type type);

/** Set grpc_tls_key_materials_config field in credentials options
    with the provided config struct whose ownership is transferred.
    Both parameters should not be NULL.
    It returns 1 on success and 0 on failure. */
GRPCAPI int grpc_tls_credentials_options_set_key_materials_config(
    grpc_tls_credentials_options* options,
    grpc_tls_key_materials_config* config);

/** Set grpc_tls_credential_reload_config field in credentials options
    with the provided config struct whose ownership is transferred.
    Both parameters should not be NULL.
    It returns 1 on success and 0 on failure. */
GRPCAPI int grpc_tls_credentials_options_set_credential_reload_config(
    grpc_tls_credentials_options* options,
    grpc_tls_credential_reload_config* config);

/** Set grpc_tls_server_authorization_check_config field in credentials options
    with the provided config struct whose ownership is transferred.
    Both parameters should not be NULL.
    It returns 1 on success and 0 on failure. */
GRPCAPI int grpc_tls_credentials_options_set_server_authorization_check_config(
    grpc_tls_credentials_options* options,
    grpc_tls_server_authorization_check_config* config);

/** Destroy a TLS credentials options. */
GRPCAPI void grpc_tls_credentials_options_destroy(
                               grpc_tls_credentials_options* options);

/** Create an HTTPS-based TLS server credential object using the provided
    options struct whose ownership is transferred. */
GRPCAPI grpc_server_credentials* grpc_tls_https_server_credentials_create(
                               const grpc_tls_credentials_options* options);

/** Create an HTTPS-based TLS channel credential object using the provided
    options struct whose ownership is transferred. */
GRPCAPI grpc_channel_credentials* grpc_tls_https_credentials_create(
                               const grpc_tls_credentials_options* options);

/** Create a SPIFFE-based TLS server credential object using the provided
    options struct whose ownership is transferred. */
GRPCAPI grpc_server_credentials* grpc_tls_spiffe_server_credentials_create(
                               const grpc_tls_credentials_options* options);

/** Create a SPIFFE-based TLS channel credential object using the provided
    options struct whose ownership is transferred. */
GRPCAPI grpc_channel_credentials* grpc_tls_spiffe_credentials_create(
                               const grpc_tls_credentials_options* options);
```

The second part of proposal introduces API's related to configs of TLS-related features that will be added to include/grpc/grpc_security.h.

```c++
/** --- TLS key materials config. --- */

/** Create an empty grpc_tls_key_materials_config instance. */
GRPCAPI grpc_tls_key_materials_config* grpc_tls_key_materials_config_create();

/** Set grpc_tls_key_materials_config instance with provided a TLS certificate.
    config will take the ownership of pem_root_certs and pem_key_cert_pairs.
    It's valid for the caller to provide nullptr pem_root_certs, in which case
    the gRPC-provided root cert will be used. pem_key_cert_pairs should not be
    NULL. It returns 1 on success and 0 on failure.
 */
GRPCAPI int grpc_tls_key_materials_config_set_key_materials(
    grpc_tls_key_materials_config* config, const char* pem_root_certs,
    const grpc_ssl_pem_key_cert_pair** pem_key_cert_pairs,
    size_t num_key_cert_pairs);

/** --- TLS credential reload config. --- */

typedef struct grpc_tls_credential_reload_arg grpc_tls_credential_reload_arg;

/** A callback function provided by gRPC to handle the result of credential
    reload. It is used when schedule API is implemented asynchronously and
    serves to bring the control back to grpc C core. */
typedef void (*grpc_tls_on_credential_reload_done_cb)(
    grpc_tls_credential_reload_arg* arg);

/** A struct containing all information necessary to schedule/cancel
    a credential reload request. cb and cb_user_data represent a gRPC-provided
    callback and an argument passed to it. key_materials is an in/output
    parameter containing currently used/newly reloaded credentials. status and
    error_details are used to hold information about errors occurred when a
    credential reload request is scheduled/cancelled. */
struct grpc_tls_credential_reload_arg {
  grpc_tls_on_credential_reload_done_cb cb;
  void* cb_user_data;
  grpc_tls_key_materials_config* key_materials_config;
  grpc_status_code status;
  const char* error_details;
};

/** Create a grpc_tls_credential_reload_config instance.
    - config_user_data is config-specific, read-only user data
      that works for all channels created with a credential using the config.
    - schedule is a pointer to an application-provided callback used to invoke
      credential reload API. The implementation of this method has to be
      non-blocking, but can be performed synchronously or asynchronously.
      1) If processing occurs synchronously, it populates arg->key_materials,
      arg->status, and arg->error_details and returns zero.
      2) If processing occurs asynchronously, it returns a non-zero value.
      The application then invokes arg->cb when processing is completed. Note
      that arg->cb cannot be invoked before schedule API returns.
    - cancel is a pointer to an application-provided callback used to cancel
      a credential reload request scheduled via an asynchronous schedule API.
      arg is used to pinpoint an exact reloading request to be cancelled.
      The operation may not have any effect if the request has already been
      processed.
    - destruct is a pointer to an application-provided callback used to clean up
      any data associated with the config.
*/
GRPCAPI grpc_tls_credential_reload_config*
grpc_tls_credential_reload_config_create(
    const void* config_user_data,
    int (*schedule)(void* config_user_data,
                    grpc_tls_credential_reload_arg* arg),
    void (*cancel)(void* config_user_data, grpc_tls_credential_reload_arg* arg),
    void (*destruct)(void* config_user_data));

/** --- TLS server authorization check config. --- */
typedef struct grpc_tls_server_authorization_check_arg
    grpc_tls_server_authorization_check_arg;

/** callback function provided by gRPC used to handle the result of server
    authorization check. It is used when schedule API is implemented
    asynchronously, and serves to bring the control back to gRPC C core. */
typedef void (*grpc_tls_on_server_authorization_check_done_cb)(
    grpc_tls_server_authorization_check_arg* arg);

/** A struct containing all information necessary to schedule/cancel a server
   authorization check request. cb and cb_user_data represent a gRPC-provided
   callback and an argument passed to it. result will store the result of
   server authorization check. target_name is the name of an endpoint the
   channel is connecting to and certificate represents a complete certificate
   chain including both signing and leaf certificates. status and error_details
   contain information about errors occurred when a server authorization check
   request is scheduled/cancelled. */
struct grpc_tls_server_authorization_check_arg {
  grpc_tls_on_server_authorization_check_done_cb cb;
  void* cb_user_data;
  int result;
  const char* target_name;
  const char* peer_cert;
  grpc_status_code status;
  const char* error_details;
};

/** Create a grpc_tls_server_authorization_check_config instance.
    - config_user_data is config-specific, read-only user data
      that works for all channels created with a credential using the config.
    - schedule is a pointer to an application-provided callback used to invoke
      server authorization check API. The implementation of this method has to
      be non-blocking, but can be performed synchronously or asynchronously.
      1)If processing occurs synchronously, it populates arg->result,
      arg->status, and arg->error_details and returns zero.
      2) If processing occurs asynchronously, it returns a non-zero value. The
      application then invokes arg->cb when processing is completed. Note that
      arg->cb cannot be invoked before schedule API returns.
    - cancel is a pointer to an application-provided callback used to cancel a
      server authorization check request scheduled via an asynchronous schedule
      API. arg is used to pinpoint an exact check request to be cancelled. The
      operation may not have any effect if the request has already been
      processed.
    - destruct is a pointer to an application-provided callback used to clean up
      any data associated with the config.
*/
GRPCAPI grpc_tls_server_authorization_check_config*
grpc_tls_server_authorization_check_config_create(
    const void* config_user_data,
    int (*schedule)(void* config_user_data,
                    grpc_tls_server_authorization_check_arg* arg),
    void (*cancel)(void* config_user_data,
                   grpc_tls_server_authorization_check_arg* arg),
    void (*destruct)(void* config_user_data));
```

## Implementation

The implementation of this proposal is straightforward as all API changes will be made to include/grpc/grpc_security.h. Note that once the new HTTPS-based TLS credential API gets implemented, the old HTTPS-based TLS credential API will get deprecated and be discouraged to use.
