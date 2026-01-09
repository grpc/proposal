HTTPS Proxy Support
----
* Author(s): Jonas Wustrack (jonas.wustrack@orange.com)
* Approver: TBD
* Status: Draft
* Implemented in: C-core (proposed)
* Last updated: 2026-01-09
* Discussion at: TBD (filled after thread exists)

## Abstract

This proposal extends [gRFC A1: HTTP CONNECT Proxy Support](https://github.com/grpc/proposal/blob/master/A1-http-connect-proxy-support.md) to add support for HTTPS proxies, where the connection to the proxy itself is TLS-encrypted. This protects proxy authentication credentials (such as BasicAuth username and password) from eavesdropping on the network path between the client and proxy.

## Background

[gRFC A1](https://github.com/grpc/proposal/blob/master/A1-http-connect-proxy-support.md) defined how gRPC supports TCP-level proxies via the HTTP CONNECT request. That proposal uses the `http_proxy` environment variable (or equivalent mechanisms) to configure proxy usage.

However, A1 only supports plain HTTP connections to the proxy. When using BasicAuth credentials with the proxy (via the `Proxy-Authorization` header), these credentials are transmitted in plaintext over the network. In security-conscious environments, this is unacceptable as it exposes credentials to potential eavesdroppers.

Many proxy deployments support HTTPS connections, where the client establishes a TLS connection to the proxy before sending the HTTP CONNECT request. This ensures that:
1. Proxy authentication credentials are encrypted
2. The target hostname in the CONNECT request is not visible to network observers
3. The proxy can be authenticated via its TLS certificate

### Related Proposals

- [gRFC A1: HTTP CONNECT Proxy Support](https://github.com/grpc/proposal/blob/master/A1-http-connect-proxy-support.md)

## Proposal

This proposal adds HTTPS proxy support to gRPC by:
1. Recognizing the `https://` scheme in proxy URIs
2. Performing a TLS handshake with the proxy before sending the HTTP CONNECT request
3. Providing configuration options for proxy TLS settings

### URI Scheme Detection

When parsing the proxy URI from:
- The `GRPC_ARG_HTTP_PROXY` channel argument
- The `grpc_proxy`, `https_proxy`, or `http_proxy` environment variables

If the scheme is `https://`, gRPC will establish a TLS connection to the proxy before proceeding with the HTTP CONNECT handshake.

Example:
```
# HTTP proxy (existing behavior)
export http_proxy="http://proxy.example.com:8080"

# HTTPS proxy (new behavior)
export https_proxy="https://username:password@proxy.example.com:8443"
```

### Configuration Options

Each gRPC implementation has its own configuration mechanism. This section defines the **semantic configuration options** that all implementations should support, while allowing each language to expose these through its native API patterns.

#### Required Configuration Capabilities

All implementations MUST support the following configuration capabilities:

| Capability | Description | Default |
|------------|-------------|---------|
| HTTPS proxy detection | Detect `https://` scheme in proxy URI | Auto-detect |
| Root CA certificates | Custom root certificates for proxy verification | System defaults |
| Server certificate verification | Enable/disable proxy certificate verification | Enabled |
| Server name override | Override expected server name for verification | Proxy hostname |
| Client certificate (mTLS) | Client certificate for mutual TLS with proxy | None |
| Client private key (mTLS) | Client private key for mutual TLS with proxy | None |

### Language-Specific Implementation

Since grpc-java and grpc-go are independent implementations (not wrapped around C-core), each language defines its own API for configuration.

#### C-core (and wrapped languages: Python, Ruby, PHP, etc.)

C-core uses **channel arguments** for configuration, which are automatically available to all wrapped languages.

**Channel Arguments:**
| Channel Argument | Type | Description |
|------------------|------|-------------|
| `grpc.http_proxy_tls_enabled` | Boolean | Explicitly enable/disable TLS to proxy |
| `grpc.http_proxy_tls_root_certs` | String (PEM) | Root CA certificates for verifying proxy |
| `grpc.http_proxy_tls_verify_server_cert` | Boolean | Whether to verify proxy certificate |
| `grpc.http_proxy_tls_server_name` | String | Expected server name for verification |
| `grpc.http_proxy_tls_cert_chain` | String (PEM) | Client certificate for mTLS |
| `grpc.http_proxy_tls_private_key` | String (PEM) | Client private key for mTLS |

**Implementation:**
1. The `HttpProxyMapper` detects the `https://` scheme and sets `grpc.http_proxy_tls_enabled` to `true`
2. The `HttpsProxyTlsHandshakerFactory` creates a TLS handshaker when this flag is set
3. The handshaker chain becomes: TCP → HTTPS Proxy TLS → HTTP CONNECT → Target TLS

#### grpc-java and grpc-go

These implementations do NOT share configuration mechanisms with C-core. They will need to provide language-specific APIs to configure HTTPS proxy TLS settings, following their existing patterns for proxy and TLS configuration (e.g., `ProxyDetector` in Java, `DialOption` in Go).

## Rationale

### Why Not Reuse Target TLS Credentials?

The proxy and target server are separate entities with potentially different:
- Certificate authorities (enterprise proxy CA vs. public CA)
- Authentication requirements (mTLS for proxy, none for target)
- Certificate verification policies

Therefore, separate TLS configuration is necessary for proxy connections.

### Why Language-Specific APIs?

Unlike some gRPC features that are implemented in C-core and wrapped by other languages (Python, Ruby, PHP), **grpc-java and grpc-go are independent implementations**. They do not use C-core's channel arguments system. Therefore:

1. Each language must define its own native API for configuration
2. The gRFC specifies **semantic requirements** (what capabilities must be supported) rather than a single API
3. Each implementation should follow its language's idioms (e.g., `DialOption` in Go, builder pattern in Java)

This approach ensures that:
- Users get a natural API for their language
- Implementations can leverage language-specific TLS libraries
- The feature can be implemented without artificial constraints

### Why Not a Separate Credentials Object?

An alternative would be to create a new `HttpProxyCredentials` type in each language. This proposal leaves that decision to each implementation, as:
- Some languages may prefer extending existing proxy configuration mechanisms
- Others may benefit from a dedicated credentials type
- The key requirement is supporting the defined capabilities, not a specific API shape

## Implementation

### Phase 1: C-core Implementation
- Implement HTTPS proxy TLS handshaker
- Add channel arguments for proxy TLS configuration
- Update HTTP proxy mapper to detect `https://` scheme
- Add unit and integration tests

### Phase 2: grpc-java Implementation
- Extend ProxyDetector for HTTPS proxy support
- Implement proxy TLS wrapper
- Add configuration via system properties or channel credentials

### Phase 3: grpc-go Implementation
- Extend proxy dialer for HTTPS support
- Add DialOption for proxy TLS configuration
- Add tests

### Phase 4: Documentation and Validation
- Update documentation across all languages
- Mark feature as stable

## Open issues (if applicable)

N/A
