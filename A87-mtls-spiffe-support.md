A87: mTLS SPIFFE Support
----
* Author(s): @markdroth, @erm-g, @gtcooke94
* Approver: @ejona86, @dfawley
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2025-02-21
* Discussion at: https://groups.google.com/g/grpc-io/c/55oIW6GNabs

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
[SPIFFE ID format]: https://github.com/spiffe/spiffe/blob/main/standards/SPIFFE-ID.md#2-spiffe-identity
[SPIFFE bundle format]: https://github.com/spiffe/spiffe/blob/main/standards/SPIFFE_Trust_Domain_and_Bundle.md#4-spiffe-bundle-format
[Publishing SPIFFE bundle format]: https://github.com/spiffe/spiffe/blob/main/standards/X509-SVID.md#61-publishing-spiffe-bundle-elements

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
   a well-formed SPIFFE ID ([SPIFFE ID format]).

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

When reading the SPIFFE trust bundle map file, the certificate provider
will parse the JSON to ensure its validity and store the map in memory.
If an error occurs during the initial load (e.g., a failure to read
the file, or the file contains invalid entries), then there is no valid
map, which means new connection attempts will fail the TLS handshake.
For subsequent loads, errors will not affect the existing in-memory
trust bundle map.  An empty map will be considered a valid (but empty)
trust bundle map.

Note that the [SPIFFE bundle format] and [Publishing SPIFFE bundle format]
are partially supported:
- Only the `keys` and `spiffe_sequence` elements of the JWK set are supported.
- Only the `kty`, `use`, and `x5c` elements of `keys` are supported.
- Instead of ignoring individual JWK entries in case of issues, we ignore
  the whole trust bundle.

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

With the adding of a SPIFFE trust bundle map file, the
`FileWatcherCertificateProvider` constructor is beginning to have a lot of
arguments. We will introduce a `FileWatcherCertificateProviderOptions` struct
and associated constructor similarly to `TlsCredentialOptions`.
We will not remove the old constructors in order to not break current users.

```
class FileWatcherCertificateProviderOptions {
   public:
     FileWatcherCertificateProviderOptions();
     // Copy constructor does a deep copy of the underlying pointer. No assignment
     // permitted
     FileWatcherCertificateProviderOptions(const FileWatcherCertificateProviderOptions& other);
     FileWatcherCertificateProviderOptions& operator=(const FileWatcherCertificateProviderOptions& other) = delete;

     FileWatcherCertificateProviderOptions& set_private_key_path(absl::string_view private_key_path);
     FileWatcherCertificateProviderOptions& set_identity_certificate_path(absl::string_view identity_certificate_path);
     FileWatcherCertificateProviderOptions& set_root_cert_path(absl::string_view root_cert_path);
     FileWatcherCertificateProviderOptions& set_spiffe_bundle_map_path(absl::string_view spiffe_bundle_map_path);
     FileWatcherCertificateProviderOptions& set_private_key_path(absl::string_view private_key_path);


   private:
     std::string private_key_path_;
     std::string identity_certificate_path_;
     std::string root_cert_path_;
     std::string spiffe_bundle_map_path_;
     unsigned int refresh_interval_sec_;
}

class FileWatcherCertificateProvider final .... {
   // existing class definitions
   FileWatcherCertificateProvider(const FileWatcherCertificateProviderOptions& options);
}
```

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
certificates (existing functionality), then it'll continue to use them
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

In Go, we can leverage [the go spiffe
package](https://pkg.go.dev/github.com/spiffe/go-spiffe/v2) for parsing and
object abstractions.  Currently, similarly to C++ above, Go already supports
[file watcher credential
providers](https://github.com/grpc/grpc-go/blob/e0d191d8adcdd73aad084154769404dd2f6b0fc6/credentials/tls/certprovider/pemfile/watcher.go#L92C4-L99)
with xDS. We can further modify these providers, specifically the file watcher,
to allow them to be configured with and return SPIFFE bundles instead of only
certificates.
Specifically, the Provider is designed to return a
[KeyMaterial](https://github.com/grpc/grpc-go/blob/e0d191d8adcdd73aad084154769404dd2f6b0fc6/credentials/tls/certprovider/provider.go#L91-L97)
struct where we could easily add a trust bundle.

When using providers, we already [create our own custom verification
function](https://github.com/grpc/grpc-go/blob/e0d191d8adcdd73aad084154769404dd2f6b0fc6/security/advancedtls/advancedtls.go#L513)
for the `tls.Config`. Any specific needs for the SPIFFE bundles during
verification can be implemented here as well.

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
    
Java implementation:
* SPIFFE ID parser https://github.com/grpc/grpc-java/pull/11490
* SPIFFE Utils for extracting SPIFFE ID from leaf certificate and extracting SPIFFE trust bundle map from JSON file https://github.com/grpc/grpc-java/pull/11575
* SPIFFE verification process https://github.com/grpc/grpc-java/pull/11627

Will be implemented in all other languages, timelines TBD.
