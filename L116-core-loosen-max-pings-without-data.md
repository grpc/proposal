## L116: C++-Core: Loosen behavior of `GRPC_ARG_HTTP2_MAX_PINGS_WITHOUT_DATA`

*   Author: Yash Tibrewal (@yashykt)
*   Approver: Mark Roth (@markdroth), Craig Tiller (@ctiller)
*   Status: Draft
*   Implemented in:
*   Last updated: 2024-04-25
*   Discussion at: https://groups.google.com/g/grpc-io/c/X_iApPGvwmo

## Abstract

Modify the behavior of `GRPC_ARG_HTTP2_MAX_PINGS_WITHOUT_DATA` to throttle pings
to a frequency of 1 minute instead of completely blocking pings when too many
pings have been sent without data/header frames being sent.

## Background

[A8: Client-side Keepalive](A8-client-side-keepalive.md) and
[A9: Server-side Connection Management](A9-server-side-conn-mgt.md) designed the
use of HTTP2 pings as a keepalive mechanism in gRPC. In gRPC C++-Core, the
following channel arguments implement keepalives. (Refer
[Keepalive User Guide for gRPC Core](https://github.com/grpc/grpc/blob/master/doc/keepalive.md)
for details.) -

*   `GRPC_ARG_KEEPALIVE_TIME_MS` - Period after which a keepalive ping is sent.
*   `GRPC_ARG_KEEPALIVE_TIMEOUT_MS` - Period after which sender of keepalive
    pings closes transport if it does not receive an acknowledgement of the
    ping.
*   `GRPC_ARG_KEEPALIVE_PERMIT_WITHOUT_CALLS` - Allows keepalive pings to be
    sent even if there are no calls in flight.
*   `GRPC_ARG_HTTP2_MIN_RECV_PING_INTERVAL_WITHOUT_DATA_MS` - Minimum allowed
    time between a server receiving successive ping frames without sending
    data/header frames.
*   `GRPC_ARG_HTTP2_MAX_PING_STRIKES` - Number of bad pings server will tolerate
    before closing the connection.

In addition to these channel arguments, the following channel arguments were
introduced in gRPC C++-Core to play nicely with proxies that break connections
that send too many pings.

*   `GRPC_ARG_HTTP2_MIN_SENT_PING_INTERVAL_WITHOUT_DATA_MS` - Minimum time
    sender of a ping would wait between consecutive ping frames without
    receiving a data/header frame.
*   `GRPC_ARG_HTTP2_MAX_PINGS_WITHOUT_DATA` - Maximum number of pings that can
    be sent without a data/header frame.

These two additional channel arguments have historically caused a lot of pain
and confusion among users of gRPC C++-Core and dependent languages when
configuring keepalives. For example, long-lived streams with sparse
communication get affected by these channel arguments and pings are blocked
completely, defeating the purpose of keepalives. To help with this,
[grpc/grpc#24063](https://github.com/grpc/grpc/pull/24063) deprecated
`GRPC_ARG_HTTP2_MIN_SENT_PING_INTERVAL_WITHOUT_DATA_MS`.

### Related Proposals:

*   [A8: Client-side Keepalive](A8-client-side-keepalive.md)
*   [A9: Server-side Connection Management](A9-server-side-conn-mgt.md)

## Proposal

Modify the behavior of `GRPC_ARG_HTTP2_MAX_PINGS_WITHOUT_DATA` to throttle pings
to a frequency of 1 minute instead of completely blocking pings when too many
pings have been sent without data/header frames being sent.

The default setting of `GRPC_ARG_HTTP2_MAX_PINGS_WITHOUT_DATA` is 2, and as
stated earlier, long-lived streams with sparse communication get blocked by this
channel arg if not explicitly disabled by setting it to 0. By loosening the
behavior to throttle the pings instead, these use-cases would be less impacted.

### Temporary experiment protection

The C++-Core experiment "max_pings_wo_data_throttle" will be used to guard this
change in behavior.

## Rationale

Ideally, we would be able to deprecate `GRPC_ARG_HTTP2_MAX_PINGS_WITHOUT_DATA`
similar to `GRPC_ARG_HTTP2_MIN_SENT_PING_INTERVAL_WITHOUT_DATA_MS` and remove
the pain completely. Unfortunately, this would break current users that go
through proxies that limit how often a ping can be sent.

If there's user interest and if the behavior of
`GRPC_ARG_HTTP2_MAX_PINGS_WITHOUT_DATA` is still considered useful, we could
consider adding a channel argument that controls how long pings are throttled.

## Implementation

Implemented in [grpc/grpc#36374](https://github.com/grpc/grpc/pull/36374).
