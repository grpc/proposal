Aspect-based Python Bazel Rules
----
* Author(s): Michael Beardsworth
* Approver: gnossen
* Status: Draft
* Implemented in: Bazel and Starlark
* Last updated: Sept. 8, 2021
* Discussion at: https://groups.google.com/g/grpc-io/c/NXdtW5xe9Js

## Abstract

Proposes modifications to gRPC's Bazel rules for Python codegen to improve
dependency tracking.

## Background

* gRPC provides build rule logic written in Starlark to support Protobuf and gRPC Python codegen with Bazel.
* These rules (`py_proto_library` and `py_grpc_library`) each generate code from a single `proto_library`'s protos.
* The produced Python libraries (or equivalently-functioning PyInfo providers) do not express dependencies on the generated Python code for the `proto_library`'s dependencies.
* Users are surprised by this behavior. See [#23769](https://github.com/grpc/grpc/issues/23769) for an example bug.
* This behavior is particularly surprising for Google-internal users. The internal versions of `py_proto_library` and `py_grpc_library` propagate Python dependencies.

### Related Proposals:
`py_proto_library` and `py_grpc_library` are implemented in [python_rules.bzl](https://github.com/grpc/grpc/blob/master/bazel/python_rules.bzl). There doesn't seem to be a proposal for that, though.

## Proposal

* We propose rebuilding the `py_proto_library` and `py_grpc_library` rules to use [aspects](https://docs.bazel.build/versions/main/skylark/aspects.html) to perform code generation steps.
  This approach corresponds to the Google-internal design for these rules.
* The `_gen_py_proto_aspect` aspect visits `proto_library` rules to generate Python code for Protobuf.
* The aspect produces a custom [providers](https://docs.bazel.build/versions/main/skylark/rules.html#providers) `PyProtoInfo` that wraps a `PyInfo` provider to avoid creating spurious dependencies for Python users that interface with the `proto_library` rules through some other means.
* The `py_proto_library` and `py_grpc_library` rules will only be responsible for collecting the `PyInfo` providers from their dependencies.
* The `plugin` attribute must be removed from `py_proto_library`. Aspects require the declaration of all possible parameter values up front, so it would not be possible for the new aspects to continue supporting arbitrary plugins. (Note that the plugin feature is not used in gRPC. It was introduced to support [GAPIC](https://github.com/googleapis/gapic-generator-python), which no longer uses the feature.)
* No behavior change should be observed by the user of `py_proto_library` or `py_grpc_library` unless they rely on the (removed) `plugin` attribute.

## Rationale

The proposed approach addresses the open bug and corrects the dependency issues. However, it requires the removal of a (likely unused) feature of the build rules.

Alternatively a set of repository rules could be introduced that allow users to inject `py_proto_library` and `py_grpc_library` implementations into the gRPC Bazel build system.
This alternative approach would allow users to work around the dependency issues by taking on additional burden.
The new build logic could be moved into a separate repository, or potentially upstreamed to a Bazel-owned repository.

## Implementation

The implementation is mostly complete by beardsworth in [#27275](https://github.com/grpc/grpc/pull/27275).

## Open issues (if applicable)

It is difficult to determine how many users rely on the `plugin` attribute. If many users rely on this behavior then it may not be feasible to adopt this proposal.
