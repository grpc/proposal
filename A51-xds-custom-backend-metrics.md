A51: xDS Custom Backend Metrics Support
----
* Author(s): [Yifei Zhuang](https://github.com/YifeiZhuang), [Mark Roth](https://github.com/markdroth)
* Approver: Eric Anderson
* Status: Ready for Implementation
* Implemented in: <language, ...>
* Last updated: 2022-02
* Discussion at: https://groups.google.com/g/grpc-io/

## Abstract
Support custom metrics injection at a gRPC server, and consumption at a gRPC client. This is useful 
for customers to implement sophisticated client load balancing strategies. 
For example, the client takes into account the
backend server CPU and memory utilization metrics to do its routing. A few pieces are involved here: 

* The API surface exposed to users to enable the metrics injection and consumption system.
* The two backend report collection mechanisms and the gRPC internal plumbing using its underlying infrastructures.
* Metrics, the wire formats, and use cases.


## Background

Previously xDS load balancer has to be configured to one of the built-in policies: e.g. round-robin or ring hash.
The [custom load balancer proposal](link place holder) makes it possible to plug in users' own LB policy. 
However, customers' own LB policy typically depends on extra input, e.g. CPU utilization, backend queue size.
In fact, many fine-tuned LB policies requires feedback signals from the
backend to the data plane clients, e.g. weighted-round-robin LB policy.
To fill the gap and complements custom LB policy, gRPC now brings APIs to allow users to inject custom 
metrics at the server and consume the metrics from the client.
The [ORCA](https://github.com/envoyproxy/envoy/issues/6614) proposal fits perfectly into the solution. 
We propose to use ORCA service and message standards defined in UDPA protos.

### Related Proposals:

* [A36: xDS-Enabled Servers][A36]
* [Open Request Cost Aggregation (ORCA)](https://github.com/envoyproxy/envoy/issues/6614)
* [xDS custom LB policy](link place holder)

[A36]: A36-xds-for-servers.md

## Proposal

Two metrics reporting mechanisms are supported:

* Per-query metrics reporting: the backend server attaches the injected custom metrics in the trailing metadata
  when the corresponding RPC finishes. This is typically useful for short RPCs like unary calls.
* Out-of-band metrics reporting: the backend server periodically pushes metrics data,
  e.g. cpu and memory utilization, to the client. This is useful for all situations: unary calls,
  long RPCs in streaming calls, or no RPCs. 

Distinguish two types of metrics:
* Request cost metrics show the amount of resources it takes to handle a request, specific for each RPC.
* Utilization metrics show the server status machine wide, independent of any RPC.

### Wire Format

Backend metric data will be encoded in the ORCA format for transmission on the wire. 
Specifically, it will be transmitted as a binary-encoded [LoadReport](https://github.com/cncf/udpa/blob/04548b0d99d4e70b29310ebccc8e01f2deeed43a/udpa/data/orca/v1/orca_load_report.proto#L14) protobuf.

### Per-Request Metrics Reporting

#### Wire Format
For metrics reported on a per-request basis, the ORCA LoadReport protobuf will be encoded in the 
trailing metadata sent by the backend.  The metadata key will be `endpoint-load-metrics-bin`.

Note that if the backend application adds this data to the trailing metadata of each RPC, 
it will be sent regardless of whether the client actually needs it.  
This could conceivably lead to wasted bandwidth, CPU, and memory if the backend is sending data 
that the client does not need, but this case should be fairly rare; 
backends will generally not be written to populate data that they do not need.  
Also, we want to avoid the complexity of introducing some kind of negotiation for the client to 
tell the backend that it wants this data.

#### The Client API
LB policies will have the ability to access the load report data 
as part of intercepting the trailing metadata sent from the server.
Note that the API must be structured such that if multiple LB policies are interested in the data (e.g., both xDS and WRR),
the protobuf deserialization needs to happen only once. The exact API for this may differ across languages, 
but here are a couple of possible approaches:
* The deserialized data can be attached to the metadata element, so that it can be deserialized once and accessed multiple times.
* The deserialized data can be stored in the call data and accessed by the LB policy code when it intercepts the trailing metadata.

In addition, we also discussed the possibility of having the LB policy get a separate callback for the load report data, 
distinct from the callback it gets to intercept trailing metadata.  
However, we think this is not a good idea, since it is likely to cause ordering problems
(there is no guarantee of the order in which the callbacks would be seen by the LB policy) 
and may incur additional synchronization overhead.

#### The Server API 
Using a server interceptor, it is convenient to attach the metrics as part of the
trailing metadata of the server call and send back with the RPC response.

Define a metric recorder that is unique per RPC. Server applications should obtain a reference and 
use it to record metrics specifically for the current RPC call. Implementing this may leverage a 
[context](https://grpc.github.io/grpc-java/javadoc/io/grpc/Context.html) or similar mechanism, 
and a language should implement one if not already existed.
The metric recoder looks like this:

```Java
public final class CallMetricRecorder {
  // Records a request cost metric measurement for the call.
  public CallMetricRecorder recordRequestCostMetric(String name, double value);

  // Records a utilization metric measurement for the call.
  public CallMetricRecorder recordUtilizationMetric(String name, double value);

  //Records the CPU utilization metric measurement for the call. 
  public CallMetricRecorder recordCPUUtilizationMetric(double value);

  // Records the memory utilization metric measurement for the call. 
  public CallMetricRecorder recordMemoryUtilizationMetric(double value);

  // Returns the call metric recorder attached to the current call.
  public static CallMetricRecorder getCurrent();
}
```

All the methods should be thread safe.
Recording one metrics multiple times overrides the previously provided values. 
The methods can be called at any time during an RPC lifecycle,
depending on the particular type of the cost or utilization measurement. For example, an RPC request size may
be recorded at the beginning of the call, while a queue-size may be reported at any time during the call life-cycle.
The data is dumped and translated to `OrcaLoadReport` at the end of each RPC by the interceptor, 
and sent out as the trailing metadata.

Note that reporting utilization metrics per-query is only useful in high QPS systems. 
In such systems, the application that measures utilization data should consider sampling the data 
to avoid performance degradation due to high rates of syscalls.


### Out of Band Metrics Reporting
For the out-of-band reporting, it requires a separate load reporting server running along with the 
backend server for as long as the server is alive.

At the client side, typically an LB policy, opens a stream on each
established connection to the backend server to request the load report, and then periodically receives 
metrics data from each backend streaming server.
The report request should specify metrics reporting interval, the lower bound of which should be capped 
to a server configurable value, and there is no upper bound unless it is needed in the future.

#### The Server API 
Provide builtin implementation of `OpenRCAService` streaming service defined by ORCA in each supported language.
A user can add such a load reporting server in their server applications
in a normal way that they register a service.

The `OpenRCAService` holds the utilization metrics data in key/value pairs using a hashmap. 
Expose set-style APIs for user's server applications to update the metrics data, pseudocode below:

```java
class OrcaServiceImpl implements OpenRcaService {

// Allow configuring minimum report interval. If not configured or badly configured non-positive, 
// the default is 30s.
// In the future, it might be useful to let users specify minimum report interval and default 
// report interval separately. If the default report interval is not configured or badly configured 
// non-positive, the default is 1 min. 
// public OpenRcaServiceImpl(long minInterval, TimeUnit timeUnit, long 
// defaultInterval, TimeUnit)
public OrcaServiceImpl(long minReportInterval, TimeUnit unit);

// Update the metrics value corresponding to the specified key.
void setUtilizationMetric(String key, double value);

// Replace the whole metrics data using the specified map.
void setAllUtilizationMetrics(Map<String, Double> metrics);

// Remove the metrics data entry corresponding to the specified key.
void deleteUtilizationMetric(String key);

// Update the CPU utilization metrics data.
void setCPUUtilizationMetric(double value);

// Clear the CPU utilization metrics data.
void deleteCPUUtilizationMetric();

// Update the memory utilization metrics data.
void setMemoryUtilizationMetric(double value);

// Clear the memory utilization metrics data.
void deleteMemoryUtilizationMetric();
}
```

All the methods above should be thread safe. 
Allowing updating each metric key individually is convenient for a server application to adopt when it 
has multiple isolated components and each generates their own set of metrics.

A minimum reporting interval (30s by default) could be configured when creating the service.
The minimum report interval is the lower bound of the OOB metrics report period. It means, in the 
client OOB request message if the `report_interval` 
goes below the service configured minimum interval then it is treated as the service minimum interval. 
If `report_interval` is not specified (equal to 0), it is also treated as service minimum.
There is no upper bound, the largest value you can specify is max integer.
However, it might be useful to have a separate option called default report interval (default 1 minute) 
that will be used when `report_interval` is not specified (equal to 0), see the code comment above.

Alternatively we can use callback-style APIs in a pull model to update the metrics data, 
however, this is suboptimal because it is not performance/scalability friendly.
In a pull model, the time complexity of doing utilization measurement syscalls reaches O(M*N),
M being the number of OOB clients and N being their frequency configuration.
In a push model, the number of syscalls decouples with the number of OOB clients and their configurations.
In fact, the server application is able to tune the performance by sampling the utilization data at a constant rate.

Internally, the load report server will save each incoming client connection(`responseObserver`) 
in a `RcaConnection` object or alike, with a custom frequency timer.

```java
class RcaConnection {
  StreamObserver<OrcaLoadReport> responseObserver;
  Timer customFrequencyTimer;
}

```

Initially receiving a new request, or when the response timer fires, the server sends a response to 
report metrics data. To generate a response, we copy the metrics data saved in the map, 
but the response proto should only be produced once and be reused.  
The report is always in a state-of-the-world fashion. The service should properly handle exceptions 
when the client disconnects: it should terminate the stream, cancel the timer and remove the 
`RcaConnection` immediately. Otherwise, the service should keep sending the report.
Note that the frequency is final because the OOB request method is a server-streaming API. 
Clients would terminate the OOB stream and reconnect if they want to change the frequency. 
Switching to a bidi stream will make it possible to change the reporting frequency in flight, 
but it should not impact the public interface.

If no one called to update the metrics data during a report interval, the data remains the same. 
And the load reporting server will keep sending the metrics data regardless of whether the data has changed or not.
There is a potential optimization, that is to skip sending the unchanged metrics data since last response. 
To do that, we should not compare maps for de-duplication, instead, we can bump up a 
generation_id of the metrics data every time users make a change. 
Each client connection only needs to compare its previous generation_id to do deduplication. 
The optimization strategy can be implemented until there are concrete use cases in the future.

#### The Client API 

Out-of-band data will be sent only if the client starts a streaming call for the `OpenRCAService/StreamCoreMetrics` method.
If the backend is designed to provide this information but the client never asks for it, it will not be used.  
Conversely, if the backend does not provide the information but the client does start a call asking for it,
the backend will terminate the call with status `UNIMPLEMENTED`, in which case the client will not retry the call.

Any LB policy that is interested in this out-of-band data may subscribe to it.  
The client will maintain an open `OpenRCAService/StreamCoreMetrics` call 
whenever there is at least one LB policy subscribing to the data.

When a policy subscribes, it will indicate the interval at which it would like to get updates. 
If multiple policies subscribe with different intervals, we will use the minimum of those intervals.  
Note that the ORCA protocol involves the client telling the server the interval at which 
it wants reports when the call starts, so if the client needs to change its requested interval, 
it will need to cancel and restart the call.
A client may change its requested interval at any time; this is expected to be a fairly rare operation 
(usually triggered by a config change for one of the subscribing LB policies),
It is important that a load balancing policy is able to re-config the frequency
and should not require heavy-weight operations like reconnecting to backends or recreating subchannels or LB policies.

Each policy that has subscribed to the data will get a callback whenever a new message is sent to the client on the call.  
As with per-request data, the protobuf deserialization will happen only once.  
The data provided to the callback should be exposed to the LB policy using the same type as used for per-request data.

The API for subscribing to out-of-band reports must be available to LB policies at any level of the routing hierarchy, 
including both leaf LB policies (whose children are subchannels) and parent LB policies (whose children are other LB policies).  
Note that there may be multiple LB policies in the routing hierarchy that are interested in subscribing.  
For example, in the case where we are using the xDS policy for cluster routing and the WRR policy for intra-cluster routing,
both policies may be interested in subscribing.  
The WRR policy will be directly managing the subchannels and will therefore be able to subscribe directly against the subchannel API, 
whereas the xDS policy will need to subscribe against the LB policy API of the WRR policy, 
which will need to forward reports from the subchannels to its external subscriber(s) (i.e., the xDS policy).  
Reports are forwarded as-is with no aggregation; each LB policy is responsible for performing whatever aggregation is appropriate for its use.

Note that the LB policy API should be designed such that individual LB policies that do not know or 
care about out-of-band reporting should not have to do anything special to support them.  
For example, it should be possible for the xDS policy to subscribe to out-of-band reports from 
a child policy like RR, which does not know anything about out-of-band-reports and will not contain any
code to handle them.

When a subchannel first establishes a connection, if at least one LB policy is subscribing to out-of-band reporting, 
the client will immediately start the StreamCoreMetrics call to the backend.  
The client does not need to wait for this call to be established before it starts sending other calls on the subchannel.

If the StreamCoreMetrics call fails with status `UNIMPLEMENTED`, the client will not retry the call, 
and the subscribing LB polic(ies) will not receive any callbacks.  
However, the client will record a channel trace event indicating that this has happened.  
It will also log a message at priority ERROR.

If the StreamCoreMetrics call returns any other status, the client will retry the call.  
To avoid hammering a server that may be experiencing problems, the client will use exponential backoff between attempts.  
When the client receives a message from the server on a given call, the backoff state is reset, 
so the next attempt will occur immediately, but any subsequent attempt will be subject to exponential backoff.

When the client subchannel is shutting down or when the backend sends a GOAWAY, 
the client will cancel the StreamCoreMetrics call.  
There is no need to wait for the final status from the server in this case.

## Use Cases

Enumerating specific metrics types and their reporting mechanisms, we have:

1. Collect Utilization Metrics using Out-of-Band Reporting

It is popular for a gRPC powered environment to be running its own intra-locality WRR LB policy plugin 
at the client side, which will need to access the backend utilization metrics feedback that is further 
translated to subchannel weights for the scheduling algorithm to function, 
e.g. CPU, queue depth. Observing these metrics is fulfilled by the gRPC custom backend metrics feature.

2. Collect Utilization Metrics using Per-Query Reporting 

In a high QPS environment, reporting request cost using per-query is desired. 
Compared with 1, the advantage of per-query is that the utilization metrics can be timely aligned with the requests. 
It also communicates the metrics data rapidly and saves cost of the idle connection compared with OOB reporting.

3. Collect Request Cost Metrics in Per-Query Reporting

Measuring the query cost from the client perspective and aggregate at a global load balancer is 
another proven traffic distribution strategy. 
A global LB can balance live user traffic between clusters, so that it can match user demand to 
available service capacity, and handle service failures in a fashion that is transparent to the users. 
gRPC’s native solution of per-query metrics reporting supports this use case.

4. Collect Request Cost Metrics in Out-of-Band Reporting 

No use case.  it is also technically more complicated than per-query reporting of request costs, 
because in OOB we need to identify which client the request cost reports is generated from and associate 
it with the corresponding OOB stream in order to deliver the report accurately. 
So generating the request cost report requires extra processing for each distinct client connection in the OOB scenario.
This scenario is not supported until there is use case in the future.

## Rationale

## Implementation

This will be implemented simultaneously in Java by @YifeiZhuang, in
Go by @, and C++ by @.

### Java

For per-query reporting, in the client side, Java provides a utility function `OrcaClientStreamTracerFactory`. 
Users can include it in the picker result in their custom LB policy’s 
`LoadBalancer.SubchannelPicker` to let gRPC push notifications to the application. 
It requires `OrcaPerRequestReportListener` implementation in order to receive the metrics report callback.


```Java
class CustomPicker extends SubchannelPicker {
  public PickResult pickSubchannel(PickSubchannelArgs args) {
  Subchannel subchannel = … // WRR picking logic
  return PickResult.withSubchannel(
    subchannel,
    OrcaUtil.newOrcaClientStreamTracerFactory(listener));
  }
}
```

For OOB reporting, Java provides a convenient function `OrcaHelperWrapper` that does the heavy 
lifting of managing the out-of-band stream life cycle. 
The OOB stream will also gracefully disable itself if the server integration is not complete, 
so users do not have to worry about client and server deployment coordination.

The following pseudocode shows how a custom load balancing policy instantiates the helper wrapper, 
installs the out-of-band metrics reporting mechanism, and then consumes the metrics reports, 
which only requires users to register an `OrcaOobReportListener` that contains their own business logic:

```java
class CustomLoadBalancer extends LoadBalancer {
  private final Helper helper;  // the original Helper
  void handleNewServer(address) {
  // Create OrcaHelperWrapper and a separate listener for each Subchannel
  OrcaListener listener = new OrcaOobReportListener();
  OrcaHelperWrapper orcaWrapper = OrcaUtil.newOrcaHelperWrapper(helper, listener);
  orcaWrapper.configure(
      OrcaReportingConfig.newBuilder().setInterval(40, SECONDS).build()); 
  Subchannel subchannel = orcaWrapper.asHelper().create(
  CreateSubchannelArgs.newBuilder().setAddresses(address).build());
  }
}

// Users implement a listener and pass it to the client library to receive
// callbacks upon metric report updates from the backend.
public interface OrcaOobReportListener {
  //receive load report in the format of ORCA protocol.
  void onLoadReport(OrcaLoadReport report);
}

```