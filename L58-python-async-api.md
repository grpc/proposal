Async API for gRPC Python
----
* Author(s): lidizheng
* Approver: gnossen
* Status: In Review
* Implemented in: Python
* Last updated: 2019-10-01
* Discussions at:
  * https://groups.google.com/forum/#!topic/grpc-io/7V7HYM_aph4
  * https://github.com/lidizheng/grpc-api-examples/pull/1
  * https://github.com/grpc/proposal/pull/155

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

`asyncio` is great, but not silver bullet to solve everything. It has its
limitations and unique advantages. gRPC Python as a framework should empower
tech-savvy users to use the cutting-edge features, in the same time as we allow
majority of our users to code in the way they familier.

The new API will be isolated from the current API. The implementation of the new
API is an entirely different stack than the current stack. On the downside, most
gRPC objects and definitions can't be shared between these two sets of API.
After all, it is not a good practice to introduce uncertainty (return coroutine
or regular object) to our API. This decision is made to respect our contract of
API since GA in 2016, including the underlying behaviors. **Developers who use
our current API should not be affected by this effort.** We plan to support two
stacks in the long-run.

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

To reduce cognitive burden, this gRFC tries to make the new async API share the same
parameters as the current API, and keep the usage similar.

### Demo Snippet of Async API

Server side:
```Python
class AsyncGreeter(helloworld_pb2_grpc.GreeterServicer):

    async def SayHello(self,
                       request: helloworld_pb2_grpc.HelloRequest,
                       context: grpc.aio.ServicerContext
                ) -> helloworld_pb2_grpc.HelloReply:
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

### New Streaming API

Existing streaming API has usability problem that its logic complexes user
application. For client side, it requires the request iterator to be defined
above the line of call, and the consumption of responses to be coded below the
line of invocation. So when the application wants to send request based on
received response, it would be quite challenging and usually involves
multi-threading, which is not optimal in Python world.

For server side, the same problem remains for non-trivial streaming servicer
handlers. The `yield` syntax constraint the message sending should happen in the
handler itself, otherwise the user needs to code a dedicated iterator class. For
larger application, it would be a pain to create such iterator class for each
streaming handler.

euroelessar@ points out several great reasons for the new streaming API:
* Enable `with` statement for streaming API, which allows clean resource management;
* Easier to buffer outbound messages when peer pushes back (e.g. `yield` will
  stop the execution of current coroutine, but `call.write` can be scheduled in
  event loop);
* Enable application to have both pending read and pending write in one
  coroutine, and react on the first finisher (with iterator syntax this will be
  quite hard to get it right).

So, this gRFC introduces a new pattern of streaming API that reads/writes
message to peer with explicit call.

#### Snippet For New Streaming API

```Python
### Client side
with stub.StreamingHi() as streaming_call:
  request = echo_pb2.EchoRequest(message="ping")
  await streaming_call.write(request)
  response = await streaming_call.read()
  while response:  # or response is not grpc.aio.EOF
      process(response)
      response = await streaming_call.read()

### Server side
class AsyncGreeter(helloworld_pb2_grpc.GreeterServicer):

    async def StreamingHi(self,
                          unused_request_iterator,
                          context: grpc.aio.ServicerContext
            ) -> None:
        bootstrap_request = await context.read()
        initialize_environment(bootstrap_request)
        while has_response():
            response = ...
            await context.write(response)
```

#### Snippet For Current Streaming API

```Python
### Client side
class RequestIterator:

    def __init__(self):
        self._queue = asyncio.Queue()

    async def send(self, message: HelloRequest):
        await self._queue.put(message)

    def __aiter__(self) -> AsyncIterable[HelloRequest]:
        return self

    async def __anext__(self) -> HelloRequest:
        return await self._queue.get(block=True)

request_iterator = RequestIterator()
# No await needed, the response_iterator is grpc.aio.Call
response_iterator = stub.StreamingHi(request_iterator)

# In sending coroutine
await request_iterator.send(proto_message)

# In receiving coroutine
async for response in response_iterator:
    process(response)

### Server side
class ResponseIterator:

    def __init__(self):
        self._queue = asyncio.Queue()

    async def send(self, message: HelloReply):
        await self._queue.put(message)

    def __aiter__(self) -> AsyncIterable[HelloReply]:
        return self

    async def __anext__(self) -> HelloReply:
        return await self._queue.get()

async def streaming_hi_worker(
        request_iterator: AsyncIterable[HelloRequest],
        response_iterator: ResponseIterator
    ) -> None:
    async for request in request_iterator:
        if request.needs_respond:
            await response_iterator.send(response)

class Greeter(helloworld_pb2_grpc.GreeterServicer):
    async def StreamingHi(self,
                          request_iterator: AsyncIterable[HelloRequest],
                          context: grpc.aio.ServicerContext
            ) -> AsyncIterable[HelloReply]:
        response_iterator = ResponseIterator()
        # Handle the write in another coroutine
        asyncio.get_event_loop.create_task(streaming_hi_worker(request_iterator, response_iterator))
        return response_iterator
