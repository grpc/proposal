Load Balancer Pick Queue Delay Observability
----
* Author(s): Madhav Bissa (@madhavbissa)
* Approver: TBD
* Status: Draft
* Implemented in: Go, Java, C++
* Last updated: 2026-04-22
* Discussion at: TBD

## Abstract

This proposal introduces observability for the duration that RPCs spend waiting in the gRPC client channel for a load balancing pick to complete. Today, when a picker indicates that no subchannel is available, the client channel then defers the RPC (either by waiting or blocking the caller) until the LB policy provides a new picker or the RPC context is canceled/times out. This latency is invisible to operators, making it difficult to diagnose which layer of the LB policy tree is causing the delay.

This gRFC defines:
1. A new histogram metric (`grpc.lb.pick_delay_duration`) recorded by the client
   channel, with bounded `grpc.lb.delay_reason` and optional `grpc.method`
   labels. The histogram emits one observation per delay reason segment,
   providing per-layer delay breakdown when an RPC transitions through multiple
   LB policy states.
2. A new up-down counter metric (`grpc.lb.rpc_waiting`) tracking the
   number of RPCs currently waiting, providing visibility into stuck traffic.
3. An enhancement to the existing "Delayed LB pick complete" span event ([A72])
   to include the delay reason and delay duration as span event attributes.
4. API changes in each language to allow pickers to expose the delay reason as a
   bounded, pre-computed string token, with support for both picker-level tokens
   (uniform policies) and per-pick tokens (RLS, cluster_manager).

## Background

gRPC uses a tree of load balancing policies to select a subchannel for each RPC.
When no suitable subchannel is available, the picker indicates this state to the client channel. The client channel then defers the RPC (either by waiting or blocking the caller) until the LB policy provides a new picker or the RPC's context is canceled or times out.
Operators currently have no visibility into:
- **How long** RPCs wait.
- **Why** they are delayed (which policy in the tree caused it).

The existing `grpc.client.attempt.duration` metric (A66) includes pick delay but
does not isolate it, and provides no information about the cause of the delay.

### Related Proposals

- [A66: OpenTelemetry Metrics][A66] — Base instrumentation and stats plugin
  architecture.
- [A72: OpenTelemetry Tracing][A72] — Tracing architecture including the
  existing "Delayed LB pick complete" event.
- [A78: gRPC Metrics for WRR, Pick First, and xDS][A78] — LB policy metrics
  patterns (`grpc.lb.*` naming convention, locality labels).
- [A79: Non-per-call Metrics Architecture][A79] — `GlobalInstrumentsRegistry`,
  `MetricsRecorder`, metric descriptor registration, and stability levels.
- [A91: Outlier Detection Metrics][A91] — Template for LB-level non-per-call
  metrics.
- [A94: Subchannel OTel Metrics][A94] — Subchannel-level metrics.
- [A56: Priority LB Policy][A56] — Priority policy, `initTimer`, failover.
- [A50: xDS Outlier Detection][A50] — Outlier detection ejection behavior.

## Proposal

Using [A79]'s non-per-call metrics architecture and [A72]'s tracing framework,
we will add a histogram metric, an active waiting RPC counter, and enhanced trace span
event that measure the time each RPC spends waiting for a load balancing
pick.

Each LB policy's picker provides a bounded string **delay reason token**
describing why it would delay RPCs (e.g., `"pick_first:connecting"`). For
policies with uniform picker state (pick_first, round_robin, ring_hash,
priority), the token is pre-computed at picker construction time. For policies
that make per-request routing decisions (RLS, cluster_manager), the token is
determined inside `Pick()` and varies per-RPC. Delegating policies compose
tokens by prepending their own prefix to their child picker's token (e.g.,
`"priority_p0:pick_first:connecting"`).

The client channel's pick loop reads this token from the picker when a
pick is delayed, records the delay start time, and emits one histogram
observation per delay reason segment — if the token changes during the wait
(e.g., an RLS lookup completes, causing a transition from `rls:lookup_pending`
to `rls:pick_first:connecting`), the current segment is closed and a new one
begins. An up-down counter tracks how many RPCs are currently waiting,
providing visibility into stuck traffic that has not yet emitted a histogram
observation.

For policies with uniform picker state, tokens are pre-computed strings stored
on the picker, so the pick path reads them by reference with zero dynamic
allocations. The token API uses optional interfaces (Go) or default-returning
methods (Java, C++) so that existing custom LB policies continue to work without
modification — they simply report an empty token, and the metric is still
recorded under the `grpc.target` label alone.

### Metric Definition

The following metric is registered via the non-per-call metrics framework
defined in [A79][A79].

