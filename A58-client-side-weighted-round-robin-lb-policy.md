A58: `weighted_round_robin` LB policy
----
* Author(s): @yousukseung
* Approver: @markdroth
* Status: Draft
* Implemented in: -
* Last updated: 2023-01-24
* Discussion at: -

## Abstract

This document proposes a design for a new load balancing policy `weighted_round_robin`. This policy supports client-side weighted round robin load balancing. Unlike `grpclb` which follows the look-aside model, `weighted_round_robin` is client-side and requires direct load reporting from backends to clients. The policy supports either per-call or periodic out-of-band load reporting per [gRFC A51][A51]. The client computes and tracks the weight of subchannels and performs weighted round-robin channel selection. The design also proposes to support `weighted_round_robin` as a custom load balancing policy in xDS as proposed in [gRFC A52][A52].

## Background

Currently, gRPC does not have any first-class support for weighted round robin load balancing. It can be done in a somewhat cumbersome way with the existing `round_robin` LB policy, by including a given address multiple times in the address list. This mechanism can be used with the legacy grpclb protocol, where the control plane can centrally collect backend metrics from the endpoints and compute weights for load balancing based on those metrics. However, this approach requires a lot of machinery.

As part of the effort to move away from `grpclb`, we will now provide first-class weighted round robin functionality via a new client-side LB policy called `weighted_round_robin`. Instead of depending on centralized backend metric collection, this LB policy will obtain the backend metrics directly from the endpoints using the mechanism described in [gRFC A51][A51], which it will use to compute the endpoint weights.

### Related Proposals:
* [gRFC A34 (pending)][A34]
* [gRFC A51][A51]
* [gRFC A52][A52]


## Proposal

Introduce a new LB policy `weighted_round_robin`. This policy implements client-side load balancing with direct load reports from backends. Either per-call or periodic out-of-band (OOB) load reporting is active at a time where per-call is the default. The policy is otherwise largely a superset of the existing policy `round_robin`.

### LB Policy Config and Parameters

The `weighted_round_robin` LB policy config will be as follows.

```textproto
message LoadBalancingConfig {
  oneof policy {
    WeightedRoundRobinLbConfig weighted_round_robin = 20 [json_name = "weighted_round_robin"];
  }
}

message WeightedRoundRobinLbConfig {
  // Whether to enable out-of-band utilization reporting collection from
  // the endpoints.  By default, per-request utilization reporting is used.
  google.protobuf.BoolValue enable_oob_load_report = 1;

  // Load reporting interval to request from the server.  Note that the
  // server may not provide reports as frequently as the client requests.
  // Used only when enable_oob_load_report is true.  Default is 10 seconds.
  google.protobuf.Duration oob_reporting_period = 2;

  // A given endpoint must report load metrics continuously for at least
  // this long before the endpoint weight will be used.  This avoids
  // churn when the set of endpoint addresses changes.  Takes effect
  // both immediately after we establish a connection to an endpoint and
  // after weight_expiration_period has caused us to stop using the most
  // recent load metrics.  Default is 10 seconds.
  google.protobuf.Duration blackout_period = 3;

  // If a given endpoint has not reported load metrics in this long,
  // then we stop using the reported weight.  This ensures that we do
  // not continue to use very stale weights.  Once we stop using a stale
  // value, if we later start seeing fresh reports again, the
  // blackout_period applies.  Defaults to 3 minutes.
  google.protobuf.Duration weight_expiration_period = 4;

  // How often endpoint weights are recalculated.  Default is 1 second.
  google.protobuf.Duration weight_update_period = 5;
}
```

### Earliest Deadline First (EDF) Scheduler

`weighted_round_robin` introduces a scheduler that manages weighted subchannel selection. A new scheduler will be created periodically to take new weight information into account. A state object is passed when created so that the scheduler can start where it last left across multiple instances. The scheduler must be light and cheap to create and allow concurrent pick calls. The scheduler returns the index in the input weight array and the caller has to map the index to a subchannel. The scheduler's state is stored in the LB policy and not the picker, and is atomically updated at the end of each pick by the scheduler.

The scheduler implements the [earliest deadline first][EDF] (EDF) scheduling. In its implementation, each subchannel is treated as a job whose deadline is inversely proportional to its weight. To avoid synchronization, each subchannel is given a random credit in the range [0, deadline] when first inserted. The subchannel with the next earliest deadline is selected each time and then inserted back with its deadline.

The scheduler resides in the picker. When compared to `round_robin`, the scheduler conceptually replaces the monotonically increasing index counter of the picker. The picker in `weighted_round_robin` otherwise works the same externally as in `round_robin`.

