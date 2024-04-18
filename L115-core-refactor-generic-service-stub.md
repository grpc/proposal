L115: C-Core Refactor generic service and generic stub
----
* Author(s): ysseung
* Approver: ctiller
* Status: Draft
* Implemented in: C Core
* Last updated: 2024/04/15
* Discussion at: https://groups.google.com/g/grpc-io/c/301w3zXYf8o

## Abstract

We will refactor `async_generic_service.h` and `generic_stub.h` to move
callback based interfaces into separate targets. Existing files will include
the callback based interfaces.

## Background

This enables services to only depend on the callback based interfaces of the
generic stub and generic service.

### Related Proposals

None.

## Proposal

We will move `CallbackGenericService` in `async_generic_service.h` to a new file
`async_generic_service_callback.h` and make it a new public target
`:generic_service_callback`. `async_generic_service.h` will include 
`async_generic_service_callback.h` and existing dependencies will continue to
include both.

We will also move move callback based interfaces of `TemplatedGenericStub` in
`generic_stub.h` to a new class `TemplatedGenericStubCallback` in
`generic_stub_callback.h` and make it a new public target
`:generic_stub_callback`. `CallbackGenericService` will inherit
`CallbackGenericServiceCallback` and existing dependencies will continue to
include both.

## Implementation

TBA
