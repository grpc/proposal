L46: C-core: New TLS Credentials API
----
* Author(s): gtcooke94, ZhenLian
* Approver: 
* Status: Draft
* Implemented in:
* Last updated: May 16, 2024

## Abstract

This proposal aims to provide the following features in a new TLS credential API:
1. credential reloading: reload transport credentials periodically from various sources, e.g. from credential files, or users' customized Provider implementations, without shutting down
1. custom verification behaviors: performing a user-specified check at the end of normal authentication and separately giving the ability to fully customize chain building and verification.
1. TLS version configuration: optionally configure the minimum and maximum of the TLS version that will be used in the handshake
1. support for certificate revocation via Certificate Revocation Lists (CRLs)

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

  // Sets a custom chain builder implementation that replaces the default chain building from the underlying SSL library.
  void set_custom_chain_builder(std::shared_ptr<CustomChainBuilder> chain_builder);
```

Here is an example of how to use this API:
```c++
/* Create option and set basic security primitives. */
TlsCredentialsOptions options = TlsCredentialsOptions();
options.set_verify_server_cert(true);
options.set_min_tls_version(TLS1_2);
options.set_max_tls_version(TLS1_3);

// The core credentials APIs are still C, so this is still a grpc_channel_credential
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


/** ------------------------------------ Provider ------------------------------------ */
/** Interface for a class that handles the process to fetch credential data.
*/
class CertificateProviderInterface {
 public:
  virtual ~CertificateProviderInterface() = default;
  // Provider implementations MUST provide a WatchStatusCallback that will be
  // called by the internal stack. This will be invoked when a new certificate
  // name is watched by a newly registered watcher, or when a certificate name
  // is no longer watched by any watchers.  Note that when the callback shows a
  // cert is no longer being watched, corresponding certificate data will be
  // deleted from the internal cache, and any corresponding errors will be
  // cleared, if there are any. This means that if the callback subsequently
  // says the same cert is now being watched again, the provider must re-provide
  // the credentials or re-invoke the errors to indicate a successful or failed
  // reloading.
  //
  // For the parameters in the callback function:
  // cert_name The name of the certificates being watched.
  // root_being_watched If the root certificates with the specific name are being
  // watched.
  // identity_being-Watched If the identity certificates with the specific name
  // are being watched.
  virtual void WatchStatusCallback(string cert_name, bool root_being_watched, bool identity_being_watched) = 0;
  // Sets the key materials based on their certificate name.
  // @param cert_name The name of the certificates being updated.
  // @param pem_root_certs The content of root certificates.
  // @param pem_key_cert_pairs The content of identity key-cert pairs.
  void SetKeyMaterials(
      const std::string& cert_name, absl::optional<std::string> pem_root_certs,
      absl::optional<grpc_core::PemKeyCertPairList> pem_key_cert_pairs);
  // Propagates an error encountered in the provider layer to the internal TLS stack.
  //
  // @param cert_name The watching cert name that the caller
  // wants to notify about when encountering error.
  // @param root_cert_error The error that the caller encounters when reloading
  // root certs.
  // @param identity_cert_error The error that the caller encounters when
  // reloading identity certs.
  void SetErrorForCert(const std::string& cert_name,
                       absl::Status root_cert_error,
                       absl::Status identity_cert_error);

 private:
  std::unique_ptr<TlsCertificateDistributor> distributor_;
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


```
### Custom Verification
We aim to provide distinct interfaces for chain building customization and for additional validation of an already built chain. These two features have distinctly different expected users. Doing some extra validation based on the contents of the peer certificate chain is an operation that anyone doing (m)TLS might be interested in, and it should not and does not require any deep X.509 expertise. Replacing how chain building works is a relatively rare requirement and it does require deep X.509 expertise to do correctly. We want extra validation to be easy and clear for users to add without worrying about interacting with chain building itself. On the other hand, chain building is complex to implement, and we suspect that most users will stay with the default behavior. However, to enable advanced use, such as dynamic selection of the trust bundle based on the peer certificate (e.g. SPIFFE federation), this is needed. 
#### Post Handshake Verification
```c++
// Contains the verification-related information associated with a connection
// request. Users should not directly create or destroy this request object, but
// shall interact with it through CertificateVerifier's Verify() and Cancel().
class TlsCustomVerificationCheckRequest {
 public:
  explicit TlsCustomVerificationCheckRequest(
      grpc_tls_custom_verification_check_request* request);
  ~TlsCustomVerificationCheckRequest() {}

  absl::string_view target_name() const;
  absl::string_view peer_cert() const;
  absl::string_view peer_cert_full_chain() const;
  absl::string_view common_name() const;
  // The subject name of the root certificate used to verify the peer chain
  // If verification fails or the peer cert is self-signed, this will be an
  // empty string. If verification is successful, it is a comma-separated list,
  // where the entries are of the form "FIELD_ABBREVIATION=string"
  // ex: "CN=testca,O=Internet Widgits Pty Ltd,ST=Some-State,C=AU"
  // ex: "CN=GTS Root R1,O=Google Trust Services LLC,C=US"
  absl::string_view verified_root_cert_subject() const;
  std::vector<absl::string_view> uri_names() const;
  std::vector<absl::string_view> dns_names() const;
  std::vector<absl::string_view> email_names() const;
  std::vector<absl::string_view> ip_names() const;

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

#### Custom Chain Building
```c++
class CustomChainBuilder {
public:
 virtual absl::StatusOr<std::vector<absl::string_view>> BuildAndVerifyChain(const std::vector<absl::string_view>& peer_cert_chain_der) = 0;
}
```

The default behavior for chain building is based on the underlying SSL library. For example, [X509_verify_cert](https://www.openssl.org/docs/man1.1.1/man3/X509_verify_cert.html) is the implementation in OpenSSL that builds and verifies a certificate chain.
gRPC calls that function by default [here](https://github.com/grpc/grpc/blob/2d2d5a3c411a2bade319a08085e55821cf2d5ed9/src/core/tsi/ssl_transport_security.cc#L1150).
Once the `CustomChainBuilder` is implemented, this will be where it is used. The `X509_verify_cert` call will be replaced by the `CustomChainBuilder's BuildAndVerifyChain` that fully replaces chain building.