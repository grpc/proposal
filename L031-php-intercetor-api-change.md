Title
----
* Author(s): ZhouyihaiDing
* Approver: mehrdada
* Status: Approved
* Implemented in: PHP
* Last updated: October 24, 2018
* Discussion at: https://groups.google.com/forum/#!searchin/grpc-io/L31|sort:date/grpc-io/DsjvtmeJJPU/QF2-99-oCAAJ
## Abstract

This proposal introduces an interceptor API change in the gRPC PHP.

____
## Background

gRPC PHP interceptor has ability to intercept each RPC by modifying `method`, `argument`,
`metadata` and `options`. It is better if we can modify the `deserialize`.

The first reason is that since the interceptor can manipulate the `method` before
the RPC starts, `deserialize` function should also be updated to be able to couple
with the `method` when the response receives.

The second reason we do not need to hide the `deserialize` because hiding `channel`
already satisfies the purpose of the interceptor.

### Related Proposals

* [PHP Client Interceptors](https://github.com/grpc/proposal/blob/master/L15-PHP-Interceptors.md)

## Proposal

Add `$deserialize` as the argument for the interceptor API.

## Implementaion
The only change is that four methods inside the `Interceptor` take `$deserialize` as
the argument. The new API looks like below:

```php
class Interceptor{
  /**
     * @param string   $method       The name of the method to call
     * @param mixed    $argument     The argument to the method
     * @param string   $deserialize  A function that deserializes the response
     * @param array    $metadata     A metadata map to send to the server(optional)
     * @param array    $options      An array of call_options (optional)
     * @param function $continuation Used to invoke the next interceptor.
     *
     * @return \Closure A function which can create a UnaryCall
     */
  public function interceptUnaryUnary($method, $argument, $deserialize, array $metadata = [], array $options = [], $continuation){}

  public function interceptStreamUnary($method, $deserialize, array $metadata = [], array $options = [], $continuation){}

  public function interceptUnaryStream($method, $argument, $deserialize, array $metadata = [], array $options = [], $continuation){}

  public function interceptStreamStream($method, $deserialize, array $metadata = [], array $options = [], $continuation){}
}
```

## Implementation

Implemented in [PHP: add deserialze as the argument for the interceptor][impl]

[impl]: https://github.com/grpc/grpc/pull/15779
