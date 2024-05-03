L46: C-core: New TLS Credentials API
----
* Author(s): gtcooke94, ZhenLian
* Approver: 
* Status: Draft
* Implemented in:
* Last updated: March 22, 2024

## Abstract

This proposal aims to provide the following features in a new TLS credential API:
1. credential reloading: reload transport credentials periodically from various sources, e.g. from credential files, or users' customized Provider implementations, without shutting down
1. custom verification check: perform a user-specified check at the end of authentication, and allow the client side to disable certificate verification and hostname verification check    
1. TLS version configuration: optionally configure the minimum and maximum of the TLS version that will be used in the handshake

## Background

The current TLS implementation in gRPC C core has some restrictions:
1. lack of the credential reloading support. Once an application starts, it can never reload its identity key/cert-chain and root certificates without shutting down
1. inflexibility in the peer verification. In a mutual TLS scenario, servers would always perform certificate verification, and clients would always perform certificate verification as well as default hostname check. No customized checks can be introduced.
1. inability to choose the maximum/minimum TLS version supported in OpenSSL/BoringSSL for use

There have always been some ad hoc requests on GitHub Issue Page for supporting these features. The demands for a more flexible and configurable API have increased with the emerging needs in the cloud platform.

One common requirement is to skip the hostname verification check if a client has a different way of validating the server's identity in different platforms. 
For example, it's becoming more and more popular to use SPIFFE ID as the identity format, but a valid certificate using SPIFFE ID doesn't necessarily contain the dnsName in the SAN field, which might cause hostname verification to fail.

Another requirement is to reload the root and identity credentials without shutting down the server, because certificates will eventually expire and the ability to rotate the certificates is essential to a cloud application.

Besides requests from Open Source Community, with the growth of GCP, we've seen more and more internal use cases of these features.   

Hence we propose an API that will meet the following requirements:  

1. Flexible enough to support multiple use cases. Users should be able to write their own reloading or verification logic if the provided ones doesn't fit their needs
1. The credentials reloading should be efficient. We shall reload only when needed rather than reloading per TLS connection
1. C core API needs to be simple enough, so that most of existing SSL credentials in wrapped languages can migrate to call this new API

### Related Proposals:

