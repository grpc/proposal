Nogil Mode Support in gRPC Python
----
* Author(s): Aditya Ajayshankar (adityaajay@google.com)
* Approver: a11r
* Status: Draft
* Implemented in: Python, Cython
* Last updated: 2025-06-22
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

This aims to allow gRPC Python to function without the Global Interpreter Lock (GIL), a mutex that restricts Python bytecode execution to a single thread at any given time. Removing the GIL will substantially boost the performance of the library's blocking concurrency model.


## Background

Python’s GIL allows only one thread to execute Python bytecode at a time, even on multi-core processors. Running Python on free-threaded mode (GIL disabled) can significantly improve the efficiency and performance of the gRPC Python library, especially for applications that are I/O-bound and leverage multithreading.

With the GIL removed, Python threads can truly execute in parallel on different CPU cores. In the case of CPU-intensive components, free-threaded mode would allow these operations to run simultaneously across multiple cores, leading to substantial speedups.

In the free-threaded mode, the overhead associated with GIL acquisition and release around Python-level operations is eliminated. This can lead to smoother context switching and better overall throughput when handling a high volume of concurrent gRPC requests. 

Refer https://github.com/grpc/grpc/issues/38762 for request for nogil mode issue.

## Context

In order to add support for nogil mode, we used the approach of running our existing test suite with the nogil mode enabled and making necessary changes to resolve the errors and hence add nogil mode support.
**Python 3.13t** free-threaded build was used to run the gRPC test suite on nogil mode through Bazel.

Test run command with GIL disabled:
```
bazel test --test_env=PYTHON_GIL=0 “//src/python/path_to_test”
```
Majority of the tests passed, around 7 aio tests were failing due to segmentation faults.


## Proposal


### Solving Segfaults in AsyncIO Unit Tests

7 aio tests failed with segmentation faults. In each test file, individual test cases were passing with status OK but the test was segfaulting at the end, suggesting that segfaults were occurring at the **tearDown()** function in the unit tests, in which the **self._server.stop(None)** function call was found to be the cause of the segfaults, as commenting out these lines in the tests was eliminating the segmentation faults.

On observing the gdb stack trace for the aio tests, the issue was found to be originating in *src/python/grpcio/grpc/_cython/_cygrpc/aio
/completion_queue.pyx.pxi*, in the _poll() function. 
A warning was noticed while running the tests:
```
performance hint: src/python/grpcio/grpc/_cython/_cygrpc/aio/completion_queue.pyx.pxi:137:22: Exception check after calling '_poll' will always require the GIL to be acquired.
Possible solutions:
	1. Declare '_poll' as 'noexcept' if you control the definition and you're sure you don't want the function to raise exceptions.
	2. Use an 'int' return type on '_poll' to allow an error code to be returned.

```
Taking hints from the warning, the proposed changes have been described in detail in the implementation section.

### Segfaults in Bazel Runner Script

**//src/python/grpcio_tests/tests/status:grpc_status_test**, **//src/python/grpcio_tests/tests_aio/status:grpc_status_test**, **//src/python/grpcio_tests/tests/observability:_observability_plugin_test**:
The above 3 tests also fail with segfaults, but don’t have any individual test cases passing like the previously mentioned aio tests.

On implementing the changes, the aio tests pass.
**//src/python/grpcio_tests/tests/status:grpc_status_test**, **//src/python/grpcio_tests/tests_aio/status:grpc_status_test**
The above mentioned status tests pass on running via Python build, but still fail due to segfaults on running through Bazel, along with the following error message:
```
Executing tests from //src/python/grpcio_tests/tests_aio/status:grpc_status_test
-----------------------------------------------------------------------------
external/bazel_tools/tools/test/test-setup.sh: line 337: 1630213 Segmentation fault      (core dumped) "${TEST_PATH}" "$@" 2>&1
     1630215 Done                    | tee -a "${XML_OUTPUT_FILE}.log"
Segmentation fault (core dumped)
```
**//src/python/grpcio_tests/tests/observability:_observability_plugin_test** gives the same error message.

