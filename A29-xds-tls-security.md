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
configuration for a gRPC channel or server. This is achieved by supplying an XdsChannelCredentials
(to a channel) or an XdsServerCredentials (to a server). These two new credentials types are an extension
of the existing channel or server credentials types in each language. Each Xds*Credentials needs to
be provided with a "fallback credentials". The fallback credentials is used in the following cases:

- when xDS is not in use such as when the `xds:` scheme is not used on the client side

- xDS is in use but the control plane does not provide security configuration. The gRPC behavior
in this case is different from Envoy's due to fallback credentials. Envoy will just
use plaintext (insecure) communication mode in this case. But with gRPC, the application
needs to use InsecureCredentials as the fallback credentials to get the same result.

The fallback credentials is *not* used in case of error situations e.g. whenever there is
an error in using the xDS provided security configuration.

Note that a user is not required to use Xds- Channel or Server Credentials even if they are
using an xDS managed channel or server. Not using Xds Channel or Server Credentials results
in not using xDS-provided TLS configuration.

On the client side, an XdsChannelCredentials is used and on the server side an
XdsServerCredentials is used as described in the example snippets below. In these examples,
a plaintext or insecure credentials is used as the fallback credentials.


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

gRPC uses CDS to configure client side security and LDS for server side security as described below.
The required fields were added to xDS v3 hence xDS v3 is a pre-requisite for this proposal.

#### CDS for Client Side Security

The CDS policy is the top level LB policy applied to all the connections under that policy.
[A27:CDS][] describes the flow. The field [`transport_socket`][CL-TS] is used to extract the
[`UpstreamTlsContext`][UTC] as described [here][CL-TS-comment].
Note that we don't (currently) support [`transport_socket_matches`][CL-TS-matches].
The security configuration extracted from the `UpstreamTlsContext` thus obtained
is passed down to all the child policies and connections. How this is done is language dependent
and is similar to how the implementations pass policy information down to child policies.

[`common_tls_context`][CTC] in the `UpstreamTlsContext` contains the required security configuration.
See below for [`CommonTlsContext`][CTC-type] processing.

A secure client at a minimum requires [`validation_context_certificate_provider_instance`][VCCPI1]
inside [`combined_validation_context`][CVC] to be able to validate the server certificate
in the TLS mode. If this field is not set, gRPC will NACK the CDS update. The client identity
certificate is configured via the [`tls_certificate_certificate_provider_instance`][TCCPI]
field which is optional. The client certificate is sent to the server if the server requests
or requires it in the TLS handshake. If the server requires the client certificate but is not
present then the TLS handshake fails.

If any of the following fields are present, gRPC will NACK the CDS update:

* [sni][SNI]
* [allow_renegotiation][ALLOW-RENEG]
* [max_session_keys][MAX-SESS-KEYS]

[A27:CDS]: A27-xds-global-load-balancing.md#cds
[CL-TS]: https://github.com/envoyproxy/envoy/blob/9aca65c395f01020080166e4795455addde167fa/api/envoy/config/cluster/v3/cluster.proto#L937
[CL-TS-comment]: https://github.com/envoyproxy/envoy/blob/9aca65c395f01020080166e4795455addde167fa/api/envoy/config/cluster/v3/cluster.proto#L932
[CL-TS-matches]: https://github.com/envoyproxy/envoy/blob/9aca65c395f01020080166e4795455addde167fa/api/envoy/config/cluster/v3/cluster.proto#L680
[CTC]: https://github.com/envoyproxy/envoy/blob/7d4b2cae486b66b62ba0d3e1e348504699bea1bf/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L37
[SNI]: https://github.com/envoyproxy/envoy/blob/7d4b2cae486b66b62ba0d3e1e348504699bea1bf/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L40
[ALLOW-RENEG]: https://github.com/envoyproxy/envoy/blob/7d4b2cae486b66b62ba0d3e1e348504699bea1bf/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L47
[MAX-SESS-KEYS]: https://github.com/envoyproxy/envoy/blob/7d4b2cae486b66b62ba0d3e1e348504699bea1bf/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L53

#### LDS for Server Side Security

