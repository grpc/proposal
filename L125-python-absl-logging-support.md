Title
----
* Author(s): sreenithi
* Approver: sergiitk, asheshvidyut
* Status: Draft
* Implemented in: Python
* Last updated: 17-09-2025
* Discussion at: <google group thread> (filled after thread exists) xxxxxxx

## Abstract

This proposal describes the changes necessary to add absl logging support in
gRPC Python hence solving the user warning:

```sh
WARNING: All log messages before absl::InitializeLog() is called are written to STDERR
```


## Background

gRPC Core changed its logging system to use absl logging as per
[L117][L117].

After this change, gRPC python users started seeing the warning log -
`WARNING: All log messages before absl::InitializeLog() is called are written to STDERR`.

For gRPC C++, the users are expected to invoke `absl::InitializeLog()`, but gRPC
Python users cannot invoke a C++ API call. Hence this proposal explains the
changes necessary to gRPC Python to implicitly invoke `absl::InitializeLog()`
in the Cython layer at the time of initialization.

> Note that if abseil fixes https://github.com/abseil/abseil-cpp/issues/1656,
> gRPC Core itself will be able to invoke absl::InitializeLog(), but until then
> this is a workaround.

### Related Proposals:
* [L117: C-core: Replace gpr Logging with absl Logging][L117]

[L117]: L117-core-replace-gpr-logging-with-abseil-logging.md

## Proposal

To resolve the warning and include the call to `absl::InitializeLog()` within
gRPC Python, the following changes must be made:

1. Add `abseil-cpp/absl/log/initialize.cc` as a dependency in the Cython layer
2. Call `absl::InitializeLog()` in `src/python/grpcio/grpc/_cython/cygrpc.pyx`
automatically at the time of initialization.

However, abseil-cpp currently
[doesn't allow `absl::InitializeLog()` to be called more than once](https://github.com/abseil/abseil-cpp/issues/1656),
and will result in an error like:

```
[globals.cc : 104] RAW: absl::log_internal::SetTimeZone() has already been called
```

The above changes may hence cause problems to a small percentage of users who
may be calling `absl::InitializeLog()` themselves or may be dependent on other
libraries which already invokes `absl::InitializeLog()`.

To solve this problem, users will be provided with the option of setting an
environment variable `DISABLE_ABSL_INIT_LOG` to opt-out of this automatic abseil
log initialization.


## Rationale

### Alternative approach

An alternate approach that was considered to provide users with the option of
disabling automatic absl log initialization is by using an additional import
statement in the code that will disable the call to `absl::InitializeLog()`
along with the usual import statement like:

```py
import grpc.no_absl_log_init
import grpc
```

However, we considered that this might cause too much overhead to users to add
an additional import statement everywhere in the codebase that calls
`import grpc`. And it may also lead to situations where some part of users'
codebase may include the additional import statement while others don't, hence
causing conflicting behaviour.

Hence, the current approach to allow users to disable using an enironment
variable was chosen.


## Implementation

`absl::InitializeLog()` will be imported in
`_cython/_cygrpc/grpc.pxi`, and the automatic call to it
will then be included in the `_initialize()` function of
`_cython/cygrpc.pyx` file as follows:

```pxi
## src/python/grpcio/grpc/_cython/_cygrpc/grpc.pxi

# absl::InitializeLog() imported in grpc.pxi
# cdef extern from "absl/log/initialize.h" namespace "absl":
#  void InitializeLog() nogil
```

```pyx
## src/python/grpcio/grpc/_cython/cygrpc.pyx

disable_absl_init_log = os.environ.get("DISABLE_ABSL_INIT_LOG")

#
# initialize gRPC
#
cdef _initialize():
  if not disable_absl_init_log:
    InitializeLog()

  # rest of the function

_initialize()
```

### Behaviour with multiple imports of gRPC

Given that `absl::InitializeLog()` must be called exactly once, we had to
consider if this current implementation can cause any issues with multiple calls
to `import grpc` in scenarios like:

1. multiple imports for type checking
2. multiple imports in a multi-threaded scenario
3. consequent imports to `grpc` and `grpc.experimental` (which implicitly
imports cygrpc again)
4. manual reload of the imported modules
5. multiprocessing environments with imports to grpc in the parent and child
process.
6. multiprocessing environments with imports to grpc only in multiple child
processes.

All the above scenarios were tested and found safe. As the InitializeLog()
function is a part of the Cython layer (which is compiled into a C binary at
build time), Python's import system guarantees that it will run only once when
a process first imports grpc even on manual reload.

All other tested scenarios like threads or child processes — either reuse this
single, first-time setup or (if they are a new, clean process) safely run their
own single setup. In all cases, the function is never run twice in a way that
would cause an error.

(For a more detailed explanation and examples of each scenario tested,
see [PR #40724](https://github.com/grpc/grpc/pull/40724/).)

### Implemented in PR:
https://github.com/grpc/grpc/pull/39779/
