## A80: gRPC Metrics for TCP connection

*   Author(s): Yash Tibrewal (@yashykt), Nana Pang (@nanahpang), Yousuk Seung
    (@yousukseung)
*   Approver: Craig Tiller (@ctiller), Mark Roth (@markdroth)
*   Status: In Review
*   Implemented in:
*   Last updated: 2025-06-27
*   Discussion at: https://groups.google.com/g/grpc-io/c/AyT0LVgoqFs

## Abstract

This document proposes adding new TCP connection metrics to gRPC for improved
network analysis and debugging.

## Background

On Linux systems, TCP socket state can be retrieved using the `getsockopt()`
system call with `level` set to `IPPROTO_TCP` and `optname` set to `TCP_INFO`.
The state is returned in a `struct tcp_info` which gives details about the TCP
connection. At present, the machinery to collect such information is available
only on Linux 2.6 or later kernels.

[A79] provides a framework for adding non-per-call metrics in gRPC. This
document uses that framework to propose per-connection TCP metrics.

### Related Proposals:

*   [A79]: gRPC Non-Per-Call Metrics Framework

[A79]: A79-non-per-call-metrics-architecture.md

## Proposal

This document proposes adding per-connection TCP metrics in gRPC to improve the
network debugging capabilities for gRPC users.

Name                                    | Type                       | Unit     | Labels | Description
--------------------------------------- | -------------------------- | -------- | ------ | -----------
grpc.tcp.min_rtt                        | Histogram (floating-point) | s        | None   | TCP's current estimate of minimum round trip time (RTT). Corresponds to `tcpi_min_rtt` from `struct tcp_info`.
grpc.tcp.delivery_rate                  | Histogram (floating-point) | bit/s    | None   | Goodput measured of the TCP connection. Corresponds to `tcpi_delivery_rate` from `struct tcp_info`.
grpc.tcp.packets_sent                   | Counter (integer)          | {packet} | None   | TCP packets sent. Corresponds to `tcpi_data_segs_out` from `struct tcp_info`.
grpc.tcp.packets_retransmitted          | Counter (integer)          | {packet} | None   | TCP packets retransmitted. Corresponds to `tcpi_total_retrans` from `struct tcp_info`.
grpc.tcp.packets_spurious_retransmitted | Counter (integer)          | {packet} | None   | TCP packets spuriously retransmitted packets. Corresponds to `tcpi_dsack_dups` from `struct tcp_info`.

Suggested algorithm to collect metrics -

*   Set TCP_CONNECTION_METRICS_RECORD_INTERVAL to a default of 5 minutes.
    Implementations can choose to make it configurable.
*   For each new connected TCP socket, set an initial alarm of 10% to 110%
    (randomly selected) of TCP_CONNECTION_METRICS_RECORD_INTERVAL.
*   When the alarm fires -
    *   Use `getsockopt()` or equivalent method to retrieve and record
        connection metrics.
    *   Re-arm the alarm with TCP_CONNECTION_METRICS_RECORD_INTERVAL and repeat.
*   Before the socket is closed, cancel the alarm set above, and retrieve and
    record connection metrics, providing observability for short-lived
    connections as well.

These metrics will only available on systems running Linux 2.6 or later kernels
because the necessary machinery is only available on those platforms.

### Metric Stability

All metrics added in this proposal will start as experimental. The long term
goal will be to de-experimentalize them and have them be on by default, but the
exact criteria for that change are TBD.

## Rationale

This section is intended to help further understand how these metrics can be
used.

### Min RTT

`grpc.tcp.min_rtt` (`tcpi_min_rtt` from `struct tcp_info`) reports TCP's current
estimate of minimum round trip time (RTT), typically used as an indication of
the network health between two endpoints. It reflects the minimum transmission
time experienced by the application, which includes:

*   The physical length of the path between sender and receiver.
*   Queueing time caused by throttling and load.
*   Other transport-level events, such as delayed acknowledgement.

A step-change in min_rtt values means that traffic is being throttled, is
experiencing congestion, or has been re-routed through a different path.

### Delivery Rate

`grpc.tcp.delivery_rate` (`tcpi_delivery_rate` from `struct tcp_info`) records
the most recent "non-app-limited" throughput. The term "non-app-limited" means
that the link is saturated by the application. In other words, the delivery_rate
is calculated when the application has enqueued enough data to cause a backlog
on the socket. In the opposite case when the connection is app-limited, TCP is
unable to reliably compute throughput, and so collecting this metric would not
be meaningful.

### Transmission Metrics

`grpc.tcp.packets_sent` (`tcpi_data_segs_out` from `struct tcp_info`) reports
the TCP packets sent. This includes retransmissions and spurious
retransmissions.

`grpc.tcp.packets_retransmitted` (`tcpi_total_retrans` from `struct tcp_info`)
reports the TCP packets sent except those sent for the sent time. A packet may
be retransmitted multiple times and counted multiple times as retransmitted.
These counts include spurious retransmissions.

`grpc.tcp.packets_spurious_retransmitted` (`tcpi_dsack_dups` from `struct
tcp_info`) reports TCP packets acknowledged for the second time or more. These
are retransmissions that TCP later discovered unnecessary. A packet may be
counted as spuriously retransmitted multiple times. TCP can under-count spurious
retransmissions if the acknowledgements to reflect the spurious retransmissions
are lost. This may occur when the return path is highly congested.

#### Packet Loss Rate

TCP packet loss is the ratio of total packets that the TCP sender did not
receive an acknowledgement for out of all packets sent. It can be calculated as
(`grpc.tcp.packets_retransmitted` - `grpc.tcp.packets_spurious_retransmitted`) /
`grpc.tcp.packets_sent`.

TCP packet loss also includes loss NOT in the network. It is what the TCP sender
sees as “lost” from the transport layer’s perspective and includes loss from
various stages such as:

*   Acknowledgements lost on the return path
*   Drops on the sender below the transport layer (e.g. drops on the NIC)
*   Drops on the receiver NIC
*   Drops on the receiver due to out-of-memory conditions

TCP can overestimate the packet loss rate if acknowledgements for any
duplicate/spurious retransmissions are lost. Also note that the loss rate in TCP
generally covers only the data (and SYN) packets. In TCP, transmission of
acknowledgements is not reliable (i.e. not retransmitted when lost) and the true
acknowledgement loss rate is not available.

#### Spurious Retransmission Rate

Spurious retransmission rate can be calculated as
`grpc.tcp.packets_spurious_retransmitted` / `grpc.tcp.packets_retransmitted`.

TCP can underestimate the spurious retransmission rate. TCP can under-count
spurious retransmissions if the acknowledgements for the duplicate/spurious
retransmissions are lost.

### Temporary environment variable protection

This proposal does not include any features enabled via external I/O, so it does
not need environment variable protection.

## Implementation

Will be implemented in C-core, Java and Go.
