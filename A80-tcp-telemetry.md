# A80: Export TCP Telemetry from gRPC

* Author(s): Aananth V (@aananthv), Nana Pang (@nanahpang), Yash Tibrewal (@yashykt), Yousuk Seung  (@yousukseung)
* Approver(s): Craig Tiller (@ctiller), Mark Roth (@markdroth)  
* Status: In Review  
* Implemented in: C-Core  
* Last updated: 2025-12-16
* Discussion at: https://groups.google.com/g/grpc-io/c/MoLrWPsFB3s

## Abstract

This document proposes collecting and exposing new TCP Endpoint Level Telemetry to gRPC for improved network analysis and debugging.

## Background

The Linux Kernel exposes two telemetry hooks that can be used to collect TCP-level metrics.

1. **TCP socket state** can be retrieved using the `getsockopt()` system call with `level` set to `IPPROTO_TCP` and `optname` set to `TCP_INFO`. The state is returned in a `struct tcp_info` which gives details about the TCP connection. At present, the machinery to collect such information is available on Linux 2.6 or later kernels.  
2. **Per-message transmission timestamps** can be collected from TCP sockets using the [SO\_TIMESTAMPING](https://docs.kernel.org/networking/timestamping.html) interface. At present, this is available on Linux 2.6 or later kernels. These timestamps can be very valuable for diagnosing network level issues and can be used to break down the time spent in the "network".

[gRFC A79][A79] provides a framework for adding non-per-call metrics in gRPC. This document uses that framework to expose the proposed TCP Latency metrics.

### *Related Proposals:*

*   [gRFC A79][A79]: gRPC Non-Per-Call Metrics Framework

[A79]: A79-non-per-call-metrics-architecture.md

## Proposal

This document proposes exporting the following TCP metrics from gRPC to improve the network debugging capabilities for gRPC users. These metrics will be implemented inside the gRPC endpoint layer. All metrics have the following optional labels:
* network.local.address
* network.local.port
* network.peer.address
* network.peer.port

### Per-Connection Metrics

| Name | Type | Unit | Description |
| ----- | :---- | :---- | :---- |
| grpc.tcp.min\_rtt | Histogram (floating-point) | s | TCP's current estimate of minimum round trip time (RTT). It can be used as an indication of the network health between two endpoints. Corresponds to `tcpi_min_rtt` from `struct tcp_info`. |
| grpc.tcp.delivery\_rate | Histogram (floating-point) | By/s | TCP’s most recent measure of the connection’s "non-app-limited" throughput. The term non-app-limited means that the link is saturated by the application. The delivery rate is only reported when it is non-app-limited.  Corresponds to `tcpi_delivery_rate` from `tcp_info` when `tcpi_delivery_rate_app_limited` is `false`. |
| grpc.tcp.packets\_sent  | Counter (integer) | {packet}  | Total packets sent by TCP including retransmissions and spurious retransmissions.  Corresponds to `tcpi_data_segs_out` from `struct tcp_info`. |
| grpc.tcp.packets\_retransmitted | Counter (integer) | {packet} | Total packets sent by TCP except those sent for the first time. A packet may be retransmitted multiple times and will be counted multiple times as retransmitted. Retransmission counts include spurious retransmissions.  Corresponds to `tcpi_total_retrans` from `struct tcp_info`. |
| grpc.tcp.packets\_spurious\_retransmitted | Counter (integer) | {packet}  | Total packets retransmitted by TCP that were later found to be unnecessary. These packets are acknowledged for the second time or more. Multiple spurious retransmissions for the same packet are counted multiple times.  Corresponds to `tcpi_dsack_dups` from `struct tcp_info`. |
| grpc.tcp.recurring\_retransmits | Counter (integer) | {packet} | The number of times the latest TCP packet ( TCP sequence) was retransmitted due to expiration of TCP retransmission timer (RTO), and not acknowledged at the time the connection was closed.  Corresponds to `tcpi_retransmits` at connection close time. |
| grpc.tcp.bytes\_sent | Counter (integer) | By | Total bytes sent by TCP including retransmissions and spurious retransmissions.  Corresponds to `tcpi_bytes_sent` from `struct tcp_info`. |
| grpc.tcp.bytes\_retransmitted | Counter (integer) | By | Total bytes sent by TCP except those sent for the first time. A byte sequence may be retransmitted multiple times and will be counted multiple times as retransmitted. Retransmission counts include spurious retransmissions.  Corresponds to `tcpi_bytes_retrans` from `struct tcp_info`. |

#### Suggested Metric Collection Algorithm

* Set TCP\_CONNECTION\_METRICS\_RECORD\_INTERVAL to a default of 5 minutes.  
  * Implementations can choose to make it configurable.  
* For each new connected TCP socket, set an initial alarm of 10% to 110% (randomly selected) of TCP\_CONNECTION\_METRICS\_RECORD\_INTERVAL.  
* When the alarm fires \-  
  * Use `getsockopt(TCP_INFO)` or equivalent method to retrieve and record connection metrics.  
  * Re-arm the alarm with 10% to 110% (randomly selected) of TCP\_CONNECTION\_METRICS\_RECORD\_INTERVAL and repeat.  
  * Before the socket is closed, cancel the alarm set above, and retrieve and record connection metrics, providing observability for short-lived connections as well. This will also allow collection of `grpc.tcp.recurring_retransmits`

### Per-Connection Op Metrics

| Name | Type | Unit | Description |
| ----- | :---- | :---- | :---- |
| grpc.tcp.connections\_created | Counter (integer) | {connection} | Number of TCP connections created. |
| grpc.tcp.connection\_count | UpDownCounter (integer) | {connection} | Number of active TCP connections. |
| grpc.tcp.syscall\_writes | Counter (integer) | {syscall} | The number of times we invoked the sendmsg (or sendmmsg) syscall and wrote data to the TCP socket. Measured at the endpoint level. |
| grpc.tcp.write\_size | Histogram (floating-point) | By | The number of bytes offered to each syscall\_write. Measured at the endpoint level. |
| grpc.tcp.syscall\_reads | Counter (integer) | {syscall} | The number of times we invoked the recvmsg (or recvmmsg or zero copy getsockopt) syscall and read data from the TCP socket. Measured at the endpoint level. |
| grpc.tcp.read\_size | Histogram (floating-point) | By | The number of bytes received by each syscall\_read. Measured at the endpoint level. |

#### Suggested Metric Collection Algorithm

* `grpc.tcp.connections_created`  will be incremented when a connection is created.
* `grpc.tcp.connection_count` will be incremented when a connection is created and decremented when it is destroyed.  
* The `grpc.tcp.syscall_writes` and `grpc.tcp.write_size` metrics will be updated whenever we write to the socket.  
* The `grpc.tcp.syscall_reads` and `grpc.tcp.read_size` metrics will be updated whenever we read from the socket.

### Per-Write Metrics

| Name | Type | Unit | Labels | Description |
| ----- | :---- | :---- | :---- | :---- |
| grpc.tcp.sender\_latency | Histogram (floating-point) | s | None | Time taken by the TCP socket to write the first byte of a write onto the NIC. This includes the latency incurred by traffic shaping, qdisc, throttling, and pacing at the sender. Corresponds to the time taken between the final `SCHED` timestamp and the `SENT` timestamp. Sampled periodically. |
| grpc.tcp.transfer\_latency | Histogram (floating-point) | s | size<sup>1</sup> | Time taken to transmit the first size bytes of a write. Transfer latency is measured from when the first byte is handed to the NIC until TCP receives the acknowledgement for the last byte. Corresponds to the time taken between the `SENT` timestamp and the `ACKED` timestamp. Sampled periodically. |

<sup>1</sup> Since transfer latency is strongly affected by the write size, it is broken down into different buckets based on the size of the write. Further, we will only measure the latencies of certain benchmark sample sizes to get measurements that are unaffected by the write sizes. The proposed buckets are:

| Buffered Size | Benchmark Size | `size` (Bytes) |
| :---- | :---- | :---- |
| \[0, 1KiB) | Whole buffer | 1024 |
| \[1KiB, 8KiB) | First 1KiB | 1024 |
| \[8KiB, 64KiB) | First 8KiB | 8196 |
| \[64KiB, 256KiB) | First 64KiB | 65536 |
| \[256KiB, 2MiB) | First 256KiB | 262144 |
| \[2MiB, \+inf) | First 2MiB | 2097152 |

Writes smaller than 1024 Bytes are labelled with `size=1024` to reduce cardinality. Further, their size does not have a big impact on transfer latency since they are able to fit inside a single TCP packet.

#### Suggested Metric Collection Algorithm

* Set TCP\_LATENCY\_RECORD\_PERIOD to a default of 1000 to denote a frequency of 1 per 1000 writes.  
  * Implementations can choose to make it configurable.  
* For each new connected TCP socket,   
  * Set `writes_since_last_latency_measurement_` to a random integer in \[0, TCP\_LATENCY\_RECORD\_PERIOD). Increment this value for every write.  
  * Perform any prerequisites needed for the socket to support timestamping.   
    * For Linux TCP, this involves making a setsockopt(SO\_TIMESTAMPING) call with the flag value set to SOF\_TIMESTAMPING\_SOFTWARE | SOF\_TIMESTAMPING\_OPT\_ID | SOF\_TIMESTAMPING\_OPT\_TSONLY | SOF\_TIMESTAMPING\_OPT\_ID\_TCP | SOF\_TIMESTAMPING\_OPT\_STATS.  
* If `writes_since_last_latency_measurement_` % TCP\_LATENCY\_RECORD\_PERIOD \= 0  
  * Enable latency measurement for the write.   
    * For Linux TCP, this involves splitting the write into two chunks based on the buckets listed above and adding a SO\_TIMESTAMPING cmsg header to the sendmsg call with the flags SOF\_TIMESTAMPING\_TX\_SCHED | SOF\_TIMESTAMPING\_TX\_SOFTWARE on the sampled chunk and SOF\_TIMESTAMPING\_TX\_ACK on the other chunk.   
    * There is only one chunk if the write is less than 1024 bytes, in that case all three flags are set on this chunk.  
  * Set `writes_since_last_latency_measurement_` to 1 and repeat.

### Metric Stability
All metrics added in this proposal will start as experimental. The long term goal will be to de-experimentalize them and potentially have some metrics be on by default, but the exact criteria for that change are TBD. We may also add new labels (eg: `grpc.lb.locality`, `grpc.lb.backend_service`) in the future. This gRFC will be amended when this happens.

### Temporary environment variable protection
This proposal does not include any features enabled via external I/O, so it does not need environment variable protection.

## Implementation

Will be implemented in C-Core to start with. Other languages may implement only a subset of the metrics based on Kernel API availability.
