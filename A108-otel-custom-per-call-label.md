A108: OpenTelemetry Custom Per-Call Metric Label
----
* Author(s): [Eric Anderson](https://github.com/ejona86)
* Approver: @markdroth
* Status: Ready for Implementation
* Implemented in: <language, ...>
* Last updated: 2025-12-05
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

Add a custom label for applications to define their own dimension for per-call
metrics.

## Background

[gRFC A66][]'s per-call metrics are being used and found to be useful. However,
applications can have their own data relevant to the metric. [Baggage][] is
OpenTelemetry's answer to this use-case.

[gRFC A79][] defined a standard metric architecture across gRPC implementations,
but it explicitly did not include per-call metrics. For this reason, it does not
include a "context" in its API, which is necessary for propagating baggage.

While supporting baggage is possible in some languages, it is currently counter
to the performance optimizations C++ has been considering for its metrics.
Determining baggage's cross-language role will be an ongoing discussion, and
depending how that discussion finalizes this gRFC will either be a stop-gap or
the long-term, cross-language, preferred solution.

It should also be noted that OpenTelemetry is not the only potential metric
implementation. Other metric libraries may not have the concept of baggage.
Using baggage also requires using an OpenTelemetry API, and a library using gRPC
may not also be aware of the precise metric library that will be used by the
application, so a gRPC-defined label allows callers to add context to their RPCs
without mandating a particular metric implementation.

[Baggage]: https://opentelemetry.io/docs/concepts/signals/baggage/
[gRFC A79]: A79-non-per-call-metrics-architecture.md

### Related Proposals:
* [gRFC A66][]: OpenTelemetry Metrics
* [gRFC A96][]: OpenTelemetry Metrics for Retries

[gRFC A66]: A66-otel-stats.md
[gRFC A96]: A96-retry-otel-stats.md

## Proposal

gRPC will add an API for applications to provide a custom label value when
starting RPCs. That value will be propagated to an optional
`grpc.client.call.custom` label of all client-side call-related instruments.
When not specified, the value will be empty string. The instruments that would
gain this label:

* Per-call instruments (gRFC A66)
  * `grpc.client.call.duration`
  * `grpc.client.attempt.started`
  * `grpc.client.attempt.duration`
  * `grpc.client.attempt.sent_total_compressed_message_size`
  * `grpc.client.attempt.rcvd_total_compressed_message_size`
* Retry instruments (gRFC A96)
  * `grpc.client.call.retries`
  * `grpc.client.call.transparent_retries`
  * `grpc.client.call.hedges`
  * `grpc.client.call.retry_delay`
* Other per-call instruments, e.g., those added by an LB policy, are encouraged
  to support this label
  * RLS is the only such case today, but is not defined in a gRFC

The per-call and retry instruments are implemented directly in gRPC's
OpenTelemetry module, so gRPC must propagate the value to the module. For LB
policies to add it to their own instruments, gRPC must propagate the value to
the LB API during picks.

### Java

gRPC Java will add a new API `io.grpc.Grpc.CALL_OPTION_CUSTOM_LABEL` with type
`CallOptions.Key<String>`. The call option will default to the empty string when
unset.

The RPC's `CallOption` is visible to `ClientInterceptor` (passed explicitly),
`ClientStreamTracer.Factory` (inside `StreamInfo`), and `SubchannelPicker`
(inside `PickSubchannelArgs`), so no additional plumbing is necessary.

### Go

gRPC Go will add a new API to set and get the custom label value in
`context.Context`. `ctx` is already passed to
`grpc.UnaryClientInterceptor`/`grpc.StreamClientInterceptor`, `stats.Handler`,
and `balancer.Picker`, so no additional plumbing is necessary.

### C++

gRPC C++ will add an API to ClientContext. Precise details TBD.

### Metric Stability

The label added in this proposal will start as experimental. The label is
intended to always be explicitly enabled by the application; even once stable,
it will not be enabled by default.

### Temporary environment variable protection

The new label is only enabled via an API call, so no environment variable
protection is necessary.

## Rationale

A single label is restrictive. We could allow applications to have multiple
(e.g., custom_label1, custom_label2, custom_label3). However, a single label
provides a substantial amount of the value to applications. This gRFC is
optimized for expediency and low complexity, and adding more labels has
diminishing returns. Also, more custom labels can be added in a future gRFC with
no additional complexity in gRPC compared to defining them now; it is a
deferrable decision.

There is obvious overlap with baggage. As discussed in the Background section,
baggage is relevant, but is considered a future enhancement for expediency.

## Implementation

@ejona86 will be implementing immediately in grpc-java. A draft is available in
[PR 12552](https://github.com/grpc/grpc-java/pull/12552). Other languages will
follow, with Go work starting potentially in a few weeks.
