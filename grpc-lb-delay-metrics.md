# RPC Delay Observability
----
* Author(s): Madhav Bissa (@madhavbissa)
* Approver: markdroth
* Implemented in: Go, Java, C++
* Last updated: 2026-06-19
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

This proposal introduces client-side metrics and tracing to measure the delays an RPC experiences within the client channel before it is sent over the network. In this context, "delay" refers to the time an RPC spends blocked or queued inside the channel waiting for name resolution, configuration parsing, or downstream connection establishment. To expose these states, this document proposes two new histograms, two dedicated tracing spans, and corresponding additions to the CallTracer and CallAttemptTracer APIs.


## Background

Existing gRPC core telemetry infrastructure, as defined in [gRPC A66 (OpenTelemetry Metrics)][A66] and [gRPC A72 (OpenTelemetry Tracing)][A72], tracks the overall end-to-end duration of RPC calls and attempts. However, these metrics and spans function as aggregate buckets that do not decompose latency, leaving delays inside the client channel invisible to operators.

Before an RPC attempt can be sent over the network, the client channel must perform several critical operations, including resolving the target name, parsing service configurations, instantiating load balancing policies, and obtaining a connectivity picker. If any of these phases stall—such as during a slow DNS lookup, a Route Lookup Service (RLS) control-plane query, or a Cluster Discovery Service (CDS) metadata fetch—the RPC is delayed. To the application, this appears as high latency or a timeout. However, because current telemetry lacks visibility into these resolution and routing states, developers cannot distinguish between a slow network, a slow backend, or a channel initialization delay. This is particularly challenging for clients utilizing the Route Lookup Service (RLS), where diagnosing RLS-related hangs or isolating whether delays stem from pending route lookups requires complex manual debugging.

While there are many potential sources of delay along the client-side pipeline (including interceptor execution, credential fetching, and filter evaluation), this proposal focuses specifically on introducing observability for the two most common bottlenecks: **name resolution** and **load balancing pick** delays. This proposal establishes a generic telemetry framework extensible to other client-side delays in the future.


### Related Proposals:
* **[gRPC A66: OpenTelemetry Metrics][A66]**: Establishes base OpenTelemetry metrics.
* **[gRPC A72: OpenTelemetry Tracing][A72]**: Establishes tracing spans and events.
* **[gRPC A56: Priority LB Policy][A56]**: Establishes the priority load balancing policy and child failover mechanics.
* **[gRPC A28: xDS Traffic Splitting and Routing][A28]**: Establishes the `xds_cluster_manager` and `weighted_target` container policies.


## Proposal

### 1. Telemetry Schema

To measure client-side delays, we introduce a duration histogram and a dedicated tracing span at the **Call Level** (for name resolution delays), and another histogram and span at the **Attempt Level** (for load balancing pick delays).

We split observability data between low-cardinality metrics and high-cardinality debug tracing across two layers:

1.  **Call Level Observability (Channel-level Delays)**:
    *   **Scope**: Measures delays that occur at the channel level before an individual network attempt is initiated. This is primarily used to observe name resolution and configuration resolution delays.
    *   **Metric**: `grpc.client.call.delay.duration` (Float64 Histogram).
    *   **Tracing Span**: A child span named **`"Call Delay"`**, created as a child of the main RPC Call span.
2.  **Attempt Level Observability (Attempt-level Delays)**:
    *   **Scope**: Measures delays that occur within the lifecycle of a specific RPC attempt, primarily during the load balancing pick loop and connection establishment.
    *   **Metric**: `grpc.client.attempt.delay.duration` (Float64 Histogram).
    *   **Tracing Span**: A child span named **`"Attempt Delay"`**, created as a child of the individual Attempt span.

#### Span Attributes and Events

For both layers, a child span is created for each distinct `grpc.delay_type`, and transitions within that type are recorded as span events:

*   **`grpc.delay_type` (Span Attribute & Metric Label)**: A low-cardinality, composed string representing the type of the delay. The value format is `[parent_prefix:]base_type` (e.g., `connecting`, `p0:connecting`, or `rls_lookup_pending`). It is recorded as a **span attribute** on the child span and as a **metric label** on the corresponding duration histogram.
*   **`grpc.delay_reason` (Span Event Attribute)**: A high-cardinality string capturing runtime details (such as target names, subchannel IP addresses, or connection error messages). 