Server-side xDS processing is described in [A36: xDS-Enabled Servers][A36].
A Listener resource may have one or more [`filter_chains`][filter-chains] and one of those
`filter_chain`s is matched against an incoming connection in order to get and apply
the security (and other) configuration for that connection as described in the
[`FilterChainMatch`][filter-chain-match]. The [`transport_socket`][transport-socket]
of the matched (selected) `filter_chain` is used to extract the
[`DownstreamTlsContext`][DTC] as described [here][transport-socket-comment].
If the `transport_socket` name is not `envoy.transport_sockets.tls` i.e.
something we don't recognize, gRPC will NACK the LDS update. Otherwise
the `DownstreamTlsContext` thus obtained is used for the incoming connection.
If it is not present (such as when [`transport_socket`][transport-socket] is
not present or is not named `"envoy.transport_sockets.tls"`) then we use the
fallback credentials for the incoming connection.

[`common_tls_context`][CTC1] in the `DownstreamTlsContext` contains the required
security configuration. See below for [`CommonTlsContext`][CTC-type] processing.

A secure server at a minimum requires an identity certificate which is provided
by [`tls_certificate_certificate_provider_instance`][TCCPI]. If that field is
empty, gRPC will NACK the update. If
[`validation_context_certificate_provider_instance`][VCCPI1] inside
[`combined_validation_context`][CVC] is provided then the server will request
the client certificate. When [`require_client_certificate`][RCC] is `true`,
gRPC requires the client certificate and will reject a connection without a valid
client certificate. When [`require_client_certificate`][RCC] is `true`, the
[`validation_context_certificate_provider_instance`][VCCPI1] value should be
present otherwise gRPC should NACK the LDS update.

If any of the following fields are present, gRPC will NACK the LDS update:

* [require_sni][REQ-SNI]
* [session_ticket_keys][SESSION-TICKET-KEYS]
* [session_ticket_keys_sds_secret_config][SESSION-TICKET-KEYS-SDS]
* [disable_stateless_session_resumption][DIS-STATELESS-SESS-RES]
* [session_timeout][SESSION-TIMEOUT]
* [ocsp_staple_policy][OCSP-STAPLE-POLICY]

[A36]: A36-xds-for-servers.md
[filter-chains]: https://github.com/envoyproxy/envoy/blob/45ec050f91407147ed53a999434b09ef77590177/api/envoy/config/listener/v3/listener.proto#L120
[filter-chain-match]: A36-xds-for-servers.md#filterchainmatch
[transport-socket]: https://github.com/envoyproxy/envoy/blob/45ec050f91407147ed53a999434b09ef77590177/api/envoy/config/listener/v3/listener_components.proto#L246
[DTC]: https://github.com/envoyproxy/envoy/blob/7d4b2cae486b66b62ba0d3e1e348504699bea1bf/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L57
[transport-socket-comment]: https://github.com/envoyproxy/envoy/blob/45ec050f91407147ed53a999434b09ef77590177/api/envoy/config/listener/v3/listener_components.proto#L241
[CTC1]: https://github.com/envoyproxy/envoy/blob/7d4b2cae486b66b62ba0d3e1e348504699bea1bf/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L84
[RCC]: https://github.com/envoyproxy/envoy/blob/7d4b2cae486b66b62ba0d3e1e348504699bea1bf/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L88
[CTC-type]: https://github.com/envoyproxy/envoy/blob/7d4b2cae486b66b62ba0d3e1e348504699bea1bf/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L129
[REQ-SNI]: https://github.com/envoyproxy/envoy/blob/7d4b2cae486b66b62ba0d3e1e348504699bea1bf/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L92
[SESSION-TICKET-KEYS]: https://github.com/envoyproxy/envoy/blob/7d4b2cae486b66b62ba0d3e1e348504699bea1bf/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L96
[SESSION-TICKET-KEYS-SDS]: https://github.com/envoyproxy/envoy/blob/7d4b2cae486b66b62ba0d3e1e348504699bea1bf/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L99
[DIS-STATELESS-SESS-RES]: https://github.com/envoyproxy/envoy/blob/7d4b2cae486b66b62ba0d3e1e348504699bea1bf/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L109
[SESSION-TIMEOUT]: https://github.com/envoyproxy/envoy/blob/7d4b2cae486b66b62ba0d3e1e348504699bea1bf/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L116
[OCSP-STAPLE-POLICY]: https://github.com/envoyproxy/envoy/blob/7d4b2cae486b66b62ba0d3e1e348504699bea1bf/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L124

