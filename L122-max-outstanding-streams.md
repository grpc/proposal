Title
----
* Author(s): Siddharth Nohria <siddharthnohria>
* Approver: Craig Tiller <ctiller>
* Status: Draft
* Implemented in: C++
* Last updated: 2025-05-12
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

Allow servers to set a server-wide `MAX_CONCURRENT_STREAMS` as part of the
Resource Quota, in addition to the current per-connection `MAX_CONCURRENT_STREAMS`.

## Background

HTTP2 protocol provides the `MAX_CONCURRENT_STREAMS` setting as a way to limit
the number of concurrently open streams on any single connection between a
client and the server.

But this in itself is not good enough to protect the servers from too many
concurrent streams; a client can just create create more connections to the same
server, and be allowed to effectively send more concurrent streams.

In addition to the overload risk due too many connections, another limitation of
a per-connection `MAX_CONCURRENT_STREAMS` is that it is static in nature. For
servers where the number of clients might be dynamic, this is sub-optimal. When
there are only a few clients connected to the server, the server might be able
to server more concurrent requests from each of these clients. So a
per-connection limit set to deal with overload situations (with 1000's of clients)
might be sub-optimal in situations with only a small number of clients.

## Proposal

Introduce `StreamQuota` as part of the `ResourceQuota` on the server side. Users
can set the `max_outstanding_streams` in the `StreamQuota`, which will be equally
distributed among all open connections to the server. If the user also specifies
a per-connection `MAX_CONCURRENT_STREAMS`, that will also be respected as an
upper bound.

## Implementation

Implemented in grpc/grpc#39125.
