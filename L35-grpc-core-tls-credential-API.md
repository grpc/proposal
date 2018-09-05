Title
----
* Author(s): yihuazhang
* Approver: vjpai, markdroth
* Status: Draft
* Implemented in:
* Last updated: September 3, 2018
* Discussion at: 

## Abstract
This proposal aims to incorporate SPIFFE-based mutual TLS into gRPC C core auth
stack. Towards the goal, it proposes a new credential API model that 1) allows flexible
addition of new TLS-related features (e.g., credential reload) without breaking the API and
2) supports both synchronous and asynchronous implementations of those features.
It then proposes SPIFFE and HTTPS-based TLS credential API's that comply with the new model.

## Background

The current TLS implementation in gRPC C core has the following problems when it
comes to support SPIFFE-based mutual TLS.

1: It is originally designed to comply with HTTPS semantics which makes it difficult
to be configured to support SPIFFE-based mutual TLS. For instance, the server authorization
check in the current implementation is enforced locally by leveraging whitelisted
endpoint information embedded in peer certificates while in SPIFFE use case,
it may be enforced via an external server authorization check service.

2: It realizes TLS-related features such as credential reload and server authorization check
in a synchronous manner. It is undesirable in SPIFFE use case as
those operations could involve expensive RPC calls to other services which may block gRPC core threads.

3: The current TLS credential API is inflexible in incorporating new TLS-related features in future
as not only does it break the API, it also requires all wrapped languages to update their
implementations in a lockstep manner.

This proposal introduces a new credential API model that allows flexible
addition of TLS-related features that can be implemented in either a synchronous or
asynchronous manner. It then proposes new credential API's for both HTTPS and SPIFFE-based TLS that
complies with the API model.

### Related Proposals:
N/A

## Proposal

The first part of proposal introduces credential options and credential create APIs
that will be added to include/grpc/grpc_security.h.

```c++

/** Config for direct provision of TLS key materials. */
typedef struct grpc_tls_key_materials_config grpc_tls_key_materials_config;

/** Config for TLS credential reload. */
typedef struct grpc_tls_credential_reload_config grpc_tls_credential_reload_config;

/** Config for TLS server authorization check. */
typedef struct grpc_tls_server_authorization_check
grpc_tls_server_authorization_check;

/** Config for TLS context customization. */
typedef struct grpc_tls_ctx_customize_config grpc_tls_ctx_customize_config;

/** TLS credentials options. */
typedef struct grpc_tls_credentials_options grpc_tls_credentials_options;

/** Create an empty TLS credentials options. */
GRPCAPI grpc_tls_credentials_options* grpc_tls_credentials_options_create();

/** Set grpc_ssl_client_certificate_request_type field in credentials options
    with the provided type. */
GRPCAPI void grpc_tls_credentials_options_set_cert_request_type(
                               grpc_tls_credentials_options* options,
                               grpc_ssl_client_certificate_request_type type);

/** Set grpc_tls_credential_reload_config field in credentials options
    with the provided config struct whose ownership is transferred. */
GRPCAPI void grpc_tls_credentials_options_set_credential_reload_config(
                               grpc_tls_credentials_options* options,
                               grpc_tls_credential_reload_config* config);


/** Set grpc_tls_ctx_customize_config field in credentials options
    with the provided config struct whose ownership is transferred. */
GRPCAPI void grpc_tls_credentials_options_set_tls_ctx_customize_config(
                               grpc_tls_credentials_options* options,
                               grpc_tls_ctx_customize_config* config);

/** Set grpc_tls_server_authorization_check_config field in credentials options
    with the provided config struct whose ownership is transferred. */
GRPCAPI void grpc_tls_credentials_options_set_server_authorization_check_config(
                               grpc_tls_credentials_options* options,
                               grpc_tls_server_authorization_check_config* config);

/** Destroy a TLS credentials options. */
GRPCAPI void grpc_tls_credentials_options_destroy(
                               grpc_tls_credentials_options* options);

/** Create an HTTPS-based TLS server credential object using the provided
    options struct whose ownership is transferred. */
GRPCAPI grpc_ssl_server_credentials* grpc_tls_https_server_credentials_create(
                               const grpc_tls_credentials_options* options);

/** Create an HTTPS-based TLS channel credential object using the provided
    options struct whose ownership is transferred. */
GRPCAPI grpc_ssl_credentials* grpc_tls_https_credentials_create(
                               const grpc_tls_credentials_options* options);

/** Create a SPIFFE-based TLS server credential object using the provided
    options struct whose ownership is transferred. */
GRPCAPI grpc_ssl_server_credentials* grpc_tls_spiffe_server_credentials_create(
                               const grpc_tls_credentials_options* options);

/** Create a SPIFFE-based TLS channel credential object using the provided
    options struct whose ownership is transferred. */
GRPCAPI grpc_ssl_credentials* grpc_tls_spiffe_credentials_create(
                               const grpc_tls_credentials_options* options);

``` 

