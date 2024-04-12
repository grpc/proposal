Privatize gpr_ headers that are not used by wrapped languages
----
* Author(s): vjpai
* Approver: nicolasnoble
* Status: Approved
* Implemented in: https://github.com/grpc/grpc/pull/14184, https://github.com/grpc/grpc/pull/14190, https://github.com/grpc/grpc/pull/14196, https://github.com/grpc/grpc/pull/14197
* Last updated: January 25, 2018
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/xdKVhaGhhAE

## Abstract

Review the contents of `include/grpc/support` to see which should
remain public (if used in wrapped languages or public API), which
should be moved to `test` (if only used for tests), and which should
be internalized in `src/core/lib/gpr` (if not needed for public-facing
features and used only in the core and C++ API implementation).

## Background

The `gpr` functions implement a variety of support features used in
gRPC core, the C++ API implementation, wrapped languages, and
tests. The publicly-exposed portion of the `gpr` API resides in
`include/grpc/support` and includes many features that are not
actually important to any language API (such as AVL trees).

### Related Proposals:

N/A

## Proposal

Curate the contents of `include/grpc/support`
- Keep entries that are used in wrapped language implementations or
any public API (e.g., `time.h`, `alloc.h`, ...)
- Move entries to `test/core/util` if they are only used to support
  tests (e.g. `cmdline.h`, `subprocess.h`)
- Move entries to `src/core/lib` if they are used only in the core
  and C++ API implementations but not used for any public-facing API
  (e.g., `avl.h`, `tls.h`). Files like `tls.h` that relate to portability
  concerns should be in `src/core/lib/gpr` while those like `avl.h`
  that are purely algorithmic should move elsewhere in `src/core/lib`
  and change the `gpr_` prefix in their public contents with the
  `grpc_` prefix.

## Rationale

The objective of this is to reduce the number of public API surface
touch points. This is particularly an option at this time as part of
the de-wrapping of the C++ API implementation.

## Implementation

This proposal will be implemented incrementally and opportunistically
across several PRs.

## Open issues (if applicable)

N/A

