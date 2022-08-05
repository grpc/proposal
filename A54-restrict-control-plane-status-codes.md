A54: Restrict Control-Plane Status Codes
----
* Author(s): [Eric Anderson](https://github.com/ejona86), [Mark
  Roth](https://github.com/markdroth), [Doug Fawley](https://github.com/dfawley)
* Approver: [Doug Fawley](https://github.com/dfawley)
* Status: In Review
* Implemented in: <language, ...>
* Last updated: 2022-07-22
* Discussion at: https://groups.google.com/g/grpc-io/c/Yp0X2IHiVlA/m/7QyU4mYKDgAJ

## Abstract

The gRPC control plane has no need to use many of the status codes yet
communicates with external services whose status codes may accidentally be
copied to the data plane. This gRFC restricts the allowed status codes the
control plane can use when failing an RPC, as a safeguard against bugs.

## Background

gRPC has a control plane consisting mainly of name resolvers and load balancers.
Separately it has a data plane that processes individual RPCs. Error information
from the control plane needs to pass to the data plane when errors are
occurring, but the gRPC data plane is restricted from generating [status codes
reserved for application use][statuscodes.md]: `INVALID_ARGUMENT`, `NOT_FOUND`,
`ALREADY_EXISTS`, `FAILED_PRECONDITION`, `ABORTED`, `OUT_OF_RANGE`, and
`DATA_LOSS`. Those status codes are reserved for each service.

Earlier in gRPC's life, there was relatively low risk of using a reserved status
code in the data plane. There was a relatively fixed amount of control plane
code, and much of it wasn't accessing gRPC-based services (e.g., DNS,
pick_first, round_robin). At that point in time, a simple audit could reveal
broken status code usages.

Now, however, it is more common to implement custom name resolvers and load
balancers and for those components to use gRPC themselves. This happened first
with gRPC-LB, but also happens with xDS and others. It is now harder to audit
all the code running in the control plane and it is very hard to find status
"leaks" from a remote control plane server into the data plane. This is a
serious risk as control plane errors can't be copied verbatim to the data plane;
they need to be reinterpreted for the new context (generally by becoming
`UNAVAILABLE` and attaching additional context).

As an example, if a name resolver is looking up the host `example.com`, that
could be implemented as a gRPC call to a service. That service might fail the
RPC with `NOT_FOUND` because the host is not registered. But if the data plane
has an RPC reading a file `funny-jokes.txt`, the RPC should fail with
`UNAVAILABLE`. Failing the RPC with `NOT_FOUND` would confuse the application
into believing `funny-jokes.txt` doesn't exist. This is the situation that
caused a serious [Spotify outage][] because of [a bug in gRPC Java][issue 8950].

[statuscodes.md]: https://github.com/grpc/grpc/blob/master/doc/statuscodes.md
[Spotify outage]: https://engineering.atspotify.com/2022/03/incident-report-spotify-outage-on-march-8/
[issue 8950]: https://github.com/grpc/grpc-java/issues/8950

## Related Proposals

 * [A6: gRPC Retry Design][gRFC A6]
 * [A31: gRPC xDS Timeout Support and Config Selector Design][gRFC A31]

[gRFC A6]: A6-client-retries.md
[gRFC A31]: A31-xds-timeout-support-and-config-selector.md

## Proposal

Add checks to each language's Channel that prevents the control plane from
passing "inappropriate statuses" to the data plane. Specifically:

 * When applying a failed Service Config result from the name resolver
 * When applying a failed Config Selector result from the name resolver (an
   internal API for xDS in [gRFC A31][])
 * When applying a failed Picker result from the load balancer

The check locations are precise and apply only to the places where the control
plane passes status codes to the data plane. At the time of this gRFC, this list
is exhaustive. Future gRFCs should add similar requirements in the case they add
an API that passes status codes from the control plane to the data plane.

"Inappropriate statuses" are statuses whose code is one of:

 * `OK` (in some APIs this is okay, but it must not be copied to an RPC)
 * `INVALID_ARGUMENT`
 * `NOT_FOUND`,
 * `ALREADY_EXISTS`
 * `FAILED_PRECONDITION`
 * `ABORTED`
 * `OUT_OF_RANGE`
 * `DATA_LOSS`

If an "inappropriate status" is detected, the status should be rewritten to use
`INTERNAL` status code and include text within the status message making it
clear a bug occurred. The original status code and message should be included
within the new status message so the error information is not lost.

If a failed Picker result occurs, retry policy status code matching may only
occur after the control plane status check and rewrite. However, if an
inappropriate status is rewritten and the retry policy allows retrying
`INTERNAL`, implementations are free to not retry the RPC. The rewrite does not
impact transparent retries as transparent retries are not performed for pick
failures. Similarly, retries are not impacted by rewriting failed Service Config
and Config Selector results as those results must be successful to determine the
retry policy.

### Temporary environment variable protection

No temporary environment variable protection will be used.

## Rationale

This gRFC does not make it "okay" for a name resolver or load balancer to
propagate status codes verbatum from the control plane. Such behavior would
still be considered a bug and the `INTERNAL` error code will cause such bugs to
fail noticably. The point of the gRFC is to reduce the damage to the system when
these bugs are triggered. This does not improve reliability directly, but it
does ease recovery.

This gRFC purposefully does not limit interceptors, even interceptors that
communicate with a control plane (e.g., [RLQS][rlqs.proto]). An interceptor like
[fault injection][gRFC A33] legitimately needs to fail RPCs with any status
code. There's no API-distinguishing feature between interceptors that
communicate with a control plane and those that do not, so we simply accept the
risk, especially since interceptors communicating with a remote control plane
are relatively rare. Because interceptors remain unrestricted, we are not aware
of any existing code that will be impacted by this gRFC.

"Inappropriate status" codes is an easy topic for debate. The listed codes are
clearly inappropriate, but there are other codes that may also seem
inappropriate, e.g., `UNIMPLEMENTED` or "anything other than `UNAVAILABLE`."
Getting agreement on additional status codes is difficult as it opens up very
detailed questions about "appropriate" load balancer behavior and what
hypothetical situations a particular status code may need to be used. A future
gRFC may restrict additional status codes, but such codes are not included in
this gRFC for expediency and because further debate provides diminishing
returns.

Simarly, using `INTERNAL` code for failures is an easy topic for debate.
`UNAVAILABLE` and `UNKNOWN` are other easy contenders. Any of these is probably
"acceptable." There are some behavior differences between the three, with
`INTERNAL` having some advantages. But the difference is small enough that the
decision can safely be seen as arbitrary and decided by fair coin toss (flip
twice, redo if heads both times). The status is only used if there is another
bug, so it should rarely be seen. Although the decision is arbitrary, the
expected behavior is defined in this gRFC and implementatiors should not choose
a different status code.

[rlqs.proto]: https://github.com/envoyproxy/envoy/blob/2222039d775c45c583f5eef833a482e2be150d21/api/envoy/service/rate_limit_quota/v3/rlqs.proto
[gRFC A33]: A33-Fault-Injection.md

## Implementation

Each language will implement this independently. The gRFC has very low
implementation burden so should be implemented as soon as feasible.
