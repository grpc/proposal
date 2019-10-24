A26: gRPCLB Selection
----
* Author(s): Doug Fawley
* Approver: a11r
* Status: Draft
* Implemented in: None
* Last updated: 2019-10-22
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

Modify the method for enabling gRPCLB to not require any special handling in
core, allowing for stabilization of the gRPCLB protocol and load balancing
policy.

## Background

The existing mechanism to enable and select gRPCLB is currently still
experimental. In addition, gRPCLB itself is deprecated, to be replaced by xDS as
the official advanced load balancing protocol for gRPC ([announcement
email](https://groups.google.com/d/msg/grpc-io/0yGihF-EFQo/A4QKdXffBwAJ)). We
would like to enable production uses of gRPCLB, but cannot do so if gRPCLB
support is not stable. Therefore, we need a mechanism for selecting gRPCLB that
can be declared stable, yet not create long-lasting maintenance problems.

## Proposal

C core will promote its existing channel arg that configures the DNS resolver to
fetch SRV records to "stable".

In Go and Java, the default DNS resolver will not fetch SRV records. However,
when gRPCLB is imported, it will override the default DNS resolver with a
resolver that does fetch SRV records. Justification for this approach: without
the gRPCLB LB policy, balancer addresses have no value. Users that do not wish
to use gRPCLB will not take the performance penalty caused by attempting to
resolve the balancer addresses through SRV records.

Additionally, in all languages, to remove special handling of gRPCLB from the
channel:
- The loadBalancingConfig field in service config will be the only way to
  specify the "grpclb" policy.
- Balancer addresses will now only be provided via side-channel data passed from
  the resolver to the balancer (similar to the xDS client instance).

## Rationale

Several other options were considered, among two categories, some of which
relied upon the previous mechanism for selecting gRPCLB:

1. The DNS resolver producing gRPCLB balancer addresses via SRV records
1. The service config selecting gRPCLB, and the gRPCLB policy resolving balancer
   addresses.

### DNS resolver produces balancer addresses via SRV records

These alternative proposals would allow the DNS resolver to be configured to
fetch SRV records optionally:

1. Use a channel option to enable SRV record lookups in the DNS resolver.
   - Con: It could be considered problematic to configure the scheme using
     client-side code in a fashion necessary for correctness, as the target
     string alone is intended to convey enough information to allow connections
     to be made.
1. Use a new URI scheme to indicate "DNS+SRV".
   - Con: Confusing for users. Go's default DNS resolver currently fetches SRV
     records and may need a "DNS-SRV" scheme.
1. Use a URI query parameter to enable SRV record lookups in the DNS resolver.
   - Cons: Confusing for users. Supporting query parameters may require parsing
     changes in some / all gRPC languages. If an older version of gRPC's DNS
     resolver ignores query parameters, this would silently not work.

### Service config selects gRPCLB; balancer policy resolves balancer addresses

For most custom load balancing policies, gRPC uses the loadBalancingConfig field
in the Service Config to select and configure the LB policy. These options use
the same mechanism to select and configure gRPCLB, as with the chosen proposal
above. However, in these cases, the gRPCLB balancer policy would be responsible
for resolving balancer addresses. There are three options for how to convey the
necessary information to the balancer in the GrpcLbConfig:

1. Encode a flag to enable/disable SRV record lookups.
   - Con: This method would still require another way to pass gRPCLB balancer
     addresses from the name resolver, for non-DNS-based systems.
1. Encode the load balancer name as an address resolved via DNS A/AAAA records.
   - Con: this method would still require another way to pass gRPCLB balancer
     addresses from the name resolver, for non-DNS-based systems.
1. Encode the load balancer as a target string.
   - As opposed to the above two options, this would allow users to implement
     custom grpclb target resolution logic.  DNS-based systems would use
     `srv:///_grpclb._tcp.hostname`.
   - Con: It may be difficult to determine the difference between SRV lookups
     failing and the A/AAAA lookup of the resulting address failing, which may
     affect desired fallback behavior.

## Implementation

### Go

Go will be updated to support both the legacy method of gRPCLB selection (via
balancer addresses produced by the resolver via the Address list), and this new
method.  This will be implemented by:

1. Add a new "attributes" field to
   [resolver.State](https://godoc.org/google.golang.org/grpc/resolver#State),
   allowing arbitrary key/value pairs to be attached to the resolver state.
1. Declare a new package to contain gRPCLB types; declare a new type in that
   package to indicate gRPCLB balancer addresses.  This package is necessary to
   avoid a circular dependency (grpc->dns->grpclb->grpc).
1. Pass balancer addresses in this type as an attribute in `resolver.State` from
   the DNS resolver.
1. Update gRPCLB to recognize this attribute.  If detected, ignore any balancer
   addresses in the address list, and use the new attribute instead.

### Java

Java will remove the legacy method of gRPCLB selection (via balancer addresses
produced by the resolver via the Address list). This was internal-only
functionality already, as it required using
`io.grpc.internal.GrpcAttributes.ATTR_LB_ADDR_AUTHORITY`.

The new approach will be implemented by:

1. Move `io.grpc.internal.GrpcAttributes.ATTR_LB_ADDR_AUTHORITY` to
   `io.grpc.grpclb.GrpclbConstants.ATTR_LB_ADDR_AUTHORITY`.
2. Add `io.grpc.grpclb.GrpclbConstants.ATTR_LB_ADDRS` with type
   `Attributes.Key<List<EquivalentAddressGroup>>`. Every
   `EquivalentAddressGroup` in the list must have a `ATTR_LB_ADDR_AUTHORITY`.
   This key would pass addresses via the Attributes arguments of
   `NameResolver.Listener.onAddresses(List<EAG>, Attributes)`
3. Change grpclb policy to look for LB addresses via ATTR_LB_ADDRS instead of
   the normal list of EAG.

### C

TBD
