gRPC Python Server Wait API
----
* Author(s): lidiz
* Approver: gnossen
* Status: Approved
* Implemented in: Python
* Last updated: June 12, 2019
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/GhJl_JHfaHA

## Abstract

Add a `wait_for_termination` API for Python server that can block in any thread
until the server is stopped or terminated.

## Background

Currently, the gRPC Python server only provides a `server.start()` API, but
there isn't any API to block forever. gRPC Python server applications need to
block on the main thread to keep serving explicitly. The solution the gRPC
Python team proposed in examples is sleeping for a while then exit. See code:

```Python
_ONE_DAY_IN_SECONDS = 60 * 60 * 24

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    route_guide_pb2_grpc.add_RouteGuideServicer_to_server(
        RouteGuideServicer(), server)
    server.add_insecure_port('[::]:50051')
    server.start()
    try:
        while True:
            time.sleep(_ONE_DAY_IN_SECONDS)
    except KeyboardInterrupt:
        server.stop(0)
```

Developers are running server longer than one day, and hope it to keep running
forever if possible. gRPC users are coming up with workarounds to block the main
thread:

1) Use other API to block like `select.select`;
2) Sleep longer.

Ideally, this feature should be part of gRPC library, because only gRPC library
have the clearest signal when the server is going to shutdown. This feature
existed in C++ as
[`Wait()`](https://grpc.github.io/grpc/cpp/classgrpc_1_1_server_interface.html#ac36477b6a7593a4e4608c7eb712b0d70),
in Java as `blockUntilShutdown()`, in Golang as `Serve`. This gRFC is
introducing the Python version of it.

### Related Proposals: 

None.

## Proposal

This gRFC proposes that we add the following API to `grpc.Server` interface:

```Python
class Server:

    def wait_for_termination(self, timeout=None):
        """Block current thread until the server stops.

        This is an EXPERIMENTAL API.

        The wait will not consume computational resources during blocking, and
        it will block until one of the two following conditions are met:

        1) The server is stopped or terminated;
        2) A timeout occurs if timeout is not `None`.
        
        The timeout argument works in the same way as `threading.Event.wait()`.
        https://docs.python.org/3/library/threading.html#threading.Event.wait

        Args:
          timeout: A floating point number specifying a timeout for the
            operation in seconds.
        """
        raise NotImplementedError()
```

## Rationale

### Alternative Design: Await For Signal

The termination of a long-running process usually come as a SIGTERM signal. So,
in this version of the design, the method adds a signal handler that cleans up
resources and delays the shutdown if necessary. However, due to that, CPython
can only register the signal handler in the main thread. Calling this function
in another thread is problematic. It means we cannot properly use it in our unit
tests.

```Python
class Server:

    def wait_for_termination(self, delay=None, grace=None):
        """Block current thread until the server stops.
        
        This is an EXPERIMENTAL API.
        
        The wait will not consume computational resources during blocking, and
        it will block indefinitely. To unblock, you will need to send a SIGTERM
        signal to current process.

        Caveat: This function only work in main thread. Invoked in other thread
        may block forever or raise exceptions.

        Args:
          delay: The length of delaying server shutdown process. In this
                 period, server is still allowed to accept new request.
          grace: The length of server in grace mode. New requests will be
                 rejected, but ongoing requests are allowed to finish.
        """
        raise NotImplementedError()
```

Besides the main-thread restriction, there are several more issues:

1) The `grace` variable set here is not necessary the source of truth. Other
   thread can call `server.Stop` as well;
2) The semantic of delay is hard to define. It has to define which subset of
   signals should it handle, and should each signal have different behaviors.

### Alternative Design: Return a Threading.Event

Proposed by @mehrdada in https://github.com/grpc/grpc/pull/19299.

A simplified design will be provide the application necessary information to
let them handle the blocking logic themselves.

```Python
class Server:

    def termination_event(self) -> threading.Event:
        """Returns a event that will set when the server terminates.

        Usage like:
        server = grpc.server(...)
        server.start()
        server.termination_event().wait()

        Returns:
          A threading.Event object.
        """
        raise NotImplementedError()
```

The down side of this deign:
1) Adds cognitive cost for users to understand how `threading.Event` work;
2) The semantic is not that clear as `wait_for_termination` suggesting.
3) @mehrdada Users may `set` or `clear` this event.

## Implementation

PR https://github.com/grpc/grpc/pull/19299

## Open issues (if applicable)

Python equivalent of C++ server's Wait() method
[#17452](https://github.com/grpc/grpc/issues/17452)
