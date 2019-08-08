Async API for gRPC Python
----
* Author(s): lidizheng
* Approver: gnossen
* Status: In Review
* Implemented in: Python
* Last updated: 2019-08-07
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/7V7HYM_aph4

## Abstract

A brand new set of async API that will solve concurrency issues and performance
issues for gRPC Python, which is available to Python 3.6+.

## Motivation

* Asynchronous processing perfectly fits IO-intensive gRPC use cases;
* Resolve a long-living design flaw of thread exhaustion problem;
* Performance is much better than the multi-threading model.

## Background

### What Is Asynchronous I/O In Python?

Quote from [`asyncio`](https://docs.python.org/3/library/asyncio.html) package
documentation:

> `asyncio` is a library to write concurrent code using the `async`/`await`
> syntax.
>
> `asyncio` is used as a foundation for multiple Python asynchronous frameworks
> that provide high-performance network and web-servers, database connection
> libraries, distributed task queues, etc.

In the asynchronous I/O model, the computation tasks are packed as generators
(aka. coroutines) that will yield the ownership of thread while it is blocked by
IO operations or wait for other tasks to complete.

The design of coroutine in Python can trace back to Python 2.5 in [PEP 342 --
Coroutines via Enhanced Generators](https://www.python.org/dev/peps/pep-0342/).
It introduces new generator methods like `send`, `throw`, and `close`. With
these methods, the caller of the generator can pass values and exceptions into a
paused generator function, hence diminish the need for wiring complex callbacks.

To further simplify the usage of a coroutine, Python 3.5 introduces
`async`/`await` syntax that can easily transform a normal function into a
coroutine ([PEP 492](https://www.python.org/dev/peps/pep-0492/)). And the
`Twisted` style event loop abstractions is introduced in [PEP
3156](https://www.python.org/dev/peps/pep-3156/).

Another important piece is the asynchronous generator. It is introduced in
Python 3.6 by [PEP 525](https://www.python.org/dev/peps/pep-0525/).

To read more about `asyncio` implementation in CPython, see:
* [C-Level
  implementation](https://github.com/python/cpython/blob/3.7/Modules/_asynciomodule.c)
* [Python-Level
  implementation](https://github.com/python/cpython/blob/3.7/Lib/asyncio)

### Python's Future

Currently, there are two futures in the standard library.

* [`concurrent.futures.Future`](https://github.com/python/cpython/blob/master/Lib/concurrent/futures/_base.py#L313)
  added in Python 3.2
* [`asyncio.Future`](https://github.com/python/cpython/blob/master/Lib/asyncio/futures.py)
  added in Python 3.4

They are built for different threading models. Luckily, the CPython maintainers
are actively trying to make them compatible (they are **almost** compatible). By
the time this design doc is written, comparing to `concurrent.futures.Future`,
`asyncio.Future` has the following difference:

> - This class is not thread-safe.
>
>- result() and exception() do not take a timeout argument and raise an
>  exception when the future isn't done yet.
>
>- Callbacks registered with add_done_callback() are always called via the event
>  loop's call_soon().
>
>- This class is not compatible with the wait() and as_completed() methods in
>  the concurrent.futures package.

gRPC Python also has its definition of
[`Future`](https://grpc.github.io/grpc/python/grpc.html#future-interfaces)
interface since 2015. The concurrency story by that time is not that clear as
now. Here is the reasoning of why gRPC Python needed a dedicated `Future`
object, and incompatibilities:

> Python doesn't have a Future interface in its standard library. In the absence
> of such a standard, three separate, incompatible implementations
> (concurrent.futures.Future, ndb.Future, and asyncio.Future) have appeared.
> This interface attempts to be as compatible as possible with
> concurrent.futures.Future. From ndb.Future it adopts a traceback-object
> accessor method.
>
> Unlike the concrete and implemented Future classes listed above, the Future
> class defined in this module is an entirely abstract interface that anyone may
> implement and use.
>
> The one known incompatibility between this interface and the interface of
> concurrent.futures.Future is that this interface defines its own
> CancelledError and TimeoutError exceptions rather than raising the
> implementation-private concurrent.futures._base.CancelledError and the
> built-in-but-only-in-3.3-and-later TimeoutError.

Although the design of `Future` in Python finally settled down, it's **not
recommended** to expose low-level API like `asyncio.Future` object. The Python
documentation suggests that we should let the application to decide which
`Future` implementation they want to use, and hide the ways to operate them
directly.

> The rule of thumb is to never expose Future objects in user-facing APIs, and
> the recommended way to create a Future object is to call loop.create_future().
> This way alternative event loop implementations can inject their own optimized
> implementations of a Future object.

### Python Coroutines in `asyncio`

In a single-threaded application, creating a coroutine object doesn't
necessarily mean it is scheduled to be executed in the event loop.

The functions defined by `async def`, underlying, is a function that returns a
Python generator. If the program calls an `async def` function, it will NOT be
executed. This behavior is one of the main reason why mixing `async def`
function with normal function is a bad idea.

There are three mechanisms to schedule coroutines:

1. Await the coroutine `await asyncio.sleep(1)`;
2. Submit the coroutine to the event loop object `loop.call_soon(coro)`;
3. Turn coroutine into an `asyncio.Task` object `asyncio.ensure_future(coro)`.

## Proposal

This gRFC intended to fully utilize `asyncio` and create a Pythonic paradigm for
gRPC Python programming in Python 3 runtime.

### Two Sets of API In One Package

The new API will be isolated from the current API. The implementation of the new
API is an entirely different stack than the current stack. On the downside, most
gRPC objects and definitions can't be shared between these two sets of API.
After all, it is not a good practice to introduce uncertainty (return coroutine
or regular object) to our API. This decision is made to respect our contract of
API since GA in 2016, including the underlying behaviors. **Developers who use
our current API should not be affected by this effort.**

For users who want to migrate to new API, the granularity migration is per
channel, per server level. For both channel-side and server-side, the adoption
of `asyncio` does not only include cosmetic changes of adding `async`/`await`
keywords throughout the code base, it also requires thinking about the change
from a potentially multi-threaded application to a single-threaded asynchronous
application. For example, the synchronization methods have to be changed to use
the `asyncio` edition (see [Synchronization
Primitives](https://docs.python.org/3/library/asyncio-sync.html)).

### Overview of Async API

All `asyncio` related API will be kept inside the `grpc.aio` module. Take the
most common use case as an example, the new API will diverge from the old API
since the creation of `Channel` and `Server` object. New creation methods are
added to instantiate async `Server` or `Channel` object:

```Python
import grpc
server = grpc.aio.server(...)
channel = grpc.aio.insecure_channel(...)
channel = grpc.aio.secure_channel(...)
channel = grpc.aio.intercept_channel(...)
```

To reduce cognitive burden, the new `asyncio` API should share the same
parameters as the current API, and keep the usage similar, except the following
cases:

* `grpc.aio.server` no longer require the application to provide a
  `ThreadPoolExecutor`;
* Interfaces returning `Future` object are replaced.
* Client RPC invocation merges from `__call__`, `with_call` and `futures` into one.

### Demo Snippet of Async API

Server side:
```Python
class AsyncGreeter(helloworld_pb2_grpc.GreeterServicer):

    async def SayHello(self, request, context):
        await asyncio.sleep(1)
        return helloworld_pb2.HelloReply(message="Hello, %s!" % request.name)

server = grpc.aio.server()
server.add_insecure_port(":50051")
helloworld_pb2_grpc.add_GreeterServicer_to_server(Greeter(), server)
server.start()
await server.wait_for_termination()
```

Client side:
```Python
async with grpc.aio.insecure_channel("localhost:50051") as channel:
    stub = echo_pb2_grpc.EchoStub(channel)
    response = await stub.Hi(echo_pb2.EchoRequest(message="ping"))

    async for response in stub.StreamingHi(...):
        process(response)
```

### Channel-Side

Changes in `grpc.aio.Channel`:
* The object returned by `unary_unary`, `unary_stream`, `stream_unary`, and
  `stream_tream` will be `MultiCallable` objects in `grpc.aio`.

Changes in `grpc.aio.Call`:
* All methods return coroutine object;
* Implements `grpc.aio.Task` API for unary response;
* Implements asynchronous generator API for stream response;

Changes for `MultiCallable` classes in `grpc.aio`:
* Returns a `grpc.aio.Call` object;
* `grpc.aio.StreamUnaryMultiCallable` and `grpc.aio.StreamStreamMultiCallable`
  takes in an asynchronous generator as `request_iterator` argument.

### Server-Side

Changes in `grpc.aio.Server`:
* `add_generic_rpc_handlers` method accepts tuple of
  `grpc.aio.GenericRpcHandler`;
* `wait_for_termination` method is a coroutine;
* `stop` returns a `asyncio.Event`.

Changes in `grpc.aio.HandlerCallDetails`:
* `invocation_metadata` method is a coroutine.

Changes in `grpc.aio.GenericRpcHandler`:
* `service` method takes in `grpc.aio.HandlerCallDetails`;
* `service` method returns a `grpc.aio.RpcMethodHandler`.

Changes in `grpc.aio.RpcMethodHandler`:
* All servicer handlers are coroutines;
* Deserializer / serializer remain normal Python functions.

Changes in `grpc.ServicerContext`:
* `invocation_metadata` method returns a coroutine object;
* `send_initial_metadata` method is a coroutine and returns an `asyncio.Task`
  object.

Changes in `grpc.aio.*_*_rpc_method_handler`:
* Accepts a coroutine;
* Returns a `grpc.aio.RpcMethodHandler`.

Changes in `grpc.aio.method_handlers_generic_handler`:
* `method_handlers` is a dictionary that maps method names to corresponding
  `grpc.aio.RpcMethodHandler`;
* Returns `grpc.aio.GenericRpcHandler`.

### Interceptor

Changes in `grpc.aio.ServerInterceptor`:
* Accepts `grpc.aio.HandlerCallDetails`;
* `continuation` is a function accepts `grpc.aio.HandlerCallDetails` and returns
  `grpc.aio.RpcMethodHandler`.

Changes in `ClientInterceptor` classes in `grpc.aio`:
* `continuation` is a coroutine that returns `Awaitable` object.

### Utility Functions

Changes in `grpc.aio.channel_ready_future`:
* Renamed into `grpc.aio.channel_ready`;
* Accepts a `grpc.aio.Channel`;
* Returns a coroutine object.

```Python
async with grpc.aio.insecure_channel(...) as channel:
    await grpc.aio.channel_ready(channel)
    ...
```

### Shared APIs

APIs in the following categories remain in top level:
* Credentials related classes including for channel, call, and server;
* Channel connectivity Enum class;
* Status code Enum class;
* Compression method Enum class;
* `grpc.RpcError` exception class;
* `grpc.RpcContext` class;
* `grpc.ServiceRpcHandler` class;

### Re-Use ProtoBuf Generated Code For Services

Here is an example of the core part of the gRPC ProtoBuf plugin generated code:

```Python
# Client-side stub
class GreeterStub(object):
  def __init__(self, channel):
    self.SayHello = channel.unary_unary(...)

# Server-side servicer
def add_GreeterServicer_to_server(servicer, server):
  ...
```

Both `channel` and `server` object are passed in by the application. The
generated code is agnostic to which set of API the application uses, as long as
we respect the method name in our API contract.

### API Reference Doc Generation

The async API related classes and functions, including shared APIs, should be
generated on the same page.

## Controversial Details

### Unified Stub Call

Currently, for RPC invocations, gRPC Python provides 3 options: `__call__`,
`with_call`, and `futures`. They behaves slightly different in whether block or
not, and the number of return values (see
[UnaryUnaryMultiCallable](https://grpc.github.io/grpc/python/grpc.html#grpc.UnaryUnaryMultiCallable)
for details). Developers have to bare in mind the subtlety between these 3
options across 4 types of RPC (not all combination are supported).

Semantically, those 3 options grant users ability to:
1. Directly getting the RPC response.
2. Check metadata of the RPC.
3. Control the life cycle of the RPC.
4. Handle the RPC failure.

It is understandable for current design to provide different options, since
Python doesn't have a consensus `Future` before. The simplicity of an RPC call
will be ruined if the original designer of gRPC Python API tries to merge those
3 options.

However, thanks to the official definition of `asyncio.Task`/`asyncio.Future`.
They can be merged into one method by returning a `grpc.aio.Call` that extends
(or composites) `asyncio.Task`.

In `asyncio`, it is expected to `await` on asynchronous operations. Hence, it is
nature for stub calls to return an `asyncio.Task` compatible object, which
representing the RPC call.

Also, the `grpc.aio.Call` object provides gRPC specific semantics to manipulate
the ongoing RPC. So, all above functionality can be solved by one single merged
method.

```Python
# Usage 1: A simple call
response = await stub.Hi(...)

# Usage 2: Check metadata
call = stub.Hi(...)
if validate(await call.initial_metadata()):
    response = await call
    print(f'Getting response [{response}] with code [{call.code()}]')
else:
    raise ValueError('Failed to validate initial metadata')

# Usage 3: Control the life cycle
call = stub.Hi(...)
await async_stuff_that_takes_time()
if call.is_active() and call.time_remaining() < REMAINING_TIME_THRESHOLD:
    call.cancel()

# Usage 4: Error handling
try:
    response = await stub.FailedHi(...)
except grpc.RpcError as rpc_error:
    print(f'RPC failed: {rpc_error.code()}')
```

### Support Thread And Process Executors

The new API intended to solve all concurrency issue with asynchronous I/O.
However, it also introduces a migration challenge for our users. For users who
want to gradually migrate from current stack to async stack, their potentially
non-asyncio native logic may block the entire thread. If the thread is blocked,
then the event loop will be blocked, then the whole process will end up
deadlocking.

However, by supporting the executors, we can **allow mixing async and sync**
**method handlers** on the server side, which further reduce the cost of
migration.

### Generator Or Asynchronous Generator?

For streaming calls, the requests on the client-side are supplied by generator
for now. If a user wants to provide a pre-defined list of request messages, they
can use build-in `iter()` function. But there isn't an equivalent function for
async generator. Should we wrap it inside our library to increase usability?

```Python
### Current usage of Python generator
stub.SayHelloStreaming(iter([
    HelloRequest(name='Golden'),
    HelloRequest(name='Retriever'),
    HelloRequest(name='Pan'),
    HelloRequest(name='Cake'),
]))

### The new usage is much verbose for same scenario
class AsyncIter:
    def __init__(self, items):    
        self.items = items    

    async def __aiter__(self):    
        for item in self.items:    
            yield item

stub.SayHelloStreaming(AsyncIter([
    HelloRequest(name='Golden'),
    HelloRequest(name='Retriever'),
    HelloRequest(name='Pan'),
    HelloRequest(name='Cake'),
]))
```

### Special Async Functions Name

Fire-and-forget is a valid use case in async programming. For the non-critical
tasks, the program may schedule the execution without checking the result. As
mentioned in "Python Coroutines in `asyncio`" section, the coroutine won't be
scheduled unless we explicitly do so. A dedicated prefix or suffix of the
function should help to remind developers to await the coroutine.

```Python
### Forget to await RPC
while 1:
    stub.ReportLoad(ReportRequest(
        timestamp=...,
        metrics=[...],
    ))  # <- RPC not sent
    await asyncio.sleep(3)

### Await on whatever function starts with "Async"
while 1:
    await stub.AsyncReportLoad(ReportRequest(
        timestamp=...,
        metrics=[...],
    ))
    await asyncio.sleep(3)
```

CPython developers also consider this problem. Their solution is that if a
coroutine object gets deallocated without execution, the interpreter will log an
`RuntimeWarning` to the standard error output.

```
RuntimeWarning: coroutine '...' was never awaited
```

## Story For Testing

For new `asyncio` related behavior, we will write unit tests dedicated to the
new stack. However, currently, there are more than a hundred test cases in gRPC
Python. It's a significant amount of work to refactor each one of them for the
new API.

A straight forward solution is building a wrapper over async API to simulate
current API. The wrapper itself shouldn't be too hard to implement. However,
some tests are tightly coupled with the implementation detail that might break;
manual labor may still be required for this approach.

We should compare the cost of manual refactoring and wrapper when we want to
reuse the existing test cases.

## Other Official Packages

Besides `grpcio` package, currently gRPC Python also own:

* `grpcio-tools` (no service)
* `grpcio-testing`
* `grpcio-reflection`
* `grpcio-health-checking`
* `grpcio-channelz`
* `grpcio-status` (no service)

Apart from `grpcio-tools` and `grpcio-status`, all other packages have at least
one gRPC service implementation. They will also require migration to adopt
`asyncio`. This design doc propose to keep their current API untouched, and add
new sets of async APIs.

## Flow Control Enforcement

To propagate HTTP/2 flow control push back, the new async API needs to aware of
the flow control mechanism. Most complex logic is handled in C-Core, except
there is a single rule the wrapper layer needs to follow: there can only be one
outstanding read/write operation on each call.

It means if the application fires two write in parallel, one of them have to
wait until the other one finishes. Also, that rule doesn't prohibit reading and
writing at the same time.

So, even if all the read/write is asynchronous in `asyncio`, we will have to
either enforce the rule ourselves by adding locks in our implementation. Or we
can pass down the synchronization responsibility to our users.

## Concrete Class Instead of Interfaces

Interface is a design pattern that defines the contract of an entity that allows
different implementation to work seamlessly in a system. In the past, gRPC
Python has been using metaclass based Python interface pattern. It works just
like interface in Golang and Java, except the error is generated in runtime
instead of compile time.

If the gRPC Python has multiple implementation for a single interface, the use
of the design pattern provides productivity in unifying their behavior. However,
almost non interfaces has second implementation, even if they do, they are
depends directly on our concrete implementation, which should better using
inheritance or composition than interfaces.

Also, in the past, dependents of gRPC Python have observed several failure
caused by the interface. The interface constraints our ability to add
experimental API. Once we change even slightly with interfaces, the downstream
implementations are likely to break.

Since this is a new opportunity for us to re-design, we need to think cautiously
about how do we empower our users to extend our classes. For majority of cases,
we are providing the only implementation for the interface. We should
convert them into concrete classes.

On the other hand, there are actually one valid use case that we should keep
abstract class -- interceptors. To be more specific, the following interfaces
won't be replaced by concrete classes:

* grpc.ServerInterceptor
* grpc.UnaryUnaryClientInterceptor
* grpc.UnaryStreamClientInterceptor
* grpc.StreamUnaryClientInterceptor
* grpc.StreamStreamClientInterceptor

## Rationale

### Compatibility With Other Asynchronous Libraries

By other asynchronous libraries, I meant libraries that provide their own
Future, Coroutine, Event-loops, like `gevent`/`Twisted`/`Tornado`. In general,
it is challenging to support multiple async mechanisms inter-operate together,
and it has to be discussed case by case.

For `Tornado`, they are using `asyncio` underneath their abstraction of
asynchronous mechanisms. After v6.0.0, they dropped support for Python 2
entirely. So, it should work fine with our new API.

For `Twisted`, with some monkey patch code to connect its `Deferred`/`Future`
object to `asyncio`'s API, it should work.

For `gevent`, unfortunately, it works by monkey-patching Python APIs that
including `threading` and various transport library, and the work is executed in
`gevent` managed event loop. `gevent` and `asyncio` doesn't compatible with each
other out of box.

### Wrapping Async Stack To Provide Current API Publicly

1) It would be quite confusing to mix functions that return `asyncio` coroutine
   and normal Python functions;
2) We don't want to imply that switching to use the new stack requires zero code
   changes.
3) Also, the current contract of API constraint ourselves from changing their
   behavior after GA for two years.

## Pending Topics

Reviewers has thrown many valuable proposals. This design doc may not be the
ideal place for those discussions.
* Re-design streaming API on the server side to replace the iterator pattern.
* Re-design the connectivity API to be consistent with C-Core.
* Design a easy way to use channel arguments [grpc#19734](https://github.com/grpc/grpc/issues/19734).

## Related Issues

* https://github.com/grpc/grpc/issues/6046
* https://github.com/grpc/grpc/issues/18376
* https://github.com/grpc/grpc/projects/16

## Implementation

* TODO
