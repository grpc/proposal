A59: gRPC Audit Logging
----
* Author(s): [Luwei Ge](https://github.com/rockspore)
* Approver: [Eric Anderson](https://github.com/ejona86)
* Status: In Review {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in:
* Last updated: 2023-03-09
* Discussion at: https://groups.google.com/g/grpc-io/c/LbnuqGqEwuM

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
decision is made, so the status only reflects that, but not the eventual
one from the server. 

|Fields             |Additional Information                                                                 |Example                |
|-------------------|---------------------------------------------------------------------------------------|-----------------------|
|RPC Method         |Method the RPC hit.                                                                    |"/pkg.service/foo"     |
|Principal          |Identity the RPC. Currently only available in certificate-based TLS authentication.    |"spiffe://foo/user1"   |
|Policy Name        |The authorization policy name (or the xDS RBAC filter name).                           |"example-policy"       |
|Matched Rule       |The matched rule (or the matched policy name in [RBAC][RBAC policy]. Empty if no match.|"admin-access"         |
|Authorized         |Boolean indicating whether the RPC is authorized or not.                               |true                   |

### xDS API Changes

xDS API changes are needed because gRPC authorization is entirely based on [xDS RBAC policy][RBAC policy].
Moreoever, audit logging is a common feature that other xDS clients (namely Envoy) should benefit from as well.

We will add an audit condition enum (see [PR](https://github.com/envoyproxy/envoy/pull/26001))
in the [xDS RBAC policy][RBAC policy] as below:

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

  // Configuration for the audit loggers.
  //
  // [#extension-category: envoy.audit_loggers]
  repeated xds.core.v3.TypedExtensionConfig audit_log = 53;
}
```

Note that `DENY` and `ALLOW` in the enum refer to the RBAC decisions, not
RBAC actions. For example, we consider it to be a `DENY` decision when a RBAC
rejects the RPC, regardless of whether it is because there is no policy match
from a RBAC with the `ALLOW` action, or because there is a match for a RBAC
with a `DENY` action. Likewise, it is an `ALLOW` decision when there are no
matches to RBACs with a `DENY` action and at least one match to an RBAC with an
`ALLOW` action.

The audit logger configuration is also placed inside each individual RBACs. The
list of configured audit loggers will all be invoked by an RBAC filter when
the audit condition is met. This is analogus to the semantics that all access
loggers will be run after the response is sent.

Some readers may immediately notice that since an HTTP filter chain can contain
an arbitrary number of RBAC filters, more than one of them could meet its audit
condition for the same RPC and multiple audit log entries occur. When this happens,
it would be impossible to uniquely identify a specific RPC from a log entry so as
to apply any sort of de-duplication. The design in this gRFC does not intend to
address this issue as we argue it is a rare use case. This will be elaborated more
in [one of the design alternatives](#audit-condition-in-the-http-connection-manager).

### gRPC Authorization Policy Changes

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
    "audit_logger": {
      "description": "Configuration for the audit logger.",
      "type": "object",
      "properties": {
        "name": {
          "description": "The name of the audit logger configuration."
            "It is mainly used for human purposes.",
          "type": "string",
        },
        "typed_config": {
          "description": "The typed config for the audit logger."
            "This needs to be a json object mapped from google.protobuf.Struct"
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
    "audit_logger": {
      "$ref": "#/definitions/audit_logger"
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
    ON_DENY_AND_ALLOW = 3;
  }

  AuditCondition audit_condition = 1;

  message AuditLogger {
    string name = 1;
    google.protobuf.Struct config = 2;  
  }
  
  AuditLogger audit_logger = 2;
}
```

Note that the definition above is only for illustration purposes and does not
actually exist anywhere as a `.proto` file. Nor does gRPC process the policy
as protobuf by any means.

The authorization policy is backed by two RBAC filters, so the audit logger
configurations will be duplicated in both generated RBAC filters. How the
audit condition gets translated into two conditions in those RBACs is less
straightforward and is explained below.

First of all, note that the RBAC with `DENY` is placed before the RBAC with
`ALLOW`. We assume users will want to audit one particular RPC once if the
condition is met.

If users want to audit on deny, then both of the RBACs will have `ON_DENY` as
the audit condition. The `ALLOW` RBAC will not be evaluated so audit logging
will never happen twice. 

If users want to audit on allow, then the `DENY` RBAC will have no audit enabled
and the `ALLOW` RBAC with `ON_ALLOW`. So audit logging will at most happen once
when the RPC passes through both RBACs.

If users want to audit on both cases, then the `DENY` RBAC needs `ON_DENY` and
the `ALLOW` RBAC needs `ON_DENY_AND_ALLOW`.

Following is the table summarizing the combinations.

|Authorization Policy  |DENY RBAC          |ALLOW RBAC           |
|----------------------|-------------------|---------------------|
|NONE                  |NONE               |NONE                 |
|ON_DENY               |ON_DENY            |ON_DENY              |
|ON_ALLOW              |NONE               |ON_ALLOW             |
|ON_DENY_AND_ALLOW     |ON_DENY            |ON_DENY_AND_ALLOW    |

Again this is a subset of what users can technically do with xDS RBACs in the way
described in the earlier section. But we think this represents the most common
use cases.

### Language Specific APIs

_This section is to be done once the language-agnostic APIs are more finalized._

_What is certain for now is we need to support third-party audit logger
implementations registered by users._

## Rationale

Audit logging helps users answer the questions of "who did what, where and when?".
It is valuable for users to detect anomalous traffic patterns or monitor accesses
to certain services.

Therefore, authentication and authorization information needs to be available
in the audit context. Since gRPC authorization is based on the [RBAC policy][RBAC policy],
we design the audit logging feature around it to provide consistent APIs across
xDS and non-xDS use cases. The proposed xDS API changes should also benefit
other xDS clients which are currently lacking this feature as well.

In both xDS and non-xDS cases, we expect users to configure the audit condition
and audit loggers in language-agnostic configs and have the capability to inject
their own logger implementations via the APIs we expose. This allows users to
plug in their own audit logic while not having to restart the servers when
reconfiguring the loggers.

For auditing purposes, users should not log every incoming RPCs. In other words,
`ON_DENY_AND_ALLOW` should seldom be used. But this is added as an option to
support special cases where logging everything is required.

Regarding alternative design options, following are what we have considered in
a number of othorgonal aspects.

### Utilizing the Access Log

The [information](https://github.com/envoyproxy/envoy/blob/main/api/envoy/data/accesslog/v3/accesslog.proto)
present in the existing access log is sufficient for audit purposes. However,
access log does not happen till the response is sent and therefore does not
fit the audit logging use case well. For example, for long-living RPCs, users
would want to audit the event right after the authorization is enfoced but
access log won't necessarily happen till hours later.

### Audit Condition in the HTTP Connection Manager

As an HTTP filter chain can contain multiple RBAC filters in general, the typical
behavior that users will expect if they want to audit on allow is audit should
only happen once after the evaluation of the last RBAC filter. The gRPC
authorization policy API assumes this expectation and handles the audit condition
for the users.

In the xDS cases, however, the audit condition could be considered as something
on top of all RBAC filters and thus configured in the [HTTP Connection Manager][HttpConnectionManager proto].
We decided not to take this approach because the component managing the filter
chain would have to be aware of the last RBAC filter constantly and inform it
if performing the audit logging. In other words, this would require some heavy
lifting implementation-wise which does not make too much sense for such a
specicialized case as audit logging.

In practice, xDS users normally do not craft RBACs on their own but instead
rely on the control plane APIs, such as Istio's Authorization Policy, to apply
RBAC filters to the workloads. Whether the control plane explicitly sets the
constraint or not, the common practice is to apply at most one authorization
policy to a workload. So it generally suffices to leave the audit condition
inside individual RBACs.

To summarize, we acknowledge the possibility of multiple log entries for the
same RPC and argue this to be an uncommon scenario. Our APIs will not disallow
such configurations but also intentionally not provide any mechanism for log
entry de-duplication.

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

Implementation will first be done for C++ and Go. Following that will be Java,
and finally wrapped languages will be implemented.