A107 - TLS Private Key Offloading
----
* Author: @gtcooke94
* Approver: ejona86
* Status: In Review
* Implemented in: C++, Go
* Last updated: 2025-12-01
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

This document outlines gRPC's plan to support TLS private key offloading, allowing a separate module (e.g., hardware) to handle private key signing during TLS handshakes for enhanced security and flexibility. 

## Background

A server possesses an identity certificate and a private key to identify itself during a handshake for authentication. During the TLS handshake, the server uses its private key to sign the transcript of the handshake, thereby validating ownership of said private key to the client. Currently, gRPC exclusively supports directly accessible, in-memory private keys; however, specialized private key storage solutions exist. TLS Private Key Offloading is a process by which an application performing a TLS handshake can use a private key to sign data *without* having direct access to the private key itself. This document will detail gRPC's prospective support for private key offloading.

This feature will have the following requirements/assumptions:

* TLS1.3 or TLS1.2 with modern ciphersuites that use ECDHE (Elliptic Curve Diffie-Hellman Ephemeral) for key exchange.  
  * HTTP/2 mandates TLS\_ECDHE\_{RSA/ECDSA}\_WITH\_AES\_128\_GCM\_SHA256 as a baseline ciphersuite, explicitly prohibiting older, non-ephemeral Diffie-Hellman methods.  
  * In non-ephemeral Diffe-Hellman key exchange, the private key could be used for other cryptographic operations (e.g. decryption with an RSA key). These will not be supported.


### Related Proposals: 
* TODO

## Proposal

The crypto libraries that each gRPC implementation uses have support for TLS Private Key Offloading. BoringSSL, for example, has the following interface:

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

For TLS\>=1.2 with ECDHE, the `decrypt` method is not needed. We will provide an interface through which users can provide a signing function, with the base functionality being a function that takes in unsigned bytes and a signature algorithm and returns signed bytes. The `complete` function is an implementation detail for gRPC to handle.

```
string signed_bytes sign(string algorithm, string unsigned_bytes)
```

 


### Temporary environment variable protection

This feature will be explicitly configured by users, thus no environment variable protection is needed. If a user does not configure TLS Private Key Offloading, it will not happen.

## Rationale