#### CommonTlsContext Processing

`CommonTlsContext` is present in both the `UpstreamTlsContext` and `DownstreamTlsContext` protos and contains the
certificate and key configuration information needed on the client and server side respectively. The configuration
tells gRPC how to obtain certificates and the keys for the TLS handshake. Although there are various ways to obtain
certificates as per this proto (which are supported by Envoy), gRPC supports only one of them and that is defined by the
[`CertificateProviderInstance`][CPI] proto which is based on the notion of CertificateProvider plugin framework
(described later). The field `instance_name` defines a CertificateProvider "instance" that gRPC looks up in the
bootstrap file (described later) to obtain the CertificateProvider configuration. This configuration along with
the CertificateProvider plugin framework enables gRPC to acquire the certificates necessary for the TLS handshake.

The field [`tls_certificate_certificate_provider_instance`][TCCPI] is used for the identity certificate and the private
key. And [`validation_context_certificate_provider_instance`][VCCPI1] inside [`combined_validation_context`][CVC]
is used for the root certificate-chain for validating peer certificates.

gRPC does not support the other certificate acquisition mechanisms specified by the following fields:

* [tls_certificates][TLS-CERT]
* [tls_certificate_sds_secret_configs][CERT-SDS]
* [tls_certificate_certificate_provider][CERT-PROVIDER]
* [validation_context_sds_secret_config][VAL-SDS]
* [validation_context_certificate_provider][VAL-PROVIDER]

When fields of type [`CertificateProviderInstance`][CPI] are present gRPC ignores the respective
unsupported fields but if the unsupported fields are present by themselves gRPC will NACK the update.

For example, in [`UpstreamTlsContext`][UTC] (on the client side), the identity certificate is not needed for the TLS mode,
which is indicated by [`tls_certificate_certificate_provider_instance`][TCCPI] being absent. However in this case
if any of the other unsupported identity certificate fields ([tls_certificates][TLS-CERT],
[tls_certificate_sds_secret_configs][CERT-SDS], [tls_certificate_certificate_provider][CERT-PROVIDER]) are
present then gRPC will NACK the update. But if [`tls_certificate_certificate_provider_instance`][TCCPI] is
present, then the unsupported fields are ignored. This allows a control plane to formulate a single
`CommonTlsContext` that works for both gRPC (which only supports [`CertificateProviderInstance`][CPI])
and Envoy (which does not support [`CertificateProviderInstance`][CPI]) to achieve consistent
security configuration between gRPC and Envoy.

The following fields are unsupported and if present will cause a NACK from gRPC:

* [tls_params][TLS-PARAMS]
* [custom_handshaker][CUSTOM-HS]

[alpn_protocols][ALPN-PROTOCOLS] on server side (inside [`DownstreamTlsContext`][DTC]) should
be absent. On the client side (inside [`UpstreamTlsContext`][UTC]) it is ignored if present.
The processing and parsing of `CommonTlsContext` inside any particular CDS or LDS response does
not take into account whether Xds credentials are in effect for the respective channel or server.

[CPI]: https://github.com/envoyproxy/envoy/blob/7d4b2cae486b66b62ba0d3e1e348504699bea1bf/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L154
[TCCPI]: https://github.com/envoyproxy/envoy/blob/7d4b2cae486b66b62ba0d3e1e348504699bea1bf/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L234
[VCCPI1]: https://github.com/envoyproxy/envoy/blob/7d4b2cae486b66b62ba0d3e1e348504699bea1bf/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L200
[TLS-CERT]: https://github.com/envoyproxy/envoy/blob/7d4b2cae486b66b62ba0d3e1e348504699bea1bf/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L215
[CERT-SDS]: https://github.com/envoyproxy/envoy/blob/7d4b2cae486b66b62ba0d3e1e348504699bea1bf/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L225
[CERT-PROVIDER]: https://github.com/envoyproxy/envoy/blob/7d4b2cae486b66b62ba0d3e1e348504699bea1bf/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L230
[VAL-SDS]: https://github.com/envoyproxy/envoy/blob/7d4b2cae486b66b62ba0d3e1e348504699bea1bf/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L242
[VAL-PROVIDER]: https://github.com/envoyproxy/envoy/blob/7d4b2cae486b66b62ba0d3e1e348504699bea1bf/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L255
[TLS-PARAMS]: https://github.com/envoyproxy/envoy/blob/7d4b2cae486b66b62ba0d3e1e348504699bea1bf/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L207
[CUSTOM-HS]: https://github.com/envoyproxy/envoy/blob/7d4b2cae486b66b62ba0d3e1e348504699bea1bf/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L276
[ALPN-PROTOCOLS]: https://github.com/envoyproxy/envoy/blob/7d4b2cae486b66b62ba0d3e1e348504699bea1bf/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L272

