A107 - TLS Private Key Offloading
----
* Author: @gtcooke94
* Approver: ejona86
* Status: C++ implemented
* Implemented in: C++, Go
* Last updated: 2026-03-18
* Discussion at: https://groups.google.com/g/grpc-io/c/N02jVxPd_4Y/m/n34PWOyKBgAJ?e=48417069

## Abstract

This document outlines gRPC's plan to support TLS private key offloading,
allowing a separate module (e.g., hardware) to handle private key signing during
TLS handshakes for enhanced security and flexibility. 

## Background

A server possesses an identity certificate and a private key to identify itself
during a handshake for authentication. During the TLS handshake, the server uses
its private key to sign the transcript of the handshake, thereby validating
ownership of said private key to the client. Currently, gRPC exclusively
supports directly accessible, in-memory private keys; however, specialized
private key storage solutions exist. TLS Private Key Offloading is a process by
which an application performing a TLS handshake can use a private key to sign
data *without* having direct access to the private key itself. This document
will detail gRPC's prospective support for private key offloading.

This feature will have the following requirements/assumptions:

* TLS1.3 or TLS1.2 with modern ciphersuites that use ECDHE (Elliptic Curve Diffie-Hellman Ephemeral) for key exchange.  
  * HTTP/2 mandates TLS\_ECDHE\_{RSA/ECDSA}\_WITH\_AES\_128\_GCM\_SHA256 as a baseline ciphersuite, explicitly prohibiting older, non-ephemeral Diffie-Hellman methods.  
  * In non-ephemeral Diffe-Hellman key exchange, the private key could be used for other cryptographic operations (e.g. decryption with an RSA key). These will not be supported.


### Related Proposals: 
* [A66]
* [A79]

[A66]: A66-otel-stats.md
[A79]: A79-non-per-call-metrics-architecture.md

## Proposal

The crypto libraries that each gRPC implementation uses have support for TLS
Private Key Offloading. BoringSSL, for example, has the following interface:

```c
struct ssl_private_key_method_st {
  enum ssl_private_key_result_t (*sign)(SSL *ssl, uint8_t *out, size_t *out_len,
                                        size_t max_out,
                                        uint16_t signature_algorithm,
                                        const uint8_t *in, size_t in_len);

  enum ssl_private_key_result_t (*decrypt)(SSL *ssl, uint8_t *out,
                                           size_t *out_len, size_t max_out,
                                           const uint8_t *in, size_t in_len);

  enum ssl_private_key_result_t (*complete)(SSL *ssl, uint8_t *out,
                                            size_t *out_len, size_t max_out);
};

```

For TLS\>=1.2 with ECDHE, a requirement for this feature, the `decrypt` method is not needed. We will provide an
interface through which users can provide a custom private key signer, with the
higher-level functionality being a function that takes in unsigned bytes and a
signature algorithm and returns signed bytes.

```
string signed_bytes sign(string algorithm, string unsigned_bytes)
```

### C-Core / C++

There are three distinct changes:
1. Creating an InMemory Certificate Provider
     * Must be done to cleanly accept the PrivateKeySigner instead of just a static PEM key or a file path.
2. Splitting Certificate Providers into Root and Identity Providers
    * Must be done to decouple how identity (certificate and key signing) and roots of trust are defined. Prior to this, roots had to be provided the same way that identities were provided, which wouldn't work for a PrivateKeySigner API.
3. The PrivateKeySigner API itself.


#### InMemory Certificate Provider
We will create an `InMemoryCertificateProvider` that takes a set of root certs
and a list of `PemKeyCertPair`. Further, these values can be manually updated by
the user. This provides a strict super-set of functionality of the current
`StaticDataCertificateProvider` - calling `UpdateRoot` and `UpdateIdentity` on
an `InMemoryCertificateProvider` at setup is the same as a
`StaticDataCertificateProvider`. Further, we will be deprecating
`StaticDataCertificateProvider`.

The updates will return an `absl::Status` to the caller, and further a
`ValidateCredentials` API is provided.

