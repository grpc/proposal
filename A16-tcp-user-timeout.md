TCP User Timeout
----
* Author(s): Yash Tibrewal
* Approver:
* Status: Draft
* Implemented in:
* Last updated: 2018-8-22
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/6VZYCFZpyTI

## Abstract

Provide an option to set the socket option, TCP_USER_TIMEOUT on TCP sockets for
Linux kernels 2.6.37 and later.

## Background

TCP_USER_TIMEOUT is a socket option that has been available on Linux since Linux
2.6.37. As described the Linux manual page, "This option takes an unsigned int
as an argument. When the value is greater than 0, it specifies the maximum
amount of time in milliseconds that transmitted data may remain unacknowledged
before TCP will forcibly close the corresponding connection and return ETIMEDOUT
to the application.  If the option value is specified as 0, TCP will to use the
system default."

At the time this proposal was written, gRPC Core faced an issue where data could
be stuck for a very long time at the gRPC TCP layer. For cases where the network
goes down, and the socket cannot accept more data for writing, an RPC could be
blocked for a very long time. Even if the deadline expired on such an RPC, gRPC
Core was incapable of shutting down the RPC, because the TCP layer would stil
have references to user data. Setting TCP_USER_TIMEOUT to a reasonable value
would put a bound on the maximum time the TCP layer would wait for TCP packets
to be acknowledged, and close the connection if the packets are not acknowledged
in time. This would end up releasing references to user data at the gRPC TCP
layer, thereby allowing the RPC to fail in a timely manner.

### Related Proposals:
N/A

## Proposal

For all Linux platforms running Linux 2.6.37 and later, gRPC will set the socket
option TCP_USER_TIMEOUT for TCP sockets to a default value of 20 seconds. (The
system defaults might be as high as 20 minutes.)

gRPC will also provide a configuration option for both channels and servers to
set the TCP_USER_TIMEOUT value.

## Rationale

This proposal solves two separately seen issues, but with the same root cause
that was described above -
[#15889](https://github.com/grpc/grpc/issues/15889),
[#15871](https://github.com/grpc/grpc/issues/15871).


## Implementation

C-Core - [#16419](https://github.com/grpc/grpc/issues/16419) sets the
TCP_USER_TIMEOUT socket option. A new channel argument,
GRPC_ARG_TCP_USER_TIMEOUT_MS will be introduced to configure the value.

JAVA -

Go -

## Open issues (if applicable)

N/A
