A120 - Post-Quantum Cryptography
----
* Author(s): @gtcooke94
* Approver: markdroth
* Implemented in: C++, Go, Java
* Last updated: 2026-06-05
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

The industry's current best estimates are that a cryptographically-relevant quantum computer (CRQC) will be available by Eo2028. Prior to recent quantum computing advances announced on [March 30th, 2026](http://arxiv.org/abs/2603.28627), and [March 31st, 2026](https://research.google/blog/safeguarding-cryptocurrency-by-disclosing-quantum-vulnerabilities-responsibly/?utm_source=cryptography-dispatches&utm_medium=email&utm_campaign=crqc-timeline), the industry consensus was that the date for a CRQC would be 2030+, thus the recent and sudden urgency. Anyone harvesting encrypted data for store-now-decrypt-later (SNDL) attacks will be able to break the current non-quantum ciphers in ~2.5 years. gRPC must change its defaults to leverage post-quantum cryptography (PQC) as soon as possible. Particularly, in the immediate future, gRPC must ensure post-quantum confidentiality in its SSL/TLS stacks by preferring the X25519-MLKEM768 key exchange algorithm.

This document is specifically scoped to post-quantum confidentiality, which is the urgent deliverable. Post-quantum authenticity is out of scope for now and will follow in late 2026 or early 2027.

## Background

