Keepalive fixed interval
----
* Author(s): Bas Claessen
* Approver: a11r
* Status: Draft
* Implemented in:
* Last updated: 2023-01-23
* Discussion at:

## Abstract

Provide an option to send keepalive PINGs at a fixed interval (boolean default false). This option will be available for the client and server.

## Background

Keepalive PINGs are only sent when the HTTP/2 connection did not receive data for `KEEPALIVE_TIME`.

L7 load balancers may be configured to load balance the HTTP/2 streams inside the HTTP/2 connection to different servers. A L7 load balancer can execute some logic to keep connections open when recieving an HTTP/2 PING. Unexpected connection drops can occur because a keepalive PING is not sent.

A connection drop by the L7 load balancer will for example occur when all conditions below are true:
* 2 HTTP/2 streams of 1 HTTP/2 connection are load balanced to 2 servers
* stream 1 is receiving data frequently enough so no keepalive PINGS are sent
* stream 2 is open and not receiving data

The new option `KEEPALIVE_FIXED_INTERVAL` will send keepalive PINGS at a fixed `KEEPALIVE_TIME`.

### Related Proposals: 
* A8: [Client-side Keepalive](A8-client-side-keepalive.md)
* A9: [Server-side Connection Management](A9-server-side-conn-mgt.md)
## Proposal
An new option `KEEPALIVE_FIXED_INTERVAL` (boolean default false) will be added to the keepalive configuration. This option will be available for the client and server.
When `KEEPALIVE_FIXED_INTERVAL` is true the keepalive PINGs will always be sent at a fixed interval defined by `KEEPALIVE_TIME`.
## Rationale
Sending keepalive PINGs at a fixed interval will likely send more keepalive PINGs than needed.

L7 load balancers could be configured to not load balance HTTP/2 streams but HTTP/2 connections. This is not the most sufficient configuration.

## Implementation
* Java
  * The client options can be specified via `ManagedChannelBuilder.keepAliveFixedInterval`
  * The server options can be specified via `NettyServerBuilder.keepAliveFixedInterval`
  * `io.grpc.internal.KeepAliveManager:onDataReceived` delay ping when state equals `PING_SCHEDULED` and `KEEPALIVE_FIXED_INTERVAL` is false

The implementation in other languages can match the Java implementation.