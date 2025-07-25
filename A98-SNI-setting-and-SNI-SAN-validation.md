A98: xDS-Based setting SNI and server certificate SAN validation
----
* Author: [Kannan Jayaprakasam](https://github.com/kannanjgithub)
* Approver: [Eric Anderson](https://github.com/ejona86)
* Status: Draft
* Implemented in:
* Last updated: 2025-07-25

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

In [A29][A29] for TLS security in xDS-managed connections, it
proposed that the `SNI` field from [UpstreamTlsContext.SNI][UTC_SNI]
would be ignored. In proposal seeks to start using this and other fields
for setting the SNI by the gRPC client.

When using `XdsChannelCredentials` for the transport, hostname validation
is turned off and instead SAN matching is performed against [UpstreamTlsContext.match_subject_alt_names][match_subject_alt_names].
This proposal adds checking of SAN against the SNI provided by the client as well.

For an overview of securing connections in the envoy proxy using SNI 
and SAN validation, see [envoy-SNI].

[UTC_SNI]: https://github.com/envoyproxy/envoy/blob/ee2bab9e40e7d7649cc88c5e1098c74e0c79501d/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L42
[A29]: A29-xds-tls-security.md
[envoy-SNI]: https://www.envoyproxy.io/docs/envoy/latest/_sources/start/quick-start/securing.rst.txt
[match_subject_alt_names]: https://github.com/envoyproxy/envoy/blob/b29d6543e7568a8a3e772c7909a1daa182acc670/api/envoy/extensions/transport_sockets/tls/v3/common.proto#L407

## Proposal
This proposal has two parts:
* Setting SNI
xDS-managed gRPC clients will set SNI for the Tls handshake for 
Tls connections using the fields from [UpstreamTlsContext][UTC]
in the CDS update.

1. If [UpstreamTlsContext][UTC] specifies the SNI to use, then
it will be used.

2. If [UpstreamTlsContext][UTC] specifies `auto_host_sni`, then
SNI will be set to the hostname, which is either the logical
DNS name for DNS clusters or the endponit hostname for EDS
clusters, as in the case of the hostname useed for authority
rewriting [Ar 81-hostname][A81-hostname].

[UTC]: https://github.com/envoyproxy/envoy/blob/ee2bab9e40e7d7649cc88c5e1098c74e0c79501d/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L29
[A81-hostname]: https://github.com/grpc/proposal/blob/4f833c5774e71e94534f72b94ee1b9763ec58516/A81-xds-authority-rewriting.md?plain=1#L85

* Server SAN validation against SNI used

If `auto_sni_san_validation` is set in the [UpstreamTlsContext][UTC] 
gRPC client will perform validation for a DNS SAN matching the SNI value 
sent.

### Related Proposals:
* [gRFC A29: xDS-Based Security for gRPC Clients and Servers][A29]
* [gRFC A81: xDS Authority Rewriting][A81]

[A29]: A29-xds-tls-security.md
[A81]: A81-xds-authority-rewriting.md

### xds_cluster_impl LB Policy Changes
#### Setting SNI
As mentioned in [A29 implementation details][A29_impl-details] the
`UpstreamTlsContext` is either passed down to child policies via
channel arguments or is put in sub-channel attribute wrapped in a
`SslContextProvider`, depending on the language. This mechanism
will be augmented in the ClusterImpl LB policy to also add the
information about SNI if any that needs to be used during the 
Tls handshake. To make this decision, the the LB policy will
make use of [UpstreamTlsContext.SNI][UTC_SNI] and the hostname
from `CdsUpdate` that is passed down by the Cds LB policy to the
ClusterImpl LB policy. In a language implementation dependent
way, this SNI value to set will be passed on to the Tls handling
code. For example, in Java, there is a `ProtocolNegotiators.ClientTlsHandler` 
that is made available the `SslContext` dynamically constructed 
based on the cert store to use as indicated by `UpstreamTlsContext`. 
The SNI value to use in the `SslContextProvider` will be made
available to the `ProtocolNegotiators.ClientTlsHandler` to use when 
creating the `SslEngine` for the transport.

[A29_impl-details]: https://github.com/grpc/proposal/blob/master/A29-xds-tls-security.md#implementation-details
[UTC_SNI]: https://github.com/envoyproxy/envoy/blob/ee2bab9e40e7d7649cc88c5e1098c74e0c79501d/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L42

#### SAN SNI validation
The server certificate validation described in [A29 SAN matching][A29_SAN-matching]
matches the Subject Alternative Names specified in the server certificate against 
[`match_subject_alt_names`][match_subject_alt_names] in `CertificateValidationContext`.
If `auto_sni_san_validation` is set in the [UpstreamTlsContext][UTC], matching will be 
performed against the SNI that was used by the client, and this validation will replace
the [`match_subject_alt_names`][match_subject_alt_names] if set. The SNI to check 
aggainst will be made available to the code performing SAN validation in a
language dependent way, for example in Java, the SNI used will be set in the 
`XdsX509TrustManager` that performs the SAN validation and is set in the `SslContext` 
used for the handshake, and this SNI will be passed on from the `SslContextProvider`
to the `XdsX509TrustManager` constructor.

[A29_SAN-matching]: https://github.com/grpc/proposal/blob/master/A29-xds-tls-security.md#server-authorization-aka-subject-alt-name-checks
[match_subject_alt_names]: https://github.com/envoyproxy/envoy/blob/b29d6543e7568a8a3e772c7909a1daa182acc670/api/envoy/extensions/transport_sockets/tls/v3/common.proto#L407
[UTC]: https://github.com/envoyproxy/envoy/blob/ee2bab9e40e7d7649cc88c5e1098c74e0c79501d/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L29

#### Temporary environment variable protection
Setting SNI and performing the SAN validation against SNI will be guarded by the `GRPC_EXPERIMENTAL_XDS_SNI`
env var. The env var guard will be removed once the feature passes interop tests.
