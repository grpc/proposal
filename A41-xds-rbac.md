A41: xDS RBAC Support
----
* Author(s): [Eric Anderson](https://github.com/ejona86), [Ashitha
  Santhosh](https://github.com/ashithasantosh) (RBAC engine behavior)
* Approver: markdroth
* Status: Draft {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2021-05-07
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

Support the [xDS RBAC HTTP filter][RBAC filter] for service- and method-scoped
client authorization (authz) on xDS-enabled gRPC servers.

[RBAC filter]: https://github.com/envoyproxy/envoy/blob/main/api/envoy/extensions/filters/http/rbac/v3/rbac.proto

## Background

[A29 xDS-Based Security][A29] introduced the ability to manage xDS-managed mTLS
meshes. As standard for TLS clients, the client verifies the server's identity
is permitted to run the service. This is normally known as "hostname
verification", but in a mesh scenario may be known as "server authorization" as
the server's certificate does not include the hostname but other identity
information.

While servers authenticated clients (i.e., the client's identity was verified),
A29 did not include mechanism to limit which clients were authorized to access a
server and for which gRPC methods (i.e., whether the client was permitted).

Envoy defines the [RBAC HTTP filter][RBAC filter] that is well-suited for
limiting which nodes can access which services within a mesh as encouraged by
the principle of least privilege.

Resources in REST are present in the HTTP path so it would seem RBAC could be
used in non-gRPC environments as a precise service-level resource authz engine.
However, this 1) requires another filter to perform end-user authentication (as
TLS just provides service-level authn) and 2) does not scale as RBAC provides no
mechanism for loading rules on-demand. Since gRPC encodes service-level
resource identifiers in the request message, RBAC does not have access to the
requested resource for gRPC traffic.

### Related Proposals:

* [A36: xDS-Enabled Servers][A36]
* [A29: xDS-Based Security for gRPC Clients and Servers][A29]
* [A39: xDS HTTP Filter Support][A39]

[A36]: https://github.com/grpc/proposal/blob/master/A36-xds-for-servers.md
<!-- FIXME: replace with final link once merged -->
[A29]: https://github.com/grpc/proposal/pull/184
[A39]: https://github.com/grpc/proposal/blob/master/A39-xds-http-filters.md

## Proposal

Support the [xDS RBAC HTTP filter][RBAC filter] as a registered filter type on
xDS-enabled gRPC Servers, and support its `RBACPerRoute` configuration override.
This does _not_ include [RBAC network filter][] support nor running the filter
on gRPC clients. xDS v2 support is not necessary.

Supporting the RBAC HTTP filter on server-side leverages [A36: xDS-Enabled
Servers][A36] and [A39: xDS HTTP Filter Support][A39]. Server-side HTTP filters
are less common than client-side at this point in time, so this may be the first
usage of `RouteConfiguration` on server-side and so likely involves adding more
complete support for A39 by creating and executing filters on server-side and
processing `VirtualHost`s and `RoutMatch`es to determine which configuration
should be provided to each filter. A39 should be consulted for the expected
behavior.

The core policy matching logic should be split into an "RBAC engine" to allow
reuse with non-xDS environments. However, there is no requirement that the RBAC
engine have a specialized API; it could simply be an interceptor.

The [RBAC policy][] has many fields. All current (Envoy-implemented) fields will
be considered in gRPC. However, some fields may have pre-determined values or
behavior. At this time, if the `RBAC.action` is `Action.LOG` then the policy
will be completely ignored, as if RBAC was not configurated. CEL fields are not
supported, so if `Policy.condition` or `Policy.checked_condition` is present the
XdsClient must NACK the configuration.

The following fields of `Permission` may not be obvious how they map to gRPC:

| Permission Field | gRPC Equivalent |
| ---------------- | --------------- |
| header           | Metadata (with caveats below) |
| url_path         | Fully-qualified RPC method name with leading slash |
| destination_ip   | Local address for this connection |
| destination_port | Local port for this connection |
| metadata         | Hard-coded as empty; never matches |
| requested_server_name | Hard-coded as empty string |

Because matching supports NOT, the matcher must still be processed even if a
rule contains references to things that don't generally match; it is not trivial
to "optimize out" the never-matching rules.

The `header` field is not entirely 1:1 with gRPC Metadata. To begin with, gRPC
Metadata isn't 100% consistent cross-language in its handling of [hop-by-hop
headers][] (e.g., `TE`; RFC 2616 has [a convenient list][hop-by-hop header
list]) and pseudo-headers. For this design, `headers` can include `:method`,
`:scheme`, `:authority`, and`:path` matchers and they should match the values
received on-the-wire independent of whether they are stored in Metadata or in
separate APIs. `:scheme` is not universally available in gRPC APIs, so it may be
hard-coded to `http` if unavailable. `:method` can be hard-coded to `POST` if
unavailable and a code audit confirms the server denies requests for all other
method types. It is unspecified whether hop-by-hop headers are matched.
Multi-valued metadata is represented as the concatenation of the values along
with a `,` (comma, no added spaces) separator, as permitted by HTTP and gRPC.
The Content-Type provided by the client must be used; not a hard-coded value.
(TODO: support binary headers?) Binary headers are represented in their
base64-encoded form, although we rarely expect binary header matchers.

