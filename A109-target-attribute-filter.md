A109: Target Attribute Filter for OpenTelemetry Metrics
----
* Author(s): [becomeStar](https://github.com/becomeStar)
* Approver: a11r
* Status: Draft
* Implemented in: C++, Java (Java implementation will follow)
* Last updated: 2025-12-22
* Discussion at: (to be filled after discussion thread is created)

## Abstract

Add an optional filter to control how the `grpc.target` attribute is recorded in
OpenTelemetry metrics, allowing rejected targets to be mapped to `"other"` to
reduce metric cardinality, while preserving existing behavior by default.

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

[gRFC A66]: A66-otel-stats.md

### Related Proposals
* [gRFC A66][]: OpenTelemetry Metrics

## Proposal

gRPC will add an API for applications to provide a filter function that
determines whether a target should be recorded as-is or mapped to `"other"` for
the `grpc.target` attribute in OpenTelemetry metrics. When no filter is
provided (default), all targets use their original target string, preserving
existing behavior.

The string `"other"` is chosen as a stable, low-cardinality placeholder value
to represent all filtered targets. This value is intentionally fixed to ensure
consistent aggregation behavior across SDKs and deployments. The placeholder
value is not configurable to avoid further cardinality growth.

The filtering applies to all client-side per-call instruments that include the
`grpc.target` attribute, including those defined in [gRFC A66][] and [gRFC
A96][].

[gRFC A96]: A96-retry-otel-stats.md

### C++

gRPC C++ will add a method `SetTargetAttributeFilter` to
`OpenTelemetryPluginBuilder` that accepts an
`absl::AnyInvocable<bool(absl::string_view)>`.

```cpp
OpenTelemetryPluginBuilder& SetTargetAttributeFilter(
    absl::AnyInvocable<bool(absl::string_view /*target*/) const>
        target_attribute_filter);
```

The filter is stored in the plugin state and applied when creating
`OpenTelemetryClientFilter`. If no filter is registered or if the filter returns
`true`, the original target string is used. Otherwise, `"other"` is used.


### Java

gRPC Java will add a new method `targetAttributeFilter` to
`GrpcOpenTelemetry.Builder` that accepts a `Predicate<String>`. The filter
defaults to `null` when unset, meaning all targets are recorded as-is.

```java
public Builder targetAttributeFilter(@Nullable Predicate<String> filter)
````

When a filter is provided, `filter.test(target)` is called when the client interceptor is created
for a channel. If  the predicate returns `true`, the original target string is used as
`grpc.target`. If it returns `false`, the string `"other"` is used instead.

The filter is applied when the client interceptor is created, meaning the
filtered target value is determined once per channel and reused for all calls
on that channel.

## Rationale

The `targetAttributeFilter` controls how the `grpc.target` attribute is recorded in OpenTelemetry metrics. Targets accepted by the filter are recorded as-is; rejected targets are replaced with `"other"` to limit metric cardinality.

This approach is already implemented in gRPC C++, and bringing it to gRPC Java ensures consistent metric semantics across languages, which is important for multi-language deployments.

**Alternative approach considered:**

* Configuring a View with `setAttributeFilter()` to discard `grpc.target`.
    * This is an all-or-nothing approach and only serves as a temporary workaround.

**Reason for selection:**

* Simple and straightforward to implement.
* Immediately addresses high-cardinality metrics.
* Aligns Java behavior with existing C++ implementation.

## Implementation

### C++

The implementation adds `SetTargetAttributeFilter` to
`OpenTelemetryPluginBuilder`:

```cpp
OpenTelemetryPluginBuilder&
OpenTelemetryPluginBuilder::SetTargetAttributeFilter(
    absl::AnyInvocable<bool(absl::string_view /*target*/) const>
        target_attribute_filter) {
  target_attribute_filter_ = std::move(target_attribute_filter);
  return *this;
}
```

The filter is stored in the plugin state and applied when creating
`OpenTelemetryClientFilter`:

```cpp
absl::StatusOr<OpenTelemetryClientFilter> OpenTelemetryClientFilter::Create(
    const grpc_core::ChannelArgs& args, ChannelFilter::Args /*filter_args*/) {
  std::string target =
      args.GetOwnedString(GRPC_ARG_SERVER_URI).value_or("");

  if (OTelPluginState().target_attribute_filter == nullptr ||
      OTelPluginState().target_attribute_filter(target)) {
    return OpenTelemetryClientFilter(std::move(target));
  }
  return OpenTelemetryClientFilter("other");
}
```

### Java

The implementation adds `targetAttributeFilter` to
`GrpcOpenTelemetry.Builder`:

```java
public Builder targetAttributeFilter(@Nullable Predicate<String> filter) {
  this.targetAttributeFilter = filter;
  return this;
}
```

The filter is passed to `OpenTelemetryMetricsModule` and applied when recording
the target:

```java
String recordTarget(String target) {
  if (targetAttributeFilter == null) {
    return target;
  }
  return targetAttributeFilter.test(target) ? target : "other";
}
```

The filtered target is determined when the client interceptor is created and is
used for all metrics that include the `grpc.target` attribute for that channel.
