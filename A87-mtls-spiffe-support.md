A87: mTLS SPIFFE Support
----
* Author(s): @erm-g, @markdroth
* Approver: @ejona86, @dfawley
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2024-11-01
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

We will add support for SPIFFE certificate verification in mTLS.  This
will be supported both with TlsCredentials and via xDS.

## Background

[SPIFFE] is a standardized way of encoding workload identity in mTLS
certificates.  Specifically, we want to support using the [SPIFFE bundle
format] in place of a single CA certificate for peer certificate
verification.  The bundle map format is defined in
https://github.com/spiffe/spiffe/pull/304.

### Related Proposals: 
* [gRFC A29: xDS-Based mTLS Security for gRPC Clients and Servers][gRFC A29]

[gRFC A29]: A29-xds-tls-security.md
[SPIFFE]: https://github.com/spiffe/spiffe/blob/main/standards/SPIFFE.md
[SPIFFE bundle format]: https://github.com/spiffe/spiffe/blob/main/standards/SPIFFE_Trust_Domain_and_Bundle.md#4-spiffe-bundle-format

## Proposal

There are two main parts to this proposal, support for SPIFFE
certificate verification in TlsCredentials, and configuring that
functionality via xDS.  We will cover the xDS case first.

### Configuring SPIFFE Verification via xDS

Currently, certificate providers as defined in [gRFC A29] can return
only a single CA certificate for use in peer certificate verification.  We
will change the certificate provider API such that the provider may
choose to return either CA certificates or a SPIFFE trust bundle
map, although it must choose one -- it may not return both for the same
certificate name.

If the certificate provider returns CA certificates, then
standard X509 certificate verification will be used, just as it is
today.  However, if the certificate provider returns a SPIFFE trust
bundle map, then SPIFFE verification will be performed, and non-SPIFFE
peer certificates will be rejected.

When performing SPIFFE verification, the following steps will be performed:

1. Upon receiving a peer certificate, verify that it is a well-formed SPIFFE
   leaf certificate.  In particular, it must have a single URI SAN containing
   a well-formed SPIFFE ID.

2. Use the trust domain in the peer certificate's SPIFFE ID to lookup
   the SPIFFE trust bundle. If the trust domain is not contained in the
   configured trust map, reject the certificate.

3. Verify the peer certificate using the default security library using
   the SPIFFE trust bundle certificates as roots.

SAN matching will be performed as configured via xDS, just as it is with
X509 verification.

gRPC currently supports only one certificate provider implementation,
which is the `file_watcher` provider.  We will add a new field to the
configuration of this provider implementation configuration called
`spiffe_trust_bundle_map_file`.  This field will specify the path to
the SPIFFE trust bundle map.

If the `spiffe_trust_bundle_map_file` field is unset, then the
`ca_certificate_file` field will be used exactly as it is today, and
the certificate provider will return CA certificates.

If the `spiffe_trust_bundle_map_file` field is set, the certificate
provider will return the SPIFFE trust bundle map from the specified file.
If the `ca_certificate_file` field is also set, it will be ignored,
which is helpful for forward compatibility: bootstrap configs can start
setting the new field regardless of what gRPC version is in use, which
results in older versions returning CA certificates and newer versions
returning the SPIFFE trust bundle map.

### SPIFFE Certificate Verification in TlsCredentials

The xDS mTLS support is built on top of existing mTLS functionality in
TlsCredentials.  This section describes how the SPIFFE functionality
will work in TlsCredentials and how the xDS functionality builds on top
of it.  The details of this are different in each language.

#### C++

In C++, TlsCredentials natively supports a CertificateProvider API with
essentially the same semantics as in xDS.  We will add the ability for
the `FileWatcherCertificateProvider` to be instantiated with a SPIFFE
trust bundle map file instead of a CA certificate file.

TODO: @erm-g to fill in API details.  We'll probably need to introduce
an options struct for FileWatcherCertificateProvider.

#### Java

In Java, we need new API to work with [SPIFFE] and [SPIFFE bundle
format].  A new `SpiffeUtil` will be developed:

```java
class SpiffeUtil {
   Optional<SpiffeId> extractSpiffeId(X509Certificate[] certChain);
   SpiffeBundle loadTrustBundleFromFile(String trustBundleFile);
}
class SpiffeId {
   String getTrustDomain();
   String getPath();
}
class SpiffeBundle {
   Map<String, List<X509Certificates>> getBundleMap();
}
```

For the xDS functionality described above, the `XdsX509TrustManager`
will gain the ability to be instantiated with a SPIFFE trust bundle
map.  In this case, it will use the SPIFFE trust bundle certificates as
trusted roots.  If the `XdsX509TrustManager` is instantiated using CA
certificates (existing functionality), then it'll contimue to use them
exactly as it does today.  The new APIs will look like this:

```java
class XdsTrustManagerFactory {
   XdsTrustManagerFactory(Map<String, List<X509Certificates>>);
   XdsTrustManagerFactory(X509Certificates[]);
}
class XdsX509TrustManager {
   XdsX509TrustManager(Map<String, XdsX509ExtendedTrustManager>);
   XdsTrustManagerFactory(XdsX509ExtendedTrustManager);
}
```

We'll also adjust existing hierarchies of `Watcher` and
`CertificateProvider` to support the new SPIFFE trust bundle map.

#### Go

TODO: @erm-g to fill this in.

### Temporary environment variable protection

The xDS functionality will be guarded via the
`GRPC_EXPERIMENTAL_XDS_MTLS_SPIFFE` environment variable.  The new
bootstrap field will not be read if this env var is unset.  This env var
guard will be removed when the feature has passed interop tests.

## Rationale

Allowing both `spiffe_trust_bundle_map_file` and `ca_certificate_file`
to be set in the `file_watcher` certificate provider config provides
forward compatibility, as described above.  The alternative was to make
these two fields mutually exclusive, which would have required using a
different bootstrap config with older and newer versions of gRPC in
order to get the proper fallback behavior (using CA certificates) for
older versions.

## Implementation

The overall flow is nearly identical to the one described in [gRFC A29 Implementation Details].
At a high level, the following components require modification to support SPIFFE:
* **SpiffeUtils**: This utility needs to be enhanced with the ability to:
    * Load a SPIFFE trust bundle from a JSON file
    * Extract a SPIFFE ID from a certificate chain
* **Certificate Provider Factory**: Needs to support the new xDS field `spiffe_trust_bundle_map_file`
* **FileWatcherCertificateProvider**: Needs to support loading Trust bundle from the provided file
* **Handshaking**: Needs to support using SPIFFE trust bundle map and extracted Spiffe ID from peer certificate chain to verify connections

The detailed description of the components and the flow is below.

**Certificate Provider Factory** receives a config with the new xDS field `spiffe_trust_bundle_map_file`, and instantiates **FileWatcherCertificateProvider** using the file path to the json file containing the SPIFFE trust bundle map. Old `ca_certificate_file` (if present) is ignored.

**FileWatcherCertificateProvider** starts to monitor the new file path to SPIFFE trust bundle map json file. It uses **SpiffeUtils** to load, parse the JSON file, and extract a mapping of trust domains to their corresponding CA certificates. If an error (for example, file doesn’t exist / isn’t readable / contains invalid entries) occurs during the initial load, the SPIFFE trust bundle map can't be created and, since there is no fallback mechanism), the certificate verification process will fail. For subsequent loads, errors will not affect the existing in-memory trust bundle map. Note - an empty map will be considered a valid (but empty) trust bundle map.

**SpiffeUtils** provides the API for constructing SPIFFE trust bundle map from json file (used by **FileWatcherCertificateProvider**), and the API for extracting SPIFFE ID from peer certificate chain (used during **Handshaking**). Note - [SPIFFE ID] spec is fully supported, [SPIFFE bundle format] and [Publishing SPIFFE bundle format] are partially supported:
* Only ‘keys’ and ‘spiffe_sequence’ elements of JWK set are supported
* Only ‘kty’, ‘use’, and ‘x5c’ elements of ‘keys’ are supported
* Instead of ignoring individual JWK entries in case of issues, we ignore the whole TrustBundle
    
**Handshaking**: The  handshaking logic incorporates the following SPIFFE-specific steps:
* If the SPIFFE feature is enabled, extract (using **SpiffeUtils**) the SPIFFE ID from the peer's certificate chain.
* Use the trust domain from the SPIFFE ID to look up the corresponding CA certificates in the SPIFFE trust bundle map.
* Use these trusted roots for certificate verification.
  
Note - If the SPIFFE ID is missing or the SPIFFE trust bundle map lacks the necessary data, the verification process will fail.



[gRFC A29 Implementation Details]: A29-xds-tls-security.md#implementation-details
[SPIFFE ID]: https://github.com/spiffe/spiffe/blob/main/standards/SPIFFE-ID.md#2-spiffe-identity
[Publishing SPIFFE bundle format]: https://github.com/spiffe/spiffe/blob/main/standards/X509-SVID.md#61-publishing-spiffe-bundle-elements

### Implementation schedule
This will first be implemented in Java by @erm-g, with implementations in Go and C++ to follow in 2025.

