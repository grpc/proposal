Client-Side Fault Injection
----
* Author(s): lidiz
* Approver: roth, ejona, dfawley
* Status: In Review
* Implemented in: all languages
* Last updated: August 13th, 2020
* Discussion at: TBD

## Abstract

Fault injection is an essential feature that assists developers in building fault-tolerant applications. To keep gRPC’s competitive advantage, this doc aims to design a fault injection mechanism on the client side for gRPC while allowing transparent transition from Envoy configuration.

## Background

### Envoy HTTPFilter

Envoy has a network level filter named HTTP Connection Manager. It is responsible to handle common functionalities against HTTP connections and requests. In its `http_filters` field, users can set up multiple HTTPFilters. Each HTTPFilter requires at least a name. The name will be the identifier that assists future updates of filter configuration. For example, users can specify an `{"name": "envoy.cors"}` filter in `http_filters` with no content, then update the CORS filter’s configuration with the `typed_per_filter_config` field of a VirtualHost in RDS.

There are currently four ways to set the filter configuration, ordered by ascending priority:

Defining filter config in `http_filters` of HTTP Connection Manager;
Overriding filter config with `typed_per_filter_config` of VirtualHost from RDS;
Overriding filter config with `typed_per_filter_config` of Route from RDS;
Overriding filter config with `typed_per_filter_config` of ClusterWeight from RDS;

