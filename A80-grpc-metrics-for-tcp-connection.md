A80: gRPC Metrics for TCP connection
----
* Author(s):  Yash Tibrewal (@yashykt), Nana Pang (@nanahpang), Yousuk Seung (@yousukseung)
* Approver: Craig Tiller (@ctiller), Mark Roth (@markdroth)
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* language: {...}
* Last updated: 2024-04-18
* Discussion at: https://groups.google.com/g/grpc-io/c/AyT0LVgoqFs

## Abstract

This document proposes adding new TCP connection metrics to gRPC for improved network analysis and debugging.

## Background

To improve the network debugging capabilities for gRPC users, we propose adding per-connection TCP metrics in gRPC. The metrics will utilize the metrics framework outlined in  [A79].

### Related Proposals: 
* [A79]: gRPC Non-Per-Call Metrics Framework

[A79]: A79-non-per-call-metrics-architecture.md

## Proposal

This document proposes changes to the following gRPC components.

#### Per-Connection TCP Metrics

We will provide the following metrics:
- `grpc.tcp.min_rtt`
- `grpc.tcp.delivery_rate`
- `grpc.tcp.packets_sent`
- `grpc.tcp.packets_retransmitted`
- `grpc.tcp.packets_spurious_retransmitted`

The metrics will have label:

| Name        | Disposition | Description |
| ----------- | ----------- | ----------- |
| grpc.tcp.server_address | optional | Store the server address info in URI format such as `ipv4:1.2.3.4:567`. For clients, this address is the same as the peer address, while on the server side, it's the same as the local address. |

The metrics will be exported as:

| Name          | Type  | Unit  | Labels  | Description |
| ------------- | ----- | ----- | ------- | ----------- |
| grpc.tcp.min_rtt | Histogram (double) | s | grpc.tcp.server_address | Records TCP's current estimate of minimum round trip time (RTT), typically used as an indication of the network health between two endpoints.<br /> RTT = packet acked timestamp - packet sent timestamp. |
| grpc.tcp.delivery_rate | Histogram (double) | bit/s | grpc.tcp.server_address | Records latest goodput measured of the TCP connection. <br /> Elapsed time = packet acked timestamp - last packet acked timestamp. <br /> Delivery rate = packet acked bytes / elapsed time. |
| grpc.tcp.packets_sent | Counter (uint64) | {packet} | grpc.tcp.server_address | Records total packets TCP sends in the calculation period. |
| grpc.tcp.packets_retransmitted | Counter (uint64) | {packet} | grpc.tcp.server_address | Records total packets lost in the calculation period, including lost or spuriously retransmitted packets. |
| grpc.tcp.packets_spurious_retransmitted | Counter (uint64) | {packet} | grpc.tcp.server_address | Records total packets spuriously retransmitted packets in the calculation period. These are retransmissions that TCP later discovered unnecessary.|


#### Metric Collection Design

A high-level approach to collecting TCP metrics (on Linux) is as follows:
1) **Enable Network Timestamps for Metric Calculation:** Enable the `SO_TIMESTAMPING` option in the kernel's TCP stack through the `setsocketopt(fd, SOL_SOCKET, SO_TIMESTAMPING, &val, sizeof(val))` system call. This enables the kernel to capture packet timestamps during transmission.
2) **Calculate Metrics from Timestamps:**  Linux kernel calculates TCP connection metrics based on the captured packet timestamps. These metrics can be retrieved using the `getsockopt(TCP_INFO)` system call. For example, the delivery_rate metric estimates the goodput—the rate of useful data transmitted—for the most recent group of outbound data packets within a single flow ([code](https://elixir.bootlin.com/linux/v5.11.1/source/net/ipv4/tcp.c#L391)).
3) **Periodically Collect Statistics:** At a specified time interval (e.g., every 5 minutes), gRPC aggregates the calculated metrics and updates the corresponding statistics records.


#### Reference: 
* Fathom: https://dl.acm.org/doi/pdf/10.1145/3603269.3604815
* Kernel TCP Timestamping: https://www.kernel.org/doc/Documentation/networking/timestamping.rst
* Delivery Rate: https://datatracker.ietf.org/doc/html/draft-cheng-iccrg-delivery-rate-estimation#name-delivery-rate

### Metric Stability

All metrics added in this proposal will start as experimental. The long term goal will be to
de-experimentalize them and have them be on by default, but the exact
criteria for that change are TBD.

### Temporary environment variable protection

This proposal does not include any features enabled via external I/O, so
it does not need environment variable protection.

## Implementation

Will be implemented in C-core, and currently have no plans to implement in other languages.