| Field | Value |
|---|---|
| **Name** | `grpc.lb.pick_delay_duration` |
| **Type** | Float64 Histogram |
| **Unit** | `s` (seconds) |
| **Description** | EXPERIMENTAL. Time an RPC spent waiting for a load balancing pick, broken down by the reason for the delay. |
| **Labels** | `grpc.target`, `grpc.lb.delay_reason` |
| **Optional Labels** | `grpc.method` |
| **Bucket Boundaries** | Same as A66 latency buckets: 0, 0.00001, 0.00005, 0.0001, 0.0003, 0.0006, 0.0008, 0.001, 0.002, 0.003, 0.004, 0.005, 0.006, 0.008, 0.01, 0.013, 0.016, 0.02, 0.025, 0.03, 0.04, 0.05, 0.065, 0.08, 0.1, 0.13, 0.16, 0.2, 0.25, 0.3, 0.4, 0.5, 0.65, 0.8, 1, 2, 5, 10, 20, 50, 100 |
| **Default Enabled** | `false` (experimental, opt-in) |

> **Note on histogram aggregation**: The gRPC non-per-call metrics framework
> ([A79]) currently supports only explicit bucket histograms. Explicit buckets
> lose fidelity for values between wide boundaries (e.g., values in the
> `[20, 50)` range are indistinguishable). When the framework adds support for
> exponential bucket histograms, this metric should be migrated to use
> exponential aggregation for better tail-latency fidelity. In the interim,
> operators can override the aggregation strategy to exponential using
> OpenTelemetry SDK Views at the application level.

If the delay reason token changes during the wait (e.g., due to an RLS lookup
completing or a priority failover), the recording loop emits one histogram
observation **per delay reason segment**. A segment ends when a new picker
arrives with a different token, or when the wait ends. A single RPC may
therefore contribute multiple histogram observations, each covering the time
spent under a specific delay reason. This per-segment approach gives operators
visibility into how much time each layer of the LB policy tree contributed to
the total pick delay.

The following gauge metric is also registered to provide visibility into RPCs
that are currently waiting and have not yet emitted a histogram observation:

| Field | Value |
|---|---|
| **Name** | `grpc.lb.rpc_waiting` |
| **Type** | Int64 UpDownCounter |
| **Unit** | `{call}` |
| **Description** | EXPERIMENTAL. Number of RPCs currently waiting for a load balancing pick. |
| **Labels** | `grpc.target` |
| **Default Enabled** | `false` (experimental, opt-in) |

The counter is incremented (`+1`) when a pick is first deferred and decremented
(`-1`) when the wait ends (successful pick or cancellation). This matches
the pattern used by `grpc.subchannel.open_connections` ([A94]) and
`grpc.tcp.connection_count` ([A80]).

#### Label Definitions

| Label | Type | Description |
|---|---|---|
| `grpc.target` | String | The target URI of the channel. Required. |
| `grpc.lb.delay_reason` | String | The bounded delay reason token from the picker. Optional label. |
| `grpc.method` | String | The full method name of the RPC (e.g., `/pkg.Service/Method`). Optional label. Available from `PickInfo` at pick time. Primarily useful for policies like RLS where wait behavior varies by method. |

### Delay Reason Token Semantics

Delay reason tokens are static, bounded strings following the grammar:

```
token       = terminal | prefix ":" token
prefix      = policy_prefix
terminal    = policy_name ":" state
policy_name = "pick_first" | "round_robin" | "ring_hash" | "rls" | "cds"
```

Dynamic values (IP addresses, cluster names, endpoint identifiers) are
**strictly forbidden** in tokens.

#### Terminal Tokens

Terminal tokens are emitted by leaf policies when they cannot complete a pick.

| Policy | Token | When Emitted |
|---|---|---|
| `pick_first` | `pick_first:connecting` | No READY subchannel; connection attempt in progress. |
| `round_robin` | `round_robin:connecting` | No READY subchannel in the round-robin set. |
| `ring_hash` | `ring_hash:connecting` | The hashed (or scanned) endpoint is in CONNECTING or IDLE state. |
| `rls` | `rls:lookup_pending` | Waiting for an RLS server response for the request key. |
| `rls` | `rls:throttled` | RLS requests are being throttled due to server errors. |
| `cds` | `cds:discovery_pending` | Waiting for the xDS CDS resource. |

#### Delegating Prefixes

Delegating policies prepend a prefix to their child picker's token.

| Policy | Prefix | Notes |
|---|---|---|
| `priority` | `priority_p{N}:` | N is the 0-based priority index. Typical deployments use ≤5 priorities, bounding cardinality. |
| `cluster_manager` | *(per-pick)* | Routes RPCs to different child pickers per-request. The token is the child picker's token, resolved at pick time. See Per-Pick Tokens. |
| `rls` | `rls:` | When routing to a resolved child policy (post-lookup). |
| `cds` | `cds:` | When delegating to a child policy (post-discovery). |
| `wrr_locality` | *(transparent)* | Passes child tokens through unmodified. |
| `outlier_detection` | *(transparent)* | Passes child tokens through unmodified; OD does not itself cause queueing. |

#### Token Composition

For most policies, token composition occurs at **picker construction time**, not
on the pick path. When a delegating policy creates its picker, it reads the
child picker's token and stores the composite string as a member of its own
picker. This ensures zero allocations during `Pick()`.

