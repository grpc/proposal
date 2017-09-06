Title
----
* Author(s): Shaun McCormick (@splittingred)
* Approver: mehrdada / apolcyn
* Status: In Review
* Implemented in: Ruby
* Last updated: Sept 6, 2017
* Discussion at: https://groups.google.com/d/msg/grpc-io/YRfZe7IH69k/FdSI_CGEAAAJ

## Abstract

Ruby gRPC client stubs and servers will present an interceptor interface, which will allow configuration of 
interceptors that will execute on any inbound or outbound operation. The server and client interceptor APIs will
behave similarly, using a compositional model for interceptors that allow for request-type-specific interception. 

## Background

The Ruby language implementation for gRPC does not currently have an interceptor interface, whereas other languages
(ex: Go, Java) do. Application concerns and extensibility are currently closed in the Ruby implementation, making
it difficult to extend the libraries for various application needs. 

### Related Proposals: 

* None

## Proposal

Add both server and client interceptors to the Ruby gRPC implementation. Similar to the Golang implementation,
interceptors are set at the initialization of the client stub and RPC Server. They provide per-call interception
for gRPC Ruby services and clients.

### Client Interceptors

This interceptor naively logs outbound requests and the method they call, and injects
a unique `request_id` metadata value into the metadata sent:

```ruby
require 'securerandom'

class AppClientInterceptor < GRPC::ClientInterceptor
  def request_response(request:, call:, method:, metadata:)
    logger.info "Sending request/response to #{method}"
    metadata['request_id'] = generate_request_id
    yield
  end
  
  def client_streamer(requests:, call:, method:, metadata:)
    logger.info "Sending client streamer to #{method}"
    metadata['request_id'] = generate_request_id
    yield
  end
  
  def server_streamer(request:, call:, method:, metadata:)
    logger.info "Sending client streamer to #{method}"
    metadata['request_id'] = generate_request_id
    yield
  end
    
  def bidi_streamer(requests:, call:, method:, metadata:)
    logger.info "Sending bidi streamer to #{method}"
    metadata['request_id'] = generate_request_id
    yield
  end
  
  private
  
  def logger
    @logger ||= Logger.new(STDOUT)
  end
  
  def generate_request_id
    SecureRandom.uuid
  end
end
```

Each interceptor can choose then to combine functionality for different request types into a protected 
or private method, or handle each request type differently, based on the needs of the interceptor.

You'll also note that the interceptor must yield back in order to complete the call. Then the interceptor
can be added to the stub via the initializer, which passes in an array of interceptors:

```ruby
MyStub.new(
  '0.0.0.0:0', 
  :this_channel_is_insecure, 
  interceptors: [AppClientInterceptor.new]
)
```

Interceptors maintain order of insertion by using the FIFO execution order native to Ruby's Array. 
In other words, if multiple interceptors are added, they will intercept in the order they were set in the 
passed array.

#### Client Interception API

Client interceptors have four separate methods, one for each request type. Their arguments are similar;
the only difference being if it is a streamed call, `request` will be `requests` and have an array of the
requests that were sent. 

For example, the arguments for a unary request/response call:

```ruby
##
# @param [Object] request The protobuf request message
# @param [GRPC::ActiveCall::InterceptableView] call A restricted view of the active call object
# @param [Method] method The method being called
# @param [Hash] metadata The call's outbound metadata
def request_response(request:, call:, method:, metadata:)
  yield
end
```

The call object is a restricted view, which only exposes the `deadline` attribute that was set on the call.

Also, the metadata hash here is able to be updated and will propagate to the server when the call executes.
This allows client interceptors to inject metadata for calls dynamically.

### Server Interceptors

This interceptor logs incoming requests and the service and method they call:

