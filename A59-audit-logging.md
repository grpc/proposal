A59: gRPC Audit Logging
----
* Author(s): [Luwei Ge](https://github.com/rockspore)
* Approver: [Eric Anderson](https://github.com/ejona86)
* Status: In Review {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in:
* Last updated: 2023-03-23
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

|Fields             |Additional Information                                                                |Example                |
|-------------------|--------------------------------------------------------------------------------------|-----------------------|
|RPC Method         |Method the RPC hit.                                                                   |"/pkg.service/foo"     |
|Principal          |Identity of the RPC. Currently only available in certificate-based TLS authentication.|"spiffe://foo/user1"   |
|Policy Name        |The authorization policy name (or the xDS RBAC filter name).                          |"example-policy"       |
|Matched Rule       |The matched rule or the matched policy name in [RBAC][RBAC policy]. Empty if no match.|"admin-access"         |
|Authorized         |A boolean indicating whether the RPC is authorized or not.                            |true                   |

### xDS API Changes

xDS API changes are needed because gRPC authorization is entirely based on [xDS RBAC policy][RBAC policy].
As a feature on top of authorization, audit logging should not be independent
from xDS and cause any divergence between gRPC authorization API and its xDS
RBAC filter support. Moreoever, audit logging is a common feature that other
xDS clients (namely Envoy) should benefit from as well.

We will add an audit condition enum (see [PR#1](https://github.com/envoyproxy/envoy/pull/26001) and [PR#2](https://github.com/envoyproxy/envoy/pull/26415))
in the [xDS RBAC policy][RBAC policy] as below:

```proto
package envoy.config.rbac.v3;

message RBAC {
  ...

  message AuditLoggingOptions {
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

    message AuditLoggerConfig {
      // Typed logger configuration.
      //
      // [#extension-category: envoy.rbac.audit_loggers]
      core.v3.TypedExtensionConfig audit_logger = 1;

      // If true, when the logger is not supported, the data plane will not NACK but simply ignore it.
      bool is_optional = 2;
    }

    // Condition for the audit logging to happen.
    // If this condition is met, all the audit loggers configured here will be invoked.
    AuditCondition audit_condition = 1 [(validate.rules).enum = {defined_only: true}];

    // Configurations for RBAC-based authorization audit loggers.
    repeated AuditLoggerConfig logger_configs = 2;
  }

  // Audit logging options that include the condition for audit logging to happen
  // and audit logger configurations.
  AuditLoggingOptions audit_logging_options = 3;
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
loggers will be run after the response is sent. It's a repeated field here since
allowing multiple audit loggers is useful when users plan to roll out a new
logger. By also keeping the existing logger along for a period they will be able
to prevent interruption.

The RBAC filter will be NACKed if the configured logger types are not supported
by gRPC, unless the filter it made optional. This is consistent with [A39: xDS HTTP Filter Support](https://github.com/grpc/proposal/blob/master/A39-xds-http-filters.md).

Some readers may immediately notice that since an HTTP filter chain can contain
an arbitrary number of RBAC filters, more than one of them could meet its audit
condition for the same RPC and multiple audit log entries occur. When this happens,
it would be impossible to uniquely identify a specific RPC from a log entry so as
to apply any sort of de-duplication. The design in this gRFC does not intend to
address this issue, the rationale of which will be elaborated more in
[one of the design alternatives](#audit-condition-in-the-http-connection-manager).

### gRPC Authorization Policy Changes

The gRPC authorization policy consists of `deny_rules` and `allow_rules`, which
are mapped to two RBACs. Overall it's a subset of the RBAC as stated in [A43: gRPC authorization API][A43].
We will keep this philosophy by adding just one `audit_logging_options` in the
policy instead of having separate ones for those two sets of rules.

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
          "description": "The name of the audit logger type.",
          "type": "string",
        },
        "config": {
          "description": "The typed config for the audit logger."
            "This needs to be a json object mapped from google.protobuf.Struct"
            "proto message.",
          "type": "object",
        },
        "is_optional": {
          "description": "Whether this logger config is optional."
            "If so, gRPC will ignore invalid or unsupported configs instead of failing to start",
          "type": "boolean"
        }
      },
    }
  },
  "properties": {
    ...
    "audit_logging_options": {
      "description": "Audit logging options that include the condition for audit"
        "logging to happen and audit logger configurations.",
      "type": "object",
      "properties": {
        "audit_condition": {
          "description": "Condition for the audit logging to happen."
            "Same as NONE if ommited.",
          "type": "string",
          "enum": ["NONE", "ON_DENY", "ON_ALLOW", "ON_DENY_AND_ALLOW"]
        },
        "audit_logger": {
          "description": "Audit logger configurations",
          "type": "array",
          "items": {
            "$ref": "#/definitions/audit_logger"
          }
        }
      }
    },
  },
  "required": ["name", "allow_rules"]
}
```

Note that the `name` field in `audit_logger` is essentially the logger type.
This is not to be confused with the `name` field in [TypedExtensionConfig](https://github.com/cncf/xds/blob/main/xds/core/v3/extension.proto).
For readers that are more familiar with proto messages, the change translates
to:

```proto
message AuthorizationPolicy {
  ...

  message AuditLoggingOptions {
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
      bool is_optional = 3;
    }
    
    repeated AuditLogger audit_loggers = 2;
  }

  AuditLoggingOptions audit_logging_options = 4;
}
```

Note that the definition above is only for illustration purposes and does not
actually exist anywhere as a `.proto` file. Nor does gRPC process the policy
as protobuf by any means.

Similar to the xDS case, unsupported logger types or any other misconfiguration
will cause the server to fail to start.

The authorization policy is backed by two RBAC filters, a DENY followed by an
ALLOW. The audit logger configurations will be duplicated in both generated
RBAC filters. How the audit condition gets translated into two conditions in
those RBACs is less straightforward and so is explained below.

First of all, note that the RBAC with `DENY` is placed before the RBAC with
`ALLOW`. We assume users will want to audit one particular RPC exactly once
if the condition is met.

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

### Built-in logger types

We plan to implement the stdout logger as a built-in logger type. The type is
named `stdout_logger`. This logger will not support any configuration and it
outputs log entries to stdout in JSON format.

Here is the example JSON configuration in the xDS case (see [PR](https://github.com/envoyproxy/envoy/pull/26453)).

```json
{
  "name": "user-defined-logger-name",
  "typed_config": {
    "@type": "envoy.extensions.rbac.audit_loggers.stream.v3.StdoutAuditLog",
  }
}
```

Here is the example in the authorization policy case.

```json
{
  "name": "stdout_logger"
}
```

Following is an example log entry.

```json
{"timestamp":"1649376211","rpc_method":"/pkg.Service/Foo","principal":"spiffe://foo/user1","policy_name":"example_policy","matched_rule":"admin_access","authorized":true}
```

The timestamp, as the number of seconds from the Unix Epoch, is generated from
the current time when the logger is invoked.

More types of loggers may be designed and implemented in the future. For users
that just need to use the built-in loggers, everything will be configured in
either the xDS RBAC filter or gRPC authorization policy. No additional code is
required.

### Language Specific APIs for third-party logger implementation

Some users may want to have their own audit logging logic which built-in
loggers do not fullfil. This section is specifically about public APIs we
will expose for users to implement their own types of audit loggers.

#### C++ APIs

A new header `audit_logging.h` declares all the classes users either need to
consume or implement.

```C++
// include/grpcpp/security/audit_logging.h

namespace grpc {
namespace experimental {

// This class contains useful information to be consumed in an audit logging
// event. It does not own any of the information so users should make copies
// when they need to use them outside the scope of this object.
class AuditContext {
 public:
  grpc::string_ref rpc_method() const;
  grpc::string_ref principal() const;
  grpc::string_ref policy_name() const;
  grpc::string_ref matched_rule() const;
  bool authorized() const;
};

// The base class for audit logger implementations.
// Users are expected to inherit this class and implement the Log() function.
class AuditLogger {
 public:
  // This function will be invoked synchronously when applicable during the
  // RBAC-based authorization process. It does not return anything and thus will
  // not impact whether the RPC will be rejected or not.
  // Implementers should ensure this method does not block the RPC. Specifically,
  // time-consuming processes should make a copy of the audit context and be fired
  // asynchronously such that this function itself can return immediately. 
  virtual void Log(const AuditContext& audit_context) = 0;
};

// The base class for audit logger factory implementations.
// Users should inherit this class and implement those declared virtual
// funcitons.
class AuditLoggerFactory {
 public:
  // The base class for the audit logger config that the factory parses.
  // Users should inherit this class to define the configuration needed for
  // their custom loggers.
  class Config {
   public:
    virtual const char* name() const = 0;
    virtual std::string ToString() = 0;
  };
  virtual const char* name() const = 0;

  // This is used to parse and validate the json format of the audit logger
  // config.
  virtual absl::StatusOr<std::unique_ptr<Config>> ParseAuditLoggerConfig(
      grpc::string_ref config_json) = 0;

  // This creates an audit logger instance given the logger config.
  // The config will be guaranteed by the caller to have been validated, so
  // implementers need to ensure this function always creates a valid instance.
  // Any runtime issues such as failing to open a file should be handled by
  // the logger implementation.
  virtual std::unique_ptr<AuditLogger> CreateAuditLogger(
      std::unique_ptr<AuditLoggerFactory::Config>) = 0;
};

// Registers an audit logger factory. This should only be called during
// initialization, i.e. before starting up the gRPC server.
void RegisterAuditLoggerFactory(std::unique_ptr<AuditLoggerFactory> factory);

}  // namespace experimental
}  // namespace grpc
```

#### Go APIs

The APIs will be in the existing `authz` package.

```Go
package authz

// RegisterAuditLoggerBuilder registers the builder in a global map
// using b.Name() as the key.
// This should only be called during initialization time (i.e. in an init() function).
// If multiple builders are registered with the same name, the one registered last
// will take effect.
func RegisterAuditLoggerBuilder(b AuditLoggerBuilder)

// GetAuditLoggerBuilder returns a builder with the given name.
// It returns nil if the builder is not found in the registry.
func GetAuditLoggerBuilder(name string) AuditLoggerBuilder

// AuditInfo contains information used by the audit logger during an audit logging event.
type AuditInfo struct {
	// RPCMethod is the method of the audited RPC, in the format of "/pkg.Service/Method".
	// For example, "/helloworld.Greeter/SayHello".
	RPCMethod string
	// Principal is the identity of the RPC. Currently it will only be available in
	// certificate-based TLS authentication.
	Principal string
	// PolicyName is the authorization policy name (or the xDS RBAC filter name).
	PolicyName string
	// MatchedRule is the matched rule (or policy name in the xDS RBAC filter). It will be
	// empty if there is no match.
	MatchedRule string
	// Authorized indicates whether the audited RPC is authorized or not.
	Authorized bool
}

// AuditLoggerConfig defines the configuration for a particular implementation of audit logger.
type AuditLoggerConfig interface {
	// auditLoggerConfig is a dummy interface requiring users to embed this
	// interface to implement it.
	auditLoggerConfig()
}

// AuditLogger is the interface for an audit logger.
// An audit logger is a logger instance that can be configured to use via the authorization policy
// or xDS HTTP RBAC filters. When the authorization decision meets the condition for audit, all the
// configured audit loggers' Log() method will be invoked to log that event with the AuditInfo.
// The method will be executed synchronously before the authorization is complete and the call is
// denied or allowed.
type AuditLogger interface {
	// Log logs the auditing event with the given information.
  // This method will be executed synchronously by gRPC so implementers must keep in mind it should
	// not block the RPC. Specifically, time-consuming processes should be fired asynchronously such
	// that this method can return immediately.
	Log(context.Context, *AuditInfo) error
}

// AuditLoggerBuilder is the interface for an audit logger builder.
// It parses and validates a config, and builds an audit logger from the parsed config. This enables
// configuring and instantiating audit loggers in the runtime.
// Users that want to implement their own audit logging logic should implement this along with
// the AuditLogger interface and register this builder by calling RegisterAuditLoggerBuilder()
// before they start the gRPC server.
type AuditLoggerBuilder interface {
	// ParseAuditLoggerConfig parses the given JSON bytes into a structured
	// logger config this builder can use to build an audit logger.
	// When users implement this method, its return type must embed the
	// AuditLoggerConfig interface.
	ParseAuditLoggerConfig(config json.RawMessage) (AuditLoggerConfig, error)
	// Build builds an audit logger with the given logger config.
	// This will only be called with valid configs returned from ParseAuditLoggerConfig()
	// and any runtime issues such as failing to create a file should be handled by the
	// logger implementation instead of failing the logger instantiation. So implementers
	// need to make sure it can return a logger without error at this stage.
	Build(AuditLoggerConfig) AuditLogger
	// Name returns the name of logger built by this builder.
	// This is used to register and pick the builder.
	Name() string
}
```

#### Java APIs

```Java
public class AuditContext {
  public String getRpcMethod();
  public String getPrincipal();
  public String getPolicyName();
  public String getMatchedRule();
  public boolean isAuthorized();
}

public interface AuditLogger {
  public abstract void audit(AuditContext context) throws Exception;
}

public interface AuditLoggerProvider {
  ConfigOrError parseAuditLoggerConfig(
      Map<String, ?> rawAuditLoggerConfig) {
    return UNKNOWN_CONFIG;
  }

  AuditLogger newAuditLogger(Object config);
  String getName();
}

public final class AuditLoggerRegistry {
  public synchronized void register(AuditLoggerProvider provider) {}
  public synchronized AuditLoggerProvider getProvider(String name) {}
}
```

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

Note that there has been ongoing effort in Envoy to support flushing access
logs at different lifecycle points of a request (see [example](https://github.com/envoyproxy/envoy/pull/26094)).
It would be a valid approach if the access log is generalized to support what
is needed here. But gRPC does not and has no plan yet to support access log,
due to the lack of such need. Therefore, we decided not to take this approach
which would require more engineering effort.

### Audit Condition in the HTTP Connection Manager

As an HTTP filter chain can contain multiple RBAC filters in general, the typical
behavior that users will expect if they want to audit on allow is audit should
only happen once after the evaluation of the last RBAC filter. The gRPC
authorization policy API assumes this expectation and handles the audit condition
for the users.

In the xDS cases, however, the audit condition could be considered as something
on top of all RBAC filters and thus configured in the [HTTP Connection Manager][HttpConnectionManager proto].
We decided not to take this approach because the component managing the filter
chain would have to be aware of the last RBAC filte and inform it if performing
the audit logging. In other words, this would require more engineering effort
which does not make too much sense for such a particular case as audit logging.

In practice, xDS users normally do not craft RBACs on their own but instead
rely on the control plane APIs, such as Istio's Authorization Policy, to apply
RBAC filters to the workloads. We hope that when these control plane APIs
start to support audit logging, they will only have a single RBAC policy o
will handle the logging behavior as what we do in the gRPC authorization policy
case.

To summarize, we acknowledge the possibility of multiple log entries for the
same RPC and argue it to be uncommon with carefully designed control plane APIs.
At the xDS RBAC level, we will not disallow such configurations but also
intentionally not provide any mechanism for log entry de-duplication.

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