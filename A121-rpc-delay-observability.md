# A121: RPC Delay Observability
----
* Author(s): Madhav Bissa (@mbissa)
* Approver: @markdroth, @ejona86, @dfawley, @easwars
* Implemented in: Go, Java, C++
* Last updated: 2026-06-25
* Discussion at: https://groups.google.com/g/grpc-io/c/NsxXJ2MxXM4

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

### 2. Delay Types & Reasons

To ensure consistency across implementations, we define the taxonomy of metric and tracing labels. 

All connection-related delays are consolidated into a single low-cardinality metric label value (`"connecting"`), while their detailed reasons are recorded inside the `"Delay state transition"` span event (as the `grpc.delay_reason` event attribute).

#### 1. Metric Delay Types (`grpc.delay_type`)
The `grpc.delay_type` label is a low-cardinality, restricted set of values. To ensure maximum clarity, these types are explicitly partitioned into Call-Level and Attempt-Level scopes, corresponding to their respective duration histograms. Structural container policies may compose the attempt-level values by prepending logical prefixes.

##### 1. Call-Level Delay Types
Recorded on the `grpc.client.call.delay.duration` histogram. These represent delays that occur before an individual attempt is created:
*   `"resolving"`: The channel is delayed waiting for name resolution.

##### 2. Attempt-Level Delay Types
Recorded on the `grpc.client.attempt.delay.duration` histogram. These represent delays that occur during a specific RPC attempt:
*   `"connecting"`: The attempt is delayed waiting for subchannel connection establishment or picker initialization.
*   `"rls_lookup_pending"`: Specifically for Route Lookup Service (RLS) control-plane cache-miss query lookups.
*   `"cds_dynamic_discovery"`: Specifically for xDS Cluster Discovery Service (CDS) dynamic metadata resource fetches.
*   `"subchannel_state_mismatch"`: The target subchannel transitioned out of `READY` (e.g., disconnected or went to `TRANSIENT_FAILURE`), but the active channel picker has not yet been updated with a new picker. *Note: This delay type is generated by the channel intercepting a stale pick, not by the picker itself.*
*   `"picker_failing_with_wait_for_ready"`: A `wait_for_ready` RPC is buffered/queued because the picker is in `TRANSIENT_FAILURE`, waiting for the next connection attempt to succeed. *Note: This delay type is generated by the channel wrapper detecting a `wait_for_ready` RPC when the picker returns an error, keeping pickers ignorant of `wait_for_ready` semantics.*

##### 3. Composed Attempt-Level Types
Structural container policies prepend their logical prefixes to the base attempt-level types using a colon separator:
*   **Priority Policy**: Prepends the active priority tier index, resulting in composed types like:
    *   `p0:connecting`
    *   `p1:subchannel_state_mismatch`
    *   `p0:picker_failing_with_wait_for_ready`
*   **Pass-Through Container Policies**: Policies like `xds_cluster_manager`, `weighted_target`, and `rls` **do not prepend any prefix or wrap the delay type**. They simply bubble up the child's `grpc.delay_type` (e.g., `"connecting"`) directly as-is.

Because only the priority policy contributes a prefix and it contributes exactly one (its active tier index), the resulting `grpc.delay_type` cardinality stays bounded at roughly *(base attempt-level types) × (configured priority tiers)*; prefixes do not stack across nested pass-through containers. Implementations SHOULD keep the prefix a small bounded token (the tier index) so metric cardinality remains low; richer per-container structure belongs in the `grpc.delay_reason` span event, not the metric label.

