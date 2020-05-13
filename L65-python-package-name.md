Additional PyPI Packages for gRPC Python
----
* Author(s): lidiz
* Approver: gnossen
* Status: Draft
* Implemented in: python
* Last updated: 05/13/2020
* Discussion at: **TBD**

## Abstract

gRPC Python is uploaded as `grpcio` on PyPI, but there is another package named
`grpc`. Hence some users are confused. This document proposes gRPC Python to
upload additional packages named `grpc` and `grpc-*` as proxies to our official
packages.

## Background

### Package `grpc`

Package `grpc` is an alternative version of gRPC Python only supports Python 2,
and the first version is uploaded in 2013. We have seen complaints about the
installation failure or unable to execute our example code. The tricky package
name is a usability issue for years.

### Package `grpcio`

Package `grpcio` is first uploaded in 2015, given the fact that the name `grpc`
already being taken, this name is likely a workaround.

## Proposal

### List of Package Name

Here is a list of new proxy packages:
* `grpc`
* `grpc-status`
* `grpc-channelz`
* `grpc-tools`
* `grpc-reflection`
* `grpc-testing`
* `grpc-health-checking`

### Proxy Mechanism

Each proxy package will raise a `RuntimeError` exception upon installation,
asking users to install with the correct package name. For example, for proxy
package `grpc`, the exception will look like:

```
RuntimeError: Please install the official package with: pip install grpcio
```

### Maintenance Cost

The maintenance cost for this solution is minimum since these packages are
agnostic to our releases.

### Package Version

All new proxy packages will be versioned as `1.0`.

## Rationale

### Why not switch the name to `grpc` and `grpc-*`?

#### Hyrum's Law

Although the package name is not ideal, it is part of our contract with our
users. If we change our name from `grpcio` to `grpc`, it is not backward
compatible. For users who perform strict dependency management, their
environment might not automatically pick-up this change. It could result in
crashes since all gRPC logic was moved to another package.

There are two valid reasons for Python users to use strict dependency
management:

1. Freezing the dependency for shipping;
2. Version conflict in the dependency tree.

For the second case, some package may pin to an old version of gRPC, but some
requires a newer version. It's impossible to make both dependency happy, so the
application has to manage its dependency manually, which might be broken by the
name switch.

#### Costly transition

For the switch to happen smoothly, we need at least 3 steps: upload both
packages, mark one of them as deprecated, wait one year before taking other
actions. It won't be an efficient use of engineering resources.

Also, to make the name switch definitive, we will need to rename our source code
directories from `grpcio*` to `grpc*`. This change is not as simple as it seems.
Companies with proprietary build system or strict building rules (like Bazel)
might be directly affected by it. It is a single `mv` command for us, but hours
of debugging for our dependent.

### Why not implicitly depend on `grpcio` (or the other package)?

This approach was proposed in
[grpc#21531](https://github.com/grpc/grpc/pull/21531). However, **the benefit of
letting users successfully install a wrong package is arguable**. Package name
typo is easy to fix (type two more letters), but the cost to keep the
transparent dependency work is non-trivial.

Implementing transparent dependency is a complex topic. The first question will
be how to version the new packages? Should we always upload them simultaneously
with built packages? Like uploading `grpc==1.29.0` and `grpcio==1.29.0` at the
same time. It would also require more work in our infrastructure. The second
question will be historical versions. Should we upload proxy packages for older
versions? Which version should they depend on?

Also, if we allow the package with the wrong name to be installed successfully,
users might diverge between these two names and generate contents
(StackOverflow, blogs, etc.) with different package names.

### Why not upload the same package to both names?

1. It effectively doubles our number of artifacts;
2. It widens the gap between two packages names;
3. Might lead to conflict if a user depends on both of them transitively.

### PyPI download stats

The official package `grpcio` has an excellent usage ranking on PyPI. Keep the
name makes the stats tracking more straightforward, comparing to split the
download stats between two packages, which means we won't see a valid number
unless running a query.

## Implementation

TBD
