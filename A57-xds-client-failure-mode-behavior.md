A57: XdsClient Failure Mode Behavior
----
* Author(s): Mark D. Roth (@markdroth)
* Approver: @ejona86, @dfawley
* Status: Draft
* Implemented in: 
* Last updated: 2022-10-26
* Discussion at: https://groups.google.com/g/grpc-io/c/rpAl1ZElzZQ

## Abstract

In order to make sure that all gRPC implementations have the same
XdsClient behavior when handling various failure modes, it is necessary
to have a written specification for that behavior.  This document provides
that specification.

## Background

As per [gRFC A27][A27], xDS-aware gRPC clients and servers have an
internal XdsClient interface that is used to manage communication with
xDS servers.  The XdsClient implementation needs to handle a number of
different failure modes:

- The ADS stream is closed.
- The client cannot establish a connection to the xDS server.
- The client subscribes to a resource that does not exist on the server.

Because these failure modes may overlap, the XdsClient must be able to
handle any combination of these failure modes that may occur concurrently.
Proper handling of all of these combinations requires careful
coordination between various events and timers.

Improper handling of these failure modes can result in misleading error
messages and/or seriously incorrect behavior (e.g., treating a resource
as if it does not exist due to a transient network failure, which can
cause gRPC to incorrectly fail data plane RPCs).

This document describes exactly how the XdsClient should behave in the
face of these failure modes.

### Related Proposals: 
* [A27: xDS-Based Global Load Balancing][A27]

[A27]: A27-xds-global-load-balancing.md

## Proposal

Handling of each failure mode is described in its own subsection below.

### Handling ADS Stream Termination

If the ADS stream is closed without ever having received a response from
the server, then the XdsClient should consider that a connectivity
error.  It should log the error and report the error to all watchers of
resources that were subscribed to on that stream.

Note that we do not consider it an error if the ADS stream was closed
after having received a response on the stream.  This is because there
are legitimate reasons why the server may need to close the stream
during normal operations, such as needing to rebalance load or the
underlying connection hitting its max connection age limit (see [gRFC
A9](A9-server-side-conn-mgt.md)).

If the ADS stream fails without having received a response from the
server, the XdsClient must use exponential backoff for each successive
attempt to establish the ADS stream, to avoid overloading the server.
The backoff algorithm should be similar to what gRPC uses for [connection
attempts](https://github.com/grpc/grpc/blob/master/doc/connection-backoff.md).
The backoff state will be reset when an ADS stream attempt does finally
receive a response from the server.

### Handling Connectivity Failure

When the xDS channel reports state `TRANSIENT_FAILURE`, XdsClient should
consider that a connectivity error.  It should log the error and report the
error to all watchers of resources that were subscribed to on that channel.

The reported error message should include the reason for the connectivity
failure, as reported by the xDS channel.  However, there are differences
between gRPC implementations in terms of how they are able to obtain
that information.

#### Connectivity Status Information via Connectivity State Notification

For gRPC implementations that can watch the xDS channel's connectivity state
in a way that directly reports status information when the channel is in
state `TRANSIENT_FAILURE`, the error can be triggered directly based on the
connectivity state notification.  This is the case for the C-core
implementation.

These gRPC implementations should use
[`wait_for_ready`](https://github.com/grpc/grpc/blob/master/doc/wait-for-ready.md)
on the ADS stream, which prevents unnecessarily increasing the backoff
delay after each stream failure.

#### Connectivity Status Information via ADS Stream Failure

If the gRPC implementation cannot get a useful status message directly
from connectivity state monitoring, then it will instead need to wait for
the ADS stream to fail, and report the status message from that stream
failure.  This is the case for the Java and Go implementations.

These gRPC implementations cannot use
[`wait_for_ready`](https://github.com/grpc/grpc/blob/master/doc/wait-for-ready.md)
on the ADS stream, because they are depending on the stream to fail when
the channel is in `TRANSIENT_FAILURE`, in order to obtain the status
message for the connectivity failure.  This has the unfortunate
side-effect of unnecessarily increasing the backoff delay after each
stream failure, which will make the client slower to restart the stream
when connectivity is restored.  However, this is considered an
acceptable trade-off to provide a better error message in the case of
connectivity failure.

Note that even though these gRPC implementations will not trigger a
connectivity error based on the xDS channel's connectivity state
notifications, they must still monitor the xDS channel's connectivity
state, since they will need this information to properly handle the
resource does-not-exist timer described below.

### Handling Resources That Do Not Exist

As per the xDS spec, the SotW protocol variants do not provide any
explicit mechanism for the server to indicate that that the client has
subscribed a resource that does not exist.  Therefore, the client is
expected to assume that the resource does not exist if the server has
not sent the resource within a [15-second
timeout](https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol#knowing-when-a-requested-resource-does-not-exist).

This per-resource timeout requires careful handling in XdsClient, because
it interacts with the timing semantics of the ADS stream.  The timeout
period should start as soon as both of the following are true:

- The first message subscribing to the resource has been sent on the stream.
- The xDS channel is reporting `READY`.

In other words, if the XdsClient starts an ADS stream and sends the initial
subscription message for a resource, but the channel is not yet reporting
`READY`, it should not yet start the timer.  This will occur most frequently
when the XdsClient first starts and the ADS stream cannot actually be sent
until the xDS channel becomes connected.  However, it can also occur if
the channel was initially connected and then becomes disconnected (i.e.,
the channel transitions from `READY` to `IDLE`, since it's using the
`pick_first` LB policy).  In that case, the original ADS stream will fail,
and XdsClient will start a new ADS stream and immediately send the
subscription requests for the resources that it needs from this server,
but because those requests will not actually go out on the wire until
the channel reports `READY`, we don't want to start the timers for those
resources until that happens.

Note that if the XdsClient does start the timers while the xDS channel
is in state `TRANSIENT_FAILURE`, this can cause the does-not-exist
timers to fire while the channel is still trying to become connected,
which can incorrectly cause gRPC to incorrectly fail data plane RPCs.
This is particularly important for gRPC implementations that use
`wait_for_ready` for the ADS stream (such as C-core), since a single ADS
stream may remain pending for a long period of time until the channel
reports `READY`.  However, it is still a problem for gRPC
implementations that do not use `wait_for_ready` for the ADS stream,
since the [default minimum connection attempt
timeout](https://github.com/grpc/grpc/blob/master/doc/connection-backoff.md)
is 20 seconds and the xDS resource does-not-exist timeout is only 15
seconds.

### Temporary environment variable protection

This document is defining how existing functionality should handle
various failure modes, not introducing new functionality, so no environment
variable protection is needed.

## Rationale

We also considered the failure mode of an xDS server that is not
responding at the application layer -- i.e., the ADS stream does not
fail, but the server never sends any response messages.  Unfortunately,
there does not seem to be a good way to detect this case in the xDS SotW
protocol, because the server is not obligated to send any responses if
none of the resources that the client has subscribed to actually exist
on the server, so the client has no way to differentiate between the
server being unresponsive and the resources not existing.

It is worth noting that the xDS incremental protocol does provide a way
for the server to explicitly indicate that a resource that the client
subscribed to does not exist.  Therefore, we may be able to introduce a
timeout to address this failure mode if/when we move to the incremental
protocol variant in the future.

## Implementation

Will be implemented in C-core, Java, Go, and Node.

## Open issues (if applicable)

N/A
