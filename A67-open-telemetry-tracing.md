A67: Open Telemetry Tracing 
----
* Author(s): @zivy
* Approver: @ejona86 @markdroth @dfawley @fengli
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2023-07
* Discussion at: 

## Abstract

This proposal adds support for OpenTelemetry tracing and migration
paths away from OpenCensus tracing. Discussions include:
* Context propagation between a gRPC client and gRPC server during migration.
* Migrate a gRPC involved software binary that has both OpenTelemetry and 
OpenCensus dependency.
* The API surface to enable and configure OpenTelemetry tracing.

Note that stats and logging are out of scope.

## Background

This work aligns with the community consensus to switch to OpenTelemetry as the 
next generation OpenCensus. The latter is no longer maintained after July 31, 2023.

Currently, gRPC supports OpenCensus based tracing in its grpc-census module, or 
alike. gRPC tracing is built with gRPC's core stream tracer and interceptor 
infrastructures. A gRPC client when intercepting the call creates a child span
with the parent span from the current context, and further creates attempt spans
upon stream creation for each attempt. [gRFC A45][A45] describes cross language 
design considerations about creating a
child span for each individual call attempt. A gRPC server uses span ID from the
incoming request header as a parent span to maintain the parent child span 
relationship with the gRPC client. To propagate span context over the wire, gRPC
uses metadata (header name: `grpc-trace-bin`) and OpenCensus's binary format for 
(de)serialization. The header name is unique from other census library 
propagators to differentiate with the application’s tracing instrumentation.

The following tracing information during an RPC lifecycle is captured:

At the client, on parent span:
* When the call is started, annotate name resolution completed if the RPC had 
name resolution delay.
* When the uncompressed size of some outbound data is revealed, annotate seq no.,
type(Received) and wire message size.
* When the call is closed, end the parent span with RPC status.

On attempt span:
* When span is created, add attributes on: previous-rpc-attempts and transparent-retry.
* When the stream is created on transport, annotate delayed load balancer pick 
complete, if any.
* When an outbound message has been sent, add a message event to capture seq no., 
type(SENT) and wire size.
* When an inbound message has been received from the transport, add a message 
event to capture seq no., type(Received) and uncompressed message size.
* When the stream is closed, end the child span with RPC status.

At the server:
* When an outbound message has been sent, add a message event to capture seq no.,
type(SENT) and size.
* When an inbound message has been read from the transport, add a message event 
to capture seq no., type(Received) and size.
* When the uncompressed size of some inbound data is revealed, annotate seq no.,
type(Received) and wire message size.
* When the stream is closed, end the span with RPC status.

### gRPC Census API
The APIs to enable gRPC tracing are different between languages, e.g. in 
grpc-java it is zero-configured: as long as the grpc-census dependency exists in
the classpath, the tracing is automatically generated. In C++, there is an API 
for the users to call to enable tracing. In Go, it is exposed via stream tracers.

### gRPC GCP Observability
Following the census tracing instrumentation, gRPC supports exporting tracing
information to GCP Stackdriver, see [user guide][grpc-observability-public-doc]. 
We distinguish and exclude GCP observability from this design: GCP observability 
is about exporting data while this design is about instrumentation. Migrating 
GCP observability to OpenTelemetry is a future project that depends on this work.

### Related Proposals and Documents: 
* [gRFC L29: C++ API Changes for OpenCensus Integration][L29]
* [gRFC A45: Exposing OpenCensus Metrics and Tracing for gRPC retry][A45]
* [Microservices observability overview] [grpc-observability-public-doc]
* [gRFC A66: OpenTelemetry Metrics][A66]

## Proposal

gRPC users depending on the grpc-census module have non-trivial migration paths 
to OpenTelemetry. Consider the following use cases:
1. Migrate the application binary where both OpenCensus and OpenTelemetry may exist
in the dependency tree. This can be the application’s own tracing code, or gRPC 
OpenCensus, or other dependencies that involve OpenCensus and/or OpenTelemetry.
2. Compatibility between a gRPC client and server as two distributed components, 
where during migration one will use OpenCensus and the other will use OpenTelemetry.

All related APIs here are experimental until metric [gRPC A66][A66] is design and implementation complete.

### Grpc OpenTelemetry module
We will add a separate module grpc-open-telemetry that integrates OpenTelemetry, 
similar to the grpc-census module. Languages will keep using the same gRPC 
infrastructures, e.g. interceptor and stream tracer to implement the feature. 
Providing both modules is to allow users who already depend on grpc-census to 
continue using it for the newer grpc version and offer grace time for the migration.