```

#### Co-existence Of New And Current Streaming API

Existing API still has advantage over simpler use cases of gRPC, and iterator
syntax is Pythonic to use. Also, for backward compatibility concern,
iterator-based API needs to stay. So, both new and current streaming API needs
to be included in the surface API.

To keep the function signature stable, the new streaming API only requires
enhancement to the context object (`grpc.aio.Call` on client-side, and
`grpc.aio.ServicerContext` on the server-side).

Notably, both new and current streaming API allows read and write messages
simultaneously in different thread or coroutine. However, **the application should**
**not mixing new and current streaming API at the same time**. Reading or writing to
the iterator and context object might have synchronization issue that the
framework does not guaranteed to protect against.

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

### Using Asynchronous Generator

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

### No Special Async Functions Naming Pattern

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

### Story For Testing

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

### Other Official Packages

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

### Flow Control Enforcement

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

### Concrete Class Instead of "Interfaces"

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

* `grpc.ServerInterceptor`
* `grpc.UnaryUnaryClientInterceptor`
* `grpc.UnaryStreamClientInterceptor`
* `grpc.StreamUnaryClientInterceptor`
* `grpc.StreamStreamClientInterceptor`

## Future Features

### Implicit Context Propagation By `contextvars`

`contextvars` is a Python 3.7 feature that allows applications to set coroutine
local variables (just like thread local variables). In the past, gRPC Python
uses thread local to achieve two goals:

* Support distributed tracing library that work across languages;
* Support deadline propagation along the chain of RPCs.

For distributed tracing library (like OpenCensus), the tracing information is
preserved as one of the metadata, and it is expected to be implicitly promoted
from inbound metadata to outbound metadata. The tracing information will be used
to monitor the life cycle of an request, and the length it has stayed in each
services.

For deadline propagation, the deadline of upstream server will be implicitly
pass down to downstream server, so downstream services can react to that
information and save computation resources or perform flow control.

Acceptence critiria of this feature:
* The implementation of such feature should supported by official package, and
users are not expected to directly access those metadata;
* Application logic has higher priority than the implicit propagation (e.g.
  setting timeout explicitly will override the propagated value);
* Application has the choice to diverse coroutine local variables;
* The exception error string should be informative (e.g. pointing out the
  timeout is due to upstream deadline).

Further discussion around this topic, please see related sections under [Rationale].

### Introduce Typing To Generated Code

Typing makes a difference. This gRFC intended to introduce typing to gRPC Python
as much as we can. To change generated code, we need to update the gRPC protoc
plugin to generate Python 3 stubs and servicers. To date, **this gRFC intended to**
**make the new async API compatible with existing generated code**. So, the Python 3
generated code will have two options:

1. Add a third generation mode to protoc plugin that generates stubs / servicers
   with typing;
2. Generating an additional file for the typed code.

## Rationale

### Compatibility With Other Asynchronous Libraries

By other asynchronous libraries, they mean libraries that provide their own
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

### Explicit Context vs. Implicit Context

TBD

## API Interfaces

### Channel-Side

```Python
# grpc.aio.Channel
class Channel:
    """Affords RPC invocation via generic methods on client-side.

    Channel objects implement the Async Context Manager type, although they need
    not support being entered and exited multiple times.
    """

    def subscribe(self,
                  callback: Callable[[ChannelConnectivity], None],
                  try_to_connect: bool=False) -> None:
        """Subscribe to this Channel's connectivity state machine.

        A Channel may be in any of the states described by ChannelConnectivity.
        This method allows application to monitor the state transitions.
        The typical use case is to debug or gain better visibility into gRPC
        runtime's state.

        Args:
          callback: A callable to be invoked with ChannelConnectivity argument.
            ChannelConnectivity describes current state of the channel.
            The callable will be invoked immediately upon subscription
            and again for every change to ChannelConnectivity until it
            is unsubscribed or this Channel object goes out of scope.
          try_to_connect: A boolean indicating whether or not this Channel
            should attempt to connect immediately. If set to False, gRPC
            runtime decides when to connect.
        """

    def unsubscribe(self,
                    callback: Callable[[ChannelConnectivity], None]) -> None:
        """Unsubscribes a subscribed callback from this Channel's connectivity.

        Args:
          callback: A callable previously registered with this Channel from
          having been passed to its "subscribe" method.
        """

    def unary_unary(self,
                    method: Text,
                    request_serializer: Optional[Callable[[Any], bytes]]=None,
                    response_deserializer: Optional[Callable[[bytes], Any]]=None
        ) -> grpc.aio.UnaryUnaryMultiCallable:
        """Creates a UnaryUnaryMultiCallable for a unary call method.

        Args:
          method: The name of the RPC method.
          request_serializer: Optional behaviour for serializing the request
            message. Request goes unserialized in case None is passed.
          response_deserializer: Optional behaviour for deserializing the
            response message. Response goes undeserialized in case None
            is passed.

        Returns:
          A UnaryUnaryMultiCallable value for the named unary call method.
        """

    def unary_stream(self,
                     method: Text,
                     request_serializer: Optional[Callable[[Any], bytes]]=None,
                     response_deserializer: Optional[Callable[[bytes], Any]]=None
        ) -> grpc.aio.UnaryStreamMultiCallable:
        """Creates a UnaryStreamMultiCallable for a server streaming method.

        Args:
          method: The name of the RPC method.
          request_serializer: Optional behaviour for serializing the request
            message. Request goes unserialized in case None is passed.
          response_deserializer: Optional behaviour for deserializing the
            response message. Response goes undeserialized in case None is
            passed.

        Returns:
          A UnaryStreamMultiCallable value for the name server streaming method.
        """

    def stream_unary(self,
                     method: Text,
                     request_serializer: Optional[Callable[[Any], bytes]]=None,
                     response_deserializer: Optional[Callable[[bytes], Any]]=None
        ) -> grpc.aio.StreamUnaryMultiCallable:
        """Creates a StreamUnaryMultiCallable for a client streaming method.

        Args:
          method: The name of the RPC method.
          request_serializer: Optional behaviour for serializing the request
            message. Request goes unserialized in case None is passed.
          response_deserializer: Optional behaviour for deserializing the
            response message. Response goes undeserialized in case None is
            passed.

        Returns:
          A StreamUnaryMultiCallable value for the named client streaming method.
        """

    def stream_stream(self,
                      method: Text,
                      request_serializer: Optional[Callable[[Any], bytes]]=None,
                      response_deserializer: Optional[Callable[[bytes], Any]]=None
        ) -> grpc.aio.StreamStreamMultiCallable:
        """Creates a StreamStreamMultiCallable for a bi-directional streaming method.

        Args:
          method: The name of the RPC method.
          request_serializer: Optional behaviour for serializing the request
            message. Request goes unserialized in case None is passed.
          response_deserializer: Optional behaviour for deserializing the
            response message. Response goes undeserialized in case None
            is passed.

        Returns:
          A StreamStreamMultiCallable value for the named bi-directional streaming method.
        """

    def close(self) -> None:
        """Closes this Channel and releases all resources held by it.

        Closing the Channel will immediately terminate all RPCs active with the
        Channel and it is not valid to invoke new RPCs with the Channel.

        This method is idempotent.
        """
