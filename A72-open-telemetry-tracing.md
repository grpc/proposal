A72: OpenTelemetry Tracing 
----
* Author(s): [Yifei Zhuang](https://github.com/YifeiZhuang), [Yash Tibrewal](https://github.com/yashykt), [Xuan Wang](https://github.com/XuanWang-Amos)
* Approver: [Eric Anderson](https://github.com/ejona86)
* Reviewers: [Mark Roth](https://github.com/markdroth), [Doug Fawley](https://github.com/dfawley),
[Feng Li](https://github.com/fengli79)
* Status: Ready for Implementation
* Implemented in: <language, ...>
* Last updated: 2024-01
* Discussion at: https://groups.google.com/g/grpc-io/c/e_ByaRmtJak

# Abstract
This proposal adds support for OpenTelemetry tracing and suggests migration
paths away from OpenCensus tracing. Discussions include:
* The API surface to enable and configure OpenTelemetry tracing.
* Context propagation between a gRPC client and server.
* Migration path from gRPC OpenCensus to OpenTelemetry, considering:
  1) The cross-process concerns during migration.
  2) In-binary migration for a gRPC involved software that has both OpenTelemetry and 
    OpenCensus dependency.

Note that stats and logging are out of scope.

# Background
This work aligns with the community consensus to switch to OpenTelemetry as the 
next generation OpenCensus. The latter is no longer maintained after July 31, 2023.

Currently, gRPC supports OpenCensus based tracing in its grpc-census plugin, or 
alike. gRPC trace is built with its core stream tracer and interceptor 
infrastructures. A gRPC client when intercepting the call creates a child span
with the parent span from the current context, and further creates attempt spans
upon stream creation for each attempt. [gRFC A45][A45] describes cross language 
design considerations about creating a child span for each individual call attempt. 
A gRPC server uses span ID from the incoming request header as a parent span to 
maintain the parent child span relationship with the gRPC client. To propagate 
span context over the wire, gRPC uses metadata (header name: `grpc-trace-bin`) 
and OpenCensus's binary format for (de)serialization. The header name is unique 
from other census library propagators to differentiate with the application’s 
tracing instrumentation.

### Related Proposals and Documents:
* [gRFC L29: C++ API Changes for OpenCensus Integration][L29]
* [gRFC A45: Exposing OpenCensus Metrics and Tracing for gRPC retry][A45]
* [gRFC A66: OpenTelemetry Metrics][A66]

# Proposal
## gRPC OpenTelemetry Tracing API
We will add tracing functions in grpc-open-telemetry plugin, along with OpenTelemetry
metrics [gRFC A66][A66]. Internally, the tracing functionality will be implemented
using existing gRPC infrastructure such as interceptors and stream tracers.
The APIs to enable and configure OpenTelemetry tracing are different among 
languages due to different underlying infrastructures.

### Java
In Java, it will be part of global interceptors, so that the interceptors are
managed in a more sustainable way and user-friendly. Currently `GrpcOpenTelemetry` is constructed
with OpenTelemetry API instance passing in for necessary configurations.
Users can also rely on SDK autoconfig extension that configures the sdk object
through environment variables or Java system properties, then pass the 
obtained sdk object to gRPC.

There are no changes to the Java API. Users would configure `TraceProvider` to the
OpenTelemetry API instance for constructing `GrpcOpenTelemetry` like below.

```Java
// Construct a TraceProvider that will be used to provide traces during instrumentation.
SdkTracerProvider sdkTracerProvider = SdkTracerProvider.builder()
     .addSpanProcessor(
         BatchSpanProcessor.builder(exporter).build())
     .build();
// Construct OpenTelemetry to be passed to gRPC OpenTelemetry module for
// traces and metrics configurations.
OpenTelemetry openTelemetry = OpenTelemetrySdk.builder()
     .setTracerProvider(sdkTracerProvider)
     .setMeterProvider(...)
     .setPropagators(ContextPropagators.create(W3CTraceContextPropagator.getInstance()))
     .build();
GrpcOpenTelemetry otModule = GrpcOpenTelemetry.newBuilder().sdk(openTelemetry).build();

// Add interceptors and StreamTracerFactory obtained from the module globally.
otModule.registerGlobal();
```

### C++
The following new methods will be added in `OpenTelemetryPluginBuilder`.

```C++
class OpenTelemetryPluginBuilder {
 public:
  // If `SetTracerProvider()` is not called, no traces are collected.
  OpenTelemetryPluginBuilder& SetTracerProvider(
      std::shared_ptr<opentelemetry::metrics::TracerProvider> tracer_provider);
  // Set one or multiple text map propagators for span context propagation, e.g.
  // the community standard ones like W3C, etc.
  OpenTelemetryPluginBuilder& SetTextMapPropagator(
      std::unique_ptr<opentelemetry::context::propagation::TextMapPropagator>
          text_map_propagator);
};
```

### Python
The following new fields will be added in `OpenTelemetryPlugin`.

```Python
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.trace.propagation.tracecontext import TraceContextTextMapPropagator

class OpenTelemetryPlugin:
    """
    If `tracer_provider` is None, no traces are collected.
    """

    def __init__(
        self,
        *,
        tracer_provider: Optional[TracerProvider] = None,
        text_map_propagator: [TraceContextTextMapPropagator] = None,
    ):
```

### Go
The following `TraceOptions` field will be added to the `opentelemetry` Options struct.
```go
import (
  "go.opentelemetry.io/otel/trace"
)

type Options struct {
  // Existing field in opentelemetry package
  MetricsOptions MetricsOptions
  TraceOptions TraceOptions
}

// TraceOptions are the trace options for OpenTelemetry instrumentation.
type TraceOptions struct {
  TraceProvider trace.TraceProvider
}

```

## Tracing Information
RPCs on the client side may undergo retry attempts, whereas on the server side, 
they do not. gRPC records both per-call tracing details (on the parent span) 
and per-attempt tracing details (on the attempt span) on the client side. 
On the server side, there is only per-call traces.  With the new OpenTelemetry 
plugin we will produce the following tracing information during an RPC lifecycle:

At the client, on parent span:
* If the RPC experienced name resolution delay, add an Event at the start of the
  call with the name "Delayed name resolution complete" upon completion of the
  name resolution process.
* When the call is closed, set RPC status and end the parent span. gRPC status "OK" 
  is recorded with status "OK", while other gRPC statuses are marked as "ERROR". 
  Non-"OK" statuses include their code as a description, the same below.

On attempt span:
* When span is created, set the attribute with key `previous-rpc-attempts` and an 
  integer value representing the count of previous attempts made for the RPC.
* When span is created, set the attribute with key `transparent-retry` and a 
  boolean value indicating whether the stream is undergoing a transparent retry.
* If the RPC experienced load balancer pick delay, add an Event with the name 
  "Delayed LB pick complete" upon creation of the stream on the transport. 
* When the application sends an outbound message, add Event(s)
  (it depends on implementation whether there is a single event or an additional 
  separate event with name "Outbound message compressed" for compressed message size) 
  with name "Outbound message sent" and the following attributes:
  * key `sequence-number` with integer value of the seq no. The seq no. indicates
    the order of the sent messages on the attempt (i.e., it starts at 0 and is 
    incremented by 1 for each message sent), the same below.
  * key named `message-size-uncompressed` if the implementation knows the message needs compression,
    otherwise named `message-size`, with integer value of uncompressed
    message size. The size is the total attempt message bytes without encryption,
    not including grpc or transport framing bytes, the same below.
  * If compression needed, add key `message-size-compressed` with integer 
    value of compressed message size. If this is reported as a separate event in 
    an implementation, the event name is "Outbound message compressed" and the 
    order of the event must be after the first event that reports the message size.
* When an inbound message has been received, add Event(s) (it depends on 
  implementation whether there is a single event or an additional separate event
  with name "Inbound message uncompressed" for uncompressed message size) with name 
  "Inbound message read" and the following attributes:
  * key `sequence-number` with integer value of the seq no. The seq no. indicates
    the order of the received messages on the attempt (i.e., it starts at 0 and is
    incremented by 1 for each message received), the same below.
  * key named `message-size-compressed` if the implementation knows the message needs decompression,
    otherwise named `message-size`, with integer value of wire message size.
  * If the message needs decompression, add key `message-size-uncompressed`
    with integer value of uncompressed message size. If this is reported as a 
    separate event in an implementation, the event name is "Inbound message uncompressed"
    and the order of the event must be after the first event that reports the wire message size.
* When the stream is closed, set RPC status and end the attempt span.

At the server:
* When the application sends an outbound message, add Event(s) 
  (it depends on implementation whether there is a single event or an additional
  separate event with name "Outbound message compressed" for compressed message size)
  with name "Outbound message sent" and the following attributes:
  * key `sequence-number` with integer value of the seq no.
  * key named `message-size-uncompressed` if the message needs compression,
    otherwise named `message-size`, with integer value of uncompressed
    message size.
  * If compression needed, add key `message-size-compressed` with integer
    value of compressed message size. If this is reported as a separate event in
    an implementation, the event name is "Outbound message compressed" and the
    order of the event must be after the first event that reports the message size.
* When an inbound message has been received, add Event(s) (it depends on
  implementation whether there is a single event or an additional separate event
  with name "Inbound message uncompressed" for uncompressed message size) with name
  "Inbound message read" and the following attributes:
  * key `sequence-number` with integer value of the seq no.
  * key named `message-size-compressed` if the message needs decompression,
    otherwise named `message-size`, with integer value of wire message size.
  * If the message needs decompression, add key `message-size-uncompressed`
    with integer value of uncompressed message size. If this is reported as a
    separate event in an implementation, the event name is "Inbound message uncompressed"
    and the order of the event must be after the first event that reports the wire message size.
* When the stream is closed, set the RPC status and end the span.

### Limitations
The timestamp information on the Events that report compressed/uncompressed message
sizes are not accurate or useful. It only gives you a relative order with other Events. 
We can tighten the timing in the future if users find this information critical.

Java has an open issue of reporting uncompressed message size upon receiving message.
It does that at a later time when deserializing. Therefore, at the client Java only 
reports the uncompressed message size for incoming messages on parent span, not attempt span.

## Propagator Wire Format
gRPC OpenTelemetry will use the existing OpenTelemetry propagators API for context propagation
by encoding them in metadata, for the following benefits:
1. Full integration with OpenTelemetry APIs that is easier for users to reason about.
2. Make it possible to plugin other propagators that the community supports.
3. Flexible API that allows clean and simple migration paths to a different propagator.

We will have OpenTelemetry propagator APIs for context propagation.
In order for the propagator to perform injecting and extracting spanContext value
from the carrier, which is the Metadata in gRPC, languages will
implement Getter and Setter corresponding to the propagator type.
Currently, OpenTelemetry propagator API only supports `TextMapPropagator`,
that is to send string key/value pairs between the client and server.
Therefore, adding Getter and Setter is to implement the TextMap carrier interface:
`TextMapCarrier` (For C++/Go), `opentelemetry.propagators.textmap.Getter/Setter` (For Python)
or `TextMapGetter`/`TextMapSetter` (For Java). (see
pseudocode in section [Migration to OpenTelemetry: Cross-process Networking Concerns](#migration-to-opentelemetry--cross-process-networking-concerns)).

## Migration from OpenCensus to OpenTelemetry

### gRPC OpenCensus API
The existing gRPC OpenCensus tracing APIs in grpc-census plugin are different between
languages, e.g. in grpc-java it is zero-configuration: as long as the grpc-census
dependency exists in the classpath, the traces are automatically generated.
In C++, there is an API for users to call to enable tracing. In Go, it is exposed 
via stream tracers. The OpenTelemetry API will coexist with the OpenCensus API.
We keep the grpc-census plugin to allow users who already depend on grpc-census to
continue using it for newer grpc versions.

gRPC users depending on the grpc-census plugin have non-trivial migration paths
to OpenTelemetry. Consider the following use cases:
1. Compatibility between a gRPC client and server as two distributed components,
where during migration one will use OpenCensus and the other will use OpenTelemetry. 
2. Migrate an application binary where both OpenCensus and OpenTelemetry maybe exist
   in the dependency tree. This can be the application’s own tracing code, or gRPC
   OpenCensus, or other dependencies that involve OpenCensus and/or OpenTelemetry.

Here are the suggested solutions for both use cases. 

### Migration to OpenTelemetry: Cross-process Networking Concerns
When users first introduce gRPC OpenTelemetry, for the time window when the
gRPC client and server have mixed plugins of OpenTelemetry and OpenCensus, 
spanContext can not directly propagate due to different header name and wire format. 
To tackle this, gRPC will expose a custom `GrpcTraceBinPropagator` 
that implements `TextMapPropagator`. This will allow gRPC to keep using `grpc-trace-bin` 
header for context propagation and also support other propagators.
When using `grpc-trace-bin` the OpenCensus spanContext and OpenTelemetry spanContext 
are identical, therefore a gRPC OpenCensus client can speak with a gRPC OpenTelemetry 
server and vice versa. It is encouraged to use `GrpcTraceBinPropagator` for the migration. 
Using the same header greatly simplifies rollout. However, there is a caveat about 
`GrpcTraceBinPropagator`:

Currently, OpenTelemetry propagator API only supports `TextMapPropagator`,
that is to send string key/value pairs between the client and server, which is
different from the binary header that gRPC currently uses. The future roadmap
to support binary propagators at OpenTelemetry is unclear. So, gRPC will use
propagator API in TextMap format with an optimization path (Go and Java) to work
around the lack of binary propagator API to support `grpc-trace-bin`. In fact,
TextMap propagator is a viable alternative to the existing binary format in gRPC 
in terms of performance, based on internal C++ micro benchmarking on W3C TextMap 
propagator. If this posed a performance problem for users, we can consider 
implementing an alternative API in C++, see [Rationale](#rationale).
Only one `grpc-trace-bin` header will be sent for a single RPC as long as only 
one of OpenTelemetry or OpenCensus is enabled for the channel.
A `grpc-trace-bin` formatter implementation for OpenTelemetry is
needed in each language, which can be similar to the OpenCensus implementation.
Go already has community support for that.

#### GrpcTraceBinPropagator and TextMapGetter/Setter in Java/Go
The pseudocode below demonstrates `GrpcTraceBinPropagator` and the corresponding
gRPC Getter/Setter with an optimization path.

```Java
public class GrpcTraceBinPropagator implements TextMapPropagator {
  @Override
  public <C> void inject(Context context, @Nullable C carrier, TextMapSetter<C> setter) {
    SpanContext spanContext = Span.fromContext(context).getSpanContext();
    byte[] value = BinaryFormat.toBytes(spanContext);
    if (setter instanceof GrpcCommonSetter) {
      // Fast path in Java and Go that passes bytes directly through API boundaries using
      // the overloaded set(Metadata, String, byte[]) method added by gRPC.
      ((GrpcCommonSetter) setter).set((Metadata) carrier, "grpc-trace-bin", value);
    } else {
      // Slow path. For the situation where GrpcTraceBinPropagator is used with
      // a TextMapSetter externally.
      setter.set(carrier, "grpc-trace-bin", Base64.getEncoder().encodeToString(value));
    }
  }

  @Override
  public <C> Context extract(Context context, @Nullable C c, TextMapGetter<C> textMapGetter) {
    byte[] bytes;
    if (textMapGetter instanceof GrpcCommonGetter) { //Fast path for Java/Go
      bytes = ((GrpcCommonGetter) textMapGetter).getBinary((Metadata) c, "grpc-trace-bin");
    } else {
      // Slow path. For the situation where GrpcTraceBinPropagator is used with a TextMapGetter
      // externally.
      String contextString = textMapGetter.get(c, "grpc-trace-bin");
      bytes = Base64.getDecoder().decode(contextString);
    }
    SpanContext spanContext = BinaryFormat.parseBytes(bytes);
    return context.with(Span.wrap(spanContext));
  }
}

```

The `GrpcTraceBinPropagator` should be compatible with any Getter/Setter, but
internally in gRPC, in Java and Go we implement a special gRPC Getter/Setter
that uses an optimization path to work around the lack of binary propagator API
and thus avoid base64 (de)encoding when passing data between API interfaces.
This special gRPC Getter/Setter will also be responsible for handling other
propagators that users will configure with gRPC OpenTelemetry (e.g. W3C),
see the pseudocode below.

```Java
@Internal
class GrpcCommonSetter implements TextMapSetter<Metadata>, GrpcBinarySetter<Metadata> {
  // Fast path for Java and Go. Overload set() method to accept bytes value to avoid 
  // base64 encoding/decoding between API boundaries.
  @Override
  void set(Metadata header, String key, byte[] value) {
    assert key.equals("grpc-trace-bin");
    header.put(Metadata.Key.of(key, BINARY_BYTE_MARSHALLER), value);
  }

  @Override
  void set(Metadata header, String key, String value) {
    if (key.equals("grpc-trace-bin")) {
      // Slower path. It shows the decoding part of the just encoded
      // String at GrpcTraceBinPropagator.inject().
      header.put(Metadata.Key.of(key, BINARY_BYTE_MARSHALLER), Base64.getDecoder().decode(value));
    } else if (key.endsWith("-bin")) {
      logger.log(Level.ERROR, "Binary propagator other than GrpcTraceBinPropagator is not supported.");
    } else {
      // Used by other TextMap propagators, e.g. W3C.
      header.put(Metadata.Key.of(key, ASCII_STRING_MARSHALLER), value);
    }
  }
}

class GrpcCommonGetter implements TextMapGetter<Metadata> {
  @Override
  public String get(@Nullable Metadata carrier, String key) {
    if (key.equals("grpc-trace-bin")) {
      // Slow path: return string encoded from bytes. Later we decode to
      // bytes in GrpcTraceBinPropagator.extract().
      byte[] value = carrier.get(Metadata.Key.of(key, BINARY_BYTE_MARSHALLER));
      return Base64.getEncoder().encodeToString(value);
    } else if (key.endsWith("-bin")) {
      logger.log(Level.ERROR, "Binary propagator other than GrpcTraceBinPropagator is not supported.");
    } else {
      // Used by other TextMap propagators, e.g. W3C.
      return carrier.get(Metadata.Key.of(key, ASCII_STRING_MARSHALLER));
    }
  }

  // Add a new method to optimize the TextMap propagator to avoid base64 encoding.
  @Override
  public byte[] getBinary(@Nullable Metadata carrier, String key) {
    assert key.equals("grpc-trace-bin");
    return carrier.get(Metadata.Key.of(key, BINARY_BYTE_MARSHALLER));
  }
}

// This interface will be implemented by gRPCCommonSetter/Getter as an optimization
// path to avoid base64 encoding between TextMap APIs due to lack of
// OpenTelemetry binary propagator API. Not for C++.
private interface GrpcBinarySetter<Metadata> {
  void set(Metadata header, String key, byte[] value);
}

```

The `GrpcCommonSetter` adds an overloaded `set()` method to directly take `byte[]`
(Java and Go) to avoid extra base64 encoding. For the normal `set()` method it
should handle both binary (`-bin`) header and ASCII header from any TextMap
propagators that users may configure.
The `GrpcCommonGetter` in Java and Go adds a new method `getBinary()` for the
optimized path for the same reason. Similarly, the normal `get()` method handles
both binary headers and TextMap propagators.

#### GrpcTraceBinPropagator and TextMapCarrier in C++
C++ will also support propagator APIs to provides API uniformity among
languages. Since gRPC C++ avoids Run-time type information (RTTI), it can not
use the same optimization path as Java/Go. This will result in an extra base64
encoding/decoding step to satisfy `TextMapPropagator` requirement that the
key/value pair be a valid HTTP field. There are possible optimizations C++ might
pursue in the future, for example, providing an explicit knob on
`GrpcTraceBinTextMapPropagator` that assumes that this propagator is being used
with gRPC and can hence skirt `TextMapPropagator` compatibility requirements.

```C++
std::unique_ptr<opentelemetry::context::TextMapPropagator>
MakeGrpcTraceBinTextMapPropagator();
```

The following shows a sketch on what the internal implementation details of this API would look within gRPC C++/Core.

```C++

namespace grpc {
namespace internal {

class GrpcTraceBinTextMapPropagator
    : public opentelemetry::context::TextMapPropagator {
 public:
  void Inject(opentelemetry::context::TextMapCarrier& carrier,
              const opentelemetry::context::Context& context) {
    auto span_context = opentelemetry::trace::GetSpan(context)->GetContext();
    if (!span_context.IsValid()) {
      return;
    }
    carrier.Set(
        "grpc-trace-bin",
        // gRPC C++ does not have RTTI, so we encode bytes to String to comply with the TextMapSetter API. 
        absl::Base64Escape(
            absl::string_view(SpanContextToGrpcTraceBinHeader(span_context))
                .data()),
        kGrpcTraceBinHeaderLen);
  }

  context::Context Extract(const context::propagation::TextMapCarrier& carrier,
                           opentelemetry::context::Context& context) {
    return trace::SetSpan(
        context, nostd::shared_ptr<Span> sp(new DefaultSpan(
                     GrpcTraceBinHeaderToSpanContext(absl::Base64Unescape(
                         carrier.Get("grpc-trace-bin"))))));
  }

 private:
  constexpr int kGrpcTraceBinHeaderLen = 29;

  std::array<uint8_t, kGrpcTraceBinHeaderLen> SpanContextToGrpcTraceBinHeader(
      const opentelemetry::trace::SpanContext& ctx) {
    std::array<uint8_t, kGrpcTraceBinHeaderLen> header;
    header[0] = 0;
    header[1] = 0;
    ctx.trace_id().CopyBytesTo(&header[2], 16);
    header[18] = 1;
    ctx.span_id().CopyBytesTo(&header[19], 8);
    header[27] = 2;
    header[28] = ctx.trace_flags().flags();
    return header;
  }

  opentelemetry::trace::SpanContext GrpcTraceBinHeaderToSpanContext(
      nostd::string_view header) {
    if (header.size() != kGrpcTraceBinHeaderLen || header[0] != 0 ||
        header[1] != 0 || header[18] != 1 || header[27] != 2) {
      return SpanContext::GetInvalid();
    }
    return SpanContext(TraceId(&header[2], 16), SpanId(&header[19], 8),
                       TraceFlags(header[28]), /*is_remote*/ true);
  }
};

class GrpcTextMapCarrier : public opentelemetry::context::TextMapCarrier {
 public:
  GrpcTextMapCarrier(grpc_metadata_batch* metadata) : metadata_(metadata) {}

  nostd::string_view Get(nostd::string_view key) {
    if (key == "grpc-trace-bin") {
      return absl::Base64Escape(metadata_->GetStringValue(key).value_or(""));
    } else if (absl::EndsWith(key, "-bin")) {
      // Maybe ok to support a custom binary propagator. Needs based64 encoding 
      // validation if so. Not for now.
      gpr_log(GPR_ERROR, "Binary propagator other than GrpcTraceBinPropagator is not supported.");
      return "";
    }
    return metadata_->GetStringValue(key);
  }

  void Set(nostd::string_view key, nostd::string_view value) {
    if (key == "grpc-trace-bin") {
      metadata_->Set(
          grpc_core::GrpcTraceBinMetadata(),
          grpc_core::Slice::FromCopiedString(absl::Base64Unescape(value)));
    } else if (absl::EndsWith(key, "-bin")) {
      gpr_log(GPR_ERROR, "Binary propagator other than GrpcTraceBinPropagator is not supported.");
      return;
    } else {
      // A propagator other than GrpcTraceBinTextMapPropagator was used.
      metadata_->Append(key, grpc_core::Slice::FromCopiedString(value));
    }
  }

 private:
  grpc_metadata_batch* metadata_;
};

}  // namespace internal

std::unique_ptr<opentelemetry::context::TextMapPropagator>
MakeGrpcTraceBinTextMapPropagator() {
  return std::make_unique<internal::GrpcTraceBinTextMapPropagator>();
}

}  // namespace grpc

```

With gRPC OpenTelemetry API, users can provide a single composite propagator that 
combines one or multiple `TextMapPropagator` for their client and server separately.
OpenTelemetry and its extension packages support multiple text map propagators.
gRPC puts all the propagator data into the wire through metadata, and receives all the
data specified from the propagator configuration.
Users can define their own migration path for context propagators in distributed components.
Configuring gRPC OpenTelemetry with this propagator when dealing with 
cross-process concerns during migration is straightforward and recommended. 
In the long term, community standardized propagators, e.g. W3C is more encouraged than `GrpcTraceBinPropagator`.
This also allows users to easily migrate a group of applications with an old propagator to 
a new propagator. An example migration path can be:
1. Configure server to accept both old and new propagators.
2. Configure the client with the desired new propagators and to drop the old propagator.
3. Make the server only accept the new propagators and complete the migration.

### Migration to OpenTelemetry: In Binary
The OpenCensus [shim](https://github.com/open-telemetry/opentelemetry-java/tree/main/opencensus-shim)
(currently available in Java, Go, Python) allows binaries that have a mix of 
OpenTelemetry and OpenCensus dependencies to export trace spans from both 
frameworks, and keep the correct parent-child relationship. This is the 
recommended approach to migrate to OpenTelemetry incrementally within a single binary. 
Note that the in-binary migration and cross-process migration can be done in parallel.

The shim package that bridges two libraries works as follows, considering the
following migration scenarios example: 

```agsl
|-- Application - Configured OpenCensus ------------------------------- |
|--  gRPC -> Using OpenCensus to generate Trace A  -------------------- |
|--  Application -> Using OpenCensus to generate a sub Trace B--------- |
```

The application may use a bridge package in the outermost layer first:
```agsl
|-- Application - Configured Otel w/ OpenCensus Shim ------------------- |
|--  gRPC -> Using OpenCensus to generate Trace A  --------------------- |
|--  Application -> Using OpenCensus to generate a sub Trace B---------- |
```

Then the application changes the instrumentation to OpenTelemetry:
```agsl
|-- Application - Configured Otel w/ OpenCensus Shim ---------------------- |
|--  gRPC -> Using OpenCensus to generate Trace A  -------------------------|
|--  Application -> Using Otel to generate a sub Trace B------------------- |
```

Finally, they switch to grpc-open-telemetry and finish the migration.
```agsl
|-- Application - Configured Otel standalone ----------------------------- |
|--  gRPC -> Using Otel to generate Trace A  ----------------------------- |
|--  Application -> Using Otel to generate a sub Trace B------------------ |
```

### OpenCensus vs OpenTelemetry Tracing Information Mapping
gRPC is generating similar tracing information for OpenTelemetry compared with OpenCensus,
but due to API differences between those two libraries, the
trace information is represented slightly differently.
In the new OpenTelemetry plugin, the client will add `Event`s (name:
`Outbound message sent` and `Inbound message read`) with corresponding attributes,
mapped from OpenCensus `MessageEvent` fields:

| OpenCensus Trace Message Event Fields | OpenTelemetry Trace Event Attribute Key |
|---------------------------------------|-----------------------------------------|
| `Type`                                | `message.event.type`                    |
| `Message Id`                          | `message.message.id`                    |
| `Uncompressed message size`           | `message.event.size.uncompressed`       |
| `Compressed message size`             | `message.event.size.compressed`         |

OpenCensus span annotation description maps to OpenTelemetry event name, and
annotation attributes keys are mapped to event attributes keys:

| OpenCensus Trace Annotation Attribute Key | OpenTelemetry Trace Event Attribute Key |
|-------------------------------------------|-----------------------------------------|
| `type`                                    | `message.event.type`                    |
| `id`                                      | `message.message.id`                    |

## Rationale
C++ will not have the optimization path in its `GrpcTraceBinPropagator` API. We
considered to have an API that enables adding `grpc-trace-bin` to the metadata 
directly, without using the propagators API. This will be a faster way that 
avoids paying for the performance cost due to
string/binary encoding between the propagator and the getter/setter.
The two APIs C++ will support for the context propagation are:
* If `GrpcTraceBinPropagator` is configured, take a slower path in the pseudocode
  described above.
* If explicitly configured, gRPC will directly use `Metadata.get()` and `Metadata.put()`
  APIs on the `grpc-trace-bin` header. No TextMapPropagator API and TextMapSetter/Getter
  will be involved. This is a faster path and mitigates performance concerns due
  to base64 encoding.

Alternatively, we can enable the fast path within C++ `GrpcTraceBinPropagator` 
instead of explicitly configure on the OpenTelemetry plugin. However, for the initial
implementation we don't have any fast path support. We leave it for the future when there
are use cases or performance concerns users may have.

## Implementation
Will be implemented in Java, C++, Go and Python.

[L29]: L29-cpp-opencensus-filter.md
[A45]: A45-retry-stats.md
[A66]: A66-otel-stats.md
