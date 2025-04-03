A95: xDS LEDS and Fallback Improvements
----
* Author(s): @markdroth
* Approver: @ejona86, @dfawley
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2025-03-18
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

This design specifies some improvements to the xDS fallback functionality
described in [A71].  Specifically, it adds a configuration knob for
controlling whether fallback is triggered solely by reachability, and
it specifies how gRPC will support [LEDS].

## Background

The xDS fallback functionality described in [A71] was designed around
the assumption that the client should prefer sticking with cached
resources from the primary rather than switching to the fallback server.
That assumption is true for mostly static configuration data, as is
commonly found in LDS, RDS, and CDS.  However, it is not always true for
dynamicly generated data like EDS, because if clients don't switch to
the fallback server and stop getting updates, then they will slowly lose
knowledge of endpoints as the set of endpoints changes over time (e.g.,
due to auto-scaling).  So what we really need here is a way to avoid
falling back if resources are already cached for some resources, while
falling back based solely on server reachability for other resources.

In addition, there are cases where this distinction applies to only part
of the EDS resource.  Today, the EDS resource contains both locality
assignments and endpoint assignments, but there are cases where we want
to use fallback for the endpoint assignments while still using the
cached data for the locality assignments.  To support this, we need to
split up the EDS data into multiple resources, which can be done using
part of the mechanism designed in [LEDS].

### Related Proposals: 
* [A71: xDS Fallback][A71]
* [A47: xDS Federation][A47]
* [LEDS: Locality Endpoint Discovery Service][LEDS]
* [xRFC TP1: xdstp:// structured resource naming, caching and federation support][xRFC TP1]

[A71]: A71-xds-fallback.md
[A47]: A47-xds-federation.md
[LEDS]: https://docs.google.com/document/d/1aZ9ddX99BOWxmfiWZevSB5kzLAfH2TS8qQDcCBHcfSE/edit?usp=sharing
[xRFC TP1]: https://github.com/cncf/xds/blob/main/proposals/TP1-xds-transport-next.md

## Proposal

This proposal has two parts:
1. Adding a knob in the xDS bootstrap config to control the fallback criteria.
2. Adding support for LEDS using list collections.

### Bootstrap Knob to Control Fallback Criteria

Currently, as per [A71], we use fallback only if both (a) the primary
server is unreachable and (b) we have uncached resources.  For the
endpoint assignment data, we want to inhibit (b) -- i.e., we want to
fallback based solely on primary server reachability.

To address this, we propose to add a per-authority (see [A47]) knob
in the bootstrap config to control this.  We will add a field in the
authority called `fallback_on_reachability_only`, whose value will be
a boolean.  If set to true, then we will fallback when the primary server
is unreachable, even if we do not have any uncached resources.

Note that this knob must be per-authority instead of per-resource-type,
since we make fallback decisions on a per-authority basis.  The intent
here is that the EDS resource can use a different authority than the other
resources, so that it can make use of the alternative fallback behavior.

### LEDS List Collection Support

The [LEDS] design was originally designed to address scalability
concerns for large proxies.  The idea is to have the EDS resource
contain only the locality assignments, but then have it refer to other
resources for the endpoint assignments, where each endpoint is
represented as a separate resoure of type
[`LbEndpoint`](https://github.com/envoyproxy/envoy/blob/c5182bcc7a5e6138c36e6c894d19af152b82d48e/api/envoy/config/endpoint/v3/endpoint_components.proto#L101).
LEDS was initially designed to use glob collections (see [xRFC TP1]) to get
each individual endpoint in its own resource, which requires the use of
the xDS incremental protocol variant.

gRPC does not yet support the incremental protocol variants, and we
don't need that level of scalability; all we actually need here is to be
able to split up the locality assignment and endpoint assignment
information into separate resources.  While we would eventually like to
support the incremental protocol variant in gRPC, that is more work that
we don't really need right now.  So instead of using a glob collection,
we will use a list collection, which does not require the incremental
protocol variant.  The list collection will be an `LbEndpointCollection`
resource, introduced in https://github.com/envoyproxy/envoy/pull/38777.

TODO: document validation rules for LbEndpointCollection resource

### Temporary environment variable protection

[Name the environment variable(s) used to enable/disable the feature(s) this proposal introduces and their default(s).  Generally, features that are enabled by I/O should include this type of control until they have passed some testing criteria, which should also be detailed here.  This section may be omitted if there are none.]

## Rationale

[A discussion of alternate approaches and the trade offs, advantages, and disadvantages of the specified approach.]


## Implementation

[A description of the steps in the implementation, who will do them, and when.  If a particular language is going to get the implementation first, this section should list the proposed order.]

## Open issues (if applicable)

[A discussion of issues relating to this proposal for which the author does not know the solution. This section may be omitted if there are none.]
