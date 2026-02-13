# A114: WRR Support for Custom Backend Metrics

- Author(s): sauravzg
- Approver: markdroth
- Status: In review
- Implemented in:
- Last updated: 2026-01-30
- Discussion at:

## Abstract

This proposal updates the client-side `weighted_round_robin` (WRR) load balancing policy to support customizable utilization metrics. It adds a new configuration field `metric_names_for_computing_utilization` to the WRR LB policy config. This allows users to specify which backend metrics should be used to compute endpoint weights, enabling the use of custom metrics (via ORCA named metrics) instead of relying solely on the default `application_utilization` or `cpu_utilization`.

## Background

The existing `weighted_round_robin` policy (defined in [gRFC A58][A58]) calculates endpoint weights based on standard metrics provided by the backend via ORCA (Open Request Cost Aggregation) load reports. Specifically, it uses `application_utilization` if available, and falls back to `cpu_utilization`.

However, services may want to drive load balancing decisions based on other resources, such as memory utilization, queue depth, or custom application-defined metrics. The [xDS Custom Backend Metrics][A51] specification (ORCA) supports reporting arbitrary named metrics, and xDS has updated its WRR implementation to allow selecting these metrics for utilization calculation.

To support advanced load balancing scenarios, gRPC's WRR policy needs to support this flexibility.

### Related Proposals

- [A58: Client-Side Weighted Round Robin LB Policy][A58]
- [A51: Custom Backend Metrics][A51]

## Proposal

### Service Config Update

We will add a new field `metric_names_for_computing_utilization` to the [`WeightedRoundRobinLbConfig`][WeightedRoundRobinLbConfigProto] message in the Service Config.

```protobuf
message WeightedRoundRobinLbConfig {
  // ... existing fields ...

  // A list of metric names to use for computing utilization.
  //
  // By default, endpoint weight is computed based on the 'application_utilization'
  // field reported by the endpoint.
  //
  // If 'application_utilization' is not set, then utilization will instead be
  // computed by taking the max of the values of the metrics specified here.
  //
  // For map fields in the ORCA proto, the string will be of the form
  // "<map_field_name>.<map_key>". For example, the string "named_metrics.foo"
  // will mean to look for the key "foo" in the ORCA "named_metrics" field.
  //
  // If none of the specified metrics are present in the load report, then
  // 'cpu_utilization' is used instead.
  repeated string metric_names_for_computing_utilization = 7;
}
```

### Weight Calculation Logic

The weight calculation logic in the WRR policy will be updated to determine the `utilization` value as follows from the [`OrcaLoadReport`][OrcaLoadReportProto]

1.  **Check `application_utilization`**: If the backend reports `application_utilization` (value > 0), use it.
2.  **Check Custom Metrics**: If `application_utilization` is not reported (or is 0), and `metric_names_for_computing_utilization` is configured:
    - Iterate through the specified metric names.
    - **Resolve Metric Value**:
      - If the name is of the format `field.key` (e.g., `named_metrics.foo`), look up the map field `field` and retrieve the value for `key`.
      - If the name is a simple field name (e.g., `cpu_utilization`, `mem_utilization`), look up the `field`.
    - **Compute Max**: Track the maximum value among all successfully resolved, positive ( > 0), finite metrics.
    - If a max value is found, use it as the `utilization`.
3.  **Fallback to `cpu_utilization`**: If neither of the above yields a value, use `cpu_utilization` (utilizing the existing logic).

#### Pseudocode

```
function GetUtilization(report, configured_metrics):
  # 1. Prefer application_utilization if present
  if report.application_utilization > 0:
    return report.application_utilization

  # 2. Check Custom Metrics
  max_util = null

  for metric_name in configured_metrics:
    value = null

    if metric_name contains ".":
      # Map lookup (e.g. "named_metrics.foo" -> map="named_metrics", key="foo")
      map_name, key = split_on_first_dot(metric_name)
      if report has map field map_name:
         value = report[map_name][key]
    else:
      # Root field lookup (e.g. "mem_utilization") via Reflection
      if report has field metric_name:
         value = report[metric_name]

    # Only consider valid, non-nan, positive values
    if value is not null and !is_nan(value) and value > 0:
      if max_util is null or value > max_util:
        max_util = value

  if max_util is not null:
    return max_util

  # 3. Fallback
  return report.cpu_utilization
```

#### Implementation Notes

Since `OrcaLoadReport` is often exposed as a language-specific proxy object rather than a raw Protobuf message (e.g., in Java and C++), implementations are **not** expected to use Protobuf reflection to look up arbitrary fields. Instead, implementations should manually handle:

- **Map Fields**: Lookups in the `named_metrics` map (e.g., `named_metrics.foo`). The string is split on the first dot: `foo.bar.baz` looks up key `bar.baz` in map `foo`.
- **Standard Fields**: Explicit lookups for known fields (e.g., `cpu_utilization`).

