A51: Custom Backend Metrics Support
----
* Author(s): [Yifei Zhuang](https://github.com/YifeiZhuang), [Mark Roth](https://github.com/markdroth)
* Approver: Eric Anderson
* Status: Ready for Implementation
* Implemented in: <language, ...>
* Last updated: 2022-02
* Discussion at: https://groups.google.com/g/grpc-io/c/mieHKzTigzg

## Abstract
Support custom metrics injection at a gRPC server, and consumption at a gRPC client. This is useful 
for customers to implement sophisticated client load balancing strategies. 
For example, the client takes into account the
backend server CPU and memory utilization metrics to do its routing. A few pieces are involved here: 

* The API surface exposed to users to enable the metrics injection and consumption system.
* The two backend report collection mechanisms and the gRPC internal plumbing using its underlying infrastructures.
* Metrics, the wire formats, and use cases.


## Background
gRPC currently allows plugging in third-party load balancing policies. 
However, there are cases where it is desirable for a policy to make load balancing decisions based 
on data from the backends, such as the backend CPU utilization or queue size. 
An example of an LB policy that might require that is a weighted round robin (WRR) policy. 
To address this requirement, this design proposes mechanisms for gRPC servers to publish metrics to 
be sent to the client and APIs to allow client-side LB policies to use those metrics.

### Related Proposals:

* [Open Request Cost Aggregation (ORCA)](https://github.com/envoyproxy/envoy/issues/6614)

## Proposal

The [ORCA](https://github.com/envoyproxy/envoy/issues/6614) proposal fits perfectly into the 
metrics reporting solution. We propose to use xDS ORCA service and message standards.

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
Specifically, it will be transmitted as a binary-encoded [xds.data.orca.v3.LoadReport](https://github.com/cncf/xds/blob/eded343319d09f30032952beda9840bbd3dcf7ac/xds/data/orca/v3/orca_load_report.proto#L15) protobuf.

### Per-Request Metrics Reporting

#### Wire Format
For metrics reported on a per-request basis, the ORCA LoadReport protobuf will be encoded in the 
trailing metadata sent by the backend.  The metadata key will be `endpoint-load-metrics-bin`.

#### The Client API
LB policies will have the ability to access the load report data 
as part of intercepting the trailing metadata sent from the server.
Note that the API must be structured such that if multiple LB policies are interested in the data (e.g., both xDS and WRR),
the protobuf deserialization needs to happen only once. The exact API for this may differ across languages, 
but here are a couple of possible approaches:
* The deserialized data can be attached to the metadata element, so that it can be deserialized once and accessed multiple times.
* The deserialized data can be stored in the call data and accessed by the LB policy code when it intercepts the trailing metadata.


#### The Server API 
Using a server interceptor, it is convenient to attach the metrics as part of the
trailing metadata of the server call and send back with the RPC response.

We will define a metric recorder that is unique per RPC, only useful when the server interceptor is
installed, in which case the server applications should obtain a reference and
use it to record metrics specifically for the current RPC call.
Implementing this may leverage a [context](https://grpc.github.io/grpc-java/javadoc/io/grpc/Context.html) or similar mechanism,
and a language can implement one if it does not exist.
The metric recoder will provide these functions, pseudo-code below:
```
// Records a request cost metric measurement for the call.
function recordRequestCostMetric(String name, double value);

// Records a utilization metric measurement for the call.
function recordUtilizationMetric(String name, double value);

// Records the CPU utilization metric measurement for the call.
function recordCPUUtilizationMetric(double value);

// Records the memory utilization metric measurement for the call.
function recordMemoryUtilizationMetric(double value);
```

Recording the same metric multiple times overrides the previously provided values. 
The methods can be called at any time during an RPC lifecycle,
depending on the particular type of the cost or utilization measurement. For example, an RPC request size may
be recorded at the beginning of the call, while a queue-size may be reported at any time during the call lifetime.
The data is dumped and translated to an `OrcaLoadReport` at the end of each RPC by the interceptor, 
and sent out as the trailing metadata.

### Out of Band Metrics Reporting
To periodically receive metrics data from a backend server, the client opens a stream on the 
established connection to the server to request the load report.
The request should specify metrics reporting interval. The interval will be validated by the server, 
see validation details below.
Meanwhile, the server registers an OOB streaming service to emit the metrics.

#### The Server API 
We will provide a builtin implementation of [xds.service.orca.v3.OpenRCAService](https://github.com/cncf/xds/blob/eded343319d09f30032952beda9840bbd3dcf7ac/xds/service/orca/v3/orca.proto#L27)
streaming service defined by ORCA in each supported language.
The implementation service holds the utilization metrics data in key/value pairs using a map.
We will expose set-style APIs for user's server applications to update the metrics data, pseudo-code as follows:

```
// Update the metrics value corresponding to the specified key.
function setUtilizationMetric(String key, double value);

// Replace the whole metrics data using the specified map.
function setAllUtilizationMetrics(Map<String, Double> metrics);

// Remove the metrics data entry corresponding to the specified key.
function deleteUtilizationMetric(String key);

// Update the CPU utilization metrics data.
function setCPUUtilizationMetric(double value);

// Clear the CPU utilization metrics data.
function deleteCPUUtilizationMetric();

// Update the memory utilization metrics data.
function setMemoryUtilizationMetric(double value);

// Clear the memory utilization metrics data.
function deleteMemoryUtilizationMetric();
```

Allowing updating each metric key individually is convenient for a server application to adopt when it 
has multiple isolated components and each generates their own set of metrics.

A minimum reporting interval (30s by default) can be configured when creating the service.
The minimum report interval is the lower bound of the OOB metrics report period. It means, if 
the `report_interval` from the client OOB request message is less than the service configured minimum
interval then it is treated as the service minimum interval. 
If `report_interval` is not specified (equals to 0), it is also treated as service minimum.
There is no upper bound.

Internally, the load report server will maintain a timer for each ORCA stream.
Initially receiving a new request, or when the response timer fires, the server sends a response to 
report metrics data. To generate a response, the service copies the metrics data saved in the map.
The report is always in a state-of-the-world fashion, including all present entries each update.
The service should properly handle exceptions
when the client disconnects: it should terminate the stream and cancel the timer immediately.
Otherwise, the service should keep sending the report.

If no one called to update the metrics data during a report interval, the data remains the same. 
And the load reporting server will keep sending the metrics data regardless of whether the data has changed or not.


#### The LB Policy API 

Out-of-band data will be sent only if the client starts a streaming call for the `OpenRCAService/StreamCoreMetrics` method.
If the backend is designed to provide this information but the client never asks for it, 
it will not be used. Conversely, if the backend does not provide the information but the client 
does start a call asking for it, the backend will terminate the call with status `UNIMPLEMENTED`, 
in which case the client must not retry the call on the same connection.

Any LB policy that is interested in this out-of-band data may subscribe to it.
There will be an API that allows LB policies to subscribe to the out-of-band data. The client will
maintain an open `OpenRCAService/StreamCoreMetrics` call whenever there is at least one LB policy 
subscribing to the data. When a policy subscribes, it will indicate the interval at which it would like to get updates. 
If multiple policies subscribe with different intervals, we will use the minimum of those intervals. 
Note that the ORCA protocol involves the client telling the server the interval at which 
it wants reports when the call starts, so if the client needs to change its requested interval, 
it will need to cancel and restart the call.
The API provided to LB policies needs to allow changing its requested interval at any time;
this is expected to be a fairly rare operation
(usually triggered by a config change for one of the subscribing LB policies).
It is important that a load balancing policy is able to re-config the frequency
and should not require heavy-weight operations like reconnecting to backends or recreating subchannels or LB policies.

Each policy that has subscribed to the data will get a callback whenever a new message is sent to the client on the call.
As with per-request data, the protobuf deserialization will happen only once.
The data provided to the callback should be exposed to the LB policy using the same type as used for per-request data.
Currently, out-of-band reporting only supports utility metrics (see the Rationale section).
The LB policy API implementation does not filter out reqeust cost metrics,
but we reserve the right to do so in the future.
The API for subscribing to out-of-band reports must be available to LB policies at any level of the routing hierarchy, 
including both leaf LB policies (whose children are subchannels) and parent LB policies (whose children are other LB policies).
Note that there may be multiple LB policies in the routing hierarchy that are interested in subscribing.
For example, when using xDS, the `xds_cluster_impl` policy may need to see backend metric data to 
generate load reports for each cluster, and the WRR policy might need to see the same data for 
endpoint-picking within a locality.
Reports are published to all the subscribers as-is with no aggregation; 
each LB policy is responsible for performing whatever aggregation is appropriate for its use.
Note that the API should be designed such that it is available to any LB policy without other
policies having to know or care about out-of-band reporting.
For example, it should be possible for a non-leaf LB policy to subscribe to out-of-band reports from
a child policy like round_robin, which does not know anything about out-of-band-reports and will not contain any
code to handle them.

When a subchannel first establishes a connection, if at least one LB policy is subscribing to out-of-band reporting, 
the client will immediately start the `StreamCoreMetrics` call to the backend. 
The client does not need to wait for this call to be established before it starts sending other calls on the subchannel.
If the `StreamCoreMetrics` call fails with status `UNIMPLEMENTED`, the client will not retry the call, 
and the subscribing LB polic(ies) will not receive any callbacks.
However, the client will record a channel trace event indicating that this has happened. 
It will also log a message at priority ERROR.
If the `StreamCoreMetrics` call returns any other status, the client will retry the call.
To avoid hammering a server that may be experiencing problems, the client will use exponential backoff between attempts.
When the client receives a message from the server on a given call, the backoff state is reset, 
so the next attempt will occur immediately, but any subsequent attempt will be subject to exponential backoff.

When the client subchannel is shutting down or when the backend sends a `GOAWAY`, 
the client will cancel the `StreamCoreMetrics` call.
There is no need to wait for the final status from the server in this case.

## Rationale
For per-query, we support both utilization and request cost metrics reporting. 
For OOB, we only support utilization metrics, not request cost metrics. That is because to 
report request cost via OOB, at the server side, we need to identify which client the request cost 
reports is generated from and associate it with the corresponding OOB stream in order to deliver the
report correctly, plus at the client side, it is likely to cause ordering problems
(there is no guarantee of the order in which the callbacks would be seen by the LB policy)
and may incur additional synchronization overhead.
We are not going to support this particular scenario because there is no known use cases as well as 
the technical difficulty.

In the per-query metrics reporting, if the backend application adds data to the trailing metadata of each RPC,
it will be sent regardless of whether the client actually needs it. 
This could conceivably lead to wasted bandwidth, CPU, and memory if the backend is sending data
that the client does not need, but this case should be fairly rare;
backends will generally not be written to populate data that they do not need.
Also, we want to avoid the complexity of introducing some kind of negotiation for the client to
tell the backend that it wants this data.

Note that reporting utilization metrics per-query is only useful in high QPS systems.
In such systems, the application that measures utilization data should consider sampling the data
to avoid performance degradation due to high rates of syscalls.

For the server APIs in the OOB reporting, alternative to the set-style APIs in a push model,
we can use callback-style APIs in a pull model to update the metrics data,
however, this is suboptimal because it is not performance/scalability friendly.
In a pull model, the time complexity of doing utilization measurement syscalls reaches O(M*N),
M being the number of OOB clients and N being their frequency configuration.
In a push model, the number of syscalls decouples with the number of OOB clients and their configurations.
In fact, the server application is able to tune the performance by sampling the utilization data at a constant rate.

It is by design that the OOB server sends the data regardless of whether it has been changed or 
not since the last time it was sent. We considered a potential optimization, that is to skip 
sending the unchanged metrics data since last response.
To do that, we should not compare maps for de-duplication, instead, we can bump up a
generation_id of the metrics data every time users make a change.
Each client connection only needs to compare its previous generation_id to do deduplication.
The optimization strategy can be implemented in the future if there are concrete use cases.

In addition to the minimum report interval configuration at the OOB server, it might be useful
in the future to also let the users specify the default report interval separately.
The reporting frequency in the OOB request is final because it is a server-streaming API.
Clients would terminate the OOB stream and reconnect if they want to change the frequency.
Switching to a bidi stream will make it possible to change the reporting frequency in flight,
but this can be done in the future if needed without impacting the public interface.

## Implementation
This will be implemented in Java, Go and C++.

### Java
#### Per-Query Reporting 

At the server side, Java will provide a per-RPC metrics recorder for the user applications to record both utilization and
request cost metrics. They can obtain a reference to the `CallMetricRecorder` using the static 
`getCurrent()` method. All the methods are thread safe.

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

At the client side, Java will provide a utility function `OrcaClientStreamTracerFactory`. 
Users can include it in the picker result in their custom LB policy’s 
`LoadBalancer.SubchannelPicker` to let gRPC push notifications to the application.
It requires users to implement `OrcaPerRequestReportListener` in order to receive the metrics report callback.
To avoid double-parsing trailers, the utility will use a CallOption in StreamInfo.
If the call option is not already present, the utility will create an object to parse the trailer
and notify listeners. The call option is then added and references that created object. If the call
option is already present, the listener will be added to the list of listeners to be notified.

```Java
class CustomPicker extends SubchannelPicker {
  public PickResult pickSubchannel(PickSubchannelArgs args) {
  Subchannel subchannel = … // WRR picking logic
  return PickResult.withSubchannel(
    subchannel,
    OrcaPerRequestUtil.newOrcaClientStreamTracerFactory(listener));
  }
}
```

#### Out-of-Band Reporting 
At the server side, Java implemented the OOB service `OrcaServiceImpl`: users can optionally provide the minimum
reporting interval configuration when creating the object. Then they can update utilization metrics
data using the exposed APIs as follows. All the methods are thread safe.

```java
public class MetricRecorder {
  // Update the metrics value corresponding to the specified key.
  public void setUtilizationMetric(String key, double value);

  // Replace the whole metrics data using the specified map.
  public void setAllUtilizationMetrics(Map<String, Double> metrics);

  // Remove the metrics data entry corresponding to the specified key.
  public void deleteUtilizationMetric(String key);

  // Update the CPU utilization metrics data.
  public void setCPUUtilizationMetric(double value);

  // Clear the CPU utilization metrics data.
  public void deleteCPUUtilizationMetric();

  // Update the memory utilization metrics data.
  public void setMemoryUtilizationMetric(double value);

  // Clear the memory utilization metrics data.
  public void deleteMemoryUtilizationMetric();
}
```

At the client side, Java will provide a utility function to wrap the `LoadBalancer.Helper`,
usable by both WRR policy and xDS LB policies. The `LoadBalancer.Helper` manages the OOB stream
lifecycle and accepts subscriptions from any routing hierarchy.
The OOB stream will also gracefully disable itself if the server integration is not complete, 
so users do not have to worry about client and server deployment coordination.

The following pseudocode shows how a custom load balancing policy instantiates the helper wrapper, 
installs the out-of-band metrics reporting mechanism, and then consumes the metrics reports, 
which only requires users to register an `OrcaOobReportListener` corresponding to the subchannel.

```java
class CustomLoadBalancer extends LoadBalancer { 
  private final Helper orcaHelper;
  
  public CustomLoadBalancer(Helper helper) {
    orcaHelper = OrcaOobUtil.newOrcaHelper(helper);
  } 
  
  private Subchannel handleNewServer(ResolvedAddresses address) {
    Subchannel subchannel = orcaHelper.createSubchannel(
        CreateSubchannelArgs.newBuilder().setAddresses(address.getAddresses()).build());
    // Create a separate listener for each Subchannel.
    OrcaOobReportListener listener = new OrcaOobReportListenerImpl(subchannel);
    OrcaOobUtil.setListener(subchannel, listener, 
            OrcaReportingConfig.newBuilder().setInterval(40, SECONDS).build());
    return subchannel;
  }
}

// Users implement a listener and pass it to the client library to receive
// callbacks upon metric report updates from the backend.
public interface OrcaOobReportListener {
  //receive load report in the format of ORCA protocol.
  void onLoadReport(OrcaLoadReport report);
}

```

### C++

C-core's LB policies do have the ability to obtain backend metric data
for both per-RPC and OOB mechanisms.  However, the C-core LB policy API
is not yet ready to be made public, so this gRFC does not describe those
APIs.  They will be described in a subsequent gRFC when the LB policy
API is ready to be made public.

This section describes only the C++ server-side APIs.

#### Per-Query Reporting APIs

TODO(nicolasnoble): Fill this in.

#### OOB Reporting APIs

C++ provides the following API for running the ORCA service:

```c++
class OrcaService {
 public:
  struct Options {
    // Minimum report interval.  If a client requests an interval lower
    // than this value, this value will be used instead.
    absl::Duration min_report_duration = absl::Seconds(30);

    Options() = default;
    Options& set_min_report_duration(absl::Duration duration) {
      min_report_duration = duration;
      return *this;
    }
  };

  explicit OrcaService(Options options);

  // Sets or removes the CPU utilization value to be reported to clients.
  void SetCpuUtilization(double cpu_utilization);
  void DeleteCpuUtilization();

  // Sets of removes the memory utilization value to be reported to clients.
  void SetMemoryUtilization(double memory_utilization);
  void DeleteMemoryUtilization();

  // Sets or removes named utilization values to be reported to clients.
  void SetNamedUtilization(std::string name, double utilization);
  void DeleteNamedUtilization(const std::string& name);
  void SetAllNamedUtilization(std::map<std::string, double> named_utilization);
};
```

To use this, an application can simply register this service via their
`ServerBuilder`, like so:

```c++
OrcaService orca_service(OrcaService::Options());
ServerBuilder server_builder;
server_builder.AddListeningPort(...);
server_builder.RegisterService(&orca_service);
auto server = server_builder.BuildAndStart();
```

The application can set utilization data on the `OrcaService` object at
any time, and it will automatically be sent to any connected clients at
the next scheduled reporting interval.
