A78: gRPC OTel Metrics for WRR, Pick First, and XdsClient
----
* Author(s): @markdroth
* Approver: @ejona86, @dfawley
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2024-09-24
* Discussion at: https://groups.google.com/g/grpc-io/c/A2Mqz8OMDys
* Updated by: [A88: xDS Data Error Handling](A88-xds-data-error-handling.md), [A94: OTel metrics for Subchannels](A94-subchannel-otel-metrics.md)

## Abstract

This document proposes some new metrics that will be added in gRPC for the
Weighted Round Robin (WRR) and Pick First LB policies and for the XdsClient.
It also adds a new optional label for the existing per-call metrics.

## Background

gRPC recently added a set of basic per-call metrics, defined in [A66].
[A79] is building upon that by providing a framework for non-per-call
metrics.  The metrics described in this document will be the first
metrics added using that new non-per-call metric framework.

### Related Proposals: 
* [A66]: OpenTelemetry Metrics
* [A79]: gRPC Non-Per-Call Metrics Framework (pending)
* [A58]: Weighted Round Robin LB Policy
* [A61]: IPv4 and IPv6 Dualstack Backend Support
* [A62]: Pick First: Sticky TRANSIENT_FAILURE and Address Order Randomization
* [A27]: xDS-Based Global Load Balancing
* [A28]: xDS Traffic Splitting and Routing
* [A71]: xDS Fallback
* [A57]: XdsClient Failure Mode Behavior

[A66]: A66-otel-stats.md
[A79]: https://github.com/grpc/proposal/pull/421
[A58]: A58-client-side-weighted-round-robin-lb-policy.md
[A61]: A61-IPv4-IPv6-dualstack-backends.md
[A62]: A62-pick-first.md
[A27]: A27-xds-global-load-balancing.md
[A28]: A28-xds-traffic-splitting-and-routing.md
[A71]: A71-xds-fallback.md
[A57]: A57-xds-client-failure-mode-behavior.md

## Proposal

This document proposes changes to the following gRPC components.

### Optional xDS Locality Label

When xDS is used, it is desirable for some metrics to include an optional
label indicating which xDS locality the metrics are associated with.
We want to provide this optional label for the metrics in both the
existing per-call metrics defined in [A66] and in the new metrics for
the WRR LB policy, described below.

If locality information is available, the value of this label will be of
the form `{region="${REGION}", zone="${ZONE}", sub_zone="${SUB_ZONE}"}`,
where `${REGION}`, `${ZONE}`, and `${SUB_ZONE}` are replaced with the
actual values.  If no locality information is available, the label will
be set to the empty string.

#### Per-Call Metrics

To support the locality label in the per-call metrics, we will provide
a mechanism for LB picker to add optional labels to the call attempt
tracer.  We will then use this mechanism in the `xds_cluster_impl`
policy's picker to set the locality label.  It will get the locality
label from the wrapped subchannel that it is already creating for load
reporting purposes, when that subchannel is returned by the child picker.

This label will be available on the following per-call metrics:
- `grpc.client.attempt.duration`
- `grpc.client.attempt.sent_total_compressed_message_size`
- `grpc.client.attempt.rcvd_total_compressed_message_size`

#### Weighted Target LB Policy

To support the locality label in the WRR metrics, we will extend the
`weighted_target` LB policy (see [A28]) to define a resolver attribute
that indicates the name of its child.  This attribute will be passed down
to each of its children with the appropriate value, so that any LB policy
that sits underneath the `weighted_target` policy will be able to use it.

### Weighted Round Robin LB Policy

The `weighted_round_robin` LB policy is described in [A58].  We propose to
add the following metrics to it.

WRR metrics will have the following labels:

| Name        | Disposition | Description |
| ----------- | ----------- | ----------- |
| grpc.target | required | Indicates the target of the gRPC channel in which WRR is used.  (Same as the attribute defined in [A66].) |
| grpc.lb.locality | optional | The locality to which the traffic is being sent. This will be set to the resolver attribute passed down from the `weighted_target` policy, or the empty string if the resolver attribute is unset. |

The following metrics will be exported:

| Name          | Type  | Unit  | Labels  | Description |
| ------------- | ----- | ----- | ------- | ----------- |
| grpc.lb.wrr.rr_fallback | Counter | {update} | grpc.target, grpc.lb.locality | Number of scheduler updates in which there were not enough endpoints with valid weight, which caused the WRR policy to fall back to RR behavior. |
| grpc.lb.wrr.endpoint_weight_not_yet_usable | Counter | {endpoint} | grpc.target, grpc.lb.locality | Number of endpoints from each scheduler update that don't yet have usable weight information (i.e., either the load report has not yet been received, or it is within the blackout period). |
| grpc.lb.wrr.endpoint_weight_stale | Counter | {endpoint} | grpc.target, grpc.lb.locality | Number of endpoints from each scheduler update whose latest weight is older than the expiration period. |
| grpc.lb.wrr.endpoint_weights | Histogram | {weight} | grpc.target, grpc.lb.locality | Weight of each endpoint, recorded on every scheduler update. Endpoints without usable weights will be recorded as weight 0. |

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

Note that these metrics are defined from the perspective of the Pick
First policy and not from the perspective of the subchannel, because
we want them to reflect the behavior from the channel's perspective.
For example, multiple subchannels may successfully establish a connection
at basically the same moment due to Happy Eyeballs (see [A61]), but only
one of them will actually be used by Pick First, so we want to increment
the counter only once.  Similar cases can also occur in C-core due to
subchannel sharing.

### XdsClient

The XdsClient component was originally described in [A27].  Note that in
[A71], we are moving from a single global XdsClient instance to a
separate global XdsClient instance for each channel target.  The
proposed metric schema here reflects that change.

XdsClient metrics will have the following labels:

| Name        | Disposition | Description |
| ----------- | ----------- | ----------- |
| grpc.target | required | For clients, indicates the target of the gRPC channel in which the XdsClient is used (i.e., the same as the attribute defined in [A66]). For servers, will be the string "#server". |
| grpc.xds.server | required | The target URI of the xDS server with which the XdsClient is communicating. |
| grpc.xds.authority | required | The xDS authority.  The value will be "#old" for old-style non-xdstp resource names. |
| grpc.xds.cache_state | required | Indicates the cache state of an xDS resource.  The value will be one of: <ul><li>"requested": The resource has been requested from the xDS server but has not yet been received.<li>"does_not_exist": The server has indicated that the resource does not exist.<li>"acked": The resource has been received and is valid.<li>"nacked": The resource was received but was not valid.<li>"nacked_but_cached": There is a version of the resource cached, but the most recent update of the resource was invalid.</ul> |
| grpc.xds.resource_type | required | Indicates an xDS resource type, such as "envoy.config.listener.v3.Listener". |

The following metrics will be exported:

| Name          | Type  | Unit  | Labels  | Description |
| ------------- | ----- | ----- | ------- | ----------- |
| grpc.xds_client.connected | Gauge | {bool} | grpc.target, grpc.xds.server | Whether or not the xDS client currently has a working ADS stream to the xDS server.  For a given server, this will be set to 1 when the stream is initially created.  It will be set to 0 when we have a connectivity failure or when the ADS stream fails without seeing a response message, as per [A57].  Once set to 0, it will be reset to 1 when we receive the first response on an ADS stream. |
| grpc.xds_client.server_failure | Counter | {failure} | grpc.target, grpc.xds.server | A counter of xDS servers going from healthy to unhealthy.  A server goes unhealthy when we have a connectivity failure or when the ADS stream fails without seeing a response message, as per gRFC A57. |
| grpc.xds_client.resource_updates_valid | Counter | {resource} | grpc.target, grpc.xds.server, grpc.xds.resource_type | A counter of resources received that were considered valid.  The counter will be incremented even for resources that have not changed. |
| grpc.xds_client.resource_updates_invalid | Counter | {resource} | grpc.target, grpc.xds.server, grpc.xds.resource_type | A counter of resources received that were considered invalid. |
| grpc.xds_client.resources | Gauge | {resource} | grpc.target, grpc.xds.authority, grpc.xds.cache_state, grpc.xds.resource_type | Number of xDS resources. |

### Metric Stability

All metrics added in this proposal will start as experimental and
therefore off by default.  The long term goal will be to
de-experimentalize them and have them be on by default, but the exact
criteria for that change are TBD.

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

C-core implementation:
- WRR metrics: https://github.com/grpc/grpc/pull/35977
- pick_first metrics: https://github.com/grpc/grpc/pull/35984
- XdsClient metrics: https://github.com/grpc/grpc/pull/36020, https://github.com/grpc/grpc/pull/36291
- locality label: https://github.com/grpc/grpc/pull/36049

Will be implemented in Java by @dnvindhya and in Go by @zasweq.
