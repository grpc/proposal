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

    i. If [UpstreamTlsContext][UTC] specifies `auto_host_sni` and the hostname is available, then SNI will be set to the hostname. The hostname
    is either the DNS name for logical DNS clusters or the endpoint hostname for EDS clusters, as in the case of the hostname used for [authority rewriting][A81-hostname].

   ii. Else, if `UpstreamTlsContext.sni` specifies the SNI to use, then it will be used.

   iii. Else, no SNI will be set for the Tls handshake.

[UTC]: https://github.com/envoyproxy/envoy/blob/ee2bab9e40e7d7649cc88c5e1098c74e0c79501d/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L29
[A81-hostname]: A81-xds-authority-rewriting.md#xds-resource-validation

2. Server SAN validation against SNI used: If `auto_sni_san_validation` is true in the [UpstreamTlsContext][UTC] 
gRPC client will perform matching for a SAN against the SNI used for the handshake. The normal matching when using
`TlsCredentials` for the channel only checks against DNS SANs in the certificate, but with `XdsChannelCredentials`
matching will be done using any of DNS / URI / IPA SAN types in the server certificate.

### Related Proposals:
* [gRFC A29: xDS-Based Security for gRPC Clients and Servers][A29]
* [gRFC A81: xDS Authority Rewriting][A81]

[A29]: A29-xds-tls-security.md
[A81]: A81-xds-authority-rewriting.md

### Setting SNI during Tls handshake
As mentioned in [A29 implementation details][A29_impl-details] the `UpstreamTlsContext` is either 
passed down to child policies via channel arguments or a similar mechanism, depending on the language.
[A29 implementation details][A29_impl-details] also talks about a `CertificateProvider` object that represents 
a plugin that provides the required certificates and keys to the gRPC implementation. When Tls handshake is
initiated for a channel that is using `XdsCredentials`, this `CertificateProvider` object is used to
provide the certs and trust roots for establishing the secure connection. During this handshake we need 
to set the SNI to use for the `ClientHello` frame of the handshake. To determine the SNI, we need both the 
SNI related fields from the parsed `UpstreamTlsContext` and the hostname for the endpoint. 
To determine the SNI `UpstreamTlsContext.sni` and `UpstreamTlsContext.auto_host_sni` from the parsed
cluster resource will also be set into the `CertificateProvider` by the xds_cluster_impl policy. 
When the Tls handling code uses the certs and trust roots from the `CertificateProvider`
to establish the connection, it will also now determine the SNI to set based on the parsed sni related fields
available in the `CertificateProvider` and the hostname in the endpoint attributes.
The precedence order mentioned at the top of the [Proposal](#proposal) section will be used to determine the SNI to use. For example,
if `UpstreamTlsContext.auto_host_sni` was set but there is no EDS hostname for the endpoint, but 
`UpstreamTlsContext.sni` is set, then it would use the value of the `UpstreamTlsContext.sni` if set. 
If no SNI value is determined, then it will not set SNI for the Tls handshake.

##### Language specific example
As an example, in Java, the ClusterImpl LB policy creates the `SslContextProviderSuppler` wrapping the
`UpstreamTlsContext` and puts it in the subchannel wrapper when its child policy creates a subchannel. At the time of Tls protocol negotiation
for the subchannel, the Tls handling code should use the hostname from the endpoint address attributes and the SNI related fields in `UpstreamTlsContext`
to determine the SNI to be used for the Tls handshake. This SNI will also be passed to the the `SslContextProviderSupplier`, in addition to the 
callback to be invoked to provide the `SslContext` when it is available. The `ClientCertificateSslContextProvider` instantiated by the `SslContextProviderSupplier`
will be passed both the callback argument and the SNI value to use, that will be used in the `XdsX509TrustManager` it creates to perform the SAN - SNI
matching. The Tls protocol negotiating code will use the SNI value determined to use when creating the the SSL engine from the `SslContext` received via the callback.

[A29_impl-details]: A29-xds-tls-security.md#implementation-details
[UTC_SNI]: https://github.com/envoyproxy/envoy/blob/ee2bab9e40e7d7649cc88c5e1098c74e0c79501d/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L42

### SAN SNI validation
The server certificate validation described in [A29 SAN matching][A29_SAN-matching]
matches the Subject Alternative Names specified in the server certificate against 
[`match_subject_alt_names`][match_subject_alt_names] in `CertificateValidationContext`.
If `auto_sni_san_validation` is set in the [UpstreamTlsContext][UTC], matching will be 
performed against the SNI that was used by the client, and this validation will replace
the [`match_subject_alt_names`][match_subject_alt_names] if set. The value of the 
`auto_sni_san_validation` field and the SNI used by the client will need to be propagated
to the certificate verifying mechanism that is used based on the settings in the 
`CertificateProvider` when using `XdsChannelCredentials` for the transport.
The SNI used by the client will be used for matching, regardless of how that SNI was determined.

#### Language specific example
For example in Java the SAN SNI validation verification occurs in the TrustManager created by the `CertProviderClientSslContextProvider` using 
the cert store indicated by `CertificateValidationContext` in `UpstreamTlsContext` which is either a managed cert store or the system root cert store. 

gRPC Java also has a Caching for the SslContext. The `SslContextProviderSupplier` (named so because it 
supplies both client and server SslContext providers) creates a provider for the client `SslContext` and today 
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

#### Validation
The Cds update will be NACKed if `UpstreamTlsContext.sni` exceeds 255 characters, similar to Envoy.

### Environment variable protections
Setting SNI and performing the SAN validation against SNI will be guarded by the `GRPC_EXPERIMENTAL_XDS_SNI`
env var. The env var guard will be removed once the feature passes interop tests.
When the SNI value to be used for the Tls handshake is not determined based on the described rules, no SNI will be
sent. Some language implementations are sending the xds channel authority today, and some customers may see
breaking behavior if no SNI is sent now. To mitigate this, an env var `GRPC_USE_CHANNEL_AUTHORITY_IF_NO_SNI_APPLICABLE`
will be provided to revert back to the old behavior of sending the xds channel authority when no SNI is determined.
