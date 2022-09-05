L101: C-Core Remove grpc_register_plugin
----
* Author(s): ctiller
* Approver: markdroth
* Status: In Review
* Implemented in: C Core
* Last updated: 2022/09/04
* Discussion at: https://groups.google.com/g/grpc-io/c/mFpiC_LyW1o

## Abstract

Remove grpc_register_plugin from the public API.

## Background

Mechanism is no longer needed.

### Related Proposals: 

None.

## Proposal

Remove grpc_register_plugin.

## Rationale

This mechanism has never been useful as a public API (it's only useful to call internal registration mechanisms - i.e. those without a public API).

It blocks the deletion of grpc_init/grpc_shutdown, which will be coming soon.

## Implementation

First roll the final usages directly into grpc_init/grpc_shutdown and then remove them from the public API.