**Example flow** for `priority → pick_first`:
1. `pick_first` creates a picker with token `"pick_first:connecting"`.
2. `priority` wraps the child, creating its picker with token
   `"priority_p0:pick_first:connecting"` (stored as a member string).

When a delegating policy has a **READY** child (no queueing), the token is the
empty string `""`, and the pick completes without queueing.

#### Per-Pick Tokens

Some policies make per-request routing decisions within `Pick()`, causing the
delay reason to vary across RPCs handled by the same picker. These policies
provide their token via the pick result rather than a picker-level method:

- **RLS**: A single `rlsPicker` performs a per-request cache lookup. A cache
  miss delays with `rls:lookup_pending`, while a cache hit that delegates to a
  child in CONNECTING state delays with `rls:pick_first:connecting`. The token
  must be determined inside `Pick()` and attached to the pending result.

- **cluster_manager**: Routes RPCs to different child pickers based on the
  cluster selected by xDS routing. Different RPCs may reach different child
  pickers in different states. The token is read from whichever child picker
  handled the specific RPC.

The mechanism for attaching per-pick tokens is described in the language-specific
API sections below.

### Language-Specific API Changes

> [!NOTE]
> The code examples in the following sections are illustrative and simplified to demonstrate the design intent. Actual implementations may vary slightly across languages to conform to idiomatic patterns and existing codebase structure.

#### Go

Go's `Picker` interface has a single `Pick()` method. Adding a method would
break all existing implementations. Two mechanisms are provided for supplying
queue reason tokens:

**Picker-level token** — for policies with uniform picker state (pick_first,
round_robin, ring_hash, priority). An optional interface that pickers may
implement:

```go
// In package balancer

// DelayMetricTokener is an optional interface that Pickers can implement
// to provide a bounded metric token describing why this picker delays RPCs.
// The token MUST be a static, pre-computed string with no dynamic components.
type DelayMetricTokener interface {
    // DelayMetricToken returns the bounded delay reason token.
    // Returns "" if the picker does not delay RPCs.
    DelayMetricToken() string
}
```

**Per-pick token** — for policies where the delay reason varies per-RPC (RLS,
cluster_manager). A structured error type that wraps `ErrNoSubConnAvailable`
with a token:

```go
// In package balancer

// PendingPickError is returned by Pick() to signal a delay with a per-pick
// metric token. It is equivalent to ErrNoSubConnAvailable but carries
// a delay reason token that may vary per-RPC.
type PendingPickError struct {
    // MetricToken is the bounded delay reason token for this specific pick.
    MetricToken string
}

func (e *PendingPickError) Error() string { return "no SubConn available" }
func (e *PendingPickError) Is(target error) bool {
    return target == ErrNoSubConnAvailable
}
```

The recording loop resolves the token with the following precedence: if the
error is a `*PendingPickError`, use its `MetricToken`; otherwise, check whether the
picker implements `DelayMetricTokener`; otherwise, use the empty string.

Built-in pickers (`pick_first`, `round_robin`, `ring_hash`, `priority`, etc.)
implement `DelayMetricTokener`. Custom LB policies that implement neither
mechanism report an empty token.

**Tracing Signals** — To support the new child span design in Go without introducing OpenTelemetry dependencies in the core library, we introduce new internal stats signals. These are emitted by the channel infrastructure and handled by the registered stats handler (e.g., the OpenTelemetry plugin):

```go
// In package stats

// DelayStart indicates that the RPC has entered a delay period.
// This is a signal to the stats handler to start a new child span.
type DelayStart struct {
    // Reason is the bounded token describing the cause of delay.
    Reason string
}

func (*DelayStart) IsClient() bool { return true }
func (*DelayStart) isRPCStats()   {}

// DelayEnd indicates that the current delay period has resolved.
// This is a signal to the stats handler to end the child span.
type DelayEnd struct{}

func (*DelayEnd) IsClient() bool { return true }
func (*DelayEnd) isRPCStats()   {}
```

**Example — `pick_first` (picker-level):**
```go
type pfPicker struct {
    subConn    balancer.SubConn
    delayToken string // set at construction: "pick_first:connecting" or ""
}

func (p *pfPicker) Pick(balancer.PickInfo) (balancer.PickResult, error) {
    return balancer.PickResult{SubConn: p.subConn}, nil
}

func (p *pfPicker) DelayMetricToken() string {
    return p.delayToken
}
```

**Example — `rls` (per-pick):**
```go
func (p *rlsPicker) Pick(info balancer.PickInfo) (balancer.PickResult, error) {
    // ... cache lookup using info.FullMethodName and request keys ...
    switch {
    case dcEntry == nil && pendingEntry == nil:
        p.sendRouteLookupRequest(cacheKey, ...)
        return balancer.PickResult{},
            &balancer.PendingPickError{MetricToken: "rls:lookup_pending"}
    case dcEntry == nil && pendingEntry != nil:
        return balancer.PickResult{},
            &balancer.PendingPickError{MetricToken: "rls:lookup_pending"}
    case dcEntry != nil:
        // Delegate to child policy — child's Pick() may return its
        // own PendingPickError or ErrNoSubConnAvailable.
        pr, err := childPicker.Pick(info)
        if qe, ok := err.(*balancer.PendingPickError); ok {
            qe.MetricToken = "rls:" + qe.MetricToken
        }
        return pr, err
    }
}
```

