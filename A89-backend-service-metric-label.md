A89: Backend Service Metric Label
----
* Author(s): [Eric Anderson](https://github.com/ejona86)
* Approver: @markdroth
* Status: Ready for Implementation
* Implemented in: <language, ...>
* Last updated: 2025-04-01
* Discussion at: https://groups.google.com/g/grpc-io/c/s4tm26RiMyI

## Abstract

Add a new optional label to per-call metrics containing the "backend service"
being used for the RPC. When using xDS, the backend service would be the xDS
cluster name.

## Background

[gRFC A78][] added the `grpc.lb.locality` per-call optional label, which also
added the infrastructure to support LBs adding optional labels to per-call
metrics. The optional label can be enabled in the gRPC/OpenTelemetry integration
API added in [gRFC A79][].

Similar to how locality metrics are useful for analyzing _where_ traffic is
being routed, the xDS cluster is useful for knowing _to whom_ it is being
routed. `grpc.target` is generally all that's necessary to know which service is
receiving traffic, but non-deterministic routing in xDS like weighted clusters,
aggregate clusters, and cluster specifier plugins mean different clusters (and
thus potentially different services or service versions) would comingle metrics
unless the selected cluster is added as a label. It can also be helpful to know
the selected cluster to confirm that deterministic routing, like path matching,
is behaving as expected.

### Related Proposals:
* [gRFC A78: gRPC OTel Metrics for WRR, Pick First, and XdsClient][gRFC A78]
* [gRFC A79: Non-per-call Metrics Architecture][gRFC A79]
* [gRFC A75: xDS Aggregate Cluster Behavior Fixes][gRFC A75]

[gRFC A78]: A78-grpc-metrics-wrr-pf-xds.md#per-call-metrics
[gRFC A79]: A79-non-per-call-metrics-architecture.md
[gRFC A75]: A75-xds-aggregate-cluster-behavior-fixes.md

## Proposal

### Add grpc.lb.backend_service to per-call metrics

We will add the `grpc.lb.backend_service` optional label to the following
per-call metrics:
- `grpc.client.attempt.duration`
- `grpc.client.attempt.sent_total_compressed_message_size`
- `grpc.client.attempt.rcvd_total_compressed_message_size`

Label definition:

| Name        | Disposition | Description |
| ----------- | ----------- | ----------- |
| grpc.lb.backend_service | optional | The backend service to which the traffic is being sent. This is relevant when a single channel target can be sent to different sets of servers. When using xDS, this will be the cluster name. When not relevant, the value will be the empty string. |

The value will be communicated to the gRPC OpenTelemetry module by the call
attempt tracer. When an LB policy provides the label value to the tracer it
will do so each pick that the information is available, regardless of the pick's
result. This allows metrics for DEADLINE_EXCEEDED and UNAVAILABLE failures to
include a relevant backend service. It is possible for later picks for the same
RPC to have a different value. This is the case for locality as well, and the
last pick's value should be used by the OpenTelemetry module.

The LB policy `xds_cluster_impl` will notify the call attempt tracer of the
`grpc.lb.backend_service` label. The value will be copied from
`xds_cluster_impl`'s service config `cluster` key.

### Add grpc.lb.backend_service to WRR metrics

We will add the `grpc.lb.backend_service` optional label, with the same
definition as for per-RPC metrics, to all existing WRR metrics:
- `grpc.lb.wrr.rr_fallback`
- `grpc.lb.wrr.endpoint_weight_not_yet_usable`
- `grpc.lb.wrr.endpoint_weight_stale`
- `grpc.lb.wrr.endpoint_weights`

The `weighted_round_robin` LB policy will read a resolver attribute that
indicates the name of the backend service. This resolver attribute should not be
considered limited to xDS. However, only `xds_cluster_impl` will set the
attribute at this time.

### Temporary environment variable protection

The new optional label requires calling an API to activate, so environment
variable protection is unnecessary.

## Rationale

We define a new concept "backend service" so that we can add this label to
non-xDS-specific code like the gRPC OpenTelemetry module. "Backend service"
isn't the best of names, but all other names seemed to be obviously worse (e.g.,
cluster) or no better. It is more important that the name not be misunderstood
than for the name to be meaningful.

Prior to A75 the only LB policy with a backend service concept is
`xds_cluster_impl`, as it needs to be below the `priority` policy. After A75
`cds` could have the code to provide the label instead, but it has no advantage
over `xds_cluster_impl` as there are no policies that would fail RPCs between
`cds` and `xds_cluster_impl`. Since A75 is not yet implemented everywhere, we'll
use `xds_cluster_impl` and can always change the policy adding the label in the
future if there becomes a reason.

gRFC A78 added `grpc.lb.locality` to per-call and WRR metrics, and this is
mirroring that approach with backend service. Adding backend service to WRR is
less essential than per-call metrics, simply because the WRR metrics are more
rarely used. But users of WRR would benefit from backend service. In general,
any metrics that have a locality label should probably also have backend
service.

In xDS, which "cluster" is used for a request is ambiguous when using aggregate
clusters, as multiple clusters are involved. For placing in a label, there are
two potential choices: the top-level aggregate cluster and the leaf cluster.
Using the leaf cluster seems to provide the most insight when using aggregate
clusters as failing over to a different priority would be significant. If the
top-level cluster is needed in the future, it can be added as well.

## Implementation

@ejona86 will immediately implement in gRPC Java. Other languages will follow as
able.