### Use of SPIFFE IDs in Certificates

To facilitate service mesh secure communication, we support [SPIFFE][] based certificates and peer certificate
verification following the [SPIFFE][] specification and the [match_subject_alt_names][] field in xDS.

[SPIFFE]: https://github.com/spiffe/spiffe
[match_subject_alt_names]: https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/transport_sockets/tls/v3/common.proto#envoy-v3-api-field-extensions-transport-sockets-tls-v3-certificatevalidationcontext-match-subject-alt-names

### Certificate Provider Plugin Framework

The pluggable Certificate Provider framework in gRPC underlies the use of `CertificateProviderInstance`
configuration. The framework supports "Certificate Providers" as pluggable and custom components (plugins)
used for dynamically fetching certificates used during the TLS handshake of gRPC channel and server connections.

This document discusses just enough of the framework to explain the use-case. Users who want to
implement new plugins will need to understand the detailed design of the framework in the
respective language.

Envoy was the original and only consumer of the xDS protocol and its security features were tied to
Envoy's security capabilities such as fetching certificates from the file-system or
using a separate SDS protocol to fetch certificates from an agent (that's not defined in the xDS
architecture). Instead of emulating Envoy's functionality and limitations in this area, we decided to
have a generic Certificate Provider plugin architecture in gRPC with the following characteristics:

* the xDS control plane is not aware of the plugin specifics including its name, instantiation
mechanism or configuration requirements.

* all of these are abstracted out into a single “plugin-instance-name” which is resolved into an
actual Certificate provider instance using the local bootstrap file and the framework (as explained
later).

* the xDS control plane only references the “plugin-instance-name”s: it uses one for the local
certificates and another one for root certificates to validate the peer certificates (aka
[`CertificateValidationContext`][CVC-proto]).

[CVC-proto]: https://github.com/envoyproxy/envoy/blob/c94e646e0280e4c521f8e613f1ae2a02b274dbbf/api/envoy/extensions/transport_sockets/tls/v3/common.proto#L236

#### Main Elements of the Framework

**`CertificateProvider`** represents an actual plugin that fetches or "mints" the required
certificates and keys. **`FileWatcherCertificateProvider`** (described below) is an
implementation of the **`CertificateProvider`** that is part of this framework. Another example
of a **`CertificateProvider`** is a plugin that periodically "mints" new certificates by creating
CSRs at regular intervals and getting those signed by a Certification Authority (CA) which is
configured into the plugin.

A "consumer" registers itself as a **`Watcher`**  of the plugin to receive certificate updates.
The plugin supports multiple **`Watcher`s** to be registered and uses a **`DistributorWatcher`**
internally to propagate a single update to multiple **`Watchers`**. A plugin caches the latest
certificate (and key) it has fetched (or minted) and delivers them to every newly registered
watcher. Note that it is possible for an implementation to use a fetch-style API instead of a
watch-style API. For example, in the Go implementation a consumer calls into the
**`CertificateProvider`** whenever it needs certificates and keys instead of registering a
watcher.

The **`CertificateProvider`** plugin also has an associated factory or
provider implementation (**`CertificateProviderFactory`**) to instantiate the plugin and this
factory is identified by a unique name e.g. `"file_watcher"` which is also the identity of the
plugin. The factory is responsible for instantiating the plugin after validating the received
configuration.

**`CertificateProviderRegistry`** is a registry of all registered plugins supported by the gRPC library.
A **`CertificateProviderFactory`** registers itself in the registry.

