A105: MAX_CONCURRENT_STREAMS Connection Scaling
----
* Author(s): @markdroth, @dfawley
* Approver: @ejona86
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2025-10-06
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

We propose allowing gRPC clients to automatically establish new
connections to the same endpoint address when it hits the HTTP/2
MAX_CONCURRENT_STREAMS limit for an existing connection.

## Background

HTTP/2 contains a connection-level setting called
[MAX_CONCURRENT_STREAMS][H2MCS] that limits the number of streams a peer
may initiate.  This is often set by servers or proxies to protect against
a single client using too many resources.  However, in an environment with
reverse proxies or virtual IPs, it may be possible to create multiple
connections to the same IP address that lead to different physical
servers, and this can be a feasible way for a client to achieve more
throughput (QPS) without overloading any server.

### Related Proposals: 
* A list of proposals this proposal builds on or supersedes.

[H2MCS]: https://httpwg.org/specs/rfc7540.html#SETTINGS_MAX_CONCURRENT_STREAMS

## Proposal

[A precise statement of the proposed change.]



### Temporary environment variable protection

[Name the environment variable(s) used to enable/disable the feature(s) this proposal introduces and their default(s).  Generally, features that are enabled by I/O should include this type of control until they have passed some testing criteria, which should also be detailed here.  This section may be omitted if there are none.]

## Rationale

[A discussion of alternate approaches and the trade offs, advantages, and disadvantages of the specified approach.]


## Implementation

[A description of the steps in the implementation, who will do them, and when.  If a particular language is going to get the implementation first, this section should list the proposed order.]

## Open issues (if applicable)

[A discussion of issues relating to this proposal for which the author does not know the solution. This section may be omitted if there are none.]
