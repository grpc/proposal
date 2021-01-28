gRPC Python Major Version Bump from 1.x to 2.x
----
* Author(s): lidiz
* Approver: gnossen, psrini
* Status: In-Review
* Implemented in: Python
* Last updated: 2021-01-26
* Discussion at: https://groups.google.com/g/grpc-io/c/ZiFK3j9H0Zc

## Abstract

This propose suggests gRPC Python should perform a major version bump to achieve two following goals:

1. Stop releasing 2.7 binaries and stop coding in Python 2/3 compatible manner;
2. Drop custom IO manager usages (e.g., gevent), which will be re-implemented via EventEngine API in the future.


## Background

### Python 2 Status in Our Dependencies and Dependents

gRPC has a clean runtime dependency list with only two major libraries: `cython`, `protobuf`, `six`, and several for build time: `pip`, `setuptools`, `wheels`. In the past years, all of our dependencies completed the support for Python 3, but none explicitly dropped 2.7 support, until `pip` version 21.0 stopped supporting 2.7 in Jan 2021.

For gRPC dependents, we have announced Python2 deprecation on [GitHub](https://github.com/grpc/grpc/issues/13602) in 2017, on [grpc-io](https://groups.google.com/g/grpc-io/c/psLfi-li4u8/m/O1PgTsu8AQAJ) in 2018, and via package [README/description](https://github.com/grpc/grpc/commit/cb9e2188ab05f983f447ed4cf8fb93b299bd224d#diff-ca2883c36eb18a58083dc1e6b953ee9a8b9317bc18287427838779705a7dd61f) in 2019. According to https://pypistats.org/packages/grpcio, the Python 2 download of gRPC Python has dropped to 5% daily.


### Deprecation of Custom IO Manager

Custom IO manager has been a blocker for cleaning technical debt in gRPC Core for years. We had several attempts in the past trying to remove it (e.g., [proposal#182](https://github.com/grpc/proposal/pull/182)), and there will be a new EventEngine API in gRPC Core soon. The new design improves the portability and flexibility. gRPC Python uses customer IO manager for 2 IO schemas: 1. supporting gevent; 2. one alternative way of supporting AsyncIO.


### Related Proposals:
Major version bump in other gRPC languages: https://github.com/grpc/proposal/pull/165, https://github.com/grpc/proposal/blob/master/L57-csharp-new-major-version.md


## Proposal

### Version Number 1.x to 2.x

The version number of gRPC Python will be bumped from 1.x to 2.x, instead of re-counting from 2.0.0. For example, if the releasing Core version is `v1.35.0`, then the corresponding Python version will be `v2.35.0`. In this way, the minor version will still be useful to trace change logs on GitHub and in gRPC Core. This approach is also picked by [C# major version](https://github.com/grpc/proposal/blob/master/L57-csharp-new-major-version.md).


### Python 3 Features

Sadly, even without Python 2, gRPC Python still can't utilize all Python 3 features since we want to support 3.5. But, to some extent, we should be able to type annotate most of our APIs with 3.5+ type annotation.


### APIs Changes

1. In the first 2.x release, the `grpc.experimental.gevent` will be removed. The `grpc.beta`/`grpc.framework` API will be removed as well.
2. Exceptions in gRPC Python have useability issues, like `abort` raises an `Exception` and it's hard to interact, so in the new release, we will promote `UsageError` and `AbortError` to stable API and change corresponding logic. As for `grpc.RpcError`, it will be merged with `grpc.aio.AioRpcError` which clearly states how to access the error information of the failed RPC.
3. Interceptors have been essential to gRPC Python for a long time now, and they should be stabilized as well.
4. gRPC Easy (dynamic proto generation + simple stub) will also be promoted to stable API and as our recommended way of using protos in our examples.


## Rationale

### Alternative: Version Number Counting from 2.0.0

We spent a huge effort to make sure there is only one ground truth in our repo for the gRPC version number (which is in `BUILD`). Counting from 2.0.0 requires either adding an extra manual version number or adding an anchor version and compute the Python version every time. The benefit of this approach is limited but adding extra cognitive burden.


## Implementation

TBD

## Open issues (if applicable)

TBD
