Provide a functionality to cleanup all static-duration objects allocated by the library
----
* Author(s): stefan301
* Approver: a11r
* Status: Draft
* Implemented in: C, C++
* Last updated: July 19, 2020
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

Provide a functionality to cleanup all static-duration objects allocated by the library.

## Background

When using gRPC in an application which does a heap check at process exit to find
"memory leaks" there should be a way to cleanup all static-duration objects
allocated by the gRPC library - otherwise the "memory leak" dump becomes useless.
Therefore a functionality similar to ShutdownProtobufLibrary() should be implemented.

https://github.com/grpc/grpc/issues/23330

### Related Proposals: 

N/A

## Proposal

### cleanup function for public use by end-user client/server applications

Add grpc_final_shutdown_library() which can be called to delete all static-duration objects allocated by the library.
There are two reasons you might want to call this:

    - You use a draconian definition of "memory leak" in which you expect every
      single malloc()/new() to have a corresponding free()/delete(), even for objects which live
      until program exit.
    - You are writing a dynamically-loaded library which needs to
      clean up after itself when the library is unloaded.

Before it's called, the normal cleanup using grpc_shutdown has to be done.

```c
GRPCAPI void grpc_final_shutdown_library(void);
```

### functions for use in the internal implementation of gRPC Core and C++

Add a function grpc_on_shutdown_callback() to register a cleanup callback with no arguments and a function
grpc_on_shutdown_callback_with_arg() to register a cleanup callback together with a void* argument.

```c
GRPCAPI void grpc_on_shutdown_callback(void (*func)());
GRPCAPI void grpc_on_shutdown_callback_with_arg(void (*f)(const void*),
                                                const void* arg);
```

### Add convenience functions OnShutdownDelete, OnShutdownFree.

These functions should take the pointer returned by new/malloc, register the delete/free call and
transparently return the pointer.

```c++
namespace grpc_core {
    
template <typename T>
T* OnShutdownDelete(T* p) {
  grpc_on_shutdown_callback_with_arg(
      [](const void* pp) { delete static_cast<const T*>(pp); }, p);
  return p;
}

inline void* OnShutdownFree(void* p) {
  grpc_on_shutdown_callback_with_arg(
      [](const void* pp) { free(const_cast<void *>(pp)); }, p);
  return p;
}

}  // namespace grpc_core
```

### Usage example 

```c++
// src\cpp\client\client_context.cc
static DefaultGlobalClientCallbacks* g_default_client_callbacks =
    grpc_core::OnShutdownDelete(new DefaultGlobalClientCallbacks());

// src\core\lib\surface\init.cc 
  g_shutting_down_cv =
      static_cast<gpr_cv*>(grpc_core::OnShutdownFree(malloc(sizeof(gpr_cv))));
```

## Rationale

N/A


## Implementation

The core functionality and some OnShutdownDelete/OnShutdownFree calls are implemented in this PR:

https://github.com/grpc/grpc/pull/23362

Any allocation of a static-duration object should now and in the future be wrapped to ensure a
full cleanup when grpc_final_shutdown_library() is called.

## Open issues (if applicable)

N/A
