Title
----
* Author(s): ZhenLian
* Approver: markdroth, yashykt, jiangtaoli2016
* Status: Draft
* Implemented in:
* Last updated: September 7, 2020

## Abstract

This proposal aims to enhance the security features provided by [this already implemented proposal](https://github.com/grpc/proposal/pull/98). 
The improvements include:
1. replacing the per-connection callback-based credential reloading mechanism with the new distributor/provider mechanism
2. changing the server authorization check to an authorization check that could be applied to both the client side and the server side
3. adding options for users to configure their TLS version 

## Background

The current [TLS credential API](https://github.com/grpc/grpc/blob/master/src/core/lib/security/credentials/tls/tls_credentials.h) in gRPC C core implemented the following two features:

1. per-connection based credential reloading callback that allows callers to reload transport credentials without shutting down
2. per-connection based server authorization check on the client side that allows callers to do custom checks at the end of the handshake 

We recently have the following two use cases that we want to support:

1. In gRPC Proxyless Service Mesh, gRPC client has no initial credential to bootstrap with, which means the credentials fetching may not be instantaneous. Therefore, the current TLS credentials reloading mechanism cannot be used directly.
2. In some internal use cases, the credentials are stored in certain file locations and updated asynchronously. Application does not need to be notified of when/how the credentials are updated. gRPC core would be responsible for reading credentials from files and reloading the credentials.

These two cases indicate the following design requirements:
1. Support both use cases above. We cannot assume there are bootstrapping key materials. 
2. The credentials reloading C core API needs to be simple, so that it can be easily wrapped by different languages. The credentials reloading C++ API needs to be user-friendly and simple to use. If possible, the API shall avoid complex asynchronous programming (schedule, cancel, mutex, etc.) by the gRPC users.
3. The credentials reloading should be efficient. We shall reload only when needed rather than reloading per handshake.

Based on the requirements, we are going to propose a new distributor/provider API that will meet these requirements. 
The distributor is responsible for caching the credentials and distributing them in each handshake. 
The users will need to specify a particular provider implementation when creating their credentials, and we will provide one implementation for reloading from files.
If that doesn't meet users' requirements, they could implement their own providers.

Besides major credential reloading change, we are also planning to:
1. make the server authorization original designed for client side only applicable to both the client side and the server side. 
This is based on some recent user requirement, and it may also be used as the CRL check since CRL is not officially supported in gRPC. 
2. add the options to configure the min/max TLS version that will be used in handshake

### Related Proposals:

[L46: Create a new TLS credential API that supports SPIFFE mutual TLS](https://github.com/grpc/proposal/pull/98)

## Proposal

### Changes around TLS credentials and Options
This part of the proposal introduces changes made to existing TLS credential APIs. Changes include:
1. Replacing all the occurrence of "*_server_authorization_*" to "*_authorization_"
2. Removing `grpc_tls_key_materials_config`, `grpc_tls_credential_reload_config` and `grpc_tls_credential_reload_arg` 
3. Adding API `grpc_tls_credentials_options_set_certificate_provider`, `grpc_ssl_server_credentials_set_min_tls_version` and `grpc_ssl_server_credentials_set_max_tls_version`

Here is how the API would look like after these changes. For simplicity, all the comments are omitted.

```c
/** --- TLS channel/server credentials ---
 * It is used for experimental purpose for now and subject to change. */

typedef struct grpc_tls_error_details grpc_tls_error_details;

typedef struct grpc_tls_authorization_check_config grpc_tls_authorization_check_config;

typedef struct grpc_tls_credentials_options grpc_tls_credentials_options;

GRPCAPI grpc_tls_credentials_options* grpc_tls_credentials_options_create(void);

GRPCAPI int grpc_tls_credentials_options_set_cert_request_type(
    grpc_tls_credentials_options* options,
    grpc_ssl_client_certificate_request_type type);

GRPCAPI int grpc_tls_credentials_options_set_verification_option(
    grpc_tls_credentials_options* options,
    grpc_tls_verification_option verification_option);

GRPCAPI int grpc_tls_credentials_options_set_authorization_check_config(
    grpc_tls_credentials_options* options,
    grpc_tls_authorization_check_config* config);

/** Sets the credential provider. */
GRPCAPI int grpc_tls_credentials_options_set_certificate_provider(
    grpc_tls_credentials_options* options,
    grpc_tls_certificate_provider* provider);

/** Sets the minimum TLS version that will be negotiated during the TLS
    handshake.  */
GRPCAPI void grpc_ssl_server_credentials_set_min_tls_version(grpc_tls_credentials_options* options,
    grpc_tls_version min_tls_version);

/** Sets the maximum TLS version that will be negotiated during the TLS
    handshake. */
GRPCAPI void grpc_ssl_server_credentials_set_max_tls_version(grpc_tls_credentials_options* options, 
    grpc_tls_version max_tls_version);

typedef struct grpc_tls_authorization_check_arg grpc_tls_authorization_check_arg;

struct grpc_tls_authorization_check_arg {
  /** nothing changed; omitted */
};

GRPCAPI grpc_tls_authorization_check_config*
grpc_tls_authorization_check_config_create(/**  omitted */);

grpc_channel_credentials* grpc_tls_credentials_create(grpc_tls_credentials_options* options);

grpc_server_credentials* grpc_tls_server_credentials_create(grpc_tls_credentials_options* options);
```

### Provider
The provider API offers a general interface for different implementations to interact with the `distributor`. 

`grpc_tls_certificate_provider` is the provider interface we will use in `grpc_tls_credentials_options`.
Callers could directly pass in the already implemented providers, e.g. file-watcher provider, or a custom provider `grpc_tls_certificate_provider_external`, whose function pointer implementations are completely determined by the callers.

```c
/* Opaque type. */
typedef struct grpc_tls_certificate_provider grpc_tls_certificate_provider;
typedef struct grpc_tls_certificate_provider_external grpc_tls_certificate_provider_external;

/* Factory function for file-watcher provider (implemented inside core). */
grpc_tls_certificate_provider* grpc_tls_certificate_provider_file_watcher_create(const char* private_key_file_name, const char* identity_certificate_file_name, const char* root_certificate_file_name);

/* API for creating external provider. */
struct grpc_tls_certificate_provider_external {
  grpc_tls_certificate_distributor* (*get_distributor)(grpc_tls_certificate_provider_external*);
  void (*destroy)(grpc_tls_certificate_provider_external*);
};

/* The ownership of grpc_tls_certificate_provider_external will be taken by grpc_tls_certificate_provider. */
grpc_tls_certificate_provider* grpc_tls_certificate_provider_external_create(
    grpc_tls_certificate_provider_external* external_provider);

```

### Distributor
The distributor is the actual component for caching and distributing the credentials to the underlying transport connections(security connectors).
```c
typedef struct grpc_tls_certificate_distributor grpc_tls_certificate_distributor;

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

## Example Use

We will provide an example showing how to put all these APIs together. Note that users wouldn't be able to interact directly with the core APIs.
Each wrap language needs to wrap these APIs in a way that meets the inherent core logic.

```c
/* Create option and set basic security primitives. */
grpc_tls_credentials_options* options = grpc_tls_credentials_options_create();
grpc_tls_credentials_options_set_verification_option(options, GRPC_TLS_SKIP_HOSTNAME_VERIFICATION);
grpc_tls_authorization_check_config* config = grpc_tls_authorization_check_config_create(.....);
grpc_tls_credentials_options_set_authorization_check_config(options, config);
grpc_ssl_server_credentials_set_min_tls_version(options, TLS1_2);
grpc_ssl_server_credentials_set_max_tls_version(options, TLS1_3);
/* Use the file-based provider for reloading certificates. */
grpc_tls_certificate_provider* file_provider = grpc_tls_certificate_provider_file_watcher_create(...);
grpc_tls_credentials_options_set_certificate_provider(options, file_provider);
/* Create TLS channel credentials. */
grpc_channel_credentials* creds = grpc_tls_credentials_create(options);
```
Wrap language should also allow users to define their own provider implementations:
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
grpc_tls_certificate_provider* provider grpc_tls_certificate_provider_external_create(external_provider);

```

## Wrap Languages
Each wrap language might have their own way to wrap these core APIs, but in general, there are a few things to consider:
1. create an interface-like class `ProviderInterface` which contains two functions GetDistributor() and Destroy()
2. create a concrete class `FileCertificateProvider` that implements `ProviderInterface` and wraps the provider created by `grpc_tls_certificate_provider_file_watcher_create`
3. create an interface-like class `CertificateProvider` that implements `ProviderInterface`, and exposes several APIs in the distributor to users, such as `grpc_tls_certificate_distributor_set_key_materials`, etc
4. users could extend `CertificateProvider` to define their own provider implementations

For point 3 and 4, here is an example of `CertificateProvider` in C++:
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
and an example of custom provider implementation:
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

