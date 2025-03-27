## A94: OTel metrics for Subchannels

*   Author(s): Yash Tibrewal (yashkt@)
*   Approver: Mark Roth (@markdroth)
*   Status: In Review
*   Implemented in:
*   Last updated: Mar 18, 2025
*   Discussion at: https://groups.google.com/g/grpc-io/c/iMdK7r4E5tU

## Abstract

Introduce metrics for subchannels. These metrics will replace the existing
pick-first metrics.

## Background

In [A78: gRPC OTel Metrics for WRR, Pick First, and XdsClient], metrics for
PickFirst load-balancing policy were proposed that provide observability on
disconnections for subchannels and connection attempts made for those
subchannels. These metrics do not currently contain information on the reason
for disconnection, the xds locality or the cluster information.

[A89: Backend Service Metric Label](https://github.com/grpc/proposal/pull/471)
is a proposal to introduce a new optional label `grpc.lb.backend_service` to
client-side per-attempt metrics. This label has xds cluster information.

### Related Proposals:

*   [A66: OpenTelemetry Metrics]
*   [A78: gRPC OTel Metrics for WRR, Pick First, and XdsClient]
*   [A79: Non-per-call Metrics Architecture]
*   [A89: Backend Service Metric Label]

[A66: OpenTelemetry Metrics]: A66-otel-stats.md
[A78: gRPC OTel Metrics for WRR, Pick First, and XdsClient]: A78-grpc-metrics-wrr-pf-xds.md
[A79: Non-per-call Metrics Architecture]: A79-non-per-call-metrics-architecture.md
[A89: Backend Service Metric Label]: https://github.com/grpc/proposal/pull/471

## Proposal

Move the existing pick-first metrics to subchannel metrics
(`grpc.lb.pick_first.*` to `grpc.subchannel.*`) with the addition of optional
labels as shown below -

Metric Name                                                                                            | Type           | Unit            | Labels                                                                                                         | Description
------------------------------------------------------------------------------------------------------ | -------------- | --------------- | -------------------------------------------------------------------------------------------------------------- | -----------
grpc.subchannel.disconnections (Old - grpc.lb.pick_first.disconnections)                               | Counter        | {disconnection} | grpc.target, grpc.lb.backend_service (optional), grpc.lb.locality (optional), grpc.disconnect_error (optional) | Number of times the selected subchannel becomes disconnected.
grpc.subchannel.connection_attempts_succeeded (Old - grpc.lb.pick_first.connection_attempts_succeeded) | Counter        | {attempt}       | grpc.target, grpc.lb.backend_service (optional), grpc.lb.locality (optional)                                   | Number of successful connection attempts.
grpc.subchannel.connection_attempts_failed (Old - grpc.lb.pick_first.connection_attempts_failed)       | Counter        | {attempt}       | grpc.target, grpc.lb.backend_service (optional), grpc.lb.locality (optional)                                   | Number of failed connection attempts.
grpc.subchannel.open_connections                                                                       | UpDown Counter | {connection}    | grpc.target, grpc.lb.backend_service (optional), grpc.lb.locality (optional)                                   | Number of open connections.

If we end up discarding connection attempts as we do with the “happy eyeballs”
algorithm (as per
[A61: IPv4 and IPv6 Dualstack Backend Support](A61-IPv4-IPv6-dualstack-backends.md)),
we should not record the connection attempt or the disconnection.

Implementations that have already implemented the pick-first metrics should give
enough time for users to transition to the new metrics. For example,
implementations should report both the old pick-first metrics and the new
subchannel metrics for 2 releases, and then remove the old pick-first metrics.

Label Name              | Disposition | Description
----------------------- | ----------- | -----------
grpc.target             | Required    | Indicates the target of the gRPC channel (defined in [A66](A66-otel-stats.md).)
grpc.lb.backend_service | Optional    | The backend service to which the RPC was routed (defined in [A89](https://github.com/grpc/proposal/pull/471))
grpc.lb.locality        | Optional    | The locality to which the traffic is being sent. This will be set to the resolver attribute passed down from the weighted_target policy, or the empty string if the resolver attribute is unset. (defined in [A78](A78-grpc-metrics-wrr-pf-xds.md))
grpc.disconnect_error   | Optional    | Reason for disconnection

List of allowed values for `grpc.disconnect_error` -

Error string         | Description
-------------------- | -----------
GOAWAY <ERROR_CODE>  | HTTP2 GOAWAY frame with error code for example (“GOAWAY NO_ERROR”, “GOAWAY PROTOCOL_ERROR”, “GOAWAY ENHANCE_YOUR_CALM”). The list of error codes is available in [RFC 9113](https://www.rfc-editor.org/rfc/rfc9113.html#name-error-codes).
subchannel shutdown  | The subchannel was shutdown. This can happen due to reasons such as the parent channel shutting down, channel becoming idle, the load balancing policy changing due to a resolver update or a change in list of endpoint addresses.
connection reset     | Connection was reset (eg. ECONNRESET, WSAECONNERESET)
connection timed out | Connection timed out, also includes connections closed due to gRPC keepalives.
connection aborted   | Connection was aborted
socket error         | Any socket error not covered by “connection reset”, “connection timed out” and “connection aborted”. Implementations that are not able to differentiate between the different socket error codes should also use this.
unknown              | Catch-all for all other reasons.

For a given connection, there can be multiple reasons reported to the subchannel
for disconnection. For example, a connection could have seen a GOAWAY frame with
`ENHANCE_YOUR_CALM` and then a socket error Broken Pipe. In such cases, the
first seen reason should be chosen, `GOAWAY ENHANCE_YOUR_CALM` in this case.

We might add more error cases to this in the future.

## Rationale

### Renaming pick-first metrics

The existing pick-first metrics provides stats on subchannel disconnections and
connection attempts as viewed from the perspective of the pick-first lb policy.
[A61: IPv4 and IPv6 Dualstack Backend Support](A61-IPv4-IPv6-dualstack-backends.md)
made pick-first lb policy the universal leaf policy. For users unfamiliar with
this, it will come as a surprise when metrics for pick-first lb policy are
populated when round_robin lb policy is configured (for example). Additionally,
the pick-first metrics are defined from the perspective of the channel. This
means that if subchannels are shared between multiple channels (as is the case
for gRPC Core and its wrapped languages - C++, Python), we will double-count the
disconnections/connection attempts.

Renaming/moving the pick-first metrics to subchannel makes this more intuitive,
and fixes the double-counting problem.

### Metric for open connections

Moving the metrics down to subchannel potentially allows us to calculate the
number of open connections by subtracting `grpc.subchannel.disconnections` from
`grpc.subchannel.connection_attempts_succeeded`. This method does not work for
exporters recording counters per period in a way that does not allow for a
simple subtraction of the two counters
(https://github.com/grpc/grpc/issues/34886).

Adding an explicit metric that records the number of open connections avoids
this.

### Use “connection timed out” as disconnect error for gRPC Keepalive timeouts

We expect most implementations of [gRPC keepalives](A8-client-side-keepalive.md)
to also set the POSIX socket option `TCP_USER_TIMEOUT` as stated in
[A18: TCP User Timeout](A18-tcp-user-timeout.md). As such, in cases where the
connection is broken, sockets that would otherwise be closed due to gRPC
keepalive timing out, would instead be closed due to `TCP_USER_TIMEOUT`.

## Implementation

TBD
