Title
----
* Author(s): ctiller@google.com (Craig Tiller)
* Approver: a11r
* Status: Draft
* Implemented in: N/A
* Last updated: 4/29/2021
* Discussion at: https://groups.google.com/g/grpc-io/c/q11WFj3KQOs

## Abstract

Assert that gRFC's are needed before large scale changes to gRPC Core are undertaken.

## Background

The gRFC process was intended to (quoting from README.md in this repository):
* Provide increased visibility to the community on upcoming changes and the design considerations around them.
* Provide the ability to reason about larger "sets" of changes that are too big to be covered either in an Issue or in a PR.
* Establish a consistent process for structured participation by the community on large changes, especially those that impact multiple runtimes and implementations.

There are now multiple teams contributing to gRPC Core, each with different motivations and organizational aims.
These teams form a community that needs to find consensus on overall direction of the gRPC core codebase.
As such, the core API should no longer be considered a cut point for gRFC's, and instead any larger change ought to be considered for such a process.

### Related Proposals: 
* This change extends P3 to expect gRFC's for changes to gRPC Cores internals and not just it's public API.

## Proposal

Any large scale change to gRPC Core should not be implemented without following the gRFC process.

A large scale change is one that has a wide-ranging impact on the implementation of gRPC Core.

Such changes include:
* Changes to extensibility points, or the types used by extensibility points.
* Changes that require downstream consumers of gRPC Core to modify their code.
* Changes with a significant performance impact.
* Changes to vocabulary types - types that are used broadly within the library.
* Changes that constrain future development.
* Changes that modify system architecture.

## Rationale

This gRFC will slow down progress on some kinds of changes for gRPC Core.
At this point in gRPC's lifetime this is probably appropriate: gRPC is a widely used product, multiple teams contribute to it's development, and any single person cannot cover all of the impact of changes on the myriad use cases for which gRPC is deployed.

By moving the design process to written discussion we lessen the need for meetings.
This helps us include more voices in the design process - people who may not work at Google, or may be intimidated to speak up in large groups.
By identifying design problems before they reach downstream overall project velocity will likely increase.

## Implementation

N/A

## Open issues (if applicable)

N/A
