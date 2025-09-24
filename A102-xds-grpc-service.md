A102: xDS `GrpcService` Support
----
* Author(s): @markdroth, @sergiitk
* Approver: @ejona86, @dfawley
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2025-09-18
* Discussion at: https://groups.google.com/g/grpc-io/c/3hguVpr8maE

## Abstract

There are several features that require the xDS control plane to configure
gRPC to talk to a side-channel service, such as rate limiting ([A77]),
ExtAuthz ([A92]), and ExtProc ([A93]).  This design specifies how the
control plane will configure the communication with these side-channel
services.  It also addresses the relevant security implications.

## Background

The control plane configures communication with side-channel services
via the xDS [`GrpcService`
proto](https://github.com/envoyproxy/envoy/blob/7ebdf6da0a49240778fd6fed42670157fde371db/api/envoy/config/core/v3/grpc_service.proto#L29).
This message tells the data plane how to find the side-channel service
and what channel credentials and call credentials to use for that
communication.

### Related Proposals: 
* [gRFC A27: xDS-Based Global Load Balancing][A27]
* [gRFC A29: xDS-Based mTLS Security for gRPC Clients and Servers][A29]
* [gRFC A77: xDS Server-Side Rate Limiting][A77] (WIP)
* [gRFC A81: xDS Authority Rewriting][A81]
* [gRFC A92: xDS ExtAuthz Support][A92] (WIP)
* [gRFC A93: xDS ExtProc Support][A93] (WIP)
* [A97: xDS JWT Call Credentials][A97]

[A27]: A27-xds-global-load-balancing.md
[A29]: A29-xds-tls-security.md
[A77]: https://github.com/grpc/proposal/pull/414
[A81]: A81-xds-authority-rewriting.md
[A92]: https://github.com/grpc/proposal/pull/481
[A93]: https://github.com/grpc/proposal/pull/484
[A97]: A97-xds-jwt-call-creds.md

## Proposal

gRPC will support the `GrpcService`
message.  In that message, gRPC will support only the
[`GoogleGrpc`](https://github.com/envoyproxy/envoy/blob/7ebdf6da0a49240778fd6fed42670157fde371db/api/envoy/config/core/v3/grpc_service.proto#L68)
target specifier, not
[`EnvoyGrpc`](https://github.com/envoyproxy/envoy/blob/7ebdf6da0a49240778fd6fed42670157fde371db/api/envoy/config/core/v3/grpc_service.proto#L33)
(see "Rationale" section below).

### Security Considerations

The control plane specifying the side-channel target and credentials
introduces a number of potential privilege-escalation attacks from a
compromised control plane.  Here are some examples of such attacks:

- Because the side-channel target name comes from the control plane
  rather than being configured locally on the client, a compromised
  control plane can tell the client to talk to an attacker-controlled
  side-channel service.  When used for functionality like ExtProc, this
  would allow the control plane to get access to the contents of data
  plane RPCs.

  - Note: Even if the client could enforce the use of a channel credential
    type like TLS that verifies that the server's identity matches the
    target name, that would not ameliorate this attack, because the
    target name itself is coming from the control plane.

  - Note: Even if the client locally configured the target name of
    the side-channel service but trusted the control plane to specify
    the credential type, the control plane could specify
    `InsecureCredentials`, and then it would just need to control the
    client's name resolution in order to send the client to an
    attacker-controlled side-channel service.

- A compromised control plane could instruct gRPC to contact an
  attacker-controlled side-channel service using a call credential that
  sends an access token, which would leak that access token.

There will be cases where it is acceptable to trust the control plane
to have that kind of privilege-escalation capability, and there will be
other cases where it is not.  To differentiate between these two cases, we
will rely on the `trused_xds_server` server feature that was added to the
gRPC xDS bootstrap config in [A81].

When gRPC receives a `GrpcService` proto from an xDS server, it will
check to see if the `trusted_xds_server` server feature is present in
the bootstrap config for that xDS server.  If so, then it will trust
the target name and credentials specified in the `GrpcService` proto.
If not, then we will provide a mechanism for it to obtain side-channel
configuration locally from the gRPC xDS bootstrap config.

Specifically, we will add the following new top-level field to the
bootstrap config:

```json5
// The list of side-channel services allowed to be configured via xDS.
"allowed_grpc_services": {
  // The key is fully-qualified target URI.
  "dns:///ratelimit.example.org:443": {
    // List of channel creds.  Client will stop at the first type it
    // supports.  This field is required and must contain at least one
    // channel creds type that the client supports.
    "channel_creds": [
      {
        "type": <string containing channel cred type>,
        // The "config" field is optional; it may be missing if the
        // credential type does not require config parameters.
        "config": <JSON object containing config for the type>
      }
    ]
    // List of call creds.  Optional.  Client will apply all call creds
    // types that it supports but will ignore any types that it does not
    // support.
    "call_creds": [
      {
        "type": <string containing call cred type>,
        // The "config" field is optional; it may be missing if the
        // credential type does not require config parameters.
        "config": <JSON object containing config for the type>
      }
    ]
  }
}
```

Note that the `channel_creds` and `call_creds` fields follow the same
format as the top-level `xds_servers` field.  See [A27] and [A97] for
details.

When gRPC receives a `GrpcService` proto from an untrusted control
plane, it will look up the target URI from the `GrpcService` proto in
the `allowed_grpc_services` map.  If the specified target URI is not
present in the map, then the `GrpcService` proto will be considered
invalid, resulting in gRPC NACKing the xDS resource.  If the specified
target URI *is* present in the map, then the `GrpcService` proto will be
considered valid, but gRPC will ignore the credential information from
the `GrpcService` proto; instead, it will use the channel credentials
and call credentials specified in the map.

### Credential Configuration in `GrpcService` Proto

In order to make channel and call credentials more pluggable, we are
introducing new extension points in `GrpcService`, as shown in
https://github.com/envoyproxy/envoy/pull/40823.  Specifically, this
introduces the following new fields:

- `channel_credentials_plugin`: This provides an extension point to
  specify channel credentials.  Just like in the gRPC xDS bootstrap
  format, in order to faciliate easier introduction of new credential
  types, this field is structured as a list, and the client will iterate
  over the list and stop at the first credential type that it supports.
  If it does not find any supported credential type in the list, that is
  a validation error, and the xDS resource will be NACKed.  If it finds
  a supported credential type but the config is invalid, then the xDS
  resource will also be NACKed.  The following extensions will be supported
  in this field:

  - `envoy.extensions.grpc_service.channel_credentials.google_default.v3.GoogleDefaultCredentials`
  - `envoy.extensions.grpc_service.channel_credentials.insecure.v3.InsecureCredentials`
  - `envoy.extensions.grpc_service.channel_credentials.local.v3.LocalCredentials`
  - `envoy.extensions.grpc_service.channel_credentials.tls.v3.TlsCredentials`:
    In this message:
    - `root_certificate_provider`: Required.  References certificate
      provider instances configured at the top level of the bootstrap
      config.  Validated the same way as in `CommonTlsContext` (see [A29]).
    - `identity_certificate_provider`: Optional.  References certificate
      provider instances configured at the top level of the bootstrap
      config.  Validated the same way as in `CommonTlsContext` (see [A29]).
  - `envoy.extensions.grpc_service.channel_credentials.xds.v3.XdsCredentials`:
    In this message:
    - `fallback_credentials`: Required. Specifies a channel credential
      plugin to be used as fallback credentials.

- `call_credentials_plugin`: This provides an extension point to specify
  call credentials.  Just like in the gRPC xDS bootstrap config format,
  in order to faciliate easier introduction of new credential types,
  this field is structured as a list, and the client will iterate over
  the list adding all credential types that it supports, ignoring any
  type that it does not support.  Note that unlike channel credentials,
  call credentials are optional, and there can be more than one, so the
  client will need to iterate over the entire list, but it's valid if
  none of the specified types are supported.  If the client finds a
  supported credential type but the config is invalid, then the xDS
  resource will be NACKed.  The following extensions will be supported
  in this field:

  - `envoy.extensions.grpc_service.call_credentials.access_token.v3.AccessTokenCredentials`:
    In this message:
    - `token`: Required.  The access token.  The token will be added as
      an `authorization` header with header `Bearer ` (note trailing
      space) followed by the value of this field.  Note that the
      token will not be sent on the wire unless the connection has
      security level PRIVACY_AND_INTEGRITY.

Note that this will require extending the channel credentials and call
credentials registries to support configuration via these protos, in
additional to the JSON formats that they already support for the gRPC
xDS bootstrap config.

### `GrpcService` Proto Validation

When validating a `GrpcService` proto, the following fields will be used:
- [`google_grpc`](https://github.com/envoyproxy/envoy/blob/7ebdf6da0a49240778fd6fed42670157fde371db/api/envoy/config/core/v3/grpc_service.proto#L303):
  This field must be set.  Inside of it:
  - [`target_uri`](https://github.com/envoyproxy/envoy/blob/7ebdf6da0a49240778fd6fed42670157fde371db/api/envoy/config/core/v3/grpc_service.proto#L254):
    Must be set to a valid gRPC target URI.  The target URI must be
    checked against the resolver registry during xDS resource
    validation.
  - `channel_credentials_plugin`: See above.
  - `call_credentials_plugin`: See above.
  - `channel_credentials`, `call_credentials`,
    `credentials_factory_name`, and `config`: Ignored; we will use the
    new credential plugin fields above instead.
  - `stat_prefix`: Ignored; not relevant to gRPC.
  - `per_stream_buffer_limit_bytes`: Ignored.  We don't have a use-case
    for this right now but could add it later if needed.
  - `channel_args`: Ignored.  Not supportable across languages in gRPC.
- [`timeout`](https://github.com/envoyproxy/envoy/blob/7ebdf6da0a49240778fd6fed42670157fde371db/api/envoy/config/core/v3/grpc_service.proto#L308):
  If set, this will be used to set the deadline on RPCs sent to the
  side-channel service.  The value must obey the restrictions specified in
  the [`google.protobuf.Duration`
  documentation](https://developers.google.com/protocol-buffers/docs/reference/google.protobuf#google.protobuf.Duration),
  and it must have a positive value.
- [`initial_metadata`](https://github.com/envoyproxy/envoy/blob/7ebdf6da0a49240778fd6fed42670157fde371db/api/envoy/config/core/v3/grpc_service.proto#L315):
  If present, specifies headers to be added to RPCs sent to the side-channel
  service.  Inside of each entry:
  - [`key`](https://github.com/envoyproxy/envoy/blob/7ebdf6da0a49240778fd6fed42670157fde371db/api/envoy/config/core/v3/base.proto#L404):
    Value length must be in the range [1, 16384).  Must be a valid
    HTTP/2 header name.
  - [`value`](https://github.com/envoyproxy/envoy/blob/7ebdf6da0a49240778fd6fed42670157fde371db/api/envoy/config/core/v3/base.proto#L415):
    Specifies the header value.  Must be shorter than 16384 bytes.  Must
    be a valid HTTP/2 header value.  Not used if `key` ends in `-bin`
    and `raw_value` is set.
  - [`raw_value`](https://github.com/envoyproxy/envoy/blob/7ebdf6da0a49240778fd6fed42670157fde371db/api/envoy/config/core/v3/base.proto#L422):
    Used only if `key` ends in `-bin`.  Must be shorter than 16384 bytes.
    Will be base64-encoded on the wire, unless the pure binary metadata
    extension from [gRFC G1: True Binary
    Metadata](G1-true-binary-metadata.md) is used.
- `envoy_grpc`: This field is not used.  See "Rationale" section below
  for details.
- `retry_policy`: This field is not used.  If retries are needed, they
  should be configured in the [service
  config](https://github.com/grpc/grpc/blob/master/doc/service_config.md)
  for the side-channel service, or by using xDS in the side-channel.

### Temporary environment variable protection

This gRFC does not describe a discrete feature; the functionality it
describes will be used only in the context of other features, which
will each have their own environment variable protection.  Therefore,
no additional environment variable protection is needed here.

Note that when implementing one of those other features, it will be
important for the appropriate environment variable guard to cover
reading the `allowed_grpc_services` field in the bootstrap config.

## Rationale

### `GoogleGrpc` vs. `EnvoyGrpc`

Envoy supports both `GoogleGrpc` and `EnvoyGrpc` target specifiers.
The latter uses Envoy's own gRPC implementation, which is essentially
a small wrapper on top of its existing HTTP/2 functionality.  Rather
than configuring the side-channel using a gRPC target URI, it specifies
the side-channel using the name of an xDS cluster, which must already be
part of the data plane's configuration.  That approach does not make
sense in gRPC, for two reasons.

First, even in Envoy, `EnvoyGrpc` makes sense only in cases where
the data plane's xDS configuration already includes a cluster for the
side-channel service; in any other case, `GoogleGrpc` would be used
instead.  But unlike Envoy, gRPC is not a general-purpose proxy that
handles routing requests for multiple services in a single instance;
instead, each gRPC channel is created for one specific target (i.e.,
one particular service) and generally contains only the configuration
for that service, which means that in practice its xDS configuration
never includes the xDS cluster for the side-channel service.

Second, even if gRPC's xDS configuration did include the cluster for
the side-channel service, gRPC's architecture does not support sending
traffic to a specific cluster.  Unlike Envoy, gRPC does not have a
distinct Cluster Manager that can be used to select a cluster to send
requests to, which means that it fundamentally doesn't make sense to
use an xDS cluster name to contact the side-channel service.

### Security Concerns

A more comprehensive approach to the security concerns would be to
provide a mechanism to cryptographically sign the xDS resources and have
the client verify the signature.  This would ensure that a compromised
control plane would not be able to send arbitrary resources to clients;
instead, the attack would have to happen where the xDS resources are
constructed and signed, which could in principle be better protected.

We are in favor of this approach, but it will require a lot more work,
so we are leaving it as a future improvement.

## Implementation

Will be implemented in C-core, Java, Go, and Node as part of either RLQS
([A77]), ExtAuthz ([A92]), or ExtProc ([A93]), whichever happens to be
implemented first in any given language.