When a delay begins, the tracer creates the child span (`"Call Delay"` or `"Attempt Delay"`) carrying the `grpc.delay_type` attribute. It also records the initial reason as a span event named `"Delay state transition"` containing the `grpc.delay_reason` string.

If the state changes but the `grpc.delay_type` remains the same (e.g., a priority policy fails over or RLS changes backend targets), the tracer emits a new `"Delay state transition"` span event on the active child span with the updated `grpc.delay_reason` string, without recreating the span.


When the delay resolves or transitions to a different delay type, the active child span is closed.

#### Span Event Schema Specification
To ensure tracing backends can programmatically parse `"Delay state transition"` events, the event MUST conform to the following structural schema:
```json
{
  "name": "Delay state transition",
  "attributes": {
    "grpc.delay_type": "string (the active delay type, e.g., 'connecting')",
    "grpc.delay_reason": "string (the new granular state description)"
  }
}
```

#### Metrics Definitions

The following metrics are registered as client-side per-call metrics, extending the instrumentation framework defined in [gRPC A66][A66].

##### 1. Call Delay Duration Histogram

| Field | Value |
|---|---|
| **Name** | `grpc.client.call.delay.duration` |
| **Type** | Float64 Histogram |
| **Unit** | `s` (seconds) |
| **Description** | EXPERIMENTAL. Time an RPC spent waiting at the call level before an attempt was initiated, such as waiting for name resolution. |
| **Labels** | `grpc.target`, `grpc.delay_type` |
| **Optional Labels** | `grpc.method` |
| **Bucket Boundaries** | Same as A66 latency buckets: 0, 0.00001, 0.00005, 0.0001, 0.0003, 0.0006, 0.0008, 0.001, 0.002, 0.003, 0.004, 0.005, 0.006, 0.008, 0.01, 0.013, 0.016, 0.02, 0.025, 0.03, 0.04, 0.05, 0.065, 0.08, 0.1, 0.13, 0.16, 0.2, 0.25, 0.3, 0.4, 0.5, 0.65, 0.8, 1, 2, 5, 10, 20, 50, 100 |
| **Default Enabled** | `false` (experimental, opt-in) |

##### 2. Attempt Delay Duration Histogram

| Field | Value |
|---|---|
| **Name** | `grpc.client.attempt.delay.duration` |
| **Type** | Float64 Histogram |
| **Unit** | `s` (seconds) |
| **Description** | EXPERIMENTAL. Time an RPC attempt spent waiting for a load balancing pick or connection establishment. |
| **Labels** | `grpc.target`, `grpc.delay_type` |
| **Optional Labels** | `grpc.method` |
| **Bucket Boundaries** | Same as A66 latency buckets: 0, 0.00001, 0.00005, 0.0001, 0.0003, 0.0006, 0.0008, 0.001, 0.002, 0.003, 0.004, 0.005, 0.006, 0.008, 0.01, 0.013, 0.016, 0.02, 0.025, 0.03, 0.04, 0.05, 0.065, 0.08, 0.1, 0.13, 0.16, 0.2, 0.25, 0.3, 0.4, 0.5, 0.65, 0.8, 1, 2, 5, 10, 20, 50, 100 |
| **Default Enabled** | `false` (experimental, opt-in) |

### 2. Telemetry Value Taxonomy

To ensure consistency across implementations, we define the taxonomy of metric and tracing labels. 

All connection-related delays are consolidated into a single low-cardinality metric label value (`"connecting"`), while their detailed reasons are recorded inside the `"Delay state transition"` span event (as the `grpc.delay_reason` event attribute).

#### 1. Metric Delay Types (`grpc.delay_type`)
The `grpc.delay_type` label is a low-cardinality, restricted set of values. To ensure maximum clarity, these types are explicitly partitioned into Call-Level and Attempt-Level scopes, corresponding to their respective duration histograms. Structural container policies may compose the attempt-level values by prepending logical prefixes.

##### 1. Call-Level Delay Types
Recorded on the `grpc.client.call.delay.duration` histogram. These represent delays that occur before an individual attempt is created:
*   `"resolving"`: The channel is delayed waiting for name resolution or configuration parsing.