**`CertificateProviderStore`** is a store or global map of all instantiated plugins. In this map
the key is the plugin-name plus its config and the value is a reference counted instantiated plugin.
Reference counting ensures that a plugin with a given name and configuration combination is only
instantiated once. A client of the framework looks up a plugin by name and config and adds itself
as a **`Watcher`**. If the plugin is not already present, the store uses the registry to
instantiate it. It then increments the reference-count and adds the new **`Watcher`** to the 
**`DistributorWatcher`** of the plugin. The plugin typically creates a new thread and a
timer to periodically fetch or mint a new certificate and send the certificate (and the key) to
all its watchers.

**`FileWatcherCertificateProvider`** is an implementation of **`CertificateProvider`** which
monitors configured file-paths in the file system and reads those files on updates to get the latest
certificates and keys and provide those to the consumers after converting them to the canonical
format e.g. `java.security.PrivateKey` and `java.security.cert.X509Certificate` in Java.

### Bootstrap File Additions for Security

The [bootstrap file][bootstrap-file] with [server side support][A36-xds-protocol] needs further
changes as described here. The following snippet shows the additions for the `file_watcher` plugin
that is already implemented in all gRPC languages.
```
{
  // "certificate_providers" lists the instances available for xDS to use.
  "certificate_providers": {
    "google_cloud_private_spiffe": { // certificate_provider_instance name
      "plugin_name": "file_watcher", // name of the plugin
      "config": {                    // config to be supplied to the plugin
        "certificate_file": "/var/run/gke-spiffe/certs/certificates.pem",
        "private_key_file": "/var/run/gke-spiffe/certs/private_key.pem",
        "ca_certificate_file": "/var/run/gke-spiffe/certs/ca_certificates.pem",
        "refresh_interval": "60s"
      }
    }
  }
}
```

`"certificate_providers"` is a new top level field whose value is a map (a JSON object).
The key in this map is the certificate_provider_instance name and the value is a JSON object
having exactly 2 fields:

* `"plugin_name"` which is the name of the plugin (a string value), and
* `"config"` which is the configuration for the plugin. The value of `"config"` is a JSON object
whose schema is defined by that plugin. For the file_watcher plugin the `"config"` consists
of the 3 file-paths to watch and the polling interval value.

[bootstrap-file]: A27-xds-global-load-balancing.md#xdsclient-and-bootstrap-file

#### Use of Bootstrap File by the Certificate Provider Plugin Framework

