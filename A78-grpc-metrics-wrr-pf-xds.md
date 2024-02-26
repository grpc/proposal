A78: gRPC OTel Metrics for WRR, Pick First, and XdsClient
----
* Author(s): @markdroth
* Approver: @ejona86, @dfawley
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2024-02-23
* Discussion at: https://groups.google.com/g/grpc-io/c/A2Mqz8OMDys

## Abstract

This document proposes some new metrics that will be added in gRPC for the
Weighted Round Robin (WRR) and Pick First LB policies and for the XdsClient.

## Background

gRPC recently added a set of basic per-call metrics, defined in [A66].
[A79] is building upon that by providing a framework for non-per-call
metrics.  The metrics described in this document will be the first
metrics added using that new non-per-call metric framework.

### Related Proposals: 
* [A66]: OpenTelemetry Metrics
* [A79]: gRPC Non-Per-Call Metrics Framework (pending)
* [A58]: Weighted Round Robin LB Policy
* [A62]: Pick First: Sticky TRANSIENT_FAILURE and Address Order Randomization
* [A27]: xDS-Based Global Load Balancing
* [A28]: xDS Traffic Splitting and Routing
* [A71]: xDS Fallback
* [A57]: XdsClient Failure Mode Behavior

[A66]: A66-otel-stats.md
[A79]: https://github.com/grpc/proposal/pull/421
[A58]: A58-client-side-weighted-round-robin-lb-policy.md
[A62]: A62-pick-first.md
[A27]: A27-xds-global-load-balancing.md
[A28]: A28-xds-traffic-splitting-and-routing.md
[A71]: A71-xds-fallback.md
[A57]: A57-xds-client-failure-mode-behavior.md

## Proposal

This document proposes changes to the following gRPC components.

### Weighted Target LB Policy

When xDS is used, it is desirable for LB policies to export metrics that
include a label indicating which xDS locality the metrics are associated
with.  In particular, we want to provide this optional label for the
metrics in the WRR LB policy, described below.

To support this, we will extend the `weighted_target` LB policy (see
[A28]) to define a resolver attribute that indicates the name of its
child.  This attribute will be passed down to each of its children with
the appropriate value, so that any LB policy that sits underneath the
`weighted_target` policy will be able to use it.

### Weighted Round Robin LB Policy

The `weighted_round_robin` LB policy is described in [A58].  We propose to
add the following metrics to it.

WRR metrics will have the following labels:

| Name        | Disposition | Description |
| ----------- | ----------- | ----------- |
| grpc.target | required | Indicates the target of the gRPC channel in which WRR is used.  (Same as the attribute defined in [A66].) |
| grpc.lb.locality | optional | The locality to which the traffic is being sent. This will be based on the resolver attribute passed down from the `weighted_target` policy. |

The following metrics will be exported:

| Name          | Type  | Unit  | Labels  | Description |
| ------------- | ----- | ----- | ------- | ----------- |
| grpc.lb.wrr.rr_fallback | Counter | {update} | grpc.target, grpc.locality | Number of scheduler updates in which there were not enough endpoints with valid weight, which caused the WRR policy to fall back to RR behavior. |
| grpc.lb.wrr.endpoint_weight_not_yet_usable | Counter | {endpoint} | grpc.target, grpc.locality | Number of endpoints from each scheduler update that don't yet have usable weight information (i.e., either the load report has not yet been received, or it is within the blackout period). |
| grpc.lb.wrr.endpoint_weight_stale | Counter | {endpoint} | grpc.target, grpc.locality | Number of endpoints from each scheduler update whose latest weight is older than the expiration period. |
| grpc.lb.wrr.endpoint_weights | Histogram | {weight} | grpc.target, grpc.locality | The histogram buckets will be endpoint weight ranges.  Each bucket will be a counter that is incremented once for every endpoint whose weight is within that range.  Note that endpoints without usable weights will have weight 0. |

### Pick First LB Policy

The Pick First LB policy predates the gRFC process but was updated in
[A62].  We propose to add the following metrics to it.

Pick First metrics will have the following labels:

| Name        | Disposition | Description |
| ----------- | ----------- | ----------- |
| grpc.target | required | Indicates the target of the gRPC channel in which PF is used.  (Same as the attribute defined in [A66].) |

The following metrics will be exported:

| Name          | Type  | Unit  | Labels  | Description |
| ------------- | ----- | ----- | ------- | ----------- |
| grpc.lb.pick_first.disconnections | Counter | {disconnection} | grpc.target | Number of times the selected subchannel becomes disconnected. |
| grpc.lb.pick_first.connection_attempts_succeeded | Counter | {attempt} | grpc.target | Number of successful connection attempts. |
| grpc.lb.pick_first.connection_attempts_failed | Counter | {attempt} | grpc.target | Number of failed connection attempts. |

### XdsClient

The XdsClient component was originally described in [A27].  Note that in
[A71], we are moving from a single global XdsClient instance to a
separate global XdsClient instance for each channel target.  The
proposed metric schema here reflects that change.

XdsClient metrics will have the following labels:

| Name        | Disposition | Description |
| ----------- | ----------- | ----------- |
| grpc.target | required | For clients, indicates the target of the gRPC channel in which the XdsClient is used (i.e., the same as the attribute defined in [A66]). For servers, will be the string "#server". |
| grpc.xds.server | required | The name of the xDS server with which the XdsClient is communicating. |
| grpc.xds.authority | required | The xDS authority.  The value will be "old" for old-style non-xdstp resource names. |
| grpc.xds.cache_state | required | Indicates the cache state of an xDS resource.  The value will be one of: <ul><li>"requested": The resource has been requested from the xDS server but has not yet been received.<li>"does_not_exist": The server has indicated that the resource does not exist.<li>"acked": The resource has been received and is valid.<li>"nacked": The resource was received but was not valid.<li>"nacked_but_cached": There is a version of the resource cached, but the most recent update of the resource was invalid.</ul> |
| grpc.xds.resource_type | required | Indicates an xDS resource type, such as "envoy.config.listener.v3.Listener". |

The following metrics will be exported:

| Name          | Type  | Unit  | Labels  | Description |
| ------------- | ----- | ----- | ------- | ----------- |
| grpc.xds_client.connected | Gauge | {bool} | grpc.target, grpc.xds.server | Whether or not the xDS client currently has a working ADS stream to the xDS server.  For a given server, this will be set to 0 when we have a connectivity failure or when the ADS stream fails without seeing a response message, as per [A57].  It will be set to 1 when we receive the first response on an ADS stream. |
| grpc.xds_client.resource_updates | Counter | {update} | grpc.target, grpc.xds.server, grpc.xds.resource_type | A counter of resource updates from the xDS server.  Note that this is a count of resources, not response messages; if a response message contains two resources, then we will increment the counter twice.  The counter will be incremented even for resources that have not changed. |
| grpc.xds_client.resources | Gauge | {resource} | grpc.target, grpc.xds.server, grpc.xds.authority, grpc.xds.cache_state, grpc.xds.resource_type | Number of xDS resources. |

### Temporary environment variable protection

This proposal does not include any features enabled via external I/O, so
it does not need environment variable protection.

## Rationale

The metrics defined here are generally a trade-off between the usefulness
of the metric and the cost of reporting it.  As an example, for the
WRR metrics, we considered exporting a histogram of how old the last
backend load report was for each RPC sent to a particular backend, but
that would have been extremely expensive.  So instead, we are reporting
the number of stale weights on each scheduler update.

## Implementation

Will be implemented in C-core by @markdroth, in Java by @dnvindhya, and
in Go by @zasweq.
