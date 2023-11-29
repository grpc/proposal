A63: xDS StringMatcher in Header Matching
----
* Author(s): @markdroth
* Approver: @ejona86
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2023-05-03
* Discussion at: https://groups.google.com/g/grpc-io/c/dJFBiMzs6C0

## Abstract

gRPC will add support for the `StringMatcher` field in xDS header matching.

## Background

gRPC introduced support for xDS routing in [gRFC A28][A28].  Since that
feature was implemented, however, a new `StringMatcher` field was added
to the xDS `HeaderMatcher` proto in
https://github.com/envoyproxy/envoy/pull/17119.  This provides a more
general-purpose matching API and adds the ability to make matches in a
case-insensitive way.

This proposal updates gRPC to support this new field.

### Related Proposals: 
* [gRFC A28: xDS Traffic Splitting and Routing][A28]
* [gRFC A41: xDS RBAC Support][A41]

## Proposal

gRPC will support the [`HeaderMatcher.string_match` field][new_xds_field].
Note that this field is part of a `oneof`, so it is an alternative to
the existing fields that gRPC already supports.

The new field provides a superset of the functionality of the existing
fields `exact_match`, `safe_regex_match`, `prefix_match`, `suffix_match`,
and `contains_match`.  Those fields are marked as deprecated in the
xDS proto.  However, those fields are still commonly used, so gRPC will
continue to support them for the foreseeable future.

The new field provides one additional feature over the old fields, which
is the ability to ignore case in matches via the [`ignore_case`
field](https://github.com/envoyproxy/envoy/blob/3fe4b8d335fa339ef6f17325c8d31f87ade7bb1a/api/envoy/type/matcher/v3/string.proto#L69).
Note that this option is ignored for regex matches.

Note that gRPC has existing code to support the `StringMatcher` proto as
part of supporting RBAC, as specified in [gRFC A41][A41].  If possible,
gRPC implementations should provide common code for evaluating header
matches that can be shared between the two features.

### Temporary environment variable protection

No environment variable protection is proposed for this feature, since
it's a simple extension of existing header matching functionality.  Unit
test coverage should be sufficient to ensure that the new fields are
handled correctly.

## Rationale

N/A

## Implementation

Implemented in C-core in https://github.com/grpc/grpc/pull/32993.

## Open issues (if applicable)

N/A

[A28]: A28-xds-traffic-splitting-and-routing.md
[A41]: A41-xds-rbac.md
[new_xds_field]: https://github.com/envoyproxy/envoy/blob/3fe4b8d335fa339ef6f17325c8d31f87ade7bb1a/api/envoy/config/route/v3/route_components.proto#L2280
