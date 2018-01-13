Require gRFCs for core API changes
----
* Author(s): vpai
* Approver: a11r
* Status: Proposed
* Implemented in: n/a
* Last updated: January 12, 2018
* Discussion at:

## Abstract

To facilitate the stabilization of gRPC core and its use in
wrapped-language APIs, any change, addition, or deletion of a core
surface API (those declared in
https://github.com/grpc/grpc/tree/master/include/grpc or its
subdirectories) should be accompanied by a gRFC in the
https://github.com/grpc/proposal repository.

## Background

The gRPC Core library is not considered public API. However, it is
directly used by numerous language bindings. Although we have allowed
changes to the core surface API in the past without the gRFC process,
this should not be a continued practice in the future as these
destabilize the use of core.

### Related Proposals:

N/A

## Proposal

Any addition, change, or deletion of a core surface API data type or
function should not be implemented without following the gRFC
process. There should be an exception for adding new APIs that are
marked as experimental, but these must go through a gRFC if they are
ever changed from experimental to non-experimental.

## Rationale

One of the objectives of this work and of recent gRPC core efforts in
general is to allow wrapped languages to move to their own
repositories, bringing in core as a submodule. As a result, these will
update their core linkage point less frequently and are susceptible to
breakages at upgrades as core APIs change.

Changes and deletions are considered API breakage that requires a bump
in the major version number of core and these will immediately break
other language-binding implementors. These language-binding
implementors should have a chance to comment before their code is
broken.

The rationale in the case of additions is more subtle. Just as in the
actual language binding APIs, an API addition should be considered a
promise of stability. The library should not make promises of
stability without careful consideration. There can be an exception for
adding new APIs that are marked experimental, as these do not offer any
implied promise of stability and the gRFC process may slow down
experimentation with a new and useful feature.

## Implementation

N/A

## Open issues (if applicable)

N/A
