A118: Authentication Telemetry
----
* Author(s): @gtcooke94
* Approver: @dfawley, @easwars, @ejona86, @mattstev, @markdroth
* Status: In Review
* Implemented in:
* Last updated: 2026-05-18
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

gRPC's authentication stack has no telemetry. This document details adding
non-per-call metrics to the TLS authentication stack. 

## Background

gRPC's authentication stack currently lacks telemetry. This document outlines
the addition of authentication-related non-per-call metrics, leveraging the
existing telemetry infrastructure defined in
[A66](https://github.com/grpc/proposal/blob/master/A66-otel-stats.md) and the
specific architecture for non-per-call metrics established in
[A79](https://github.com/grpc/proposal/blob/master/A79-non-per-call-metrics-architecture.md).
Because authentication and handshakers are connection-level abstractions, they
inherently require non-per-call instrumentation. Currently, application owners
face challenges in diagnosing handshake failures, relying on verbose logging
without a structured mechanism for aggregation and analysis.

### Related Proposals:
* [A66: OpenTelemetry Metrics/Stats](https://github.com/grpc/proposal/blob/master/A66-otel-stats.md)
* [A79: OpenTelemetry Non-Per-Call Metrics Architecture](https://github.com/grpc/proposal/blob/master/A79-non-per-call-metrics-architecture.md)

## Proposal

All metrics will be scoped to TLS exclusively.

### TLS Telemetry Result Enum

The handshaker status will be represented by an enum that indicates success or
provides information on why the handshake failed. This value must manage a
balance of low-cardinality while being fine-grained enough to be useful;
therefore, an enum containing subdomains of authentication errors will be
created. This is presented as a C++ enum below, but will be identical in all
languages. In cases where we cannot categorize an error or cannot get enough
granularity in a given implementation and/or language, `UNKNOWN_FAILURE` will be
the catch-all error code.

```c++
enum class TlsTelemetryResult {
  UNKNOWN_FAILURE,  
  SUCCESS,
  // Peer certificate verification failures.
  CERTIFICATE_VERIFICATION_FAILED,
  CERTIFICATE_REVOKED,
  CERTIFICATE_EXPIRED,
  CERTIFICATE_NOT_YET_VALID,
  CERTIFICATE_AUTHORITY_INVALID,
  // TLS negotiation mismatch failures
  CERTIFICATE_HOSTNAME_MISMATCH,
  CERTIFICATE_MALFORMED,
  CIPHER_SUITE_MISMATCH,
  PROTOCOL_VERSION_UNSUPPORTED,
  INAPPROPRIATE_FALLBACK,
  NO_APPLICATION_PROTOCOL,
  // Cryptographic failures
  SIGNATURE_VERIFICATION_FAILED,
  DECRYPTION_FAILED,
  KEY_EXCHANGE_FAILURE,
  // Other failures
  UNEXPECTED_MESSAGE,
  HANDSHAKE_TIMEOUT,
  PEER_CONNECTION_CLOSED
};
```

### TLS Resumption Type Enum

Further, whether the handshake is resumed or not is also critical for
understanding authentication behavior. An enum describing the type of resumption
used (or none) will be created - it can be extended in the future to make
finer-grained distinctions between the type of resumption that is used (e.g.,
ticket-based resumption vs. session-based resumption). This is presented as a
C++ enum below, but will be identical in all languages.

```c++
enum class TlsResumptionType {
  FULL_HANDSHAKE,
  RESUMED_HANDSHAKE,
};
```

### Metrics Definitions

The following metrics are count metrics, with the primary information coming
from the labels.

* `grpc.client.tls.handshakes`

| Label Name | Required/Optional | Description |
| :--- | :--- | :--- |
| `grpc.tls.handshake.result` | Required | The `TlsTelemetryResult` enum indicating success or the reason for handshake failure |
| `grpc.target` | Required | The target string (as defined in A66) passed to the channel. |
| `grpc.tls.handshake.resumed` | Optional | The `TlsResumptionType` enum |

* `grpc.server.tls.handshakes`

| Label Name | Required/Optional | Description |
| :--- | :--- | :--- |
| `grpc.tls.handshake.result` | Required | The `TlsTelemetryResult` enum indicating success or the reason for handshake failure |
| `grpc.tls.handshake.resumed` | Optional | The `TlsResumptionType` enum |

### TLS Offload Specific Metrics

The following metrics are non-per-call bucketed latency metrics that report the duration of offloaded cryptographic operations.

* `grpc.server.tls.offload_certificate_selection_duration` (unit: float64, type: histogram - latency buckets defined in A66)

| Label Name | Required/Optional | Description |
| :--- | :--- | :--- |
| `grpc.status` | Required | Result of the certificate selection offloading, in the format of a gRPC status code (as defined in A66). |

Note - there is no associated client certificate selection metric. This is a
server specific feature.

* `grpc.client.tls.offload_private_key_operation_duration` (unit: float64, type: histogram - latency buckets defined in A66)

| Label Name | Required/Optional | Description |
| :--- | :--- | :--- |
| `grpc.status` | Required | Result of the certificate selection offloading, in the format of a gRPC status code (as defined in A66). |
| `grpc.target` | Required | The target string (as defined in A66) passed to the channel. |
| `grpc.tls.private_key_algorithm` | Optional | An algorithm enum indicating how the offloaded private key operation was done, e.g. “RsaPkcs1Sha256”. |

* `grpc.server.tls.offload_private_key_operation_duration` (unit: float64, type: histogram - latency buckets defined in A66)

| Label Name | Required/Optional | Description |
| :--- | :--- | :--- |
| `grpc.status` | Required | Result of the certificate selection offloading, in the format of a gRPC status code (as defined in A66). |
| `grpc.tls.private_key_algorithm` | Optional | An algorithm enum indicating how the offloaded private key operation was done, e.g. “RsaPkcs1Sha256”. |

### Temporary environment variable protection

This feature will be explicitly configured by users, thus no environment
variable protection is needed. If a user does not configure TLS metrics or
offloading, telemetry won't be collected under this mechanism unless general
telemetry/stats plugins are active.

## Rationale

The alternative of splitting generic handshaker metrics and TLS-specific metrics
was considered, but opened many rabbit holes as to what would qualify as a
handshaker (e.g. are HTTP Connect, TCP, RPC Switch, etc. all in scope, or just
alternative protocols to TLS such as ALTS). Thus, the decision was made to scope
the metrics to TLS.

## Implementation

### C++

In the C++ implementation, the transport security interface (TSI) has
historically been decoupled from gRPC. However, this design choice has been
broken over time, and the two have been coupled for years now. The reasons
behind decoupling TSI and gRPC are no longer relevant, therefore we will fully
accept this coupling. Thus, the TSI code can contain gRPC monitoring specifics.
In the few use-cases where TSI is not called via gRPC, we will ensure that
metric incrementation is not performed.

We will add the Channel's `StatsPluginGroup` as an optional argument to TSI
handshaker creation functions. This ensures that we don't break any existing
users of TSI and that we never increment metrics when TSI is used outside of
gRPC. When TSI is called from gRPC, we will pass this argument. This will be
stored on the handshaker, and in `ssl_transport_security.cc` we will access this
from the handshaker to increment metrics.

```diff
 tsi_result tsi_ssl_client_handshaker_factory_create_handshaker(
     tsi_ssl_client_handshaker_factory* factory,
     const char* server_name_indication, size_t network_bio_buf_size,
     size_t ssl_bio_buf_size,
     std::optional<std::string> alpn_preferred_protocol_list,
-    tsi_handshaker** handshaker);
+    tsi_handshaker** handshaker,
+    std::shared_ptr<grpc_core::GlobalStatsPluginRegistry::StatsPluginGroup> stats_plugin_group = nullptr);

```

```diff
 tsi_result tsi_ssl_server_handshaker_factory_create_handshaker(
     tsi_ssl_server_handshaker_factory* factory, size_t network_bio_buf_size,
-    size_t ssl_bio_buf_size, tsi_handshaker** handshaker);
+    size_t ssl_bio_buf_size, tsi_handshaker** handshaker,
+ std::shared_ptr<grpc_core::GlobalStatsPluginRegistry::StatsPluginGroup> stats_plugin_group = nullptr);
```
### Golang

For the implementation of the general handshake metric, we leverage gRPC-Go's existing transport-level architecture.

The abstraction layer for general handshake information is the Transport
Connection Layer (`internal/transport`). Client Handshakes are performed in a
single place: `internal/transport/http2_client.go` inside `NewHTTP2Client` via
`transportCreds.ClientHandshake`. Server Handshakes are also performed in a
single place: `internal/transport/http2_server.go` inside `NewServerTransport`
via `config.Credentials.ServerHandshake`. We can incremement the metrics here
only in the case where TLS is the protocol being used.

```diff
--- a/internal/transport/http2_client.go
+++ b/internal/transport/http2_client.go
@@ -294,7 +294,23 @@
 	if transportCreds != nil {
+		isTLS := transportCreds.Info().SecurityProtocol == "tls"
+		var startTime time.Time
+		if isTLS {
+			startTime = time.Now()
+		}
 		conn, authInfo, err = transportCreds.ClientHandshake(connectCtx, addr.ServerName, conn)
+		if isTLS {
+			duration := time.Since(startTime).Seconds()
+     <increment metric>
+		}
+
 		if err != nil {
 			return nil, connectionErrorf(isTemporary(err), err, "transport: authentication handshake failed: %v", err)
 		}

```

To pass out resumption information, we will need to augment the [`TLSInfo`](https://github.com/grpc/grpc-go/blob/660208049b96ff6232e8c7212905b3e357b5bf42/credentials/tls.go#L41) to include [`ConnectionState.DidResume`](https://github.com/golang/go/blob/e62d3e6e897175a07aa44a7b2c7f99700072f22f/src/crypto/tls/common.go#L257).

The TLS specific offload metrics will go with their implementation. This feature is not yet written in Go, so we cannot discuss specific metric implementation details.

### Java

In gRPC-Java, to support shading where core classes are relocated per-consumer, all metric instruments used in core/ or transport modules (like netty/) must be defined within the api/ module.

Following the pattern established by `MIN_RTT_INSTRUMENT` in `InternalTcpMetrics.java`, we will:

1. Define a new `InternalSecurityMetrics.java` class in the api/ module under the `io.grpc` package.  
2. Expose `MetricRecorder` from `GrpcHttp2ConnectionHandler` in the netty/ module.  
3. Instrument the Netty `ClientTlsHandler` and `ServerTlsHandler` in `ProtocolNegotiators.java` to track handshake duration and record it.  

The TLS specific offload metrics will go with their implementation. This feature is not yet written in Java, so we cannot discuss specific metric implementation details.

### Wrapped Languages

Wrapped languages that support non-per-call metrics should be able to get this "for free" from the Core implementation. However, Python for example, does not currently support non-per-call metrics.
