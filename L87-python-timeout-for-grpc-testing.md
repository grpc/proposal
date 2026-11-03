Add timeout parameter to TestingChannel in grpc_testing (Python)
----
* Author(s): Charles Saternos
* Approver: @gnossen
* Status: Draft + PR
* Implemented in: python
* Last updated: 2021-10-22
* PR: https://github.com/grpc/grpc/pull/27312
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

This change adds an optional `timeout` parameter to
`TestingChannel.take_{unary_unary,unary_stream,stream_unary,stream_stream}`
in the Python `grpc_testing` library. This allows the user to specify a timeout that
will prevent the test from blocking the main thread indefinitely if the client
fails to make an RPC due to an issue in the test.

## Background

This RFC attempts to address a challenge developers face when writing unit
tests for gRPC clients with the Python `grpc_testing` library. If the
client fails to make an RPC request to the testing channel, the server
blocks the main thread indefinitely. If the test is working properly,
this isn't an issue, since the client will make the necessary RPC that keeps
the test moving. However, if an exception is thrown in the client code,
the server thread will hang since the channel isn't populated with the expected
request.

This makes writing tests challenging since each request must be precisely paired
up in the client and server. If the server doesn't get the request it's expecting,
it will hang the whole program (and test suite).

Consider the following test code:

```python
def test_request_response(self):
  pool = logging_pool.pool(1)
  channel = grpc_testing.channel(services_pb2.DESCRIPTOR.services_by_name.values(),
                                 grpc_testing.strict_real_time())
  app_future = pool.submit(client_make_request, channel)
  meta, request, rpc = channel.take_unary_unary(method_descriptor)
  rpc.send_initial_metadata(())
  rpc.terminate(services_pb2.Response(data='foo'), (), grpc.StatusCode.OK, '')
  response_to_client = app_future.result()
  self.assert_stuff(response_to_client)
```

If `client_make_request` throws an exception before it can actually make the gRPC
request, `channel.take_unary_unary` will hang, waiting for a request that will
never come because the client is dead. The desired behavior would be that the
test fails since the client failed to make the necessary request.

Currently there isn't an easy way of terminating the test if the
client fails in this way, since the `channel.take_unary_unary` call is
blocking. The only workaround I found is to spawn the server function as
a daemon thread, then call `server_thread.join(timeout=timeout_seconds)`,
which will at least timeout the server functionality and unblock the test.
But it still leaves the server thread running in the background until
the process exits, which isn't great since it leaks a new thread
every time an expected request isn't sent.

### Related Proposals:

None that I'm aware of.

## Proposal

Add a timeout kwarg parameter to the `TestingChannel.take_*` functions. This
change is backwards compatible, and completely optional if a user doesn't want to
opt into a timeout.

Example usage:

```python
def test_request_response_with_timeout(self):
  # ... set up request ...
  try:
     meta, request, rpc = channel.take_unary_unary(method_descriptor,
                                                   timeout=timeout_in_seconds)
  except TimeoutException:
     self.fail("Server never received expected request from client" + str(method_descriptor))
  # ... send the response, check the client response result, etc ...
```

The change is implemented in a PR here: https://github.com/grpc/grpc/pull/27312

## Rationale

This changes is a small, non-breaking, backwards compatible way of dealing with
client failures in a test case. The change allows developers to write and run
gRPC client unit tests more confidently since failures won't hang the rest of
the test suite.

## Implementation

The change is implemented in a PR here: https://github.com/grpc/grpc/pull/27312

The implementation adds a `timeout` parameter to `TestingChannel.take_{unary_unary,unary_stream,stream_unary,stream_stream}`
and passes the value down to the timeout on the `threading.Condition` in
`State.take_rpc_state` that blocks the main thread when the expected request
isn't on the channel.

The change to `take_rpc_state` is simply:

```
-    def take_rpc_state(self, method_descriptor):
+    def take_rpc_state(self, method_descriptor, timeout):
         method_full_rpc_name = '/{}/{}'.format(
             method_descriptor.containing_service.full_name,
             method_descriptor.name)
@@ -44,4 +44,5 @@ def take_rpc_state(self, method_descriptor):
                 if method_rpc_states:
                     return method_rpc_states.pop(0)
                 else:
-                    self._condition.wait()
+                    if not self._condition.wait(timeout=timeout):
+                        raise TimeoutException("Timeout while waiting for rpc.")
```

If the `TestingChannel.take_*` function doesn't specify a timeout, the default value
`None` is passed to the condition which will make the `wait` run without a timeout
maintaining backwards compatible behavior with the current implementation.
