A46: xDS NACK Semantics Improvement
----
* Author(s): Mark D. Roth (markdroth)
* Approver: ejona86, dfawley
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2021-09-03
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

This proposal clarifies xDS NACK semantics used in gRPC.

## Background

The [xDS
spec](https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol)
says that, at the wire protocol level, a NACK is sent on a per-response
basis, not a per-resource basis.  Although not explicitly stated
in the spec, this implies that the client must reject (i.e., not
actually use) even the valid resources from that response, since that
would allow the server to know the client's actual state (although see
[Rationale](#rationale) below).  That is the behavior that gRPC currently
implements.

However, this approach causes problems in a case where a single resource
is invalid and it causes the client to ignore *all* of the resources in
the update, especially for LDS and CDS, where the server must send all
resources in every response.  For example, if there is a single invalid
Cluster resource in a CDS response, all of the Cluster resources will
be rejected.  If this happens on the first CDS response after a client
starts up (i.e., when the client has not already accepted previous
versions of the CDS resources that it can continue to use), that will
cause the client to have no valid CDS resources at all, which means
that the problem will prevent *all* clusters from functioning instead of
affecting only the invalid resource.  This is particularly problematic
now that gRPC shares its XdsClient between channels, because a single
invalid Cluster resource can basically cause all of the client's channels
to stop working all at once.

This behavior makes it challenging for xDS servers to safely deploy
changes in environments in which the clients are not centrally controlled.
For example, older gRPC clients support only the `ROUND_ROBIN`
LB policy, as per [gRFC A27](A27-xds-global-load-balancing.md),
but newer clients now support the `RING_HASH` policy, as per [gRFC
A42](A42-xds-ring-hash-lb-policy.md).  If a Cluster resource specifies
an unsupported LB policy, clients will consider the resource invalid,
which will cause them to NACK the response.  So if an xDS server cannot
be sure that all of its clients have been upgraded to a version that
supports the `RING_HASH` policy, then it cannot safely send a Cluster
resource configuring that policy, because that change would cause older
clients to stop functioning.

Note that this document is not attempting to solve the general problem
that an individual resource that sets a supported field to an unsupported
value will be considered invalid; that behavior is intentional, and there
is nothing we can do to prevent the need for clients to be upgraded to
use a configuration resource that requires features that they do not yet
support.  However, this document *is* intending to solve the problem of
that one invalid resource causing other *valid* resources to be ignored.

This document proposes a behavior change to address this problem.

### Related Proposals: 
* [gRFC A27: xDS-Based Global Load Balancing](A27-xds-global-load-balancing.md)
* [gRFC A42: xDS Ring Hash LB Policy](A42-xds-ring-hash-lb-policy.md)

## Proposal

We will change gRPC's behavior such that when a response is NACKed, gRPC
will still use all valid resources from the response; it will ignore
only the invalid resources.

Note that the xDS wire protocol behavior is not changing at all; the
protocol currently still requires NACKs to be done on a per-response
basis instead of a per-resource basis (although the latter is something
that is expected to be added to the protocol in the future).  The only
change is that the client will actually use the valid resources in the
response.

### Temporary environment variable protection

N/A

## Rationale

Note that gRPC's original behavior was intended to make sure that the
control plane had a clear picture of what configuration the client is
using.  However, it turns out that Envoy currently has cases where it
will apply some valid resources from a response that it NACKs, which
means that the control plane *already* did not have that kind of clear
picture.  To ensure that the control plane will have that kind of clear
picture, there will be subsequent work to extend the xDS protocol to
allow per-resource NACKing instead of per-response NACKing, although
that will be a broader project.

We considered the following alternatives:

- We could have just waited for the xDS protocol changes that will allow
  per-resource NACKing instead of per-response NACKing.  However, that
  is likely to take more work to design and implement in both clients
  and control planes, and we wanted to do something quickly to alleviate
  the problem described above.

- For the case of unknown LB policies specifically, we could fall back
  to `ROUND_ROBIN` instead of considering the resource invalid.
  However, this is not semantically correct behavior; it could cause
  unexpected load on servers and a large number of unnecessary connections
  to backends.  Also, while it would address the specific case that
  triggered this design, it would not address the general case of seeing
  an unsupported value in a supported field.

## Implementation

[A description of the steps in the implementation, who will do them, and when.  If a particular language is going to get the implementation first, this section should list the proposed order.]

## Open issues (if applicable)

N/A