[hop-by-hop headers]: https://datatracker.ietf.org/doc/html/rfc7230#section-6.1
[hop-by-hop header list]: https://datatracker.ietf.org/doc/html/rfc2616#section-13.5.1

As documented for `HeaderMatcher`, Envoy aliases `:authority` and `Host` in its
header map implementation, so they should be treated equivalent for the RBAC
matchers; there must be no behavior change depending on which of the two header
names is used in the _RBAC policy_.

TODO: Determine how to handle _requests_ (not policies) that have Host header,
since gRPC generally doesn't observe this header yet the RBAC policy could.
Options: deny such requests, move Host header to :authority, drop Host header.
Host and :authority could also disagree, and that needs to be handled.

`metadata` will never match as `ValueMatcher` can only match if the value is
present (even `NullMatch`). Be strongly aware that Envoy Metadata has no
relation to gRPC's Metadata.

`requested_server_name` can match if the matcher accepts empty string.

The following fields of `Principal` may not be obvious how they map to gRPC:

| Principal Field  | gRPC Equivalent |
| ---------------- | --------------- |
| authenticated.principal_name | The first URI/DNS SAN; same as Envoy |
| source_ip        | Peer address for this connection |
| direct_remote_ip | Peer address for this connection |
| remote_ip        | Peer address for this connection |
| header           | Same as in Permission |
| url_path         | Same as in Permission |
| metadata         | Same as in Permission |

TODO: Need to NACK to be consistent on `remote_ip` `source_ip` if
x-forwarded-for, proxy protocol, etc is configured.

[RBAC policy]: https://github.com/envoyproxy/envoy/blob/10c17a7cd90b013c38dfbfbf715d3c24fdd0477c/api/envoy/config/rbac/v3/rbac.proto
[RBAC network filter]: https://github.com/envoyproxy/envoy/blob/main/api/envoy/extensions/filters/network/rbac/v3/rbac.proto

### Temporary environment variable protection

The environment variable `GRPC_XDS_EXPERIMENTAL_RBAC` will be used to gate
the feature until it is deemed stable for a particular implementation. If unset
or not `true`, all RBAC logic presented in this gRFC will not come into effect.

## Rationale

Few design decisions were necessary; RBAC was a pretty strong fit with gRPC's
needs for an initial authorization policy. The name seems like a misnomer as it
lacks a level of indirection (the "role") between principals and permissions as
would be expected from "role based access control", but that is a fair design
decision and does not impact applicability at this time.

Fuller-fledged authz policy supporting end-user authorization for
application-specific resources would be useful, but needs to be future work as
it has multiple technical challenges including need for delayed authz policy
loading (i.e., delegating to a remote authz server) to be able to scale, a way
to extract the application-level resource from the request, and support for
authenticating end users.

Envoy also defines an [RBAC network filter][]. Such a filter is helpful for TCP
or non-HTTP proxying, but could be used for HTTP traffic. Such a network filter
has the advantage over an HTTP filter as it would 1) have lower ALLOW overhead
as the check is once per connection and 2) reduce the attack surface to
malicious clients. When used with HTTP/2, however, avoiding wildly out-of-date
authz checks would require limiting the lifetime of HTTP/2 connections. In
addition, there is no way to report clear error messages to the client. Since
it is unable to support path-based verification it would likely need to be used
in conjunction with an RBAC HTTP filter for usage in gRPC. It may be useful to
support one day, but would most likely be a more advanced feature.

## Implementation

[A description of the steps in the implementation, who will do them, and when.  If a particular language is going to get the implementation first, this section should list the proposed order.]

## Open issues (if applicable)

[A discussion of issues relating to this proposal for which the author does not know the solution. This section may be omitted if there are none.]