The tests seem to be failing due to some issue in the bazel source, as seen in the above message, and are also most likely unrelated to the free-threaded mode, as the status tests do run successfully on running through Python source build. Debugging the bazel source issue is beyond the scope of this project, and it is proposed to skip these tests when running on free-threaded mode via Bazel.

### UnicodeDecodeError in Observability Tests

**//src/python/grpcio_tests/tests/observability:_open_telemetry_observability_test** fails due to timeout with the following error:
```
File "src/python/grpcio_observability/grpc_observability/_cyobservability.pyx", line 371, in _cyobservability._decode
UnicodeDecodeError: 'utf-8' codec can't decode byte 0xdd in position 0: invalid continuation byte

```

## Rationale

The proposed changes do not solve the issue of the status tests failing through Bazel, and skipping those particular tests on Bazel might not be the best approach, and need to be investigated upon in the future.


## Implementation

For the aio tests, in *src/python/grpcio/grpc/_cython/_cygrpc/aio/completion_queue.pyx.pxi*, the _poll() function was declared as ‘noexcept’ which eliminated the segmentation faults in the tests:
```
cdef void _poll(self) noexcept nogil:
    cdef grpc_event event
    cdef CallbackContext *context

    while not self._shutdown:
        event = grpc_completion_queue_next(self._cq,_GPR_INF_FUTURE, NULL)
        if event.type == GRPC_QUEUE_TIMEOUT:
            with gil:
                raise AssertionError("Core should not return GRPC_QUEUE_TIMEOUT!")
        elif event.type == GRPC_QUEUE_SHUTDOWN:
            self._shutdown = True
        else:
            self._queue_mutex.lock()
            self._queue.push(event)
            self._queue_mutex.unlock()
            if _has_fd_monitoring:
                _unified_socket_write(self._write_fd)
            else:
                with gil:
                    self._handle_events(None)
```

However, this is not sufficient as a fix, as the function won’t raise any Python exceptions (in case any were to occur) as it has been marked as noexcept. 
Hence, make the function of int return type to be able to safely return error codes without any need of a with gil block, because returning an error code as an integer is simply a C level operation.

```
cdef int _poll(self) noexcept nogil:
    cdef grpc_event event
    cdef CallbackContext *context

    while not self._shutdown:
        event = grpc_completion_queue_next(self._cq,_GPR_INF_FUTURE, NULL)

        if event.type == GRPC_QUEUE_TIMEOUT:
            return 1
        elif event.type == GRPC_QUEUE_SHUTDOWN:
            self._shutdown = True
        else:
            self._queue_mutex.lock()
            self._queue.push(event)
            self._queue_mutex.unlock()
            if _has_fd_monitoring:
                _unified_socket_write(self._write_fd)
            else:
                with gil:
                    self._handle_events(None)
    return 0

def _poll_wrapper(self):
    cdef int poll_result 
    with nogil:
        poll_result = self._poll()
    if poll_result == 1:
        raise AssertionError("Core should not return GRPC_QUEUE_TIMEOUT!")
```
Now the *_poll_wrapper()* function receives an error code from the *_poll()* function and can safely raise a Python exception as it is a Python function.
By marking the *_poll()* function as *noexcept*, we are making a strong promise to Cython that this function will not propagate Python exceptions. This tells Cython to disable automatic GIL acquisition/exception checks on the return path from *_poll* to *_poll_wrapper*. The *cdef _poll()* function initially without the "noexcept" would require the GIL to be held for exception checking, which would be a risky operation in the free-threaded mode and lead to race conditions/segfaults.
The int return type correctly communicates to Cython that *_poll* returns a C-level integer, allowing *_poll_wrapper* to explicitly check for error codes.

(Refer: https://github.com/grpc/grpc/pull/37922, https://github.com/grpc/grpc/pull/37917)