Also, there is the upcoming Filter Config Discovery Service (FCDS) (see [issue](https://github.com/envoyproxy/envoy/issues/7867)). It allows application to stream / delta / fetch filter configs. It’s not yet released and lacks documentation. From its source code, it updates the global `http_filters`. So, its priority should be between item 1 and item 2.

### Fault Injection in Envoy xDS API

Fault injection in xDS API is an HTTPFilter named `HTTPFault`. As mentioned above, the fault injection config can be provided by **LDS** (HTTP Connection Manager) and **RDS** (VirtualHost, Route, ClusterWeight). Besides, the fault injection filter offers [global Runtime settings](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/fault_filter#runtime) that can override global default fault injection filter config. For the explanation of Envoy Runtime, see section below.

Currently, the Traffic Director (TD) isn’t forwarding HTTPFilter configs in HTTP Connection Manager from LDS, and there isn’t an exposed API allowing users to directly config fault injection for all HTTP connections. To pave gRPC’s way for more extensive control plane compatibility, we need to support fetching filter config from HTTP Connection Manager among LDS responses.

The Envoy Runtime isn’t yet supported by TD or gRPC. The Runtime path also has a lower priority to implement.

On the other hand, TD has a released support for fault injection via route configuration for HTTP. So, **this design doc focuses on getting fault injection filter config from LDS and RDS**.

### Fault Injection Features

Fault Injection Options
The full definition of Envoy v3 fault injection proto can be found [here](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/http/fault/v3/fault.proto#extensions-filters-http-fault-v3-httpfault). In short, it allows users to configure following options:

1. Fixed delay before forwarding the request;
1. Overridden gRPC status code;
1. Traffic matcher (upstream cluster name, downstream cluster name, and headers);
1. Max active faults;
1. Per-Stream Response rate limit (kbps, and selected by percentage);
1. Options to override global runtime settings. 

Feature 1-4 should be implemented to make gRPC compatible with OSS control planes, and we should aim to ship them in the initial implementation. Feature 5 is postponed until we have a consensence on its behavior (see timeout issue below). Feature 6 is ignored since there is no plan to support Runtime settings.

To support all features above, it requires an API as powerful as the interceptors, which can intercept each individual RPC in their early stages and apply fault injection policy.

**Definition of active faults:** Active fault number is a gauge representing the number of ongoing fault injected RPCs. The gauge increases when an RPC is injected by one or more faults, and it decreases when the injected RPC is finished (see [FaultFilter::onDestroy](**https://source.corp.google.com/piper///depot/google3/third_party/envoy/src/source/extensions/filters/http/fault/fault_filter.cc;l=411;bpv=1;bpt=1**)).

**Response rate limit algorithm:** Envoy uses the token bucket algorithm to throttle the speed of letting response pass through this filter. It will try to write as much as possible from the buffer, then it will schedule another write when the next token is available.

Each token permits to write `(max_kbps * 1024) / SecondDivisor` of bytes. The `SecondDivisor` is a constant 16, which splits 1 second into 16 segments (~63ms apart).

Only the body of the HTTP stream is throttled by the response rate limit algorithm. Not headers or trailers.

**Response rate limit and timeout:** Envoy implements fault injection is an `HTTPFilter`, which can choose to operate on bytes or HTTP messages or events. It is unclear if gRPC can perform the precise same behavior, since gRPC generally operates on proto messages (Core has visibility to bytes). The timeout timer may start at the beginning of an RPC, or the receival of the first bytes from peer, or after the entire message is received.

### Fault Injection in Traffic Director

The traffic director implemented a fraction of Envoy’s fault injections features (see [TD’s urlMap](https://cloud.google.com/compute/docs/reference/rest/v1/urlMaps)). It only exposed two configurations: **fixed delay injection**, **RPC abort**.

Here is the available config for TD configuration:
```json
"faultInjectionPolicy": {
        "delay": {
                "fixedDelay": {
                        "seconds": string,
                        "nanos": integer
                },
                "percentage": number
        },
        "abort": {
                "httpStatus": integer,
                "percentage": number
        }
}
```

### Absence of Envoy Runtime in gRPC

Envoy has a mechanism to allow reload of configuration without restarting the entire process named [Runtime](https://www.envoyproxy.io/docs/envoy/latest/operations/runtime). It can be loaded from a static file, updated with a local file system, or overridden from an Admin console ([reference](https://www.envoyproxy.io/docs/envoy/latest/configuration/operations/runtime)).

However, as a library, gRPC doesn’t have a decent standing to maintain an “Runtime” state. Available options are 1) ignore Runtime settings; 2) allow initial Runtime setting (from a file or env variables). According to previous design decisions, the Runtime settings are usually ignored (see [gRPC xDS Traffic Splitting and Routing](https://github.com/grpc/proposal/blob/master/A28-xds-traffic-splitting-and-routing.md)).

### Fault Injection HTTP Headers in Envoy

Envoy allows applications to specify [HTTP headers](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/fault_filter#controlling-fault-injection-via-http-headers) to enable fault injection. This feature still needs a fault injection filter to be specified, and the delay or the abort field needs to explicitly say `HeaderDelay` or `HeaderAbort`.

### Prior Art: Envoy HTTP Filter

Envoy filters have very granular logic injection points at different stages of a request (or stream). For example, while decoding headers / data / trailer, encoding headers / data / trailer, or completion of decoding / encoding. In each injection point, the return value dictates whether Envoy should proceed to other filters or stop. This design gives immense flexibility for filters. The delay injection could be a headache since it can’t block the thread. But fault injection filters utilize this design and are able to schedule callback (rest of request processing logic) asynchronous.

## Proposal

### Priority Between Fault Injection And Retry

In Envoy, HTTP filters (including fault injection filter) are executed before router filters. Retry is part of the router filter. If an RPC is aborted, the router filter won’t see this RPC and the RPC won’t be retried. In gRPC, we will align our behavior with Envoy.

Based on a local experiment, if an RPC is delayed and failed by peer (not fault injected), Envoy will retry without any additional delay. There is an ongoing GitHub thread about the conflict between fault injection and retry at [istio/istio#13705](https://github.com/istio/istio/issues/13705).

In summary, **if retry and delay both take effect, only delay once before sending any traffic; if retry and abort both take effect, retry will be disabled; if retry, delay and abort all take effect, the RPC will be delayed then aborted without retry.**

### Enforce Fault Injection

Take Envoy as examples, it utilizes a powerful traffic interception API to implement fault injection. Here are possible solutions to implement FI:

1. **Returning an interceptor from the ConfigSelector.** In the [ConfigSelector design doc](https://github.com/grpc/proposal/pull/192/files/b2fe83ad515e8a72f669dd3f84d185b7f00c07f3#diff-560d8f55a3a659fca6dfaa0823d4e1d1R56), supporting interceptor is one of the proposed features. But it’s not available in actual implementations yet. (Viable for Java, Go)
2. **Defining a fault injection global interceptor.** The interceptor inspects the method config of each RPC. Then it makes the decision whether to delay or to abort. (Viable for Java, Go, possibly for filters in Core)
3. **Implementing fault injection as a new RPC feature controlled by service config.** In this way, fault injection will have similar architecture as timeout, wait for ready, and retry (gRFC A6). The logic needs to be implemented in native RPC call processing code path. (Viable for Core, Java, Go)
4. **Using load balancer policy.** Load balancer is capable of aborting an RPC, and able to mimic delay by returning “no subchannel, but wait for X seconds before trying again”. (Viable for Core, Java, Go)

**Java and Go will implement via solution 1 (ConfigSelector)**, and **Core will implement fault injection in client channel** with an non-service-config mechanism to fetch the FI config.

### Fault Injection Features

#### RPC delay injection

The injection of RPC delay translates to **injecting delay before sending any traffic for an RPC**. As discussed above, the delay should be injected only once regardless of how many following retries await.

#### RPC abort injection

This feature aborts RPC with given status code (see status code mapping section below for more details). In Envoy, the abort behavior in HTTP/2 happens immediately after the stream reaches the fault injection filter and the injected delay. In gRPC, it translates to that for **both unary and streaming RPCs the abort will happen before sending any traffic**.

Also, once an RPC is aborted, the enforcer should push back on following retry attempts.

#### Max active fault

This feature limits the maximum number of concurrent active fault injected RPC for the entire process. Apparently, this requires a global counter (gauge). This value is designed to be updated via multiple channels. So, **it needs to be thread-safe**, but the actual implementation depends on each stack. For example, volatile variable, atomic variable, memory order, locking. Anything works best for the language.

The way to implement the tracking of RPC lifespan also depends on the language. For example, Core & Java may use the deallocation method of the RPC object or enforcer. Go’s Finalizer might not work well enough, so the enforcer (interceptor) needs to update the active fault counter when RPC is finished.

#### Header matching

This doc proposes to support as many metadata as we can (see [detailed definitions](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/fault_filter.html?highlight=fault%20injection#controlling-fault-injection-via-http-headers)):

```bash
# Aborts HTTP requests with status code 503
x-envoy-fault-abort-request=503

# Aborts gRPC requests with INVALID_ARGUMENT
X-envoy-fault-abort-grpc-request=3

# The percentage of requests to be failed. This header only sets numerator. The denominator was set in the HTTPFault filter, which by default is 100.
# 5% of requests should fail.
X-envoy-fault-abort-request-percentage=5

# Delays request for 53 milliseconds.
X-envoy-fault-delay-request=53

# The percentage of requests to be delayed. This header only sets numerator. The denominator was set in the HTTPFault filter, which by default is 100.
# 20% of requests should delay
X-envoy-fault-delay-request-percentage=20

# Not supported
X-envoy-fault-throughput-response=''
X-envoy-fault-throughput-response-percentage=''
```

**Fault inject config in HTTP filter and headers won’t be conflicted**, because the header only being evaluated if the delay or abort is set to `HeaderDelay` or `HeaderAbort` instead of concrete values.

### Status Code Mapping

`grpc_status` is only available in xDS v3 API. In xDS v2 API, no matter TD or Envoy, they all only allow `http_status`. Mapping between HTTP status code and gRPC status code is well defined (see [doc](https://github.com/grpc/grpc/blob/master/doc/http-grpc-status-mapping.md)), but that means users are limited to inject only 7 out of 16 gRPC status codes (including `OK`). The underlying schema for abort looks like [FaultAbort](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/http/fault/v3/fault.proto#extensions-filters-http-fault-v3-faultabort). **This doc proposes to update TD configuration to allow more expressive status code injection, once xDS v3 API is rolled out.**

We still accept fault injection settings with only HTTP status code. **The ConfigSelector should translate the HTTP status code into gRPC status code** before sending the config to the fault injection enforcer.

### No multiple fault injection filter

Multiple fault injection is allowed in Envoy. Fault injection filters are normal HTTP filters in Envoy. All HTTP filters are executed consequently. Confirmed with experiment that 2 fault injection filters work fine. Also, if Envoy specified any Runtime fault injection parameter, it overrides all HTTP fault injection filters and only one fault injection will be working.

In TD and Istio, there could only be one fault injection setting in routing configuration. Each RPC can only have 1 matched routing configuration. The fault injection policy in the routing configuration will be executed. For simplicity, **this doc proposes to only allow one fault injection filter**.

### Evaluate Possibility Fraction

According to the [Envoy implementation](https://source.corp.google.com/piper///depot/google3/third_party/envoy/src/source/extensions/filters/http/fault/fault_filter.cc;l=223), each feature of fault injection is checked with a different random number. Compared to using a single random number for all features, this affects the probability of an RPC being affected by fault injection. For example, the fault injection has 20% chance to delay and 5% chance to abort. Under the single random number solution, only 20% RPC will be affected. But if two features are independent, 24% RPC will be affected. So, **this doc suggests to make each feature independent by using a different random number each time.**

## Alternatives

### Alternative 1: Fault Injection Config via ProtoBuf Metadata

After ConfigSeletor calculated the final fault injection config, it can encode the config proto and set it as metadata for that RPC. This approach is straightforward and allows more cohesive logic in the fault injection enforcer. On the other hand, it requires additional serialization and deserialization of proto messages, and it can reduce debuggability.

```bash
grpc-fault-injection-policy-bin=<...>  # The encoded FI config proto message
```

However, setting additional metadata may leak implementation details onto wire, which is undesired.

### Alternative 2: Fault Injection Config via Multiple Metadata

Fault injection config is part of RDS. However, if we want to support fault injection in non-xDS cases, we have to have a more generic expression of FI config. This approach uses metadata as the final fault injection options for each RPC:

```bash
grpc-fault-injection-delay-ms=1.5       # 1.5ms delay before sending first request
grpc-fault-injection-delay-percent=0.2  # This call has 20% chance to be delayed
grpc-fault-injection-abort-code=7       # Reject the call as PERMISSION_DENIED
grpc-fault-injection-abort-percent=0.5  # This call has 50% chance to be rejected
```

In this way, users can utilize fault injection without configuring xDS. It also allows decouple the enforcement of FI from xDS control path.

But similar to above approach, adding extra metadata is not a good idea.

### Alternative 3: Fault Injection Config via ServiceConfig

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

   // The possibility that this RPC will be delayed.
   //
   // The default value is 0. The possibility is represented as a fraction over
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

   // The possibility that this RPC will be aborted.
   //
   // The default value is 0. The possibility is represented as a fraction over
   // 1 million (1000000). For example, 0.625% will be 6250, 12.5% will be
   // 125000. Its value should be clamped into the interval [0, 1000000].
   uint32 abort_fraction = 4;
 }

 FaultInjectionPolicy fault_injection_policy = 8;
}
```

This approach requires more engineering effort, but provides the fault injection feature to non-xDS gRPC applications.

## Implementation

TBD
