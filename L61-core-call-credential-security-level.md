gRPC security level negotiation between call credentials and channels.
----
* Author(s): Yihua Zhang
* Approver: Mark Roth
* Status: In review
* Implemented in: https: //github.com/grpc/grpc/pull/21215
* Last updated: 2019-12-05
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/ZtXYweohOwE

## Abstract

Associate a security level (privacy+integrity, integrity-only, insecure)
to each `grpc_call_credentials` implementation to dictate the level of
transport security an underlying connection should satisfy in order to transfer
a corresponding call credential. Whenever a call credential is about to be
transferred on a channel we check if the security level of channel is greater than or
equal to that of call credential and fail the transfer if it is not.

## Background

In gRPC Java and Go, each call credential can dictate whether or not
a secure channel (i.e., ensuring both privacy and integrity) is needed
for its communication. The use of secure channel is mandatory if a call
credential is used to access Google APIs. In gRPC C core and its wrapped
languages, a call credential can only be communicated over a secure channel
which is also acceptable from the security perspective. However with the
introduction of local credential that can be used in both UDS and TCP loopback
connections, any call credential (including the ones used to access Google APIs)
can be transferred over a TCP loopback connection that is considered insecure, which
greatly lowers the security bar.

### Related Proposals:

N/A

## Proposal

- A call credential should have an ability to dictate the security level of
  a connection over which it will be communicated. It is an inherent property that
  should be determined by its implementation, not the application that uses it.
  The security level should be one of privacy+integrity, integrity-only, and insecure.
  In case of a composite call credential, its security level should be set to the highest
  level of any of its individual call credentials.

- A security level should be also associated to each connection created by a channel
  credential to indicate the level of transport security that connection is granted.
  For a local connection, its security level will be set to privacy+integrity for UDS and
  insecure for TCP loopback.


- Before sending a call credential over a connection, we will extract the security levels
  of call credential and the connection and check if the security level of
  connection is greater than or equal to that of call credential and only allow
  the transfer of call credential if the above check succeeds.


### Add enum `grpc_security_level` to `include/grpc/grpc_security_constants.h`

``` C

/* Security level of grpc transport security */
typedef enum {
  GRPC_SECURITY_MIN,
  GRPC_SECURITY_NONE = GRPC_SECURITY_MIN,
  GRPC_INTEGRITY_ONLY,
  GRPC_PRIVACY_AND_INTEGRITY,
  GRPC_SECURITY_MAX = GRPC_PRIVACY_AND_INTEGRITY,
} grpc_security_level;

```
### Associate security level - `GRPC_PRIVACY_AND_INTEGRITY` to all call credentials used to access Google APIs.

It will be realized by hard-coding `GRPC_PRIVACY_AND_INTEGRITY` in the
default `grpc_call_credentials` constructor.

### Add a new constructor for `grpc_call_credentials` that takes `grpc_security_level` as a parameter.

``` C

grpc_call_credentials(const char* type, grpc_security_level level);

```

### Add a new constructor for `grpc_plugin_credentials` that takes `grpc_security_level` as a parameter.

``` C

grpc_plugin_credentials(grpc_metadata_credentials_plugin plugin, grpc_security_level level);

```

### Add a new `grpc_plugin_credentials` creation API that takes `grpc_security_level` as a parameter.

``` C

GRPCAPI grpc_call_credentials* grpc_metadata_credentials_create_from_plugin(
    grpc_metadata_credentials_plugin plugin, grpc_security_level level, void* reserved);

```

## Rationale
Notice that we only allow security level to be configurable for plugin credentials
since it may have different implementations that require different security
levels (e.g., privacy+integrity or integrity-only).

## Implementation
All the relevant details are described in the proposal section above