**Example — `priority` (picker-level, delegating):**
```go
type priorityPicker struct {
    childPicker balancer.Picker
    delayToken  string // pre-composed for static children
    prefix      string // "priority_p0:"
}

func newPriorityPicker(child balancer.Picker, priorityName string) balancer.Picker {
    prefix := priorityName + ":"
    p := &priorityPicker{childPicker: child, prefix: prefix}
    if dt, ok := child.(balancer.DelayMetricTokener); ok {
        childToken := dt.DelayMetricToken()
        if childToken != "" {
            p.delayToken = prefix + childToken
        }
    }
    return p
}

func (p *priorityPicker) Pick(info balancer.PickInfo) (balancer.PickResult, error) {
    res, err := p.childPicker.Pick(info)
    if err != nil {
        // If child returned a per-pick token, prepend our prefix
        if pe, ok := err.(*balancer.PendingPickError); ok {
            return res, &balancer.PendingPickError{
                MetricToken: p.prefix + pe.MetricToken,
            }
        }
    }
    return res, err
}

func (p *priorityPicker) DelayMetricToken() string {
    return p.delayToken
}
```

#### Java

Java's `PickResult` is `final` and `@Immutable`. The per-pick token is carried
on the `PickResult` itself via a new factory method, which naturally supports
policies like RLS where the delay reason varies per-RPC. The `SubchannelPicker`
also gets a default method for delegating policies that have uniform state:

```java
// In LoadBalancer.PickResult

// New field (private, immutable)
@Nullable private final String delayMetricToken;

// New factory method for delayed results with a reason token
public static PickResult withNoResult(String delayMetricToken) {
    return new PickResult(null, null, Status.OK, false, null, delayMetricToken);
}

// Getter
public String getDelayMetricToken() {
    return delayMetricToken != null ? delayMetricToken : "";
}
```

The existing `withNoResult()` continues to return the `NO_RESULT` singleton with
an empty token. The `SubchannelPicker` also gets a default method for
delegating policies to read the child's token:

```java
// In LoadBalancer.SubchannelPicker

/**
 * Returns the bounded metric token describing why this picker delays RPCs.
 * Default returns empty string (no delay information).
 */
public String getDelayMetricToken() { return ""; }
```

**Tracing APIs** — To support the new child span design in Java (and avoiding span events), we add new default methods to `ClientStreamTracer`. These allow the channel to notify the tracer when a delay starts and ends. (Note: `ClientStreamTracer` corresponds to `CallAttemptTracer` in C++. If a separate `CallTracer` is introduced for call-level tracking, it would have similar methods).

```java
// In package io.grpc

public abstract class ClientStreamTracer extends StreamTracer {
    // ... existing methods ...

    /**
     * Called when the RPC enters a delay period (e.g., waiting for a pick).
     * The implementation should create a new child span for the delay.
     *
     * @param reason The bounded token describing the cause of delay.
     */
    public void recordDelayStart(String reason) {}

    /**
     * Called when the current delay period resolves.
     * The implementation should end the child span.
     */
    public void recordDelayEnd() {}
}
```

**Example implementation in `PickFirstLeafLoadBalancer`:**
```java
private class Picker extends SubchannelPicker {
    private final String delayMetricToken;

    Picker(String token) { this.delayMetricToken = token; }

    @Override
    public PickResult pickSubchannel(PickSubchannelArgs args) {
        // When delaying, use the token-aware factory:
        return PickResult.withNoResult(delayMetricToken);
    }

    @Override
    public String getDelayMetricToken() { return delayMetricToken; }
}
```

**Example implementation in a delegating load balancer (`PriorityLoadBalancer`):**
```java
private class PriorityPicker extends SubchannelPicker {
    private final SubchannelPicker childPicker;
    private final String delayMetricToken;
    private final String prefix;

    PriorityPicker(SubchannelPicker child, String prefix) {
        this.childPicker = child;
        this.prefix = prefix;
        // Pre-compose for static children
        String childToken = child.getDelayMetricToken();
        this.delayMetricToken = childToken.isEmpty() ? "" : prefix + childToken;
    }

    @Override
    public PickResult pickSubchannel(PickSubchannelArgs args) {
        PickResult result = childPicker.pickSubchannel(args);
        // If child returned a per-pick delay token, prepend our prefix
        String childDynamicToken = result.getDelayMetricToken();
        if (!childDynamicToken.isEmpty() && !childDynamicToken.equals(childPicker.getDelayMetricToken())) {
             return PickResult.withNoResult(prefix + childDynamicToken);
        }
        return result;
    }

    @Override
    public String getDelayMetricToken() {
        return delayMetricToken;
    }
}
```

#### C++ (Core)

The `PickResult::Queue` struct is extended to hold a `string_view` carrying the
per-pick token. Since `Queue` is returned from `Pick()`, this naturally supports
policies where the token varies per-RPC. The `string_view` points to a
picker-owned string, involving zero dynamic allocations on the pick path:

