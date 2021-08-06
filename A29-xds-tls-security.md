A29: xDS-Based Security for gRPC Clients and Servers
----
* Author(s): [Sanjay M. Pujare](https://github.com/sanjaypujare), [Easwar Swaminathan](https://github.com/easwars), [Yash Tibrewal](https://github.com/yashykt)
* Approver: markdroth
* Status: Draft
* Implemented in: C-core, Java, and Go
* Last updated: 2021-06-08
* Discussion at: 

## Abstract

This proposal seeks to add transport security (i.e. TLS or mTLS) to xDS-managed
gRPC connections. (m)TLS adds encryption and authentication capabilities to the
connections. RBAC or client authorization is not part of this proposal but is covered in
[A41:xDS RBAC Support][A41].

[A41]: https://github.com/grpc/proposal/pull/237/files

## Background

With the addition of server side xDS support
[A36: xDS-Enabled Servers][A36],
it is now possible to configure both client (for TLS origination) and server (for TLS termination)
ends of a gRPC connection from the control plane when the infrastructure provides the required
security or certificate capabilities.

xDS [Listener][] provides server side TLS configuration in
[DownstreamTlsContext][DTC] and the xDS [Cluster][] provides client side TLS configuration in
[UpstreamTlsContext][UTC]. Through these configurations, the xDS-control plane can instruct
xDS endpoints to use the infrastructure provided certificates and keys for their connections
which replaces manual provisioning and management of these certificates. We provide a
specification for interpreting and applying these configurations in gRPC.
We also extend the gRPC programming API and the xDS support in gRPC so developers can programmatically
enable (or disable) the use of these configurations.

[Listener]: https://github.com/envoyproxy/envoy/blob/main/api/envoy/config/listener/v3/listener.proto
[Cluster]: https://github.com/envoyproxy/envoy/blob/main/api/envoy/config/cluster/v3/cluster.proto

### Related Proposals

 * [A27: xDS-Based Global Load Balancing][A27]
 * [A36: xDS-Enabled Servers][A36]
 * [L74: Java Channel and Server Credentials][L74]

[A27]: A27-xds-global-load-balancing.md
[L74]: L74-java-channel-creds.md

## Proposal

### Programming API

We need a mechanism by which a programmer can "opt in" to allow the use of xDS provided security
configuration for a gRPC channel or server. This is achieved by supplying an `XdsChannelCredentials`
(to a channel) or an `XdsServerCredentials` (to a server). These two new credentials types extend
the existing channel or server credentials types in each language. Each `Xds*Credentials` needs to
be provided with _fallback credentials_. The fallback credentials are used in the following cases:

- when xDS is not in use (such as when the `xds:` scheme is not used on the client side)

- xDS is in use, but the control plane does not provide security configuration. The gRPC behavior
in this case is different from Envoy's due to fallback credentials. Envoy will
use plaintext (insecure) communication mode in this case. But with gRPC, the application
needs to use InsecureCredentials as the fallback credentials to get the same result.

Note that the fallback credentials is *not* used in case of errors. For example, if there is
an error encountered while using the xDS provided security configuration a connection will be
terminated rather than using the fallback credentials.

A user is not required to use Xds- Channel or Server Credentials even if they are
using an xDS managed channel or server. Not using Xds- Channel or Server Credentials results
in not using the xDS provided TLS configuration.

The example snippets below show the use of `XdsChannelCredentials` and `XdsServerCredentials`.
In these examples a plaintext or insecure credentials is used as the fallback credentials.

#### C++ 

Use of XdsChannelCredentials:

```C++
    std::shared_ptr<grpc::ChannelCredentials> credentials =
        grpc::XdsCredentials(grpc::InsecureChannelCredentials());
    std::shared_ptr<grpc::Channel> channel = grpc::CreateChannel(target, credentials);
```

Use of XdsServerCredentials:

```C++
   grpc::XdsServerBuilder builder;
   builder.RegisterService(&service);
   builder.AddListeningPort(listening_address,
                                 grpc::XdsServerCredentials(
                                     grpc::InsecureServerCredentials()));
   builder.BuildAndStart()->Wait();
```

#### Java

Use of XdsChannelCredentials:

```Java
    ChannelCredentials credentials = XdsChannelCredentials.create(InsecureChannelCredentials.create());
    ManagedChannel channel = Grpc.newChannelBuilder(target, credentials).build();
```

Use of XdsServerCredentials:

```Java
    ServerCredentials credentials = XdsServerCredentials.create(InsecureServerCredentials.create());
    Server server = XdsServerBuilder.forPort(port, credentials)
      .addService(new HostnameGreeter(hostname))
      .build()
      .start();
```

#### Go

Use of XdsChannelCredentials:

```Go
   import (
           xdscreds "google.golang.org/grpc/credentials/xds"
   )

   if creds, err := xdscreds.NewClientCredentials(xdscreds.ClientOptions{FallbackCreds: insecure.NewCredentials()}); err != nil {
           log.Fatalf("Failed to create xDS client credentials: %v", err)
   }
   conn, err := grpc.Dial(*target, grpc.WithTransportCredentials(creds))
```

Use of XdsServerCredentials:

```Go
   import (
           xdscreds "google.golang.org/grpc/credentials/xds"
           "google.golang.org/grpc/xds"
   )

   if creds, err := xdscreds.NewServerCredentials(xdscreds.ServerOptions{FallbackCreds: insecure.NewCredentials()}); err != nil {
           log.Fatalf("Failed to create xDS server credentials: %v", err)
   }
   server := xds.NewGRPCServer(grpc.Creds(creds))
```

### xDS Protocol

xDS v3 is a pre-requisite for this proposal because the required fields were added to xDS v3. 
gRPC uses CDS to configure client side security and LDS for server side security as described in the
sections below. The sections describe how the CDS or LDS updates are processed by gRPC. Note that gRPC
always validates all security configurations in CDS or LDS regardless of whether an XdsChannelCredentials
or XdsServerCredentials is in force. If validation fails on an update, the update is NACKed.

An unsupported field is normally ignored. However if ignoring a field compromises security, or
if the unsupported field affects how we interpret other fields, we NACK the update when the
field is present.

When an update is accepted, the security configuration contained in that update is used if the
application used Xds*Credentials.

#### CDS for Client Side Security

The CDS policy is the top level LB policy applied to all the connections under that policy.
[A27:CDS][] describes the flow. The field [`transport_socket`][CL-TS] is used to extract the
[`UpstreamTlsContext`][UTC] as described [here][CL-TS-comment].
Note that we don't (currently) support [`transport_socket_matches`][CL-TS-matches].
The security configuration extracted from the `UpstreamTlsContext` thus obtained
is passed down to all the child policies and connections similar to how gRPC passes
load balancer policy information down to child policies.

[`common_tls_context`][CTC] in the `UpstreamTlsContext` contains the required security
configuration. See below for [`CommonTlsContext`][CTC-type] processing.

A secure client at a minimum requires a CA root certificate to be able to validate the server
certificate in the TLS mode and this is provided by [ca_certificate_provider_instance][CCPI]
inside the [`default_validation_context`][default_validation_context] of
[combined_validation_context][CVC]. For backward compatibility gRPC will use the deprecated
field [`validation_context_certificate_provider_instance`][VCCPI1] inside
[`combined_validation_context`][CVC] if [ca_certificate_provider_instance][CCPI] is not
present. If [`UpstreamTlsContext`][UTC] is present but
[`combined_validation_context`][CVC] is not present or neither of
[ca_certificate_provider_instance][CCPI] and [`validation_context_certificate_provider_instance`][VCCPI1]
is set, then gRPC will NACK the CDS update. The client identity certificate is configured via
[`tls_certificate_provider_instance`][TCPI] (or the deprecated
[`tls_certificate_certificate_provider_instance`][TCCPI] if [`tls_certificate_provider_instance`][TCPI]
is not set). Note that the identity certificate on the client side is optional. The
client certificate is sent to the server if the server requests or requires it in the
TLS handshake. If the server requires the client certificate but is not configured on the
client side then the TLS handshake will fail because of the client's inability to send the
certificate.

The following fields in a CDS update are ignored:

* [sni][SNI]
* [allow_renegotiation][ALLOW-RENEG]
* [max_session_keys][MAX-SESS-KEYS]

[A27:CDS]: A27-xds-global-load-balancing.md#cds
[CL-TS]: https://github.com/envoyproxy/envoy/blob/9aca65c395f01020080166e4795455addde167fa/api/envoy/config/cluster/v3/cluster.proto#L937
[CL-TS-comment]: https://github.com/envoyproxy/envoy/blob/9aca65c395f01020080166e4795455addde167fa/api/envoy/config/cluster/v3/cluster.proto#L932
[CL-TS-matches]: https://github.com/envoyproxy/envoy/blob/9aca65c395f01020080166e4795455addde167fa/api/envoy/config/cluster/v3/cluster.proto#L680
[CTC]: https://github.com/envoyproxy/envoy/blob/b29d6543e7568a8a3e772c7909a1daa182acc670/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L37
[SNI]: https://github.com/envoyproxy/envoy/blob/b29d6543e7568a8a3e772c7909a1daa182acc670/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L40
[ALLOW-RENEG]: https://github.com/envoyproxy/envoy/blob/b29d6543e7568a8a3e772c7909a1daa182acc670/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L47
[MAX-SESS-KEYS]: https://github.com/envoyproxy/envoy/blob/b29d6543e7568a8a3e772c7909a1daa182acc670/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L53

##### Server Authorization aka Subject-Alt-Name Checks

"Server authorization" is performed by the client where the client authorizes
a server for the connection as described below. This check is in lieu of the
canonical [hostname check in Web PKI][RFC6125-6].

If [`match_subject_alt_names`][match_subject_alt_names] in
[`default_validation_context`][default_validation_context] of
[combined_validation_context][CVC] of the received `UpstreamTlsContext` is populated
then gRPC checks the SAN entries in the leaf server certificate against the
`match_subject_alt_names` values as follows:

* if the `match_subject_alt_names` list is empty then there is nothing to match and the
check succeeds.

* if there is no leaf server certificate, or there are no SAN entries in the certificate
the check fails.

* each SAN entry of type DNS, URI, email and IP address is considered for the below match
logic. An IP address is converted to its canonical string representation. For
IPv6 this includes [maximum zero compression][zero-compr], [no leading zeros][no-leading-0]
and [lower case][lower-case] e.g. `2001:db8::1` but *not* `2001:DB8:0::01`.
If an entry matches as per this logic, the check completes successfully.

  * if a SAN entry is empty then the match fails for that entry.

  * a SAN entry is matched against each value of
[`match_subject_alt_names`][match_subject_alt_names] as follows.
    * A `match_subject_alt_names` value is a [`StringMatcher`][StringMatcher] and
    the match is performed as per the semantics described for each `match_pattern`
    in the [`StringMatcher`][StringMatcher] type. Note that [exact match][ExactMatch]
    supports subdomain matching for wildcard DNS SAN entries as described
    [here][DNS-wildcard]. If there is a match, then the check completes successfully.

If the check fails, the connection attempt fails with "certificate check failure".

Individual implementations will use hooks provided by the underlying TLS framework to
implement server authorization. As an example, Java uses a custom `X509TrustManager`
implementation through the `SslContext` provided to the client sub-channel.

[match_subject_alt_names]: https://github.com/envoyproxy/envoy/blob/b29d6543e7568a8a3e772c7909a1daa182acc670/api/envoy/extensions/transport_sockets/tls/v3/common.proto#L407
[default_validation_context]: https://github.com/envoyproxy/envoy/blob/b29d6543e7568a8a3e772c7909a1daa182acc670/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L188
[CVC]: https://github.com/envoyproxy/envoy/blob/b29d6543e7568a8a3e772c7909a1daa182acc670/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L274
[StringMatcher]: https://github.com/envoyproxy/envoy/blob/6321e5d95f7e435625d762ea82316b7a9f7071a4/api/envoy/type/matcher/string.proto#L20
[ExactMatch]: https://github.com/envoyproxy/envoy/blob/6321e5d95f7e435625d762ea82316b7a9f7071a4/api/envoy/type/matcher/string.proto#L29
[RFC6125-6]: https://datatracker.ietf.org/doc/html/rfc6125#section-6
[zero-compr]: https://datatracker.ietf.org/doc/html/rfc5952#section-2.2
[no-leading-0]: https://datatracker.ietf.org/doc/html/rfc5952#section-2.1
[lower-case]: https://datatracker.ietf.org/doc/html/rfc5952#section-2.3
[DNS-wildcard]: https://github.com/envoyproxy/envoy/blob/b29d6543e7568a8a3e772c7909a1daa182acc670/api/envoy/extensions/transport_sockets/tls/v3/common.proto#L392-L394

#### LDS for Server Side Security

Server-side xDS processing is described in [A36: xDS-Enabled Servers][A36].
A Listener resource has zero or more [`filter_chains`][filter-chains] and one of those
`filter_chain`s is matched against an incoming connection (or the
[`default_filter_chain`][default-filter-chain] if there is no match) in order to get and apply
the security (and other) configuration for that connection as described in the
[`FilterChainMatch`][filter-chain-match] section. The [`transport_socket`][transport-socket]
of the matched (selected) `filter_chain` is used to extract the
[`DownstreamTlsContext`][DTC] as described [here][transport-socket-comment].
As part of validating all LDS updates, if a `transport_socket` name is not
`envoy.transport_sockets.tls` i.e. something we don't recognize, gRPC will
NACK an LDS update. Otherwise the `DownstreamTlsContext` thus obtained is used
for the incoming connection if the server is using `XdsServerCredentials`.
If there is no `DownstreamTlsContext` (such as when [`transport_socket`][transport-socket]
is not present) then gRPC uses the fallback credentials for the incoming connection.

[`common_tls_context`][CTC1] in the `DownstreamTlsContext` contains the required
security configuration. See below for [`CommonTlsContext`][CTC-type] processing.

A secure server at a minimum requires an identity certificate which is provided
by [`tls_certificate_provider_instance`][TCPI] (or in its absence the deprecated
field [`tls_certificate_certificate_provider_instance`][TCCPI]). If both fields are
empty, gRPC will NACK an update as part of validating all LDS updates. If
[ca_certificate_provider_instance][CCPI] inside the
[`default_validation_context`][default_validation_context] of [combined_validation_context][CVC]
(or in its absence the deprecated field
[`validation_context_certificate_provider_instance`][VCCPI1] inside
[`combined_validation_context`][CVC]) is provided then the server will request
the client certificate. When [`require_client_certificate`][RCC] is `true`,
gRPC requires the client certificate and will reject a connection without a valid
client certificate. When [`require_client_certificate`][RCC] is `true`, the
[ca_certificate_provider_instance][CCPI] field (or in its absence
[`validation_context_certificate_provider_instance`][VCCPI1]) should be
present otherwise gRPC will NACK an LDS update as part of validating all LDS
updates.

The following fields in an LDS update are ignored:

* [disable_stateless_session_resumption][DIS-STATELESS-SESS-RES]
* [session_ticket_keys][SESSION-TICKET-KEYS]
* [session_ticket_keys_sds_secret_config][SESSION-TICKET-KEYS-SDS]
* [session_timeout][SESSION-TIMEOUT]

If any of the following fields are present, gRPC will NACK an LDS update as
part of validating all LDS updates:

* [require_sni][REQ-SNI]
* [ocsp_staple_policy][OCSP-STAPLE-POLICY]: any value other than `LENIENT_STAPLING` causes NACK.

[A36]: A36-xds-for-servers.md
[filter-chains]: https://github.com/envoyproxy/envoy/blob/45ec050f91407147ed53a999434b09ef77590177/api/envoy/config/listener/v3/listener.proto#L120
[default-filter-chain]: https://github.com/envoyproxy/envoy/blob/45ec050f91407147ed53a999434b09ef77590177/api/envoy/config/listener/v3/listener.proto#L131
[filter-chain-match]: A36-xds-for-servers.md#filterchainmatch
[transport-socket]: https://github.com/envoyproxy/envoy/blob/45ec050f91407147ed53a999434b09ef77590177/api/envoy/config/listener/v3/listener_components.proto#L246
[DTC]: https://github.com/envoyproxy/envoy/blob/b29d6543e7568a8a3e772c7909a1daa182acc670/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L57
[transport-socket-comment]: https://github.com/envoyproxy/envoy/blob/45ec050f91407147ed53a999434b09ef77590177/api/envoy/config/listener/v3/listener_components.proto#L241
[CTC1]: https://github.com/envoyproxy/envoy/blob/b29d6543e7568a8a3e772c7909a1daa182acc670/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L84
[RCC]: https://github.com/envoyproxy/envoy/blob/b29d6543e7568a8a3e772c7909a1daa182acc670/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L88
[CTC-type]: https://github.com/envoyproxy/envoy/blob/b29d6543e7568a8a3e772c7909a1daa182acc670/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L129
[REQ-SNI]: https://github.com/envoyproxy/envoy/blob/b29d6543e7568a8a3e772c7909a1daa182acc670/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L92
[SESSION-TICKET-KEYS]: https://github.com/envoyproxy/envoy/blob/b29d6543e7568a8a3e772c7909a1daa182acc670/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L96
[SESSION-TICKET-KEYS-SDS]: https://github.com/envoyproxy/envoy/blob/b29d6543e7568a8a3e772c7909a1daa182acc670/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L99
[DIS-STATELESS-SESS-RES]: https://github.com/envoyproxy/envoy/blob/b29d6543e7568a8a3e772c7909a1daa182acc670/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L109
[SESSION-TIMEOUT]: https://github.com/envoyproxy/envoy/blob/b29d6543e7568a8a3e772c7909a1daa182acc670/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L116
[OCSP-STAPLE-POLICY]: https://github.com/envoyproxy/envoy/blob/b29d6543e7568a8a3e772c7909a1daa182acc670/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L124

#### CommonTlsContext Processing

`CommonTlsContext` is present in both the `UpstreamTlsContext` and `DownstreamTlsContext` protos and contains the
certificate and key configuration information needed on the client and server side respectively. The configuration
tells gRPC how to obtain certificates and the keys for the TLS handshake. Although there are various ways to obtain
certificates as per this proto (which are supported by Envoy), gRPC supports only two of them. The preferred mechanism
is the [`CertificateProviderPluginInstance`][CPPI] proto and the less preferred mechanism is
[`CertificateProviderInstance`][CPI] which is a deprecated proto but looks identical to
[`CertificateProviderPluginInstance`][CPPI]. They are both based on the notion of CertificateProvider plugin framework
(described later). The field `instance_name` defines a CertificateProvider "instance" that gRPC looks up in the bootstrap
file (described later) to obtain the CertificateProvider configuration. This configuration along with the CertificateProvider
plugin framework enables gRPC to acquire the certificates necessary for the TLS handshake.

The field [`tls_certificate_provider_instance`][TCPI] (or in its absence the deprecated
field [`tls_certificate_certificate_provider_instance`][TCCPI]) is used for the identity certificate and the
private key. And [`ca_certificate_provider_instance`][CCPI] inside the
[`default_validation_context`][default_validation_context] of [`combined_validation_context`][CVC]
(or in its absence the deprecated field [`validation_context_certificate_provider_instance`][VCCPI1] inside
[`combined_validation_context`][CVC]) is used for the root certificates for validating peer certificates.

gRPC does not support the other certificate acquisition mechanisms specified by the following fields:

* [tls_certificates][TLS-CERT]
* [tls_certificate_sds_secret_configs][CERT-SDS]
* [tls_certificate_certificate_provider][CERT-PROVIDER]
* [validation_context_sds_secret_config][VAL-SDS]
* [validation_context_certificate_provider][VAL-PROVIDER]

When fields of type [`CertificateProviderPluginInstance`][CPPI] (or the deprecated
[`CertificateProviderInstance`][CPI]) are present gRPC ignores the respective
unsupported fields but if the unsupported fields are present by themselves gRPC will NACK an update
as part of validating all CDS/LDS updates.

For example, in [`UpstreamTlsContext`][UTC] (on the client side), the identity certificate is not
needed for the TLS mode, which is indicated by the absence of both [`tls_certificate_provider_instance`][TCPI]
and [`tls_certificate_certificate_provider_instance`][TCCPI]. However in this case if any of the other
unsupported identity certificate fields ([tls_certificates][TLS-CERT], [tls_certificate_sds_secret_configs][CERT-SDS],
[tls_certificate_certificate_provider][CERT-PROVIDER]) are present then gRPC will NACK a CDS update.
But if [`tls_certificate_provider_instance`][TCPI] (or in its absence the deprecated
[`tls_certificate_certificate_provider_instance`][TCCPI]) is present, then the unsupported
fields are ignored. This allows a control plane to formulate a single `CommonTlsContext` that works
for both gRPC (which only supports [`CertificateProviderPluginInstance`][CPPI] or the deprecated
[`CertificateProviderInstance`][CPI]) and Envoy (which supports neither [`CertificateProviderPluginInstance`][CPPI]
nor [`CertificateProviderInstance`][CPI]) to achieve consistent security configuration between
gRPC and Envoy.

The following fields are unsupported and if present will cause a NACK from gRPC for any update
because ignoring these fields compromises security:

* [tls_params][TLS-PARAMS]
* [custom_handshaker][CUSTOM-HS]

[alpn_protocols][ALPN-PROTOCOLS] on the server side (inside [`DownstreamTlsContext`][DTC]) and
on the client side (inside [`UpstreamTlsContext`][UTC]) is ignored if present.
The processing and parsing of `CommonTlsContext` inside any particular CDS or LDS response does
not take into account whether Xds credentials are in effect for the respective channel or server.

[`combined_validation_context`][CVC] is validated in any CDS/LDS update as follows:
The field [match_subject_alt_names][] inside [default_validation_context][] is accepted only on
the client side i.e. inside `UpstreamTlsContext` and is processed as described [above][server-authz].
If any of the other fields (listed below) are present in [default_validation_context][], the update
is NACKed because ignoring these fields compromises security:

* `verify_certificate_spki`
* `verify_certificate_hash`
* `require_signed_certificate_timestamp`
* `crl`
* `custom_validator_config`

The following fields from [default_validation_context][] are ignored:

* `trusted_ca`
* `watched_directory`
* `allow_expired_certificate`
* `trust_chain_verification`

In [`combined_validation_context`][CVC], (other than [default_validation_context][]) only
[`validation_context_certificate_provider_instance`][VCCPI1] is accepted as a deprecated alternative to
[ca_certificate_provider_instance][CCPI]. When one of those fields ([ca_certificate_provider_instance][CCPI]
or [`validation_context_certificate_provider_instance`][VCCPI1]) is present, the other fields
[`validation_context_certificate_provider`][VCCP1] and [`validation_context_sds_secret_config`][VCSSC1]
are ignored otherwise the presence of any of those fields will cause a NACK.

[CPI]: https://github.com/envoyproxy/envoy/blob/b29d6543e7568a8a3e772c7909a1daa182acc670/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L165
[TCCPI]: https://github.com/envoyproxy/envoy/blob/b29d6543e7568a8a3e772c7909a1daa182acc670/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L256
[TCPI]: https://github.com/envoyproxy/envoy/blob/b29d6543e7568a8a3e772c7909a1daa182acc670/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L247
[VCCPI1]: https://github.com/envoyproxy/envoy/blob/b29d6543e7568a8a3e772c7909a1daa182acc670/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L205
[TLS-CERT]: https://github.com/envoyproxy/envoy/blob/b29d6543e7568a8a3e772c7909a1daa182acc670/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L226
[CERT-SDS]: https://github.com/envoyproxy/envoy/blob/b29d6543e7568a8a3e772c7909a1daa182acc670/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L239
[CERT-PROVIDER]: https://github.com/envoyproxy/envoy/blob/b29d6543e7568a8a3e772c7909a1daa182acc670/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L251
[VAL-SDS]: https://github.com/envoyproxy/envoy/blob/b29d6543e7568a8a3e772c7909a1daa182acc670/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L265
[VAL-PROVIDER]: https://github.com/envoyproxy/envoy/blob/b29d6543e7568a8a3e772c7909a1daa182acc670/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L278
[TLS-PARAMS]: https://github.com/envoyproxy/envoy/blob/b29d6543e7568a8a3e772c7909a1daa182acc670/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L212
[CUSTOM-HS]: https://github.com/envoyproxy/envoy/blob/b29d6543e7568a8a3e772c7909a1daa182acc670/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L301
[ALPN-PROTOCOLS]: https://github.com/envoyproxy/envoy/blob/b29d6543e7568a8a3e772c7909a1daa182acc670/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L297
[UTC]: https://github.com/envoyproxy/envoy/blob/b29d6543e7568a8a3e772c7909a1daa182acc670/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L26
[server-authz]: #server-authorization-aka-subject-alt-name-checks
[VCCP1]: https://github.com/envoyproxy/envoy/blob/b29d6543e7568a8a3e772c7909a1daa182acc670/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L199
[VCSSC1]: https://github.com/envoyproxy/envoy/blob/b29d6543e7568a8a3e772c7909a1daa182acc670/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L193
[CCPI]: https://github.com/envoyproxy/envoy/blob/b29d6543e7568a8a3e772c7909a1daa182acc670/api/envoy/extensions/transport_sockets/tls/v3/common.proto#L315
[CPPI]: https://github.com/envoyproxy/envoy/blob/b29d6543e7568a8a3e772c7909a1daa182acc670/api/envoy/extensions/transport_sockets/tls/v3/common.proto#L241

### Certificate Provider Plugin Framework

xDS supports many different configurations for endpoints to obtain certificates but gRPC supports only
[`CertificateProviderPluginInstance`][CPPI] (or its deprecated version [`CertificateProviderInstance`][CPI]).
In this approach, the control plane tells the xDS client the [name of a certificate][CERT-NAME] (which
is currently ignored) and the ["instance" name][INST-NAME] of a provider to use to obtain a certificate.
This provider [instance name][INST-NAME] is translated into a provider implementation and a configuration
for that implementation using the client's own configuration instead of being sent by the control plane.
This enables flexibility in heterogeneous deployments where different clients can use different
implementations for the same provider instance name. For example, an on-prem client may get its certificate
using a local certificate provider whereas a client running in cloud may use a cloud-vendor provided
implementation to obtain the certificate.

gRPC offers a `CertificateProvider` plugin API that can support multiple implementations. The plugin API
is not currently public, so applications cannot currently add their own provider implementations although
we might make this API public in the future. However gRPC currently includes a `file_watcher` provider
implementation - descibed [below][FILE-WATCHER-LINK] - which reads certificates from the local file system.

[FILE-WATCHER-LINK]: #file_watcher-certificate-provider
[CERT-NAME]: https://github.com/envoyproxy/envoy/blob/b29d6543e7568a8a3e772c7909a1daa182acc670/api/envoy/extensions/transport_sockets/tls/v3/common.proto#L253
[INST-NAME]: https://github.com/envoyproxy/envoy/blob/b29d6543e7568a8a3e772c7909a1daa182acc670/api/envoy/extensions/transport_sockets/tls/v3/common.proto#L247

Certificate provider plugin instances are configured via the xDS bootstrap file.  There is a new
top-level field called `"certificate_providers"` whose value is a map (a JSON object).  The key
in this map is the certificate_provider_instance name and the value is a JSON object
having exactly 2 fields:

* `"plugin_name"` which is the name of the plugin implementation (a string value), and
* `"config"` which is the configuration for the plugin. The value of `"config"` is a JSON object
whose schema is defined by that plugin.

The structure in the bootstrap file is as follows:

```jsonc
"certificate_providers": {
  "instance_name": {
    "plugin_name": "implementation_name",
    "config": {
      // ...config for implementation_name plugin...
    }
  }
}
```

This defines a plugin instance for the instance name `instance_name` which consists of a
plugin implementation called `implementation_name` with the corresponding configuration
defined as the value for the key `config`. When the xDS server tells the client to obtain
a certificate from `instance_name`, the client will use this plugin instance.

#### `file_watcher` Certificate Provider

As mentioned before, we currently only have one plugin implementation called `file_watcher`.
The configuration for this plugin has the following fields:
- `certificate_file`: The path to the file containing the identity certificate or certificate chain.
The file should contain a PEM-formatted X.509 conforming certificate or certificate chain.
- `private_key_file`: The path to the file containing the private key. The file should contain a
PEM-formatted PKCS encoded private key.
- `ca_certificate_file`: The path to the file containing the root certificates aka trust bundle.
The file should contain PEM-formatted X.509 conforming certificates.
- `refresh_interval`: Specifies how frequently the plugin should read the files. The value must be
in the [JSON format described for a `Duration`][DURATION-JSON] protobuf message.

For example, the bootstrap file might contain the following:

```jsonc
  "certificate_providers": {
    "google_cloud_private_spiffe": { // certificate_provider_instance name
      "plugin_name": "file_watcher", // name of the plugin implementation
      "config": {                    // config to be supplied to the plugin instance
        "certificate_file": "/var/run/gke-spiffe/certs/certificates.pem",
        "private_key_file": "/var/run/gke-spiffe/certs/private_key.pem",
        "ca_certificate_file": "/var/run/gke-spiffe/certs/ca_certificates.pem",
        "refresh_interval": "60s"
      }
    }
  }
```

With this configuration, when the xDS server tells the client to get a certificate from
plugin instance `"google_cloud_private_spiffe"` (the value of [`instance_name`][INST-NAME]),
the client will load the certificate data from the specified files. Note the
[certificate_name][CERT-NAME] value is currently ignored by this plugin.

[DURATION-JSON]: https://developers.google.com/protocol-buffers/docs/proto3#json.

## Implementation Details

The gRPC client xDS flow is described in [gRPC Client Architecture][A27:CDS]. After
obtaining the cluster load balancing policy configuration and optional security
configuration (in [`UpstreamTlsContext`][UTC]), gRPC passes it down to child policies
via channel arguments or a similar mechanism depending on the language. For example,
C-core uses channel arguments to pass down the configuration, whereas Java constructs
a dynamic `SslContextProvider` that is made available to sub-channels via channel
attributes. The `SslContextProvider` is used by a sub-channel's TLS handshaker to
build an `SslContext` for the pending TLS handshake.

Server-side xDS processing is described in [A36: xDS-Enabled Servers][A36]. After
obtaining the routing and optional security configuration (in
[`DownstreamTlsContext`][DTC]), gRPC makes it available to each incoming connection.
How this is done is language dependent. For example, Java uses a `ProtocolNegotiator`
to pick a [`DownstreamTlsContext`][DTC] for each incoming connection and construct a
dynamic `SslContextProvider` that is passed to the channel handler for that
connection. The channel handler uses the `SslContextProvider` to build an `SslContext`
for the pending TLS handshake.

A **`CertificateProvider`** object represents a plugin that provides the required
certificates and keys to the gRPC application. A component registers itself as a **`Watcher`**
of the plugin to receive certificate updates. The plugin supports multiple
**`Watcher`s** and uses a **`DistributorWatcher`** internally to propagate a
single update to multiple **`Watchers`**. The plugin caches the last
certificate (and key) and delivers them to every newly registered
watcher. Note that it is possible for an implementation to use a fetch-style API instead of a
watch-style API. For example, in the Go implementation a consumer calls into the
**`CertificateProvider`** whenever it needs certificates and keys instead of registering a
watcher.

The **`CertificateProvider`** plugin also has an associated factory
(**`CertificateProviderFactory`**) that is used to instantiate the plugin. The
factory is identified by a unique name e.g. `"file_watcher"` that is also the identity of the
plugin. The factory is used to instantiate the plugin after validating the received
configuration. The framework uses deduping and reference counting to ensure that a plugin
of a given kind and configuration is instantiated only once.

A plugin typically creates a new thread and a timer to periodically query and 
send the certificate (and the key) to all its watchers. **`FileWatcherCertificateProvider`**
(aka `file_watcher` described [above][FILE-WATCHER-LINK]) is an implementation of
the **`CertificateProvider`**. This plugin monitors configured file-paths in the file
system and reads those files on updates to get the latest certificates and keys and
provide those to the watchers after converting them to the canonical format. For example,
Java uses `java.security.PrivateKey` and `java.security.cert.X509Certificate` for a
private key and a certificate respectively.

## Rationale

gRPC has been steadily adding support for various xDS features such as load balancing and
advanced traffic management. xDS-based security was next in the roadmap and this proposal
addresses that requirement. gRPC makes a proxyless service mesh (PSM) possible. A secure
proxyless service mesh requires the security features described in this proposal.

The Certificate Provider Plugin Framework provides a generic alternative to the SDS
server/agent based solution and eliminates the dependency on the SDS protocol. An SDS
protocol based client can still be added just as another plugin implementation in the
framework. The plugin framework allows different environments to have different
dependencies because the xDS client environment is abstracted out via the plugin
framework and the control plane can be agnostic to the specific client environment.
The framework has also made the certificate provider functionality extensible
and pluggable which enables support for new certificate providers without requiring
changes to the xDS protocol or the control plane. These advantages plus the extensions
made to the xDS protocol are providing an impetus to the Envoy community so that Envoy
will hopefully have a similar framework in the near future to support pluggable and
extensible certificate providers.

In any case the addition of xDS based security has brought gRPC xDS support closer to
Envoy in terms of capabilities. This enables various use-cases such as migration to
proxyless and interop or coexistence with Envoy.

## Implementation Status

Java had an early prototype implementation of xDS-based security that used SDS similar to Envoy. It was
then modified and refined to match this spec. Other languages followed soon after to complete
the implementation. The implementations (in Java, C++ and Go) are currently "hidden" behind the
environment variable `GRPC_XDS_EXPERIMENTAL_SECURITY_SUPPORT` which will be removed
once this proposal is approved.
