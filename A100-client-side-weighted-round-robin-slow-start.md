A100: Client-side weighted round robin slow start configuration
----
* Author(s): [Anurag Agarwal](https://github.com/anuragagarwal561994)
* Approver: a11r
* Status: Draft
* Last updated: 2025-05-31
* Discussion at: TBD

## Abstract

This proposal introduces an enhancement to the existing client-side weighted_round_robin (WRR) load balancing policy in gRPC by incorporating a configurable `slow_start_config` mechanism. The intent of this feature is to gradually increase traffic to backend endpoints that are newly introduced or have recently rejoined the cluster, allowing them time to warm up and reach their optimal performance level before handling their full share of traffic. This change increases system stability and resilience in environments with dynamic scaling and volatile workloads.

The design borrows from production-ready practices in other load balancers such as Envoy, where gradual traffic ramp-up (slow start) is a [well-established technique][Envoy Slow Start Documentation] for avoiding performance degradation and request failures during backend startup or recovery. The slow start feature gradually increases the traffic sent to newly added endpoints during a warmup period, allowing them to warm up their caches and establish connections before receiving full traffic load.

## Background

gRPC's WRR load balancing policy allows clients to route requests to backend endpoints in proportion to assigned weights. These weights are usually derived from backend metrics, such as CPU usage, QPS, and error rates published by the backend servers. This allows gRPC clients to adapt traffic distribution dynamically based on backend capacity.

However, the current WRR implementation lacks support for any ramp-up or warm-up phase. When a new endpoint appears—either due to autoscaling, replacement, or recovery from a failure—the client immediately starts routing requests based on the full weight reported for that endpoint. This can result in overloading the endpoint before it is fully initialized, negatively impacting response times and service reliability. It is especially problematic for systems with cold caches, JIT compilation warm-up delays, or dependency initialization steps.

In contrast, many modern systems adopt slow start strategies in load balancing to address these issues. These strategies allow endpoints to ramp up traffic gradually over a defined window, smoothing transitions and mitigating the risks of traffic spikes. Similar functionality exists in Envoy's load balancing policies, where slow start is implemented for round robin and least request policies.

Introducing a `slow_start_config` configuration in gRPC WRR will offer these benefits within the native client policy, reducing reliance on external traffic-shaping mechanisms or manual intervention.

### Related Proposals:
* [gRFC A58: weighted_round_robin LB policy][A58]

## Proposal

Add slow start configuration to the `weighted_round_robin` load balancing policy. The slow start feature will scale the computed weights for endpoints during their warmup period, gradually increasing the traffic they receive.

### LB Policy Config and Parameters

The `weighted_round_robin` LB policy config will be extended to include slow start configuration:

```textproto
message LoadBalancingConfig {
  oneof policy {
    ClientSideWeightedRoundRobin weighted_round_robin = 20 [json_name = "weighted_round_robin"];
  }
}

message ClientSideWeightedRoundRobin {
  // ... existing fields ...

  // Configuration for slow start feature
  SlowStartConfig slow_start_config = 8;
}

message SlowStartConfig {
  // Represents the size of slow start window.
  // If set, the newly created host remains in slow start mode starting from its creation time
  // for the duration of slow start window.
  google.protobuf.Duration slow_start_window = 1;

  // This parameter controls the speed of traffic increase over the slow start window. Defaults to 1.0,
  // so that endpoint would get linearly increasing amount of traffic.
  // When increasing the value for this parameter, the speed of traffic ramp-up increases non-linearly.
  // The value of aggression parameter should be greater than 0.0.
  // By tuning the parameter, it is possible to achieve polynomial or exponential shape of ramp-up curve.
  //
  // During slow start window, effective weight of an endpoint would be scaled with time factor and aggression:
  // ``new_weight = weight * max(min_weight_percent, time_factor ^ (1 / aggression))``,
  // where ``time_factor=(time_since_start_seconds / slow_start_time_seconds)``.
  //
  // As time progresses, more and more traffic would be sent to endpoint, which is in slow start window.
  // Once host exits slow start, time_factor and aggression no longer affect its weight.
  google.protobuf.FloatValue aggression = 2;

  // Configures the minimum percentage of origin weight that avoids too small new weight,
  // which may cause endpoints in slow start mode receive no traffic in slow start window.
  // If not specified, the default is 10%.
  google.protobuf.UInt32Value min_weight_percent = 3;
}
```

### Weight Scaling During Warmup

When an endpoint is ready after being in a non-ready state, it enters the warmup period. During this period, its weight will be scaled by a factor that increases non-linearly from `min_weight_percent` to 100% over the duration of `slow_start_window`.

The scale factor is calculated as follows:

```
time_factor = max(time_since_start_in_seconds, 1) / slow_start_window_seconds
scale = max(min_weight_percent, time_factor ^ (1/aggression))
```

The following image shows how different aggression values affect the scaling factor over time during the slow start window (in milliseconds):

![Effect of aggression parameter on slow start scaling](A123_graphics/aggression_scaling.png)

The graph illustrates how the scaling factor (effective weight) ramps up from zero to the backend weight for various aggression values, with time shown in milliseconds.

The final weight used for load balancing will be:
```
final_weight = computed_weight * scale
```

### Subchannel Weights

The existing `non_empty_since` timestamp, which is already used to check for `blackout_period`, can be repurposed to track the start of the warmup period for each endpoint. This timestamp tracks when the first non-zero load report was received, marking the beginning of its warmup period. This approach eliminates the need for additional timestamp tracking while maintaining the existing functionality.

Weight calculation in the WRR policy follows a two-step process. First, the base weight for each endpoint is computed using the formula from [gRFC A58][A58]:

$$weight = \dfrac{qps}{utilization + \dfrac{eps}{qps} * error\\_utilization\\_penalty}$$

This base weight represents the endpoint's capacity based on its current performance metrics. During the warmup period, this weight is then scaled using the slow start formula:

$$scaled\\_weight = weight * max(min\\_weight\\_percent, time\\_factor ^ {1/aggression})$$

where,

$$time\\_factor = \dfrac{max(time\\_since\\_non\\_empty\\_seconds, 1)}{slow\\_start\\_window\\_seconds}$$

The scaling formula ensures that new endpoints receive a gradually increasing share of traffic while maintaining a minimum threshold to prevent starvation. The `aggression` parameter allows fine-tuning of the ramp-up curve, enabling either more aggressive initial scaling (values > 1.0) or more conservative approaches (values < 1.0).

When an endpoint is not in the warmup period, the scale factor is set to 1.0, meaning the original weight is used without modification. This ensures that the slow start mechanism only affects endpoints during their initial warmup phase, after which they participate in normal load balancing based on their actual performance metrics.

### Blackout Period vs Slow Start

The WRR load balancing policy offers two independent mechanisms for handling new endpoints: the blackout period and slow start. These mechanisms can be used independently or in combination, allowing operators to choose the approach that best fits their needs.

The blackout period, which defaults to 10 seconds, begins when an endpoint receives its first non-zero load report. During this period, the endpoint continues to receive traffic, but instead of using the weights reported by the backend servers, the load balancer uses the mean of all backend-reported weights. This period helps prevent churn in the load balancing decisions when the set of endpoint addresses changes, ensuring that the weights used are based on stable, continuous load reporting.

The slow start period also begins when the endpoint receives its first non-zero load report and applies a gradual scaling factor to the weights over a configurable duration (default 30 seconds). This scaling is applied to whatever weight is being used (either the mean weight during blackout period or the actual backend-reported weight after blackout period). The slow start period operates independently of the blackout period, meaning it will continue to scale the weights regardless of whether the blackout period is still active or has ended.

It is recommended to keep the blackout period shorter than the slow start period. This is because when the blackout period ends, the endpoint's weight will suddenly change from the mean weight to its actual backend-reported weight. If this weight is significantly higher than the mean (e.g., 2x the mean weight), it could cause a sudden traffic spike that defeats the purpose of gradual traffic increase. By having a longer slow start period, the scaling factor will continue to gradually increase the weight even after the blackout period ends, ensuring a smooth transition to the full backend-reported weight.

When endpoint weights become stale after the `weight_expiration_period`, the load balancer will continue to use the mean weight for load balancing. This is different from the blackout period as it's a response to weight staleness rather than initial endpoint setup. In these cases, since the endpoint weights were previously active, the slow start period is typically not triggered. However, when new weights arrive after expiration, the endpoint will enter the blackout period again, and if slow start is configured, the weights will be scaled up gradually during this period.

These mechanisms can be configured in different ways:
- Using only blackout period: Ensures stable weight reporting by using mean weights before switching to backend weights
- Using only slow start: Allows immediate use of backend weights but scales them gradually
- Using both: Provides both stable weight reporting and gradual scaling
  - The slow start period will scale the mean weights during blackout period
  - After blackout period ends, it will continue to scale the actual backend-reported weights

This flexible design allows operators to tune the behavior based on their specific needs, whether they want to prioritize stable weight reporting, faster weight adoption, or gradual traffic ramp-up.

### Example Implementation

```pseudo
// Configuration parameters
slow_start_window: Duration
min_weight_percent: float
aggression: float

// State
non_empty_since: Time  // Time when first non-zero load report was received

function get_scale() -> float:
    time_elapsed_since_start = current_time - non_empty_since
    if time_elapsed_since_start >= slow_start_window:
        return 1.0
        
    time_factor = max(time_elapsed_since_start, 1.0) / slow_start_window
    min_scale = min_weight_percent / 100.0
    
    // Apply non-linear scaling based on aggression factor
    scale = max(min_scale, time_factor ^ (1.0 / aggression))
    return scale

function get_final_weight(endpoint_weight: float, scaling_factor: float) -> float:
    // endpoint_weight is either:
    // - mean_weight for stale endpoints or endpoints in blackout period
    // - actual backend-reported weight after blackout period
    return endpoint_weight * scaling_factor
```

### xDS Integration

The slow start configuration will be added to the xDS proto for the weighted round robin policy:

```textproto
package envoy.extensions.load_balancing_policies.client_side_weighted_round_robin.v3;

message ClientSideWeightedRoundRobin {
  // ... existing fields ...
  cluster.v3.Cluster.SlowStartConfig slow_start_config = 8;
}

message SlowStartConfig {
  google.protobuf.Duration slow_start_window = 1;
  core.v3.RuntimeDouble aggression = 2;
  type.v3.Percent min_weight_percent = 3;
}
```

## Rationale

The slow start feature helps prevent overwhelming new endpoints by gradually increasing their traffic load. This is particularly important in systems where endpoints need time to:
- Warm up their caches
- Establish connections to downstream services
- Initialize internal state
- Build up connection pools

By scaling the weights during the warmup period, we allow new endpoints to handle a smaller portion of traffic initially, reducing the risk of errors and high latency during the critical warmup phase.

### Effectiveness Considerations

The slow start feature is most effective in scenarios where:
- Few new endpoints are added at a time (e.g., scale events in Kubernetes)
- Endpoints need time to warm up caches or establish connections
- The system has sufficient traffic to gradually increase load

The feature may be less effective when:
- All endpoints are relatively new (e.g., new deployment in Kubernetes)
- The system has low traffic volume
- There are many endpoints in the cluster

In these cases, the slow start feature may lead to:
- Endpoint starvation (low probability of receiving requests)
- Non-gradual traffic increases when requests do arrive
- Reduced effectiveness of the load balancing algorithm

## Scope and Limitations

This proposal specifically focuses on implementing slow start for the weighted round robin load balancing policy. While similar slow start functionality could potentially be implemented for other load balancing algorithms like Round Robin and Least Request, these are not included in this proposal for the following reasons:

1. These algorithms don't use weights to determine endpoint selection, making the implementation of slow start more complex
2. Additional considerations would be needed for how to gradually increase traffic to new endpoints in these algorithms
3. The implementation details would likely differ significantly from the weighted round robin approach

These other load balancing algorithms can be considered for slow start implementation in future proposals, with their own specific design considerations and requirements.

## Metrics

The following metric will be exposed to help monitor the slow start behavior:

`grpc.lb.wrr.endpoints_in_slow_start`
- Type: Gauge
- Description: Number of endpoints currently in slow start period
- Labels:
  - `grpc.lb.locality`: The locality of the endpoints
  - `grpc.lb.backend_service`: The backend service name

This metric will help operators monitor the number of endpoints currently in slow start mode across different localities and backend services.

## Implementation

This will be implemented in all languages C++, Java, and Go.

[Envoy Slow Start Documentation]: https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/slow_start
[A58]: A58-client-side-weighted-round-robin-lb-policy.md