##### 2. Attempt-Level Delay Types
Recorded on the `grpc.client.attempt.delay.duration` histogram. These represent delays that occur during a specific RPC attempt:
*   `"connecting"`: The attempt is delayed waiting for subchannel connection establishment or picker initialization.
*   `"rls_lookup_pending"`: Specifically for Route Lookup Service (RLS) control-plane cache-miss query lookups.
*   `"cds_dynamic_discovery"`: Specifically for xDS Cluster Discovery Service (CDS) dynamic metadata resource fetches.
*   `"subchannel_state_mismatch"`: The target subchannel transitioned out of `READY` (e.g., disconnected or went to `TRANSIENT_FAILURE`), but the active channel picker has not yet been updated with a new picker.
*   `"picker_failing_with_wait_for_ready"`: A `wait_for_ready` RPC is buffered/queued because the picker is in `TRANSIENT_FAILURE`, waiting for the next connection attempt to succeed.

##### 3. Composed Attempt-Level Types
Structural container policies prepend their logical prefixes to the base attempt-level types using a colon separator:
*   **Priority Policy**: Prepends the active priority tier index, resulting in composed types like:
    *   `p0:connecting`
    *   `p1:subchannel_state_mismatch`
    *   `p0:picker_failing_with_wait_for_ready`
*   **Pass-Through Container Policies**: Policies like `xds_cluster_manager`, `weighted_target`, and `rls` **do not prepend any prefix or wrap the delay type**. They simply bubble up the child's `grpc.delay_type` (e.g., `"connecting"`) directly as-is.

##### Metadata Propagation & Composition
The prepending and wrapping logic is handled entirely inside the Balancer Picker tree hierarchy, keeping the channel's attempt-routing wrapper and tracer plugins completely decoupled and simple:
1.  **Leaf Pickers** (such as `pick_first` or `round_robin`): Generate the base leaf types (e.g., `delay_type = "connecting"`) and the initial, descriptive `delay_reason` (e.g., `"subchannel connecting: TCP handshake in progress"`).
2.  **Container Pickers**: Intercept the child's deferred pick result as it bubbles up the picker tree. The `priority` picker prepends its active tier index to the type (e.g., producing `"p0:connecting"`). A pass-through picker (such as `xds_cluster_manager`) forwards the child's `delay_type` unmodified, but can enrich the `delay_reason` with its own structural details.
3.  **The Channel Wrapper**: Receives the final, fully-composed `delay_type` and `delay_reason` strings from the root picker and passes them directly to the tracer (`recordAttemptDelayStart`) without any parsing, branching, or type checks.

---

#### 2. Taxonomy of Delay Reasons (Span Event Attribute: `grpc.delay_reason`)
Unlike the strict, low-cardinality metric types, the `grpc.delay_reason` is an **unconstrained, free-form debug string** designed to convey maximum troubleshooting context. 
*   It is written as a **human-readable, spaced string** (e.g. `"subchannel is connecting"` or `"waiting for DNS query to complete"`), **not** a `snake_case` token or closed enum.
*   It is designed to contain **high-cardinality metadata** (such as specific subchannel IP addresses, target names, cache keys, or raw connection error messages) to provide rich diagnostics inside the trace span.

To assist implementers, we explain the physical scenarios that are associated with each `grpc.delay_type` below. These scenarios represent common connection/resolver bottlenecks but are **non-exhaustive**; implementations are encouraged to append additional debug details.

##### Category A: Resolver Scenarios (Type: `"resolving"`)
*   **DNS Resolver Pending**: The channel is waiting for the initial name resolution query to complete. The reason string should describe the pending resolver query (e.g., `"waiting for DNS query to complete for target example.com"`).

##### Category B: Control-Plane & Metadata Scenarios (Type: `"rls_lookup_pending"`, `"cds_dynamic_discovery"`)
*   **RLS Cache-Miss Lookup**: An RPC is blocked because RLS is executing a control-plane query. The reason string describes the pending query and can include the RLS server target address or RLS cache keys (e.g., `"Route Lookup Service query pending on rls-server:8080"`).
*   **CDS Metadata Fetch**: An xDS channel is waiting for dynamic CDS cluster resource definitions. The reason string describes the dynamic discovery request and can include the targeted cluster name (e.g., `"waiting for CDS resource definition for cluster cluster_abc"`).

