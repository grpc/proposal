L46: C-core: New TLS Credentials API
----
* Author(s): ZhenLian
* Approver: markdroth, yashykt, jiangtaoli2016
* Status: Draft
* Implemented in:
* Last updated: September 9, 2020

## Abstract

This proposal aims to provide the following features in a new TLS credential API:
1. credential reloading: reload transport credentials periodically from various sources, e.g. from credential files, or users' customized Provider implementations, without shutting down
2. custom authorization check: support the following cases
    - on the client side:
        - perform both the certificate and default hostname check(default option)
        - perform certificate check but skip default hostname check(a custom authorization check is mandatory)
        - skip both certificate check and default hostname check(a custom authorization check is mandatory)
        - perform a custom authorization check, if needed 
    - on the server side:
        - perform a custom authorization check, if needed    
3. TLS version configuration: optionally configure the minimum and maximum of the TLS version that will be used in the handshake

## Background

The current TLS implementation in gRPC C core has some restrictions:
1. lack of the credential reloading support. Once an application starts, it can never reload its identity key/cert-chain and root certificates without shutting down
2. inflexibility in the peer verification. In a mutual TLS scenario, servers would always perform certificate verification, and clients would always perform certificate verification as well as default hostname check. No customized checks can be introduced.
3. inability to choose the maximum/minimum TLS version supported in OpenSSL/BoringSSL for use

There have always been some ad hoc requests on GitHub Issue Page for supporting these features. The demands for a more flexible and configurable API have increased with the emerging needs in cloud platform.

One common example is to support SPIFFE ID as the identity format, which requires us to disable the default hostname check on the client side. 
The default hostname check is an exact check on the hostname sent from client and the identity shown on server's certificate, but in SPIFFE world, the identity on server's certificate is a SPIFEE ID, which doesn't necessarily match up with the host name. 

Another example is to reload the root and identity credentials without shutting down the server, because certificates will eventually expire and the ability to rotate the certificates is essential to a cloud application.

Hence we propose an API that will meet the following requirements:  

1. Flexible enough to support multiple use cases. Users should be able to write their own reloading or authorization logic if the provided ones doesn't fit their needs
2. The credentials reloading should be efficient. We shall reload only when needed rather than reloading per TLS connection
3. C core API needs to be simple enough, so that most of existing SSL credentials in wrapped languages can migrate to call this new API

### Related Proposals:

[L46: Create a new TLS credential API that supports SPIFFE mutual TLS](https://github.com/grpc/proposal/pull/98)

## Proposal

### TLS credentials and TLS Options
This part of the proposal introduces `grpc_tls_credentials_options` and TLS credentials. Users use `grpc_tls_credentials_options` to configure the specific security properties they want, and build the client/server credential by passing `grpc_tls_credentials_options` in.
This section only covers the configuration not related to credential reloading and custom verification. Configurations related to those two topics are explained in the latter sections.

```c
/** --- TLS channel/server credentials ---
 * It is used for experimental purpose for now and subject to change. */

typedef struct grpc_tls_credentials_options grpc_tls_credentials_options;

GRPCAPI grpc_tls_credentials_options* grpc_tls_credentials_options_create(void);

/**
 * Set grpc_ssl_client_certificate_request_type in credentials options.
 * If not set, we will use the default value GRPC_SSL_DONT_REQUEST_CLIENT_CERTIFICATE.
 */
GRPCAPI int grpc_tls_credentials_options_set_cert_request_type(
    grpc_tls_credentials_options* options,
    grpc_ssl_client_certificate_request_type type);

/**
 * Set grpc_tls_server_verification_option. 
 * If not set, we will use the default value GRPC_TLS_SERVER_VERIFICATION.
 * This should be called only on the client side.
 */
GRPCAPI int grpc_tls_credentials_options_set_server_verification_option(
    grpc_tls_credentials_options* options,
    grpc_tls_server_verification_option server_verification_option);

/**
 * Sets the minimum TLS version that will be negotiated during the TLS handshake.
 * If not set, the underlying SSL library will pick the version automatically.
 */
GRPCAPI void grpc_tls_credentials_options_set_min_tls_version(grpc_tls_credentials_options* options,
    grpc_tls_version min_tls_version);

/**
 * Sets the maximum TLS version that will be negotiated during the TLS handshake.
 * If not set, the underlying SSL library will pick the version automatically.
 */
GRPCAPI void grpc_tls_credentials_options_set_max_tls_version(grpc_tls_credentials_options* options, 
    grpc_tls_version max_tls_version);

/**
 * This method creates a TLS channel credential object.
 * It takes ownership of the options parameter. 
 */
grpc_channel_credentials* grpc_tls_credentials_create(grpc_tls_credentials_options* options);

grpc_server_credentials* grpc_tls_server_credentials_create(grpc_tls_credentials_options* options);
```
Here is an example of how to use this API:
```c
/* Create option and set basic security primitives. */
grpc_tls_credentials_options* options = grpc_tls_credentials_options_create();
grpc_tls_credentials_options_set_server_verification_option(options, GRPC_TLS_SKIP_HOSTNAME_VERIFICATION);
grpc_tls_server_credentials_set_min_tls_version(options, TLS1_2);
grpc_tls_server_credentials_set_max_tls_version(options, TLS1_3);
/* Create TLS channel credentials. */
grpc_channel_credentials* creds = grpc_tls_credentials_create(options);
```

### Credential Reloading
The credential reloading API basically consists of two parts: the top-level `Provider` and the low-level `Distributor`.
The `Distributor` is the actual component for caching and distributing the credentials to the underlying transport connections(security connectors).
The `Provider` offers a general interface for different implementations to interact with the `Distributor`. 
```c
/* Opaque type. */
typedef struct grpc_tls_certificate_provider grpc_tls_certificate_provider;
typedef struct grpc_tls_certificate_distributor grpc_tls_certificate_distributor;
typedef struct grpc_tls_certificate_provider_external grpc_tls_certificate_provider_external;

/* The ownership of grpc_tls_certificate_provider_external will be taken by grpc_tls_certificate_provider. */
grpc_tls_certificate_provider* grpc_tls_certificate_provider_external_create(
    grpc_tls_certificate_provider_external* external_provider);

/**
 * Sets the name of the root certificates being used in the distributor. 
 * Most users don't need to set this value.
 * If not set, we will use the default name, which is an empty string.
 */
GRPCAPI int grpc_tls_credentials_options_set_root_cert_name(
    grpc_tls_credentials_options* options,
    const char* root_cert_name);

/**
 * Sets the name of the identity certificates being used in the distributor. 
 * Most users don't need to set this value.
 * If not set, we will use the default name, which is an empty string.
 */
GRPCAPI int grpc_tls_credentials_options_set_identity_cert_name(
    grpc_tls_credentials_options* options,
    const char* identity_cert_name);

/** Sets the credential provider. */
GRPCAPI int grpc_tls_credentials_options_set_certificate_provider(
    grpc_tls_credentials_options* options,
    grpc_tls_certificate_provider* provider);

/** ------------------------------------ Provider ------------------------------------ */

/* Factory function for file-watcher provider (implemented inside core). */
grpc_tls_certificate_provider* grpc_tls_certificate_provider_file_watcher_create(const char* private_key_file_name, const char* identity_certificate_file_name, const char* root_certificate_file_name);
/* Factory function for static file provider (implemented inside core). */
grpc_tls_certificate_provider* grpc_tls_certificate_provider_file_static_create(const char* private_key_file_name, const char* identity_certificate_file_name, const char* root_certificate_file_name);

/* API for creating external provider. */
struct grpc_tls_certificate_provider_external {
  grpc_tls_certificate_distributor* (*get_distributor)(grpc_tls_certificate_provider_external*);
  void (*destroy)(grpc_tls_certificate_provider_external*);
};

/** ------------------------------------ Distributor ------------------------------------ */
/* Creates a TLS certificate distributor object. This object is ref-counted. */
GRPCAPI grpc_tls_certificate_distributor* grpc_tls_certificate_distributor_create();

/* Unrefs a TLS certificate distributor object. */
GRPCAPI void grpc_tls_certificate_distributor_unref(grpc_tls_certificate_distributor* distributor);

/* Sets root certificates and key and certificate chain pairs. The Ownerships of
   cert_name, pem_root_certs and pem_key_cert_pairs are not transferred. */
GRPCAPI int grpc_tls_certificate_distributor_set_key_materials(
  grpc_tls_certificate_distributor* distributor,
  const char* cert_name,
  const char* pem_root_certs,
  const grpc_ssl_pem_key_cert_pair** pem_key_cert_pairs, size_t num_key_cert_pairs);

/* Sets root certificates. The Ownership of cert_name and pem_root_certs are not transferred. */
GRPCAPI int grpc_tls_certificate_distributor_set_root_certs(
  grpc_tls_certificate_distributor* distributor, const char* cert_name,
  const char* pem_root_certs);

/* Sets key and certificate chain pairs. The Ownership of cert_name, and pem_key_cert_pairs are not transferred. */
GRPCAPI int grpc_tls_certificate_distributor_set_key_cert_pairs(
  grpc_tls_certificate_distributor* distributor, 
  const char* cert_name,
  const grpc_ssl_pem_key_cert_pair** pem_key_cert_pairs, size_t num_key_cert_pairs);

/* Sets errors for both the root and the identity certificates of name |cert_name|. 
   At least one of root_cert_error and identity_cert_error must be set. */
GRPCAPI int grpc_tls_certificate_distributor_set_error_for_key_materials(
  grpc_tls_certificate_distributor* distributor,
  const char* cert_name,
  grpc_tls_error_details* root_cert_error,
  grpc_tls_error_details* identity_cert_error);

/* Callback function to be called by the TLS certificate distributor to inform
   the TLS certificate provider when the watch status changed.
   - user_data is the opaque pointer that was passed to TLS certificate distributor
   - watching_root_certs is a boolean value indicating that root certificates are
     being watched.
   - watching_key_cert_pairs is a boolean value indicating that the key certificate
     pairs are being watched. 
   - cert_name is the name of the certificates being watched.
*/
typedef void (*grpc_tls_certificate_watch_status_cb)(
    void* user_data, const char* cert_name, bool watching_root_certs, bool watching_key_cert_pairs);

/* Sets the watch status callback on the distributor.
   Callbacks are invoked synchronously. The user_data parameter will be passed
   to the callback when it is invoked.
   The callback can be set to null to disable callbacks.  Callers should generally
   set it to null before they unref the distributor, to ensure that no subsequent
   callbacks are invoked. */
GRPCAPI void grpc_tls_certificate_distributor_set_watch_status_callback(
    grpc_tls_certificate_distributor* distributor,
    grpc_tls_certificate_watch_status_cb cb, void* user_data);
```
If only the already implemented providers are needed, e.g. file-watcher provider or static-cert provider, those provider implementations can be directly passed in without caring about `Distributor`. 
In that case, How to wrap it should be straightforward. 
The Core API will be called like this: 
```c
grpc_tls_credentials_options* options = grpc_tls_credentials_options_create();
// Use the file-based provider for reloading certificates.
grpc_tls_certificate_provider* file_provider = grpc_tls_certificate_provider_file_watcher_create(...);
grpc_tls_credentials_options_set_certificate_provider(options, file_provider);
```
And wrap languages can just wrap `grpc_tls_certificate_provider_file_watcher_create`.
 
If the wrap language also wants users to be able to implement their own `Provider` implementations, a bit of the design is needed, because the wrapper class needs to expose part of the `Distributor` API.
The Core API will be called like this: 
```c
static grpc_tls_certificate_distributor* GetDistributor(grpc_tls_certificate_provider_external* external_provider) {
    /* .... */
}
static void Destroy(grpc_tls_certificate_provider_external* external_provider) {
    /* .... */
    delete  external_provider;
}
grpc_tls_certificate_provider_external* external_provider = new grpc_tls_certificate_provider_external(); 
external_provider->get_distributor = GetDistributor;
external_provider->destroy = Destroy;
/* The ownership will be taken by provider. */
grpc_tls_certificate_provider* my_provider = grpc_tls_certificate_provider_external_create(external_provider);
grpc_tls_credentials_options_set_certificate_provider(options, my_provider);
```
Each wrap language is free to choose the design suitable for its language characteristics, but consider some general advices first:
1. create an interface-like class `ProviderInterface` which contains two functions GetDistributor() and Destroy()
2. create a concrete class `FileCertificateProvider` that implements `ProviderInterface` and wraps the provider created by `grpc_tls_certificate_provider_file_watcher_create`
3. create a concrete class `StaticCertificateProvider` that implements `ProviderInterface` and wraps the provider created by `grpc_tls_certificate_provider_file_static_create`
4. create an interface-like class `CertificateProvider` that implements `ProviderInterface`, and exposes several APIs in the distributor to users, such as `grpc_tls_certificate_distributor_set_key_materials`, etc
5. users could extend `CertificateProvider` to define their own provider implementations

As a reference, here is how `CertificateProvider` might be defined in C++:
```cpp
namespace grpc {

template <class Subclass>
class CertificateProvider {
 public:
  CertificateProvider()
      : distributor_(grpc_tls_certificate_distributor_create()) {
    base.get_distributor = GetDistributor;
    base.destroy = Destroy;
    grpc_tls_certificate_distributor_set_watch_status_callback(
        OnWatchStatusChangedCallback, this);
  }

  virtual ~CertificateProvider() {
    grpc_tls_certificate_distributor_destroy(distributor_);
  }

  grpc_tls_certificate_provider* c_provider() const {
    return grpc_tls_certificate_provider_external_create(&base_);
  }

  virtual void OnWatchStatusChanged(
      const std::string& cert_name, bool watching_root,
      bool watching_identity) = 0;

  void SetRootCert(const std::string& cert_name,
                   const std::string& contents) {
    grpc_tls_certificate_distributor_set_root_certs(
        distributor_, cert_name.c_str(), contents.c_str());
  }

  // ...similar functions for SetCertKeyPairs() and SetKeyMaterials()...

 private:
  static grpc_tls_certificate_distributor* GetDistributor(
      grpc_tls_certificate_provider_external* external_provider) {
    auto* provider = static_cast<Subclass*>(external_provider);
    return provider->distributor_;
  }

  static void Destroy(
      grpc_tls_certificate_provider_external* external_provider) {
    auto* provider = static_cast<Subclass*>(external_provider);
    delete provider;
  }

  static void OnWatchStatusChangedCallback(
      void* arg, const std::string& cert_name, bool watching_root,
      bool watching_identity) {
    auto* provider = static_cast<Subclass*>(arg);
    provider->OnWatchStatusChanged(cert_name, watching_root,
                                   watching_identity);
  }

  // First data element to simulate inheritance.
  grpc_tls_certificate_provider_external base_;

  grpc_tls_certificate_distributor* distributor_;
};

}  // namespace grpc
```
and an example of custom provider implementation in C++:
```cpp
class MyCertificateProvider: public grpc::CertificateProvider<MyCertificateProvider> {
 public:
  void OnWatchStatusChanged(
      const std::string& cert_name, bool watching_root,
      bool watching_identity) override {
    if (!watching_root && !watching_identity) {
      certs_watching_.erase(cert_name);
    } else {
      certs_watching_[cert_name] = {watching_root, watching_identity};
      // ...start or stop watching as needed...
    }
  }

 private:
  struct CertsWatching {
    bool watching_root = false;
    bool watching_identity = false;
  };
  std::map<std::string /*cert_name*/, CertsWatching> certs_watching_;
};
```
### Custom Authorization
```c
typedef struct grpc_tls_authorization_check_arg  grpc_tls_authorization_check_arg;
typedef struct grpc_tls_authorization_check_config  grpc_tls_authorization_check_config;

GRPCAPI int grpc_tls_credentials_options_set_authorization_check_config(
    grpc_tls_credentials_options* options,
    grpc_tls_authorization_check_config* config);

/** ------------------------------------ Authorization Check ------------------------------------ */

/** callback function provided by gRPC used to handle the result of server
    authorization check. It is used when schedule API is implemented
    asynchronously, and serves to bring the control back to gRPC C core.  */
typedef void (*grpc_tls_on_authorization_check_done_cb)(
    grpc_tls_authorization_check_arg* arg);

/** A struct containing all information necessary to schedule/cancel a server
    authorization check request.
    - cb and cb_user_data represent a gRPC-provided callback and an argument
      passed to it.
    - success will store the result of server authorization check. That is,
      if success returns a non-zero value, it means the authorization check
      passes and if returning zero, it means the check fails.
   - target_name is the name of an endpoint the channel is connecting to.
   - peer_cert represents a complete certificate chain including both
     signing and leaf certificates.
   - status and error_details contain information
     about errors occurred when a server authorization check request is
     scheduled/cancelled.
   - config is a pointer to the unique
     grpc_tls_authorization_check_config instance that this argument
     corresponds to.
   - context is a pointer to a wrapped language implementation of this
     grpc_tls_authorization_check_arg instance.
   - destroy_context is a pointer to a caller-provided method that cleans
      up any data associated with the context pointer.
*/
struct grpc_tls_authorization_check_arg {
  grpc_tls_on_authorization_check_done_cb cb;
  void* cb_user_data;
  int success;
  const char* target_name;
  const char* peer_cert;
  const char* peer_cert_full_chain;
  grpc_status_code status;
  grpc_tls_error_details* error_details;
  grpc_tls_authorization_check_config* config;
  void* context;
  void (*destroy_context)(void* ctx);
};

/** Create a grpc_tls_authorization_check_config instance.
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
GRPCAPI grpc_tls_authorization_check_config*
grpc_tls_authorization_check_config_create(
    const void* config_user_data,
    int (*schedule)(void* config_user_data,
                    grpc_tls_authorization_check_arg* arg),
    void (*cancel)(void* config_user_data,
                   grpc_tls_authorization_check_arg* arg),
    void (*destruct)(void* config_user_data));
```
Custom authorization can be performed either synchronously or asynchronously.
Here is an example of how it could be done synchronously:
```c
int TlsAuthorizationCheckSchedule(
    void* /*config_user_data*/, grpc_tls_authorization_check_arg* arg) {
  if (arg->target_name == "google.com") {
    arg->status = GRPC_STATUS_OK;
    arg->success = 1;
    return 0;
  }
  arg->status = GRPC_STATUS_CANCELLED;
  arg->success = 0;
  return 0;
}
grpc_tls_authorization_check_config* config = grpc_tls_authorization_check_config_create(nullptr, 
    &TlsAuthorizationCheckSchedule, nullptr, nullptr);
grpc_tls_credentials_options_set_authorization_check_config(options, config);
```
And example for asynchronous check:
```c
void TimeConsumingOperation(grpc_tls_authorization_check_arg* arg) {
  // ... Some time-consuming operations ...
  if (arg->target_name == "google.com") {
    arg->status = GRPC_STATUS_OK;
    arg->success = 1;
    (*arg->cb)(arg);
  } else {
    arg->status = GRPC_STATUS_CANCELLED;
    arg->success = 0;
    (*arg->cb)(arg);
  }
}
int TlsAuthorizationCheckSchedule(
    void* /*config_user_data*/, grpc_tls_authorization_check_arg* arg) {
  pthread_t thread;
  pthread_create(&thread, NULL, TimeConsumingOperation, arg);
  return 1;
}
grpc_tls_authorization_check_config* config = grpc_tls_authorization_check_config_create(nullptr, 
    &TlsAuthorizationCheckSchedule, nullptr, nullptr);
grpc_tls_credentials_options_set_authorization_check_config(options, config);
```