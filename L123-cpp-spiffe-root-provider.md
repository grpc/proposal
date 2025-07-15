L123: Spiffe Bundle Map support in Root Providers
----
* Author(s): gtcooke94
* Approver: markdroth
* Status: Draft
* Implemented in: core, cpp
* Last updated: 2025-07-15
* Discussion at: TODO

## Abstract

The purpose of this proposal is to add public API support for SPIFFE bundle maps in root certificate file watcher providers. [gRFC A87](https://github.com/grpc/proposal/blob/master/A87-mtls-spiffe-support.md) details the broader internals for this support.

## Background

gRPC supports SPIFFE bundle maps as root certificate material per [gRFC A87](https://github.com/grpc/proposal/blob/master/A87-mtls-spiffe-support.md). Public APIs to configure these roots is needed.

### Related Proposals:
* [gRFC A87](https://github.com/grpc/proposal/blob/master/A87-mtls-spiffe-support.md)

## Proposal

This document proposes to extend the C-Core and C++ APIs as follows:


### C-Core
C-Core APIs are always subject to change - we will simply add an argument to the existing constructor in https://github.com/grpc/grpc/blob/79769b35d04535259592ac1b0a98d65f63203f06/include/grpc/credentials.h#L663-L666.
```
GRPCAPI grpc_tls_certificate_provider*
grpc_tls_certificate_provider_file_watcher_create(
    const char* private_key_path, const char* identity_certificate_path,
    const char* root_cert_path, const char* spiffe_bundle_map_path, unsigned int refresh_interval_sec);
```

### C++
While the existing C++ API is marked experimental, we don't _want_ to break existing users. Thus, in https://github.com/grpc/grpc/blob/79769b35d04535259592ac1b0a98d65f63203f06/include/grpcpp/security/tls_certificate_provider.h#L109-L112, we will add a constructor with the `spiffe_bundle_map_path` argument.
In order to not break current users, we will make the existing constructors support this by supplying an empty SPIFFE bundle map path.
```

FileWatcherCertificateProvider(const std::string& private_key_path,
                                const std::string& identity_certificate_path,
                                const std::string& root_cert_path,
                                const std::string& spiffe_bundle_map_path,
                                unsigned int refresh_interval_sec);
```

### Other Providers
This proposal _only_ aims to support file-based SPIFFE Bundle Maps via the file watcher providers. The `StaticDataCertificateProvider` structure is left as future work - we do not want to expose our internal SPIFFE utilities.