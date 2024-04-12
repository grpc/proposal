L99: C-Core Eliminate Corking
----
* Author(s): ctiller
* Approver: markdroth
* Status: In Review
* Implemented in: C Core
* Last updated: 8/2/2022
* Discussion at: https://groups.google.com/g/grpc-io/c/6GzDzAoiySk

## Abstract

Remove the GRPC_INITIAL_METADATA_CORKED flag.

## Background

This flag does nothing inside core.

## Proposal

Remove the flag and all references in public API.

## Rationale

This flag was originally introduced to support corking initial metadata, but that's now entirely handled in binding layers, so the flag is no longer useful.

## Implementation

Remove the flag and references to it: https://github.com/grpc/grpc/pull/30443
