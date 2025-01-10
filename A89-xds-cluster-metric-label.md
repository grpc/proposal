A89: xDS Cluster Metric Label
----
* Author(s): [Eric Anderson](https://github.com/ejona86)
* Approver: @markdroth
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2025-01-10
* Discussion at: https://groups.google.com/g/grpc-io/c/s4tm26RiMyI

## Abstract

Add a new optional label to per-call metrics containing the xDS cluster being
used for the RPC.

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

[gRFC A78]: A78-grpc-metrics-wrr-pf-xds.md#per-call-metrics
[gRFC A79]: A79-non-per-call-metrics-architecture.md

## Proposal

Each pick in the `xds_cluster_impl` policy, `xds_cluster_impl` will add the
optional label `grpc.xds.cluster` to the call attempt tracer. The value will be
copied from `xds_cluster_impl`'s service config `cluster` key. This is done
regardless of the pick's result. It is possible for later picks for the same RPC
to have a different value. This is the case for locality as well, and the last
pick's value should be used.

The `grpc.xds.cluster` label will be available on the following per-call
metrics:
- `grpc.client.attempt.duration`
- `grpc.client.attempt.sent_total_compressed_message_size`
- `grpc.client.attempt.rcvd_total_compressed_message_size`

### Temporary environment variable protection

The new optional label requires calling an API to activate, so environment
variable protection is unnecessary.

## Rationale

gRFC A78 added `grpc.lb.locality` to per-call and WRR metrics, while this is
only adding the new label to per-call. Cluster is an xDS-specific concept, so
it is more awkward to add to WRR and that is left as potential future work.

Which "cluster" was used for a request is ambiguous when using aggregate
clusters as multiple clusters are involved. For placing in a label, there are
two potential choices: the top-level aggregate cluster and the leaf cluster.
Using the leaf cluster seems to provide the most insight when using aggregate
clusters as failing over to a different priority would be significant. If the
top-level cluster is needed in the future, it can be added as well.

## Implementation

@ejona86 will immediately implement in gRPC Java. Other languages will follow as
able. The implementation is very quick.