The second part of proposal introduces API's related to configs of TLS-related
features that will be added to include/grpc/grpc_security.h.

```c++

/** --- TLS key materials config. --- */

/** Create an empty grpc_tls_key_materials_config instance. */
GRPCAPI grpc_tls_key_materials_config* grpc_tls_key_materials_config_create();

/** Add a set of root certficiates, private key and certificate chain to a config
    instance.
    - config is an instance of grpc_tls_key_materials_config struct.
    - pem_root_certs is the NULL-terminated string containing the PEM encoding of
      root certificates. If it is NULL, a default system root certificate or the
      one shipped with gRPC package will be used.
    - private_key is the NULL-terminated string containing the PEM encoding
      of private key.
    - cert_chain is the NULL-terminated string containing the PEM encoding of
      certificate chain.
*/
GRPCAPI void grpc_tls_key_materials_config_add(
                                        grpc_tls_key_materials_config* config,
                                        const char* pem_root_certs,
                                        const char* private_key,
                                        const char* cert_chain);

/** Destroy a grpc_tls_key_materials_config instance. */
GRPCAPI void grpc_tls_key_materials_config_destroy(grpc_tls_key_materials_config* config);

/** --- TLS credential reload config. --- */

/** A callback function provided by gRPC to handle the result of credential reload.
    It is used when schedule API is implemented asynchronously, and serves to
    bring the control back to grpc C core. */
typedef void (*grpc_tls_on_credential_reload_done_cb)(void* user_data);

/** A struct containing all information necessary to schedule/cancel
a credential reload request. cb and cb_user_data represent a gRPC-provided
callback and an argument passed to it. key_materials is an in/output parameter
containing currently used/newly reloaded credentials. status and error_details
are used to hold information about errors occurred when a credential reload request
is scheduled/cancelled. */
typedef struct {
  grpc_tls_on_credential_reload_done_cb cb;
  void *cb_user_data;
  grpc_tls_key_materials *key_materials;
  grpc_status_code *status;
  const char **error_details;
} grpc_tls_credential_reload_arg;

/** Create a grpc_tls_credential_reload_config instance.
    - config_user_data is config-specific, read-only user data
      that works for all channels created with a credential using the config.
    - schedule is a pointer to an application-provided callback used to invoke credential
      reload API. The implementation of this method has to be non-blocking,
      but can be performed synchronously or asynchronously.
      1) If processing occurs synchronously, it populates arg->key_materials,
      arg->status, and arg->error_details and returns zero.
      2) If processing occurs asynchronously, it returns a non-zero value.
      The application then invokes arg->cb when processing is completed. Note that
      arg->cb cannot be invoked before schedule API returns.
    - cancel is a pointer to an application-provided callback used to cancel a credential
      reload request scheduled via an asynchronous schedule API.
      arg is used to pinpoint an exact reloading request to be cancelled.
      The operation may not have any effect if the request has already been processed.
    - destruct is a pointer to an application-provided callback used to clean up any data
      associated with the config.
*/
GRPCAPI grpc_tls_credential_reload_config*
                                      grpc_tls_credential_reload_config_create(
                                      const void* config_user_data, 
                                      int (*schedule)(const void* config_user_data,
                                           grpc_tls_credential_reload_arg* arg), 
                                      void (*cancel)(const void* config_user_data,
                                            grpc_tls_credential_reload_arg* arg),
                                      void (*destruct)(const void* config_user_data));


/** Destroy a grpc_tls_credential_reload_config instance. */
GRPCAPI void grpc_tls_credential_reload_config_destroy(grpc_tls_credential_reload_config* config);

/** --- TLS context customization config. --- */

/** A struct containing all information necessary to customize a TLS context.
    context represents the underlying TLS context whose ownership is not transferred.
    status and error_details are used to hold information about errors occurred
    when scheduling a TLS context customize request.
*/
typedef struct {
  void* context;
  grpc_status_code* status;
  const char** error_details;
} grpc_tls_ctx_customize_arg;

/** Create a grpc_tls_ctx_customize_config instance.
    - config_user_data is config-specific, read-only user data
      that works for all channels created with a credential using the config.
    - schedule is a pointer to an application-provided callback used to invoke TLS context
      customization API. This method should be implemented synchronously.
    - destruct is a pointer to an application-provided callback used to clean up any data
      associated with the config.
*/
GRPCAPI grpc_tls_ctx_customize_config* grpc_tls_ctx_customize_config_create(
                                      const void* config_user_data,
                                      void (*schedule)(const void* config_user_data,
                                            grpc_tls_ctx_customize_arg* arg),
                                      void (*destruct)(const void* config_user_data));

/** Destroy a grpc_tls_ctx_customize_config instance. */
GRPCAPI void grpc_tls_ctx_customize_config_destroy(grpc_tls_ctx_customize_config* config);

/** --- TLS server authorization check config. --- */

/** callback function provided by gRPC used to handle the result of server authorization
    check. It is used when schedule API is implemented asynchronously, and
    serves to bring the control back to gRPC C core. */
typedef void (*grpc_tls_on_server_authorization_check_done_cb) (void* user_data);

/** A struct containing all information necessary to schedule/cancel a server
    authorization check request. cb and cb_user_data represent a gRPC-provided callback and
    an argument passed to it. result will store the result of server authorization
    check. target_name is the name of an endpoint the channel is connecting to and
    certificate represents a complete certificate chain including both signing and
    leaf certificates. status and error_details contain information about errors
    occurred when a server authorization check request is scheduled/cancelled. */
typedef struct {
   grpc_tls_on_server_authorization_check_done_cb cb;
   void *cb_user_data;
   bool *result;
   const char *target_name;
   const char *certificate;
   grpc_status_code *status;
   const char **error_details;
} grpc_tls_server_authorization_check_arg; 

/** Create a grpc_tls_server_authorization_check_config instance.
    - config_user_data is config-specific, read-only user data
      that works for all channels created with a credential using the config.
    - schedule is a pointer to an application-provided callback used to invoke server
      authorization check API. The implementation of this method has to be non-blocking,
      but can be performed synchronously or asynchronously.
      1) If processing occurs synchronously, it populates arg->result,
      arg->status, and arg->error_details and returns zero.
      2) If processing occurs asynchronously, it returns a non-zero value.
      The application then invokes arg->cb when processing is completed. Note that
      arg->cb cannot be invoked before schedule API returns.
    - cancel is a pointer to an application-provided callback used to cancel a server
      authorization check request scheduled via an asynchronous schedule API.
      arg is used to pinpoint an exact check request to be cancelled.
      The operation may not have any effect if the request has already been processed.
    - destruct is a pointer to an application-provided callback used to clean up any data
      associated with the config.
*/
GRPCAPI grpc_tls_server_authorization_check_config*
                                      grpc_tls_server_authorization_check_config_create(
                                      const void* config_user_data, 
                                      int (*schedule)(const void* config_user_data,
                                           grpc_tls_server_authorization_check_arg* arg), 
                                      void (*cancel)(const void* config_user_data,
                                            grpc_tls_server_authorization_check_arg* arg),
                                      void (*destruct)(const void* config_user_data));

/** Destroy a grpc_tls_server_authorization_check_config instance. */
GRPCAPI void grpc_tls_server_authorization_check_config_destroy(grpc_tls_server_authorization_check_config* config);

```

## Rationale

Regarding `grpc_tls_ctx_customize_config`, the config is used when a caller wants to
configure an underlying TLS context with customized primitives. For example, there could
be a use case in which raw private keys are not directly accessible (e.g., hardware-backed) to a caller,
but private key methods (e.g., “Sign” functions) are, in which case the config provides a means
to customize an TLS context with an `SSL_PRIVATE_KEY_METHOD` object and associated signing algorithms.

## Implementation

The implementation of this proposal is straightforward as all API changes will be
made to include/grpc/grpc_security.h.
