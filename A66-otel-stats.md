# OpenTelemetry Metrics

*   Author: Yash Tibrewal (@yashykt)
*   Approver: Mark Roth (@markdroth)
*   Status: In Review
*   Implemented in: <language, ...>
*   Last updated: Jul 20, 2023
*   Discussion at: https://groups.google.com/g/grpc-io/c/po-deqYEQzE

## Abstract

Propose a metrics data model for gRPC OpenTelemetry metrics.

## Background

There are a collection of
[metrics](https://github.com/census-instrumentation/opencensus-specs/blob/master/stats/gRPC.md)
proposed by OpenCensus for gRPC. OpenCensus is no longer being actively
maintained and is being
[deprecated](https://opentelemetry.io/blog/2023/sunsetting-opencensus/#:~:text=Compatibility%20specification%204.-,What%20to%20Expect%20After%20July%2031st%2C%202023,found%20will%20not%20be%20patched.),
with OpenTelemetry suggested as the successor framework.

### Related Proposals:

*   [A45: Exposing OpenCensus Metrics and Tracing for gRPC retry](A45-retry-stats.md)

## Proposal

### Metrics Schema

#### Units

Following the
[OpenTelemetry Metrics Semantic Conventions](https://opentelemetry.io/docs/specs/otel/metrics/semantic_conventions/),
the following units are used -

*   Latencies are measured in float64 seconds, `s`
*   Sizes are measured in bytes, `By`
*   Counts for number of calls are measured in `{call}`
*   Counts for number of attempts are measured in `{attempt}`

Buckets for histograms in default views should be as follows -

*   Latency : 0, 0.00001, 0.00005, 0.0001, 0.0003, 0.0006, 0.0008, 0.001, 0.002,
    0.003, 0.004, 0.005, 0.006, 0.008, 0.01, 0.013, 0.016, 0.02, 0.025, 0.03,
    0.04, 0.05, 0.065, 0.08, 0.1, 0.13, 0.16, 0.2, 0.25, 0.3, 0.4, 0.5, 0.65,
    0.8, 1, 2, 5, 10, 20, 50, 100
*   Size : 0, 1024, 2048, 4096, 16384, 65536, 262144, 1048576, 4194304,
    16777216, 67108864, 268435456, 1073741824, 4294967296
*   Count : 0, 1, 2, 4, 8, 16, 32, 64, 128, 256, 512, 1024, 2048, 4096, 8192,
    16384, 32768, 65536

These buckets were chosen to maintain compatibility with the gRPC OpenCensus
spec. The OpenTelemetry API has added an experimental feature for
[advice](https://opentelemetry.io/docs/specs/otel/metrics/api/#instrument-advice)
that would allow the gRPC library to provide these buckets as a hint. Since this
is still an experimental feature and not yet implemented in all languages, it is
upto the user to choose the right bucket boundaries.

Also note that, as per an
[OpenTelemetry proposal on stability](https://docs.google.com/document/d/1Nvcf1wio7nDUVcrXxVUN_f8MNmcs0OzVAZLvlth1lYY/edit#heading=h.dy1cg9doaq26)
though, changes to bucket boundaries might not be considered a breaking change.
Depending on the proposal, this recommendation would change to use
`ExponentialHistogram`s instead, which would allow for automatic adjustments of
the scale to better fit the data.

#### Attributes

*   `grpc.method` : Full gRPC method name, including package, service and
    method, e.g. "google.bigtable.v2.Bigtable/CheckAndMutateRow". Note that some
    gRPC implementations allow server to handle generic method names, i.e., not
    registering method names in advance with the server. This allows clients to
    send arbitrary method names that could potentially open up the server to
    malicious attacks that result in metrics being stored with a high
    cardinality. To prevent this, unregistered/generic method names should by
    default be reported with "generic" value instead. Implementations can
    provide the option to override this behavior to allow recording generic
    method names as well.
*   `grpc.status` : gRPC server status code received, e.g. "OK", "CANCELLED",
    "DEADLINE_EXCEEDED"
*   `grpc.target` : Target URI used when creating gRPC Channel, e.g.
    "dns:///pubsub.googleapis.com:443", "xds:///helloworld-gke:8000"

#### Client Per-Attempt Instruments

*   **grpc.client.attempt.started** <br>
    The total number of RPC attempts started, including those that have not completed. <br>
    *Attributes*: grpc.method, grpc.target <br>
    *Type*: Counter <br>
    *Unit*: {attempt} <br>
*   **grpc.client.attempt.duration** <br>
    End-to-end time taken to complete an RPC attempt including the time it takes to pick a subchannel. <br>
    *Attributes*: grpc.method, grpc.target, grpc.status <br>
    *Type*: Histogram (Latency Buckets) <br>
*   **grpc.client.attempt.sent_total_compressed_message_size** <br>
    Total bytes (compressed but not encrypted) sent across all request messages (metadata excluded) per RPC attempt; does not include grpc or transport framing bytes. <br>
    Attributes: grpc.method, grpc.target, grpc.status <br>
    Type: Histogram (Size Buckets) <br>
*   **grpc.client.attempt.rcvd_total_compressed_message_size** <br>
    Total bytes (compressed but not encrypted) received across all response messages (metadata excluded) per RPC attempt; does not include grpc or transport framing bytes. <br>
    *Attributes*: grpc.method, grpc.target, grpc.status <br>
    *Type*: Histogram (Size Buckets) <br>

#### Client Per-Call Instruments

*   **grpc.client.call.duration** <br>
    This metric aims to measure the end-to-end time the gRPC library takes to complete an RPC from the application’s perspective. <br>
    Start timestamp - After the client application starts the RPC. <br>
    End timestamp - Before the status of the RPC is delivered to the application. <br>
    If the implementation uses an interceptor then the exact start and end timestamps would depend on the ordering of the interceptors. Non-interceptor implementations should record the timestamps as close as possible to the top of the gRPC stack, i.e., payload serialization should be included in the measurement. <br>
    *Attributes*: grpc.method, grpc.target, grpc.status <br>
    *Type*: Histogram (Latency Buckets) <br>

#### Server Instruments

*   **grpc.server.call.started** <br>
    The total number of RPCs started, including those that have not completed. <br>
    *Attributes*: grpc.method, grpc.authority <br>
    *Type*: counter <br>
    *Unit*: {call} <br>
*   **grpc.server.call.sent_total_compressed_message_size** <br>
    Total bytes (compressed but not encrypted) sent across all response messages (metadata excluded) per RPC; does not include grpc or transport framing bytes. <br>
    *Attributes*: grpc.method, grpc.authority, grpc.status <br>
    *Type*: Histogram (Size Buckets) <br>
*   **grpc.server.call.rcvd_total_compressed_message_size** <br>
    Total bytes (compressed but not encrypted) received across all request messages (metadata excluded) per RPC; does not include grpc or transport framing bytes. <br>
    *Attributes*: grpc.method, grpc.authority, grpc.status <br>
    *Type*: Histogram (Size Buckets) <br>
*   **grpc.server.call.duration** <br>
    This metric aims to measure the end2end time an RPC takes from the server transport’s (HTTP2/ inproc / cronet) perspective. <br>
    Start timestamp - After the transport knows that it's got a new stream. For HTTP2, this would be after the first header frame for the stream has been received and decoded. Whether the timestamp is recorded before or after HPACK is left to the implementation. <br>
    End timestamp - Ends at the first point where the transport considers the stream done. For HTTP2, this would be when scheduling a trailing header with END_STREAM to be written, or RST_STREAM, or a connection abort. Note that this wouldn’t necessarily mean that the bytes have also been immediately scheduled to be written by TCP. <br>
    *Attributes*: grpc.method, grpc.authority, grpc.status <br>
    *Type*: Histogram (Latency Buckets) <br>

## OpenTelemetry Plugin Architecture

This section describes a `CallTracer` approach to collect the client and server
per-attempt/call metrics. A CallTracer is a class that is instantiated for every
call. This class has various methods that are invoked during the lifetime of the
call. On the client side, the CallTracer knows about multiple attempts on the
same call, and creates a `CallAttemptTracer` object for each attempt, and the
`CallAttemptTracer` gets invoked during the lifetime of the attempt.

The OTel plugin will basically be a way of configuring CallTracer factories on
gRPC clients and servers.

Implementations should allow multiple call/attempt tracers to be registered to a
single call since there could be multiple plugins registered. For example, there
could be an OpenCensus and an OpenTelemetry stats plugin registered together. It
should also allow multiple OpenTelemetry plugins to be registered providing the
ability to configure the different plugins with different MeterProviders.

A sample implementation of this approach is available in
[gRPC Core](https://github.com/grpc/grpc/blob/v1.57.x/src/core/lib/channel/call_tracer.h).

## Language-Specific Details

Each language implementation will provide an API for registering an
OpenTelemetry plugin. Overall, the APIs should have the following capabilities -

*   Allow installing multiple OpenTelemetry plugins.
*   Allow setting a
    [MeterProvider](https://opentelemetry.io/docs/specs/otel/metrics/api/#meterprovider)
    on individual plugins. Implementations should require a MeterProvider being
    set. A MeterProvider not being set should either not be allowed, fail
    registering of the plugin or result in a no-op. Some OpenTelemetry language
    APIs have a global MeterProvider. gRPC implementations should *NOT* fallback
    on this global.
*   Optionally allow enabling/disabling metrics. This would allow optimizations
    to avoid computation and collection of expensive stats within the gRPC
    library. Note that even without this capability, users of OpenTelemetry
    would be able to customize
    [views](https://opentelemetry.io/docs/specs/otel/metrics/sdk/#view) through
    the MeterProvider.
*   Optionally allow setting of a OpenTelemetry plugin for a specific channel or
    server, instead of setting it globally.

Note that implementations of the gRPC OpenTelemetry plugin
[should prefer](https://opentelemetry.io/docs/specs/otel/overview/) to only
depend on the OpenTelemetry API and not the OpenTelemetry SDK.

The [Meter](https://opentelemetry.io/docs/specs/otel/metrics/api/#get-a-meter)
creation should use a `name` that identifies the library, for example,
"grpc-c++", "grpc-java", "grpc-go". The `version` should be the same as the
release version of the gRPC library, for example, "1.57.1".

### C++

```c++
Class OpenTelemetryPluginBuilder {
 public:
  // Enables base set of metrics by default
  OpenTelemetryPluginBuilder(shared_ptr<opentelemetry::metrics::MeterProvider> meter_provider) = default;
  // Enable metric \a metric_name.
  OpenTelemetryPluginBuilder&  EnableMetric(absl::string_view metric_name);
  // Disable metric \a metric_name
  OpenTelemetryPluginBuilder&  DisableMetric(absl::string_view metric_name);
  // If set, is invoked by gRPC when a generic method type RPC is seen. \a generic_method_filter should return true if the generic method name should be recorded. Returning false results in the method name being replaced with "generic" in the recorded metrics.
  OpenTelemetryPluginBuilder&  SetGenericMethodFilter(absl::AnyInvocable<bool(absl::string_view /*method_name*/)> generic_method_filter);
  // Builds and registers a OpenTelemetry Plugin
  void BuildAndRegisterGlobal();
};

```

In the future, additional API might be provided to allow registering the plugin
for a particular channel or server builder.

### Java

To be filled

### Go

To be filled

### Python

To be filled

## Migration from OpenCensus

The following sections show the differences between the gRPC OpenCensus spec and
the proposed gRPC OpenTelemetry spec and the mapping of metrics between the two.
It also presents metrics present in OpenCensus spec that do not map to a metric
in the OpenTelemetry spec at present. Two migration strategies are also proposed
for customers who are satisfied with the stats coverage provided by the current
OpenTelemetry spec.

### Metric Schema Comparison

#### Differences from gRPC OpenCensus Spec

*   OpenTelemetry instrument names don’t allow ‘/’ so we use ‘.’ as the
    separator. We also get rid of the “.io” suffix in “grpc.io” as it doesn’t
    seem to add any value and is consistent with other names in the metrics spec
    from OpenTelemetry.
*   We also use this opportunity to resolve ambiguities from the gRPC OpenCensus
    spec (detailed below).
*   OpenTelemetry has attributes similar to tags in OpenCensus, and the
    OpenCensus tag names already seem to match the OpenTelemetry spec - except
    for ‘\_’ vs ‘.’ for namespaces. So we just replace the ‘\_’ with ‘.’. Note
    that the 'client' and 'server' distinction has also been removed since it
    does not add any benefit.
    *   grpc_client_method -> grpc.method
    *   grpc_client_status -> grpc.status
    *   grpc_server_method -> grpc.method
    *   grpc_server_status -> grpc.status
*   Two new attributes have been added.
    *   grpc.target - Added on client metrics
    *   grpc.authority - Added on server metrics
*   Latency metrics in the OpenTelemetry spec use the recommended `s` unit
    instead of `ms`.

#### Metrics with Corresponding Equivalent

The following OpenCensus metrics have an equivalent in the OpenTelemetry spec
(with the above noted differences) allowing for receivers of the telemetry data
to join the views from the two metrics for continuity.

gRPC OpenCensus                  | gRPC OpenTelemetry
-------------------------------- | ---------------------------------------------
grpc.io/client/started_rpcs      | grpc.client.attempt.started
grpc.io/client/completed_rpcs    | (Derivable from grpc.client.attempt.duration)
grpc.io/client/roundtrip_latency | grpc.client.attempt.duration
grpc.io/server/started_rpcs      | grpc.server.call.started
grpc.io/server/completed_rpcs    | (Derivable from grpc.server.call.duration)
grpc.io/server/server_latency    | grpc.server.call.duration

#### Metrics with Nuanced Differences

Unfortunately, the implementations of the gRPC OpenCensus spec in the various
languages do not agree on the definition of the following size metrics. Go
records uncompressed message bytes for the OpenCensus metric, while C++ and Java
record the compressed message bytes. The OpenTelemetry spec proposed here calls
for recording the compressed message bytes, resulting in an equivalence between
the metrics definitions for C++ and Java, but not for Go.

gRPC OpenCensus                       | gRPC OpenTelemetry
------------------------------------- | ------------------
grpc.io/client/sent_bytes_per_rpc     | grpc.client.attempt.sent_total_compressed_message_size
grpc.io/client/received_bytes_per_rpc | grpc.client.attempt.rcvd_total_compressed_message_size
grpc.io/server/sent_bytes_per_rpc     | grpc.server.call.sent_total_compressed_message_size
grpc.io/server/received_bytes_per_rpc | grpc.server.call.rcvd_total_compressed_message_size

#### OpenCensus Metrics not Initially Supported in OpenTelemetry

There are some additional metrics defined in the gRPC OpenCensus spec and retry
stats which we will not be supporting in the first iteration of the
OpenTelemetry plugin. Some of these will eventually be accepted into the
OpenTelemetry spec with the appropriate changes.

*   Client Views
    *   grpc.io/client/sent_messages_per_rpc
    *   grpc.io/client/received_messages_per_rpc
    *   grpc.io/client/server_latency
    *   grpc.io/client/sent_messages_per_method
    *   grpc.io/client/received_messages_per_method
    *   grpc.io/client/sent_bytes_per_method
    *   grpc.io/client/received_bytes_per_method
*   Server Views
    *   grpc.io/server/sent_messages_per_rpc
    *   grpc.io/server/received_messages_per_rpc
    *   grpc.io/server/sent_messages_per_method
    *   grpc.io/server/received_messages_per_method
    *   grpc.io/server/sent_bytes_per_method
    *   grpc.io/server/received_bytes_per_method
*   Retry Views
    *   grpc.io/client/retries_per_call
    *   grpc.io/client/retries
    *   grpc.io/client/transparent_retries_per_call
    *   grpc.io/client/transparent_retries
    *   grpc.io/client/retry_delay_per_call

## Migration Strategies

### Migrate on a Per-Client Basis

*   Update telemetry dashboards and alerts to join the results from the
    OpenCensus metrics and the OpenTelemetry metrics.
*   Roll out changes to client and server binaries to register the OpenTelemetry
    plugin instead of the OpenCensus plugin.
*   After 100% rollout and some duration (to maintain previous history), update
    telemetry dashboards and alerts to not query OpenCensus metrics.

### Duplicate Metrics During Migration

For this strategy, gRPC stacks need to support registration of both the
OpenCensus and the OpenTelemetry plugins at the same time and allow both metrics
to be exported. This allows users to experiment with OpenTelemetry before
disabling the OpenCensus plugin.

*   Both plugins are registered to gRPC, resulting in both metrics being
    exported. (Note the cost of reporting stats from two plugins at the same
    time.)
*   Separate dashboards and alerts are created for the OpenTelemetry metrics.
    (No join is needed anymore.)
*   Remove registration of OpenCensus plugin when monitoring from OpenTelemetry
    plugin is deemed satisfactory.

## Rationale

OpenCensus is no longer being actively maintained and is being deprecated, with
OpenTelemetry suggested as the successor framework. The OpenTelemetry spec aims
to maintain compatibility with the gRPC OpenCensus spec wherever reasonable to
allow for an easy migration path.

There is a
[General RPC conventions](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/metrics/semantic_conventions/rpc-metrics.md)
doc that is currently in `experimental` status. Given the different nuances that
each RPC system has, it seems difficult to adopt one convention that would make
sense for all systems. For gRPC specifically, the following differences are
immediately obvious -

*   gRPC differentiates between the concept of a `call` and an `attempt`. Each
    `call` can have multiple `attempts` with retries/hedging.
*   The various gRPC implementations can record the compressed message lengths,
    but not all implementations can get the uncompressed message length (as
    recommended by OTel RPC conventions.)

This gRFC, hence, intends to override the [General RPC conventions] for gRPC's
purposes.

## Implementation

Implementations for the OpenTelemetry plugin are currently planned for C++,
Java, Go and Python.

*   C++ - A basic stats functionality for OpenTelemetry (though still internal)
    has been implemented in https://github.com/grpc/grpc/pull/33650. This would
    be expanded on and moved to experimental status once the API is approved and
    implemented. Note that this PR only added bazel support for the plugin.
    CMake support will also be added shortly.
*   Java - TBD but assumed to be implemented by @DNVindhya.
*   Go - TBD but assumed to be implemented by @zasweq.
*   Python - TBD but assumed to be implemented by @XuanWang-Amos.
