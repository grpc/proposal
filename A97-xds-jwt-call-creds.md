A97: xDS JWT Call Credentials
----
* Author(s): @markdroth
* Approver: @ejona86, @dfawley
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2025-06-04
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

This document proposes a new call credential type for JWT tokens and the
ability to configure that call credential for use in calls to the xDS
server via the xDS bootstrap config.

## Background

As part of xDS fallback support ([A71]), gRPC switched from a single,
globally shared XdsClient instance shared across all channels to
a separate XdsClient for each data plane target.  This means that
if the application creates channels to multiple targets, the client
will now create multiple concurrent connections to the xDS server.
Unfortunately, this change has broken the ability to use proxyless gRPC
in Istio due to a limitation in Istio's xDS proxy of supporting only one
concurrent xDS client connection at a time, as per istio/istio#53532.

Istio does not want to fix this limitation in their proxy, so we need a
way for gRPC to work without the proxy.  The only reason that gRPC uses
the Istio proxy is that it allows gRPC to connect to the proxy locally
using plaintext, and let the proxy figure out the authentication details
for talking to the Istio xDS server remotely.  Therefore, we need gRPC
to handle those authentication details directly.

Talking to the Istio xDS server requires two things.  First, it requires
configuring TLS credentials, as introduced in [A65].  And second, it
requires use of a JWT token as a call credential, which is described in
this document.

### Related Proposals: 
* [A27: xDS-Based Global Load Balancing][A27]
* [A65: mTLS Credentials in xDS Bootstrap File][A65]
* [A71: xDS Fallback][A71]
* [A83: xDS GCP Authentication Filter][A83]

[A27]: A27-xds-global-load-balancing.md
[A65]: A65-xds-mtls-creds-in-bootstrap.md
[A71]: A71-xds-fallback.md
[A83]: A83-xds-gcp-authn-filter.md

## Proposal

We will add a new type of call credentials called JwtTokenCallCredentials
that loads a JWT token from a local file, with the ability to
periodically reload the token.  We will also provide hooks to configure
the use of this new call credentials type in the xDS bootstrap file,
which was originally described in [A27].

### JwtTokenCallCredentials

Note: This section is intended for gRPC implementations that need to
implement a new call credential type for loading JWT tokens from a file.
Implementations that already support this functionality may continue to
use their existing functionality, even if the behavior differs in small
ways from what is described in this section.

gRPC will support a JwtTokenCallCredentials call credentials type, which
is not xDS-specific.  The design for this call credential type is modeled
after GcpServiceAccountIdentityCallCredentials, described in [A83].

JwtTokenCallCredentials will be instantiated with one parameter, the
path to the file containing the JWT token.  The credential object will
handle loading the token on-demand and caching it based on the token's
expiration time.

To handle potential clock skew issues and to account for processing time
on the server, the credential will set the cache expiration time to be
30 seconds before the expiration time encoded in the token.  All logic in
the call credential code will use this modified expiration time instead
of the expiration time encoded in the token.

When the credential is asked for a token for a data plane RPC, if the
token is not yet cached or the cached token will expire within some
fixed refresh interval (typically 1 minute), the credential will start
re-reading it from the file.

When a data plane RPC starts, if the token is cached and is not expired,
the token will immediately be added to the RPC, and the RPC will
continue.  Otherwise (i.e., before the token is initially obtained or
after the cached token has expired), the data plane RPC will be queued
until the file read completes.  When the file read completes, the
result (either success or failure, as described below) will be applied
to all queued data plane RPCs.

Note that when the token's expiration time is less than the refresh
interval in the future, a new data plane RPC being started will trigger
a new file read, but the cached token value will still be used for
that data plane RPC.  This pre-emptive re-fetching is intended to avoid
periodic latency spikes when refreshing the token.

If reading the file fails, all queued data plane RPCs will be failed
with UNAVAILABLE status.

If reading the file succeeds, the content of the file will contain the JWT
token, which the client will cache.  The client does not need to do full
[RFC-7519](https://datatracker.ietf.org/doc/html/rfc7519) validation
of the token (that is the responsibility of the server side), but it
does need to extract the `exp` field for caching purposes. If the `exp`
field cannot be extracted (i.e., the JWT token is invalid), all queued
data plane RPCs will be failed with status UNAUTHENTICATED. Otherwise,
the cache is updated, and the returned token is added to all queued data
plane RPCs, which may then continue.

If reading the file does not result in the cache being updated (i.e.,
if reading the file fails or if it returns an invalid JWT token),
[backoff](https://github.com/grpc/grpc/blob/master/doc/connection-backoff.md)
must be applied before the next attempt may be started.  If a data plane
RPC is started when there is no cached token available and while in
backoff delay, it will be failed with the status from the last attempt
to read the file.  When the backoff delay expires, the next data plane
RPC will trigger a new attempt.  Note that no attempt should be started
until and unless a data plane RPC is started, since we do not want to
unnecessarily retry if the channel is idle.  The backoff state will be
reset once the file has been read successfully.

To add the token to a data plane RPC, the call credential will add
a header named `authorization`.  The header value will be the string
`Bearer ` (note trailing space) followed by the token value.

### Configuring Call Credentials in xDS Bootstrap File

We will add a new field to the xDS bootstrap representation of an xDS
server:

```json5
{
  "server_uri": <string containing URI of xds server>,

  // List of channel creds; client will stop at the first type it
  // supports.  This field is required and must contain at least one
  // channel creds type that the client supports.
  "channel_creds": [
    {
      "type": <string containing channel cred type>,
      // The "config" field is optional; it may be missing if the
      // credential type does not require config parameters.
      "config": <JSON object containing config for the type>
    }
  ],

  "server_features": [ ... ],

  // NEW FIELD!
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
  ],
}
```

The new "call_creds" field uses essentially the same format as the
existing "channel_creds" field, but with the following key differences:
- The supported types for call creds use a different namespace than the
  supported types for channel creds.
- Unlike channel creds, where we must configure exactly one type, we can
  use multiple call creds instances together.  Therefore, instead of
  stopping at the first supported type, the client will look through
  the entire list and use all types that it supports.
- Unlike channel creds, call creds are optional, so if the call_creds
  field is not specified or does not contain any supported call creds
  type, the client will proceed without configuring any call creds.

For now, we will support only one type of call credentials,
"jwt_token", whose configuration will look like this:

```json5
{
  // Path to JWT token file.  Required.
  "jwt_token_file": <string>,
}
```

Implementations may implement a general-purpose registry of call
credentials types, or they may simply hard-code the single supported type.

### Temporary environment variable protection

The new bootstrap fields will be guarded by the
`GRPC_EXPERIMENTAL_XDS_BOOTSTRAP_CALL_CREDS` environment variable.  We
do not plan to do interop testing for this feature, but we can remove
the env var protection once we hear from OSS users that they have
successfully used this functionality in Istio.

## Rationale

The alternative to this would have been for Istio to fix their xDS
proxy, but they did not want to do so due to concerns about complexity.

## Implementation

Will be implemented in C-core, Java, Go, and Node.
