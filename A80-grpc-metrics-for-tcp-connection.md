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
* [A79]: gRPC Non-Per-Call Metrics Framework (pending)

[A79]: https://github.com/grpc/proposal/pull/421

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
| grpc.tcp.peer_address | optional | Store the peer address info in URI format such as `ipv4:1.2.3.4:567`. |
| grpc.tcp.local_address | optional | Store the local address info in URI format such as `ipv4:1.2.3.4:567`. |

The metrics will be exported as:

| Name          | Type  | Unit  | Labels  | Description |
| ------------- | ----- | ----- | ------- | ----------- |
| grpc.tcp.min_rtt | Histogram (double) | s | grpc.tcp.peer_address, grpc.tcp.local_address | Records TCP's current estimate of minimum round trip time (RTT), typically used as an indication of the network health between two endpoints.  |
| grpc.tcp.delivery_rate | Histogram (double) | bit/s | grpc.tcp.peer_address, grpc.tcp.local_address | Records latest throughput measured of the TCP connection. |
| grpc.tcp.packets_sent | Counter (int64) | {packet} | grpc.tcp.peer_address, grpc.tcp.local_address | Records total packets TCP sends in the calculation period. |
| grpc.tcp.packets_retransmitted | Counter (int64) | {packet} | grpc.tcp.peer_address, grpc.tcp.local_address | Records total packets lost in the calculation period, including lost or spuriously retransmitted packets. |
| grpc.tcp.packets_spurious_retransmitted | Counter (int64) | {packet} | grpc.tcp.peer_address, grpc.tcp.local_address | Records total packets spuriously retransmitted packets in the calculation period. These are retransmissions that TCP later discovered unnecessary.|

The metrics are acquired by enabling the `SO_TIMESTAMPING` option in the kernel's TCP stack via the `setsocketopt(fd, SOL_SOCKET, SO_TIMESTAMPING, &val, sizeof(val))` system call. This configuration allows the kernel to capture packet timestamps during transmission and subsequently provide relevant socket information when `getsockopt(TCP_INFO)` is invoked.

#### Reference: 
* Fathom: https://dl.acm.org/doi/pdf/10.1145/3603269.3604815
* Kernel TCP Timestamping: https://www.kernel.org/doc/Documentation/networking/timestamping.rst

### Metric Stability

All metrics added in this proposal will start as experimental. The long term goal will be to
de-experimentalize them and have them be on by default, but the exact
criteria for that change are TBD.

### Temporary environment variable protection

This proposal does not include any features enabled via external I/O, so
it does not need environment variable protection.

## Implementation

Will be implemented in C-core, and currently have no plans to implement in other languages.