```ruby
class ServerRequestLogInterceptor < GRPC::ServerInterceptor
  def request_response(request:, call:, method:)
    response = yield
    log(method: method, response: response)
    response
  end
  
  def client_streamer(call:, method:)
    response = yield
    log(method: method, response: response)
    response
  end
  
  def server_streamer(request:, call:, method:)
    response = yield
    log(method: method, response: response)
    response
  end
  
  def bidi_streamer(requests:, call:, method:)
    response = yield
    log(method: method, response: response)
    response
  end
  
  private
  
  def log(method:, response:)
    mth = "#{method.owner.name}.#{method.name}"
    if response.is_a?(GRPC::BadStatus)
      logger.info "[GRPC::Ok] (#{mth})"
    else
      logger.error "[#{response.class.name}] (#{mth}) #{response.message}"
    end
  end
  
  def logger
    @logger ||= Logger.new(STDOUT)
  end
end
```

The interceptor can then be loaded via the initializer for the service:


```ruby
server = GRPC::RpcServer.new(
  interceptors: [ServerRequestLogInterceptor.new] 
)
server.add_http2_port('0.0.0.0:0', :this_port_is_insecure)
server.handle(MyService)
server.run_till_terminated
```

This example will produce logs like:

> [GRPC::Ok] (MyService.a_rpc_method)

> [GRPC::Internal] (MyService.a_failing_rpc_method) An error occurred in my service!

### Server Interception API

Server interceptors act as decorators around handlers. Similar to client interceptors, server interceptors
have four separate methods, one for each request type. 

Unary and server streamer interceptor calls will have a single request,
`SingleReqView` view of the active call, and the method being requested. Client streamer interceptor calls 
have only the `MultiReqView` and method passed (similar to the handler). Finally, the bidi call will receive 
an enumerable of requests that are passed.

For example, the arguments for a unary request/response call:

```ruby
##
# @param [Object] request The protobuf request message
# @param [GRPC::ActiveCall::SingleReqView] call A restricted view of the active call object
# @param [Method] method The method being called
def request_response(request:, call:, method:)
  yield
end
```

Trailing metadata is able to be modified as well in interceptors, via the normal call API:

```ruby
call.output_metadata[:my_key] = 'my_value'
```

## Rationale

To provide support for intercepting both at the client and server level, which is currently not supported,
and to bring parity for Ruby with the other language implementations.

Methods per-request-type were done as opposed to a singular interceptor method (such as a `intercept` method
on each interceptor) in order to clearly distinguish the type of request that is coming in. Interceptors can
from that consolidate their behavior, but the separation makes for cleaner delineation between the types of 
requests and makes explicit the differences between each in the arguments.

## Implementation

Implementation is currently finishing review here: https://github.com/grpc/grpc/pull/12100

### General Changes

* Introduce a `GRPC::Interceptor` base class that client and server interceptors derive from, and provide
  the internal API for an options Hash in the interceptor class itself that derived interceptors can utilize
* Introduce a new `GRPC::InterceptorRegistry` class that centrally handles the storage of the interceptors
* Introduce an `GRPC::InterceptionContext` class that centralizes the logic required to recursively handle
  the interception of calls for both client and server interceptors
* Refactor `GRPC::Generic::BidiCall` to return the `replies` generated so interceptors can receive it 

### Client Interceptor Changes

* Add an `interceptors` argument that accepts an Array to the `ClientStub` initializer and builds a
  `GRPC::InterceptorRegistry` instance from it
* Add an `InterceptableView` view that restricts access to the call to only be the `deadline` attribute
  and a new `interceptable` method to the `ActiveCall` object that returns the view
* Add wrapping for each request type (such as `request_response`, `client_streamer`, etc) 
  in the `ClientStub` class that builds an interception context and wraps the `ActiveCall` call in an
  interception block

### Server Interceptor Changes

* Add an `interceptors` argument that accepts an Array to the `GRPC::RpcServer` initializer and 
  builds an `GRPC::InterceptorRegistry` instance from it
* Update the `handle_*` methods in `GRPC::Generic::RpcDesc` to take in a `GRPC::InterceptionContext` 
  instance, which then wraps the calls to the handler in an interception block 
