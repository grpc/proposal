A41: xDS RBAC Support
----
* Author(s): [Eric Anderson](https://github.com/ejona86), [Ashitha
  Santhosh](https://github.com/ashithasantosh)
* Approver: markdroth
* Status: Ready for Implementation
* Implemented in: <language, ...>
* Last updated: 2021-09-08
* Discussion at: https://groups.google.com/g/grpc-io/c/DQJfShLTrTQ/

## Abstract

Support the [xDS RBAC HTTP filter][RBAC filter] for service- and method-scoped
client authorization (authz) on xDS-enabled gRPC servers.

[RBAC filter]: https://github.com/envoyproxy/envoy/blob/main/api/envoy/extensions/filters/http/rbac/v3/rbac.proto

## Background

[A29 xDS-Based Security][A29] introduced the ability to have xDS-managed mTLS
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

Services that have state managed by their clients tend to need precise
authorization to that state, commonly driven by per-resource ACLs. RBAC is not
well suited for this precise authorization, even if the resource name
is exposed to the engine. Per-resource authorization 1) commonly uses end-user
identity so requires another filter to perform end-user authentication and
2) is too great of scale to enumerate all resource's ACLs in a single policy and
to absorb the rate of change to that policy. Thus RBAC is only intended for
service-to-service authorization and resource authorization is outside the scope
of this gRFC.

### Related Proposals:

* [A36: xDS-Enabled Servers][A36]
* [A29: xDS-Based Security for gRPC Clients and Servers][A29]
* [A39: xDS HTTP Filter Support][A39]

[A36]: A36-xds-for-servers.md
[A29]: A29-xds-tls-security.md
[A39]: A39-xds-http-filters.md

## Proposal

Support the [xDS RBAC HTTP filter][RBAC filter] as a registered filter type on
xDS-enabled gRPC Servers, and support its `RBACPerRoute` configuration override.
This does _not_ include [RBAC network filter][] support nor running the filter
on gRPC clients. xDS v2 support is not necessary. Shadow rules will not be
supported and is left as a potential future enhancement; only `RBAC.rules` will
be supported as they can function without stats.

Supporting the RBAC HTTP filter on server-side leverages [A36: xDS-Enabled
Servers][A36] and [A39: xDS HTTP Filter Support][A39]. Server-side HTTP filters
are less common than client-side at this point in time, so this may be the first
usage of `RouteConfiguration` on server-side and so likely involves adding more
complete support for A39 by creating and executing filters on server-side and
processing `VirtualHost`s and `RoutMatch`es to determine which configuration
should be provided to each filter. A39 should be consulted for the expected
behavior.

New validation should occur for `HttpConnectionManager` to allow equating
RBAC's `direct_remote_ip` and `remote_ip`. If the RBAC implementation does not
distinguish between these fields, then
`HttpConnectionManager.xff_num_trusted_hops` must be unset or zero and
`HttpConnectionManager.original_ip_detection_extensions` must be empty. If
either field has an incorrect value, the Listener must be NACKed. For
simplicity, this behavior applies independent of the Listener type (both
client-side and server-side).

The core policy matching logic should be split into an "RBAC engine" to allow
internal reuse with a non-xDS API. Any non-xDS API will not be RBAC, so this
reuse is merely an implementation detail. The API is not part of this gRFC.
There is no requirement that the RBAC engine have a specialized API; it could
simply be an interceptor.

The [RBAC policy][] has many fields. All current (Envoy-implemented) fields will
be considered in gRPC. However, some fields may have pre-determined values or
behavior. At this time, if the `RBAC.action` is `Action.LOG` then the policy
will be completely ignored, as if RBAC was not configurated. CEL fields are not
supported, so `Policy.condition` and `Policy.checked_condition` must cause a
validation failure if present. It is also a validation failure if Permission or
Principal has a `header` matcher for a `grpc-`-prefixed header name or
`:scheme`. As described in A39, validation failures for filter configuration
causes the listener to be NACKed.

The following fields of `Permission` may not be obvious how they map to gRPC:

| Permission Field | gRPC Equivalent |
| ---------------- | --------------- |
| header           | Metadata (with caveats below) |
| url_path         | Fully-qualified RPC method name with leading slash. Same as `:path` header |
| destination_ip   | Local address for this connection |
| destination_port | Local port for this connection |
| metadata         | Hard-coded as empty; never matches |
| requested_server_name | Hard-coded as empty string |

Because matching supports NOT, the matcher must still be processed even if a
rule contains references to things that don't generally match; it is not trivial
to "optimize out" the never-matching rules.