```cpp
// In LoadBalancingPolicy::PickResult

struct Queue {
    // Bounded queue reason token. Points to a string owned by the Picker.
    // The Picker's lifetime exceeds the pick, so this is safe.
    // Empty string_view means no token is provided.
    absl::string_view metric_token;

    Queue() = default;
    explicit Queue(absl::string_view token) : metric_token(token) {}
};
```

The `SubchannelPicker` base class gets a virtual method for delegating policies:

```cpp
class SubchannelPicker : public DualRefCounted<SubchannelPicker> {
 public:
    virtual PickResult Pick(PickArgs args) = 0;

    // Returns the bounded delay reason token. Default returns empty.
    virtual absl::string_view GetDelayMetricToken() const { return ""; }
};
```

**Tracing APIs** — To support the new child span design in C++ (and avoiding span events), we add new virtual methods to both `ClientCallTracerInterface` and `ClientCallTracerInterface::CallAttemptTracer`. These allow the channel to notify the tracer when a delay starts and ends at both the call and attempt levels:

```cpp
// In src/core/telemetry/call_tracer.h

class ClientCallTracerInterface : public CallTracerAnnotationInterface {
 public:
    // ... existing methods ...

    // Records the start of a delay period at the call level.
    virtual void RecordDelayStart(absl::string_view reason) = 0;

    // Records the end of the current delay period at the call level.
    virtual void RecordDelayEnd() = 0;

    class CallAttemptTracer : public CallTracerInterface {
     public:
        // ... existing methods ...

        // Records the start of a delay period at the attempt level.
        virtual void RecordDelayStart(absl::string_view reason) = 0;

        // Records the end of the current delay period at the attempt level.
        virtual void RecordDelayEnd() = 0;
    };
};
```

**Example implementation in a leaf picker:**
```cpp
class PickFirstQueuePicker final : public SubchannelPicker {
 public:
    PickResult Pick(PickArgs) override {
        return PickResult::Queue(kToken);
    }
    absl::string_view GetDelayMetricToken() const override { return kToken; }

 private:
    static constexpr absl::string_view kToken = "pick_first:connecting";
};
```

**Example delegating picker:**
```cpp
class PriorityPicker final : public SubchannelPicker {
 public:
    PriorityPicker(int priority_index,
                   RefCountedPtr<SubchannelPicker> child)
        : child_(std::move(child)) {
        // Compose token at construction time (stored as member string).
        auto child_token = child_->GetDelayMetricToken();
        if (!child_token.empty()) {
            composite_token_ = absl::StrCat("priority_p",
                                             priority_index, ":",
                                             child_token);
        }
    }

    PickResult Pick(PickArgs args) override {
        auto result = child_->Pick(args);
        if (auto* q = absl::get_if<PickResult::Queue>(&result.result)) {
            q->metric_token = composite_token_;
        }
        return result;
    }

    absl::string_view GetDelayMetricToken() const override {
        return composite_token_;
    }

 private:
    RefCountedPtr<SubchannelPicker> child_;
    std::string composite_token_;  // owned by this picker
};
```

### Language-Specific Recording Logic

Recording is performed by the client channel's pick deferring infrastructure, not
by the LB policy itself. The LB policy provides the token; the channel measures
time and records the metric.

#### Go — `picker_wrapper.go`

In the `pick()` method's blocking loop, the recording logic resolves the token
from either a `PendingPickError` or the `DelayMetricTokener` interface, and emits a
histogram observation each time the token changes (per-segment emission):

```go
func (pw *pickerWrapper) pick(ctx context.Context, failfast bool,
    info balancer.PickInfo) (pick, error) {
    var delayStartTime time.Time
    var delayToken string
    var delayed bool
    method := info.FullMethodName

    for {
        // ... existing picker load and blocking logic ...

        pickResult, err := p.Pick(info)
        if err != nil {
            if errors.Is(err, balancer.ErrNoSubConnAvailable) {
                // Resolve per-pick token from PendingPickError, or fall
                // back to picker-level DelayMetricTokener.
                currentToken := ""
                if qe, ok := err.(*balancer.PendingPickError); ok {
                    currentToken = qe.MetricToken
                } else if qt, ok := p.(balancer.DelayMetricTokener); ok {
                    currentToken = qt.DelayMetricToken()
                }

                if !delayed {
                    // First delay event — start timing, increment counter.
                    delayed = true
                    delayStartTime = time.Now()
                    delayToken = currentToken
                    rpcWaitingMetric.Record(metricsRecorder, 1, target)
                } else if currentToken != delayToken {
                    // Token changed (new picker with different state).
                    // Emit segment for the previous token.
                    duration := time.Since(delayStartTime).Seconds()
                    pickDelayDurationMetric.Record(metricsRecorder,
                        duration, target, delayToken, method)
                    // Start new segment.
                    delayStartTime = time.Now()
                    delayToken = currentToken
                }
                continue
            }
            // ... existing error handling ...
        }

        // Pick succeeded. If we were delayed, record final segment.
        if delayed {
            duration := time.Since(delayStartTime).Seconds()
            pickDelayDurationMetric.Record(metricsRecorder,
                duration, target, delayToken, method)
            rpcWaitingMetric.Record(metricsRecorder, -1, target)
            if attemptSpan != nil {
                attemptSpan.AddEvent("Delayed LB pick complete",
                    attribute.Float64("delay_duration", duration),
                    attribute.String("delay_reason", delayToken))
            }
        }
        // ... existing transport handling ...
    }
}
```

