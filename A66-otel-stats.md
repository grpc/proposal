# OpenTelemetry Metrics

*   Author: Yash Tibrewal (@yashykt), Zach Reyes (@zasweq), Vindhya Ningegowda
    (@DNVindhya), Xuan Wang (@XuanWang-Amos)
*   Approver: Mark Roth (@markdroth)
*   Status: Final
*   Implemented in: <language, ...>
*   Last updated: Sep 28, 2023
*   Discussion at: https://groups.google.com/g/grpc-io/c/po-deqYEQzE
*   Updated by: [gRFC A108: OpenTelemetry Custom Per-Call Metric
    Label](A108-otel-custom-per-rpc-label.md)

## Abstract

Describe a cross-language plugin architecture for collecting OpenTelemetry
metrics in the various gRPC implementations and propose a data model for gRPC
OpenTelemetry metrics.

## Background

There are a collection of
[metrics](https://github.com/census-instrumentation/opencensus-specs/blob/master/stats/gRPC.md)
proposed by OpenCensus for gRPC. OpenCensus is
[no longer being actively maintained](https://opentelemetry.io/blog/2023/sunsetting-opencensus/),
with OpenTelemetry suggested as the successor framework.

### Related Proposals:

*   [A6: gRPC Retry Design](A6-client-retries.md)
*   [A39: xDS HTTP Filter Support](A39-xds-http-filters.md)
*   [A45: Exposing OpenCensus Metrics and Tracing for gRPC retry](A45-retry-stats.md)

## Proposal

### OpenTelemetry Plugin Architecture

This section describes a `CallTracer` approach to collect the client and server
per-attempt/call metrics. Implementations are free to choose different ways of
representing/naming the classes and methods described here as long as the
overall capabilities remain equivalent.

A CallTracer is a class that is instantiated for every call. This class has
various methods that are invoked during the lifetime of the call. On the
client-side, the CallTracer knows about multiple attempts on the same call (due
to retries or hedging), and creates a `CallAttemptTracer` object for each
attempt, and the `CallAttemptTracer` gets invoked during the lifetime of the
attempt. On the server-side, we have an equivalent `ServerCallTracer`. (There is
no concept of an attempt on the server-side.)

The OpenTelemetry plugin will configure CallTracer factories on gRPC channels
and servers.

A CallTracer needs to know the channel's target in the canonical form, and the
fully qualified method name for filling in the attributes needed on the metrics.
Similarly on the server-side, the `ServerCallTracer` needs to know the method of
the incoming call. Depending on the implementation details, the method may be
propagated as part of the initial metadata.

The following call-outs are needed on the `CallTracer` -

*   When the call has been created. This call-out should be before payload
    serialization.
*   When new attempts are created on the call along with information on whether
    the attempt was a transparent retry or not. (Attempts are created after name
    resolution and after any xDS HTTP filters but before the LB pick.) This is
    also when it's expected for the `CallAttemptTracer` to be created.
*   When an attempt ends. This will be needed for future stats around retries
    and hedging. This information can also be propagated through the
    `CallAttemptTracer` if the `CallAttemptTracer` keeps a reference to the
    parent `CallTracer` object.
*   When the call ends. This along with the call creation call-out allows the
    `CallTracer` to calculate the call duration.

The following call-outs are needed on the `CallAttemptTracer` -

*   When a new message is sent/received. The message should be in its compressed
    form.
*   When the trailing metadata/status is received for the attempt. Receipt of
    this indicates that the attempt has ended. Implementations may choose to
    delegate the responsibility of notifying the `CallTracer` about the attempt
    end to the `CallAttemptTracer`.

The following call-outs are needed on the `ServerCallTracer` -

*   When initial metadata is received by the transport for a call. This
    indicates the start time of a new call.
*   When a new message is sent/received. The message should be in its compressed
    form.
*   When trailing metadata/status is sent. This call-out should be as close to
    the transport as possible to be able to capture the total time of the call.

Implementations should allow multiple call/attempt tracers to be registered to a
single call since there could be multiple plugins registered. For example, there
could be an OpenCensus and an OpenTelemetry stats plugin registered together. It
should also allow multiple OpenTelemetry plugins to be registered providing the
ability to configure the different plugins with different MeterProviders.

A sample implementation of this approach is available in
[gRPC Core](https://github.com/grpc/grpc/blob/v1.57.x/src/core/lib/channel/call_tracer.h).

In grpc-java, a client interceptor is provided by the gRPC OpenTelemetry plugin.
This interceptor adds a `CallAttemptTracerFactory` to the client call. This
factory is equivalent to the `CallTracer`. For each attempt, this factory is
invoked to create a `ClientStreamTracer` analogous to `CallAttemptTracer` for
each attempt. On the server-side, a `ServerStreamTracer.Factory` is used to
create tracers analogous to `ServerCallTracer` for each incoming call.

In grpc-go, similar to grpc-java, an interceptor is invoked per call. This
interceptor is registered when the OpenTelemetry Dial Option is passed in to the
channel, and has access to a context scoped to the call. `StatsHandler` object
owned by the channel gets call-outs for each event that happens on the lifetime
of an attempt. Along with each call-out gets, a context object scoped to the
attempt is passed in, making it equivalent to the functionality of the
`CallAttemptTracer`. On the server side, a `StatsHandler` object gets call-outs
similarly along with a server call scoped context object, to get
`ServerCallTracer` equivalent functionality.

### Language-Specific Details

Each language implementation will provide an API for registering an
OpenTelemetry plugin. Overall, the APIs should have the following capabilities -

*   Allow installing multiple OpenTelemetry plugins.
*   Implementations must provide an option to set
    [MeterProvider](https://opentelemetry.io/docs/specs/otel/metrics/api/#meterprovider)
    on individual plugins. A MeterProvider not being set should result in a
    no-op. Some OpenTelemetry language APIs have a global MeterProvider. gRPC
    implementations should *NOT* fallback on this global.

Note that implementations of the gRPC OpenTelemetry plugin
[should prefer](https://opentelemetry.io/docs/specs/otel/overview/) to only
depend on the OpenTelemetry API and not the OpenTelemetry SDK.

The [Meter](https://opentelemetry.io/docs/specs/otel/metrics/api/#get-a-meter)
creation should use a `name` that identifies the library, for example,
"grpc-c++", "grpc-java", "grpc-go". The `version` should be the same as the
release version of the gRPC library, for example, "1.57.1". The instruments
described above will be created from this meter.

Users of the gRPC OpenTelemetry plugin will use the OpenTelemetry SDK's
MeterProvider to
[control the views](https://opentelemetry.io/docs/specs/otel/metrics/sdk/#view)
and customize the metrics that will be exported.

#### C++

```c++
class OpenTelemetryPluginBuilder {
 public:
  OpenTelemetryPluginBuilder();
  // If `SetMeterProvider()` is not called, no metrics are collected.
  OpenTelemetryPluginBuilder& SetMeterProvider(
      std::shared_ptr<opentelemetry::metrics::MeterProvider> meter_provider);
  // If set, \a generic_method_attribute_filter is called per call with a
  // generic method type to decide whether to record the method name or to
  // replace it with "other". Non-generic or pre-registered methods remain
  // unaffected. If not set, by default, generic method names are replaced with
  // "other" when recording metrics.
  OpenTelemetryPluginBuilder& SetGenericMethodAttributeFilter(
      absl::AnyInvocable<bool(absl::string_view /*generic_method*/) const>
          generic_method_attribute_filter);
  // Registers a global plugin that acts on all channels and servers running on
  // the process.
  // The most common way to use this API is -
  //
  // OpenTelemetryPluginBuilder().SetMeterProvider(provider)
  //    .BuildAndRegisterGlobal();
  //
  // The set of instruments available are -
  // grpc.client.attempt.started
  // grpc.client.attempt.duration
  // grpc.client.attempt.sent_total_compressed_message_size
  // grpc.client.attempt.rcvd_total_compressed_message_size
  // grpc.server.call.started
  // grpc.server.call.duration
  // grpc.server.call.sent_total_compressed_message_size
  // grpc.server.call.rcvd_total_compressed_message_size
  void BuildAndRegisterGlobal();
};

```

In the future, additional API might be provided to allow registering the plugin
for a particular channel or server builder.

#### Java

```java
public final class GrpcOpenTelemetry {

  /**
   * Builder for configuring GrpcOpenTelemetry.
   */
  public static class Builder {
    /**
     * OpenTelemetry instance is used to configure metrics settings.
     *
     * Sample
     *    SdkMeterProvider sdkMeterProvider = SdkMeterProvider.builder()
     *         .registerMetricReader(
     *             PeriodicMetricReader.builder(
     *                OtlpGrpcMetricExporter.builder().build()).build())
     *         .build();
     *
     *     OpenTelemetry openTelemetry = OpenTelemetrySdk.builder()
     *         .setMeterProvider(sdkMeterProvider)
     *         .build();
     *
     *     GrpcOpenTelemetry grpcOpenTelemetry = GrpcOpenTelemetry.newBuilder()
     *         .sdk(openTelemetry)
     *         .build();
     *
     * If MeterProvider is not configured, no-op meterProvider will be used by default.
     * It provides meters which do not record or emit.
     */
    public Builder sdk(OpenTelemetry openTelemetry);

    public GrpcOpenTelemetry build();
  }

  /**
   * Creates an empty builder.
   */
  public static Builder newBuilder();

  /**
   * Establishes GrpcOpenTelemetry instance as the global instrumentation provider for gRPC opentelemetry,
   * automatically applying its configuration to all gRPC channels and servers created after this call.
   *
   * Sample
   *    GrpcOpenTelemetry grpcOpenTelemetry = GrpcOpenTelemetry.newBuilder()
   *         .sdk(openTelemetry)
   *         .build();
   *
   *    grpcOpenTelemetry.registerGlobal();
   *
   * <p> Note: Only one of GrpcOpenTelemetry instance can be registered globally. Any subsequent call to
   * registerGlobal() will throw an IllegalStateException.
   */
  public void registerGlobal();

  /**
   * Configures the given ManagedChannelBuilder with OpenTelemetry metrics instrumentation.
   */
  public void configureChannelBuilder(ManagedChannelBuilder<?> builder);

  /**
   * Configures the given ServerBuilder with OpenTelemetry metrics instrumentation.
   */
  public void configureServerBuilder(ServerBuilder<?> serverBuilder);
}
```

Note:
- For non-generated methods, method names are recorded as "other" for
`grpc.method` attribute. If you are interested in recording the method names for
these methods, set
[`isSampledToLocalTracing`](https://grpc.github.io/grpc-java/javadoc/io/grpc/MethodDescriptor.html#isSampledToLocalTracing\(\))
to `true` while defining your methods in
[`HandlerRegistry`](https://grpc.github.io/grpc-java/javadoc/io/grpc/HandlerRegistry.html).
- A single `GrpcOpenTelemetry` instance can be registered either globally or on a
per-channel and/or per-server basis. Registering the same instance more than
once may lead to duplicated data.

#### Go

```go
import (
  "go.opentelemetry.io/otel/attribute"
  "go.opentelemetry.io/otel/metric"
)


package opentelemetry

// MetricsOptions are the metrics options for OpenTelemetry instrumentation.
type MetricsOptions struct {
  // MeterProvider is the MeterProvider instance that will be used for access
  // to Named Meter instances to instrument an application. To enable metrics
  // collection, set a meter provider. If unset, no metrics will be recorded.
  MeterProvider metric.MeterProvider
  // MethodAttributeFilter is a callback that takes the method string and
  // returns a bool representing whether to use method as a label value or use
  // the string "other". If unset, will use the method string as is. This is
  // used only for generic methods, and not registered methods.
  MethodAttributeFilter func(string) bool
}

// DialOption returns a dial option which enables OpenTelemetry instrumentation
// code for a grpc.ClientConn.
//
// Client applications interested in instrumenting their grpc.ClientConn should
// pass the dial option returned from this function as a dial option to
// grpc.Dial().
func DialOption(mo MetricsOptions) grpc.DialOption {}

// ServerOption returns a server option which enables OpenTelemetry
// instrumentation code for a grpc.Server.
//
// Server applications interested in instrumenting their grpc.Server should pass
// the server option returned from this function as an argument to
// grpc.NewServer().
func ServerOption(mo MetricsOptions) grpc.ServerOption {}
```

#### Python

```python

from opentelemetry.sdk.metrics import MeterProvider

class OpenTelemetryPlugin:
    """Describes a Plugin for OpenTelemetry observability.

    This is class is part of an EXPERIMENTAL API.
    """

    def get_meter_provider(self) -> Optional[MeterProvider]:
        """
        This function will be used to get the MeterProvider for this OpenTelemetryPlugin
        instance.

        Returns:
            A MeterProvider which will be used to collect telemetry data, or None which
            means no metrics will be collected.
        """
        return None

    def generic_method_attribute_filter(
        self, method: str
    ) -> bool:
        """
        If set, this will be called with a generic method type to decide whether to
        record the method name or to replace it with "other".

        Note that pre-registered methods will always be recorded no matter what this
        function returns.

        Args:
            method: The method name for the RPC.

        Returns:
            bool: True means the original method name will be used, False means method name
            will be replaced with "other".
        """
        return False
```

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
up to the user of the gRPC OpenTelemetry plugin to choose the right bucket
boundaries and set it through the
[OpenTelemetry SDK](https://opentelemetry.io/docs/specs/otel/metrics/sdk/#view).

Note that, according to an
[OpenTelemetry proposal on stability](https://docs.google.com/document/d/1Nvcf1wio7nDUVcrXxVUN_f8MNmcs0OzVAZLvlth1lYY/edit#heading=h.dy1cg9doaq26),
changes to bucket boundaries may not be considered as breaking. Depending on the
proposal, this recommendation would change to use `ExponentialHistogram`s
instead, which would allow for automatic adjustments of the scale to better fit
the data.

#### Attributes

*   `grpc.method` : Full gRPC method name, including package, service and
    method, e.g. "google.bigtable.v2.Bigtable/CheckAndMutateRow". Note that gRPC
    servers can receive arbitrary method names, i.e., method names that have not
    been registered in advance with the server. This normally results in those
    RPCs being rejected with an UNIMPLEMENTED status. Some gRPC implementations
    allow servers to handle such generic method names. Since the stats plugin
    would be recording all of these RPCs, this could open up the server to
    malicious attacks that result in metrics being stored with a high
    cardinality. To prevent this, unregistered/generic method names should by
    default be reported with "other" value instead. Implementations should
    provide the option to override this behavior to allow recording generic
    method names as well.
*   `grpc.status` : gRPC server status code received, e.g. "OK", "CANCELLED",
    "DEADLINE_EXCEEDED".
    [(Full list)](https://grpc.github.io/grpc/core/md_doc_statuscodes.html)
*   `grpc.target` : Canonicalized target URI used when creating gRPC Channel,
    e.g. "dns:///pubsub.googleapis.com:443", "xds:///helloworld-gke:8000".
    Canonicalized target URI is the form with the scheme included if the user
    didn't mention the scheme (`scheme://[authority]/path`). For channels such
    as inprocess channels where a target URI is not available, implementations
    can synthesize a target URI. It is possible for some channels to use IP
    addresses as target strings and this might again blow up the cardinality. In
    the future, we can consider adding the ability to override recorded target
    names to avoid this.

#### Client Per-Attempt Instruments

*   **grpc.client.attempt.started** <br>
    The total number of RPC attempts started, including those that have not completed. <br>
    *Attributes*: grpc.method, grpc.target <br>
    *Type*: Counter <br>
    *Unit*: `{attempt}` <br>
*   **grpc.client.attempt.duration** <br>
    End-to-end time taken to complete an RPC attempt including the time it takes to pick a subchannel. <br>
    *Attributes*: grpc.method, grpc.target, grpc.status <br>
    *Type*: Histogram (Latency Buckets) <br>
    *Unit*: `s` <br>
*   **grpc.client.attempt.sent_total_compressed_message_size** <br>
    Total bytes (compressed but not encrypted) sent across all request messages (metadata excluded) per RPC attempt; does not include grpc or transport framing bytes. <br>
    Attributes: grpc.method, grpc.target, grpc.status <br>
    Type: Histogram (Size Buckets) <br>
    *Unit*: `By` <br>
*   **grpc.client.attempt.rcvd_total_compressed_message_size** <br>
    Total bytes (compressed but not encrypted) received across all response messages (metadata excluded) per RPC attempt; does not include grpc or transport framing bytes. <br>
    *Attributes*: grpc.method, grpc.target, grpc.status <br>
    *Type*: Histogram (Size Buckets) <br>
    *Unit*: `By` <br>

#### Client Per-Call Instruments

*   **grpc.client.call.duration** <br>
    This metric aims to measure the end-to-end time the gRPC library takes to complete an RPC from the application’s perspective. <br>
    Start timestamp - After the client application starts the RPC. <br>
    End timestamp - Before the status of the RPC is delivered to the application. <br>
    If the implementation uses an interceptor then the exact start and end timestamps would depend on the ordering of the interceptors. Non-interceptor implementations should record the timestamps as close as possible to the top of the gRPC stack, i.e., payload serialization should be included in the measurement. <br>
    *Attributes*: grpc.method, grpc.target, grpc.status <br>
    *Type*: Histogram (Latency Buckets) <br>
    *Unit*: `s` <br>

#### Server Instruments

*   **grpc.server.call.started** <br>
    The total number of RPCs started, including those that have not completed. <br>
    *Attributes*: grpc.method <br>
    *Type*: counter <br>
    *Unit*: {call} <br>
*   **grpc.server.call.sent_total_compressed_message_size** <br>
    Total bytes (compressed but not encrypted) sent across all response messages (metadata excluded) per RPC; does not include grpc or transport framing bytes. <br>
    *Attributes*: grpc.method, grpc.status <br>
    *Type*: Histogram (Size Buckets) <br>
    *Unit*: `By` <br>
*   **grpc.server.call.rcvd_total_compressed_message_size** <br>
    Total bytes (compressed but not encrypted) received across all request messages (metadata excluded) per RPC; does not include grpc or transport framing bytes. <br>
    *Attributes*: grpc.method, grpc.status <br>
    *Type*: Histogram (Size Buckets) <br>
    *Unit*: `By` <br>
*   **grpc.server.call.duration** <br>
    This metric aims to measure the end2end time an RPC takes from the server transport’s (HTTP2/ inproc) perspective. <br>
    Start timestamp - After the transport knows that it's got a new stream. For HTTP2, this would be after the first header frame for the stream has been received and decoded. Whether the timestamp is recorded before or after HPACK is left to the implementation. <br>
    End timestamp - Ends at the first point where the transport considers the stream done. For HTTP2, this would be when scheduling a trailing header with END_STREAM to be written, or RST_STREAM, or a connection abort. Note that this wouldn’t necessarily mean that the bytes have also been immediately scheduled to be written by TCP. <br>
    *Attributes*: grpc.method, grpc.status <br>
    *Type*: Histogram (Latency Buckets) <br>
    *Unit*: `s` <br>

### Migration from OpenCensus

The following sections show the differences between the gRPC OpenCensus spec and
the proposed gRPC OpenTelemetry spec and the mapping of metrics between the two.
It also presents metrics present in OpenCensus spec that do not map to a metric
in the OpenTelemetry spec at present. Two migration strategies are also proposed
for customers who are satisfied with the stats coverage provided by this spec.

#### Metric Schema Comparison

##### Differences from gRPC OpenCensus Spec

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
*   One new attribute has been added.
    *   grpc.target - Added on client metrics
*   Latency metrics in the OpenTelemetry spec use the recommended `s` unit
    instead of `ms`.

##### Metrics with Corresponding Equivalent

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

##### Metrics with Nuanced Differences

Unfortunately, the implementations of the gRPC OpenCensus spec in the various
languages do not agree on the definition of the following message size metrics.
Go records uncompressed message bytes for the OpenCensus metric, while C++ and
Java record the compressed message bytes. The OpenTelemetry spec proposed here
calls for recording the compressed message bytes, resulting in an equivalence
between the metrics definitions for C++ and Java, but not for Go.

gRPC OpenCensus                       | gRPC OpenTelemetry
------------------------------------- | ------------------
grpc.io/client/sent_bytes_per_rpc     | grpc.client.attempt.sent_total_compressed_message_size
grpc.io/client/received_bytes_per_rpc | grpc.client.attempt.rcvd_total_compressed_message_size
grpc.io/server/sent_bytes_per_rpc     | grpc.server.call.sent_total_compressed_message_size
grpc.io/server/received_bytes_per_rpc | grpc.server.call.rcvd_total_compressed_message_size

##### OpenCensus Metrics not Initially Supported in OpenTelemetry

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

#### Migration Strategies

##### Migrate on a Per-Client Basis

*   Update telemetry dashboards and alerts to join the results from the
    OpenCensus metrics and the OpenTelemetry metrics.
*   Roll out changes to client and server binaries to register the OpenTelemetry
    plugin instead of the OpenCensus plugin.
*   After 100% rollout and some duration (to maintain previous history), update
    telemetry dashboards and alerts to not query OpenCensus metrics.

##### Duplicate Metrics During Migration

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

OpenCensus is no longer being actively maintained, with OpenTelemetry suggested
as the successor framework. The OpenTelemetry spec aims to maintain
compatibility with the gRPC OpenCensus spec wherever reasonable to allow for an
easy migration path.

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
    recommended by OpenTelemetry RPC conventions.)

This gRFC, hence, intends to override the
[General RPC conventions](https://opentelemetry.io/docs/specs/otel/metrics/semantic_conventions/rpc-metrics/)
for gRPC's purposes.

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
*   Python - Basic functionalities have been implemented and are expected to be
    available in version 1.62.0.