##### Category C: Subchannel Connection Scenarios (Type: `"connecting"`)
Recorded when an RPC attempt is queued waiting for a subchannel to establish a transport:
*   **Idle Subchannel Trigger**: The picker selects a subchannel that is in `IDLE` state, triggering a new connection attempt. The reason string captures this transition (e.g., `"subchannel was idle, triggering connection attempt"`).
*   **Active TCP/TLS Handshake**: The subchannel is actively connecting. The reason string describes that the TCP or TLS handshake is in progress, and can include the target subchannel's remote address (e.g., `"subchannel connecting: TCP handshake in progress to 192.168.1.50:8080"`).
*   **A105 Connection Scaling**: Per gRFC A105, an established subchannel connection is scaling up to add additional connections to satisfy high concurrent streams. The reason string captures this scaling state (e.g., `"scaling up additional subchannel connections per A105 limits"`).
*   **Waiting for Health Report**: The physical transport connection is established, but the picker is blocked waiting for the initial Out-of-Band (OOB) health check stream to return `SERVING` (e.g., `"transport connected, waiting for initial health check report"`).
*   **Subchannel Non-Existence**: The load balancer has not yet created or resolved the target subchannel. The reason string describes the missing subchannel state.
*   **Round Robin / WRR Scenarios**: The policy is connecting because it is waiting for any of its configured subchannels to become ready. The reason string describes the policy's connectivity attempt and can list the subchannels being attempted.
*   **Ring Hash Scenarios**: The hash ring is waiting for subchannels in the ring to connect. The reason string describes that the ring is connecting and can list the specific ring nodes being attempted.
*   **xDS Override Host Scenarios**: The policy is attempting to connect to a specific overridden host (subchannel) but the host is not yet connected. The reason string explains the overridden host state.

##### Category D: Picker and State Mismatch Scenarios (Type: `"subchannel_state_mismatch"`, `"picker_failing_with_wait_for_ready"`)
*   **Subchannel State Mismatch**: The subchannel transitioned out of `READY` (e.g., disconnected or entered `TRANSIENT_FAILURE`) before the active picker could be updated. The reason string captures the mismatch and the subchannel's target.
*   **Wait-For-Ready Buffering**: A `wait_for_ready` RPC is buffered because the picker is in `TRANSIENT_FAILURE`. The reason string captures the last connection error message (e.g., `"picker is in transient failure: connection refused by peer"`).

##### Category E: Priority Policy Behavior (Type: Composed `p0:`, `p1:` prefix)
The priority load balancing policy ([gRPC A56][A56]) manages failover between multiple priority groups (e.g. `p0`, `p1`, `p2`). Its delay telemetry behaves as follows:
*   When the priority policy is waiting on its primary tier (`p0`) to connect, the overall metric delay type is `"p0:connecting"`.
*   If `p0` fails (enters `TRANSIENT_FAILURE`) and the policy fails over to `p1`, the picker updates the active delay type to `"p1:connecting"`.
*   Because the delay type has transitioned, the active child span is closed, a new child span `"p1:connecting"` is opened, and the picker records the exact failover reason as a spaced string event (which can include the logical priority name or child policy name from the configuration, e.g., `"waiting on priority group p1 (child 'tier-1-backup') (p0 failed: connection timeout)"`). This provides a clear, high-fidelity timeline of the failover sequence in the trace.

##### Category F: Pass-Through Container Policy Scenarios (Type: `"connecting"` bubbled up directly)
Policies like `xds_cluster_manager`, `weighted_target`, and `rls` do not modify the metric `grpc.delay_type` (it remains strictly `"connecting"` or whatever leaf type is bubbled up from the child).
However, to provide visibility in tracing, the parent container policy's structural details (such as the target cluster name, RLS route shard, or weight group) are recorded inside the trace event's free-form description or as span attributes (e.g., `"waiting for child cluster 'cluster-abc' to connect"`).


### 3. Tracer API Changes

#### Lifecycle & State Machine

The client channel, load balancer, and telemetry tracer coordinate synchronously to record delays without dynamic memory allocation during routing.