```

```Python
# grpc.aio.UnaryUnaryMultiCallable
class UnaryUnaryMultiCallable:
    """Affords invoking an async unary RPC from client-side."""

    def __call__(self,
                 request: Any,
                 timeout: Optional[int]=None,
                 metadata: Optional[Sequence[Tuple[Text, Text]]]=None,
                 credentials: Optional[grpc.CallCredentials]=None,
                 wait_for_ready: Optional[bool]=None,
                 compression: Optional[grpc.Compression]=None
    ) -> grpc.aio.Call[Any]:
        """Schedules the underlying RPC.

        Args:
          request: The request value for the RPC.
          timeout: An optional duration of time in seconds to allow
            for the RPC.
          metadata: Optional :term:`metadata` to be transmitted to the
            service-side of the RPC.
          credentials: An optional CallCredentials for the RPC. Only valid for
            secure Channel.
          wait_for_ready: This is an EXPERIMENTAL argument. An optional
            flag to enable wait for ready mechanism
          compression: An element of grpc.compression, e.g.
            grpc.compression.Gzip. This is an EXPERIMENTAL option.

        Returns:
          An awaitable object grpc.aio.Call that returns the response value.

        Raises:
          RpcError: Indicating that the RPC terminated with non-OK status. The
            raised RpcError will also be a Call for the RPC affording the RPC's
            metadata, status code, and details.
        """

# grpc.aio.UnaryStreamMultiCallable
class UnaryStreamMultiCallable:
    """Affords invoking an async server streaming RPC from client-side."""

    def __call__(self,
                 request: Any,
                 timeout: Optional[int]=None,
                 metadata: Optional[Sequence[Tuple[Text, Text]]]=None,
                 credentials: Optional[grpc.CallCredentials]=None,
                 wait_for_ready: Optional[bool]=None,
                 compression: Optional[grpc.Compression]=None
    ) -> grpc.aio.Call[AsyncIterable[Any]]:
        """Schedules the underlying RPC.

        Args:
          request: The request value for the RPC.
          timeout: An optional duration of time in seconds to allow
            for the RPC.
          metadata: Optional :term:`metadata` to be transmitted to the
            service-side of the RPC.
          credentials: An optional CallCredentials for the RPC. Only valid for
            secure Channel.
          wait_for_ready: This is an EXPERIMENTAL argument. An optional
            flag to enable wait for ready mechanism
          compression: An element of grpc.compression, e.g.
            grpc.compression.Gzip. This is an EXPERIMENTAL option.

        Returns:
          An awaitable object grpc.aio.Call that returns the async iterator of response values.

        Raises:
          RpcError: Indicating that the RPC terminated with non-OK status. The
            raised RpcError will also be a Call for the RPC affording the RPC's
            metadata, status code, and details.
        """

