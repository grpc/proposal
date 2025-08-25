A101: xDS-Based setting SNI and server certificate SAN validation
----
* Author: [Kannan Jayaprakasam](https://github.com/kannanjgithub)
* Approver: [Eric Anderson](https://github.com/ejona86)
* Status: Draft
* Implemented in:
* Last updated: 2025-07-28

## Abstract

gRPC will add support for setting Server Name Indication (SNI) and validation of server certificate's
Subject Alternative Names (SANs) aginst the SNI that was used.

### Background

During Tls handshake, the server presents its certificate to the client for authentication. For servers
serving multiple domains, the client needs to indicate which domain it is requesting, so that the server
can present the certificate is has for that domain. The client does this at the time of Tls handshaking
via Server Name Indication (SNI). When using `XdsChannelCredentials` for a channel, the gRPC client needs
to be configured by the xDS server with what value to send for SNI and the gRPC client should use it for
the Tls handshake.

In [A29][A29] for TLS security in xDS-managed connections, the `sni` field from [UpstreamTlsContext.sni][UTC_SNI]
was ignored. 

When using `XdsChannelCredentials` for the channel, hostname validation
is turned off and instead SAN matching is performed against [UpstreamTlsContext.match_subject_alt_names][match_subject_alt_names]
instead of a typical hostname. This proposal adds SAN matching for the same name as the client used for SNI.

For an overview of securing connections in the envoy proxy using SNI 
and SAN validation, see [envoy-SNI].

[UTC_SNI]: https://github.com/envoyproxy/envoy/blob/ee2bab9e40e7d7649cc88c5e1098c74e0c79501d/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L42
[A29]: A29-xds-tls-security.md
[envoy-SNI]: https://www.envoyproxy.io/docs/envoy/latest/start/quick-start/securing
[match_subject_alt_names]: https://github.com/envoyproxy/envoy/blob/b29d6543e7568a8a3e772c7909a1daa182acc670/api/envoy/extensions/transport_sockets/tls/v3/common.proto#L407

## Proposal
This proposal has two parts:
1. Setting SNI: When using `XdsChannelCredentials` for the channel, gRPC clients will set SNI for the Tls handshake for 
Tls connections using the fields from [UpstreamTlsContext][UTC] in the CDS update.    

    i. If [UpstreamTlsContext][UTC] specifies `auto_host_sni`, then SNI will be set to the hostname, which is either the DNS name for
logical DNS clusters or the endpoint hostname for EDS clusters, as in the case of the hostname used for [authority rewriting][A81-hostname].

   ii. If `UpstreamTlsContext.sni` specifies the SNI to use, then it will be used.

   iii. Otherwise no SNI will be set for the Tls handshake.

[UTC]: https://github.com/envoyproxy/envoy/blob/ee2bab9e40e7d7649cc88c5e1098c74e0c79501d/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L29
[A81-hostname]: A81-xds-authority-rewriting.md#xds-resource-validation

2. Server SAN validation against SNI used: If `auto_sni_san_validation` is true in the [UpstreamTlsContext][UTC] 
gRPC client will perform validation for a DNS SAN matching the SNI value 
sent. The normal matching when using `TlsCredentials` for the channel 
allows other SAN types, but only the DNS type will be checked here.

### Related Proposals:
* [gRFC A29: xDS-Based Security for gRPC Clients and Servers][A29]
* [gRFC A81: xDS Authority Rewriting][A81]

[A29]: A29-xds-tls-security.md
[A81]: A81-xds-authority-rewriting.md

### Setting SNI during Tls handshake
As mentioned in [A29 implementation details][A29_impl-details] the `UpstreamTlsContext` is either 
passed down to child policies via channel arguments or a similar mechanism, depending on the language, 
and the SslContext is instantiated using the truststore location indicated by the `UpstreamTlsContext`. 
This SslContext is then used to initiate the Tls handshake for the transport and this is when the SNI is sent
for the `ClientHello` frame of the handshake. To determine the SNI, we need both the `UpstreamTlsContext` and 
the hostname for the endpoint. The hostname attribute is already stored in the subchannel wrapper by the 
xds_cluster_impl policy when its child policy creates a subchannel. Once the `SslContext` is available during the 
Tls handshake phase of the transport creation (the creation of which depends on the choice of the certificate provider 
infra to use as indicated by the `UpstreamTlsContext`), the fields from `UpstreamTlsContext` and the hostname 
from the channel attributes will be used to determine the SNI to set for the handshake.

##### Language specific example
As an example, in Java, the ClusterImpl LB policy creates the `SslContextProviderSuppler` wrapping the
`UpstreamTlsContext` and puts it in the subchannel wrapper when its child policy creates a subchannel. At the time of Tls protocol negotiation
for the subchannel, the hostname from the channel attributes also should be passed to this provider supplier to determine the SNI to be set for 
the Tls handshake. The hostname will be set in the callback object that is given to the `SslContextProviderSupplier`, to be invoked with the 
`SslContext` when the client Ssl Provider instantiated by this supplier has the `SslContext` available. This value along with the 
`UpstreamTlsContext` available in the `SslContextProviderSupplier` will be used to decide the SNI to be used for the handshake.

[A29_impl-details]: https://github.com/grpc/proposal/blob/master/A29-xds-tls-security.md#implementation-details
[UTC_SNI]: https://github.com/envoyproxy/envoy/blob/ee2bab9e40e7d7649cc88c5e1098c74e0c79501d/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L42

### SAN SNI validation
The server certificate validation described in [A29 SAN matching][A29_SAN-matching]
matches the Subject Alternative Names specified in the server certificate against 
[`match_subject_alt_names`][match_subject_alt_names] in `CertificateValidationContext`.
If `auto_sni_san_validation` is set in the [UpstreamTlsContext][UTC], matching will be 
performed against the SNI that was used by the client, and this validation will replace
the [`match_subject_alt_names`][match_subject_alt_names] if set. This verification occurs
in the TrustManager of the SslContext which is created using the cert store indicated by 
`CertificateValidationContext` in `UpstreamTlsContext` which is either a managed cert store
or the system root cert store. 

#### Caching for the SslContext 
The `SslContextProviderSupplier` (named so because it supplies both client and server
SslContext providers) creates a provider for the client `SslContext` and today 
maintains a cache of `UpstreamTlsContext` to the client `SslContext` provider instances. 
For the SNI requirement, the `TrustManager` in the `SslContext` needs to 
be aware of the SNI to validate the SAN against, so a different `TrustManager` instance needs 
to be created for each SNI to use for the same `UpstreamTlsContext`, so this cache's key will 
need to be enhanced to be <UpstreamTlsContext, String> to hold the SNI as well, and the client
`SslContext` provider for a particular key will create a `TrustManager` instance that takes the 
SNI to validate the SANs against and set it in the `SslContext` it provides.

[A29_SAN-matching]: A29-xds-tls-security.md#server-authorization-aka-subject-alt-name-checks
[match_subject_alt_names]: https://github.com/envoyproxy/envoy/blob/b29d6543e7568a8a3e772c7909a1daa182acc670/api/envoy/extensions/transport_sockets/tls/v3/common.proto#L407
[UTC]: https://github.com/envoyproxy/envoy/blob/ee2bab9e40e7d7649cc88c5e1098c74e0c79501d/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L29

#### Behavior when SNI is not indicated in UpstreamTlsContext
When `UpstreamTlsContext` has neither of `SNI` nor `auto_sni_host` values set, the current behavior will continue, i.e. SNI will be set to the xds hostname from `GrpcRoute`.

#### Validation
The Cds update will be NACKed if `UpstreamTlsContext.sni` exceeds 255 characters, similar to Envoy.

### Temporary environment variable protection
Setting SNI and performing the SAN validation against SNI will be guarded by the `GRPC_EXPERIMENTAL_XDS_SNI`
env var. The env var guard will be removed once the feature passes interop tests.
