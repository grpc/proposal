A55: Exposing OpenCensus Metrics for gRPC retry
----
* Author(s): dapengzhang0, dfawley, ejona86, markdroth
* Approver: markdroth
* Status: In Review
* Implemented in: C-core, Java
* Last updated: 2021-08-02
* Discussion at: https://groups.google.com/g/grpc-io/c/EAbFqaJWp5w

## Abstract

This document proposes a family of metrics that gRPC retry should expose to OpenCensus and a high-level requirement for the APIs that record the measurements. It can be considered as an augment of the [retry gRFC](https://github.com/grpc/proposal/blob/master/A6-client-retries.md).

## Background

There are a collection of OpenCensus [metrics](https://github.com/census-instrumentation/opencensus-specs/blob/master/stats/gRPC.md#measures) for gRPC. When adding retry support to gRPC, we need to clearly define the measurement points for those metrics and add some retry specific metrics to OpenCensus library to present a clearer picture of retry attempts. The retry gRFC discussed exposing [three additional per-method metrics]((https://github.com/grpc/proposal/blob/master/A6-client-retries.md#retry-and-hedging-statistics)) for retry. However, no detail on how to record and expose these metrics was discussed. Also, these metrics still provide very limited information and the histogram view (the third metric) is not very useful because `maxAttempts` is always less than or equal 5. Besides the number of retry attempts, users may also be interested in the total number of transparent retries and the total amount of delay time caused by retry backoff.

### Related Proposals: 
* [gRPC Retry Design](https://github.com/grpc/proposal/blob/master/A6-client-retries.md)

## Proposal

### High-level Requirement

Per-call stats from the perspective of the application should continue to be gathered from the interceptor/filter level, just as they are today.

For each call, there should be some way to attach an object that will gather data for individual call attempts.  Events on the underlying call attempts will be recorded via that object.

On the parent call object, there should be some way to record an event when we do the first LB pick attempt for a given call attempt.

There should be some way to indicate whether a given call attempt was triggered via a transparent retry.

With that general structure in the gRPC stats interface, it should be possible for whatever plugs into the stats interface (e.g., Census, OpenCensus, etc) to determine the data to export in the metrics defined here.


### Metrics to Expose

gRPC will treat each retry attempt or hedged RPC as a distinct RPC with regards to the current set of [metrics]((https://github.com/census-instrumentation/opencensus-specs/blob/master/stats/gRPC.md#measures)).

We add the following per-overall-client-call metrics to OpenCensus:

#### Measures

| Measure name                                   | Unit | Description                                                                                                                                                    |
| ---------------------------------------------- | ---- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| grpc.io/client/retries\_per\_call              | 1    | Number of retry or hedging attempts excluding transparent retries made during the client call. The original attempt is not counted as a retry/hedging attempt. |
| grpc.io/client/transparent\_retries\_per\_call | 1    | Number of transparent retries made during the client call.                                                                                                     |
| grpc.io/client/retry\_delay\_per\_call         | ms   | Total time of delay while there is no active attempt during the client call.           

#### Views

| View name                                      | Measure                         | Aggregation  | tags                 |
| ---------------------------------------------- | ------------------------------- | ------------ | -------------------- |
| grpc.io/client/retries\_per\_call              | retries\_per\_call              | distribution | grpc\_client\_method |
| grpc.io/client/retries                         | retries\_per\_call              | sum          | grpc\_client\_method |
| grpc.io/client/transparent\_retries\_per\_call | transparent\_retries\_per\_call | distribution | grpc\_client\_method |
| grpc.io/client/transparent\_retries            | transparent\_retries\_per\_call | sum          | grpc\_client\_method |
| grpc.io/client/retry\_delay\_per\_call         | retry\_delay\_per\_call         | distribution | grpc\_client\_method |


## Rationale

Transparent retry should also be treated as a distinct RPC with regards to the per-attempt metrics, because for OpenCensus tracing, it's natural to create a child span for transparent retry as with ordinary retry attempt, so for OpenCensus stats, we should be consistent. Also, there is no effective way to distinguish the two types of transparent retry, and one of them actually sends data to the wire, so the outbound bytes should be recorded.

## Implementation

API plumbing to expose the metrics is needed in each language.

### Java

#### Public API change:

Add the following API to `io.grpc.ClientStreamTracer`

```java
/**
 * The stream is being created on a ready transport.
 *
 * @param headers the mutable initial metadata.
 *     Modifications to it will be sent to the socket but
 *     not be seen by client interceptors and the application.
 */ 
public void streamCreated(Attributes transportAttrs, Metadata headers)
```

Deprecate `io.grpc.ClientStreamTracer.StreamInfo.getTransportAttrs()` and `io.grpc.ClientStreamTracer.StreamInfo.Builder.setTransportAttrs()`

We will no longer call `Factory.newClientStreamTracer(info, headers)` to create a tracer instance after a ready transport for the stream is available. Instead, we create a tracer instance prior to the first time picking a subchannel for the stream, and once we are about to create a real stream on a ready transport we call `ClientStreamTracer.streamCreated(transportAttrs, headers)`.

#### Internal API change:

Change the signature of `ClientTransport.newStream()` by adding tracers argument:

```java
/**
 * @param tracers a non-empty array of tracers. 
 *        The last element in it is reserved to be set by
 *        the load balancer's pick result and otherwise is null.
 */
ClientStream newStream(
      MethodDescriptor<?, ?> method, Metadata headers, CallOptions callOptions,
      ClientStreamTracer[] tracers);
```

For creating a `PendingStream` or `FailingClientStream`, we call this method providing a tracers array of length
`callOptions.getStreamTracerFactories().size() + 1`, with the last element unset/a no-op tracer and other elements created from the `callOptions.getStreamTracerFactories()`. 

For creating a real stream, we create the same tracers array as above and set the last element of the tracers array with a tracer created from LB result, and then call this method.


### C-core

See the [`CallTracer` class](https://github.com/grpc/grpc/blob/master/src/core/lib/channel/call_tracer.h).

### Go

TBD
