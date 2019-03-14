Configuring and auto-tuning backpressure thresholds
---------------------------------------------------

* Author(s): David Li
* Approver: a11r
* Status: Draft
* Implemented in:
* Last updated: 2019/03/14
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/2ZUdlq0IKDg

## Abstract

gRPC implementations should auto-tune the backpressure threshold for
streaming RPCs, and expose APIs to configure or provide hints about
this threshold for services with different needs.

## Background

Some gRPC implementations (including Go and Java) will queue data
before writing it to the wire. To prevent unbounded memory usage, they
will only queue up to a certain amount of data before applying
backpressure through language-specific mechanisms. For instance, Go
will block in the stream's `SendMsg` function, while Java will toggle
`CallStreamObserver#isReady()`.

Currently, this threshold is hard-coded in each
implementation[^go-threshold][^java-threshold], and is set to a
relatively small value. This limits throughput for certain
applications, particularly ones dealing with large message payloads,
for which the default threshold would cause backpressure immediately
upon sending any message.

### Related Proposals:

None.

## Proposal

We propose to make this threshold configurable per-RPC, on both the
client and the server. The option would only be a hint; the
implementation may ignore it. The hint would be available both for
clients and servers sending messages on streaming RPCs, and would be
set while handling a call (on the server side) or when initiating a
call (on the client side).

Furthermore, we propose that implementations should automatically tune
the backpressure threshold in response to network conditions and/or
available server resources. Currently, it is unclear how to tune the
value, and so we avoid proposing an exact mechanism.

## Rationale

Making the threshold configurable would immediately benefit
applications sending large amounts of data on a stream, such as
[Apache Arrow Flight][arrow-flight]. It would also benefit
applications that are sensitive to memory usage, and in general let
services make their own tradeoffs between throughput and memory
usage. However, it would also increase the API surface for gRPC
implementations, and naive usage could lead to excessive memory
consumption or reduced throughput.

Additionally, the current backpressure threshold is set per-RPC and
does not account for network conditions. Tuning the value based on
available bandwidth and network delay could improve RPC
throughput. While some implementations already perform BDP probing and
advertise potentially large flow control windows, backpressure
completely ignores this. Furthermore, it could account for all RPCs in
flight, and prioritize certain streams or distribute the backpressure
fairly based on application needs.

## Implementation

For implementing backpressure hints:

Go - On the client side, a call option would be added that sets a
hint. It would then be read in place of the default when [creating the
write quota][go-client-write-quota]. On the server side,
`ServerStream` should have a function to pass a hint down to the
underlying `transport.Stream`, which in turn can adjust the write
quota.

Java - Tracked in [issue #5433][java-5433]. On the client side, a
`CallOption` would be added to set a hint. The options
would be passed into the `ClientStream`, which could directly read the
option. On the server side, a method would be added to the
`ServerCallStreamObserver` interface to provide this hint to the
underlying `ServerCall`, which in turn would provide the hint to the
underlying `ServerStream`.

C Core - [there is currently no such threshold, but the maintainers
are considering
one](https://github.com/grpc/grpc-java/issues/5433#issuecomment-472603484).

We propose to implement this for Java first, and are willing to
contribute the implementation. We may also be willing to contribute an
implementation for the C core.

## Open issues (if applicable)

The exact mechanism for auto-tuning the threshold is unclear. More
experimentation and benchmarking would be needed before starting on a
full implementation, or deciding if a fuller API is needed to
configure a backpressure policy.

[^go-threshold]: https://github.com/grpc/grpc-go/blob/v1.19.x/internal/transport/defaults.go#L43-L46
[^java-threshold]: https://github.com/grpc/grpc-java/blob/v1.19.x/core/src/main/java/io/grpc/internal/AbstractStream.java#L104-L109

[java-5433]: https://github.com/grpc/grpc-java/issues/5433
[go-client-write-quota]: https://github.com/grpc/grpc-go/blob/79c9bc679419ff2f57deccc996e33477a64b6762/internal/transport/http2_client.go#L354
[arrow-flight]: https://arrow.apache.org/blog/2018/10/09/0.11.0-release/