```c++
// Implements a provider that uses in-memory data that can be modified in a thread-safe manner.
class InMemoryCertificateProvider : grpc_tls_certificate_provider {
 public:
  InMemoryCertificateProvider(std::string root_certificates, PemKeyCertPairList pem_key_cert_pairs);
  
  // Thread safe updates
  absl::Status UpdateRoot(std::string root_certificates);
  absl::Status UpdateIdentityKeyCertPair(
      const std::vector<IdentityKeyCertPair>& identity_key_cert_pairs);

  // Returns an OK status if the following conditions hold:
  // - the root certificates consist of one or more valid PEM blocks, and
  // - every identity key-cert pair has a certificate chain that consists of
  //   chain that consists of valid PEM blocks and has a private key is a valid
  //   PEM block.
  absl::Status ValidateCredentials() const;
}
```

#### Splitting Certificate Providers into Identity and Root Providers
We will also decouple the root provider and the identity provider in [the
tls\_credentials\_options](https://github.com/grpc/grpc/blob/fa77e332f6a4a88902c012a83703071c251dc4bb/src/core/credentials/transport/tls/grpc_tls_credentials_options.h#L40)
and through the stack.  Users should be able to specify, for example, an
`InMemoryCertificateProvider` that implements private key offloading and a
`FileWatcherCertificateProvider` for the root certificates. We will mark the
[coupled
API](https://github.com/grpc/grpc/blob/fa77e332f6a4a88902c012a83703071c251dc4bb/include/grpcpp/security/tls_credentials_options.h#L58)
as deprecated.  This is required for this effort, as specifying a private key
signer must be done for the identity, but is unrelated to how the root is
provided. Currently, identity (private key and certificate chain) presentation
to gRPC is coupled with the roots presentation to gRPC.

```c++
class TlsCredentialsOptions {
 public:
  // The existing method.
  [[deprecated(
      "Use set_root_certificate_provider() or "
      "set_identity_certificate_provider() instead.")]]
  void set_certificate_provider(
      std::shared_ptr<CertificateProviderInterface> certificate_provider);
  // The new method proposed in this gRFC.
  void set_root_certificate_provider(
      std::shared_ptr<CertificateProviderInterface> certificate_provider);
  // The new method proposed in this gRFC.
  void set_identity_certificate_provider(
      std::shared_ptr<CertificateProviderInterface> certificate_provider);
```

#### Private Key Signing

BoringSSL provides an asynchronous API for private key signing, so we will
provide an asynchronous, cancellable API using callbacks. Thus, this is only supported in gRPC builds with BoringSSL. If a
`PrivateKeySigner` is used in a non-BoringSSL build, the user should expect
failure.

The implementation of `PrivateKeySigner::Sign` can choose to return
synchronously or asynchronously via a callback. The implementer must also
implement cancellation.  `Cancel` will be called by gRPC in the case that an
in-flight operation must be shutdown, and the implementer should use this
function to ensure any resources being used for an asynchronous signing
operation are properly shut down and released.


```c++

// Implementations of this class must be thread-safe.
class PrivateKeySigner {
 public:
  // A handle for an asynchronous signing operation.
  //
  // When `PrivateKeySigner::Sign` is implemented asynchronously, it returns an
  // instance of a concrete implementation of this class. This handle is used
  // to manage the asynchronous signing operation and can be used to cancel the
  // operation via `PrivateKeySigner::Cancel`.
  //
  // Users must provide their own concrete implementation of this class. The
  // handle can store any state needed for the asynchronous operation.
  class AsyncSigningHandle {
   public:
    virtual ~AsyncSigningHandle() = default;
  };

  // Enum class representing TLS signature algorithm identifiers from BoringSSL.
  // The values correspond to the SSL_SIGN_* macros in <openssl/ssl.h>.
  enum class SignatureAlgorithm {
    kRsaPkcs1Sha256,
    kRsaPkcs1Sha384,
    kRsaPkcs1Sha512,
    kEcdsaSecp256r1Sha256,
    kEcdsaSecp384r1Sha384,
    kEcdsaSecp521r1Sha512,
    kRsaPssRsaeSha256,
    kRsaPssRsaeSha384,
    kRsaPssRsaeSha512,
  };

  // A callback that is invoked when an asynchronous signing operation is
  // complete. The argument should contain the signed bytes on success, or a
  // non-OK status on failure.
  using OnSignComplete = absl::AnyInvocable<void(absl::StatusOr<std::string>)>;

  virtual ~PrivateKeySigner() = default;

  // Signs data_to_sign.
  // May return either synchronously or asynchronously.
  // For synchronous returns, directly returns either the signed bytes
  // or a failed status, and the callback will never be invoked.
  // For asynchronous implementations, returns a handle for the asynchronous
  // signing operation. The function argument on_sign_complete must be called by
  // the implementer when the async signing operation is complete.
  // on_sign_complete must not be invoked synchronously within Sign().
  virtual std::variant<absl::StatusOr<std::string>,
                       std::shared_ptr<AsyncSigningHandle>>
  Sign(absl::string_view data_to_sign, SignatureAlgorithm signature_algorithm,
       OnSignComplete on_sign_complete) = 0;

  // Cancels an in-flight async signing operation using a handle returned
  // from a previous call to Sign().
  virtual void Cancel(std::shared_ptr<AsyncSigningHandle> handle) = 0;
};
```

The C-Core APIs used
to configure this feature will rely on a new certificate provider
implementation. 

```c
// This already exists - it will be deprecated and replaced with `IdentityKeyOrSignerCertPair` where necessary.
struct [[deprecated("Use IdentityKeyOrSignerCertPair instead")]] GRPCXX_DLL
    IdentityKeyCertPair {
  std::string private_key;
  std::string certificate_chain;
};

// A struct that stores the credential data presented to the peer in handshake
// to show local identity. The private_key and certificate_chain should always
// match. The private_key can be either a PEM string or a PrivateKeySigner.
// The PrivateKeySigner will only work with gRPC binaries compiled with
// BoringSSL.
struct GRPCXX_DLL IdentityKeyOrSignerCertPair {
  std::variant<std::string, std::shared_ptr<PrivateKeySigner>> private_key;
  std::string certificate_chain;
};

// A new overload on the InMemoryCertificateProvider to take this new struct
absl::Status UpdateIdentityKeyCertPair(
    std::vector<IdentityKeyOrSignerCertPair>
        identity_key_or_signer_cert_pairs);

```
### Python

Python wraps the C-Core implementation. Currently, Python's security
configuration wraps the `SslCredentials` instead of the `TlsCredentials`. We
will update gRPC-Python internally to use `TlsCredentials` (PR
[\#40878](https://github.com/grpc/grpc/pull/40878)). Then, the task of
supporting this feature in Python is similar to C++. We will wrap the new types
and split the certificate provider on `tls_credentials_options` to support
different root and identity providers.

The current Python API takes a function rather than attempting to do the
interface-based approach from C++.  It returns either how to cancel the async
operation in the async case or a signature in the sync case.

Note: gevent is NOT supported.

```py

# A Callable to return in the async case
# See the `ssl_channel_credentials_with_custom_signer` docstring for more detail on usage.
PrivateKeySignCancel = Callable[[], None]
PrivateKeySignatureAlgorithm = _cygrpc.PrivateKeySignatureAlgorithm
PrivateKeySignOnComplete = Callable[[Union[bytes, Exception]], None]

# See the `ssl_channel_credentials_with_custom_signer` docstring for more detail on usage.
# The custom signing function for a user to implement and pass to gRPC Python.
CustomPrivateKeySign = Callable[
    [
        bytes,
        PrivateKeySignatureAlgorithm,
        "PrivateKeySignOnComplete",
    ],
    Union[bytes, "PrivateKeySignCancel"],
]


@experimental_api
def ssl_channel_credentials_with_custom_signer(
    *,
    private_key_sign_fn: "CustomPrivateKeySign",
    root_certificates: Optional[bytes] = None,
    certificate_chain: bytes,
) -> grpc.ChannelCredentials:
    """Creates a ChannelCredentials for use with an SSL-enabled Channel with a custom signer.

    THIS IS AN EXPERIMENTAL API.
    This API will be removed in a future version and combined with `grpc.ssl_channel_credentials`.

    Args:
      private_key_sign_fn: a function with the signature of
        `CustomPrivateKeySign`. This function can return synchronously or
        asynchronously.  To return synchronously, return the signed bytes.  To
        return asynchronously, return a callable matching the
        `PrivateKeySignCancel` signature.This can be a no-op if no cancellation is
        needed. In the async case, this function must return this callable
        quickly, then call the passed in `PrivateKeySignOnComplete` when the async
        signing operation is complete to trigger gRPC to continue the handshake.
      root_certificates: The PEM-encoded root certificates as a byte string,
        or None to retrieve them from a default location chosen by gRPC
        runtime.
      certificate_chain: The PEM-encoded certificate chain as a byte string
        to use

    Returns:
      A ChannelCredentials for use with an SSL-enabled Channel.
    """


```

The most complex piece of this is the implementation \- C-Core/Cython/Python
must handle the calling of the user provided Python signing function from C
which must invoke a callback that is passed to it. This will involve creating
bridge types between the Python user sign function and the expected
`absl::AnyInvocable` as well as bridging the callback that is passed to the user
sign function while managing the GIL and asynchronous nature of the signing.
This is technically feasible with Cython.


```py
# Example Usage

def sync_client_private_key_signer(
    data_to_sign,
    signature_algorithm,
    on_complete,
):
    """
    Takes in data_to_sign and signs it using the test private key with a sync return
    """
    private_key_bytes = client_private_key()
    signature = sign_private_key(
        data_to_sign, private_key_bytes, signature_algorithm
    )
    return signature

def async_signer_worker(data_to_sign, signature_algorithm, on_complete):
    """
    Meant to be used as an async function for a thread, for example
    """
    private_key_bytes = client_private_key()
    signature = sign_private_key(
        data_to_sign, private_key_bytes, signature_algorithm
    )
    on_complete(signature)


def no_op_cancel():
    pass


def async_client_private_key_signer(
    data_to_sign, signature_algorithm, on_complete
):
    """
    Takes in data_to_sign and signs it using the test private key
    """
    threading.Thread(
        target=async_signer_worker,
        args=(data_to_sign, signature_algorithm, on_complete),
    ).start()
    return no_op_cancel




# Now the user is in their application configuring gRPC 
# Create creds with the custom signer
creds = ssl_channel_credentials_with_custom_signer(<some_root>, <sync or async example_signer, <some_chain>)

```

### Go

Golang's `crypto/tls` package does not directly support asynchronous operations
for the `Signer` interface, but this is not a problem due to the structure of
goroutines. The API will simply look like implementing the `crypto/tls` `Signer`
API. In this library, the [`Certificate`
type's](https://pkg.go.dev/crypto/tls#Certificate)
`PrivateKey` is a [`crypto.PrivateKey`](https://pkg.go.dev/crypto#PrivateKey),
which is `Any`, but it must implement the [`Signer`
interface](https://pkg.go.dev/crypto#Signer). Notably, in Golang, the user's
function should expect a *digest* of the transcript as input.

```go
// From crypto.tls
type Signer interface {
	// Public returns the public key corresponding to the opaque,
	// private key.
	Public() PublicKey

	// Sign signs digest with the private key, possibly using entropy from
	// rand. For an RSA key, the resulting signature should be either a
	// PKCS #1 v1.5 or PSS signature (as indicated by opts). For an (EC)DSA
	// key, it should be a DER-serialised, ASN.1 signature structure.
	//
	// Hash implements the SignerOpts interface and, in most cases, one can
	// simply pass in the hash function used as opts. Sign may also attempt
	// to type assert opts to other types in order to obtain algorithm
	// specific values. See the documentation in each package for details.
	//
	// Note that when a signature of a hash of a larger message is needed,
	// the caller is responsible for hashing the larger message and passing
	// the hash (as digest) and the hash function (as opts) to Sign.
	Sign(rand io.Reader, digest []byte, opts SignerOpts) (signature []byte, err error)
}

cert = tls.Certificate{
Certificate: derChain, //[][]byte
PrivateKey:  PrivateKeySigner, //crypto.PrivateKey
}
```

This is the interface that a user will implement and pass to gRPC-Go via a
`tls.Certificate`.  We will create a similar `InMemoryCertProvider` that takes
and provides a `tls.Certificate`. This will live in the advancedTLS package.

```go

// InMemoryCertProvider is a certprovider.Provider implementation that holds and serves a
// single, predefined tls.Certificate instance.
type InMemoryCertProvider struct {
    tlsCert *tls.Certificate
}

func NewInMemoryCertProvider(cert *tls.Certificate) (*InMemoryCertProvider, error) {
    // Checks 
    return &InMemoryCertProvider{tlsCert: *cert}, nil
}

// Implements the `KeyMaterial` interface
// Thread safe access
func (i *InMemoryCertProvider) KeyMaterial(_ context.Context) (*KeyMaterial, error) {...}

// Allow the user to update with new data
// Thread safe updates
func (provider *InMemoryCertProvider) UpdateCertificate(tlsCert *tls.Certificate) {...}

```

This will be piped around by the existing infrastructure to configure the
`crypto/tls` library with no additional changes needed.

```go
// Example Usage

// ExampleSigner is a basic implementation of crypto.Signer
type ExampleSigner struct {}

// Public returns the public key corresponding to the opaque, private key.
func (s *ExampleSigner) Public() crypto.PublicKey {
	return <some key>
}

// Sign signs the digest with the private key.
func (s *ExampleSigner) Sign(r io.Reader, digest []byte, opts crypto.SignerOpts) ([]byte, error) {
       signed_bytes := <do stuff with digest>
	return signed_bytes, nil
}

// Now the user is in their application configuring gRPC 
// Create the custom signer  
customSigner, err := NewExampleSigner()

// Create the tls.Certificate
tlsCert := &tls.Certificate{
    Certificate: <pem-certificate>
    Signer: customSigner
}

// Create the InMemoryCertProvider and configure gRPC with it
provider, err := NewInMemoryCertProvider(tlsCert)
serverOpts := &advancedtls.Options{
	IdentityOptions: advancedtls.IdentityCertificateOptions{
		IdentityProvider: provider,
	},
}

// Down the line update
provider.UpdateCertificate(&tls.Certificate(<some updated thing>))

```

### Java

After initial investigation, we will not pursue implementing this feature in
Java right now. Due to the different designs of the options of underlying
security library APIs, there is no cohesive convenience API that we could add to
gRPC-Java. A user can still implement this themselves and use the
`AdvancedTlsKeyProvider` APIs to do whatever they want with the `PrivateKey`
interface. Particularly, a user could create their own signature provider and
globally register it to be used. gRPC-Java, as a library, will not do global
registration.

 


### Temporary environment variable protection

This feature will be explicitly configured by users, thus no environment
variable protection is needed. If a user does not configure TLS Private Key
Offloading, it will not happen.

## Rationale


Private key offloading is designed to support signing outside of the existing
process, for example in a hardware module or via an RPC \- thus this API should
support asynchronous operations in languages where that is possible (Golang's
crypto/tls does **not** support asynchronous private key signing). 

We are largely restricted by the underlying security libraries in each language.
In the following sections, each language's API will be discussed as they are
dependent upon the SSL library interfaces. Further, each language has different
expectations for the sign functions on whether raw bytes or a digest is
expected.


## Implementation

* C-Core/C++
    * https://github.com/grpc/grpc/pull/40878 - Migrate Python to TlsCredentials under the hood
    * https://github.com/grpc/grpc/pull/41490 - Separate cert provider into a root and identity provider
    * https://github.com/grpc/grpc/pull/41484 - Create InMemoryCertificateProvider
    * https://github.com/grpc/grpc/pull/41606 - Implement PrivateKeySigner in C-Core and C++

* Python
    * https://github.com/grpc/grpc/pull/41701 - Implement in Python and Cython


# **Observability**

We will add a new metric using the non-per-call metric framework described in
[A79]. This will will allow a user insight into the offloaded operations and
will be an aid in debugging failures.

The new metric will use the following labels:

| Label Name | Disposition | Description |
| :---- | :---- | :---- |
| `grpc.target` | required | Indicates the target of the gRPC channel for which this handshake is occurring. Defined in [A66]. |
| `grpc.tls.private_key_sign_algorithm` | optional | The signature algorithm used to sign. Contains both a name and key length. For example, `RsaPkcs1Sha256`. This will be a consistent string between languages.  |
| `grpc.status` | optional | The status code return for the private key sign function. From [A66] |


Here is the new metric we will add:

| Metric | Type | Unit | Labels | Description |
| :---- | :---- | :---- | :---- | :---- |
| `grpc.security.handshaker.offloaded_private_key_sign_duration` | Histogram | float64/double s | `grpc.target, grpc.tls.private_key_sign_algorithm, grpc.status` | How long the offloaded private key signing took |

For the latency metric, we will use the buckets as defined in [gRFC
A66](https://github.com/grpc/proposal/blob/fcabdfdbd50b3c088f5a5c2bf925755781ec076e/A66-otel-stats.md?plain=1#L360-L369).