In C/C++, the scheduler is reference counted and its access is mutex protected. We may consider optimization in the future such as using RCU synchronization when available. Java/Go implementation accesses the scheduler with an atomic reference object.

The picker starts a timer to swap the scheduler to update update weights every `weight_update_period`. This timer iterates over the subchannel list (wrapper) of the picker, loads the latest weight from each, builds a new scheduler and replaces it with the current one. Note that synchronization must be used to access the weights, since they are going to be set based on backend metric reports in either RPC reports or OOB responses but then accessed in this timer. When the scheduler is swapped, ongoing pick calls that already took a reference of the outgoing scheduler can continue to refer to the old scheduler until the call returns.

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

### Handling Parent/Resolver Updates

When `enable_oob_load_report` value changes, either per-call or OOB load report handling is enabled and the other is disabled, depending on the current value. For the change to per-call reporting, since `weighted_round_robin` always creates a new picker when it gets an update, the policy will configure the new picker as to whether it should request per-call load report data. For enabling or disabling OOB reporting the exact mechanism depends on the language. In C/C++, a new pending subchannel list is created in both cases to enable or disable the OOB load report watcher. In Java/Go, the OOB listener is started/stopped on the existing subchannels. Note this requires API changes, see below for more. In Go, the OOB load report handling implementation is pending.

When `oob_reporting_period` changes, the client implementation starts a new OOB load reporting stream as outlined in [gRFC A51][A51].

Changes to `weight_update_period`, `blackout_period`, and `weight_expiration_period` take effective with a new picker that we always create upon receiving a resolver update.

In all languages, `weighted_round_robin` policy de-duplicates redundant addresses in resolver updates. This behavior is different from `round_robin` where it uses the number of representations in the list as a resolver supplied weight.

### Handling Subchannel Connectivity State Notifications
The behavior remains the same as in `round_robin` in all languages.

The rules to determine the aggregate connectivity state from subchannel states are the same as in `round_robin`: `READY` if any subchannel is `READY`, else `CONNECTING` if any is `CONNECTING`, else `TRANSIENT_FAILURE` if all are in `TRANSIENT_FAILURE`.

In C/C++, the LB policy creates a new pending subchannel list upon getting a new address list and starts watching the connectivity state of its subchannels. The pending list is promoted to the current list when it becomes connected. When the pending list is promoted to the current list, or when the connectivity state of a subchannel on the current list changes, the LB policy updates the aggregate connectivity state and returns a new picker.

In Java, the current subchannel list is updated directly as in `round_robin`. Upon getting an updated address list, it adds new subchannels to the subchannel list, starts watching the connectivity state of it, and requests connection. It also deletes removed subchannels from the subchannel list and shuts them down. It then updates the aggregate connectivity state, and if there is at least one `READY` subchannel, creates a new picker. The kind of picker created depends on the aggregate state: an active Picker (with a scheduler) if `READY`, EmptyPicker otherwise. When the connectivity state of a subchannel changes, it triggers updating the aggregate connectivity state.

In Go, `baseBalancer` handles the address update as in `round_robin`. When the address set changes, the `baseBuilder` calls the picker builder of `weighted_round_robin` if the aggregate connectivity state is `READY`. When the connectivity state of a subchannel changes, it triggers the same process as when the address set changes.

In all languages, when a subchannel state is updated to `IDLE`, the LB policy immediately requests a connection and treats it as if it is in `CONNECTING` for the aggregation purpose.

### Subchannel Weights

To enforce `blackout_period` and `weight_expiration_period`, the LB policy tracks two timestamps along with the weight. `last_updated` tracks the last time a non-zero report was received. `non_empty_since` tracks when the first non-zero load report was received, and it is reset when a subchannel comes back to `READY` or the weight expires so that it can be updated with the next non-zero load report. The blackout and expiration are enforced when the weight value is looked up.

The weight of a backend is calculated as ***weight = qps / cpuutilization***.

In Java and Go, the weight will be stored in a wrapped subchannel. However, in C++, that approach won't work, because we create a new subchannel object for each address whenever we get a new address list, and we don't want to lose existing weight information when that happens. So instead, we store the weights in a map keyed by address, and each subchannel list will take its own ref to the entry in the map for each subchannel in the list.

```
function UpdateWeight(qps, cpu_utilization) {
  var new_weight =
     cpu_utilization == 0 ? 0 : qps / cpu_utilization;
  if (new_weight == 0) return;
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
   if (blackout_period_ > 0 &&
       now() - non_empty_since_ < blackout_period_) return 0;
   return weight_;
}

```

### Per-Call Load Reporting