### Propagator Wireformat
While gRPC OpenCensus directly interacts with metadata API, gRPC OpenTelemetry 
will use the standardized propagators API for context propagation, for the 
following benefits:
1. Fully integration with OpenTelemetry APIs that is easier for users to reason about.
2. Make it possible to plugin other propagators that the community supports.

As of today, OpenTelemetry propagator API only supports `TextMapPropagator`, 
that is to send string key/value pairs between the client and server, which is 
different from the binary header that gRPC currently uses. The future roadmap 
to support binary propagators at Open Telemetry is unclear. So, gRPC will use 
propagator API in TextMap format with optimization path (Go and Java) to work 
around the binary propagator API. Once the OpenTelemetry binary propagator API 
is available in the future, we can continuously integrate with those API with 
little effort. In fact, textmap propagator has negligible performance impact for
C++, so a binary propagator API is not critical.

gRPC will expose a custom `grpcTraceBinPropagator` that implements `TextMapPropagator`. 
This grpc-provided propagator still uses the `grpc-trace-bin` header for context 
propagation. The OpenCensus spanContext and OpenTelemetry spanContext transmitted
in binary over the wire are identical, therefore a gRPC OpenCensus client can 
speak with a gRPC OpenTelemetry server and vice versa. Users can provide a 
single composite propagator that combines one or multiple `TextMapPropagator` 
for their client and server separately. This way, users can define their own 
migration path for context propagators in distributed components. Configuring 
gRPC OpenTelemetry with this propagator when dealing with cross-cutting concerns
is straightforward and recommended.

####  Propagator API in Java/Go
The pseudocode below shows `grpcTraceBinPropagator` and the corresponding 
gRPC setter/getters with an optimization path.

```Java
public class GrpcTraceBinTextMapPropagator implements TextMapPropagator {
  @Override
  public <C> void inject(Context context, @Nullable C carrier, TextMapSetter<C> setter) {
    SpanContext spanContext = Span.fromContext(context).getSpanContext();
    byte[] value = BinaryFormat.toBytes(spanContext);
    if (setter instanceof GrpcTraceBinSetter) {
      // Fast path in Java and Go that passes bytes directly through API boundaries using
      // the overloaded set(Metadata, String, byte[]) method added by gRPC.
      ((GrpcTraceBinSetter) setter).set((Metadata) carrier, "grpc-trace-bin", value);
    } else {
      // Slow path for C++. gRPC C++ does not have type checking, so we encode bytes to 
      // String to comply with the TextMapSetter API. This code path is also used in the 
      // situation where GrpcTraceBinTextMapPropagator is used with a TextMapSetter 
      // externally.
      setter.set(carrier, "grpc-trace-bin", Base64.getEncoder().encodeToString(value));
    }
  }

  @Override
  public <C> Context extract(Context context, @Nullable C c, TextMapGetter<C> textMapGetter) {
    byte[] bytes;
    if (textMapGetter instanceof GrpcTraceBinGetter) { //Fast path for Java/Go
      bytes = ((GrpcTraceBinGetter) textMapGetter).getBinary((Metadata) c, "grpc-trace-bin");
    } else {
      // Slow path for C++. gRPC C++ does not have type checking, so we decode String
      // from TextMapGetter API to bytes. This code path applies to the situation 
      // where GrpcTraceBinTextMapPropagator is used with a TextMapGetter externally.
      String contextString = textMapGetter.get(c, "grpc-trace-bin");
      bytes = Base64.getDecoder().decode(contextString);
    }
    SpanContext spanContext = BinaryFormat.parseBytes(bytes);
    return context.with(Span.wrap(spanContext));
  }
}

```

The `grpcTraceBinPropagator` should be compatible with any Getter/Setter, but 
internally in gRPC, in Java and Go we implement a special gRPC Getter/Setter 
that use an optimization path to work around the lack of binary propagator API 
and avoid base64 (de)encoding when passing data between API interfaces. This special 
gRPC Getter/Setter will also be responsible for handling other propagators, 
see pseudocode below. 

