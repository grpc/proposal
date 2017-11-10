Title
----
* Author(s): ZhouyihaiDing
* Approver: mehrdada
* Status: Draft
* Implemented in: PHP
* Last updated: 
* Discussion at:
## Abstract

This proposal introduces a client-side facility to intercept RPC invocations in gRPC PHP.

____
## Background

The PHP language implementation for gRPC does not currently have an
interceptor interface, whereas other languages like Go and Java do.
The interceptor facility make it easier to extend the gRPC PHP
stack and helps in adding functionality such as logging, adding additional
headers to outgoing calls, or caching responses.


### Related Proposals

* [Python Client and Server Interceptors](https://github.com/grpc/proposal/pull/39)
* [Ruby Client and Server Interceptors](https://github.com/grpc/proposal/pull/34)
* [C# Client and Server Interceptors](https://github.com/grpc/proposal/pull/38)

## Proposal

Add client interceptors to the PHP gRPC implementation. In the implementation, 
the channel is wrapper around an intercepted channel and the stub can use it the
same way as the normal channel. RPCs dispatched from such stubs then go through
the interceptor chain for processing before they hit the underlying gRPC channel. 

## Implementaion
In this proposal, intercepted channels are created using the
`new Grpc\Interceptor::intercept($channel, $interceptor)` function.
This function takes a channel, and one or more interceptor objects and returns an
intercepted channel. The return value is an object that shares the same public
interface as the `Grpc\Channel` class and can be used in place of a `Grpc\Channel`
object wherever that is accepted in the API, e.g. when instantiating a stub,
or to pass it to the `Grpc\Interceptor::intercept` function.

When a stub is created with an intercepted channel, the RPCs invoked through that stub will be intercepted by the interceptors registered on the channel that is used to create the stub.
 
```php
Grpc\Interceptor::intercept(	// return intercepted channel
	Channel				// underlying normal channel or intercepted channel
	Grpc\Interceptor interceptor    // interceptors to add
)
```

`Grpc\Interceptor` provides four methods for the user to override. In each method, the user
has access to the method, metadata and call options which are used to create the Call
object. In unary client method, users can access to the argument(request) and can modify it
directly, while in stream client method, the user needs to wrap the call to access to it, which
is described later. The interceptor provides per-call interception for gRPC PHP
services and clients.

By `Grpc\Interceptor`, the user can do the same intercept thing for both unary and stream
call within one interceptor. Each interceptor can choose then to combine
functionality for different request types into a protected or private method,
or handle each request type differently, based on the needs of the interceptor.
The structure for methods in `Grpc\Interceptor` is shown below:

```php
class Interceptor{
  /**
     * @param string   $method       The name of the method to call
     * @param mixed    $argument     The argument to the method
     * @param array    $metadata     A metadata map to send to the server(optional)
     * @param array    $options      An array of call_options (optional)
     * @param function $continuation Used to invoke the next interceptor.
     *
     * @return \Closure A function which can create a UnaryCall
     */
  public function interceptUnaryUnary($method, $argument, array $metadata = [], array $options = [], $continuation){}

  public function interceptStreamUnary($method, array $metadata = [], array $options = [], $continuation){}

  public function interceptUnaryStream($method, $argument,array $metadata = [], array $options = [], $continuation){}

  public function interceptStreamStream($method,array $metadata = [], array $options = [], $continuation){}
}
```

The last parameter `$continuation()` is a callable value which lets the user to decide
to invoke the next Interceptor or stop RPC. By default each method will invoke the next
interceptor and the return value will be the Call object from the next wrapper level.
The user can stop the RPC by not calling the `$continuation()` and return anything like \Exception or no return.

As you can see from the `Grpc\Interceptor`, stream client doesn’t have the request
argument. The user need to wrap the return value `$continuation()` which is a Call
object into a new Call object to intercept the request. The way to intercept a
request looks like below:

```php
Class RequestInterceptCall(
	private $call;
	public construct($call){
		$this->call = $call;
	}
	public function write($request){
		anything_intercept_request($request);
		return $this->call->write($request);
  }
  Public function wait(){
    return $this->call->wait();
  }	
)
class UserInterceptor extends Grpc\Interceptor{
  public function StreamUnary($method, array $metadata = [], array $options = [], $continuation){
    return new RequestInterceptCall($continuation($method, $argument, $metadata, $options));
  }
  public function StreamStream($method, array $metadata = [], array $options = [], $continuation){
    return new RequestInterceptCall($continuation($method, $argument, $metadata, $options));
  }
}
```

The Call intercept will be executed with the order same as the `Grpc\Interceptor`.
In other words, if there are multiple Call intercept followed by multiple
`Grpc\Interceptor`, the order of execution `$call->wait($request)` will start from
the most outside wrapped call to the next level, until it hits the deepest Call object.

### Interceptor Examples
#### Illustrating Example 1: Logging Interceptor

This interceptor naively logs outbound requests and the method they call, and
injects a unique request_id metadata value into the metadata sent:

```php
class LogInterceptor extends \Grpc\Interceptor
{
    public function __construct(){
        ini_set("log_errors", 1);
        ini_set("error_log", "php-error.log");
    }

    public function UnaryUnary($method, $argument, $metadata, $options, $continuation){
        error_log("Sending request/response to");
        error_log($method);
        $metadata['request_id'] = array((string) rand());
        return $continuation($method, $argument, $metadata, $options);
    }

    public function StreamUnary($method, $metadata, $options, $continuation){
        error_log("Sending client streamer to");
        error_log($method);
        $metadata['request_id'] = array((string) rand());
        return $continuation($request, $call, $method, $metadata);
    }
    
    public function UnaryStream($method, $argument, $metadata, $options, $continuation) {
        error_log("Sending server streamer to");
        error_log($method);
        $metadata['request_id'] = array((string) rand());
        return $continuation($request, $call, $method, $metadata);
    }

    public function StreamStream($method, $metadata, $options, $continuation) {
        error_log("Sending bidi streamer to");
        error_log($method);
        $metadata['request_id'] = array((string) rand());
        return $continuation($request, $call, $method, $metadata);
    }
}
```

#### Illustrating Example 2: Modify request and metadata in unary client
```php
class MyInterceptor extends \Grpc\Interceptor {
    public function UnaryUnary($method, $argument, $metadata, $options, $continuation) {
        $metadata["foo"] = array('interceptor_from_request_response');
        return $continuation($method, $argument, $metadata, $options);
    }
}
class MyInterceptor2 extends \Grpc\Interceptor {
    public function UnaryUnary($method, $argument, $metadata, $options, $continuation) {
        $argument->setName("world from interceptor");
        return $continuation($method, $argument, $metadata, $options);
    }
}
$change_metadata_interceptor = new MyInterceptor();
$change_request_interceptor = new MyInterceptor2();
$log_interceptor = new LogInterceptor();
$channel_intercept1 = Grpc\Interceptor::intercept($real_channel, $change_metadata_interceptor);
$channel_intercept2 = Grpc\Interceptor::intercept($node1, $change_request_interceptor);
// or $channel_intercept2 = Grpc\Interceptor::intercept($real_channel, [
// $change_metadata_interceptor, $change_request_interceptor];
$client = new Helloworld\GreeterClient('localhost:50051', [
    'credentials' => Grpc\ChannelCredentials::createInsecure(),
    ],
    $channel_intercept2
);
```

#### Illustrating Example 3: Modify request in stream client
```php
class MyInterceptRequestCall {
    private $call;
    public function __construct($call){
        $this->call = $call;
    }
    public function write($request){
        $a = $request->getLatitude();
        $b = $request->getLongitude();
        $request->setLatitude($a/2);
        $request->setLongitude($b/2);
        $this->call->write($request);
    }
    public function wait(){
        return $this->call->wait();
    }
}
class changeMetadataInterceptor extends Grpc\Interceptor{
    public function StreamUnary($method, $deserialize, array $metadata = [], array $options = [], $continuation){
        $metadata["foo"] = array('interceptor_from_request_response');
        return new MyInterceptRequestCall($continuation($method, $deserialize, $metadata, $options));
    }
}

$change_metadata_interceptor = new changeMetadataInterceptor();
$intercept_metadata_channel = Grpc\InterceptorHelper::intercept($channel, $change_metadata_interceptor);
$client = new Routeguide\RouteGuideClient('localhost:50051', [
    'credentials' => Grpc\ChannelCredentials::createInsecure()],
     $intercept_metadata_channel
);
$call = $client->RecordRoute();
for ($i = 0; $i < 10; ++$i) {
    $point = new Point();
    $call->write($point);
}
```

## Implementation

Implemented in [gRPC PHP Client Interceptor pull request][impl]

[impl]: https://github.com/grpc/grpc/pull/13342