##### Metadata Propagation & Composition
The prepending and wrapping logic is handled entirely inside the Balancer Picker tree hierarchy, keeping the channel's attempt-routing wrapper and tracer plugins completely decoupled and simple:
1.  **Leaf Pickers** (such as `pick_first` or `round_robin`): Generate the base delay types (e.g., `delay_type = "connecting"`) and the initial, descriptive `delay_reason` (e.g., `"subchannel connecting: TCP handshake in progress"`).
2.  **Container Pickers**: Intercept the child's deferred pick result as it bubbles up the picker tree. The `priority` picker prepends its active tier index to the type (e.g., producing `"p0:connecting"`). A pass-through picker (such as `xds_cluster_manager`) forwards the child's `delay_type` unmodified, but can enrich the `delay_reason` with its own structural details.
3.  **The Channel Wrapper**: Receives the final, fully-composed `delay_type` and `delay_reason` strings from the root picker and passes them directly to the tracer (`recordAttemptDelayStart`). The channel wrapper is also responsible for intercepting specific channel-level states (e.g., generating `"picker_failing_with_wait_for_ready"` when a picker returns an error for a `wait_for_ready` RPC, or `"subchannel_state_mismatch"` when a transport disconnects before the picker is updated).

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


### 3. Telemetry API

#### Lifecycle & State Machine

The client channel, load balancer, and telemetry tracer coordinate synchronously to record delays without dynamic memory allocation during routing.

##### 1. Timer Orchestration
The channel owns the **lifecycle** of a delay segment (deciding when it starts, transitions, and ends) while the telemetry plugin owns the **timing** of that segment (it timestamps the start on `recordDelayStart`, computes the elapsed duration on `recordDelayEnd`, and emits the histogram). The channel therefore does not maintain a separate duration clock; "logical timer" below refers to this plugin-side measurement, delimited by the channel's start/end signals.
*   **Resolver / Control-Plane (Call-Level Delay)**: When an RPC is blocked waiting for name resolution, the channel invokes `recordCallDelayStart("resolving", reason)` to open the `"Call Delay"` child span carrying the `grpc.delay_type = "resolving"` attribute. When the resolver successfully applies the first valid service config and endpoints, the channel invokes `recordCallDelayEnd()`. (Note: If resolution completes before any RPC is blocked, no delay is recorded.)
*   **LB Picker (Attempt-Level Delay)**: When an RPC attempt is initiated, the picker executes. If the picker defers the pick (i.e. it cannot return a ready connection yet), it surfaces the metric delay type (e.g., `"connecting"`) and the free-form debug reason alongside the existing "no connection available" signal. The channel's attempt-routing wrapper invokes `recordAttemptDelayStart(type, reason)` to open the `"Attempt Delay"` child span. As the attempt remains buffered, subsequent picker evaluations may return different reasons, which the channel updates using `recordAttemptDelayReasonChanged(reason)` to append span events. When a picker evaluation successfully assigns a subchannel, the channel invokes `recordAttemptDelayEnd()`.
*   **Scope Resolution**: The decision of whether a delay segment is Call-Level (recorded to `grpc.client.call.delay.duration`) or Attempt-Level (recorded to `grpc.client.attempt.delay.duration`) is determined entirely by the scope of the tracer object on which the callbacks are invoked. Call-level delays (such as `"resolving"`) are invoked strictly on the call-scoped tracer object, while attempt-level delays (such as `"connecting"`) are invoked strictly on the attempt-scoped tracer object. This mapping is enforced statically by the respective language API signatures.

##### 2. Emission
Both metric duration recording and trace span management are fully delegated to the registered telemetry plugin (e.g., OpenTelemetry) via the tracer API:
*   **Span Lifecycle**: Upon receiving the `recordDelayStart` signal, the telemetry plugin instantiates the child trace span. Subsequent `recordDelayReasonChanged` calls append `"Delay state transition"` events to the active span.
*   **Simultaneous Metric & Span Close**: Upon receiving the `recordDelayEnd` signal, the telemetry plugin **simultaneously ends the child trace span and emits the final duration metric** to the corresponding duration histogram (`grpc.client.call.delay.duration` or `grpc.client.attempt.delay.duration`), measuring the elapsed time of the logical timer.

