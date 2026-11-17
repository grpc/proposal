Provide a functionality to cleanup all static-duration objects allocated by the library
----
* Author(s): stefan301
* Approver: a11r
* Status: Draft
* Implemented in: C, C++
* Last updated: July 19, 2020
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/oLLAsP8hcKM

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

Add `grpc_final_shutdown_library()` which can be called to delete all static-duration objects allocated by the library.
There are two reasons you might want to call this:

    - You use a draconian definition of "memory leak" in which you expect every
      single malloc()/new() to have a corresponding free()/delete(), even for objects which live
      until program exit.
    - You are writing a dynamically-loaded library which needs to
      clean up after itself when the library is unloaded.

Before it's called, the normal cleanup using `grpc_shutdown()` has to be done.  
The library can be (re)initialized and shutdown as often as needed with `grpc_init()`/`grpc_shutdown()`.
`grpc_final_shutdown_library()` is not a replacement for `grpc_shutdown()` but can be called after the normal
cleanup with `grpc_shutdown()` when using automatic leak-detection at process exit.  
If you don't use automatic leak-detection then you don't need to call this function.
After calling `grpc_final_shutdown_library()` the library cannot be reinitialized again.


```c
GRPCAPI void grpc_final_shutdown_library(void);
```

### functions for use in the internal implementation of gRPC Core and C++

Add a function `grpc_on_shutdown_callback()` to register a cleanup callback with no arguments and a function
`grpc_on_shutdown_callback_with_arg()` to register a cleanup callback together with a void* argument.

```c
GRPCAPI void grpc_on_shutdown_callback(void (*func)());
GRPCAPI void grpc_on_shutdown_callback_with_arg(void (*f)(const void*),
                                                const void* arg);
```

### Add convenience functions OnShutdownDelete, OnShutdownFree.

These functions should take the pointer returned by `new`/`malloc`, register the `delete`/`free` call and
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

### Usage example (in the internal implementation of gRPC Core and C++)

```c++
// src\cpp\client\client_context.cc
static DefaultGlobalClientCallbacks* g_default_client_callbacks =
    grpc_core::OnShutdownDelete(new DefaultGlobalClientCallbacks());

// src\core\lib\surface\init.cc 
  g_shutting_down_cv =
      static_cast<gpr_cv*>(grpc_core::OnShutdownFree(malloc(sizeof(gpr_cv))));
```

### Usage example (end-user client/server applications)

```c++
int main(int argc, char** argv) {

  // use gRPC: grpc_init(), ..., grpc_shutdown(), grpc_init(), ..., grpc_shutdown()
  
  grpc_final_shutdown_library();
  return 0;
}
```

## Rationale

This approach was chosen because it already works for libprotobuf for many years.
  
It minimises the effort for the library programmer as well as for the library user and
it is not a breaking change, only one new public function that can be called if needed.

## Implementation

The core functionality and some `OnShutdownDelete`/`OnShutdownFree` calls are implemented in this PR:

https://github.com/grpc/grpc/pull/23362

Any allocation of a static-duration object should now and in the future be wrapped to ensure a
full cleanup when `grpc_final_shutdown_library()` is called.

## Open issues (if applicable)

N/A

## Testing

I don't know how testing can be done in a portable way for gRPC. On Windows after changing the constructor
of TestEnvironment to enable leak-detection nearly every test dumps memory leaks.

```c++
TestEnvironment::TestEnvironment(int argc, char** argv) {
  _CrtSetReportMode(_CRT_WARN, _CRTDBG_MODE_FILE);
  _CrtSetReportFile(_CRT_WARN, _CRTDBG_FILE_STDOUT);
  _CrtSetDbgFlag(_CRTDBG_ALLOC_MEM_DF | _CRTDBG_LEAK_CHECK_DF);
  grpc_test_init(argc, argv);
}
```
for example init_test.exe:

```
init_test.exe
D0720 23:13:35.451000000 34824 test_config.cc:386] test slowdown factor: sanitizer=1, fixture=1, poller=1, total=1
Detected memory leaks!
Dumping objects ->
{161} normal block at 0x01537F58, 8 bytes long.
 Data: < z< 0   > 04 7A 3C 00 30 D3 FA 01
{160} normal block at 0x01537C48, 8 bytes long.
 Data: <X<S     > 58 3C 53 01 00 00 00 00
{159} normal block at 0x01533C58, 44 bytes long.
 Data: <H|S X S ` S ` S > 48 7C 53 01 58 7F 53 01 60 7F 53 01 60 7F 53 01
{158} normal block at 0x01FAD330, 4 bytes long.
 Data: <    > 00 00 00 00
Object dump complete.
```

After adding grpc_final_shutdown_library() to the destructor of TestEnvironment no
memory leaks are dumped except real memory leaks or when grpc_core::OnShutdownDelete or
grpc_core::OnShutdownFree is missing.

```c++
TestEnvironment::~TestEnvironment() {
  ...
  gpr_log(GPR_INFO, "TestEnvironment ends");
  grpc_final_shutdown_library();
}
```

```
init_test.exe
D0720 23:17:30.048000000 11500 test_config.cc:386] test slowdown factor: sanitizer=1, fixture=1, poller=1, total=1
```
