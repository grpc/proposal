TCP User Timeout
----
* Author(s): Yash Tibrewal
* Approver:
* Status: Draft
* Implemented in:
* Last updated: 2018-8-22
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/6VZYCFZpyTI

## Abstract

Provide an option to set the socket option, TCP_USER_TIMEOUT on TCP sockets for Linux kernels 2.6.37 and later.

## Background

TCP_USER_TIMEOUT is a socket option that has been available on Linux since kernel version 2.6.37. As described the Linux manual page, "This option takes an unsigned int as an argument. When the value is greater than 0, it specifies the maximum amount of time in milliseconds that transmitted data may remain unacknowledged before TCP will forcibly close the corresponding connection and return ETIMEDOUT to the application.  If the option value is specified as 0, TCP will to use the system default."

At the time this proposal was written, gRPC Core faced an issue where data could be stuck for a very long time at the gRPC TCP layer. For cases where the network goes down, and the socket cannot accept more data for writing, an RPC could be blocked for a very long time. Even if the deadline expired on such an RPC, gRPC Core was incapable of shutting down the RPC, because the TCP layer would still have references to user data. Setting TCP_USER_TIMEOUT to a reasonable value would put a bound on the maximum time the TCP layer would wait for TCP packets to be acknowledged, and close the connection if the packets are not acknowledged in time. This would end up releasing references to user data at the gRPC TCP layer, thereby allowing the RPC to fail in a timely manner.

### Related Proposals:
N/A

## Proposal

For all Linux platforms running Linux 2.6.37 and later, gRPC will set the socket option TCP_USER_TIMEOUT for TCP sockets to a default value of 20 seconds. (The system defaults might be as high as 20 minutes.)

gRPC will also provide a configuration option for both channels and servers to set the TCP_USER_TIMEOUT value. This configuration option will have millisecond accuracy and will have the same semantics as the socket option. Setting it to 0 will configure TCP_USER_TIMEOUT with the system default and not the gRPC default of 20 seconds. No minimum value to this option will be enforced by gRPC. It will be the responsibilty of the application to be prudent while setting this option. A very small TCP_USER_TIMEOUT value can affect TCP transmissions over paths with a high latency or packet loss. If the timeout occurs before the acknowledgement arrives, then the connection will be dropped.

## Rationale

This proposal can help mitigate two separately seen issues, but with the same root cause that was described above -
* [#15889](https://github.com/grpc/grpc/issues/15889) - gRPC would potentially block indefinitely and not respect RPC deadlines.
* [#15871](https://github.com/grpc/grpc/issues/15871) - Heavy loads prevent keepalives from destroying the connection.
Note that both issues occurred when a network was lost. The first issue mentioned in #15889 could also be solved by fixing the reference issue. This could be achieved by -
* making a copy of the user data - This would result in a significant user cost
* adding API to gRPC Core's TCP layer to be able to cancel writes for certain streams - This would require gRPC's TCP layer to understand streams and this breaks layers, not to mention the additional bookkeeping and overhead in doing so.

Even by doing one of the two currently known solutions, the issue mentioned in #15871 would still remain, since we require sending the HTTP2 GOAWAY frame before closing the connection in certain scenarios.

Other than solving the two described issues, TCP_USER_TIMEOUT would also provide a fairly reliable way to detect TCP connection breakage without any additional cost.

## Implementation

C-Core - [#16419](https://github.com/grpc/grpc/issues/16419) sets the TCP_USER_TIMEOUT socket option. The same pull request will be modified to add the new channel argument too.
A new channel argument, GRPC_ARG_TCP_USER_TIMEOUT_MS will be introduced to configure the value. The channel argument value will be forwarded as the socket option value.

JAVA - For JAVA, the OkHttp library we use does not support setting such an option. Netty with NIO also does not support this. Netty with epoll/kqueue allows setting this option and the user is already capable of configuring this.

Go -

For non-Linux kernels and Linux kernels before 2.6.37, we do not yet have an alternative that works for all.

## Open issues (if applicable)

N/A