1. [L46: Create a new TLS credential API that supports SPIFFE mutual TLS](https://github.com/grpc/proposal/pull/98) - this was subsumed into this proposal.
1. [A69: Crl Providers](https://github.com/grpc/proposal/pull/382)

## Proposal

### TLS credentials and TLS Options
This part of the proposal introduces `TlsCredentialsOptions` and TLS credentials. Users use `TlsCredentialsOptions` to configure the specific security properties they want, and build the client/server credential by passing in `TlsCredentialsOptions`.
This section only covers the configuration not related to credential reloading and custom verification. Configurations related to those two topics are explained in the latter sections.

```c++
// Base class of configurable options specified by users to configure their
// certain security features supported in TLS. It is used for experimental
// purposes for now and it is subject to change.
class TlsCredentialsOptions {
 public:
  // Constructor for base class TlsCredentialsOptions.
  //
  // @param certificate_provider the provider which fetches TLS credentials that
  // will be used in the TLS handshake
  TlsCredentialsOptions();
  ~TlsCredentialsOptions();

  // Copy constructor does a deep copy of the underlying pointer. No assignment
  // permitted
  TlsCredentialsOptions(const TlsCredentialsOptions& other);
  TlsCredentialsOptions& operator=(const TlsCredentialsOptions& other) = delete;

  // ---- Setters for member fields ----
  // Sets the certificate provider used to store root certs and identity certs.
  void set_certificate_provider(
      std::shared_ptr<CertificateProviderInterface> certificate_provider);
  // Watches the updates of root certificates with name |root_cert_name|.
  // If used in TLS credentials, setting this field is optional for both the
  // client side and the server side.
  // If this is not set on the client side, we will use the root certificates
  // stored in the default system location, since client side must provide root
  // certificates in TLS(no matter single-side TLS or mutual TLS).
  // If this is not set on the server side, we will not watch any root
  // certificate updates, and assume no root certificates needed for the server
  // (in the one-side TLS scenario, the server is not required to provide root
  // certs). We don't support default root certs on server side.
  void watch_root_certs();
  // Sets the name of root certificates being watched, if |watch_root_certs| is
  // called. If not set, an empty string will be used as the name.
  //
  // @param root_cert_name the name of root certs being set.
  void set_root_cert_name(const std::string& root_cert_name);
  // Watches the updates of identity key-cert pairs with name
  // |identity_cert_name|. If used in TLS credentials, it is required to be set
  // on the server side, and optional for the client side(in the one-side
  // TLS scenario, the client is not required to provide identity certs).
  void watch_identity_key_cert_pairs();
  // Sets the name of identity key-cert pairs being watched, if
  // |watch_identity_key_cert_pairs| is called. If not set, an empty string will
  // be used as the name.
  //
  // @param identity_cert_name the name of identity key-cert pairs being set.
  void set_identity_cert_name(const std::string& identity_cert_name);
  // Sets the Tls session key logging configuration. If not set, tls
  // session key logging is disabled. Note that this should be used only for
  // debugging purposes. It should never be used in a production environment
  // due to security concerns.
  //
  // @param tls_session_key_log_file_path: Path where tls session keys would
  // be logged.
  void set_tls_session_key_log_file_path(
      const std::string& tls_session_key_log_file_path);
  // Sets the certificate verifier used to perform post-handshake peer identity
  // checks.
  void set_certificate_verifier(
      std::shared_ptr<CertificateVerifier> certificate_verifier);

  // Sets the crl provider, see CrlProvider for more details.
  void set_crl_provider(std::shared_ptr<CrlProvider> crl_provider);

  // Sets the minimum TLS version that will be negotiated during the TLS
  // handshake. If not set, the underlying SSL library will use TLS v1.2.
  // @param tls_version: The minimum TLS version.
  void set_min_tls_version(grpc_tls_version tls_version);
  // Sets the maximum TLS version that will be negotiated during the TLS
  // handshake. If not set, the underlying SSL library will use TLS v1.3.
  // @param tls_version: The maximum TLS version.
  void set_max_tls_version(grpc_tls_version tls_version);
```

Here is an example of how to use this API:
```c++
/* Create option and set basic security primitives. */
TlsCredentialsOptions options = TlsCredentialsOptions();
options.set_verify_server_cert(true);
options.set_min_tls_version(TLS1_2);
options.set_max_tls_version(TLS1_3);

//TODO(gtcooke94 C++ify)
/* Create TLS channel credentials. */
grpc_channel_credentials* creds = grpc_tls_credentials_create(options);
```

### Credential Reloading
The credential reloading API basically consists of two parts: the top-level `Provider` and the low-level `Distributor`.
The `Distributor` is the actual component for caching and distributing the credentials to the underlying transport connections(security connectors).
The `Provider` offers a general interface for different implementations to interact with the `Distributor`. 
```c
/* Opaque types. */
// A struct that stores the credential data presented to the peer in handshake
// to show local identity. The private_key and certificate_chain should always
// match.
struct GRPCXX_DLL IdentityKeyCertPair {
  std::string private_key;
  std::string certificate_chain;
};

/**
 * Sets the name of the root certificates being used in the distributor. 
 * Most users don't need to set this value.
 * If not set, we will use the default name, which is an empty string.
 */
GRPCAPI void grpc_tls_credentials_options_set_root_cert_name(
    grpc_tls_credentials_options* options,
    const char* root_cert_name);

/**
 * Sets the name of the identity certificates being used in the distributor. 
 * Most users don't need to set this value.
 * If not set, we will use the default name, which is an empty string.
 */
GRPCAPI void grpc_tls_credentials_options_set_identity_cert_name(
    grpc_tls_credentials_options* options,
    const char* identity_cert_name);

/** Sets the credential provider.
 * Sets the credential provider in the options.
 * The |options| will implicitly take a new ref to the |provider|.
 */
GRPCAPI void grpc_tls_credentials_options_set_certificate_provider(
    grpc_tls_credentials_options* options,
    grpc_tls_certificate_provider* provider);

/**
 * If set, gRPC stack will keep watching the root certificates with
 * name |root_cert_name|.
 * If this is not set on the client side, we will use the root certificates
 * stored in the default system location, since client side must provide root
 * certificates in TLS.
 * If this is not set on the server side, we will not watch any root certificate
 * updates, and assume no root certificates needed for the server(single-side
 * TLS). Default root certs on the server side is not supported.
 */
GRPCAPI void grpc_tls_credentials_options_watch_root_certs(
    grpc_tls_credentials_options* options);

/**
 * If set, gRPC stack will keep watching the identity key-cert pairs
 * with name |identity_cert_name|.
 * This is required on the server side, and optional on the client side.
 */
GRPCAPI void grpc_tls_credentials_options_watch_identity_key_cert_pairs(
    grpc_tls_credentials_options* options);

/** ------------------------------------ Provider ------------------------------------ */
/** Interface for a class that handles the process to fetch credential data.
* Implementations should be a wrapper class of an internal provider
* implementation.
*/
class CertificateProviderInterface {
 public:
  virtual ~CertificateProviderInterface() = default;
};

// A basic CertificateProviderInterface implementation that will load credential
// data from static string during initialization. This provider will always
// return the same cert data for all cert names, and reloading is not supported.
class StaticDataCertificateProvider
    : public CertificateProviderInterface {
 public:
  StaticDataCertificateProvider(
      const std::string& root_certificate,
      const std::vector<IdentityKeyCertPair>& identity_key_cert_pairs);

  explicit StaticDataCertificateProvider(const std::string& root_certificate);

  explicit StaticDataCertificateProvider(
      const std::vector<IdentityKeyCertPair>& identity_key_cert_pairs);
    }

// A CertificateProviderInterface implementation that will watch the credential
// changes on the file system. This provider will always return the up-to-date
// cert data for all the cert names callers set through |TlsCredentialsOptions|.
// Several things to note:
// 1. This API only supports one key-cert file and hence one set of identity
// key-cert pair, so SNI(Server Name Indication) is not supported.
// 2. The private key and identity certificate should always match. This API
// guarantees atomic read, and it is the callers' responsibility to do atomic
// updates. There are many ways to atomically update the key and certs in the
// file system. To name a few:
//   1)  creating a new directory, renaming the old directory to a new name, and
//   then renaming the new directory to the original name of the old directory.
//   2)  using a symlink for the directory. When need to change, put new
//   credential data in a new directory, and change symlink.
class GRPCXX_DLL FileWatcherCertificateProvider final
    : public CertificateProviderInterface {
 public:
  // Constructor to get credential updates from root and identity file paths.
  //
  // @param private_key_path is the file path of the private key.
  // @param identity_certificate_path is the file path of the identity
  // certificate chain.
  // @param root_cert_path is the file path to the root certificate bundle.
  // @param refresh_interval_sec is the refreshing interval that we will check
  // the files for updates.
  FileWatcherCertificateProvider(const std::string& private_key_path,
                                 const std::string& identity_certificate_path,
                                 const std::string& root_cert_path,
                                 unsigned int refresh_interval_sec);
  // Constructor to get credential updates from identity file paths only.
  FileWatcherCertificateProvider(const std::string& private_key_path,
                                 const std::string& identity_certificate_path,
                                 unsigned int refresh_interval_sec);

  // Constructor to get credential updates from root file path only.
  FileWatcherCertificateProvider(const std::string& root_cert_path,
                                 unsigned int refresh_interval_sec);
    }

// TODO(gtcooke94) C++ify 
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
In that case, how to wrap it should be straightforward. 

Each wrap language is free to choose the design suitable for its language characteristics, but consider some general advice first:
1. create an interface-like class `ProviderInterface` which contains two functions GetDistributor() and Destroy()
2. create a concrete class `FileCertificateProvider` that implements `ProviderInterface` and wraps the provider created by `grpc_tls_certificate_provider_file_watcher_create`
3. create a concrete class `StaticCertificateProvider` that implements `ProviderInterface` and wraps the provider created by `grpc_tls_certificate_provider_file_static_create`
4. create an interface-like class `CertificateProvider` that implements `ProviderInterface`, and exposes several APIs in the distributor to users, such as `grpc_tls_certificate_distributor_set_key_materials`, etc
5. users could extend `CertificateProvider` to define their own provider implementations

// TODO(gtcooke94) we can directly use C++ APIs, remove external stuff from the below definitions, and C++ will just alias the C-core

An example of custom provider implementation in C++:
// TODO(gtcooke94) precise semantics
```cpp
class MyCertificateProvider: public grpc::CertificateProviderInterface {
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
### Custom Verification
```c++
// Contains the verification-related information associated with a connection
// request. Users should not directly create or destroy this request object, but
// shall interact with it through CertificateVerifier's Verify() and Cancel().
class TlsCustomVerificationCheckRequest {
 public:
  explicit TlsCustomVerificationCheckRequest(
      grpc_tls_custom_verification_check_request* request);
  ~TlsCustomVerificationCheckRequest() {}

  grpc::string_ref target_name() const;
  grpc::string_ref peer_cert() const;
  grpc::string_ref peer_cert_full_chain() const;
  grpc::string_ref common_name() const;
  // The subject name of the root certificate used to verify the peer chain
  // If verification fails or the peer cert is self-signed, this will be an
  // empty string. If verification is successful, it is a comma-separated list,
  // where the entries are of the form "FIELD_ABBREVIATION=string"
  // ex: "CN=testca,O=Internet Widgits Pty Ltd,ST=Some-State,C=AU"
  // ex: "CN=GTS Root R1,O=Google Trust Services LLC,C=US"
  grpc::string_ref verified_root_cert_subject() const;
  std::vector<grpc::string_ref> uri_names() const;
  std::vector<grpc::string_ref> dns_names() const;
  std::vector<grpc::string_ref> email_names() const;
  std::vector<grpc::string_ref> ip_names() const;

}

// The base class of all internal verifier implementations, and the ultimate
// class that all external verifiers will eventually be transformed into.
// To implement a custom verifier, do not extend this class; instead,
// implement a subclass of ExternalCertificateVerifier. Note that custom
// verifier implementations can compose their functionality with existing
// implementations of this interface, such as HostnameVerifier, by delegating
// to an instance of that class.
class CertificateVerifier {
 public:
  ~CertificateVerifier();

  // Verifies a connection request. The check on each internal verifier could be
  // either synchronous or asynchronous, and we will need to use return value to
  // know.
  //
  // request: the verification information associated with this request
  // callback: This will only take effect if the verifier is asynchronous.
  //           The function that gRPC will invoke when the verifier has already
  //           completed its asynchronous check. Callers can use this function
  //           to perform any additional checks. The input parameter of the
  //           std::function indicates the status of the verifier check.
  // sync_status: This will only be useful if the verifier is synchronous.
  //              The status of the verifier as it has already done it's
  //              synchronous check.
  // return: return true if executed synchronously, otherwise return false
  bool Verify(TlsCustomVerificationCheckRequest* request,
              std::function<void(grpc::Status)> callback,
              grpc::Status* sync_status);

  // Cancels a verification request previously started via Verify().
  // Used when the connection attempt times out or is cancelled while an async
  // verification request is pending.
  //
  // request: the verification information associated with this request
  void Cancel(TlsCustomVerificationCheckRequest* request);
};

// A CertificateVerifier that doesn't perform any additional checks other than
// certificate verification, if specified.
// Note: using this solely without any other authentication mechanisms on the
// peer identity will leave your applications to the MITM(Man-In-The-Middle)
// attacks. Users should avoid doing so in production environments.
class NoOpCertificateVerifier : public CertificateVerifier {
 public:
  NoOpCertificateVerifier();
};

// A CertificateVerifier that will perform hostname verification, to see if the
// target name set from the client side matches the identity information
// specified on the server's certificate.
class HostNameCertificateVerifier : public CertificateVerifier {
 public:
  HostNameCertificateVerifier();
};
```





