A59: gRPC Audit Logging
----
* Author(s): [Luwei Ge](https://github.com/rockspore)
* Approver:
* Status: Draft {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in:
* Last updated: 2023-03-09
* Discussion at: 

## Abstract

Support audit logging based on authorization events in gRPC servers in both
xDS-enabled and gRPC authorization policy based use cases.

## Background

gRPC's authorization support is based on the [RBAC policy][]. In
xDS-enabled cases, users can configure authorization policies in the control
plane which applies [RBAC HTTP filters][RBAC filter] to the gRPC servers. When
xDS is not an option, users can use the [gRPC authorization policy][A43] to
get similar authorization enforment.

This proposal describes the audit logging APIs that allow users to audit
specific authorization events in their gRPC servers with the necessary context.
We aim to offer consistent APIs to support both aforementioned use cases.

[RBAC filter]: https://github.com/envoyproxy/envoy/blob/main/api/envoy/extensions/filters/http/rbac/v3/rbac.proto
[RBAC policy]: https://github.com/envoyproxy/envoy/blob/main/api/envoy/config/rbac/v3/rbac.proto
[HttpConnectionManager proto]: https://github.com/envoyproxy/envoy/blob/main/api/envoy/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto

### Related Proposals: 
* [gRFC A41: xDS RBAC Support][A41]
* [gRFC A43: gRPC Authorization API][A43]

[A41]: A41-xds-rbac.md
[A43]: A43-grpc-authorization-api.md

## Proposal

### Audit Context

The table below lists the metadata we will include in the audit context,
which will be made available to audit loggers during an audit event.

It is worth noting that audit happens right after the authorization
decision is made, so the status only reflects that but not the eventual
one from the server. 

|Fields             |Additional Information                                                             |Example                |
|-------------------|-----------------------------------------------------------------------------------|-----------------------|
|RPC Method         |Method the RPC hit.                                                                |"/pkg.service/foo"     |
|Principal          |Identity the RPC. Currently only available in certificate-based TLS authentication.|"spiffe://foo/user1"   |
|Timestamp          |Time when the RPC happens.                                                         |1649376211             |
|Denying Policy Name|Policy name that denied the RPC. Empty if RPC is allowed.                          |"policy_name"          |
|Status             |gRPC status from the authorization enforcement, not the eventual RPC status.       |OK or PERMISSION_DENIED|

### xDS API Changes

xDS API changes are needed because gRPC authorization is entirely based on [xDS RBAC policy][RBAC policy]. Moreoever, audit logging is a common feature
that other xDS clients (namely Envoy) should benefit from as well.

We will add an audit condition enum in the [xDS RBAC policy][RBAC policy] as
below:

```
package envoy.config.rbac.v3;

message RBAC {
  ...

  // Deny and allow here refer to RBAC decisions, not actions.
  enum AuditCondition {
    // Never audit.
    NONE = 0;

    // Audit when RBAC denies the request.
    ON_DENY = 1;

    // Audit when RBAC allows the request.
    ON_ALLOW = 2;

    // Audit whether RBAC allows or denies the request.
    ON_DENY_AND_ALLOW = 3;
  }

  // Condition for the audit logging to be happen, specific to HTTP filters.
  // If condition is met, all the audit loggers configured in the HCM will be invoked.
  //
  AuditCondition audit_condition = 3;
}
```

Note that `DENY` and `ALLOW` in the enum refer to the RBAC decisions, not
RBAC actions.

For the audit logger configuration, we will follow the existing `access_log`
field in the [HTTP Connection Manager][HttpConnectionManager proto] by adding
a typed extension category `audit_log`:

```
package envoy.extensions.filters.network.http_connection_manager.v3;

message HttpConnectionManager {
  ...
  
  // Configuration for the audit log.
  // The audit_loggers instantiated from this will be used in all RBAC filters.
  //
  // [#extension-category: envoy.audit_loggers]
  repeated xds.core.v3.TypedExtensionConfig audit_log = 53;
}
```

The list of configured audit loggers will all be invoked by an RBAC filter when
the audit condition is met. This is analogus to the semantics that all access
loggers will be run after the response is sent.

### gRPC Authorzation Policy Changes

The gRPC authorization policy consists of `deny_rules` and `allow_rules`, which
are mapped to two RBACs. Overall it's a subset of the RBAC as stated in [A43: gRPC authorization API][A43].
We will keep this philosophy by adding just one `audit_condition` in the policy
instead of having separate conditions for those two set of rules. Likewise, only
one audit logger can be specified, the configuration of which is the json
representation of the typed config used in xDS.

Following are changes to the JSON schema of gRPC Authorization Policy introduced
in [A43: gRPC authorization API][A43]:

```json
{
  "title": "AuthorizationPolicy",
  "definitions": {
    ...
    "audit_log": {
      "description": "Configuration for the audit log.",
      "type": "object",
      "properties": {
        "name": {
          "description": "The name of the audit_log configuration."
            "It is mainly used for human purposes.",
          "type": "string",
        },
        "typed_config": {
          "description": "The typed config for the audit_log."
            "This needs to be a json object mapped from google.protobuf.Any"
            "proto message.",
          "type": "object",
        }
      },
    }
  },
  "properties": {
    ...
    "audit_condition": {
      "description": "Condition for the audit logging to be happen."
        "Same as NONE if ommited.",
      "type": "string",
      "enum": ["NONE", "ON_DENY", "ON_ALLOW", "ON_DENY_AND_ALLOW"]
    },
    "audit_log": {
      "$ref": "#/definitions/audit_log"
    }
  },
  "required": ["name", "allow_rules"]
}
```

For readers that are more familiar with proto messages, the change translates
to:

```
message AuthorizationPolicy {
  …

  enum AuditCondition {
    NONE = 0;
    ON_DENY = 1;
    ON_ALLOW = 2;
    ALWAYS = 3;
  }

  AuditCondition audit_condition = 1;

  // See https://github.com/cncf/xds/blob/main/xds/core/v3/extension.proto.
  xds.core.v3.TypedExtensionConfig audit_log = 2;
}
```

Note that the definition above is only for illustration purposes and does not
actually exist anywhere as a `.proto` file. Nor does gRPC process the policy
as protobuf by any means.

### Language Specific APIs

_This section is to be done once the language-agnostic APIs are more finalized._

_What is certain for now is we need to support third-party audit logger
implementations registered by users._

## Rationale

So far, we have considered alternatives in a number of othorgonal aspects.

### Utilizing the Access Log

The [information](https://github.com/envoyproxy/envoy/blob/main/api/envoy/data/accesslog/v3/accesslog.proto)
present in the existing access log is sufficient for audit purposes. However,
access log does not happen till the response is sent and therefore does not
fit the audit logging use case well. For example, for long-living RPCs, users
would want to audit the event right after the authorization is enfoced but
access log won't necessarily happen till hours later.

### Audit Condition in the HTTP Connection Manager

As an HTTP filter chain can contain multiple RBAC filters in general. The typical
behavior that users will expect if they want to audit on allow is audit should
only happen once after the evaluation of the last RBAC filter. From this
perspective, the audit condition could be considered as something on top of all
RBAC filters and thus configured in the [HTTP Connection Manager][HttpConnectionManager proto].

However, given that xDS users normally do not craft RBACs on their own but instead
rely on the control plane APIs, such as Istio Authorization Policy, it is
reasonable to let the control plane handle the translation properly.
Meanwhile, configuring audit condition in individual RBACs offers the flexibility
when users really want to audit events related to a particular RBAC.

### Audit Log Configured In gRPC Bootstrap

Instead of having the audit log configuration delivered from the control plane,
we could follow the xDS TLS certificate provider in [A29: xDS-Based Security for gRPC Clients and Servers](A29-xds-tls-security.md)
where the configuration is placed in the gRPC bootstrap file.

The advantage of that apporoach is the control plane would not need to know
the platform-specific configuration options in the data plane, if any.
But we concluded that in audit logging, the logger configuration should
generally be uniform across the data plane and thus centralizing the configuration
makes more sense. Moreover, the existing access log follows such a pattern
in xDS, albeit not currently supported by gRPC.

## Implementation

The implementation order will be C++, Go, Java and then wrapped languages.