```Java
@Internal
class GrpcCommonSetter implements TextMapSetter<Metadata>, GrpcBinarySetter<Metadata> {
  // Fast path for Java and Go. Overload set() method to accept bytes value to avoid 
  // base64 encoding/decoding between API boundaries.
  void set(Metadata header, String key, byte[] value) {
    assert key.endsWith("-bin");
    header.put(Metadata.Key.of(key, BINARY_BYTE_MARSHALLER), value);
  }

  void set(Metadata header, String key, String value) {
    if (key.endsWith("-bin")) {
      // Slower path in C++. It shows the decoding part of the just encoded String at 
      // GrpcTraceBinTextMapPropagator.inject(). 
      // This can also be used to propagate any other binary header.
      header.put(Metadata.Key.of(key, BINARY_BYTE_MARSHALLER), Base64.getDecoder().decode(value));
    } else {
      // Used by other TextMap propagators, e.g. w3c.
      header.put(Metadata.Key.of(key, ASCII_STRING_MARSHALLER), value);
    }
  }
}

class GrpcCommonGetter implements TextMapGetter<Metadata> {
  public String get(@Nullable Metadata carrier, String key) {   
    if (key.endsWith("-bin")) { 
      // Slow path for C++: encode the bytes to String. Later we decode to bytes
      // in GrpcTraceBinTextMapPropagator.extract().
      byte[] value = carrier.get(Metadata.Key.of(key, BINARY_BYTE_MARSHALLER));
      return Base64.getEncoder().encodeToString(value);
    } else {
      // Used by other TextMap propagators, e.g. w3c.
      return carrier.get(Metadata.Key.of(key, ASCII_STRING_MARSHALLER));
    }
  }

  // Add a new method to optimize the TextMap propagator.
  public byte[] getBinary(@Nullable Metadata carrier, String key) {
    assert key.endsWith("bin");
    return carrier.get(Metadata.Key.of(key, BINARY_BYTE_MARSHALLER));
  }
}

// This interface will be implemented by gRPCCommonSetter/Getter as an optimization path 
// to avoid base64 encoding between TextMap APIs due to lack of
// OpenTelemetry binary propagator API.
private interface GrpcBinarySetter<Metadata> {
    void set(Metadata header, String key, byte[] value);
}

```


The `GrpcCommonSetter` adds an overloaded `set()` method to directly take `bytes[]` 
(Java and Go) to avoid extra encoding/decoding. For the normal `set()` method it 
should handle both binary (`-bin`) header ( e.g. for C++ `grpc-trace-bin`) and 
ASCII header which can be used by other text map propagators that gRPC 
OpenTelemetry supports (e.g. w3c).
The `GrpcTraceBinGetter` adds new method `getBinary()` for the optimized path 
in Java and Go. The normal get() method handles both binary header 
(for C++ `grpc-trace-bin`) and ASCII header propagators, like w3c.

#### Propagation API in C++
C++ will also support the propagator API. But due to the language restriction, 
C++ can not take the optimization path to workaround lacking the binary 
propagator API. However, C++ can be configured to interact with metadata 
directly, like the current gRPC OpenCensus, instead of using propagator API, so 
that it avoids paying for the performance cost due to string/binary encoding. 
The two APIs C++ will support for context propagation are:
* If the propagator is configured, take a slower path in the pseudocode described
above. Supporting propagator API puts us in the API uniformity among languages.

* If explicitly configured, gRPC will directly use `Metadata.get()` and `Metadata.put()` 
APIs on the “grpc-trace-bin” header. No TextMapPropagator API and TextMapSetter/Getter 
will be involved. This is a faster path and mitigates performance concerns due 
to base64 encoding.

TODO: add pseudocode here in C++ for `grpcTraceBinPropagator`,`GrpcCommonSetter` 
and `GrpcCommonSetter`.

The `GrpcCommonSetter.set()` and `GrpcCommonGetter.get()` method in C++
should handle both binary (`-bin`) header 
(e.g. `grpc-trace-bin`) and ASCII header which can be used by other text map 
propagators that gRPC OpenTelemetry supports (e.g. w3c).


### Grpc OpenTelemetry API
We provide APIs to allow users to enable and configure OpenTelemetry, including 
the propagators. The OpenTelemetry API will coexist with the OpenCensus API until
the latter is dropped. Only one "grpc-trace-bin" header will be sent for a single RPC.

The APIs are different among languages due to different underlying infrastructures.
In Java, it will be part of global interceptors, so that the interceptors are 
managed in a more sustainable way and user-friendly. As a prerequisite, the stream
tracer factory API will be stabilized. OpenTelemetryModule will be created with 
an OpenTelemetryAPI parameter passing in for necessary configurations. 
Users can also rely on SDK autoconfig extension that configure the sdk object 
through environmental variables or Java system properties, then obtain the sdk 
object passed in to gRPC.

