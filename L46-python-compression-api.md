Title
----
* Author(s): gnossen
* Approver: lidizheng
* Status: Draft
* Implemented in: python
* Last updated: 2/28/2019
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/c5SVRTiIjmA

## Abstract

Python clients and servers will provide an interface by which the user can
specify a compression method for RPC calls.

## Background

Whereas the Java, Go, and C++ implementations of gRPC provide APIs that allow
both clients and servers to specify a compression algorithm for requests and
responses, Python offers no comparable API.


### Related Proposals:

N/A

## Proposal

I propose adding a new `Compression` class defined as follows.

```python
class Compression(enum.IntEnum):
    NoCompression = _cygrpc.GRPC_COMPRESS_NONE
    Deflate = _cygrpc.GRPC_COMPRESS_DEFLATE
    Gzip = _cygrpc.GRPC_COMPRESS_GZIP
```

These contants mirror those in gRPC core.

In addition, I propose adding a `compression` keyword argument to the
`__call__`, `with_call`, and `future` methods of the `XXXMultiCallable` family
of interfaces. This argument will set the compression method to use for an
individual call.

```python
class UnaryUnaryMultiCallable(six.with_metaclass(abc.ABCMeta)):

    @abc.abstractmethod
    def __call__(self,
                 request,
                 timeout=None,
                 metadata=None,
                 credentials=None,
                 wait_for_ready=None,
                 compression=None):
        ...

    @abc.abstractmethod
    def with_call(self,
                  request,
                  timeout=None,
                  metadata=None,
                  credentials=None,
                  wait_for_ready=None,
                  compression=None):
        ...

    @abc.abstractmethod
    def future(self,
               request,
               timeout=None,
               metadata=None,
               credentials=None,
               wait_for_ready=None,
               compression=None):
        ...


class UnaryStreamMultiCallable(six.with_metaclass(abc.ABCMeta)):

    @abc.abstractmethod
    def __call__(self,
                 request,
                 timeout=None,
                 metadata=None,
                 credentials=None,
                 wait_for_ready=None,
                 compression=None):
        ...


class StreamUnaryMultiCallable(six.with_metaclass(abc.ABCMeta)):

    @abc.abstractmethod
    def __call__(self,
                 request_iterator,
                 timeout=None,
                 metadata=None,
                 credentials=None,
                 wait_for_ready=None,
                 compression=None):
        ...

    @abc.abstractmethod
    def with_call(self,
                  request_iterator,
                  timeout=None,
                  metadata=None,
                  credentials=None,
                  wait_for_ready=None,
                  compression=None):
        ...

    @abc.abstractmethod
    def future(self,
               request_iterator,
               timeout=None,
               metadata=None,
               credentials=None,
               wait_for_ready=None,
               compression=None):
        ...


class StreamStreamMultiCallable(six.with_metaclass(abc.ABCMeta)):

    @abc.abstractmethod
    def __call__(self,
                 request_iterator,
                 timeout=None,
                 metadata=None,
                 credentials=None,
                 wait_for_ready=None,
                 compression=None):
        ...
```

Analogously, a `compression` attrbute will be added to ClientCallDetails,
rendering its interface as follows:

```python
class ClientCallDetails(six.with_metaclass(abc.ABCMeta)):
    """Describes an RPC to be invoked.
    This is an EXPERIMENTAL API.
    Attributes:
      method: The method name of the RPC.
      timeout: An optional duration of time in seconds to allow for the RPC.
      metadata: Optional :term:`metadata` to be transmitted to
        the service-side of the RPC.
      credentials: An optional CallCredentials for the RPC.
      wait_for_ready: This is an EXPERIMENTAL argument. An optional flag t
        enable wait for ready mechanism.
      compression: An optional value specifying the type of compression to be
        used.
    """
```

In addition, a `compression` keyword argument will be aded to the
`insecure_channel`, `secure_channel`, and `server` functions. This will define
the compression method used throughout the lifetime of the channel or server.

Finally, `set_response_compression` method will be added to the
`ServicerContext` interface. This will allow servers to set the compression
method for an individual response.

If a type of compression is specified both at channel or server creation time
and for an individual message, then the type of compression specified for the
invidual message will be used.

If a type of compression is neither specified at channel or server creation nor
for an individual message, then no compression will be used.

### Examples

At channel creation time.
```python
with grpc.insecure_channel('localhost:50051',
                            compression=grpc.compression.Gzip) as channel:
    stub = helloworld_pb2_grpc.GreeterStub(channel)
    response = stub.SayHello(helloworld_pb2.HelloRequest(name='you'))
```

For individual messages on the client side.
```python
with grpc.insecure_channel('localhost:50051') as channel:
    stub = helloworld_pb2_grpc.GreeterStub(channel)
    response = stub.SayHello(
                helloworld_pb2.HelloRequest(name='you'),
                compression=grpc.compression.Gzip)
```

At server creation time.
```python
server = grpc.server(
            futures.ThreadPoolExecutor(max_workers=10),
            compression=grpc.DEFLATE)
helloworld_pb2_grpc.add_GreeterServicer_to_server(Greeter(), server)
server.add_insecure_port('[::]:50051')
server.start()
```

For individual responses.
```python
class Greeter(helloworld_pb2_grpc.GreeterServicer):

    def SayHello(self, request, context):
        context.set_response_compression(grpc.compression.NoCompression)
        return helloworld_pb2.HelloReply(message='Hello, %s!' % request.name)
```

## Rationale

Other implementations offer richer APIs that allow a user to define their own
compression algorithms. However, given the structure of gRPC core, it would not
be simple to implement this for gRPC Python. We are essentially limited to the
options offered by core. The proposal detailed here simply surfaces those
interfaces for Python users.


## Implementation

I (@gnossen) have implemented these changes in [this PR](https://github.com/grpc/grpc/pull/18564)
and will merge them after the proposal has been accepted.
