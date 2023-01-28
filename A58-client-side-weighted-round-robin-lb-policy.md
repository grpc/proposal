A58: `weighted_round_robin` LB policy
----
* Author(s): @yousukseung
* Approver: @markdroth
* Status: Draft
* Implemented in: -
* Last updated: 2023-01-24
* Discussion at: -

## Abstract

This document proposes a design for a new load balancing policy `weighted_round_robin`. This policy supports client-side weighted round robin load balancing. Unlike `grpclb` which follows the look-aside model, `weighted_round_robin` is client-side and requires direct load reporting from backends to clients. The policy supports either per-call or periodic out-of-band load reporting and depends on the [Open Request Cost Aggregation (ORCA)] support and [gRFC A51]. The client computes and tracks the weight of subchannels and performs weighted round-robin channel selection. The design also proposes to support `weighted_round_robin` as a custom load balancing policy in xDS as proposed in [gRFC A52].

## Background

Today in gRPC weighted endpoint load balancing is possible with resolver supplied weights combined with the round robin policy on the client. This requires the control plane to centrally collect load metrics and compute weights for load balancing based on the backend load metrics. Client-side weighted round-robin enables load balancing between clients and backends without involving a centralized load collection. The necessary support to transmit load metrics from backends to clients is available such as [Open Request Cost Aggregation (ORCA)] and [gRFC A51]. This design proposes a new top-level load balancing policy `weighted_round_robin` that implements client-side load balancing.


### Related Proposals:
* [gRFC A51]
* [gRFC A52]
* [Open Request Cost Aggregation (ORCA)]


## Proposal

Introduce a new LB policy `weighted_round_robin`. This policy implements client-side load balancing with direct load reports from backends. Either per-call or periodic out-of-band (OOB) load reporting is active at a time where per-call is the default. The policy is otherwise largely a superset of the existing policy `round_robin`.

### LB Policy Config and Parameters