##### 3. Cancellation & Timeout
If an active RPC call or attempt is cancelled (due to client cancellation, `DEADLINE_EXCEEDED` timeouts, or channel shutdown) while a delay is logically active:
*   The core channel **MUST** invoke the tracer's end callback (`recordCallDelayEnd()` or `recordAttemptDelayEnd()`), which finalizes the plugin-side timing for the open segment. 
*   This ensures the telemetry plugin can capture the elapsed duration up to the point of failure, close the trace span, and emit the partial duration metric, guaranteeing that bottlenecks preceding a failure remain fully observable.

##### 4. Retries & Hedging
Each RPC attempt is tracked independently:
*   **Attempt Isolation**: A transparent retry or hedged attempt will instantiate its own attempt-level tracer. 
*   **Independent Timing**: If the new attempt's picker is deferred, the attempt starts its own independent delay logical timer and child trace span, without affecting the call-level resolver timing or the timing of other concurrent attempts.

##### 5. Attempt-Level Channel-Interceded Delays
Specific attempt-level delays are not generated by load-balancing pickers themselves but are interceded and synthesized directly by the channel wrapper when handling transport-level states:
*   **`picker_failing_with_wait_for_ready` (Wait-For-Ready Buffering)**:
    1.  *Trigger*: When an RPC is initiated and the root picker returns a failing/deferred result, the channel wrapper checks the RPC's `wait_for_ready` configuration. If `wait_for_ready` is `false`, the RPC fails immediately (no delay is timed). If `wait_for_ready` is `true`, the channel wrapper queues the RPC, starts a logical timer, and invokes `recordAttemptDelayStart("picker_failing_with_wait_for_ready", error_reason)`.
    2.  *Transition*: While the RPC remains queued, if the channel receives an updated picker that continues to return a different error, the channel wrapper updates the reason via `recordAttemptDelayReasonChanged(new_error_reason)`.
    3.  *Termination*: When an updated picker successfully assigns a ready subchannel, the channel wrapper stops the timer, invokes `recordAttemptDelayEnd()`, and creates the stream. If the RPC's deadline expires or the call is cancelled while queued, the channel wrapper invokes `recordAttemptDelayEnd()` to capture the partial duration.
*   **`subchannel_state_mismatch` (Post-Pick Transport Race)**:
    1.  *Trigger*: When the root picker returns a successful pick (assigning a subchannel), the channel wrapper attempts to create a transport stream on it. If the stream creation fails immediately because the subchannel has transitioned out of `READY` (a race condition before the picker is updated), the channel wrapper initiates a transparent retry. During this retry, the channel wrapper starts a logical timer and invokes `recordAttemptDelayStart("subchannel_state_mismatch", socket_disconnect_reason)`.
    2.  *Termination*: Once the load balancer publishes an updated picker and the channel wrapper successfully executes a pick that assigns a new, actually ready subchannel, the channel wrapper stops the timer and invokes `recordAttemptDelayEnd()`.

#### API Definitions