The xDS control plane specifies a certificate_provider_instance name as described above in
[CommonTlsContext Processing](#commontlscontext-processing). For this example, let's assume
it uses `"google_cloud_private_spiffe"` as the `"instance_name"` in
`tls_certificate_certificate_provider_instance`. gRPC looks up
`"google_cloud_private_spiffe"` in the `"certificate_providers"` map loaded from the
bootstrap file and use the JSON object value to extract the plugin name (value of
`"plugin_name"` as a string) and configuration (value of `"config"` as a JSON object).
It passes on these values to the `CertificateProviderStore` to retrieve
(after instantiating if necessary) the associated `CertificateProvider` plugin.
The caller of the framework registers a watcher with the plugin to receive
certificate updates as described below.

### Implementing Security in the xDS Flow

Let's see how gRPC enables security in xDS managed clients and servers by putting it all
together. Note that gRPC uses the existing low level TLS handshaker code/libraries
(aka security connector or handshaker) after making the certificates and keys available
to that code. The TLS handshaker functionality (which is language dependent) is
out of scope of this document.

#### Client Side Flow

The gRPC client xDS flow is described in [gRPC Client Architecture][A27-client-arch].
The resolver returns a `Cluster` resource for a channel and the CDS for that cluster has the
[UpstreamTlsContext][UTC] containing the security configuration to be applied to all the
child policies of that cluster. When a channel is using `XdsChannelCredentials`, gRPC
processes the `UpstreamTlsContext` as described [above](#cds-for-client-side-security).

At this stage the CDS policy uses the flow described above in
[Use of Bootstrap File by the Certificate Provider Plugin Framework][use-of-bootstrap]
to fetch the requisite `CertificateProvider`s (one or two as the case may be). These `CertificateProvider`s,
in one form or another, need to be made available to all the sub-channels of the cluster. How this is
done is language dependent. For example, C-core uses the channel args to pass the individual
`CertificateProvider`s whereas Java constructs a dynamic `SslContextProvider` that is directly
usable by the sub-channel's TLS handshaker to build an `SslContext` for the impending handshake.

[A27-client-arch]: ./A27-xds-global-load-balancing.md#grpc-client-architecture
[UTC]: https://github.com/envoyproxy/envoy/blob/7d4b2cae486b66b62ba0d3e1e348504699bea1bf/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L26
[use-of-bootstrap]: #use-of-bootstrap-file-by-the-certificate-provider-plugin-framework

##### Server Authorization aka Subject-Alt-Name Checks

If [match_subject_alt_names][] is populated in the [combined_validation_context][CVC]
of the received `UpstreamTlsContext` then gRPC validates the SAN entries in the server certificate
by matching the `match_subject_alt_names` values using the [StringMatcher][] semantics.
This is called server authorization because this is how a client "authorizes" a server for
the connection.

The implementation is language dependent. As an example, Java uses an implementation
of `X509TrustManager` through the `SslContext` provided to the client sub-channel. The other
languages use similar hooks provided by the underlying TLS framework to implement server
authorization.

[match_subject_alt_names]: https://github.com/envoyproxy/envoy/blob/c94e646e0280e4c521f8e613f1ae2a02b274dbbf/api/envoy/extensions/transport_sockets/tls/v3/common.proto#L373
[CVC]: https://github.com/envoyproxy/envoy/blob/7d4b2cae486b66b62ba0d3e1e348504699bea1bf/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L251
[StringMatcher]: https://github.com/envoyproxy/envoy/blob/6321e5d95f7e435625d762ea82316b7a9f7071a4/api/envoy/type/matcher/string.proto#L20

#### Server Side Flow

The gRPC server xDS flow is described in [gRPC Server][A36-xds-protocol]. The server
configuration comes from the [`Listener`][Listener] resource which contains a list of
[`FilterChain`][Filter-chain]s. The `FilterChain` contains the `DownstreamTlsContext`
that is applied to an incoming connection
after selecting the best-matching `FilterChain` as described in
[FilterChainMatch][A36-filter-chain-match]. If the server is using `XdsServerCredentials`,
gRPC processes the `DownstreamTlsContext` as described [above](#lds-for-server-side-security) and then
uses the flow described above in
[Use of Bootstrap File by the Certificate Provider Plugin Framework](#use-of-bootstrap-file-by-the-certificate-provider-plugin-framework)
to fetch the requisite `CertificateProvider`s (one or two as the case may be). These `CertificateProvider`s,
in one form or another, need to be made available to the incoming connection. Similar to the
client side, this part of the flow is language dependent. For example, Java constructs a dynamic
`SslContextProvider` that is directly usable by the connection's TLS handshaker to build an
`SslContext` for the impending handshake.

[A36-xds-protocol]: A36-xds-for-servers.md#xds-protocol
[Listener]: https://github.com/envoyproxy/envoy/blob/45ec050f91407147ed53a999434b09ef77590177/api/envoy/config/listener/v3/listener.proto#L39
[Filter-chain]: https://github.com/envoyproxy/envoy/blob/45ec050f91407147ed53a999434b09ef77590177/api/envoy/config/listener/v3/listener_components.proto#L193
[A36-filter-chain-match]: A36-xds-for-servers.md#filterchainmatch

## Rationale

gRPC has been steadily adding support for various xDS features including load balancing and
advanced traffic management. xDS-based security was next in the roadmap and this proposal
addresses that. gRPC makes a proxyless service mesh (PSM) possible. A secure proxyless
service mesh requires the security features described in this proposal.

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
proxyless and interop or co-existence with Envoy.

## Implementation

Java had an early prototype implementation of xDS-based security that used SDS similar to Envoy. It was
then modified and refined to match this spec. Other languages followed soon after to complete
the implementation. The implementations (in Java, C++ and Go) are currently "hidden" behind the
environment variable `GRPC_XDS_EXPERIMENTAL_SECURITY_SUPPORT` which will be removed
once this proposal is official.
