Additional PyPI Packages for gRPC Python
----
* Author(s): lidiz
* Approver: gnossen
* Status: Approved
* Implemented in: python
* Last updated: 05/14/2020
* Discussion at: https://groups.google.com/g/grpc-io/c/6thuMVK8fTo

## Abstract

gRPC Python is uploaded as `grpcio` on PyPI, but there is another package named
`grpc`. Hence, some users are confused. This document proposes to upload
additional packages named `grpc` and `grpc-*` to guide users to official
packages.

## Background

### Package `grpc`

Package `grpc` is an alternative version of gRPC Python which only supports
Python 2, and its first version is uploaded in 2013. We have seen complaints
about the installation failure or unable to execute our example code. The tricky
package name is a usability issue for years. In 2019, gRPC team claimed the
package name `grpc` under [PEP 541](https://www.python.org/dev/peps/pep-0541/)
(see [thread](https://github.com/pypa/pypi-support/issues/3)).

### Package `grpcio`

Package `grpcio` is first uploaded in 2015, given the fact that the name `grpc`
already being taken, this name is likely a workaround.

## Proposal

### Package Content

Upon installation, each new package raises a `RuntimeError` exception, asking
users to install with the correct package name. For example, for the new
package `grpc`, the exception will look like:

```
RuntimeError: Please install the official package with: pip install grpcio
```

### Maintenance Cost

The maintenance cost for this solution is minimum since these packages are
agnostic to our releases.

### List of New Packages

Here is a list of new packages:
* `grpc`
* `grpc-status`
* `grpc-channelz`
* `grpc-tools`
* `grpc-reflection`
* `grpc-testing`
* `grpc-health-checking`

### Package Version

Rather than releasing new versions of the `grpc*` packages in lockstep with the
canonical `grpcio*` packages for each release, we will instead upload only one
new artifact for each package name, versioned as `v1.0`. A stable version number
suggests this package's behavior won't change actively.

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
question will be historical versions. Should we upload `grpc`-prefixed packages
for older versions? And which version should they depend on?

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
