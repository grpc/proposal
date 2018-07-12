Client-side Keepalive
----
* Author(s): [Eric Anderson](https://github.com/ejona86)
* Approver: a11r
* Status: Implemented
* Implemented in: C, Java, Go
* Last updated: 2017-03-31
* Discussion at:
  https://groups.google.com/d/topic/grpc-io/Yc5_vSUJgwQ/discussion

## Abstract

Mobile has very poor network connectivity and TCP connection failures are common
(commonly due to NATs). However, in the absence of writes the OS will not detect
such failures and in the presence of writes detection can take many minutes.

L4 proxies may be configured to disconnect "idle" connections. Google Cloud load
balancers disconnect apparently-idle connections after [10
minutes](https://cloud.google.com/compute/docs/troubleshooting#communicatewithinternet).
AWS ELB disconnects after [60
seconds](http://aws.amazon.com/articles/1636185810492479).

gRPC supports long-lived streams. A simple use-case is a long-lived stream where
the server notifies clients of events, as an alternative to long polling. Thus a
connection being idle on the network doesn't imply that no RPCs are outstanding.

We want gRPC to be reliable in these situations, which necessitates some form of
keepalive mechanism. We want to support avoiding connection breaks and detecting
them when the occur. But it is important to minimize accidental DDoS risk.

## Background

The problem exists for both clients and servers. Since the failures involve the
connection between client and server being severed, it needs to be handled
on each side. Because DDoS risk is limited for a server-side keepalive,
server-side can have much higher latency when detecting dead connections, and
server-side is more centrally configured, it is considered sufficiently
different enough to have a separate design and is not discussed here further.

Because there may be many clients maintaining connections open to servers it is
very easy to DDoS yourself with keepalives, or simply waste a lot of network and
CPU. Although a particular keepalive is small and trivial to respond to, if you
have 1 million phones sending a PING every 10 seconds, that is 100 thousand QPS
for no work. Thus this feature should be used conservatively and in
concert/deference to other more scalable methods (e.g., configuring the TCP
connection to be closed on idle).

There is a related but separate concern called "health checking." It tends to
exist at a higher level and is usually whether a _service_ is healthy (vs a
specific hop-by-hop connection). As such, keepalive and health checking have
different failure models and are independent (keepalive failing implies nothing
about health checking, and vise versa). However, in some common failure cases
the two are coorelated.

To be scalable, health checking should generally be centralized and interact
with the name resolution or load balancing systems. However, such systems take
time to develop and deploy and are generally unavailable on "day 1." Thus, this
keepalive design includes support for being used as a "poor man's health
checking." It should generally be viewed as a short-term solution, however.

### Design Background

[TCP keepalive](https://en.wikipedia.org/wiki/Keepalive#TCP_keepalive) is a
well-known method of maintaining connections and detecting broken connections.
It is disabled by default, but when enabled after an implementation-specific
duration of inactivity (on the order of 1-2 hours, but sysadmin configurable)
will begin sending redundant packets waiting for their ACK. If ACK, then
connection seems good. If no ACK after repeated attempts, the connection is
deemed broken. Configuration of its variables is provided by the OS on a
per-socket level, but is commonly not exposed to higher-level APIs.

TCP keepalive has three parameters to tune:
 1. time (time since last receipt before sending a keepalive),
 2. interval (interval between keepalives when not receiving reply), and
 3. retry (number of times to retry sending keepalives).

gRPC uses HTTP/2 which provides a mandatory
[PING](http://httpwg.org/specs/rfc7540.html#PING) frame that causes the receiver
to immediately reply with a PING ACK. There are no semantics to PING other than
the need to reply promptly. It can be used to estimate round-trip time,
bandwidth-delay product, or test the connection. Chrome uses them for similar
reasons as described here, but we are not aware of their precise usage. In
passing I've heard they "sent a PING every 45 seconds." [Some
documentation](https://insouciant.org/tech/connection-management-in-chromium/#blackholed_tcp)
suggests it may be more nuanced than that. Although PING support is mandatory in
HTTP/2, it may be subject to abuse restrictions.

While gRPC implementations have tight control of the HTTP/2 stack, this doesn't
mean it isn't necessary to interoperate with other implementations. When
considering the design, recognize the keepalive may be happening between a gRPC
client and a generic HTTP/2 proxy.

### Related Proposals:
* A9: [Server-side Connection Management](A9-server-side-conn-mgt.md)

## Proposal

To be clear, the design does not require service owners support keepalive.
**client authors must coordinate with service owners** for whether a particular
client-side setting is acceptable. Service owners decide what they are willing
to support, including whether they are willing to receive keepalives at all.

### Basic Keepalive

Implement an application-level keepalive conceptually based on TCP keepalive,
using HTTP/2's PING. Interval and retry don't quite apply to PING because the
transport is reliable, so they will be replaced with timeout (equivalent to
interval * retry), the time between sending a PING and not receiving any bytes
to declare the connection dead.

Doing some form of keepalive is relatively straightforward. But avoiding DDoS is
not as easy. Thus, avoiding DDoS is the most important part of the design. To
mitigate DDoS the design:

* Disables keepalive for HTTP/2 connections with no outstanding streams, and
* Suggests for clients to avoid configuring their keepalive much below one
  minute (see Server Enforcement section for additional details)

Most RPCs are unary with quick replies, so keepalive is less likely to be
triggered. It would primarily be triggered when there is a long-lived RPC.

Since keepalive is not occurring on HTTP/2 connections without any streams,
there will be a higher chance of failure for new RPCs following a long period of
inactivity. To reduce the tail latency for these RPCs, it is important to not
reset the _keepalive time_ when a connection becomes active; if a new stream is
created and there has been greater than 'keepalive time' since the last read
byte, then a keepalive PING should be sent (ideally before the HEADERS frame).
Doing so detects the broken connection with a latency of _keepalive timeout_
instead of _keepalive time + timeout_.

_keepalive time_ is ideally measured from the time of the last byte read.
However, simplistic implementations may choose to measure from the time of the
last keepalive PING ack (a.k.a., polling). Such implementations should take
extra precautions to avoid issues due to latency added by outbound buffers, such
as limiting the outbound buffer size and using a larger _keepalive timeout_.
Implementations must not measure from the last keepalive PING sent, to avoid
triggering abuse detection on servers that are configured with settings that
exactly match behavior of the client.

As an optional optimization, when _keepalive timeout_ is exceeded, don't kill
the connection. Instead, start a new connection. If the new connection becomes
ready and the old connection still hasn't received any bytes, then kill the old
connection. If the old connection wins the race, then kill the new connection
mid-startup.

The _keepalive time_ and _keepalive timeout_ are expected to be
application-configurable options, with at least second precision. Applications
should ensure _keepalive timeout_ is at least multiple times the round-trip time
to allow for lost packets and TCP retransmits. It may also need to be higher to
account for long garbage collector pauses. Since networks are not static,
implementations are permitted to adjust the timeout based on network latency.

When a client receives a GOAWAY with error code ENHANCE\_YOUR\_CALM and debug
data equal to ASCII "too\_many\_pings", it should log the occurrence at a log
level that is enabled by default and double the configure KEEPALIVE\_TIME used
for new connections on that channel.

### Extending for Basic Health Checking

Services failures are more frequent than the TCP failures, so detection delay
needs to be decreased in order to reduce number of RPCs affected by undiscovered
failures.

Thus we make these from the basic keepalive:
* ~~Disables keepalive for HTTP/2 connections with no outstanding streams~~, and
* ~~Suggests~~ Restricts clients to avoid configuring their keepalive below ~~one
  minute~~ ten seconds

That is, "keepalive" may continue even without any outstanding streams and the
minimum is reduced to 10 seconds but also enforced on client-side.

Clients will have three channel settings for configuration:
* `KEEPALIVE_TIME`, defaulting to `infinite`. If smaller than ten seconds, ten
  seconds will be used instead.
* `KEEPALIVE_TIMEOUT`, defaulting to `20 seconds`
* `KEEPALIVE_WITHOUT_CALLS`, defaulting to `false`

### Server Enforcement

Servers need to respond to misbehaving clients by sending GOAWAY with error
code ENHANCE\_YOUR\_CALM and additional debug data of ASCII "too\_many\_pings"
followed by immediately closing the connection. Immediately closing the
connection fails any in-progress RPCs which increases the chance of the client
author detecting the misconfiguration.

Servers will have two settings for enforcement:

* `PERMIT_KEEPALIVE_TIME`, defaulting to `5 minutes`
* `PERMIT_KEEPALIVE_WITHOUT_CALLS`, defaulting to `false`

PINGs have other uses than keepalive, like measuring latency and network
bandwidth, thus servers should permit receiving a PING after _sending_ HEADERS
and DATA frames and generally support some "fuzziness" to their usage.

Suggested algorithm:

* Initialize `MAX_PING_STRIKES = 2; ping_strikes = 0`
* Initialize `last_valid_ping_time = epoch`
  * The clock used need not be precise; one second is enough precision,
    although millisecond precision would be encouraged as future-proofing. But
    it should be accurate for measuring durations; a monotonic clock should be
    used
* When a PING frame is *received*:
  * If `active_streams == 0 && PERMIT_KEEPALIVE_WITHOUT_CALLS == false`, verify
    `last_valid_ping_time + 2 hours <= now()`
  * Otherwise verify `last_valid_ping_time + PERMIT_KEEPALIVE_TIME <= now()`
  * If either verification failed, ping\_strikes++
    * if `ping_strikes > MAX_PING_STRIKES`, client is misbehaving
  * otherwise `last_valid_ping_time = now()`
* When a HEADERS or DATA frame is *sent*, set `last_valid_ping_time = epoch;
  ping_strikes = 0`

The "2 hours" restricts the number of PINGS to an implementation equivalent to
TCP Keep-Alive, whose interval is specified to default to [no less than two
hours](https://tools.ietf.org/html/rfc1122#page-101).

To allow changing clients to be more aggressive in the future, server responses
should include the Server header (analogous to User-Agent) that includes its
version. (TODO: replace with SETTINGS-based version)

## Rationale

TCP keepalive is hard to configure in Java and Go. Enabling is easy, but one
hour is far too infrequent to be useful; an application-level keepalive seems
beneficial for configuration.

TCP keepalive is active even if there are no open streams. This wastes a
substantial amount of battery on mobile; an application-level keepalive seems
beneficial for optimization.

Application-level keepalive implies HTTP/2 PING.

Supporting health checking unfortunately complicates the design noticeably.
Without health checking it would be feasible to ignore server-side enforcement.
But it is needed by enough users (even temporarily) to appear to be
worth-while.

There are no known generally available methods usable to gRPC for detecting
abusive usage of TCP keepalive.

## Implementation

* C. TODO get current progress; close, if not done. Being done by @y-zeng
  * Client options, mapping from the option name in this spec to the C core
    option name:
    * `KEEPALIVE_TIME`: `GRPC_ARG_KEEPALIVE_TIME_MS`
    * `KEEPALIVE_TIMEOUT`: `GRPC_ARG_KEEPALIVE_TIMEOUT_MS`
    * `KEEPALIVE_WITHOUT_CALLS`: `GRPC_ARG_KEEPALIVE_PERMIT_WITHOUT_CALLS`
  * Server options, mapping from the option name in this spec to the C core
    option name:
    * `PERMIT_KEEPALIVE_TIME`:
      `GRPC_ARG_HTTP2_MIN_RECV_PING_INTERVAL_WITHOUT_DATA_MS`
    * `PERMIT_KEEPALIVE_WITHOUT_CALLS`:
      `GRPC_ARG_KEEPALIVE_PERMIT_WITHOUT_CALLS`
* Java. Basic Keepalive has been available in OkHttp transport since 1.0 by
  @zsurocking in grpc/grpc-java#1992. Full spec completed in OkHttp and Netty
  since grpc-java v1.3.0 or grpc/grpc-java@393ebf7c
  * The client options can be specified via
    `ManagedChannelBuilder.keepAliveTime()`, `keepAliveTimeout()`, and
    `keepAliveWithoutCalls()`.
  * The server options can be specified via
    `NettyServerBuilder.permitKeepAliveTime()` and
    `permitKeepAliveWithoutCalls()`.
* Go. TODO get current progress; close, if not done. Being done by @MakMukhi