gRPC’s TLS stack now enables users to configure their own ciphersuites (for example, [this PR contains the C++ API and implementation](https://github.com/grpc/grpc/pull/41870)). This was a prerequisite to changing the default behavior for gRPC, as we want users to be able to opt-out of post-quantum confidentiality if they desire.

Maintainers of the individual pieces of the general TLS ecosystem are in consensus that moving to PQC is of the highest priority. The general consensus is to secure quickly and optimize for performance later.

Performance concerns are the primary source of anticipated pushback. However, we align with the broader ecosystem: prioritizing rapid PQC readiness is critical. Performance-sensitive users that cannot take this approach, can choose to opt-out by configuring their ciphers. In addition, gRPC SSL/TLS users are not historically sensitive to ~600 μs handshake latency delta. For example, gRPC-C++ does not enable TLS resumption by default, which would save much more than that.

## Proposal

The gRPC TLS stacks must prefer the X25519-MLKEM768 key exchange group ASAP. The current defaults vary per language. Notably, this is a **prefer**, not a **require**, so we can allow for graceful fallback from a PQC-safe cipher to the previous defaults (e.g. to allow interop with old applications). In the TLS handshake, key exchange groups are negotiated as follows, per [RFC8446](https://datatracker.ietf.org/doc/html/rfc8446#section-4.2.7): the client sends an ordered list of key exchange groups that it supports, and the server selects a key exchange group from that list. The RFC does not specify whether the client or server preference should "win". In BoringSSL, the server selects the first item in the client list that it supports. However, in Go, the server preferences are preferred. 

In the case where the security library is already preferring PQC, we will not take over the defaults. In the case that the underlying security libraries are not already preferring PQC, gRPC will take an opinionated default of {X25519-MLKEM768, x25519, secp256r1, secp384r1, secp521r1}.

### Why X25519-MLKEM768?

X25519-MLKEM768 is a hybrid algorithm. Specifically, it combines the classical X25519 cipher and the post-quantum MLKEM768 cipher. Think of the two ciphers as two separate "locks" on the data - an adversary has to break both locks individually to capture the data. We have decades of confidence in classical cryptography protecting against normal computing, so we keep that and add post-quantum protections on top. In this way, gRPC traffic is still protected against potentially unknown classical attacks against the post-quantum MLKEM768 cipher. This choice is consistent with what the rest of the industry is doing (e.g. across the broader TLS ecosystem and the Web).

As to why we select the X25519-MLKEM768 combination specifically, simply put, it is ready, whereas the CNSA 2.0 recommendation (X25519-MLKEM1024) is not available in any TLS libraries at present. We plan to add support for X25519-MLKEM1024 when it is available.

### C++

In OpenSSL < 3 (and thus BoringSSL), if no key exchange groups are set gRPC currently defaults as follows:

| BoringSSL (1.1.1) | P-256 | gRPC Default |
| :---- | :---- | :---- |
| OpenSSL < 3 | P-256 | gRPC Default |
| 3 <= OpenSSL < 3.5 | {x25519, P-256, x448, ... many more ...} | [OpenSSL Default](https://github.com/openssl/openssl/blob/89cd17a031e022211684eb7eb41190cf1910f9fa/ssl/t1_lib.c#L195-L213) |
| OpenSSL >= 3.5 | {X25519MLKEM768, X25519, P-256, X448, P-384, P-521, ffdhe2048, ffdhe3072} | [OpenSSL Default](https://github.com/openssl/openssl/blob/286ddeaac037533bbdce65b3c689e3f7ffebf0f6/ssl/t1_lib.c#L204) |

The following table describes what the default behavior should be for each version. MLKEM768 is only supported in BoringSSL and OpenSSL3.5+. Thus, for all OpenSSL < 3.5, we will keep the P-256 default.

| BoringSSL (1.1.1) | {X25519MLKEM768, X25519, P-256, P-384, P-521} with SSL_CTX_set1_groups_list (Until the better defaults in BoringSSL) |
| :---- | :---- |
| OpenSSL < 3 | {X25519, P-256, P-384, P-521} with SSL_CTX_set1_groups/SSL_CTX_set1_curves |
| OpenSSL 3+ | (Use OpenSSL defaults) < 3.5 {X25519, P-256, P-384, P-521} 3.5+ {X25519MLKEM768, …} |

#### Performance Comparison

The C++ gRPC benchmark suite evaluates connection setup overhead:

| Benchmark | Real Time / Op | CPU Time / Op | Instructions / Op | Peak Mem (Bytes) | Allocs / Op |
| :---- | :---: | :---: | :---: | :---: | :---: |
| BM_TlsConnection (Default) | 965.7 µs ± 2% | 1.291 ms ± 6% | 96.15k ± 1% | 741.1k ± 9% | 2.715k ± 5% |
| BM_TlsConnection_X25519 (Forced X25519) | 931.0 µs ± 2% | 1.258 ms ± 2% | 96.12k ± 0% | 741.2k ± 9% | 2.697k ± 4% |
| BM_TlsConnection_X25519_MLKEM768 (Hybrid) | 1.042 ms ± 1% | 1.368 ms ± 2% | 96.16k ± 0% | 741.6k ± 9% | 2.706k ± 4% |
| | | | | | |
| BM_MtlsConnection (Default) | 1.171 ms ± 1% | 1.513 ms ± 3% | 96.17k ± 0% | 751.9k ± 2% | 3.077k ± 0% |
| BM_MtlsConnection_X25519 (Forced X25519) | 1.138 ms ± 2% | 1.474 ms ± 1% | 96.18k ± 0% | 755.6k ± 8% | 3.059k ± 3% |
| BM_MtlsConnection_X25519_MLKEM768 (Hybrid) | 1.248 ms ± 1% | 1.580 ms ± 1% | 96.25k ± 0% | 754.6k ± 9% | 3.064k ± 3% |

### Go

In Golang, this is configured via [tls.Config.CurvePreferences](https://pkg.go.dev/crypto/tls#Config). However, `advancedtls` **does not** currently support the user passing through `CurvePreferences`.

This means grpc-go is fully dependent on the Golang defaults, shown in the below codeblock.

```go
// From standard Go crypto/tls package

// defaultCurvePreferences is the default set of supported key exchanges, as
// well as the preference order.
func defaultCurvePreferences() []CurveID {
	switch {
	// tlsmlkem=0 restores the pre-Go 1.24 default.
	case tlsmlkem.Value() == "0":
		return []CurveID{X25519, CurveP256, CurveP384, CurveP521}
	// tlssecpmlkem=0 restores the pre-Go 1.26 default.
	case tlssecpmlkem.Value() == "0":
		return []CurveID{X25519MLKEM768, X25519, CurveP256, CurveP384, CurveP521}
	default:
		return []CurveID{
			X25519MLKEM768, SecP256r1MLKEM768, SecP384r1MLKEM1024,
			X25519, CurveP256, CurveP384, CurveP521,
		}
	}
}
```

| Go < 1.24 | `{X25519, CurveP256, CurveP384, CurveP521}` |
| :---- | :---- |
| 1.24 <= Go < 1.26 | `{X25519MLKEM768, X25519, CurveP256, CurveP384, CurveP521}` |
| Go >= 1.26 | `{X25519MLKEM768, SecP256r1MLKEM768, SecP384r1MLKEM1024, X25519, CurveP256, CurveP384, CurveP521}` |

We will add a passthrough setting in `advancedtls` for the user to explicitly configure `CurvePreferences`. It will be `advancedtls.Options.CurvePreferences` and will be a simple pass through to the `tls.Config`.

#### Performance Comparison

The Go gRPC credentials benchmark evaluates connection latency and allocation overhead:

| Benchmark | Latency / Connection | Delta vs. Std TLS (%) | Allocations / Connection | Memory (KiB) / Connection |
| :---- | :---: | :---: | :---: | :---: |
| **Standard TLS** *(BenchmarkTlsConnection)* | **819.9 µs ± 0%** | - | **1,681 ± 0%** | **197.5 KiB ± 0%** |
| **Standard mTLS** *(BenchmarkMtlsConnection)* | **1.013 ms ± 0%** | **+23.5%** | **1,901 ± 0%** | **223.0 KiB ± 0%** |
| **PQ TLS** *(BenchmarkTlsConnection_X25519_MLKEM768)* | **1.060 ms ± 0%** | **+29.3%** | **1,700 ± 0%** | **243.7 KiB ± 0%** |
| **PQ mTLS** *(BenchmarkMtlsConnection_X25519_MLKEM768)* | **1.248 ms ± 0%** | **+52.2%** | **1,918 ± 0%** | **266.5 KiB ± 0%** |

### Java

Similar to Go, gRPC-Java does not currently make opinionated preferences on the key exchange algorithm, and the defaults are dependent upon the security provider used. The providers have significant differences and this creates confusion for users. Further, X25519-MLKEM768 is not supported in all java versions, nor in all security providers. We propose that we update gRPC-Java to use netty-tcnative 4.2 (w/ BoringSSL) to cover most important use cases. Otherwise, OpenJDK 27 is the first version in which X25519-MLKEM768 is supported, so we are constrained by the language.

* netty-tcnative 4.1 (w/ BoringSSL) - {x25519, secp256r1, secp384r1, secp521r1}
* netty-tcnative 4.2 (w/ BoringSSL) - {X25519MLKEM768, x25519, secp256r1, secp384r1, secp521r1}
* OpenJDK 27+ post-quantum is supported by default - the exact list is {X25519MLKEM768, x25519, secp256r1, secp384r1, secp521r1, x448, ffdhe2048, ffdhe3072, ffdhe4096, ffdhe6144, ffdhe8192}
  * See [JEP 527](https://openjdk.org/jeps/527) for more detail.
* Conscrypt does not support X25519MLKEM768 (no recent build with newer BoringSSL)

In Java 21+, `SSLParameters.setNamedGroups` option will allow a user to configure their algorithms directly. We will ensure APIs exist for users to configure this where supported.

```java
// Get current SSLParameters
SSLParameters params = socket.getSSLParameters();

// Set the prioritized named groups, including the hybrid PQ-scheme
params.setNamedGroups(new String[] { 
    "X25519MLKEM768", // Post-quantum hybrid (preferred)
    "x25519",         // Classical fallback
    "secp256r1" 
});

// Update the socket with the new parameters
socket.setSSLParameters(params);
```

#### Performance Comparison

The Java gRPC credentials benchmark evaluates connection setup latency:

| Benchmark | Latency / Connection (us) | Delta vs. Std TLS (%) |
| :---- | :---: | :---: |
| **Standard TLS** *(BenchmarkTlsConnection)* | **820.92 µs** | - |
| **PQ TLS** *(BenchmarkTlsConnection_X25519_MLKEM768)* | **931.81 µs** | **+13.5%** (+110.89 µs) |
| **Standard mTLS** *(BenchmarkMtlsConnection)* | **1,168.98 µs** | **+42.4%** (+348.06 µs) |
| **PQ mTLS** *(BenchmarkMtlsConnection_X25519_MLKEM768)* | **1,287.70 µs** | **+56.8%** (+466.78 µs) |

### Temporary environment variable protection

No temporary environment variable protection is required. This change prefers X25519-MLKEM768 by default but allows graceful fallback, and users can opt-out entirely by configuring their ciphersuites or curve preferences.

## Rationale

Prioritizing rapid PQC readiness is critical as SNDL attacks present an immediate threat. Preferring X25519-MLKEM768 provides hybrid post-quantum confidentiality while preserving decades of classical security guarantees and allowing graceful fallback for legacy interop.

## Implementation

* C++: Keep P-256 default for OpenSSL < 3.5; prefer X25519-MLKEM768 by default for BoringSSL and OpenSSL 3.5+.
* Go: Add `advancedtls.Options.CurvePreferences` passthrough to `tls.Config`.
* Java: Update gRPC-Java to use netty-tcnative 4.2 (w/ BoringSSL) and ensure `SSLParameters.setNamedGroups` APIs exist for user configuration where supported.


