Load Balancing and DNS
----
* Author(s): Mark D. Roth (roth@google.com)
* Approver: a11r
* Status: Implemented
* Implemented in: C-core (Go and Java in progress)
* Last updated: 2017-09-13
* Discussion at: https://groups.google.com/d/topic/grpc-io/6be1QsHyZkk/discussion

## Abstract

This document describes how additional name service data for load
balancing should be encoded in DNS.

## Background

As described in the [name
resolution](https://github.com/grpc/grpc/blob/master/doc/naming.md) doc,
a resolver implementation is expected to return two additional pieces of
data with each address, both related to [load
balancing](https://github.com/grpc/grpc/blob/master/doc/load-balancing.md).
Those two pieces of data are:

- A boolean indicating whether the address is a backend address (i.e.,
  the address to use to contact the server directly) or a balancer
  address.

- The name of the balancer, if the address is a balancer address.
  This will be used to perform peer authorization.

This document describes how those two pieces of data should be encoded
when the name service is DNS.

### Related Proposals: 

N/A

## Proposal

This proposal makes use of DNS SRV records, as described in
[RFC-2782](https://tools.ietf.org/html/rfc2782).  The SRV record for
grpclb will use the following values:

- service name: `grpclb`
- protocol: `tcp`
- priority: this will be `0` for all grpclb SRV records
- weight: this will be `0` for all grpclb SRV records

When the gRPC client library opens a channel for a given server name,
it will ask DNS for these SRV records in addition to the normal address
records.  For each SRV record that comes back in the response, the
client library will then do a DNS lookup for address records for the name
specified by the SRV record.  It will then return each of the resulting
addresses with the extra fields indicating that the addresses are
balancer addresses and the name from the SRV record.

Since the SRV service name is the same for plain text and encrypted, there is
not a way to provide different ports depending on the presence of encryption. If
simultaneous support for http+https is necessary, we recommend using different
DNS host names for insecure vs secure.

Note that the gRPC library will ignore the values of the `priority` and
`weight` fields.  Instead, it will put all balancer addresses into a
single list (in whatever order the DNS server returns them) and use the
`pick_first` load-balancing policy to decide which one of them to use.
However, in order to leave open the possibility of adding support for
priority and weight in the future, we recommend that these fields be set
to 0.  That way, if we do add support for these fields in the future,
existing records will not break.

### Example

For example, let's say that we have the following DNS zone file:

```
$ORIGIN example.com.
; grpclb for server.example.com goes to lb.example.com:1234
_grpclb._tcp.server IN SRV 0 0 1234 lb
; lb.example.com has 3 IP addresses
lb                  IN A 10.0.0.1
                    IN A 10.0.0.2
                    IN A 10.0.0.3
```

With this data, a gRPC client will resolve the name `server.example.com`
to the following addresses:

```
address=10.0.0.1:1234, is_balancer=true, balancer_name=lb.example.com
address=10.0.0.2:1234, is_balancer=true, balancer_name=lb.example.com
address=10.0.0.3:1234, is_balancer=true, balancer_name=lb.example.com
```

### Future Work: Providing Both Server and Balancer Results

For redundancy, it may be desirable to return both server and balancer
addresses for the same server name.  In principle, this will allow the
client to fall back to directly contacting the servers if the load
balancers are unreachable.  However, that fallback functionality is not
yet implemented, so it will be the subject of future work.

However, it is still possible for service owners to set up their DNS
zone files to publish both types of addresses, so that they are prepared
for an eventual future when this fallback functionality has been
implemented.  For example:

```
$ORIGIN example.com.
server              IN A 10.0.0.11
                    IN A 10.0.0.12
; grpclb for server.example.com goes to lb.example.com:1234
_grpclb._tcp.server IN SRV 0 0 1234 lb
; lb.example.com has 3 IP addresses
lb                  IN A 10.0.0.1
                    IN A 10.0.0.2
                    IN A 10.0.0.3
```

For now, if both address records and SRV records are present for the
same server name, the gRPC client will ignore the address records and
return only the SRV records -- in other words, the result for a lookup
of `server.example.com` will be exactly the same in this case as it
would be in the previous example.

In the future, when we do implement support for this kind of fallback,
this lookup will result in the following addresses:

```
address=10.0.0.11:443, is_balancer=false, balancer_name=<unset>
address=10.0.0.12:443, is_balancer=false, balancer_name=<unset>
address=10.0.0.1:1234, is_balancer=true, balancer_name=lb.example.com
address=10.0.0.2:1234, is_balancer=true, balancer_name=lb.example.com
address=10.0.0.3:1234, is_balancer=true, balancer_name=lb.example.com
```

Note that port 443 is the default if not specified by the server name
passed to the client library by the application.  If the server name
passed in by the application does specify a port, that would be used for
the resulting server addresses, but it would not affect the port used
for the balancer addresses.

## Rationale

We considered one alternative, which was adding additional fields to a
TXT record instead of using an SRV record.  However, the SRV record
seems to be a much better fit for this purpose.

## Implementation

### C-core

In C-core, this implementation will rely on use of the c-ares DNS
library, which was added in
[grpc/grpc#7771](https://github.com/grpc/grpc/pull/7771).

Specifically, we will change the c-ares resolver implementation to
automatically look up the grpclb SRV records for the specified name.  If
found, it will then look up the name(s) that the SRV records point to
and return those as balancer addresses.

Note that, due to platform support issues, we will initially *not*
support the c-ares resolver under Windows or for Node.  Alternatives
will need to be found for these environments.

### Java

Java will depend on JNDI, Netty DNS, dnsjava, or a another DNS library
to do SRV record resolution. The existing name resolver, `DnsNameResolver`,
will be modified to resolve the additional records and include them in
the Attributes presented to the load balancer.

### Go

Go will use the `LookupSRV` function in package `net` to do SRV record
resolution.

## Open issues (if applicable)

N/A