##### 1. Timer Orchestration
*   **Resolver / Control-Plane (Call-Level Delay)**: When the client channel is initialized or name resolution is re-triggered, the channel starts a logical timer and invokes `recordCallDelayStart("resolving", reason)` to create the `"Call Delay"` child span carrying the `grpc.delay_type = "resolving"` attribute. When the resolver successfully applies the first valid service config and endpoints, the channel stops the logical timer and invokes `recordCallDelayEnd()`.
*   **LB Picker (Attempt-Level Delay)**: When an RPC attempt is initiated, the picker executes. If the picker defers the pick (e.g. returning `ErrNoSubConnAvailable`), it returns a pending result containing the metric delay type (e.g., `"connecting"`) and the free-form debug reason. The channel's attempt-routing wrapper starts a logical timer and invokes `recordAttemptDelayStart(type, reason)` to create the `"Attempt Delay"` child span. As the attempt remains buffered, subsequent picker evaluations may return different reasons, which the channel updates using `recordAttemptDelayReasonChanged(reason)` to append span events. When a picker evaluation successfully assigns a subchannel, the channel stops the logical timer and invokes `recordAttemptDelayEnd()`.
*   **Scope Resolution**: The decision of whether a delay segment is Call-Level (recorded to `grpc.client.call.delay.duration`) or Attempt-Level (recorded to `grpc.client.attempt.delay.duration`) is determined entirely by the scope of the tracer object on which the callbacks are invoked. Call-level delays (such as `"resolving"`) are invoked strictly on the call-scoped tracer (e.g. `ClientCallTracer` in Go, `ClientStreamTracer.Factory` in Java, or `ClientCallTracerInterface` in C++). Attempt-level delays (such as `"connecting"`) are invoked strictly on the attempt-scoped tracer (e.g. `ClientCallAttemptTracer` in Go, `ClientStreamTracer` in Java, or `CallAttemptTracer` in C++).

##### 2. Emission
Both metric duration recording and trace span management are fully delegated to the registered telemetry plugin (e.g., OpenTelemetry) via the tracer API:
*   **Span Lifecycle**: Upon receiving the `recordDelayStart` signal, the telemetry plugin instantiates the child trace span. Subsequent `recordDelayReasonChanged` calls append `"Delay state transition"` events to the active span.
*   **Simultaneous Metric & Span Close**: Upon receiving the `recordDelayEnd` signal, the telemetry plugin **simultaneously ends the child trace span and emits the final duration metric** to the corresponding duration histogram (`grpc.client.call.delay.duration` or `grpc.client.attempt.delay.duration`), measuring the elapsed time of the logical timer.

##### 3. Cancellation & Timeout
If an active RPC call or attempt is cancelled (due to client cancellation, `DEADLINE_EXCEEDED` timeouts, or channel shutdown) while a delay is logically active:
*   The core channel **MUST** stop the logical timer.
*   The core channel **MUST** invoke the tracer's end callback (`recordCallDelayEnd()` or `recordAttemptDelayEnd()`). 
*   This ensures the telemetry plugin can capture the elapsed duration up to the point of failure, close the trace span, and emit the partial duration metric, guaranteeing that bottlenecks preceding a failure remain fully observable.

##### 4. Retries & Hedging
Each RPC attempt is tracked independently:
*   **Attempt Isolation**: A transparent retry or hedged attempt will instantiate its own attempt-level tracer. 
*   **Independent Timing**: If the new attempt's picker is deferred, the attempt starts its own independent delay logical timer and child trace span, without affecting the call-level resolver timing or the timing of other concurrent attempts.

#### Language-Specific API Definitions

##### Go (Experimental V2 Stats Handler Framework)
In gRPC-Go, the current `stats.Handler` interface has several critical limitations that prevent it from supporting modern telemetry needs and call-level observability:
1.  **Strictly Attempt-Scoped**: All RPC events in `stats.Handler` (`TagRPC`, `HandleRPC`) are executed on individual HTTP/2 streams (attempts). There is no "Call-scoped" hook to track milestones that span multiple attempts or occur before an attempt is created (such as DNS name resolution, configuration parsing, or RLS control-plane queries).
2.  **Type-Assertion Overhead**: The V1 `HandleRPC(context.Context, RPCStats)` method passes stats via the empty interface `RPCStats`. Implementations must perform runtime type assertions (e.g. `switch s := stat.(type)`) to process events. This degrades CPU performance and prevents compile-time contract enforcement.
3.  **Dynamic Memory Allocations**: Wrapping every telemetry event in a struct (e.g. `stats.Begin`, `stats.InPayload`) and casting it to `RPCStats` requires heap allocations, which adds garbage collection pressure in high-throughput RPC paths.
4.  **Coupled Connection & RPC Lifecycles**: Connection-level stats and RPC-level stats are mixed in a single interface, preventing modular plugin registration.