Support for any new standard fields added to `OrcaLoadReport` in the future will require explicit code changes in the implementation. This behavior is consistent with the [current Envoy implementation](https://github.com/envoyproxy/envoy/blob/35749578db375f5fe8ac5dd293cb7c4efb689611/source/common/orca/orca_load_metrics.cc#L47-L73).

#### Validity and Edge Cases

- **Nan Values**: As shown above, `NaN` values in reports are explicitly ignored to prevent undefined behavior in weight calculations. This is relevant because behavior of `max` on `NaN` in c++ is inconsistent based on order.
- **Zero and Negative Values**: Any value <= 0.0 is treated as missing and ignored. This is consistent with [gRFC A58][A58] and the [Envoy implementation](https://github.com/envoyproxy/envoy/blob/113f2c2d1015913256a2a8c6f9f97d0622623f45/source/extensions/load_balancing_policies/client_side_weighted_round_robin/client_side_weighted_round_robin_lb.cc#L214-L218).
- **Bound Checks**: The final selected `utilization` is subject to the standard validation logic from [gRFC A58][A58] (e.g., ensuring the value is positive) before being used to compute weight to avoid undefined behavior in weight calculations.
- **Normalization**: The WRR policy does not normalize reported metrics; the application is responsible for this.

The rest of the weight calculation formula (using QPS, EPS, and penalty) from [gRFC A58][A58] remains unchanged.

### xDS Integration

We will support the `metric_names_for_computing_utilization` field in the xDS [`ClientSideWeightedRoundRobin`][ClientSideWeightedRoundRobinProto] policy.

When converting the xDS configuration to the gRPC Service Config `WeightedRoundRobinLbConfig`, the `metric_names_for_computing_utilization` field should be copied over directly.

### Temporary environment variable protection

The features described in this proposal will be guarded by the environment variable `GRPC_EXPERIMENTAL_WRR_CUSTOM_METRICS`, which defaults to `false`.

## Rationale

- **Consistency with Envoy**: This design mirrors the corresponding feature in Envoy, ensuring consistent behavior for xDS-controlled clients.
- **Flexibility**: Allows users to define load balancing weights based on the actual bottleneck resource of their application (e.g., memory-bound services).
- **Backward Compatibility**: The default behavior (using `application_utilization` or `cpu_utilization`) remains unchanged if the new field is not configured.

## Implementation

This will be implemented in all languages C++, Java, and Go.

### C++

- **xDS Integration**: Update [`ClientSideWeightedRoundRobinLbPolicyConfigFactory`](https://github.com/grpc/grpc/blob/f7f13023412c1a589af7558eb0b9f8f664a76431/src/core/xds/grpc/xds_lb_policy_registry.cc#L68) to copy over the new field from the xDS configuration.
- **Config**: Update [`WeightedRoundRobinLbConfig`](https://github.com/grpc/grpc/blob/f7f13023412c1a589af7558eb0b9f8f664a76431/src/core/load_balancing/weighted_round_robin/weighted_round_robin.cc#L133) to include `metric_names_for_computing_utilization`.
- **Weight Calculation Logic**: Update callers of [`EndpointWeight::MaybeUpdateWeight`](https://github.com/grpc/grpc/blob/f7f13023412c1a589af7558eb0b9f8f664a76431/src/core/load_balancing/weighted_round_robin/weighted_round_robin.cc#L212) to implement the new utilisation selection logic.

### Java

- **xDS Integration**: Update [`convertWeightedRoundRobinConfig`](https://github.com/grpc/grpc-java/blob/a9f73f4c0aa5617aa2b6ae6ac805693915899b6a/xds/src/main/java/io/grpc/xds/LoadBalancerConfigFactory.java#L286) to copy over the field from the xDS configuration.
- **Config**: Update [`WeightedRoundRobinLoadBalancerConfig`](https://github.com/grpc/grpc-java/blob/a9f73f4c0aa5617aa2b6ae6ac805693915899b6a/xds/src/main/java/io/grpc/xds/WeightedRoundRobinLoadBalancer.java#L716) to include `metric_names_for_computing_utilization`.
- **Weight Calculation Logic**: Update [`OrcaPerRequestUtil`](https://github.com/grpc/grpc-java/blob/a9f73f4c0aa5617aa2b6ae6ac805693915899b6a/xds/src/main/java/io/grpc/xds/WeightedRoundRobinLoadBalancer.java#L364) to implement the new utilisation selection logic.

### Go

- **xDS Integration**: Update the configuration struct and conversion logic in [`converter.go`](https://github.com/grpc/grpc-go/blob/c05cfb3693bf18086810e671a6f9c05f296e0183/internal/xds/xdsclient/xdslbregistry/converter/converter.go#L241) to copy over the new field from the xDS configuration.
- **Config**: Update the configuration struct in [`config.go`](https://github.com/grpc/grpc-go/blob/7985bb44d26ecbeb8950996d028e38a0de08070b/balancer/weightedroundrobin/config.go#L26) to include `metric_names_for_computing_utilization`.
- **Weight Calculation Logic**: Update the weight update function in [`balancer.go`](https://github.com/grpc/grpc-go/blob/7985bb44d26ecbeb8950996d028e38a0de08070b/balancer/weightedroundrobin/balancer.go#L546) to implement the new utilisation selection logic.

[A58]: A58-client-side-weighted-round-robin-lb-policy.md
[A51]: A51-custom-backend-metrics.md
[ClientSideWeightedRoundRobinProto]: https://github.com/envoyproxy/envoy/blob/7242d5ad170523d7936849e596d261e3502c3886/api/envoy/extensions/load_balancing_policies/client_side_weighted_round_robin/v3/client_side_weighted_round_robin.proto#L86
[WeightedRoundRobinLbConfigProto]: https://github.com/grpc/grpc-proto/blob/ec99424f3b7dae9db194f848b4cea52ecfae07af/grpc/service_config/service_config.proto#L205
[OrcaLoadReportProto]: https://github.com/cncf/xds/blob/0feb69152e9f7e8a45c8a3cfe8c7dd93bca3512f/xds/data/orca/v3/orca_load_report.proto#L15
