Improved handling of subdirectories and external repositories for py_proto_library
----
* Author(s): Thomas K&ouml;ppe (tkoeppe)
* Approver: gnossen
* Status: Draft
* Implemented in: Starlark
* Last updated: 2021-11-29
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

The current implementation of `py_proto_library` fails if a dependency of a
proto is located in a strict subdirectory of its package. This is because it
uses `basename` to derive paths of generated files, which discards directory
information. The proposal is to change the implementation to a more general
prefix removal that retains the package-relative directory structure.

Furthermore, the current implementation fails if a proto is taken from an
external repository and has generated sources. This time this is due to a use of
`rsplit` that discards directory information and results in the wrong import
path being added to the list of PYTHONPATHs. We propose to use the same improved
prefix removal to compute the correct import path.

## Background

The current Starlark implementation of `py_proto_library` appears to make
certain assumptions on the location of `.proto` files (namely 1 that they are
located in the root of their package, and 2 that the are not "virtual"
(i.e. generated) if they come from an external repository); these assumptions
are combined with knowledge of Bazel's output tree structure to derive the
locations of various generated files. The current logic discards part of the
proto's source information that is necessary to make both proto imports and
Python module imports work.


### Related Proposals:

(None.)

## Proposal

See pull requests:

* https://github.com/grpc/grpc/pull/28040 introduces the more general
  `_make_prefix` helper function to compute the prefix in the output tree of the
  proto file (regardless of whether the file is on disk or generated, or in the
  same repository or in an external one). Then, `source_file.basename` is
  replaced with `source_file.path[len(prefix):]`.
* https://github.com/grpc/grpc/pull/28103 fixes the import path that is added to
  the PYTHONPATH list so as to add the correct path that contains the generated
  Python module for a proto library from an external repository whose sources
  are generated (a so-called "virtual import").

## Rationale

It appears to be an outright defect of the current implementation that certain
kinds of `proto_library` dependencies are not supported. Nothing in the current
API suggests constraints on `proto_library` dependencies; this proposal adjusts
the implementation to conform to the API.

## Implementation

The proposal has been implemented; see list of pull requests above.
