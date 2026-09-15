A109: Target Attribute Filter for OpenTelemetry Metrics
----
* Author(s): [becomeStar](https://github.com/becomeStar)
* Approver: a11r
* Status: In Review
* Implemented in: Java ([grpc/grpc-java#12587][])
* Last updated: 2026-07-07
* Discussion at: (to be filled after the grpc-io notification is sent)

## Abstract

Add an optional Java API to control how the `grpc.target` attribute is recorded
in gRPC Java OpenTelemetry client metrics, allowing rejected targets to be
mapped to `"other"` to reduce metric cardinality, while preserving existing
behavior by default.

## Background

[gRFC A66][]'s per-call metrics include the `grpc.target` attribute, which can
have very high cardinality in large-scale deployments where clients connect to
many different server targets. This high cardinality can cause OpenTelemetry SDK
warnings (see [issue #12322](https://github.com/grpc/grpc-java/issues/12322))
when the maximum allowed cardinality (default 2000, warning at 1999+) is
exceeded for instruments such as:

* `grpc.client.attempt.started`
* `grpc.client.attempt.duration`
* `grpc.client.attempt.sent_total_compressed_message_size`
* `grpc.client.attempt.rcvd_total_compressed_message_size`
* `grpc.client.call.duration`

A workaround exists using OpenTelemetry Views with `setAttributeFilter()` to
discard `grpc.target` entirely, but this is an all-or-nothing approach and not
a suitable replacement for selective filtering.

A66 left overriding recorded target names as possible future work, and
[grpc/proposal#431][] later removed the target attribute filter requirement
from A66 because the requirements had not yet arisen. [grpc/grpc-java#12322][]
is such a requirement: Java users can exceed OpenTelemetry cardinality limits
when clients connect to thousands of targets.

The Java implementation was reviewed and merged in [grpc/grpc-java#12587][] on
2026-01-09.

### Related History: C++ target filtering

gRPC C++ previously added a similar API in [grpc/grpc#34285][] based on the
original A66 proposal, then marked it deprecated in [grpc/grpc#36567][] after
[grpc/proposal#431][] removed the target attribute filter requirement from A66.
This proposal treats the C++ API as related history rather than proposing
broader cross-language behavior.

The current C++ public header says that the filtering only applies to per-call
metrics and does not filter `grpc.target` on load-balancing policy metrics such
as those defined by [gRFC A78][]. In other words, this proposal intentionally
does not define a general mechanism for rewriting every `grpc.target` attribute
emitted by gRPC.

[gRFC A66]: A66-otel-stats.md
[grpc/grpc-java#12322]: https://github.com/grpc/grpc-java/issues/12322
[grpc/grpc-java#12587]: https://github.com/grpc/grpc-java/pull/12587
[grpc/grpc#34285]: https://github.com/grpc/grpc/pull/34285
[grpc/grpc#36567]: https://github.com/grpc/grpc/pull/36567
[grpc/proposal#431]: https://github.com/grpc/proposal/pull/431
[gRFC A78]: A78-grpc-metrics-wrr-pf-xds.md

### Related Proposals
* [gRFC A66][]: OpenTelemetry Metrics
* [gRFC A78][]: OpenTelemetry Metrics for WRR, Pick First, and XdsClient
* [gRFC A96][]: OpenTelemetry Metrics for Retries

## Proposal

gRPC Java will add an API for applications to provide a filter function that
determines whether a target should be recorded as-is or mapped to `"other"` for
the `grpc.target` attribute in Java OpenTelemetry client metrics. When no filter
is provided (default), all targets use their original target string, preserving
existing behavior.

The string `"other"` is chosen as a stable, low-cardinality placeholder value
to represent all filtered targets. This value is intentionally fixed to ensure
consistent aggregation behavior across SDKs and deployments. The placeholder
value is not configurable to avoid further cardinality growth.

This proposal is limited to Java client-side OpenTelemetry metrics emitted by
gRPC Java's OpenTelemetry module that include the `grpc.target` attribute,
including the relevant metrics defined in [gRFC A66][] and [gRFC A96][]. It
does not change channel behavior, target resolution, OpenTelemetry View
behavior, or the deprecated C++ target filter API.

[gRFC A96]: A96-retry-otel-stats.md

### Java

gRPC Java adds a new method `targetAttributeFilter` to
`GrpcOpenTelemetry.Builder` that accepts a `Predicate<String>`. To ensure
compatibility with Android API levels < 24, where `Predicate` is not available
without desugaring, the filter is converted and stored internally using a
package-private interface.
The filter defaults to `null` when unset, meaning all targets are recorded as-is.

```java
@ExperimentalApi("https://github.com/grpc/grpc-java/issues/12595")
@IgnoreJRERequirement
public Builder targetAttributeFilter(@Nullable Predicate<String> filter)
```

When a filter is provided, `filter.test(target)` is called when the client interceptor is created
for a channel. If the predicate returns `true`, the original target string is used as
`grpc.target`. If it returns `false`, the string `"other"` is used instead.

The filter is applied when the client interceptor is created, meaning the
filtered target value is determined once per channel and reused for all calls
on that channel.

## Rationale

The `targetAttributeFilter` controls how the `grpc.target` attribute is recorded
in gRPC Java OpenTelemetry client metrics. Targets accepted by the filter are
recorded as-is; rejected targets are replaced with `"other"` to limit metric
cardinality.

This addresses the concrete Java cardinality problem reported in
[grpc/grpc-java#12322][]. This proposal only allows applications to keep the
original target or replace it with `"other"` for Java OpenTelemetry client
metrics; it is not a general API for rewriting target names across gRPC.

**Alternative approach considered:**

* Configuring a View with `setAttributeFilter()` to discard `grpc.target`.
    * This is an all-or-nothing approach and only serves as a temporary workaround.

**Reason for selection:**

* Simple and straightforward to implement.
* Immediately addresses high-cardinality metrics.
* Preserves existing behavior by default.
* Allows selective target retention, unlike an OpenTelemetry View that removes
  `grpc.target` entirely.

## Implementation

### Java

The Java implementation was merged in [grpc/grpc-java#12587][]. It adds
`targetAttributeFilter` to `GrpcOpenTelemetry.Builder` and uses an internal
interface for storage.

```java
interface TargetFilter {
  boolean test(String target);
}

public Builder targetAttributeFilter(@Nullable Predicate<String> filter) {
  if (filter == null) {
    this.targetFilter = null;
  } else {
    this.targetFilter = filter::test;
  }
  return this;
}
```

The filter is passed to `OpenTelemetryMetricsModule` and applied when recording
the target:

```java
String recordTarget(String target) {
  if (targetAttributeFilter == null || target == null) {
    return target;
  }
  return targetAttributeFilter.test(target) ? target : "other";
}
```

The filtered target is determined when the client interceptor is created and is
used for all metrics that include the `grpc.target` attribute for that channel.