Private key offloading is designed to support signing outside of the existing process, for example in a hardware module or via an RPC \- thus this API should support asynchronous operations in languages where that is possible (Golang's crypto/tls does **not** support asynchronous private key signing). 

We are largely restricted by the underlying security libraries in each language. In the following sections, each language's API will be discussed as they are dependent upon the SSL library interfaces. Further, each language has different expectations for the sign functions on whether raw bytes or a digest is expected.


## Implementation

### C-Core

#### *APIs*

BoringSSL provides an asynchronous API for private key signing, so we will provide an asynchronous, functional API using callbacks. The C-Core APIs used to configure this feature will rely on a new certificate provider implementation.

We will create an `InMemoryCertificateProvider` that takes a set of root certs and a list of `PemKeyCertPair`. Further, these values can be manually updated by the user.  
We will also decouple the root provider and the identity provider in [the tls\_credentials\_options](http://google3/third_party/grpc/src/core/credentials/transport/tls/grpc_tls_credentials_options.h;l=77;rcl=731487339). Users should be able to specify, for example, an `InMemoryCertificateProvider` that implements private key offloading and a `FileWatcherCertificateProvider` for the root certificates. We will mark the [coupled API](http://google3/third_party/grpc/include/grpcpp/security/tls_credentials_options.h;l=55-56;rcl=682352913) as deprecated.

```c
using PrivateKey = std::variant<absl::string_view, CustomPrivateKeySign>

class PemKeyCertPair {
 public:
  PemKeyCertPair(PrivateKey private_key, absl::string_view cert_chain);
}

// Implements a provider that uses in-memory data that can be modified in a thread-safe manner.
class InMemoryCertificateProvider : grpc_tls_certificate_provider {
 public:
  InMemoryCertificateProvider(std::string root_certificates, PemKeyCertPairList pem_key_cert_pairs);
  
  // thread safe updates
  UpdateRoot(std::string root_certificates);
  UpdateIdentity(PemKeyCertPairList pem_key_cert_pairs); 
}

struct grpc_tls_credentials_options {
  // Deprecated. Use `set_root_certificate_provider` and 
  // `set_identity_certificate_provider` instead. 
  void set_certificate_provider(grpc_core::RefCountedPtr<grpc_tls_certificate_provider>   certificate_provider);
}

// Sets the `grpc_tls_certificate_provider` to provide identity data.
void set_identity_certificate_provider(grpc_core::RefCountedPtr<grpc_tls_certificate_provider> certificate_provider);

// Sets the `grpc_tls_certificate_provider` to provide root data.
void set_root_certificate_provider(grpc_core::RefCountedPtr<grpc_tls_certificate_provider> certificate_provider);
```

#### *Internals*

1. User configures gRPC credentials with a PemKeyCertPair providing the CustomPrivateKeySign function.

   [`PemKeyCertPair` already exists](http://google3/third_party/grpc/src/core/credentials/transport/tls/ssl_utils.h;l=153-167;rcl=786762336) but only accepts a `string` private key.  We will modify this to optionally accept a private key signing function.

```c

// A user's implementation MUST invoke `done_callback` with the signed bytes. This will let gRPC take control when the async operation is complete.
// MUST not block
// MUST support concurrent calls
using CustomPrivateKeySign = absl::AnyInvocable<void(
    absl::string_view data_to_sign,
    SignatureAlgorithm signature_algorithm,
    absl::AnyInvocable<void(absl::StatusOr<std::string> signed_data)> done_callback
)>;


struct PemKeyCertPair {
  std::string private_key;  // PEM-encoded private key.
  std::string cert_chain;   // PEM-encoded certificate chain.
  // Asynchronous private key signing function. Mutually exclusive with private_key.
  CustomPrivateKeySign private_key_sign;
  // Associated ctors, etc
}



// Enum class representing TLS signature algorithm identifiers from BoringSSL.
// The values correspond to the SSL_SIGN_* macros in <openssl/ssl.h>.
enum class SignatureAlgorithm : uint16_t {
  kRsaPkcs1Sha256 = 0x0401,           // SSL_SIGN_RSA_PKCS1_SHA256
  kRsaPkcs1Sha384 = 0x0501,           // SSL_SIGN_RSA_PKCS1_SHA384
  kRsaPkcs1Sha512 = 0x0601,           // SSL_SIGN_RSA_PKCS1_SHA512
  kEcdsaSecp256r1Sha256 = 0x0403,    // SSL_SIGN_ECDSA_SECP256R1_SHA256
  kEcdsaSecp384r1Sha384 = 0x0503,    // SSL_SIGN_ECDSA_SECP384R1_SHA384
  kEcdsaSecp521r1Sha512 = 0x0603,    // SSL_SIGN_ECDSA_SECP521R1_SHA512
  kRsaPssRsaeSha256 = 0x0804,         // SSL_SIGN_RSA_PSS_RSAE_SHA256
  kRsaPssRsaeSha384 = 0x0805,         // SSL_SIGN_RSA_PSS_RSAE_SHA384
  kRsaPssRsaeSha512 = 0x0806,         // SSL_SIGN_RSA_PSS_RSAE_SHA512
};



```

2. gRPC's TLS stack configures the SSL\_CTX with the custom SSL\_PRIVATE\_KEY\_METHOD.

   We will add a type, `TlsPrivateKeyOffloadContext` to manage the state of this operation and store it as `ex_data` on the SSL object.

```c

// State associated with an SSL object for async private key operations.
struct TlsPrivateKeyOffloadContext {
  CustomPrivateKeySign private_key_sign;
  absl::StatusOr<std::string> signed_bytes;

  // TSI handshake state needed to resume.
  tsi_handshaker* handshaker;
  tsi_handshaker_on_next_done_cb notify_cb;
}



// Callback function to be invoked when the user's async sign operation is complete.
// This function is curried with 'ctx' using absl::bind_front.
static void TlsOffloadSignDoneCallback(TlsPrivateKeyOffloadContext* ctx,
                                         absl::StatusOr<std::string> signed_data) {

  if (signed_data.ok()) {
    ctx->signed_data = std::move(signed_data);
  }
  //... handle other cases

  // Notify the TSI layer to re-enter the handshake.
  // This call is thread-safe as per TSI requirements for the callback.
  if (ctx->notify_cb) {
    ctx->notify_cb(ctx->handshaker, ctx->notify_user_data, TSI_OK);
  }
}

```

 

We then will implement [BoringSSL's ssl\_private\_key\_method\_st](http://google3/third_party/openssl/boringssl/src/include/openssl/ssl.h;l=1352-1356;rcl=814357088) with the user's signing function.

```c

static enum ssl_private_key_result_t TlsPrivateKeySignWrapper(
    SSL* ssl, uint8_t* out, size_t* out_len, size_t max_out,
    uint16_t signature_algorithm, const uint8_t* in, size_t in_len) {
  // Get and fill the TlsPrivateKeyOffloadContext.
  TlsPrivateKeyOffloadContext* ctx = static_cast<TlsPrivateKeyOffloadContext*>(
      SSL_get_ex_data(...)
  // Fill important info
  // ... 
  // Create the completion callback by binding the current context.
  auto done_callback =
      absl::bind_front(TlsOffloadSignDoneCallback, ctx);

  // Call the user's async sign function
  // The contract with the user is that they MUST invoke the callback when complete in their implementation, and their impl MUST not block. 
  ctx->private_key_sign(data_to_sign, signature_algorithm,
                       std::move(done_callback));

  return ssl_private_key_retry;
}

static enum ssl_private_key_result_t TlsPrivateKeyOffloadComplete(
    SSL* ssl, uint8_t* out, size_t* out_len, size_t max_out) {
  // Get TlsPrivateKeyOffloadContext.
  TlsPrivateKeyOffloadContext* ctx = static_cast<TlsPrivateKeyOffloadContext*>(
      SSL_get_ex_data(...);
  // Various checks
  // ...
  // Important bit is moving the signed data where it needs to go 
  memcpy(out, ctx->signed_data.data(), ctx->signed_data.length());
  *out_len = ctx->signed_data.length();
  // Tell BoringSSL we're done
  return ssl_private_key_success;
}

static const SSL_PRIVATE_KEY_METHOD TlsOffloadPrivateKeyMethod = {
    TlsPrivateKeySignWrapper,
    nullptr,  // decrypt not implemented for this use case
    TlsPrivateKeyOffloadComplete};

```

3. When a handshake starts, tsi\_handshaker\_next is called.  
4. The TSI layer creates and configures an SSL object, attaching the TlsPrivateKeyOffloadContext (including the user's function and TSI callback info) as ex\_data.  
5. tsi\_handshaker\_next calls SslDoHandshake (in ssl\_util.cc), which calls SSL\_do\_handshake.  
6. BoringSSL eventually requires the signature, calls our SSL\_PRIVATE\_KEY\_METHOD sign impl.  
7. TlsPrivateKeySignWrapper invokes the user's private\_key\_sign, providing the `TlsOffloadSignDoneCallback`. It returns ssl\_private\_key\_retry.  
8. SSL\_do\_handshake returns, SSL\_get\_error is SSL\_ERROR\_WANT\_PRIVATE\_KEY\_OPERATION.  
9. Seeing SSL\_ERROR\_WANT\_PRIVATE\_KEY\_OPERATION, SslDoHandshake returns TSI\_ASYNC . tsi\_handshaker\_next returns TSI\_ASYNC. [gRPC core yields control.](http://google3/third_party/grpc/src/core/handshaker/security/security_handshaker.cc;l=426-431;rcl=807452496)  
10. The user's async operation completes, invoking the `TlsOffloadSignDoneCallback`.  
11. The callback stores the result in TlsPrivateKeyOffloadContext and calls the notify\_cb (which is tsi\_handshaker\_on\_next\_done\_cb).

    This callback is part of TSI and is [provided by the caller](http://google3/third_party/grpc/src/core/handshaker/security/security_handshaker.cc;l=422-425;rcl=807452496) of `tsi_handshaker_next`. In our case in gRPC, this is [the `OnHandshakeNextDoneGrpcWrapper` function](http://google3/third_party/grpc/src/core/handshaker/security/security_handshaker.cc;l=402;rcl=807452496) .

    This returns control to gRPC Core, resuming the handshake.

12. gRPC core calls tsi\_handshaker\_next again.  
13. This leads to SSL\_do\_handshake being called again.  
14. BoringSSL, seeing the pending operation, now calls TlsPrivateKeyOffloadComplete.  
15. TlsPrivateKeyOffloadComplete retrieves the signed\_data from TlsPrivateKeyOffloadContext and provides it to BoringSSL.  
16. The handshake continues with the signed data.

### C++

For the C++ APIs, we will try to alias as much of the C-Core API as we can. C++ has its own `IdentityKeyCertPair` that converts to `PemKeyCertPair` from C-Core, so we will add similar code to that struct to support a signing function.  
The `TlsCredentialsOptions` will be similarly updated to differentiate between an identity provider and a root provider.  
We will also create a C++ wrapper around the `InMemoryCertificateProvider` in the same way the other provider implementations are wrapped.

```c

using grpc_core::SignatureAlgorithm;
using grpc_core::CustomPrivateKeySignCallback;
using grpc_core::PrivateKey;

// This is the C++ API for PemKeyCertPair
struct IdentityKeyCertPair {
  PrivateKey private_key;
  std::string certificate_chain;
  // Associated ctors, etc
}

class TlsCredentialsOptions {
  // Deprecated. Use `set_identity_certificate_provider` and 
  // `set_root_certificate_provider` instead.
  void set_certificate_provider(
      std::shared_ptr<CertificateProviderInterface> certificate_provider);

  void set_identity_certificate_provider(
      std::shared_ptr<CertificateProviderInterface> certificate_provider);

  void set_root_certificate_provider(
      std::shared_ptr<CertificateProviderInterface> certificate_provider);
}

// Basic wrapping of C-Core
class InMemoryCertificateProvider : public CertificateProviderInterface {

  static std::shared_ptr<InMemoryCertificateProvider> Create(std::string root_cert, std::vector<IdentityKeyCertPair> key_cert_pairs);

  // thread safe updates
  UpdateRoot(std::string root_certificates);
  UpdateIdentity(absl::Span<IdentityKeyCertPair> key_cert_pairs); 
}

// To use, set the provider on the TlsCredentialsOptions
```

### Python

Python wraps the C-Core implementation. Currently, Python's security configuration wraps the `SslCredentials` instead of the `TlsCredentials`. We will update gRPC-Python internally to use `TlsCredentials` (PR [\#40878](https://github.com/grpc/grpc/pull/40878)). Then, the task of supporting this feature in Python is similar to C++. We will wrap the new types and split the certificate provider on `tls_credentials_options` to support different root and identity providers.

The current Python API is very simple:

```py
PrivateKeySignDoneCallback = Callable[[Optional[bytes], bool], None]
# Note: SignatureAlgorithm corresponds to C-core's enum class SignatureAlgorithm.
CustomPrivateKeySign = Callable[[bytes, SignatureAlgorithm, PrivateKeySignDoneCallback], None]


def ssl_channel_credentials_with_custom_signer(
    *,
    private_key_sign_fn: CustomPrivateKeySign,
    root_certificates: Optional[bytes] = None,
    certificate_chain: Optional[bytes] = None,
) -> ChannelCredentials:
    """Creates a ChannelCredentials for use with an SSL-enabled Channel.

    Args:

      private_key_sign_fn: A function implementing
        `CustomPrivateKeySign`.
        `CustomPrivateKeySign` MUST NOT block, MUST be thread-safe, and MUST call
        `PrivateKeySignDoneCallback` on completion.
      root_certificates: The PEM-encoded root certificates as a byte string,
        or None to retrieve them from a default location chosen by gRPC
        runtime.
      certificate_chain: The PEM-encoded certificate chain as a byte string
        to use or None if no certificate chain should be used.

    Returns:
      A ChannelCredentials for use with an SSL-enabled Channel.
    """


```

We won't significantly refactor the Python API surface \- instead we will allow the `private_key` input to be a signing function.

The most complex piece of this is the implementation \- C-Core/Cython/Python must handle the calling of the user provided Python signing function from C which must invoke a callback that is passed to it. This will involve creating bridge types between the Python user sign function and the expected `absl::AnyInvocable` as well as bridging the callback that is passed to the user sign function while managing the GIL and asynchronous nature of the signing. This is technically feasible with Cython. A proof-of-concept of this structure [is written here.](https://source.corp.google.com/piper///depot/google3/experimental/users/gregorycooke/python_cpp_wrapping/) 

```py
# Example Usage

# example_signer is a basic implementation CustomPrivateKeySign
def example_signer(
    unsigned_data: bytes,
    algorithm: SignatureAlgorithm,
    on_done: PrivateKeySignDoneCallback,
) -> None:
    # Manually sign the bytes.
    signed_bytes = ...
    if signed_bytes:
        on_done(signed_bytes, True)
    else:
        on_done(None, False)




# Now the user is in their application configuring gRPC 
# Create creds with the custom signer
creds = ssl_channel_credentials_with_custom_signer(<some_root>, example_signer, <some_chain>)

```

### Go

Golang's `crypto/tls` package does not directly support asynchronous operations for the `Signer` interface, but this is not a problem due to the structure of goroutines. The API will simply look like implementing the `crypto/tls` `Signer` API. In this library, the [`Certificate` type's](http://google3/third_party/go/gc/src/crypto/tls/common.go;l=1559;rcl=815505302)  `PrivateKey` is a [`crypto.PrivateKey`](https://pkg.go.dev/crypto#PrivateKey), which is `Any`, but it must implement the [`Signer` interface](https://pkg.go.dev/crypto#Signer). Notably, in Golang, the user's function should expect a *digest* of the transcript as input.

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

This is the interface that a user will implement and pass to gRPC-Go via a `tls.Certificate`.  
We will create a similar `InMemoryCertProvider` that takes and provides a `tls.Certificate`. This will live in the advancedTLS package.

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

This will be piped around by the existing infrastructure to configure the `crypto/tls` library with no additional changes needed.

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

After initial investigation, we will not pursue implementing this feature in Java right now. Due to the different designs of the options of underlying security library APIs, there is no cohesive convenience API that we could add to gRPC-Java. A user can still implement this themselves and use the `AdvancedTlsKeyProvider` APIs to do whatever they want with the `PrivateKey` interface. Particularly, a user could create their own signature provider and globally register it to be used. gRPC-Java, as a library, will not do global registration.

# **Observability**

The following non-per-call metric and labels will be added. These will allow a user insight into the offloaded operations and will be an aid in debugging failures.

| Name | Disposition | Description |
| :---- | :---- | :---- |
| `grpc.target` | required | Indicates the target of the gRPC channel for which this handshake is occurring |
| `grpc.security.handshaker.offloaded_private_key_sign.algorithm` | optional | The signature algorithm used to sign |
| `grpc.security.handshaker.offloaded_private_key_sign.status` | optional | The status code return for the private key sign function |

| Metric | Type | Unit | Labels | Description |
| :---- | :---- | :---- | :---- | :---- |
| `grpc.security.handshaker.offloaded_private_key_sign_duration` | Histogram | float64/double s | `grpc.target, grpc.security.handshaker.offloaded_private_key_sign.algorithm, grpc.security.handshaker.offloaded_private_key_sign.status` | How long the offloaded private key signing took |

For the latency metric, we will use the buckets as defined in [gRFC A66](https://github.com/grpc/proposal/blob/fcabdfdbd50b3c088f5a5c2bf925755781ec076e/A66-otel-stats.md?plain=1#L360-L369).

The `grpc.security.handshaker.offloaded_private_key_sign.algorithm` label will contain both a name and key length.


## Open issues (if applicable)

* The C APIs may be built around a class rather than straight passing `absl::AnyInvocable` or `std::function`.
* The exact additions vs. breakages to the C-Core APIs and C++ APIs will be decided during PR reviews. Some of the APIs we are modifying are experimental-but-old, so we want to be careful when making breaking changes.
