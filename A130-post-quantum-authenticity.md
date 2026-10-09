A1XX - Post-Quantum Cryptography: Authenticity (Signature Algorithms)
----
* Author(s): @gtcooke94
* Approver: markdroth, ejona86, easwars, matthewstevenson88, dfawley
* Implemented in: C++, Go, Java
* Last updated: 2026-10-09
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

The industry's current best estimates are that a cryptographically-relevant
quantum computer (CRQC) will be available by Eo2028. Following the [March 30th,
2026](http://arxiv.org/abs/2603.28627) and [March 31st,
2026](https://research.google/blog/safeguarding-cryptocurrency-by-disclosing-quantum-vulnerabilities-responsibly/?utm_source=cryptography-dispatches&utm_medium=email&utm_campaign=crqc-timeline)
quantum computing announcements, gRPC addressed the immediate
store-now-decrypt-later (SNDL) threat to confidentiality in [A120] by preferring
the `X25519-MLKEM768` key exchange group. The next step in gRPC's post-quantum
cryptography (PQC) transition is ensuring post-quantum **authenticity** in its
SSL/TLS stacks.  Once a CRQC is available, an active adversary will be able to
forge classical digital signatures (RSA, ECDSA, Ed25519) in real time and
impersonate TLS endpoints. gRPC must support post-quantum TLS 1.3 signature
algorithms—specifically NIST FIPS 204 `ML-DSA-65` and `ML-DSA-87`—and expose
public APIs across C++, Go, and Java to configure TLS signature algorithms.



## Background

In [A120], post-quantum authenticity was scoped out of the initial high-priority
push because digital signatures are not vulnerable to passive
store-now-decrypt-later (SNDL) attacks; rather, an attacker must forge a
handshake signature live during connection establishment. An attacker could also
forge certificates themselves to successfully execute impersonation attacks
Because this is a live-attack, it lacks the immediate urgency of the
post-quantum confidentiality transition.  However, migrating PKIs across
production use-cases of gRPC could take significant wall-time, so there is value
in giving the users the tools to do that for gRPC deployments as soon as
possible.

In TLS 1.3 ([RFC 8446 Section 4.2.3](https://datatracker.ietf.org/doc/html/rfc8446#section-4.2.3)),
signature algorithms are negotiated via the `signature_algorithms` extension in
the `ClientHello` (for server authentication) and `CertificateRequest` (for
mTLS client authentication). A peer may only authenticate using a certificate
and `CertificateVerify` signature whose `SignatureScheme` was advertised by the
verifying peer.

In addition, there is another TLS extension named `signature_algorithms_cert`.
If unset, the `signature_algorithms` value is used. If set, this separately
represents a list of algorithms for which we can **verify** certificates (and
`signature_algorithms` represents algorithms we can **sign** with). An example:
we only hold classical private keys and certificates and can only produce RSA
signatures. However, our BoringSSL version does have support for MLDSA87, so we
can verify MLDSA87 signatures. We can separately advertise to a peer that we can
verify their MLDSA87 certificates (`signature_algorithms_cert`), while also
advertising that we cannot produce an MLDSA87 signature ourselves
(`signature_algorithms`). In gRPC use-cases, we don't actively see a strong
reason to prioritize `siganture_algorithms_cert` support at this time, and
instead will focus soley on implementing support for MLDSA and
`signature_algorithms` as soon as possible.

Currently, support for ML-DSA in the default TLS `signature_algorithms` list
varies across underlying cryptographic libraries. Most notably, BoringSSL
supports signing with `ML-DSA`, but does **not** include `ML-DSA` in its default
verification list (`kVerifySignatureAlgorithms`); negotiating an ML-DSA
certificate in BoringSSL requires explicitly configuring the signature
algorithms list via `SSL_CTX_set1_sigalgs_list`.

Furthermore, not all of gRPC's TLS stacks enable users to configure their own
signature algorithms. Exposing a signature algorithms API is both a direct
requirement for enabling ML-DSA today and a prerequisite for allowing users to
enforce PQC-only authentication policies or opt out of post-quantum signature
schemes if desired.


### Related Proposals

[A120] Post Quantum Cryptography

[A120]: A120-postquantum-cryptography.md


## Proposal

Each gRPC language implementation (C++, Go, and Java) will provide an API to
configure the ordered preference list of TLS signature algorithms (signature
schemes), including the NIST FIPS 204 Module-Lattice-Based Digital Signature
Standard algorithms standardized for TLS 1.3 in
[`draft-ietf-tls-mldsa`](https://datatracker.ietf.org/doc/html/draft-ietf-tls-mldsa)
(`ML-DSA-65` and `ML-DSA-87`, as well as `ML-DSA-44` where supported by the
underlying runtime) alongside standard classical algorithms.

In the TLS 1.3 handshake, signature algorithms are negotiated as follows, per
[RFC 8446](https://datatracker.ietf.org/doc/html/rfc8446#section-4.2.3): the
client sends an ordered list of `SignatureScheme` values it is willing to verify
in `ClientHello.signature_algorithms` (and in mTLS, the server sends its
supported verification schemes in `CertificateRequest.signature_algorithms`),
and the authenticating peer selects a certificate and signs `CertificateVerify`
using a mutually supported algorithm.

When a user configures a list of signature algorithms on the gRPC TLS
credentials options, gRPC will pass that ordered list through to the underlying
SSL/TLS library. When no signature algorithms are explicitly configured by the
user, gRPC will preserve the underlying security library's default signature
algorithms.

Of note, much of the challenging work of migrating a production gRPC setup lies
in migrating from classical to MLDSA signatures. This migration is NOT within
gRPC's scope as we cannot know user's PKI details. We will ensure that gRPC
contains the tools for users to implement their own migration strategy - namely
two items:

1. Supporting MLDSA in general
1. Adding APIs to configure the `signature_algorithms` TLS1.3 extension.
1. Ensuring the CertificateProvider interface abstractions work for
multi-certificate setups with mixed signatures and negotiation via configuring
the `signature_algorithms` extension.

### C++

| Library / Version | Default Verification Signature Algorithms | Source / Notes |
| :--- | :--- | :--- |
| BoringSSL | **Verify (`kVerifySignatureAlgorithms`, advertised in `ClientHello` / `CertificateRequest`)**:<br>`{ecdsa_secp256r1_sha256, rsa_pss_rsae_sha256, ...}`<br><br>**Sign (`kSignSignatureAlgorithms`)**:<br>`{mldsa44, mldsa65, mldsa87, ed25519, ecdsa_secp256r1_sha256, ...}` | [BoringSSL `kVerifySignatureAlgorithms`](https://github.com/google/boringssl/blob/fbb876dcee91c83a78d930dc9fab79db0c12053e/ssl/extensions.cc#L286-L302)<br>[BoringSSL `kSignSignatureAlgorithms`](https://github.com/google/boringssl/blob/fbb876dcee91c83a78d930dc9fab79db0c12053e/ssl/extensions.cc#L306-L332) | 
| OpenSSL < 3.5 | `{ecdsa_secp256r1_sha256, ecdsa_secp384r1_sha384, ecdsa_secp521r1_sha512, ed25519, ed448, rsa_pss_pss_*, rsa_pss_rsae_*, rsa_pkcs1_*}` | OpenSSL Default [`tls12_sigalgs`](https://github.com/openssl/openssl/blob/89cd17a031e022211684eb7eb41190cf1910f9fa/ssl/t1_lib.c#L977-L1015) |
| OpenSSL >= 3.5 | `{mldsa65, mldsa87, mldsa44, ecdsa_secp256r1_sha256, ecdsa_secp384r1_sha384, ecdsa_secp521r1_sha512, ed25519, ed448, ...}` | OpenSSL Default [`tls12_sigalgs`](https://github.com/openssl/openssl/blob/286ddeaac037533bbdce65b3c689e3f7ffebf0f6/ssl/t1_lib.c#L1930-L1974) |

The following table describes how gRPC C++ will configure signature algorithms
across SSL libraries. `ML-DSA-65` and `ML-DSA-87` are supported in BoringSSL and
OpenSSL 3.5+.

| Library / Version | Configuration / Behavior |
| :--- | :--- |
| BoringSSL | If `signature_algorithms` is set on `TlsCredentialsOptions`, configure via `SSL_CTX_set1_sigalgs_list` (required to advertise/verify `mldsa65` and `mldsa87`). Otherwise, prepend `mldsa87, mldsa65, mldsa44` to BoringSSL defaults. This is consistent with our approach in [A120] for post-quantum key exchange.|
| OpenSSL < 3.5 | If `signature_algorithms` is set, configure classical algorithms via `SSL_CTX_set1_sigalgs_list` (`ML-DSA` returns `TSI_INVALID_ARGUMENT`). Otherwise, use OpenSSL defaults. |
| OpenSSL >= 3.5 | If `signature_algorithms` is set, configure via `SSL_CTX_set1_sigalgs_list`. Otherwise, use OpenSSL 3.5+ defaults (`{mldsa65, mldsa87, mldsa44, ...}`). |

C-Core API Changes:
```c
typedef enum {
  GRPC_TLS_SIGNATURE_ALGORITHM_UNSPECIFIED,
  GRPC_TLS_SIGNATURE_ALGORITHM_RSA_PKCS1_SHA256,
  GRPC_TLS_SIGNATURE_ALGORITHM_RSA_PKCS1_SHA384,
  GRPC_TLS_SIGNATURE_ALGORITHM_RSA_PKCS1_SHA512,
  GRPC_TLS_SIGNATURE_ALGORITHM_ECDSA_SECP256R1_SHA256,
  GRPC_TLS_SIGNATURE_ALGORITHM_ECDSA_SECP384R1_SHA384,
  GRPC_TLS_SIGNATURE_ALGORITHM_ECDSA_SECP521R1_SHA512,
  GRPC_TLS_SIGNATURE_ALGORITHM_RSA_PSS_RSAE_SHA256,
  GRPC_TLS_SIGNATURE_ALGORITHM_RSA_PSS_RSAE_SHA384,
  GRPC_TLS_SIGNATURE_ALGORITHM_RSA_PSS_RSAE_SHA512,
  GRPC_TLS_SIGNATURE_ALGORITHM_ML_DSA_44,
  GRPC_TLS_SIGNATURE_ALGORITHM_ML_DSA_65,
  GRPC_TLS_SIGNATURE_ALGORITHM_ML_DSA_87,
} grpc_tls_signature_algorithm;

/**
 * Sets the supported TLS signature algorithms on the credentials options in
 * order of preference. If not set (or set to an empty list), the default
 * signature algorithms of the underlying SSL library are used.
 */
GRPCAPI void grpc_tls_credentials_options_set_signature_algorithms(
    grpc_tls_credentials_options* options,
    const grpc_tls_signature_algorithm* algorithms, size_t num_algorithms);
```

C++ API Changes:
```c++
class TlsCredentialsOptions {
 public:
  // ...
  // Sets the supported TLS signature algorithms in order of preference.
  // If not set (or set to an empty vector), the default signature algorithms
  // of the underlying SSL library are used.
  void set_signature_algorithms(
      const std::vector<grpc_tls_signature_algorithm>& signature_algorithms);
  // ...
};
```

### Go

In Golang, the `crypto/tls` library is part of the core language. Starting in Go
1.27, `crypto/mldsa`, `crypto/x509`, and `crypto/tls` natively support
`tls.MLDSA44`, `tls.MLDSA65`, and `tls.MLDSA87`, and include them at the front
of the default TLS 1.3 signature algorithms list
(`defaultSupportedSignatureAlgorithms`).

However, unlike `tls.Config.CurvePreferences`, Go's `tls.Config` does not
currently expose a top-level `SignatureAlgorithms` field to override the
`signature_algorithms` extension sent in `ClientHello` and `CertificateRequest`.
Instead, `crypto/tls` exposes:
* `tls.Certificate.SupportedSignatureAlgorithms` (`[]tls.SignatureScheme`) to
  restrict which signature schemes a local private key may be used for when
  signing.
* `tls.ClientHelloInfo.SignatureSchemes` and
  `tls.CertificateRequestInfo.SignatureSchemes` (`[]tls.SignatureScheme`) during
  certificate selection callbacks.

We propose that `advancedtls` will provide a direct option for users to restrict
or configure `SignatureAlgorithms`. We will add
`advancedtls.Options.SignatureAlgorithms` (`[]tls.SignatureScheme`), which will:
1. Populate `SupportedSignatureAlgorithms` on certificates managed by
   `advancedtls` so signing is restricted to the configured schemes.
2. Validate during handshake verification (and pass through to `tls.Config` if
   Go adds a connection-level `tls.Config` field in the future) that the peer's
   certificate and handshake signature use an allowed `tls.SignatureScheme`.

Go API Changes:
```go
package advancedtls

type Options struct {
	// ...
	// SignatureAlgorithms contains the signature schemes that will be used
	// during the TLS handshake, in preference order. If empty, the default
	// will be used.
	SignatureAlgorithms []tls.SignatureScheme
	// ...
}
```

| Go version | Defaults (`defaultSupportedSignatureAlgorithms` in TLS 1.3) |
| :---- | :---- |
| Go < 1.27 | `{PSSWithSHA256, ECDSAWithP256AndSHA256, Ed25519, PSSWithSHA384, PSSWithSHA512, ECDSAWithP384AndSHA384, ECDSAWithP521AndSHA512}` |
| Go >= 1.27 | `{MLDSA44, MLDSA65, MLDSA87, PSSWithSHA256, ECDSAWithP256AndSHA256, Ed25519, PSSWithSHA384, PSSWithSHA512, ECDSAWithP384AndSHA384, ECDSAWithP521AndSHA512}` |

### Java

Similar to key exchange groups in A120, gRPC-Java does not currently make
opinionated preferences on TLS signature algorithms, and the defaults depend on
the underlying security provider:

* `netty-tcnative` 4.1 (w/ BoringSSL) - `{ecdsa_secp256r1_sha256, rsa_pss_rsae_sha256, rsa_pkcs1_sha256, ecdsa_secp384r1_sha384, rsa_pss_rsae_sha384, rsa_pkcs1_sha384, rsa_pss_rsae_sha512, rsa_pkcs1_sha512, rsa_pkcs1_sha1}`
* `netty-tcnative` 4.2 (w/ BoringSSL) - Supports `mldsa65`, `mldsa87`, and `mldsa44` when explicitly configured via `SSLParameters.setSignatureSchemes` / `SSL_CTX_set1_sigalgs_list`, while defaulting to classical verification signature schemes until BoringSSL updates `kVerifySignatureAlgorithms`.
* OpenJDK 24+ added `ML-DSA` cryptographic primitives ([JEP 497](https://openjdk.org/jeps/497)), and OpenJDK 27+ supports `ML-DSA` in TLS 1.3 by default - the default TLS 1.3 list begins with `{mldsa65, mldsa87, mldsa44, ecdsa_secp256r1_sha256, ecdsa_secp384r1_sha384, ecdsa_secp521r1_sha512, ed25519, ed448, rsa_pss_rsae_sha256, rsa_pss_rsae_sha384, rsa_pss_rsae_sha512, rsa_pss_pss_sha256, rsa_pss_pss_sha384, rsa_pss_pss_sha512}`.
* Conscrypt does not currently support `ML-DSA` in TLS 1.3 handshakes.

In Java 19+, the `SSLParameters.setSignatureSchemes(String[] signatureSchemes)`
method allows gRPC to create a passthrough API that lets users configure their
signature algorithms directly using standard IANA / JSSE scheme names (e.g.,
`"mldsa65"`, `"mldsa87"`, `"ecdsa_secp256r1_sha256"`, `"rsa_pss_rsae_sha256"`).

A field will be added to both `TlsChannelCredentials` and
`TlsServerCredentials`, along with a new `Feature.SIGNATURE_ALGORITHMS` enum
value on each. This will pass through to `SSLParameters.setSignatureSchemes`.

Java API Changes:
```java
public final class TlsChannelCredentials extends ChannelCredentials {
  // ...
  /**
   * Returns an immutable list of preferred signature algorithm names (e.g.,
   * "mldsa65", "mldsa87", "ecdsa_secp256r1_sha256"), or {@code null} if the
   * provider defaults should be used.
   */
  public List<String> getSignatureAlgorithms() {
    return signatureAlgorithms;
  }

  public enum Feature {
    FAKE,
    MTLS,
    CUSTOM_MANAGERS,
    /**
     * Signature algorithms may be specified to restrict and prioritize TLS
     * handshake signature schemes. Requires observing {@link
     * #getSignatureAlgorithms()}.
     */
    SIGNATURE_ALGORITHMS,
    ;
  }

  public static final class Builder {
    // ...
    /**
     * Sets the preferred TLS signature algorithms in order of preference.
     * Passes through to {@link javax.net.ssl.SSLParameters#setSignatureSchemes}.
     */
    public Builder signatureAlgorithms(String... signatureAlgorithms) { ... }
  }
}

public final class TlsServerCredentials extends ServerCredentials {
  // ...
  public List<String> getSignatureAlgorithms() {
    return signatureAlgorithms;
  }

  public enum Feature {
    FAKE,
    MTLS,
    CUSTOM_MANAGERS,
    SIGNATURE_ALGORITHMS,
    ;
  }

  public static final class Builder {
    // ...
    public Builder signatureAlgorithms(String... signatureAlgorithms) { ... }
  }
}
```

### Temporary environment variable protection

No temporary environment variable protection is required. This change introduces
an opt-in configuration API. We will change the BoringSSL supported list by
default to match the modern behavior of other libraries, and users can opt-out
of that if desired.

## Rationale

Enabling post-quantum signature algorithms and
providing a consistent cross-language configuration API allows gRPC users to
begin deploying and testing post-quantum PKI hierarchies well ahead of the
Eo2028 CRQC horizon, without disrupting existing classical deployments.

### Why ML-DSA-65 and ML-DSA-87?

`ML-DSA` (Module-Lattice-Based Digital Signature Standard, standardized by NIST
in [FIPS 204](https://csrc.nist.gov/pubs/fips/204/final) from CRYSTALS-Dilithium)
is the primary post-quantum digital signature algorithm standardized by NIST and
the IETF TLS working group ([`draft-ietf-tls-mldsa`](https://datatracker.ietf.org/doc/html/draft-ietf-tls-mldsa)).

Specifically:
* **`ML-DSA-65`** provides NIST Security Category 3 strength
  (comparable to AES-192 / SHA-384) with a 1,952-byte public key and 3,309-byte
  signature, serving as the general-purpose industry default for TLS 1.3.
* **`ML-DSA-87`** provides NIST Security Category 5 strength
  (comparable to AES-256 / SHA-512) with a 2,592-byte public key and 4,627-byte
  signature, and is mandated by CNSA 2.0 for high-security environments.

Unlike key exchange—where ephemeral hybrid constructions (`X25519-MLKEM768`) are
trivial to negotiate within a single handshake without changing certificate
provisioning—hybrid/composite X.509 certificates and dual-certificate TLS
extensions (`draft-ietf-tls-tlsflags`) are still maturing across PKI tooling and
TLS implementations. Supporting pure `ML-DSA-65` and `ML-DSA-87` certificates
first aligns with what BoringSSL, OpenSSL 3.5+, Go 1.27+, and OpenJDK 27+
implement today, while the `signature_algorithms` API naturally accommodates
additional signature schemes as they are standardized.

## Performance Analysis

MLDSA keys and signatures are significantly larger than classical signatures. This means certificates will increase in size.

| Algorithm / Parameter Set | Security Profile | Public Key Size | Signature Size | Total Overhead (Key + Sig) | Size Increase vs. ECDSA P-256 |
| :--- | :--- | :---: | :---: | :---: | :---: |
| **ECDSA (P-256)** | Classical (128-bit) | 64 B (Uncompressed) | 64 B | 128 B | Baseline |
| **RSA-2048** | Classical (112-bit) | 256 B | 256 B | 512 B | ~4× larger |
| **RSA-3072** | Classical (128-bit) | 384 B | 384 B | 768 B | ~6× larger |
| **ML-DSA-44** | NIST Level 2 (128-bit) | 1,312 B | 2,420 B | 3,732 B | ~29× larger |
| **ML-DSA-65** | NIST Level 3 (192-bit) | 1,952 B | 3,309 B | 5,261 B | ~41× larger |
| **ML-DSA-87** | NIST Level 5 (256-bit) | 2,592 B | 4,627 B | 7,219 B | ~56× larger |

MLDSA signatures are also more costly to compute. The below performance benchmarks will be added as support is added in gRPC.

### C++ Performance Comparison

| Benchmark | Real Time / Op | CPU Time / Op | Instructions / Op | Peak Mem (Bytes) | Allocs / Op |
| :---- | :---: | :---: | :---: | :---: | :---: |
| BM_TlsConnection (Default ECDSA P-256) | TBD | TBD | TBD | TBD | TBD |
| BM_TlsConnection_MLDSA65 | TBD | TBD | TBD | TBD | TBD |
| BM_TlsConnection_MLDSA87 | TBD | TBD | TBD | TBD | TBD |
| | | | | | |
| BM_MtlsConnection (Default ECDSA P-256) | TBD | TBD | TBD | TBD | TBD |
| BM_MtlsConnection_MLDSA65 | TBD | TBD | TBD | TBD | TBD |
| BM_MtlsConnection_MLDSA87 | TBD | TBD | TBD | TBD | TBD |