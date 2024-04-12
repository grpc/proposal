Special case localhost and ip literals for service-config and grpclb-in-DNS
----
* Author(s): apolcyn
* Approver: markdroth
* Status: Draft
* Implemented in: none currently, suited for all languages
* Last updated: 2018-10-29
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/Ln6zeUDoo3M

## Abstract

This document discusses a problem and solution to bring down channel
setup latency for grpc clients that implement
[grpclb-in-DNS](https://github.com/grpc/proposal/blob/master/A5-grpclb-in-dns.md)
and
[service-configs-in-DNS](https://github.com/grpc/proposal/blob/master/A2-service-configs-in-dns.md),
in a special and degenerate case.

## Background

Clients that support grpclb-in-DNS and service-configs-in-DNS must
query for TXT and SRV records of `_grpclb._tcp.<target>` and
`_grpc_config.<target>`, along with the A/AAAA records of
`<target>`, when creating a channel to `<target>`. In the case
that `<target>` exists in the machines "hosts" file and
grpclb-in-DNS/service-configs-in-DNS are not set up for `<target>` (common
when the target is "localhost" or an ipv4/v6 literal),
channel setup can take much longer than desired due to the need
to make these TXT and SRV lookups over the network and then
wait for NXDOMAIN responses. This is particularly problematic
for automated tests, which commonly create large numbers of client
channels having targets of "localhost".

While clients which don't implement grpclb-in-DNS or
service-configs-in-DNS potentially only need to do a "hosts" file
lookup when the target is "localhost", clients which do implement
them add `lookup -> NXDOMAIN` RTT to the setup of every channel.

Another problem which is unrelated to latency but which also caused
by SRV and TXT record lookups is related to the way in which
Java treats DNS resolution failures. Internally, gRPC-Java raises a type
of fatal error when a DNS query fails. Normally, gRPC-Java internally
catches these types of errors and carries on, but there are certain
automated test environments in which the Java runtime is adjusted to
crash upon seeing these types of errors. Normally, A/AAAA queries
always succeed in these types of test environments because it's
unexpected for the client to target anything other than "localhost", but
SRV and TXT record lookups change things so that clients in these
test environments almost always see failed DNS queries, and so they
present a problem.

## Proposal

This gRFC proposes that gRPC clients special case "localhost" and
ipv4/v6 literals (e.g. "1.2.3.4" and "::1"). That is, this gRFC
proposes that gRPC clients should not query for SRV or TXT
records when the target is "localhost" or an ipv4/v6 literal,
and thus that gRPC clients should not implement grpclb-in-DNS or
service-configs-in-DNS for such targets.

## Rationale

The latency problem is likely to be noticeable
only for "localhost" and ipv4/ipv6 literal targets, because grpc clients
connecting to targets which are not in a machine's "hosts" file and/or parseable as
an ip literal need to reach out to DNS servers and wait for A/AAAA lookup time anyways
(unless the grpc client is using a resolver that does client-side caching). This
problem could perhaps be fixed by implementing DNS caches within grpc clients, but
that would be more complex.

The problem related to Java error handling in certain environments only practically
affects gRPC-Java clients which target "localhost" or ip literals, and so this
problem only needs to be dealt with for "localhost" and ip literals.

Also note that while special-casing "localhost" seems wrong in principle
because its special status is really a convention rather than an
inherent property of the DNS protocol, the convention here is so strong
that anyone violating it is likely to have many other problems, and we
therefore think it's reasonable to have our code treat it specially.

## Implementation

This should be done for all languages which implement
grpclb-in-DNS and/or service-configs-in-DNS.
