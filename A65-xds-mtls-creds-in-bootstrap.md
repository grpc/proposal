A65: mTLS Credentials in xDS Bootstrap File
----
* Author(s): @markdroth
* Approver: @ejona86
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2023-05-24
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

This proposal adds support for configuring the use of mTLS for
communicating with the xDS server via the xDS bootstrap file.

## Background

[gRFC A27][A27] defines the xDS bootstrap file format for setting
the channel creds used to communicate with the xDS server, with the
initial options of `google_default` and `insecure`.  It suggested that a
general-purpose mechanism could be added to configure arbitrary channel
creds types, which has subsequently been done, albeit not intended as
a public API.

[gRFC A29][A29] describes the certificate provider framework used for
configuring mTLS for data plane communication in xDS.  It also defines
the file-watcher certificate provider.

This proposal defines a new channel creds type to be used in the
bootstrap file for using mTLS with the xDS server, leveraging the
functionality of the file-watcher certificate provider.

### Related Proposals: 
* [gRFC A27: xDS-Based Global Load Balancing][A27]
* [gRFC A29: xDS-Based Security for gRPC Clients and Servers][A29]

## Proposal

We will define a new credential type in the bootstrap file called `tls`.
Its configuration will be the same as that of the file-watcher
certificate provider described in [gRFC A29][A29].  Specifically, the
config will look like this:

```json
{
  // Path to identity certificate file.
  // If set, mTLS will be used; if unset, normal TLS will be used.
  "certificate_file": <string>,

  // Paths to private key and CA certificate files.
  // If either of these fields are set, both must be set.
  // If neither is set, system-wide root certs are used.
  "private_key_file": <string>,
  "ca_certificate_file": <string>,

  // How often to re-read the certificate files.
  // Value is the JSON format described for a google.protobuf.Duration
  // message in https://protobuf.dev/programming-guides/proto3/#json.
  // If unset, defaults to 10 minutes.
  "refresh_interval": <string>
}
```

Implementations should be able to internally configure the use of the
file-watcher certificate provider for the certificate-reloading
functionality.

### Temporary environment variable protection

This feature is not enabled via I/O, so no env var protection is needed.

## Rationale

We considered phrasing the credential config in a way that exposes the
certificate provider framework directly, since that would have allowed it
to automatically support any new cert provider implementations we may
add in the future.  However, the plumbing for that turned out to be a
bit challenging, so we decided to only directly expose the file-watcher
certificate provider mechanism.  If we add other certificate providers
in the future, we can consider adding fields to expose them in this
configuration.

## Implementation

Implemented in C-core in https://github.com/grpc/grpc/pull/33234.

## Open issues (if applicable)

N/A

[A27]: A27-xds-global-load-balancing.md
[A29]: A29-xds-tls-security.md
