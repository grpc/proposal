Title
----
* Author(s): sreenithi
* Approver: sergiitk, asheshvidyut
* Status: Draft
* Implemented in: Python
* Last updated: 04-07-2025
* Discussion at: <google group thread> (filled after thread exists)
xxxxxxx
## Abstract

This proposal describes the cahnges necessary to add absl logging support in gRPC Python hence solving the user warning:

```sh
WARNING: All log messages before absl::InitializeLog() is called are written to STDERR
``` 


## Background

gRPC Core changed its logging system to use absl logging as per [L117](https://github.com/grpc/proposal/blob/master/L117-core-replace-gpr-logging-with-abseil-logging.md).

After this change, gRPC python users started seeing the warning log - 
`WARNING: All log messages before absl::InitializeLog() is called are written to STDERR`.

For gRPC C++, the users are expected to invoke `absl::InitializeLog()`, but gRPC Python users cannot invoke a C++ API call. Hence this proposal explains the changes necessary to gRPC Python to implicitly invoke `absl::InitializeLog()` in the Cython layer at the time of initialization.

> Note that if abseil fixes https://github.com/abseil/abseil-cpp/issues/1656, gRPC Core itself will be able to invoke absl::InitializeLog(), but until then this is a workaround.

### Related Proposals: 
* https://github.com/grpc/proposal/blob/master/L117-core-replace-gpr-logging-with-abseil-logging.md

## Proposal

To resolve the warning and include the call to `absl::InitializeLog()` within gRPC Python, the following changes must be made:

1. Add `abseil-cpp/absl/log/initialize.cc` as a dependency in the Cython layer
2. Call `absl::InitializeLog()` in `src/python/grpcio/grpc/_cython/cygrpc.pyx` automatically at the time of initialization.

However, abseil-cpp currently [doesn't allow `absl::InitializeLog()` to be called more than once](https://github.com/abseil/abseil-cpp/issues/1656), and will result in an error like:

```
[globals.cc : 105] RAW: absl::log_internal::SetTimeZone() has already been called
```

The above changes may hence cause problems to a small percentage of users who may be calling `absl::InitializeLog()` themselves or may be dependent on other libraries which already invokes `absl::InitializeLog()`.

To solve this problem, users will be provided with the option of setting an environment variable `DISABLE_ABSL_INIT_LOG` to opt-out of this automatic abseil log initialization.


## Rationale


An alternate approach that was considered to provide users with the option of disabling automatic absl log initialization is by using an additional import statement in the code that will disable the call to `absl::InitializeLog()` along with the usual import statement like:

```py
import grpc.no_absl_log_init
import grpc
```

However, we considered that this might cause too much overhead to users to add an additional import statement everywhere in the codebase that calls `import grpc`. And it may also lead to situations where some part of users' codebase may include the additional import statement while others don't, hence causing conflicting behaviour.

Hence, the current approach to allow users to disable using an enironment variable was chosen.


Secondly, we also tried adding `absl/log/initialize.cc` directly as a Core dependency instead of a specific Cython level dependency. But as we have multiple languages wrapped on the core layer, this implied that every wrapped language got an additional dependency on `absl/log/initialize.cc`. This is not too much of an overhead given that this is a single additional file and Core already depends on the abseil package. 

However this file is only needed for Python, and hence it seemed to be a better option to use this file as Python specific dependency without affecting Core and other wrapped languages.


## Implementation

The automatic call to `absl::InitializeLog()` will be included in the `_initialize()` Cython function of `src/python/grpcio/grpc/_cython/cygrpc.pyx` file as follows:
```py
## src/python/grpcio/grpc/_cython/cygrpc.pyx

# absl::InitializeLog() imported in grpc.pxi
# cdef extern from "absl/log/initialize.h" namespace "absl":
#  void InitializeLog() nogil

disable_absl_init_log = os.getenv("DISABLE_ABSL_INIT_LOG")

#
# initialize gRPC
#
cdef _initialize():
  if not disable_absl_init_log:
    InitializeLog()

  # rest of the function

_initialize()
```

### Implemented in PR:
https://github.com/grpc/grpc/pull/39779/