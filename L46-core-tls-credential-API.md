L46: C-core: New TLS Credentials API
----
* Author(s): gtcooke94, ZhenLian
* Approver: 
* Status: Draft
* Implemented in:
* Last updated: May 16, 2024

## Abstract

This proposal aims to provide a new TLS credentials API with the following features:
1. Credential reloading: reload transport credentials periodically from various sources, e.g. from credential files, or users' customized Provider implementations, without restarting the process.
1. Customizable verification of the peer certificate: perform custom validation checks on the peer's certificate chain after it has been verified to chain up to a root of trust, or fully customize path building and cryptographic verification.
1. TLS version configuration: optionally configure the minimum and maximum of the TLS versions to be negotiated during the handshake.
1. Support for certificate revocation via Certificate Revocation Lists (CRLs).

## Background

The current TLS implementation (the SSL credentials API) in gRPC C core has some restrictions:
1. Lack of the credential reloading support. Once an application starts, it can never reload its private key, certificate chain conveying the application's identity, and root certificates without shutting down.
1. Inflexibility of how the peer certificate chain is verified. There is no API to add additional custom verification checks.
1. Inability to choose the maximum/minimum TLS version to be negotiated during the TLS handshake.
1. No support for certificate revocation.

We propose an API that supports the above features and meets the following requirements:  

1. Flexible enough to support multiple use cases. Users should be able to write their own reloading or verification logic if the provided ones do not fit their needs.
1. The credential reloading should be efficient. We shall reload only when needed rather than reloading per TLS connection.
1. C core API needs to be simple enough, so that most of existing SSL credentials in wrapped languages can migrate to call this new API.

### Related Proposals:

1. [L46: Create a new TLS credential API that supports SPIFFE mutual TLS](https://github.com/grpc/proposal/pull/98) - this was subsumed into this proposal.
1. [A69: Crl Providers](https://github.com/grpc/proposal/pull/382)

## Proposal

### TLS credentials and TLS Options
This part of the proposal introduces the base options class, called `TlsCredentialsOptions`, that has a large set of options for configuring TLS connections. There are 2 derived classes of `TlsCredentialsOptions`, called `TlsChannelCredentialsOptions` and `TlsServerCredentialsOptions`, that users create and can use to build channel credentials or server credentials instances, respectively. The API for building these channel/server credentials instances are called `TlsCredentials`/`TlsServerCredentials` respectively.

```c++
// Base class of TLS options.
class TlsCredentialsOptions {
 public:
  ~TlsCredentialsOptions();

  // Copy constructor does a deep copy. No assignment permitted.
  TlsCredentialsOptions(const TlsCredentialsOptions& other);

  TlsCredentialsOptions& operator=(const TlsCredentialsOptions& other) = delete;

  // ---- Setters for member fields ----
  // Sets the certificate provider which is used to store: 
  // 1. The root certificates used to (cryptographically) verify peer certificate chains.
  // 2. The certificate chain conveying the application's identity and the corresponding private key.
  void set_certificate_provider(
      std::shared_ptr<CertificateProviderInterface> certificate_provider);

  // Watches the updates of root certificates with name |root_cert_name| (set by
  // the user via set_root_cert_name).  If used in TLS credentials, setting this
  // field is optional for both the client side and the server side.
  // If this is not set on the client side, we will use the root certificates
  // stored in the default system location, since client side must provide root
  // certificates in TLS (no matter single-side TLS or mutual TLS).
  // If this is not set on the server side, we will not watch any root
  // certificate updates, and assume no root certificates needed for the server
  // (in the one-side TLS scenario, the server is not required to provide root
  // certificates). We don't support default root certs on server side.
  void watch_root_certificates();

  // Sets the name of root certificates being watched, if |watch_root_certificates| is
  // called. If not set, an empty string will be used as the name. For simple
  // use cases this is not needed, but a certificate provider could be
  // implemented that watches many sets of certificates. This allows the user to
  // differentiate between different root certificates.
  // 
  // @param root_cert_name the name of root certificates being set.
  void set_root_cert_name(const std::string& root_cert_name);

  // Watches the updates of identity key-certificate pairs with name
  // |identity_cert_name|. If used in TLS credentials, it is required to be set
  // on the server side, and optional for the client side(in the one-side
  // TLS scenario, the client is not required to provide identity certificates).
  void watch_identity_key_cert_pairs();

  // Sets the name of identity key-certificate pairs being watched, if
  // |watch_identity_key_cert_pairs| is called. If not set, an empty string will
  // be used as the name.
  //
  // @param identity_cert_name the name of identity key-certificate pairs being set.
  void set_identity_cert_name(const std::string& identity_cert_name);

  // WARNING: EXPERT USE ONLY. MISUSE CAN LEAD TO SIGNIFICANT SECURITY DEGRADATION.
  // 
  // Sets the TLS session key logging file path. If not set, TLS
  // session key logging is disabled. Note that this should be used only for
  // debugging purposes. It should never be used in a production environment
  // due to security concerns - anyone who can obtain the (logged) session key
  // can decrypt all traffic on a connection.
  // 
  // @param tls_session_key_log_file_path: Path where TLS session keys should
  // be logged.
  void set_tls_session_key_log_file_path(
      const std::string& tls_session_key_log_file_path);

  // Sets the certificate verifier. The certificate verifier performs checks on
  // the peer certificate chain after the chain has been (cryptographically)
  // verified to chain up to a trusted root.
  void set_certificate_verifier(
      std::shared_ptr<CertificateVerifier> certificate_verifier);

  // Sets the crl provider, see CrlProvider for more details.
  void set_crl_provider(std::shared_ptr<CrlProvider> crl_provider);

  // Sets the minimum TLS version that will be negotiated during the TLS
  // handshake. If not set, the underlying SSL library will default to TLS v1.2.
  // @param tls_version: The minimum TLS version.
  void set_min_tls_version(grpc_tls_version tls_version);

  // Sets the maximum TLS version that will be negotiated during the TLS
  // handshake. If not set, the underlying SSL library will default to TLS v1.3.
  // @param tls_version: The maximum TLS version.
  void set_max_tls_version(grpc_tls_version tls_version);

  // WARNING: EXPERT USE ONLY. MISUSE CAN LEAD TO SIGNIFICANT SECURITY DEGRADATION.
  //
  // Sets a custom chain builder implementation that replaces the default chain
  // building from the underlying SSL library. Fully replacing and implementing
  // chain building is a complex task and has dangerous security implications if
  // done wrong, thus this API is inteded for expert use only.
  void set_custom_chain_builder(std::shared_ptr<CustomChainBuilder>
  chain_builder);
}

// Server-specific options for configuring TLS.
class TlsServerCredentialsOptions final : public TlsCredentialsOptions {
 public:
  // A certificate provider that provides identity credentials is required,
  // because a server must always present identity credentials during any TLS
  // handshake. The certificate provider may optionally provide root
  // certificates, in case the server requests client certificates.
  explicit TlsServerCredentialsOptions(
      std::shared_ptr<CertificateProviderInterface> certificate_provider)
      : TlsCredentialsOptions() {
    set_certificate_provider(certificate_provider);
  }


  // Sets requirements for if client certificates are requested, if they
  // are required, and if the client certificate must be trusted. The
  // default is GRPC_SSL_DONT_REQUEST_CLIENT_CERTIFICATE, which represents
  // normal TLS.
  void set_cert_request_type(
      grpc_ssl_client_certificate_request_type cert_request_type);
}

// Client-specific options for configuring TLS.
//
// A client may optionally set a certificate provider.
// If there is no certificate provider, the system default root certificates will be used to
// verify server certificates. 
// If a certificate provider is set and it provides root certifices that root
// will be used.
// If a certificate provider is set and it provides identity credentials those
// identity credentials will be used.
class TlsChannelCredentialsOptions final : public TlsCredentialsOptions {
 public:
  // Sets the decision of whether to do a crypto check on the server certificates.
  // The default is true.
  void set_verify_server_certificates(bool verify_server_certs);
}

// Builds a ChannelCredentials instance that establishes TLS connections in the manner specified by options.
std::shared_ptr<ChannelCredentials> TlsCredentials(
    const TlsChannelCredentialsOptions& options);
}

// Builds a ServerCredentials instance that establishes TLS connections in the manner specified by options.
std::shared_ptr<ServerCredentials> TlsServerCredentials(
    const TlsServerCredentialsOptions& options);
```

To make migration from the `SslCredentials` stack to the `TlsCredentials` stack easier,
we will also have a constructor for `TlsChannelCredentialsOptions` and
`TlsServerCredentialsOptions` that accept their corresponding `SslOptions`
structs.

### Credential Reloading
Credential reloading will be implemented via a `CertificateProviderInterface`.
Note that there is a naming mismatch here - there exist credentials beyond
certificates that this _could_ theoretically support.  This provider is
responsible for sourcing key material and supplying it to the
internal stack via the `CertificateProvider`'s `SetKeyMaterials` API.  Two
reference implementations of the `CertificateProviderInterface` are provided -
`StaticDataCertificateProvider` and `FileWatcherCertificateProvider`. The
`StaticDataCertificateProvider` provides certificates from raw string data. The
`FileWatcherCertificateProvider` will periodically and asynchronously refresh
certificate data from a file, allowing changes to certificates without
restarting the process. More details about these reference implementations
are provided below. These implementations should cover many use cases.  If these
providers do not support the intricacies for a specific use case, a user can
provide their own implementations of the `CertificateProviderInterface`.

```c
/* Opaque types. */
// A struct that stores the credential data presented to the peer in handshake
// to show local identity. The private key and certificate chain must be PEM
// encoded and the public key in the leaf certificate must correspond to the
// given private key.
struct IdentityKeyCertPair {
  std::string private_key_pem;
  std::string certificate_chain_pem;
};


/** ------------------------------------ Provider ------------------------------------ */
/** Provides identity credentials and root certificates.
*/
class CertificateProviderInterface {
 public:
  virtual ~CertificateProviderInterface() = default;

  // Provider implementations MUST provide a WatchStatusCallback that will be
  // called by the internal stack. This will be invoked when a new certificate
  // name is starting to be used internally, or when a certificate name is no
  // longer being used internally. When being invoked for a new certificate,
  // this callback should call `SetKeyMaterials` to do the initial population
  // the certificate data in the internal stack.
  // 
  // For the parameters in the callback function:
  // cert_name The name of the certificates being watched.
  // root_being_watched If the root certificates with the specific name are being
  // watched.
  // identity_being-Watched If the identity certificates with the specific name
  // are being watched.
  virtual void WatchStatusCallback(std::string cert_name, bool root_being_watched, bool identity_being_watched) = 0;

  // Sets the key materials based on their certificate name.
  // @param cert_name The name of the certificates being updated.
  // @param pem_root_certificates The content of root certificates.
  // @param pem_key_cert_pairs The content of identity key-certificate pairs.
  void SetKeyMaterials(
      const std::string& cert_name, absl::optional<std::string> pem_root_certificates,
      absl::optional<grpc_core::PemKeyCertPairList> pem_key_cert_pairs);

  // Propagates an error encountered in the provider layer to the internal TLS stack.
  //
  // @param cert_name The watching certificate name that the caller
  // wants to notify about when encountering error.
  // @param root_cert_error The error that the caller encounters when reloading
  // root certificates.
  // @param identity_cert_error The error that the caller encounters when
  // reloading identity certificates.
  void SetErrorForCert(const std::string& cert_name,
                       absl::Status root_cert_error,
                       absl::Status identity_cert_error);

 private:
  std::unique_ptr<TlsCertificateDistributor> distributor_;
};

/** ------------------------------------ Provider ------------------------------------ */
/** Provides identity credentials and root certificates.
*/
// This represents a potential decoupling of roots and identity chains, with
// further extension points for something like SPIFFE bundles
class CertificateProviderInterface {
 public:
  virtual ~CertificateProviderInterface() = default;
  // Provider implementations MUST provide a LoadAndSetCredentialsCallback that
  // will be called by the internal stack. This will be invoked when a new
  // certificate name is starting to be used internally, or when a certificate
  // name is no longer being used internally. When being invoked for a new
  // certificate, this callback should call SetRootCertificates or
  // SetIdentityChainAndPrivateKey to do the initial population the certificate
  // data in the internal stack.
  // 
  // For the parameters in the callback function: cert_name The name of the
  // certificates being watched.  type The type of certificates being watched.
  virtual void LoadAndSetCredentialsCallback(std::string name, TODOType type) = 0;

  // Must be called after the constructor.
  // Does important internal setup steps.
  virtual void Init() final;

  enum TODOType {
    RootCertificates,
    IdentityChainAndPrivateKey
  }

protected:
  // Sets the root certificates based on their name.
  virtual void SetRootCertificates(const std::string& name, std::string pem_root_certificates) final;

  // Sets the identity chain and private key based on their name.
  virtual void SetIdentityChainAndPrivateKey(const std::string& name, grpc_core::PemKeyCertPairList pem_key_cert_pairs) final;

  // Propagates an error encountered in the provider layer to the internal TLS stack.
  virtual void SetError(const std::string& name, TODOType type, absl::Status error) final;

 private:
  std::unique_ptr<TlsCertificateDistributor> distributor_;
};

// A basic CertificateProviderInterface implementation that will load credential
// data from static string during initialization. This provider will always
// return the same certificate data for all cert names, and reloading is not supported.
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
// certificate data for all the cert names callers set through |TlsCredentialsOptions|.
// Several things to note:
// 1. This API only supports one key-certificate file and hence one set of identity
// key-certificate pair, so SNI(Server Name Indication) is not supported.
// 2. The private key and identity certificate should always match. This API
// guarantees atomic read, and it is the callers' responsibility to do atomic
// updates. There are many ways to atomically update the key and certificates in the
// file system. To name a few:
//   1)  creating a new directory, renaming the old directory to a new name, and
//   then renaming the new directory to the original name of the old directory.
//   2)  using a symlink for the directory. When need to change, put new
//   credential data in a new directory, and change symlink.
class FileWatcherCertificateProvider final
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
### Customizable Verification of the Peer Certificate

Users should be able to customize verification of the peer certificate, and we provide two distinct interfaces for how this can be done:

1. Allow users to perform custom validation checks on the peer's certificate chain after it has been verified to chain up to a root of trust, but before the TLS handshake completes. This is a common customization that might be needed whenever a private PKI is involved, and does not require the user to have any significant domain knowledge.

2. Allow users to fully customize path building and cryptographic verification. This is a relatively rare requirement from users and requires deep X.509 expertise to do correctly and without introducing significant security vulnerabilities. However, it is unavoidable in order to unblock some advanced use cases, e.g. SPIFFE federation.

The next sections explain these interfaces in more detail.

#### Custom Validation Checks on the Peer Certificate Chain
```c++
// Contains the information from the (verified) peer certificate chain that can
// be used to perform custom validation checks. Users should not directly
// create or destroy this request object, but shall interact with it through
// CertificateVerifier's Verify() and Cancel().
// TODO(gtcooke94) Add expected
// formats for all string values here.
class TlsCustomVerificationCheckRequest {
 public:
  // The target hostname that is expected to appear in the server leaf
  // certicate, e.g. github.com. Empty on the server-side.
  absl::string_view target_name() const;

  // The PEM-encoded leaf cert.
  absl::string_view peer_cert() const;

  // The PEM-encoded unverified peer cert chain, ordered from leaf to root, but
  // not including the root.
  absl::string_view peer_cert_full_chain() const;

  // The common name string from the Subject extension in the peer's leaf
  // certificate.
  absl::string_view common_name() const;

  // The subject name of the root certificate used to verify the peer chain
  // If verification fails or the peer certificate is self-signed, this will be an
  // empty string. If verification is successful, it is a comma-separated list,
  // where the entries are of the form "FIELD_ABBREVIATION=string"
  // ex: "CN=testca,O=Internet Widgits Pty Ltd,ST=Some-State,C=AU"
  // ex: "CN=GTS Root R1,O=Google Trust Services LLC,C=US"
  absl::string_view verified_root_cert_subject() const;

  // The (possibly empty) list of URI names from the Subject Alternative Name
  // extension in the peer's leaf certificate.
  std::vector<absl::string_view> uri_names() const;

  // The (possibly empty) list of DNS names from the Subject Alternative Name
  // extension in the peer's leaf certificate.
  std::vector<absl::string_view> dns_names() const;

  // The (possibly empty) list of email names from the Subject Alternative Name
  // extension in the peer's leaf certificate.
  std::vector<absl::string_view> email_names() const;

  // The (possibly empty) list of IP names from the Subject Alternative Name
  // extension in the peer's leaf certificate.
  std::vector<absl::string_view> ip_names() const;
}

// The base class of all verifier implementations. Note that custom
// verifier implementations can compose their functionality with existing
// implementations of this interface, such as HostnameVerifier, by delegating
// to an instance of that class.
class CertificateVerifier {
 public:
  virtual ~CertificateVerifier() = default;

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
              std::function<void(Status)> callback,
              Status* sync_status);

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
class CustomChainBuilderInterface {
public:
 virtual ~CustomChainBuilderInterface() = default;

 // Returns the verified DER-encoded certificate chain, ordered from leaf to
 // root. Both the leaf and the root are included. Returns an error status if
 // verification fails for any reason.
 virtual absl::StatusOr<std::vector<absl::string_view>> BuildAndVerifyChain(const std::vector<absl::string_view>& peer_cert_chain_der) = 0;
}
```

The default behavior for chain building is to defer to the underlying SSL library, which has a built-in chain building API that has been hardened over many years of use with the web PKI. For example, [X509_verify_cert](https://www.openssl.org/docs/man1.1.1/man3/X509_verify_cert.html) is the implementation in OpenSSL that builds a chain and verifies that it is trusted by one of the root certificates.

```c++
class SPIFFEFederationChainBuilder : public CustomChainBuilderInterface {
public:
 ~SPIFFEFederationChainBuilder() override;

 // Builds a trusted and validated certificate chain based on SPIFFE trust maps
 // rather than the standard X509 approach.
 absl::StatusOr<std::vector<absl::string_view>> BuildAndVerifyChain(const std::vector<absl::string_view>& peer_cert_chain_der) override;
}
```