To completely replace the V1 stats handler with a high-performance, pluggable, and fully observable framework, gRPC-Go will introduce a new **V2 Stats Handler** framework as part of the existing `google.golang.org/grpc/stats` package. The V2 framework moves away from V1's struct-based event bubbling and adopts a **direct method/callback-based interface** with three isolated tracer scopes, decorated with standard Go experimental doc comments.

###### 1. Interface Definitions in `google.golang.org/grpc/stats`

```go
package stats

import (
	"context"
	"net"
	"time"

	"google.golang.org/grpc/metadata"
)

// HandlerV2 is the factory interface that telemetry plugins implement.
// It instantiates stateful, scoped tracers for calls, attempts, and connections.
//
// Experimental: this interface is experimental and subject to change.
type HandlerV2 interface {
	// NewCallTracer instantiates a CallTracer to monitor a client-side overall RPC call.
	// The returned context is used throughout the lifetime of the call.
	NewCallTracer(ctx context.Context, info *CallInfo) (context.Context, CallTracer)

	// NewAttemptTracer instantiates an AttemptTracer to monitor an individual RPC stream attempt.
	// The returned context is used throughout the lifetime of the attempt.
	NewAttemptTracer(ctx context.Context, info *AttemptInfo) (context.Context, AttemptTracer)

	// NewConnTracer instantiates a ConnTracer to monitor a physical transport connection.
	// The returned context is used throughout the lifetime of the connection.
	NewConnTracer(ctx context.Context, info *ConnInfo) (context.Context, ConnTracer)
}

// CallTracer is a call-scoped interface tracking the overall client RPC call.
//
// Experimental: this interface is experimental and subject to change.
type CallTracer interface {
	// RecordDelayStart indicates the start of a call-level delay (e.g., "resolving").
	RecordDelayStart(delayType string, reason string)
	// RecordDelayReasonChanged indicates a transition in the call-level delay reason.
	RecordDelayReasonChanged(reason string)
	// RecordDelayEnd indicates the end of the call-level delay.
	RecordDelayEnd()
	// OnCallEnd is called when the overall RPC call completes.
	OnCallEnd(err error)
}

// AttemptTracer is an attempt-scoped interface tracking an individual stream attempt.
//
// Experimental: this interface is experimental and subject to change.
type AttemptTracer interface {
	// RecordDelayStart indicates the start of an attempt-level delay (e.g., "connecting").
	RecordDelayStart(delayType string, reason string)
	// RecordDelayReasonChanged indicates a transition in the attempt-level delay reason.
	RecordDelayReasonChanged(reason string)
	// RecordDelayEnd indicates the end of the attempt-level delay.
	RecordDelayEnd()
	
	// OnHeaderSent/OnHeaderRecv replace V1 OutHeader and InHeader.
	OnHeaderSent(compression string, md metadata.MD)
	OnHeaderRecv(wireLength int, compression string, md metadata.MD)
	
	// OnPayloadSent/OnPayloadRecv replace V1 OutPayload and InPayload.
	OnPayloadSent(payload any, length, compressedLength, wireLength int, sentTime time.Time)
	OnPayloadRecv(payload any, length, compressedLength, wireLength int, recvTime time.Time)
	
	// OnTrailerSent/OnTrailerRecv replace V1 OutTrailer and InTrailer.
	OnTrailerSent(md metadata.MD)
	OnTrailerRecv(wireLength int, md metadata.MD)
	
	// OnAttemptEnd is called when this specific stream attempt completes (replacing V1 End).
	OnAttemptEnd(err error)
}

// ConnTracer is a connection-scoped interface tracking physical transport connections.
type ConnTracer interface {
	// OnConnEnd is called when the transport connection terminates (replacing V1 ConnEnd).
	OnConnEnd()
}

// Supporting Metadata Structs
type CallInfo struct {
	FullMethod string
	FailFast   bool
}

type AttemptInfo struct {
	IsTransparentRetry bool
	IsHedged           bool
	RemoteAddr         net.Addr
	LocalAddr          net.Addr
}

type ConnInfo struct {
	RemoteAddr net.Addr
	LocalAddr  net.Addr
}
```