# grpc.aio.StreamUnaryMultiCallable
class StreamUnaryMultiCallable:
    """Affords invoking an async client streaming RPC from client-side."""

    def __call__(self,
                 request_iterator: Optional[AsyncIterable[Any]]=None,
                 timeout: Optional[int]=None,
                 metadata: Optional[Sequence[Tuple[Text, Text]]]=None,
                 credentials: Optional[grpc.CallCredentials]=None,
                 wait_for_ready: Optional[bool]=None,
                 compression: Optional[grpc.Compression]=None
    ) -> grpc.aio.Call[Any]:
        """Schedules the underlying RPC.

        Args:
          request_iterator: An async iterator that yields request values for
            the RPC.
          timeout: An optional duration of time in seconds to allow
            for the RPC.
          metadata: Optional :term:`metadata` to be transmitted to the
            service-side of the RPC.
          credentials: An optional CallCredentials for the RPC. Only valid for
            secure Channel.
          wait_for_ready: This is an EXPERIMENTAL argument. An optional
            flag to enable wait for ready mechanism
          compression: An element of grpc.compression, e.g.
            grpc.compression.Gzip. This is an EXPERIMENTAL option.

        Returns:
          An awaitable object grpc.aio.Call presents the RPC.

        Raises:
          RpcError: Indicating that the RPC terminated with non-OK status. The
            raised RpcError will also be a Call for the RPC affording the RPC's
            metadata, status code, and details.
        """

# grpc.aio.StreamStreamMultiCallable
class StreamStreamMultiCallable:
    """Affords invoking an async bi-directional RPC from client-side."""

    def __call__(self,
                 request_iterator: Optional[AsyncIterable[Any]]=None,
                 timeout: Optional[int]=None,
                 metadata: Optional[Sequence[Tuple[Text, Text]]]=None,
                 credentials: Optional[grpc.CallCredentials]=None,
                 wait_for_ready: Optional[bool]=None,
                 compression: Optional[grpc.Compression]=None
    ) -> grpc.aio.Call[AsyncIterable[Any]]:
        """Schedules the underlying RPC.

        Args:
          request_iterator: An async iterator that yields request values for
            the RPC.
          timeout: An optional duration of time in seconds to allow
            for the RPC.
          metadata: Optional :term:`metadata` to be transmitted to the
            service-side of the RPC.
          credentials: An optional CallCredentials for the RPC. Only valid for
            secure Channel.
          wait_for_ready: This is an EXPERIMENTAL argument. An optional
            flag to enable wait for ready mechanism
          compression: An element of grpc.compression, e.g.
            grpc.compression.Gzip. This is an EXPERIMENTAL option.

        Returns:
          An awaitable object grpc.aio.Call that returns the async iterator of response values.

        Raises:
          RpcError: Indicating that the RPC terminated with non-OK status. The
            raised RpcError will also be a Call for the RPC affording the RPC's
            metadata, status code, and details.
        """
```

```Python
# grpc.aio.Call
class Call(typing.Awaitable[T], grpc.RpcContext):
    """The representation of an RPC on the client-side."""

    def is_active(self) -> bool:
        """Describes whether the RPC is active or has terminated.

        Returns:
          True if RPC is active, False otherwise.
        """

    def time_remaining(self) -> float:
        """Describes the length of allowed time remaining for the RPC.

        Returns:
          A nonnegative float indicating the length of allowed time in seconds
          remaining for the RPC to complete before it is considered to have
          timed out, or None if no deadline was specified for the RPC.
        """

    def cancel(self) -> None:
        """Cancels the RPC.

        Idempotent and has no effect if the RPC has already terminated.
        """

    def add_callback(self, callback: Callable[None, None]) -> None:
        """Registers a callback to be called on RPC termination.

        Args:
          callback: A no-parameter callable to be called on RPC termination.

        Returns:
          True if the callback was added and will be called later; False if
            the callback was not added and will not be called (because the RPC
            already terminated or some other reason).
        """

    async def initial_metadata(self) -> Sequence[Tuple[Text, Text]]:
        """Accesses the initial metadata sent by the server.

        Coroutine continues once the value is available.

        Returns:
          The initial :term:`metadata`.
        """

    async def trailing_metadata(self) -> Sequence[Tuple[Text, Text]]:
        """Accesses the trailing metadata sent by the server.

        Coroutine continues once the value is available.

        Returns:
          The trailing :term:`metadata`.
        """

    async def code(self) -> grpc.StatusCode:
        """Accesses the status code sent by the server.

        Coroutine continues once the value is available.

        Returns:
          The StatusCode value for the RPC.
        """

    async def details(self) -> Text:
        """Accesses the details sent by the server.

        Coroutine continues once the value is available.

        Returns:
          The details string of the RPC.
        """

    def __aiter__(self) -> AsyncIterable[Any]:
        """Returns the async iterable representation that yields messages.

        Returns:
          An async iterable object that yields messages.
        """

    async def read(self) -> Any:
        """Reads one message from the RPC.

        Only one read operation is allowed simultaneously. Mixing new streaming API and old
        streaming API will resulted in undefined behavior.

        Returns:
          A response message of the RPC.
        
        Raises:
          An RpcError exception if the read failed.
        """

    async def write(self, message: Any) -> None:
        """Writes one message to the RPC.

        Only one write operation is allowed simultaneously. Mixing new streaming API and old
        streaming API will resulted in undefined behavior.

        Raises:
          An RpcError exception if the write failed.
        """
