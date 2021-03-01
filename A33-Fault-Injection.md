Client-Side Fault Injection
----
* Author(s): lidiz
* Approver: markdroth
* Status: Approved
* Implemented in: all languages
* Last updated: Feb 23rd, 2021
* Discussion at: https://groups.google.com/g/grpc-io/c/d9dI5Hy9zzk

## Abstract

Fault injection is an essential feature that assists developers in building fault-tolerant applications. To keep gRPC’s competitive advantage, this doc aims to design a fault injection mechanism on the client side for gRPC while allowing transparent transition from Envoy configuration.


## Background

### Envoy HTTPFilter

Please refer to [HTTP connection management](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/http/http_connection_management#arch-overview-http-conn-man) section in Envoy document for how the filters are setup in xDS.

Please refer to [HTTP filters](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/http/http_filters) for how HTTP filters process traffic in xDS.

Please refer to [A39: xDS HTTP Filters](https://github.com/grpc/proposal/pull/219) for how xDS HTTP Filters are processed in gRPC.


### Fault Injection in xDS

Please refer to [Fault Injection](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/fault_filter) section in Envoy document.


## Proposal

### Overview

The full definition of Envoy v3 fault injection proto can be found at [HTTPFault](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/http/fault/v3/fault.proto#extensions-filters-http-fault-v3-httpfault). The provided feature can be categorized as 7 items. This proposal will talk about the ones that gRPC plans to support, the ones that will be deferred to future releases, and ones that will not be supported.


#### Supported Fault Injection Features

1. Injecting delays before starting a request;
1. Failing RPC with a particular gRPC status code;
1. Imposing a limit on the maximum number of currently active faults;
1. Controlling fault injection policies via HTTP headers (aka. gRPC metadata).


#### Deferred Fault Injection Features

1. Restricting per-stream response rate limit (in kbps, and selected by percentage) (see alternative section below for deferring rationale).
1. Selecting fault injecting traffic based on upstream cluster name, downstream node names, and headers (postponed until a valid use-case presents itself).


#### Unsupported Fault Injection Features

1. Configuring fault injection based on global runtime settings (because gRPC doesn't have the concept of a global runtime);


### Fault Injection Features

#### RPC delay injection

The injection of RPC delay translates to **injecting delay before sending any traffic for an RPC**.


#### RPC abort injection

This feature aborts RPC with given status code (see status code mapping section below for more details). In Envoy, the abort behavior in HTTP/2 happens immediately after the stream reaches the fault injection filter and the injected delay. In gRPC, it translates to that for **both unary and streaming RPCs the abort will happen before sending any traffic**.


#### Maximum active faults

Active fault number is a gauge representing the number of ongoing fault injected RPCs. The gauge increases when an RPC is injected by one or more faults, and it decreases when the injected fault is finished. For example, the delay injection is considered finished after the delay is finished, the abort injection is considered finished after the RPC is aborted.

This feature limits the maximum number of concurrent active fault injected RPC for the entire process. Apparently, this requires a global counter (gauge). This value is designed to be updated via multiple channels. So, **it needs to be thread-safe**, but the actual implementation depends on each stack. For example, volatile variable, atomic variable, memory order, locking. Anything works best for the language.


#### Header matching

Envoy allows applications to specify [HTTP headers](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/fault_filter#controlling-fault-injection-via-http-headers) to enable fault injection. This feature still needs a fault injection filter to be specified in the HTTP connection manager, and the delay or the abort field needs to explicitly say `HeaderDelay` or `HeaderAbort`.

Here is the set of fault injection policy configuring HTTP headers that gRPC will support:

```bash
# Aborts HTTP requests with status code 503
x-envoy-fault-abort-request=503

# Aborts gRPC requests with INVALID_ARGUMENT
x-envoy-fault-abort-grpc-request=3

# The percentage of requests to be failed. This header only sets numerator. The denominator was set in the HTTPFault filter, which by default is 100.
# 5% of requests should fail.
x-envoy-fault-abort-request-percentage=5

# Delays request for 53 milliseconds.
x-envoy-fault-delay-request=53

# The percentage of requests to be delayed. This header only sets numerator. The denominator was set in the HTTPFault filter, which by default is 100.
# 20% of requests should delay
x-envoy-fault-delay-request-percentage=20

# Not supported
x-envoy-fault-throughput-response=''
x-envoy-fault-throughput-response-percentage=''
```

Note that for both `X-envoy-fault-abort-request-percentage` and `x-envoy-fault-delay-request-percentage`, the percentage value is capped by the percentage value provided by the effective fault injection filter config. For example, if the abort fractional percentage is 500000/1000000 based on LDS and RDS, and the header can only change the fractional percentage to [0, 500000], above 500000 will be limited to 500000. Here is the quote from the [HTTP headers](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/fault_filter#controlling-fault-injection-via-http-headers) section:

> ...the percentage of request to apply aborts to and must be greater or equal to 0 and its maximum value is capped by the value of the numerator of percentage field.

Note that if any header is invalid, for example, passing non-numerical value as percentage header, the invalid headers will be ignored.

Note if there are multiple valid headers with the same key name, gRPC implementations may choose any one of them.

**Fault inject config in HTTP filter and headers won’t be conflicted**, because the header only being evaluated if the delay or abort is set to `HeaderDelay` or `HeaderAbort` instead of concrete values.


### Status Code Mapping

Both `http_status` and `grpc_status` are available in xDS v3 API. Mapping between HTTP status code and gRPC status code is well defined (see [doc](https://github.com/grpc/grpc/blob/master/doc/http-grpc-status-mapping.md)), but using HTTP status code means users are limited to inject only 7 out of 16 gRPC status codes (including `OK`). The underlying schema for abort looks like [FaultAbort](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/http/fault/v3/fault.proto#extensions-filters-http-fault-v3-faultabort).

gRPC should accept fault injection settings with either HTTP status code or gRPC status code. **The ConfigSelector should translate the HTTP status code into gRPC status code** before sending the config to the fault injection enforcer.


### 200/OK Error Injection

In order for gRPC to provide behavior consistent with Envoy, there are two counter-intuitive edge cases that are worth calling out explicitly.

If the filter config sets `http_status` to 200, gRPC will abort with gRPC status `UNKNOWN`.  This is consistent with what gRPC would do if it received an HTTP 200 status with no `grpc-status` header, because this is the normal mapping for the HTTP 200 status in the [HTTP-gRPC status mapping]((https://github.com/grpc/grpc/blob/master/doc/http-grpc-status-mapping.md)) mentioned above.

If the filter config sets `grpc_status` to `OK`, gRPC will react as if it has immediately received trailing metadata from the server with the status `OK`. Note that in this case, the exact behavior seen by the application depends on the type of the RPC and implementation of gRPC. For unary and client-streaming RPCs, the client may return a non-OK status to the application, because it was expecting to receive exactly one message from the server, which never arrived. For server-streaming and bidi-streaming RPCs, the client may return status `OK` to the application without returning any messages from the server.


### Evaluate Possibility Fraction

According to the [Envoy implementation](https://github.com/envoyproxy/envoy/blob/main/source/extensions/filters/http/fault/fault_filter.cc#L222), each feature of fault injection is checked with a different random number. Compared to using a single random number for all features, this affects the probability of an RPC being affected by fault injection. For example, the fault injection has 20% chance to delay and 5% chance to abort. Under the single random number solution, only 20% RPC will be affected. But if two features are independent, 24% RPC will be affected. So, **this doc suggests to make each feature independent by using a different random number each time.**


### Experimental Environment Variable Protection

`GRPC_XDS_EXPERIMENTAL_FAULT_INJECTION` as an environment variable will be used to guard this feature for the initial release. Once it is set, gRPC will start to interpret xDS HTTP filters ([A39: xDS HTTP Filters](https://github.com/grpc/proposal/pull/219)) and enforce fault injection if configured. This environment variable protection will be removed once the new feature has proven to be stable.


## Alternatives

### Alternative 1: Fault Injection Config via ProtoBuf Metadata

After ConfigSeletor calculated the final fault injection config, it can encode the config proto and set it as metadata for that RPC. This approach is straightforward and allows more cohesive logic in the fault injection enforcer. On the other hand, it requires additional serialization and deserialization of proto messages, and it can reduce debuggability.

```bash
grpc-fault-injection-policy-bin=<...>  # The encoded FI config proto message
```

However, setting additional metadata may leak implementation details onto wire, which is undesired.


### Alternative 2: Fault Injection Config via ServiceConfig

Across Core, Java and Go, Service Config is a mechanism to supply advanced configuration for specific service or method. It’s natural to add a new section of fault injection policy into the MethodConfig. The proposed proto configuration looks like:

```protobuf
message MethodConfig {

 // ...ignoring other fields

 // The fault injection policy for outbound RPCs.
 //
 // The fault injection offers delay injection and aborting RPC functionality.
 // Both of them can be specified at the same time. If both features are
 // triggered, the RPC will be delayed first then abort with the status code.
 message FaultInjectionPolicy {
   // The fixed delay before starting the RPC.
   //
   // No traffic should be sent until the delay phase ends. By default, the
   // delay duration is 0 (0 seconds and 0 nanos). If the delay is 0, the RPC
   // delay feature is disabled and delay_fraction will be ignored.
   google.protobuf.Duration delay = 1;

   // The probability that this RPC will be delayed.
   //
   // The default value is 0. The probability is represented as a fraction over
   // 1 million (1000000). For example, 0.625% will be 6250, 12.5% will be
   // 125000. Its value should be clamped into the interval [0, 1000000].
   uint32 delay_fraction = 2;

   // The RPC status (code and message) for the aborted RPC.
   //
   // By default the status code is 0 (OK), which disables the RPC abort
   // feature and ignores abort_percent. If the control plane only offers
   // HTTP status code, the application should map it to cardinal RPC status
   // code.
   //
   // By default the status message is empty. If the status message is
   // specified with non-OK status code, then the status message will be added
   // to the aborted RPC.
   google.rpc.Status abort_status = 3;

   // The probability that this RPC will be aborted.
   //
   // The default value is 0. The probability is represented as a fraction over
   // 1 million (1000000). For example, 0.625% will be 6250, 12.5% will be
   // 125000. Its value should be clamped into the interval [0, 1000000].
   uint32 abort_fraction = 4;
 }

 FaultInjectionPolicy fault_injection_policy = 8;
}
```

Turned out, this approach requires more engineering effort, but makes the fault injection feature available to non-xDS gRPC applications. gRPC team does consider to improve service config with fault injection settings, and will carry-on this work in a separate proposal in future.


### Support for Response Rate Limit Algorithm

Envoy uses the token bucket algorithm to throttle the speed of letting response pass through this filter. It will try to write as much as possible from the buffer, then it will schedule another write when the next token is available. Each token permits to write `(max_kbps * 1024) / SecondDivisor` of bytes. The `SecondDivisor` is a constant 16, which splits 1 second into 16 segments (~63ms apart). Only the body of the HTTP stream is throttled by the response rate limit algorithm. Not headers or trailers.

Envoy implements fault injection is an `HTTPFilter`, which can choose to operate on bytes or HTTP messages or events. It is unclear if gRPC can perform the precise same behavior, since gRPC generally operates on proto messages (only Core has visibility to bytes, not Java/Golang). More importantly, there is not enough consensus for if the timeout timer should start at the beginning of an RPC, or the receipt of the first bytes from peer, or after the entire message is received.


## Implementation

gRPC Core: https://github.com/grpc/grpc/pull/24354