###### 2. Coexistence & Migration Strategy
During the experimental phase, gRPC-Go will support both V1 and V2 interfaces concurrently to avoid breaking existing ecosystems (such as OpenCensus and older OpenTelemetry plugins):
*   **Registration**: The channel option `grpc.WithStatsHandler()` will accept both `Handler` (V1) and `HandlerV2` (V2) types using interface checks.
*   **Adaptation**: An internal adapter will be provided to wrap a V1 `Handler` into a V2 `HandlerV2` (converting method callbacks back into struct events and dispatching them to `HandleRPC`), ensuring older plugins continue to function.
*   **Native Performance**: If a native V2 handler is registered (such as the new OpenTelemetry stats handler), the channel will bypass all V1 event struct allocations and type-assertions, running completely on the high-performance, zero-allocation callback path.

##### Java
In `io.grpc.ClientStreamTracer` and `ClientStreamTracer.Factory`:
*Note: Call-level delays (like name resolution) occur before an attempt is created. Because `ClientStreamTracer` is attempt-scoped, the call-level delay APIs are added to `ClientStreamTracer.Factory` (which is call-scoped and plumbed throughout the channel), while the attempt-level delay APIs remain on the `ClientStreamTracer` instance.*
```java
package io.grpc;

public abstract class ClientStreamTracer extends StreamTracer {
  /**
   * Called when an attempt-level delay segment (e.g. LB Pick connection) starts.
   */
  public void recordAttemptDelayStart(String delayType, String delayReason) {}

  /**
   * Called when an attempt-level delay reason changes.
   */
  public void recordAttemptDelayReasonChanged(String delayReason) {}

  /**
   * Called when an attempt-level delay segment ends.
   */
  public void recordAttemptDelayEnd() {}
}

// Nested inside ClientStreamTracer:
public static abstract class Factory {
  /**
   * Called when a call-level delay segment (e.g. name resolution) starts.
   */
  public void recordCallDelayStart(String delayType, String delayReason) {}

  /**
   * Called when a call-level delay reason changes.
   */
  public void recordCallDelayReasonChanged(String delayReason) {}

  /**
   * Called when a call-level delay segment ends.
   */
  public void recordCallDelayEnd() {}

  public abstract ClientStreamTracer newClientStreamTracer(StreamInfo info, Metadata headers);
}
```

##### C++ (Core)
In `src/core/telemetry/call_tracer.h`:
*Note: To prevent core interface bloat and align with the core thinning refactoring, we leverage the existing C++ Annotation framework rather than adding new virtual methods.*
```cpp
namespace grpc_core {

class DelayAnnotation final : public CallTracerAnnotationInterface::Annotation {
 public:
  enum class Stage { kStart, kReasonChanged, kEnd };
  
  DelayAnnotation(Stage stage, absl::string_view type, absl::string_view reason)
      : Annotation(CallTracerAnnotationInterface::AnnotationType::kDelay),
        stage_(stage), type_(type), reason_(reason) {}
        
  Stage stage() const { return stage_; }
  absl::string_view type() const { return type_; }
  absl::string_view reason() const { return reason_; }

 private:
  Stage stage_;
  absl::string_view type_;
  absl::string_view reason_;
};

} // namespace grpc_core
```

### Feature Flag


All delay metrics, tracing, and API hooks will be guarded by a feature flag:
* **Go/Java Env Var**: `GRPC_EXPERIMENTAL_ENABLE_DELAY_OBSERVABILITY` (Default: `false`)
* **C++ Core Experiment**: `IsExperimentEnabled("client_delay_observability")` (registered in `experiments.h`)

## Rationale

We implement recording at the client channel level rather than inside specific load balancing policies because only the client channel manages the buffering, queueing, and context cancellation lifecycles of RPC calls. This decouples policy-level state reporting from duration measurement.

To minimize overhead, pickers pre-compute tokens at configuration time, avoiding dynamic string allocations during picks. Dynamic tokens are only used for policies that perform per-request routing (e.g., RLS, xDS cluster manager).

## Implementation

We will implement this in Go, Java, and C++ (Core).



[A66]: A66-otel-stats.md
[A72]: A72-open-telemetry-tracing.md
[A56]: A56-priority-lb-policy.md
[A28]: A28-xds-traffic-splitting-and-routing.md