```

### Server-Side

```Python
# grpc.aio.Server
class Server:
    """Serves RPCs."""

    def add_generic_rpc_handlers(
            self,
            generic_rpc_handlers: Iterable[grpc.aio.GenericRpcHandlers]
        ) -> None:
        """Registers GenericRpcHandlers with this Server.

        This method is only safe to call before the server is started.

        Args:
          generic_rpc_handlers: An iterable of GenericRpcHandlers that will be
          used to service RPCs.
        """

    def add_insecure_port(self, address: Text) -> int:
        """Opens an insecure port for accepting RPCs.

        This method may only be called before starting the server.

        Args:
          address: The address for which to open a port. If the port is 0,
            or not specified in the address, then gRPC runtime will choose a port.

        Returns:
          An integer port on which server will accept RPC requests.
        """

    def add_secure_port(self,
                        address: Text,
                        server_credentials: grpc.ServerCredentials) -> int:
        """Opens a secure port for accepting RPCs.

        This method may only be called before starting the server.

        Args:
          address: The address for which to open a port.
            if the port is 0, or not specified in the address, then gRPC
            runtime will choose a port.
          server_credentials: A ServerCredentials object.

        Returns:
          An integer port on which server will accept RPC requests.
        """

    def start(self) -> None:
        """Starts this Server.

        This method may only be called once. (i.e. it is not idempotent).
        """

    def stop(self, grace: Optional[float]) -> asyncio.Event:
        """Stops this Server.

        This method immediately stop service of new RPCs in all cases.

        If a grace period is specified, this method returns immediately
        and all RPCs active at the end of the grace period are aborted.
        If a grace period is not specified (by passing None for `grace`),
        all existing RPCs are aborted immediately and this method
        blocks until the last RPC handler terminates.

        This method is idempotent and may be called at any time.
        Passing a smaller grace value in a subsequent call will have
        the effect of stopping the Server sooner (passing None will
        have the effect of stopping the server immediately). Passing
        a larger grace value in a subsequent call *will not* have the
        effect of stopping the server later (i.e. the most restrictive
        grace value is used).

        Args:
          grace: A duration of time in seconds or None.

        Returns:
          A threading.Event that will be set when this Server has completely
          stopped, i.e. when running RPCs either complete or are aborted and
          all handlers have terminated.
        """

    async def wait_for_termination(self, timeout: Optional[float]=None) -> bool:
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

        Returns:
          A bool indicates if the operation times out.
        """
```

```Python
# grpc.aio.GenericRpcHandler
class GenericRpcHandler:
    """An implementation of arbitrarily many RPC methods."""

    def service(self,
                handler_call_details: grpc.HandlerCallDetails
        ) -> Union[grpc.RpcMethodHandler, grpc.aio.RpcMethodHandler, None]:
        """Returns the handler for servicing the RPC.

        Args:
          handler_call_details: A HandlerCallDetails describing the RPC.

        Returns:
          An grpc.RpcMethodHandler if the handler is a normal Python function;
          or an grpc.aio.RpcMethodHandler if the handler is a coroutine;
          otherwise, None.
        """
```

```Python
# grpc.aio.RpcMethodHandler
class RpcMethodHandler:
    """An implementation of a single RPC method.

    Attributes:
      request_streaming: Whether the RPC supports exactly one request message
        or any arbitrary number of request messages.
      response_streaming: Whether the RPC supports exactly one response message
        or any arbitrary number of response messages.
      request_deserializer: A callable behavior that accepts a byte string and
        returns an object suitable to be passed to this object's business
        logic, or None to indicate that this object's business logic should be
        passed the raw request bytes.
      response_serializer: A callable behavior that accepts an object produced
        by this object's business logic and returns a byte string, or None to
        indicate that the byte strings produced by this object's business logic
        should be transmitted on the wire as they are.
      unary_unary: This object's application-specific business logic as a
        callable value that takes a request value and a ServicerContext object
        and returns a response value. Only non-None if both request_streaming
        and response_streaming are False.
      unary_stream: This object's application-specific business logic as a
        callable value that takes a request value and a ServicerContext object
        and returns an iterator of response values. Only non-None if
        request_streaming is False and response_streaming is True.
      stream_unary: This object's application-specific business logic as a
        callable value that takes an iterator of request values and a
        ServicerContext object and returns a response value. Only non-None if
        request_streaming is True and response_streaming is False.
      stream_stream: This object's application-specific business logic as a
        callable value that takes an iterator of request values and a
        ServicerContext object and returns an iterator of response values.
        Only non-None if request_streaming and response_streaming are both
        True.
    """
    request_streaming: bool
    response_streaming: bool
    request_deserializer: Optional[Callable[[bytes], Any]]
    response_serializer: Optional[Callable[[Any], bytes]]
    unary_unary: Optional[Callable[[Any, grpc.aio.ServicerContext], Any]]
    unary_stream: Optional[Callable[[Any, grpc.aio.ServicerContext], Optional[AsyncIterable[Any]]]]
    stream_unary: Optional[Callable[[AsyncIterable[Any], grpc.aio.ServicerContext], Any]]
    stream_stream: Optional[Callable[[AsyncIterable[Any], grpc.aio.ServicerContext], Optional[AsyncIterable[Any]]]]
```

```Python
# grpc.aio.ServicerContext
class ServicerContext(grpc.RpcContext):
    """A context object passed to method implementations."""

    def invocation_metadata(self) -> Optional[Sequence[Tuple[Text, Text]]]:
        """Accesses the metadata from the sent by the client.

        Returns:
          The invocation :term:`metadata`.
        """

    def peer(self) -> Text:
        """Identifies the peer that invoked the RPC being serviced.

        Returns:
          A string identifying the peer that invoked the RPC being serviced.
          The string format is determined by gRPC runtime.
        """

    def peer_identities(self) -> Iterable[Text]:
        """Gets one or more peer identity(s).

        Equivalent to
        servicer_context.auth_context().get(servicer_context.peer_identity_key())

        Returns:
          An iterable of the identities, or None if the call is not
          authenticated. Each identity is returned as a raw bytes type.
        """

    def peer_identity_key(self) -> Text:
        """The auth property used to identify the peer.

        For example, "x509_common_name" or "x509_subject_alternative_name" are
        used to identify an SSL peer.

        Returns:
          The auth property (string) that indicates the
          peer identity, or None if the call is not authenticated.
        """

    def auth_context(self) -> Mapping[Text, Iterable[bytes]]:
        """Gets the auth context for the call.

        Returns:
          A map of strings to an iterable of bytes for each auth property.
        """

    def set_compression(self, compression: grpc.Compression) -> None:
        """Set the compression algorithm to be used for the entire call.

        This is an EXPERIMENTAL method.

        Args:
          compression: An element of grpc.compression, e.g.
            grpc.compression.Gzip.
        """

    async def send_initial_metadata(self, initial_metadata: Sequence[Tuple[Text, Text]]) -> None:
        """Sends the initial metadata value to the client.

        This method need not be called by implementations if they have no
        metadata to add to what the gRPC runtime will transmit.

        Args:
          initial_metadata: The initial :term:`metadata`.
        """

    async def set_trailing_metadata(self, trailing_metadata: Sequence[Tuple[Text, Text]]) -> None:
        """Sends the trailing metadata for the RPC.

        This method need not be called by implementations if they have no
        metadata to add to what the gRPC runtime will transmit.

        Args:
          trailing_metadata: The trailing :term:`metadata`.
        """

    def abort(self, code: grpc.StatusCode, details: Text) -> NoReturn:
        """Raises an exception to terminate the RPC with a non-OK status.

        The code and details passed as arguments will supercede any existing
        ones.

        Args:
          code: A StatusCode object to be sent to the client.
            It must not be StatusCode.OK.
          details: A UTF-8-encodable string to be sent to the client upon
            termination of the RPC.

        Raises:
          Exception: An exception is always raised to signal the abortion the
            RPC to the gRPC runtime.
        """

    def abort_with_status(self, status: grpc.Status) -> NoReturn:
        """Raises an exception to terminate the RPC with a non-OK status.

        The status passed as argument will supercede any existing status code,
        status message and trailing metadata.

        This is an EXPERIMENTAL API.

        Args:
          status: A grpc.Status object. The status code in it must not be
            StatusCode.OK.

        Raises:
          Exception: An exception is always raised to signal the abortion the
            RPC to the gRPC runtime.
        """

    def set_code(self, code: grpc.StatusCode) -> None:
        """Sets the value to be used as status code upon RPC completion.

        This method need not be called by method implementations if they wish
        the gRPC runtime to determine the status code of the RPC.

        Args:
          code: A StatusCode object to be sent to the client.
        """

    def set_details(self, details: Text) -> None:
        """Sets the value to be used as detail string upon RPC completion.

        This method need not be called by method implementations if they have
        no details to transmit.

        Args:
          details: A UTF-8-encodable string to be sent to the client upon
            termination of the RPC.
        """

    def disable_next_message_compression(self) -> None:
        """Disables compression for the next response message.

        This is an EXPERIMENTAL method.

        This method will override any compression configuration set during
        server creation or set on the call.
        """

    async def read(self) -> Any:
        """Reads one message from the RPC.

        Only one read operation is allowed simultaneously. Mixing new streaming API and old
        streaming API will resulted in undefined behavior.

        Returns:
          A response message of the RPC.
        
        Raises:
          An RpcError exception if the read failed.
        """

    async def write(self, message: Any) -> None:
        """Writes one message to the RPC.

        Only one write operation is allowed simultaneously. Mixing new streaming API and old
        streaming API will resulted in undefined behavior.

        Raises:
          An RpcError exception if the write failed.
        """
```

```Python
# grpc.aio.unary_unary_rpc_method_handler
def unary_unary_rpc_method_handler(behavior: Callable[[Any, grpc.aio.ServicerContext], Any],
                                   request_deserializer: Optional[Callable[[bytes], Any]]=None,
                                   response_serializer: Optional[Callable[[Any], bytes]]=None
    ) -> grpc.aio.RpcMethodHandler:
    """Creates an RpcMethodHandler for a unary-unary RPC method.

    Args:
      behavior: The implementation of an RPC that accepts one request
        and returns one response.
      request_deserializer: An optional behavior for request deserialization.
      response_serializer: An optional behavior for response serialization.

    Returns:
      An RpcMethodHandler object that is typically used by grpc.Server.
    """


# grpc.aio.unary_stream_rpc_method_handler
def unary_stream_rpc_method_handler(behavior: Callable[[Any, grpc.aio.ServicerContext], Optional[AsyncIterable[Any]]],
                                    request_deserializer: Optional[Callable[[bytes], Any]]=None,
                                    response_serializer: Optional[Callable[[Any], bytes]]=None
    ) -> grpc.aio.RpcMethodHandler:    
    """Creates an RpcMethodHandler for a unary-stream RPC method.

    Args:
      behavior: The implementation of an RPC that accepts one request
        and returns an iterator of response values.
      request_deserializer: An optional behavior for request deserialization.
      response_serializer: An optional behavior for response serialization.

    Returns:
      An RpcMethodHandler object that is typically used by grpc.Server.
    """


# grpc.aio.stream_unary_rpc_method_handler
def stream_unary_rpc_method_handler(behavior: Callable[[AsyncIterable[Any], grpc.aio.ServicerContext], Any],
                                    request_deserializer: Optional[Callable[[bytes], Any]]=None,
                                    response_serializer: Optional[Callable[[Any], bytes]]=None
    ) -> grpc.aio.RpcMethodHandler:
    """Creates an RpcMethodHandler for a stream-unary RPC method.

    Args:
      behavior: The implementation of an RPC that accepts an iterator of
        request values and returns a single response value.
      request_deserializer: An optional behavior for request deserialization.
      response_serializer: An optional behavior for response serialization.

    Returns:
      An RpcMethodHandler object that is typically used by grpc.Server.
    """


# grpc.aio.stream_stream_rpc_method_handler
def stream_stream_rpc_method_handler(behavior: Callable[[AsyncIterable[Any], grpc.aio.ServicerContext], Optional[AsyncIterable[Any]]],
                                     request_deserializer: Optional[Callable[[bytes], Any]]=None,
                                     response_serializer: Optional[Callable[[Any], bytes]]=None
    ) -> grpc.aio.RpcMethodHandler:
    """Creates an RpcMethodHandler for a stream-stream RPC method.

    Args:
      behavior: The implementation of an RPC that accepts an iterator of
        request values and returns an iterator of response values.
      request_deserializer: An optional behavior for request deserialization.
      response_serializer: An optional behavior for response serialization.

    Returns:
      An RpcMethodHandler object that is typically used by grpc.Server.
    """
```

```Python
# grpc.aio.method_handlers_generic_handler
def method_handlers_generic_handler(service: Text,
                                    method_handlers: Mapping[Text, grpc.aio.RpcMethodHandler]
    ) -> grpc.aio.GenericRpcHandler:
    """Creates a GenericRpcHandler from RpcMethodHandlers.

    Args:
      service: The name of the service that is implemented by the
        method_handlers.
      method_handlers: A dictionary that maps method names to corresponding
        RpcMethodHandler.

    Returns:
      A GenericRpcHandler. This is typically added to the grpc.Server object
      with add_generic_rpc_handlers() before starting the server.
    """
```

### Server Interceptor

```Python
# grpc.aio.ServerInterceptor
class ServerInterceptor:
    """Affords intercepting incoming RPCs on the service-side.

    This is an EXPERIMENTAL API.
    """

    async def intercept_service(self,
                          continuation: Callable[[grpc.HandlerCallDetails], grpc.aio.RpcMethodHandler],
                          handler_call_details: grpc.HandlerCallDetails
        ) -> grpc.aio.RpcMethodHandler:
        """Intercepts incoming RPCs before handing them over to a handler.

        Args:
          continuation: A function that takes a HandlerCallDetails and
            proceeds to invoke the next interceptor in the chain, if any,
            or the RPC handler lookup logic, with the call details passed
            as an argument, and returns an RpcMethodHandler instance if
            the RPC is considered serviced, or None otherwise.
          handler_call_details: A HandlerCallDetails describing the RPC.

        Returns:
          An RpcMethodHandler with which the RPC may be serviced if the
          interceptor chooses to service this RPC, or None otherwise.
        """
```

### Client Interceptor

```Python
# grpc.aio.UnaryUnaryClientInterceptor
class UnaryUnaryClientInterceptor:
    """Affords intercepting unary-unary invocations."""

    async def intercept_unary_unary(self,
                              continuation: Callable[[grpc.ClientCallDetails, Any], grpc.aio.Call[Any]],
                              client_call_details: grpc.ClientCallDetails,
                              request: Any
        ) -> grpc.aio.Call[Any]:
        """Intercepts a unary-unary invocation asynchronously.

        Args:
          continuation: A function that proceeds with the invocation by
            executing the next interceptor in chain or invoking the
            actual RPC on the underlying Channel. It is the interceptor's
            responsibility to call it if it decides to move the RPC forward.
            The interceptor can use
            `response_future = continuation(client_call_details, request)`
            to continue with the RPC. `continuation` returns an object that is
            both a Call for the RPC and a Future. In the event of RPC
            completion, the return Call-Future's result value will be
            the response message of the RPC. Should the event terminate
            with non-OK status, the returned Call-Future's exception value
            will be an RpcError.
          client_call_details: A ClientCallDetails object describing the
            outgoing RPC.
          request: The request value for the RPC.

        Returns:
            An object that is both a Call for the RPC and a Future.
            In the event of RPC completion, the return Call-Future's
            result value will be the response message of the RPC.
            Should the event terminate with non-OK status, the returned
            Call-Future's exception value will be an RpcError.
        """


# grpc.aio.UnaryStreamClientInterceptor
class UnaryStreamClientInterceptor:
    """Affords intercepting unary-stream invocations."""

    async def intercept_unary_stream(self,
                               continuation: Callable[[grpc.ClientCallDetails, Any], grpc.aio.Call[AsyncIterable[Any]]],
                               client_call_details: grpc.ClientCallDetails,
                               request: Any
        ) -> grpc.aio.Call[AsyncIterable[Any]]:
        """Intercepts a unary-stream invocation.

        Args:
          continuation: A function that proceeds with the invocation by
            executing the next interceptor in chain or invoking the
            actual RPC on the underlying Channel. It is the interceptor's
            responsibility to call it if it decides to move the RPC forward.
            The interceptor can use
            `response_iterator = continuation(client_call_details, request)`
            to continue with the RPC. `continuation` returns an object that is
            both a Call for the RPC and an iterator for response values.
            Drawing response values from the returned Call-iterator may
            raise RpcError indicating termination of the RPC with non-OK
            status.
          client_call_details: A ClientCallDetails object describing the
            outgoing RPC.
          request: The request value for the RPC.

        Returns:
            An object that is both a Call for the RPC and an iterator of
            response values. Drawing response values from the returned
            Call-iterator may raise RpcError indicating termination of
            the RPC with non-OK status.
        """


# grpc.aio.StreamUnaryClientInterceptor
class StreamUnaryClientInterceptor:
    """Affords intercepting stream-unary invocations."""

    async def intercept_stream_unary(self,
                               continuation: Callable[[grpc.ClientCallDetails, Optional[AsyncIterable[Any]]], grpc.aio.Call[Any]],
                               client_call_details: grpc.ClientCallDetails,
                               request_iterator: AsyncIterable[Any]
        ) -> grpc.aio.Call[Any]:
        """Intercepts a stream-unary invocation asynchronously.

        Args:
          continuation: A function that proceeds with the invocation by
            executing the next interceptor in chain or invoking the
            actual RPC on the underlying Channel. It is the interceptor's
            responsibility to call it if it decides to move the RPC forward.
            The interceptor can use
            `response_future = continuation(client_call_details, request_iterator)`
            to continue with the RPC. `continuation` returns an object that is
            both a Call for the RPC and a Future. In the event of RPC completion,
            the return Call-Future's result value will be the response message
            of the RPC. Should the event terminate with non-OK status, the
            returned Call-Future's exception value will be an RpcError.
          client_call_details: A ClientCallDetails object describing the
            outgoing RPC.
          request_iterator: An iterator that yields request values for the RPC.

        Returns:
          An object that is both a Call for the RPC and a Future.
          In the event of RPC completion, the return Call-Future's
          result value will be the response message of the RPC.
          Should the event terminate with non-OK status, the returned
          Call-Future's exception value will be an RpcError.
        """


# grpc.aio.StreamStreamClientInterceptor
class StreamStreamClientInterceptor:
    """Affords intercepting stream-stream invocations."""

    async def intercept_stream_stream(self,
                                continuation: Callable[[grpc.ClientCallDetails, Optional[AsyncIterable[Any]]], grpc.aio.Call[AsyncIterable[Any]]],
                                client_call_details: grpc.ClientCallDetails,
                                request_iterator: AsyncIterable[Any]
        ) -> grpc.aio.Call[AsyncIterable[Any]]:
        """Intercepts a stream-stream invocation.

        Args:
          continuation: A function that proceeds with the invocation by
            executing the next interceptor in chain or invoking the
            actual RPC on the underlying Channel. It is the interceptor's
            responsibility to call it if it decides to move the RPC forward.
            The interceptor can use
            `response_iterator = continuation(client_call_details, request_iterator)`
            to continue with the RPC. `continuation` returns an object that is
            both a Call for the RPC and an iterator for response values.
            Drawing response values from the returned Call-iterator may
            raise RpcError indicating termination of the RPC with non-OK
            status.
          client_call_details: A ClientCallDetails object describing the
            outgoing RPC.
          request_iterator: An iterator that yields request values for the RPC.

        Returns:
          An object that is both a Call for the RPC and an iterator of
          response values. Drawing response values from the returned
          Call-iterator may raise RpcError indicating termination of
          the RPC with non-OK status.
        """
```

### Utility Functions

```Python
# grpc.aio.EOF is a unique object per process that evaluates to False
class _EOF:
    def __bool__(self):
        return False

EOF = _EOF()
```

```Python
# grpc.aio.channel_ready
async def channel_ready(channel: grpc.aio.Channel) -> None:
    """Creates a coroutine that ends when a Channel is ready.

    Args:
      channel: A Channel object.
    """
```

### Shared APIs

APIs in the following categories remain in top level:
* Credentials related classes including for channel, call, and server;
* Channel connectivity Enum class;
* Status code Enum class;
* Compression method Enum class;
* `grpc.RpcError` exception class;
* `grpc.RpcContext` class;
* `grpc.ClientCallDetails` class;
* `grpc.HandlerCallDetails` class;


## Pending Topics

Reviewers has thrown many valuable proposals. This design doc may not be the
ideal place for those discussions.
* Re-design the connectivity API to be consistent with C-Core.
* Design a easy way to use channel arguments [grpc#19734](https://github.com/grpc/grpc/issues/19734).

## Related Issues

* https://github.com/grpc/grpc/issues/6046
* https://github.com/grpc/grpc/issues/18376
* https://github.com/grpc/grpc/projects/16

## Implementation

* TODO