##### 1. Picker-Side API
When a picker defers a pick (it cannot return a ready connection yet), it surfaces two optional values **alongside the existing "no connection available" deferral signal**, carried on the runtime's existing pick-result value so that no new return channel is introduced:
*   `delay_type`: the bounded metric label for the current wait (e.g. `"connecting"`), composed by container pickers per [Section 2](#2-delay-types--reasons).
*   `delay_reason`: the free-form diagnostic string for the current wait.

The channel reads these only on the deferred path; a successful pick ignores them. Leaf pickers populate the base values and container pickers compose them as the result bubbles up the picker tree ([Section 2](#2-delay-types--reasons)). Pickers remain ignorant of `wait_for_ready` semantics and transport races — the channel itself synthesizes `picker_failing_with_wait_for_ready` and `subchannel_state_mismatch` ([Section 2](#2-delay-types--reasons), [Rationale](#rationale)).

Concretely, each runtime extends the type it already uses to express a deferred pick:
*   **Go**: two optional fields on `balancer.PickResult`. Because a deferred pick is signalled today through the `ErrNoSubConnAvailable` sentinel rather than a populated `PickResult`, the deferred path is extended to also surface a populated result carrying the delay metadata to the pick wrapper.
*   **Java**: carried on `PickResult` (which already transports an LB-supplied `ClientStreamTracer.Factory`), read where the channel interprets a no-result pick.
*   **C++ (Core)**: core has no synchronous picker-tree return to a channel wrapper, so the equivalent `delay_type`/`delay_reason` are recorded at the LB-pick point of the call's promise chain rather than returned upward (see Tracer-Side API).

##### 2. Tracer-Side API

The channel drives the telemetry plugin through six logical operations, split across the two existing telemetry scopes. The end callbacks carry no duration argument because the plugin owns timing ([3.1](#lifecycle--state-machine)):

| Logical operation | Scope | Effect |
|---|---|---|
| `recordCallDelayStart(delay_type, reason)` | call | open the `"Call Delay"` span; begin timing |
| `recordCallDelayReasonChanged(reason)` | call | append a `"Delay state transition"` event |
| `recordCallDelayEnd()` | call | close the span; emit `grpc.client.call.delay.duration` |
| `recordAttemptDelayStart(delay_type, reason)` | attempt | open the `"Attempt Delay"` span; begin timing |
| `recordAttemptDelayReasonChanged(reason)` | attempt | append a `"Delay state transition"` event |
| `recordAttemptDelayEnd()` | attempt | close the span; emit `grpc.client.attempt.delay.duration` |

The **scope of the receiving tracer object** (call-scoped vs attempt-scoped) statically determines which histogram a segment is recorded to ([3.1](#lifecycle--state-machine), Scope Resolution). Each language binds these operations onto its client-side telemetry API as follows:

| Scope | Go | Java | C++ (Core) |
|---|---|---|---|
| Call-level delay hooks | a **new** call-scoped tracer (new V2 stats-handler API) | new methods on the existing `ClientStreamTracer.Factory` (call-scoped, plumbed through the channel) | new `DelayAnnotation` subtype recorded on the existing call tracer |
| Attempt-level delay hooks | a **new** attempt-scoped tracer (new V2 stats-handler API) | new methods on the existing `ClientStreamTracer` (attempt-scoped) | new `DelayAnnotation` subtype recorded on the existing attempt tracer |

###### Language-specific bindings
The binding *mechanism* is deliberately asymmetric, because each language's current telemetry API differs; only the six logical operations and their two scopes are common. This asymmetry is inherent to the existing APIs, not a difference in the delay design itself:
*   **Go**: Introduces a **new** call-scoped and attempt-scoped tracer API (gRPC-Go's V2 stats handler). A new API is required because the V1 `stats.Handler` is attempt-scoped only and cannot host a call-level hook. (The closest existing V1 signal, `stats.DelayedPickComplete`, is attempt-scoped and merely marks the *end* of a blocked pick — it is an analogue, not a base we extend.) The broader V2 API is out of scope here beyond the delay hooks it must expose.
*   **Java**: **Reuses the existing tracer types** but adds new no-op-default methods to them — call hooks on `ClientStreamTracer.Factory` (which is call-scoped and exists before the first attempt is spawned) and attempt hooks on `ClientStreamTracer` (attempt-scoped). The `"resolving"` hooks build on the channel's stream-buffering path (such as `createPendingStream()`). Since a `ClientStreamTracer` instance is instantiated prior to the first pick attempt (per the gRPC-Java retry architecture), the attempt-level `"connecting"` delay is recorded **directly on the attempt-scoped `ClientStreamTracer` instance**, ensuring attempt-level telemetry remains cleanly isolated to the individual attempt's tracer.
*   **C++ (Core)**: **Reuses the existing annotation framework**, adding only a new `DelayAnnotation` subtype that replaces the current free-form `"Delayed name resolution complete."` / `"Delayed LB pick complete."` string annotations.

In short: C++ reuses its mechanism wholesale (one new subtype), Java reuses its tracer types but extends them with new methods, and Go introduces new tracer types outright.

###### Illustrative binding: C++ structured annotation
The delay transition is carried as an `Annotation` subtype on the C++ Core annotation framework. It implements **both** pure-virtual hooks of the base `Annotation` — `ToString()` (human-readable form) and `ForEachKeyValue()` (structured `grpc.delay_type`/`grpc.delay_reason` pairs) — and **owns** its strings so the annotation may safely outlive the stack frame that produced it:
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

  // Both overrides are required: the base Annotation declares them pure-virtual.
  std::string ToString() const override;
  void ForEachKeyValue(
      absl::FunctionRef<void(absl::string_view, ValueType)> callback)
      const override;

 private:
  Stage stage_;
  std::string type_;    // owned (not a string_view) to avoid a dangling view
  std::string reason_;  // owned
};

}  // namespace grpc_core
```

### Feature Flag


All delay metrics, tracing, and API hooks will be guarded by a feature flag:
* **Go/Java Env Var**: `GRPC_EXPERIMENTAL_ENABLE_DELAY_OBSERVABILITY` (Default: `false`)
* **C++ Core Experiment**: register `client_delay_observability` in `experiments.yaml`; gate via the generated `IsClientDelayObservabilityEnabled()` accessor (core uses generated per-experiment accessors, not string-keyed lookups).

## Metric Registration and Recording

Metric registration and recording will follow the established architectural patterns defined in [gRPC A66][A66] and [gRPC A94][A94]. The channel and attempts will delegate duration measurement directly to the registered telemetry plugin (e.g., OpenTelemetry) to avoid code duplication across the core stack. The specific plugin API registrations are omitted here for brevity but will conform to language-idiomatic OTel APIs (e.g., `RegisterFloat64Histo` in Go, `registerDoubleHistogram` in Java, and `RegisterDoubleHistogram` in C++).

## Rationale

**Call vs. Attempt Metrics:** We split metrics into two distinct histograms (Call-level vs Attempt-level) to clearly differentiate between channel initialization bottlenecks (like DNS or service configuration) and per-attempt routing bottlenecks. Combining them into a single histogram would obscure whether a delay occurred before or during connection establishment.

**Child Spans vs. Events:** We use child spans for delays instead of attaching events to the parent RPC span because it provides better visual flame-graph isolation in tracing backends, especially when delays are long or when multiple attempts are made (e.g., hedging or retries).

**Free-form `delay_reason`:** We explicitly chose to allow unconstrained, free-form debug strings for the trace event's `delay_reason` rather than bounded enum tokens. While this risks higher cardinality in tracing backends, it provides immense diagnostic value by allowing raw IP addresses, subchannel targets, and detailed connection error messages to be embedded directly into the trace. The metric label (`grpc.delay_type`) remains strictly bounded to ensure metric cardinality is kept low.

**Channel-Intercepted Delays:** We implement specific delay types like `picker_failing_with_wait_for_ready` and `subchannel_state_mismatch` as channel-level interceptions rather than picker-generated types. This intentionally keeps leaf pickers ignorant of `wait_for_ready` semantics and transport-level race conditions, enforcing a clean separation of concerns where the picker only reports its current state, and the channel dictates queueing behavior.

**Delegated Recording:** We implement duration recording at the client channel wrapper level rather than inside specific load balancing policies because only the client channel manages the buffering, queueing, and context cancellation lifecycles of RPC calls. This decouples policy-level state reporting from duration measurement.

## Implementation

We will implement this in Go, Java, and C++ (Core).



[A66]: A66-otel-stats.md
[A72]: A72-open-telemetry-tracing.md
[A56]: A56-priority-lb-policy.md
[A28]: A28-xds-traffic-splitting-and-routing.md