The `header` field is not entirely 1:1 with gRPC Metadata, in part because which
HTTP headers are present in Metadata is not 100% consistent cross-language.
For this design, `headers` can include `:method`, `:authority`, and `:path`
matchers and they should match the values received on-the-wire independent of
whether they are stored in Metadata or in separate APIs. `:method` can be
hard-coded to `POST` if unavailable and a code audit confirms the server denies
requests for all other method types. Implementations must consider the
request's [hop-by-hop headers][] to not be present. Since hop-by-hop headers
[are not used in HTTP/2 except for `te: trailers`][rfc7540 connection header],
transports must consider requests containing the `Connection` header as
malformed, independent of xDS or RBAC, and only the `TE` header may need special
handling. If the transport exposes `TE` in Metadata, then RBAC must special-case
the header to treat it as not present. Multi-valued metadata is represented as
the concatenation of the values along with a `,` (comma, no added spaces)
separator, as permitted by HTTP and gRPC. The `Content-Type` provided by the
client must be used; not a hard-coded value. Binary headers are represented in
their base64-encoded form, although we rarely expect binary header matchers
other than presence-checking.

[hop-by-hop headers]: https://datatracker.ietf.org/doc/html/rfc7230#section-6.1
[rfc7540 connection header]: https://datatracker.ietf.org/doc/html/rfc7540#section-8.1.2.2

As documented for `HeaderMatcher`, Envoy aliases `:authority` and `Host` in its
header map implementation, so they should be treated equivalent for the RBAC
matchers; there must be no behavior change depending on which of the two header
names is used in the _RBAC policy_.

The core gRPC implementation (not just xDS or RBAC) must observe both :authority
and Host headers. If :authority is missing, Host must be renamed to :authority.
If :authority is present, Host must be discarded. If multiple Host headers or
multiple :authority headers are present, the request must be rejected with an
HTTP status code 400 as required by Host validation in RFC 7230 §5.4, gRPC
status code INTERNAL, or RST_STREAM with HTTP/2 error code PROTOCOL_ERROR. These
restrictions and behavior produce a singular, unambiguous authority for every
request to be used by RBAC and the application itself.

If a header is not present, `HeaderMatch` will not match _except_ for
`present_match` when `present_match == invert_match`. This is because
`HeaderMatcher.invert_match` inverts the comparison operation so
something like `exact_match` changes from an `==` comparison to a `!=`
comparison, and both fail to match if the header is not present.

In RBAC `metadata` refers to the Envoy metadata which has no relation to gRPC
metadata. Envoy metadata is generic state shared between filters, which has no
gRPC equivalent. RBAC implementations in gRPC will treat Envoy metadata as an
empty map. Since `ValueMatcher` can only match if a value is present (even
`NullMatch`), the `metadata` matcher is guaranteed not to match.

`requested_server_name` can match if the matcher accepts empty string.

The following fields of `Principal` may not be obvious how they map to gRPC:

| Principal Field  | gRPC Equivalent |
| ---------------- | --------------- |
| authenticated.principal_name | The URI/DNS SAN or Subject; same as Envoy |
| source_ip        | Peer address for this connection |
| direct_remote_ip | Peer address for this connection |
| remote_ip        | Peer address for this connection |
| header           | Same as in Permission |
| url_path         | Same as in Permission |
| metadata         | Same as in Permission |

The `authenticated.principal_name` will use the same definition as its
`rbac.proto` comment, although it checks multiple values which isn't clear from
the comment. If `principal_name` is unset, then `Authenticated` is said to match
if the connection uses TLS; a client certificate is not necessary. Other
values being checked are derived from the client's certificate. The process is:

1. Check if any SubjectAltName entry with URI type (type 6) matches. If any
   entry matches, the `principal_name` is said to match
2. If there is no SAN with URI type, check if any SAN entry with DNS type (type
   2) matches. If any entry matches, the `principal_name` is said to match
3. If there are no SAN with URI or DNS types, check if the Subject's
   distinguished name formatted as an RFC 2253 Name matches. If it matches, the
   `principal_name` is said to match
4. If there is no client certificate (thus no SAN nor Subject), check if `""`
   (empty string) matches. If it matches, the `principal_name` is said to match

`source_ip` is the same as `direct_remote_ip` only as long as
[envoy.extensions.filters.listener.proxy_protocol.v3.ProxyProtocol][] is
unsupported. If a future gRFC adds ProxyProtocol support it must also update
`source_ip` and `remote_ip` handling in RBAC.

`remote_ip` is the same as `direct_remote_ip` only as long as ProxyProtocol and
`xff_num_trusted_hops` are unsupported.

[RBAC policy]: https://github.com/envoyproxy/envoy/blob/10c17a7cd90b013c38dfbfbf715d3c24fdd0477c/api/envoy/config/rbac/v3/rbac.proto
[RBAC network filter]: https://github.com/envoyproxy/envoy/blob/main/api/envoy/extensions/filters/network/rbac/v3/rbac.proto
[envoy.extensions.filters.listener.proxy_protocol.v3.ProxyProtocol]: https://github.com/envoyproxy/envoy/blob/main/api/envoy/extensions/filters/listener/proxy_protocol/v3/proxy_protocol.proto

### Temporary environment variable protection

The environment variable `GRPC_XDS_EXPERIMENTAL_RBAC` will be used to gate
the feature until it is deemed stable for a particular implementation. If unset
or not `true`, all logic presented in this gRFC will not come into effect.

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

This will be implemented simultaneously in Java by @YifeiZhuang and @voidzcy, in
Go by @zasweq, and C++ by @yashykt.