On context cancellation (existing `case <-ctx.Done():` block), the same
recording logic applies:
```go
case <-ctx.Done():
    if delayed {
        duration := time.Since(delayStartTime).Seconds()
        pickDelayDurationMetric.Record(metricsRecorder,
            duration, target, delayToken, method)
        rpcWaitingMetric.Record(metricsRecorder, -1, target)
        if attemptSpan != nil {
            attemptSpan.AddEvent("Delayed LB pick complete",
                attribute.Float64("delay_duration", duration),
                attribute.String("delay_reason", delayToken))
        }
    }
    // ... existing error return ...
```

The `metricsRecorder` and `target` are available from the `ClientConn` that owns
the `pickerWrapper`.

#### Java — `DelayedClientTransport.java`

When a `PendingStream` is created in `createPendingStream()`, the delay start
time and token are captured:

```java
private PendingStream createPendingStream(PickSubchannelArgs args,
    ClientStreamTracer[] tracers, PickResult pickResult) {
    PendingStream pendingStream = new PendingStream(args, tracers);
    pendingStream.delayStartNanos = System.nanoTime();
    
    // Resolve token: prefer PickResult token (dynamic), fallback to Picker token (static)
    String token = pickResult != null ? pickResult.getDelayMetricToken() : "";
    if (token.isEmpty() && pickerState.lastPicker != null) {
        token = pickerState.lastPicker.getDelayMetricToken();
    }
    pendingStream.delayToken = token;
    // ... existing logic ...
}
```

When the stream is dequeued in `reprocess()` (transport obtained), or cancelled,
the duration is recorded:

```java
if (transport != null) {
    long durationNanos = System.nanoTime() - stream.delayStartNanos;
    double durationSeconds = durationNanos / 1_000_000_000.0;
    metricsRecorder.recordPickDelayDuration(durationSeconds,
        target, stream.delayToken);
    stream.getTracer().recordAnnotation("Delayed LB pick complete",
        Attributes.of(
            "delay_duration", durationSeconds,
            "delay_reason", stream.delayToken));
    // ... existing stream creation ...
}
```

#### C++ (Core) — `load_balanced_call_destination.cc`

In the pick loop, when `PickResult::Queue` is returned, the start time and token
are captured. If the token changes during the wait, the current segment is recorded
and a new one begins. When the pick eventually succeeds or the call is cancelled, the
final duration is recorded:

```cpp
// In the pick processing lambda:
auto delay_func = [&](LoadBalancingPolicy::PickResult::Queue* delay_pick) {
    std::string current_token = std::string(delay_pick->metric_token);
    if (!delay_start_time.has_value()) {
        delay_start_time = Timestamp::Now();
        delay_token = current_token;
    } else if (current_token != delay_token) {
        // Token changed: Emit segment for the previous token
        Duration delay_duration = Timestamp::Now() - *delay_start_time;
        stats_plugin_group.RecordHistogram(
            kPickDelayDurationHandle,
            delay_duration.seconds(),
            {target}, {delay_token});
        // Start new segment
        delay_start_time = Timestamp::Now();
        delay_token = current_token;
    }
    return Continue{};
};

// On successful pick (in the completion callback):
if (delay_start_time.has_value()) {
    Duration delay_duration = Timestamp::Now() - *delay_start_time;
    stats_plugin_group.RecordHistogram(
        kPickDelayDurationHandle,
        delay_duration.seconds(),
        {target}, {delay_token});
    call_tracer->RecordAnnotation("Delayed LB pick complete", {
        {"delay_duration", absl::StrCat(delay_duration.seconds())},
        {"delay_reason", delay_token}
    });
}
```

#### C-Core Specifics: RLS Cache Misses and HTTP/2 Max Concurrent Streams

In `grpc-core`, the boundary between Load Balancing and the Transport layer introduces two highly specific queuing scenarios that must be handled distinctly in the `PickResult::Queue` tokenization.

**1. RLS Cache Misses (Dynamic Token Lifetime)**
Unlike static policies (like `pick_first`), the RLS policy in C-Core evaluates targets dynamically per-call. When a cache miss occurs, the RLS policy initiates a control-plane request and returns `PickResult::Queue`.
*   **The C-Core Constraint:** The `PickResult::Queue` struct uses an `absl::string_view` to prevent allocations on the hot path. However, because an RLS cache miss is dynamic, the string it points to must safely outlive the pick.
*   **Implementation:** The RLS LB Policy must maintain a static, pre-allocated pool of `std::string` constants for its state transitions (e.g., `static const std::string kRlsPending = "rls:lookup_pending";`). During a cache miss `Pick()`, the policy returns a `string_view` pointing explicitly to this constant memory address, ensuring safe reference lifecycle without dynamic memory allocation per RPC.