Add the policy config with parameters as follows. This policy is enabled as the endpoint picking policy of `WrrLocality` in [gRFC A52], see [below](#xds-integration) for more.
```textproto
message LoadBalancingConfig {
   // …
   WeightedRoundRobin weighted_round_robin_experimental = 18 [json_name = "weighted_round_robin_experimental "];
}

message WeightedRoundRobin {
   google.protobuf.Duration blackout_period = 1;
   google.protobuf.Duration weight_expiration_period = 2;
   bool enable_oob_load_report = 3;
   google.protobuf.Duration oob_reporting_period = 4;
   google.protobuf.Duration weight_update_period = 5;
}

```

`blackout_period` is to handle backend churns. `weighted_round_robin` rejects load reports until a backend reports valid weights for at least `blackout_period` during which the average weight value is used instead. Disables blackout when set to a negative value. Defaults to 10 seconds.

Load metrics for a subchannel expire after `weight_expiration_period` is passed without any update from the backend, including with per-call or OOB load reporting. Once expired, the average weight value is used instead. Defaults to 3 minutes. Cannot be disabled but the period has no max value allowed.

The client initiates OOB load reporting by sending a request only when `enable_oob_load_report` is set. Defaults to false. When set, this policy does not parse per-call load reports. See below for more on load reporting. Note this does not affect the per-call load reports in xDS LRS.

`oob_reporting_period` is only used when OOB load reporting is used. Note the effective reporting period is determined as outlined in [gRFC A51] and may be different. The client requests the minimum of values from all interested LB policies, but the server enforces its own minimum reporting period. Defaults to 10 seconds.

The client updates effective weights periodically every `weight_update_period`. This parameter defaults to 1 second. The lowest value allowed is 100ms. When set to a value lower than 100ms the period is set to 100ms. Weight updates triggered by changes in the connectivity state or backend addresses are not affected.

### Earliest Deadline First (EDF) Scheduler

`weighted_round_robin` introduces a scheduler that manages weighted subchannel selection. A new scheduler instance is created every time weights are updated.  A state object is passed when created so that the scheduler can start where it last left across multiple instances. The scheduler must be light and cheap to create and allow concurrent pick calls. The scheduler returns the index in the input weight array and the caller has to map the index to a subchannel.

The scheduler resides in the picker. When compared to `round_robin`, the scheduler conceptually replaces the monotonically increasing index counter of the picker. The picker in `weighted_round_robin` otherwise works the same externally as in `round_robin`.

In C/C++, the scheduler is reference counted and its access is mutex protected. We may consider optimization in the future such as using RCU synchronization when available. Java/Go implementation accesses the scheduler with an atomic reference object.

The picker starts a timer to swap the scheduler to update update weights every `weight_update_period`. This timer iterates over the subchannel list (wrapper) of the picker, loads the latest weight from each, builds a new scheduler and replaces it with the current one. Note this timer does not run in the control plane, but weights are atomic and reading weights does not require extra synchronization with the control plane. When the scheduler is swapped, ongoing pick calls that already took a reference of the outgoing scheduler can continue to refer to the old scheduler until the call returns.

Note only subchannels in `READY` state are passed to the scheduler. Subchannels that are included with a zero weight are still picked but with the average weight. When less than two subchannels have load info, it degrades to `round_robin`.

`weighted_round_robin` uses the earliest deadline first (EDF) scheduling with a monotonically increasing counter as the state. This counter is persisted across multiple scheduler instances for continuity between weight updates.

```
EDFScheduler {
  // Creates a scheduler with given weight values.
  // `next_seq_fn` returns a monotonically increasing
  // number that wraps, must allow concurrent calls.
  function EDFScheduler New(float[] weights, lambda () -> uint32 next_seq_fn);
  // Returns the index of the subchannel selected.
  // Can be called concurrently.
  function Pick();
};
```

### Control Plane Behaviors

`weighted_round_robin` control plane works identically to `round_robin` except the handling load reports and weights and the scheduler. The current `round_robin` implementation in each language should serve as full reference. This section outlines the differences and key behaviors.

#### Aggregate Connectivity State
The rules to determine the aggregate connectivity state from subchannel states are the same as in `round_robin`: `READY` if any subchannel is `READY`, else `CONNECTING` if any is `CONNECTING`, else `TRANSIENT_FAILURE` if all are in `TRANSIENT_FAILURE`.

#### Handling Parent/Resolver Updates

When `enable_oob_load_report` value changes, either per-call or OOB load report handling is enabled and the other is disabled, depending on the current value. For the change to per-call reporting, since `weighted_round_robin` always creates a new picker when it gets an update, the policy will configure the new picker as to whether it should request per-call load report data. For enabling or disabling OOB reporting the exact mechanism depends on the language. In C/C++, a new pending subchannel list is created in both cases to enable or disable the OOB load report watcher. In Java/Go, the OOB listener is started/stopped on the existing subchannels. Note this requires API changes, see below for more. In Go, the OOB load report handling implementation is pending.

When `oob_reporting_period` changes, the client implementation starts a new OOB load reporting stream as outlined in [gRFC A51].

Changes to `weight_update_period`, `blackout_period`, and `weight_expiration_period` take effective with a new picker that we always create upon receiving a resolver update.

In all languages, `weighted_round_robin` policy de-duplicates redundant addresses in resolver updates. This behavior is different from `round_robin` where it uses the number of representations in the list as a resolver supplied weight.

#### Managing Subchannels

The behavior remains the same as in `round_robin` in all languages.

In C/C++, the control plane creates a new pending subchannel list upon getting a new address list and starts watching the connectivity state of its subchannels. Only subchannels in the `READY` state are included. The pending list is immediately promoted to the current list if it’s an initial update or the new list is empty (i.e. no subchannel). Otherwise the pending list is later promoted to the current list if the current list has none `READY`, or the pending list has at least one `READY`, or all subchannels in the pending list are `TRANSIENT_FAILURE`. The rest of the process is triggered upon the connectivity state of a subchannel changes in either the pending or current list. When the connectivity state of a subchannel changes and it is in the pending subchannel list, it checks whether to promote the pending list to current. If it’s in the current list it updates the aggregate connectivity state and creates a new picker. The kind of picker created depends on the aggregate state: an active Picker (w/ a scheduler) if `READY`, a QueuePicker if `CONNECTING`, or a TransientFailurePicker otherwise.

In Java, the current subchannel list is updated directly as in `round_robin`. Upon getting an updated address list, it adds new subchannels to the subchannel list, starts watching the connectivity state of it, and requests connection. It also deletes removed subchannels from the subchannel list and shuts them down. It then updates the aggregate connectivity state, and if there is at least one `READY` subchannel, creates a new picker. The kind of picker created depends on the aggregate state: an active Picker (with a scheduler) if `READY`, EmptyPicker otherwise. When the connectivity state of a subchannel changes, it triggers updating the aggregate connectivity state.

In Go, `baseBalancer` handles the address update as in `round_robin`. When the address set changes, the `baseBuilder` calls the picker builder of `weighted_round_robin` if the aggregate connectivity state is `READY`. When the connectivity state of a subchannel changes, it triggers the same process as when the address set changes.

In all languages, when a subchannel state is updated to `IDLE`, the control plane immediately requests a connection and treats it as if it is in `CONNECTING` for the aggregation purpose.

### Subchannel Weights

To enforce `blackout_period` and `weight_expiration_period`, it tracks two timestamps along with the weight. `last_updated` tracks the last time a non-zero report was received. `non_empty_since` tracks when the first non-zero load report was received, and it is reset when a subchannel comes back to `READY` or the weight expires so that it can be updated with the next non-zero load report. The blackout and expiration are enforced when the weight value is looked up. When the weight expires, `non_empty_since` is reset.

The weight of a backend is calculated as ***weight = qps / cpuutilization***.

```
function UpdateWeight(qps, cpu_utilization) {
  var new_eight =
     cpu_utilization == 0 ? 0 : qps / cpu_utilization;
  if (new_eight == 0) return;
  if (non_empty_since_ == infy) non_empty_since_ = now();
  last_updated_ = now();
  weight_ = new_weight;
}
function OnSubchannelBecomesReady() {non_empty_since_ = infy;}
function GetWeight() {
   if (now() - last_updated_ >= weight_expiration_period_) {
      non_empty_since_ = infy;
      return 0;
   }
   if (now() - non_empty_since_ < blackout_period_) return 0;
   return weight_;
}

```

### Per-Call Load Reporting

Necessary support for per-call load report handling in clients depends on [gRFC A51]. This section outlines how this features works with `weighted_round_robin`. xDS `cluster_impl` policy can serve as an example.

`weighted_round_robin` defaults to per-call load reporting. The client does nothing when the per-call load report is expected but missing in the trailing metadata of the response. While `enable_oob_load_report` is set, per-call load reports are never parsed on the client side. The client does nothing when the per-call load report is present in the response when not expected.

#### C/C++
Processing per-call load reports is supported with `SubchannelCallTrackerInterface`. `weighted_round_robin` implements this interface and attaches it to a subchannel `PickResult`. Later Finish() is called back with the load metrics and updates the weight.

#### Java
`OrcaPerRequestUtil.OrcaReportingTracerFactory` is available in `io.grpc.xds.orca` to process per-call backend load reports in `ClusterImplLoadBalancer`. This needs to be moved to a new package and shared with `weighted_round_robin`. Then the picker attaches this tracer to the subchannel selection and the listener takes the reference to the subchannel to which it updates the weight to in `onLoadReport()`.

#### Go
Per-call load report handling is fully implemented. Picker selection result `PickResult` has an optional callback `Done()` that is passed with the backend load metrics parsed from the response if available. With `weighted_round_robin`, the picker only sets Done() in `PickResult` when per-call load reporting is enabled, and the callback updates the weight.

### OOB Load Reporting

OOB load report handling in clients depends on [gRFC A51]. This section outlines how this feature works with `weighted_round_robin`.

In all languages, `report_interval` in `OrcaLoadReportRequest` is determined by the service config parameter `oob_reporting_period`. When it changes, a new stream request is sent. In Java/Go, the implementation needs to add support to enable and disable the OOB listener when `enable_oob_load_report` is updated. In C++ this happens automatically as a new subchannel list is created.

`weighted_round_robin` initiates OOB load reporting when the LB policy parameter `enable_oob_load_report` changes to true. When the backend returns `UNIMPLEMENTED` to the OOB load report request it proceeds with a log message as if it is not enabled and does not retry.

#### C/C++

Processing OOB load reports is supported with `OobBackendMetricWatcher`. `weighted_round_robin` implements this interface. The watcher takes the reference of the subchannel weight object and update weights to it.

#### Java

`OrcaOobUtil` is available in `io.grpc.xds.orca` but currently not in use. This needs to be moved to a new package and shared with `weighted_round_robin`, which implements `ForwardingLoadBalancerHelper` that takes the helper from `OrcaOobUtil`. The listener takes the reference to the subchannel to which it updates the weight to in `onLoadReport()`. A new API is needed to set and unset the listener in `OrcaOobUtil` to toggle the OOB load report processing.

#### Go

`orca.OOBListener` is available for OOB load reporting. `weighted_round_robin` extends it and attach it to the subchannel to which it updates the weight in `onLoadReport()`. A new API is needed to support enabling and disabling OOB load reporting.


### xDS Integration

`weighted_round_robin` is enabled as the endpoint picking policy of `WrrLocality` of [gRFC A52]. The implementation is available in C++ and Java but not in Go yet.

`weighted_round_robin` is added as a new LB policy per [gRFC A52] and is added to `XdsLbPolicyRegistry`. The name is prefixed with `ClientSide` to avoid confusion with the existing weighted round robin support via `ROUND_ROBIN`.

```textproto
package envoy.extensions.load_balancing_policies.client_side_weighted_round_robin.v3;

message ClientSideWeightedRoundRobin {
   google.protobuf.Duration blackout_period = 1;
   google.protobuf.Duration weight_expiration_period = 2;
   bool enable_oob_load_report = 2;
   google.protobuf.Duration oob_reporting_period = 3;
   google.protobuf.Duration weight_update_period = 4;
}
```

Example C++ code snippet that adds `weighted_round_robin` to `XdsLbPolicyRegistry`.
```c
//envoy.extensions.load_balancing_policies.client_side_weighted_round_robin.v3.ClientSideWeightedRoundRobin
class ClientSideWeightedRounbRobinLbPolicyConfigFactory
    : public XdsLbPolicyRegistry::ConfigFactory { … };

XdsLbPolicyRegistry::XdsLbPolicyRegistry() {
  // …
  policy_config_factories_.emplace(
      ClientSideWeightedRounbRobinLbPolicyConfigFactory::Type(),
      absl::make_unique<ClientSideWeightedRounbRobinLbPolicyConfigFactory>());
}
```

In the xDS configuration for the client, `client_side_weighted_round_robin` is wrapped in `wrr_locality` as a child policy. Below is an example xDS configuration with `client_side_weighted_round_robin`. The config includes `round_robin` as a backup policy to fallback in case the client does not support `client_side_weighted_round_robin`. On the client the new xDS LB hierarchy in [gRFC A52] creates the gRPC LB policy `weighted_round_robin` as the endpoint-picking policy.

```json
{ // Cluster
  name: "..."
  type: EDS
  load_balancing_policy: {
    policies: [
      typed_extension_config: {
        name: "..."
        typed_config: {
          "@type": "type.googleapis.com/envoy.extensions.load_balancing_policies.wrr_locality.v3.WrrLocality"
          endpoint_picking_policy: {
            policies: [
              typed_extension_config: {
                name: "..."
                typed_config: {
                  "@type": "type.googleapis.com/envoy.extensions.load_balancing_policies.client_side_weighted_round_robin.v3.ClientSideWeightedRoundRobin"
                  type_url: "..."
                  fields: [
                    {
                      key: "enable_oob_load_report"
                      value: false
                    },
                    // …
                  ]
                }
              }
              // Backup RR policy to fall back if WRR is unsupported.
              typed_extension_config: {
                name: "..."
                typed_config: {
                  "@type": "type.googleapis.com/envoy.extensions.load_balancing_policies.round_robin.v3.RoundRobin"
                }
              }
    // …
}
```

## Implementation

This will be implemented in all languages C++, Java, and Go.

[Open Request Cost Aggregation (ORCA)]: https://github.com/envoyproxy/envoy/issues/6614