```Java
// Construct OpenTelemetry to be passed to gRPC OT module for trace/meter 
// configurations
SdkTracerProvider sdkTracerProvider = SdkTracerProvider
  .builder()
  .addSpanProcessor(
     BatchSpanProcessor.builder(exporter).build())
  .build();

OpenTelemetry openTelemetry = OpenTelemetrySdk.builder()
  .setTracerProvider(sdkTracerProvider)
  .setMeterProvider(...)
  .setPropagators(
     ContextPropagators.create(GrpcTraceBinTextMapPropagator.getInstance()))
  .build();

// Alternatively, use auto configuration:
// OpenTelemetry openTelemetry = 
// AutoConfiguredOpenTelemetrySdk.getOpenTelemetrySdk().

// OpenTelemetryModule.getInstance() will be using GlobalOpenTelemetry
OpenTelemetryModule otModule = OpenTelemetryModule.getInstance(openTelemetry);

GlobalInterceptors.setInterceptors(
    Arrays.asList(otModule.getClientTracingInterceptor()),
    Arrays.asList(otModule.getServerTracerFactory()));

```

In C++, it will be a method that mirrors OpenCensus API, e.g.

TODO: update C++ API:

```C++
// Enable OpenTelemetry based tracing. Similar to
// RegisterOpenCensusPlugin(). TracerProvider is configured via sdk separately.
void RegisterOpenTelemetryTracingPlugin();

```

In Go, the OpenTelemetry stream tracers and interceptors will be provided for users to install.

TODO: add Go API

### Migrate to Open Telemetry: cross-process networking concerns
When clients first introduce gRPC OpenTelemetry, for the time window when the 
gRPC client and server have mixed plugins of OpenTelemetry and OpenCensus, 
with grpcTraceBinPropagator users can do migration easily with the compatibility
guaranteed. It is encouraged to use `grpc-trace-bin` header for migration because 
of the following advantages:
* Simplified migration path, no migration phase deployments.
* Binary header is more efficient.

A binary formatter implementation for OpenTelemetry is needed, which can be 
similar to OpenCensus implementation. Go already has community support for that.

OpenTelemetry and its extension packages support multiple text map propagators, 
e.g. W3C trace context or b3. The API supports specifying multiple propagators: 
either public standard ones or custom propagators that implement the OpenTelemetry 
propagator API interface. The API composites the propagators and gRPC puts all 
the propagator data into the wire through metadata. This allows users to migrate
a group of applications with an old propagator to a new propagator. An example 
migration path can be:
1. Set server to accept both old and new propagators.
2. Add desired new propagators to the client.
3. Make the server only accept the new propagators and complete the migration.

### Migrate to OpenTelemetry: in binary
The OpenCensus [shim](https://github.com/open-telemetry/opentelemetry-java/tree/main/opencensus-shim)
(currently available in Java, Go, Python) allows binaries that have a mix of 
OpenTelemetry and OpenCensus dependencies to export trace spans from both with 
the correct parent-child relationship. This is the recommended approach to migrate
to OpenTelemetry in one binary gradually. Note that the in-binary migration and 
cross-cutting concerns migration can be done in parallel.

The shim packages that bridge two libraries works as follows, considering the
following migration scenarios example: 

|-- Application - Configured OpenCensus --------------------------------- |\
|--  gRPC -> Using OpenCensus to generate Trace A  --------- |\
|--  Application -> Using OpenCensus to generate a sub Trace B-- |

The application may use a bridge package in the outermost layer first:\
|-- Application - Configured Otel w/ OpenCensus Shim ------------------- |\
|--  gRPC -> Using OpenCensus to generate Trace A  --------- |\
|--  Application -> Using OpenCensus to generate a sub Trace B-- |

Then the application change the instrumentation to Open Telemetry:\
|-- Application - Configured Otel w/ OpenCensus Shim ---------------------- |\
|--  gRPC -> Using Otel to generate Trace A  --------- |\
|--  Application -> Using Otel to generate a sub Trace B-- |

Finally, they switch to grpc-open-telemetry and finish the migration.\
|-- Application - Configured Otel standalone ----------------------------- |\
|--  gRPC -> Using Otel to generate Trace A  --------- |\
|--  Application -> Using Otel to generate a sub Trace B-- |

## Rationale

[L29]: L29-cpp-opencensus-filter.md
[A45]: A45-retry-stats.md
[grpc-observability-public-doc]: https://cloud.google.com/stackdriver/docs/solutions/grpc
[A66]: A66-otel-stats.md