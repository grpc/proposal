gRPC security level negotiation between call credentials and channels.
----
* Author(s): Yihua Zhang
* Approver: Mark Roth
* Status: In review
* Implemented in: https://github.com/grpc/grpc/pull/21215
* Last updated: 2019-12-05
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/ZtXYweohOwE

## Abstract

Associate a security level (privacy+integrity, integrity-only, insecure)
to each `grpc_call_credentials` implementation to dictate the minimum level of
transport security an underlying connection should satisfy in order to be
allowed to transfer a corresponding call credential. A call credential will
be allowed to be transferred only if a channel's security level is equal to
or greater than that of call credential. If the requirement is not satisfied,
the entire call will fail.

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
greatly lowers the security bar. In more detals, UDS connections can be treated
as relatively secure (assuming a correct directory permission is set on a file path)
as a remote peer can be authenticated and data confidentiality can be guaranteed
against passive adversaries. However for local TCP, we cannot authenticate a
remote peer and thus both data confidentiality and integrity become meaningless.

### Related Proposals:

N/A

## Proposal

- We introduce a new concept - security level which represents the level of
  transport security an underlying connection provides. It is an inherent
  property of a connection and is dictated by the channel cedential used to
  create the connection. A security level can be assigned one of the three
  levels: privacy+integrity, integrity-only, and insecure, which can be ordered
  from the highest (privacy+integrity) to the lowest (insecure). The privacy+integrity
  means a connection provides both data confidentiality and integrity protection.
  The integrity-only means a connection only provides integrity protection.
  For these two levels, a remote peer is authenticated e.g., via a secure handshake.
  For an insecure connection, it does not provide any protection and a remote peer is
  not authenticated as well.

- We assign an immutable security level to each connection created with
  a channel credential: For SSL and TLS, the security level will be privacy+integrity;
  For ALTS, the security level will be privacy+integrity; For local, a UDS connection will
  be assigned privacy+integrity while a local TCP connection will be assigned
  insecure. For fake, its security level will be insecure.

- A call credential should have an ability to dictate the minimum security level of
  a connection over which it will be communicated. It is an inherent property that
  should be determined by its implementation, not the application that uses it.
  If we give applications the power of setting security level, there will be risks that
  applications could set it incorrectly either intentionally or by mistake.
  Furthermore, by treating the security level as an inherent property the
  behavior will be consistent with gRPC Java and Go implementations.
  In case of a composite call credential, its security level should be set
  to the highest level of any of its individual call credentials.

- Before sending a call credential over a connection, we will extract the security levels
  of call credential and the connection and check if the security level of
  the connection is greater than or equal to that of call credential and allow
  the transfer of call credential only if the above check succeeds. If the connection
  does not provide the required security level, the entire call will fail.


### Add enum `grpc_security_level` to `include/grpc/grpc_security_constants.h`

``` C

/* Security level of grpc transport security */
typedef enum {
  GRPC_SECURITY_MIN = 0,
  GRPC_SECURITY_NONE = GRPC_SECURITY_MIN,
  GRPC_INTEGRITY_ONLY = 5,
  GRPC_PRIVACY_AND_INTEGRITY = 10,
  GRPC_SECURITY_MAX = 15,
} grpc_security_level;

```
### Associate a default security level - `GRPC_PRIVACY_AND_INTEGRITY` to all call credentials used to access Google APIs.

Regarding backward compatibility, since existing channel credentials (SSL,
TLS, ALTS, local UDS) provide privacy+integrity security level, any call
credentials used to be transferred over the connections created with those
credentials will not be affected. For local TCP, the call credential transfer
will not be allowed any more but since the local credential is currently
experimental, we reserve the right to make the change. For insecure connections,
there is currently no mechanism to allow call credentials to be transferred
over them and thus, they will not get affected as well.

Notice that we allow security levels to be configurable for plugin credentials
since the plugin credentials will be inherited by other call credential
implementations which will define their own security levels and without this
configuring capability they will all end up with the same security level.

## Implementation
All the relevant details are described in the proposal section above.