`weighted_round_robin` defaults to per-call load reporting. The client does nothing when the per-call load report is expected but missing in the trailing metadata of the response. While `enable_oob_load_report` is set, per-call load reports are never parsed on the client side. The client does nothing when the per-call load report is present in the response when not expected.

#### C/C++
Processing per-call load reports is supported with `SubchannelCallTrackerInterface`. `weighted_round_robin` implements this interface and attaches it to a subchannel `PickResult`. Later Finish() is called back with the load metrics and updates the weight.

#### Java
`OrcaPerRequestUtil.OrcaReportingTracerFactory` is available in `io.grpc.xds.orca` to process per-call backend load reports in `ClusterImplLoadBalancer`. This needs to be moved to a new package and shared with `weighted_round_robin`. Then the picker attaches this tracer to the subchannel selection and the listener takes the reference to the subchannel to which it updates the weight to in `onLoadReport()`.

#### Go
Picker selection result `PickResult` has an optional callback `Done()` that is passed with the backend load metrics parsed from the response if available. With `weighted_round_robin`, the picker only sets Done() in `PickResult` when per-call load reporting is enabled, and the callback updates the weight.

### OOB Load Reporting

When OOB reporting is enabled, the LB policy gets data via an OOB watch on each subchannel with the requested reporting interval set to the value of the `oob_reporting_period` LB config field. The watcher is started whenever the `enable_oob_load_report` LB config field becomes true and stopped whenever it becomes false, and it is restarted whenever the value of the `oob_reporting_period` field changes. (In C++, these changes happen automatically, because we create a new subchannel for each resolver update, and we create a new OOB watcher if appropriate for each new subchannel object.)

#### C/C++

Processing OOB load reports is supported with `OobBackendMetricWatcher`. `weighted_round_robin` implements this interface. The watcher takes the reference of the subchannel weight object and update weights to it.

#### Java

`OrcaOobUtil` is available in `io.grpc.xds.orca` but currently not in use. This needs to be moved to a new package and shared with `weighted_round_robin`, which implements `ForwardingLoadBalancerHelper` that takes the helper from `OrcaOobUtil`. The listener takes the reference to the subchannel to which it updates the weight to in `onLoadReport()`. A new API is needed to set and unset the listener in `OrcaOobUtil` to toggle the OOB load report processing.

#### Go

`orca.OOBListener` is available for OOB load reporting. `weighted_round_robin` extends it and attach it to the subchannel to which it updates the weight in `onLoadReport()`. A new API is needed to support enabling and disabling OOB load reporting.


### xDS Integration

`weighted_round_robin` is enabled as the endpoint picking policy of `WrrLocality` of [gRFC A52][A52].

`weighted_round_robin` is added as a new LB policy per [gRFC A52][A52] and is added to `XdsLbPolicyRegistry`. The name is prefixed with `ClientSide` to avoid confusion with the [gRFC A34 (pending)][A34] where the endpoint weights are sent by the control plane.

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

In the xDS configuration for the client, `client_side_weighted_round_robin` is wrapped in `wrr_locality` as a child policy. Below is an example xDS configuration with `client_side_weighted_round_robin`. The config includes `round_robin` as a backup policy to fallback in case the client does not support `client_side_weighted_round_robin`. On the client the new xDS LB hierarchy in [gRFC A52][A52] creates the gRPC LB policy `weighted_round_robin` as the endpoint-picking policy.

```textproto
{ // Cluster
  name: "..."
  type: EDS
  load_balancing_policy {
    policies {
      typed_extension_config {
        name: "..."
        typed_config {
          [type.googleapis.com/envoy.extensions.load_balancing_policies.wrr_locality.v3.WrrLocality] {
          endpoint_picking_policy {
            policies {
              typed_extension_config {
                name: "weighted_round_robin"
                typed_config {
                  [type.googleapis.com/envoy.extensions.load_balancing_policies.client_side_weighted_round_robin.v3.ClientSideWeightedRoundRobin] {}
                }
              }
              // Backup RR policy to fall back if WRR is unsupported.
              typed_extension_config: {
                name: "round_robin"
                typed_config {
                  [type.googleapis.com/envoy.extensions.load_balancing_policies.round_robin.v3.RoundRobin] {}
                }
              }
    // …
}
```

## Implementation

This will be implemented in all languages C++, Java, and Go.

[EDF]: https://en.wikipedia.org/wiki/Earliest_deadline_first_scheduling
[A34]: https://github.com/grpc/proposal/pull/202
[A51]: A51-custom-backend-metrics.md
[A52]: A52-xds-custom-lb-policies.md