**2. MAX_CONCURRENT_STREAMS (Transport-Level Delay)**
In C-Core, waiting can occur even when the LB Policy successfully finds a `READY` subchannel. If the HTTP/2 transport for that subchannel has reached its `SETTINGS_MAX_CONCURRENT_STREAMS` limit, the `grpc_call` must wait.
*   **The Distinction:** This is a *transport* delay, not an *LB routing* delay.
*   **Implementation:** The LB policy's `Pick()` method will actually succeed and return a `READY` subchannel. However, the `client_channel` filter will detect the transport exhaustion. To maintain observability, the `client_channel` filter itself will inject a synthetic token: `"transport:max_concurrent_streams"`. The delay timer will start, and the RPC will wait in the filter's pending list until a stream becomes available or the context deadline is exceeded. This clearly separates network capacity limits from LB routing failures in the resulting telemetry.

### Metric Instrument Registration

#### Go

```go
import estats "google.golang.org/grpc/experimental/stats"

var pickDelayDurationMetric = estats.RegisterFloat64Histo(
    estats.MetricDescriptor{
        Name:           "grpc.lb.pick_delay_duration",
        Description:    "EXPERIMENTAL. Time an RPC spent waiting " +
                        "for a load balancing pick.",
        Unit:            "s",
        Labels:          []string{"grpc.target", "grpc.lb.delay_reason"},
        OptionalLabels: []string{"grpc.method"},
        Default:        false,
    })

var rpcWaitingMetric = estats.RegisterInt64UpDownCount(
    estats.MetricDescriptor{
        Name:           "grpc.lb.rpc_waiting",
        Description:    "EXPERIMENTAL. Number of RPCs currently waiting " +
                        "for a load balancing pick.",
        Unit:           "{call}",
        Labels:         []string{"grpc.target"},
        Default:        false,
    })
```

#### Java

```java
private static final LongHistogramInstrumentDescriptor PICK_DELAY_DURATION =
    InstrumentRegistry.registerDoubleHistogram(
        "grpc.lb.pick_delay_duration",
        "EXPERIMENTAL. Time an RPC spent waiting "
            + "for a load balancing pick.",
        "s",
        List.of("grpc.target", "grpc.lb.delay_reason"),  // required labels
        List.of("grpc.method"),                           // optional labels
        false);                                          // default disabled

private static final LongUpDownCounterMetricInstrument RPC_WAITING =
    InstrumentRegistry.registerLongUpDownCounter(
        "grpc.lb.rpc_waiting",
        "EXPERIMENTAL. Number of RPCs currently waiting "
            + "for a load balancing pick.",
        "{call}",
        List.of("grpc.target"),
        List.of(),
        false);
```

#### C++ (Core)

```cpp
const auto kPickDelayDurationHandle =
    GlobalInstrumentsRegistry::RegisterDoubleHistogram(
        "grpc.lb.pick_delay_duration",
        "EXPERIMENTAL. Time an RPC spent waiting "
        "for a load balancing pick.",
        "s",
        /*label_keys=*/{"grpc.target", "grpc.lb.delay_reason"},
        /*optional_label_keys=*/{"grpc.method"},
        /*enable_by_default=*/false);

const auto kRpcWaitingHandle =
    GlobalInstrumentsRegistry::RegisterUpDownInt64Counter(
        "grpc.lb.rpc_waiting",
        "EXPERIMENTAL. Number of RPCs currently waiting "
        "for a load balancing pick.",
        "{call}",
        /*label_keys=*/{"grpc.target"},
        /*optional_label_keys=*/{},
        /*enable_by_default=*/false);
```

### Metric Stability

Per [A79][A79], this metric starts as **experimental** and **disabled by
default**. Users must explicitly opt in via their OpenTelemetry plugin
configuration. The metric description is prefixed with `EXPERIMENTAL.` to
signal this status.

The metric will be promoted to stable after it has been implemented and validated
in all three languages, with at least one release cycle of user feedback.

### Temporary environment variable protection

The metric will be guarded by the environment variable
`GRPC_EXPERIMENTAL_ENABLE_PICK_DELAY_METRIC`. When set to `true`, the metric
will be registered and reported even if the user has not explicitly enabled it
via the OpenTelemetry plugin configuration. This guard will be removed once the
feature is deemed stable.

### Tracing Enhancement

To provide high-fidelity visibility into delays, this proposal introduces a **generic delay framework** using child spans, rather than just adding events to existing spans. This allows capturing the start, end, and specific reason for any processing delay (not limited to load balancing).

When an RPC experiences a delay (such as waiting for a load balancer pick), the channel will create a **new child span** named simply **`"Delay"`**.

This child span will have the following attributes:
*   `delay_type`: Describes the category of delay. For this proposal, it will be `"load_balancing"`.
*   `delay_reason`: The specific bounded token describing why the delay happened (e.g., `"pick_first:connecting"`, `"rls:lookup_pending"`).

