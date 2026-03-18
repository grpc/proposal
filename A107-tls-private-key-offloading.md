A107 - TLS Private Key Offloading
----
* Author: @gtcooke94
* Approver: ejona86
* Status: C++ implemented
* Implemented in: C++, Go
* Last updated: 2026-03-18
* Discussion at: https://groups.google.com/g/grpc-io/c/N02jVxPd_4Y/m/n34PWOyKBgAJ?e=48417069## Abstract

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

For TLS\>=1.2 with ECDHE, the `decrypt` method is not needed. We will provide an
interface through which users can provide a signing function, with the base
functionality being a function that takes in unsigned bytes and a signature
algorithm and returns signed bytes. The `complete` function is an implementation
detail for gRPC to handle.

```
string signed_bytes sign(string algorithm, string unsigned_bytes)
```

 


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

### C-Core

#### *APIs*

BoringSSL provides an asynchronous API for private key signing, so we will
provide an asynchronous, cancellable API using callbacks. The C-Core APIs used
to configure this feature will rely on a new certificate provider
implementation.

We will create an `InMemoryCertificateProvider` that takes a set of root certs
and a list of `PemKeyCertPair`. Further, these values can be manually updated by
the user.  We will also decouple the root provider and the identity provider in
[the
tls\_credentials\_options](http://google3/third_party/grpc/src/core/credentials/transport/tls/grpc_tls_credentials_options.h;l=77;rcl=731487339).
Users should be able to specify, for example, an `InMemoryCertificateProvider`
that implements private key offloading and a `FileWatcherCertificateProvider`
for the root certificates. We will mark the [coupled
API](http://google3/third_party/grpc/include/grpcpp/security/tls_credentials_options.h;l=55-56;rcl=682352913)
as deprecated.

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

We detail the async flow below. In the sync case, a signature is simply returned when it is asked for.

1. User configures gRPC credentials with a PemKeyCertPair providing the private key signer.

   [`PemKeyCertPair` already exists](http://google3/third_party/grpc/src/core/credentials/transport/tls/ssl_utils.h;l=153-167;rcl=786762336) but only accepts a `string` private key.  We will modify this to optionally accept a private key signer.

```c
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

2. gRPC's TLS stack configures the SSL\_CTX with the custom SSL\_PRIVATE\_KEY\_METHOD.

   We will add a type, `TlsPrivateKeyOffloadContext` to manage the state of this operation and store it on the `handshaker`.

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

  // Call the user's sign function
  // The contract with the user is that they MUST invoke the callback when complete in their implementation, and their impl MUST not block. 
  ctx->private_key_signer->sign(data_to_sign, signature_algorithm,
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

For the C++ APIs, we will simply be able to alias the `PrivateKeySigner` class outwards.

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
type's](http://google3/third_party/go/gc/src/crypto/tls/common.go;l=1559;rcl=815505302)
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

# **Observability**

The following non-per-call metric and labels will be added. These will allow a
user insight into the offloaded operations and will be an aid in debugging
failures.

| Name | Disposition | Description |
| :---- | :---- | :---- |
| `grpc.target` | required | Indicates the target of the gRPC channel for which this handshake is occurring |
| `grpc.security.handshaker.offloaded_private_key_sign.algorithm` | optional | The signature algorithm used to sign |
| `grpc.security.handshaker.offloaded_private_key_sign.status` | optional | The status code return for the private key sign function |

| Metric | Type | Unit | Labels | Description |
| :---- | :---- | :---- | :---- | :---- |
| `grpc.security.handshaker.offloaded_private_key_sign_duration` | Histogram | float64/double s | `grpc.target, grpc.security.handshaker.offloaded_private_key_sign.algorithm, grpc.security.handshaker.offloaded_private_key_sign.status` | How long the offloaded private key signing took |

For the latency metric, we will use the buckets as defined in [gRFC
A66](https://github.com/grpc/proposal/blob/fcabdfdbd50b3c088f5a5c2bf925755781ec076e/A66-otel-stats.md?plain=1#L360-L369).

The `grpc.security.handshaker.offloaded_private_key_sign.algorithm` label will contain both a name and key length.