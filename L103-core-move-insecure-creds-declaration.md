L103: C-core: Move function declarations for each credential type from `grpc/grpc_security.h` to its own header file
----
* Author(s): [Cheng-Yu Chung (@ralphchung)](https://github.com/ralphchung)
* Approver: [@markdroth](https://github.com/markdroth)
* Status: Ready for Implementation
* Implemented in: C Core
* Last updated: 2022-11-16
* Discussion at: https://groups.google.com/g/grpc-io/c/6qvo-UVs-uI

## Abstract

Move function declarations for each credential type from `grpc/grpc_security.h` to its own header file.

## Background

Previously, the PR https://github.com/grpc/grpc/pull/25586 helps move us to an eventual future where each type of credentials is in its own build target, so applications can link in only the specific one(s) they need.

However, the issue https://github.com/grpc/grpc/issues/31012 points out the fact that `grpc/grpc_security.h` contains functions that has nothing to do with secure credentials, which contradicts with the goal we would like to achieve in the PR above.

## Proposal

Move function declarations for each credential type from `grpc/grpc_security.h` to its own header file. The following is the list of mapping.

* google_default_credentials: grpc/channel_credentials/google_default.h
* ssl_credentials: grpc/channel_credentials/ssl.h
* alts_credentials: grpc/channel_credentials/alts.h
* local_credentials: grpc/channel_credentials/local.h
* tls_credentials: grpc/channel_credentials/tls.h
* insecure_credentials: grpc/channel_credentials/insecure.h
* xds_credentials: grpc/channel_credentials/xds.h

## Rationale

Moving function declarations to `grpc/grpc.h` seems to be convenient but not right. Moving them to their own files makes more sense because our goal is to make every credential type have its own target.

Note that we do not plan to reserve backward compatibility by including `grpc/credentials/*.h` in `grpc/grpc_security.h` since we do not promise backward compatibility for C-core API.

## Implementation

Move function declarations for each credential type from `grpc/grpc_security.h` to its own header file as indicated in the "Proposal" section.

## Open issues (if applicable)

https://github.com/grpc/grpc/issues/31012
