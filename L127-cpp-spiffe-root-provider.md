L127: C++: SPIFFE Bundle Map support in Root Providers
----
* Author(s): gtcooke94
* Approver: markdroth
* Status: In Review
* Implemented in: core, cpp
* Last updated: 2025-07-17
* Discussion at: https://groups.google.com/g/grpc-io/c/G47BjLsF4JQ

## Abstract

The purpose of this proposal is to add public API support for SPIFFE bundle maps in root certificate file watcher providers. [A87] details the broader internals for this support.

## Background

gRPC supports SPIFFE bundle maps as root certificate material per [A87]. Public APIs to configure these roots are needed.

### Related Proposals:
* [A87]

[A87]: A87-mtls-spiffe-support.md

## Proposal

This document proposes to extend the C-Core and C++ APIs as follows:


### C-Core
In the C-core API, we will add a new `spiffe_bundle_map_path` parameter to the `grpc_tls_certificate_provider_file_watcher_create()` function, which will now look like this:

```
GRPCAPI grpc_tls_certificate_provider*
grpc_tls_certificate_provider_file_watcher_create(
    const char* private_key_path, const char* identity_certificate_path,
    const char* root_cert_path, const char* spiffe_bundle_map_path, unsigned int refresh_interval_sec);
```

If the `spiffe_bundle_map_path` is set, the `root_cert_path` will be ignored. This holds even in the case where the `spiffe_bundle_map_path` ends up being invalid.

### C++
While the existing C++ API is marked experimental, we don't _want_ to break existing users. Thus, we will add a constructor with the `spiffe_bundle_map_path` argument to the `FileWatcherCertificateProvider`.
In order to not break current users, we will make the existing constructors support this by supplying an empty SPIFFE bundle map path.
```

FileWatcherCertificateProvider(const std::string& private_key_path,
                                const std::string& identity_certificate_path,
                                const std::string& root_cert_path,
                                const std::string& spiffe_bundle_map_path,
                                unsigned int refresh_interval_sec);
```

### Other Providers
This proposal _only_ aims to support file-based SPIFFE Bundle Maps via the file watcher providers. The `StaticDataCertificateProvider` structure is left as future work. This will involve broadening the API surface to expose a type for the SPIFFE bundle map.

## Implementation
PR will be linked when created.