The span is created when the delay condition begins and is ended when the condition resolves or transitions to a new reason (in which case a new segment span is created). This provides a clear, time-accurate visualization of delay segments in the trace.

To support this in each language:
*   **C++**: New APIs in `ClientCallTracerInterface::CallAttemptTracer` to start and end delay spans.
*   **Java**: New default methods in `ClientStreamTracer` to start and end delay spans.
*   **Go**: New internal stats signals (`stats.DelayStart` and `stats.DelayEnd`) to notify the stats handler when to manage the delay span lifecycle.

## Rationale

The metric is recorded by the client channel's pick deferring infrastructure
rather than by the LB policy itself, because the LB policy knows its internal
state but does not know how many RPCs are waiting or for how long. Only the
channel physically buffers RPCs and can accurately measure per-RPC delay
duration. This mirrors the existing design where per-call metrics ([A66]) are
recorded by the channel, not by LB policies.

Two mechanisms are provided for supplying delay reason tokens — a picker-level
optional interface (`DelayMetricTokener`) and a per-pick error/result type
(`PendingPickError` / `PickResult.withNoResult(token)` / `Queue::metric_token`). Most
LB policies have uniform picker state: all RPCs see the same delay reason from a
given picker instance (pick_first, round_robin, ring_hash, priority). These
policies use the picker-level mechanism, which involves zero allocations on the
pick path. However, RLS and cluster_manager make per-request routing decisions
within `Pick()` — RLS does a per-request cache lookup, and cluster_manager
routes RPCs to different child pickers based on the xDS-selected cluster. For
these policies, the delay reason varies per-RPC, requiring the token to be
determined inside `Pick()` and attached to the pending result.

The recording loop emits one histogram observation per delay reason segment
rather than a single observation per RPC. An RPC that transitions through
multiple delay reasons (e.g., `rls:lookup_pending` for 10 seconds, then
`rls:pick_first:connecting` for 0.1 seconds after the lookup completes) produces
two histogram observations, one for each segment. This per-segment approach
provides visibility into how much time each layer of the LB policy tree
contributed to the total pick delay. Without it, a single 10.1-second
observation attributed to `rls:lookup_pending` would obscure the fact that the
RLS lookup accounted for the vast majority of the delay.

The `grpc.lb.rpc_waiting` up-down counter is provided because the
histogram metric is emitted only when a wait ends. RPCs that are
indefinitely delayed (e.g., target unreachable, infinite backoff)
never emit a histogram observation. The counter gives operators a real-time
count of currently-waiting RPCs, enabling alerting on stuck traffic. Every LB
policy in the tree can cause indefinite waiting (pick_first cycling through
CONNECTING/TF, RLS server unreachable, CDS resource never arriving), making this
counter essential. An up-down counter is used rather than a callback gauge
because the recording loop has the exact synchronous entry/exit points, matching
the pattern used by `grpc.subchannel.open_connections` ([A94]) and
`grpc.tcp.connection_count` ([A80]).

`grpc.method` is included as an optional label because some policies (RLS,
cluster_manager) make routing decisions based on the method name, causing wait
behavior to vary across methods. For simpler policies (pick_first,
round_robin), the delay behavior is method-independent, but operators may still
want per-method delay analysis to correlate with per-call attempt metrics. As an
optional label, it defaults to off and only adds cardinality when explicitly
enabled.

`wrr_locality` and `outlier_detection` are treated as transparent because
neither makes independent delaying decisions. `wrr_locality` is a configuration
wrapper; `outlier_detection` manipulates subchannel state but the actual
delaying is decided by the child policy's picker. When outlier detection ejects
100% of endpoints, the child policy (e.g., pick_first) enters
TRANSIENT_FAILURE and returns an error rather than a pending result, so no delay
metric is recorded. In partial ejection scenarios, the child is genuinely in
CONNECTING state and the token is accurate. Operators can correlate delay
duration with the existing `grpc.lb.outlier_detection.ejections_enforced` metric
([A91]) for root cause analysis.

The tracing enhancement reuses the same bounded delay reason token as the metric
label rather than introducing a separate high-cardinality trace reason string.
This keeps the API surface minimal — one token serves both metrics and traces —
and avoids the need for pickers to maintain two separate strings. If richer,
dynamic trace context (e.g., IP addresses, RLS keys) is needed in the future,
it can be added as additional span event attributes without changing the token
API.

## Implementation

Implementation will proceed in Go, Java, and C++ (Core), in that order.

[A66]: A66-otel-stats.md
[A72]: A72-open-telemetry-tracing.md
[A78]: A78-grpc-metrics-wrr-pf-xds.md
[A79]: A79-non-per-call-metrics-architecture.md
[A80]: A80-tcp-metrics.md
[A91]: A91-outlier-detection-metrics.md
[A94]: A94-subchannel-otel-metrics.md
[A56]: A56-priority-lb-policy.md
[A50]: A50-xds-outlier-detection.md