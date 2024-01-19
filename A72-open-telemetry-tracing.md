A72: OpenTelemetry Tracing 
----
* Author(s): [Yifei Zhuang](https://github.com/YifeiZhuang), [Yash Tibrewal](https://github.com/yashykt)
* Approver: [Eric Anderson](https://github.com/ejona86)
* Reviewers: [Mark Roth](https://github.com/markdroth), [Doug Fawley](https://github.com/dfawley),
[Feng Li](https://github.com/fengli79)
* Status: Ready for Implementation
* Implemented in: <language, ...>
* Last updated: 2024-01
* Discussion at: https://groups.google.com/g/grpc-io/c/e_ByaRmtJak

## Abstract
This proposal adds support for OpenTelemetry tracing and suggests migration
paths away from OpenCensus tracing. Discussions include:
* Context propagation between a gRPC client and server during migration.
* Migrate a gRPC involved software binary that has both OpenTelemetry and 
OpenCensus dependency.
* The API surface to enable and configure OpenTelemetry tracing.

Note that stats and logging are out of scope.

## Background
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

The following tracing information during an RPC lifecycle should be captured:

At the client, on parent span:
* When the call is started, annotate name resolution completed if the RPC had 
name resolution delay.
* When the call is closed, end the parent span with RPC status.

On attempt span:
* When span is created, add attribute `previous-rpc-attempts` that captures the 
number of preceding attempts for the RPC, and attribute `transparent-retry` that 
shows whether stream is a transparent retry.
* When the stream is created on transport, annotate delayed load balancer pick 
complete, if any.
* When an outbound message has been sent, add message events to capture seq no.,
type(SENT), uncompressed message size, and compressed message size if any 
compression. The seq no. is a sequence of integer numbers starting from 0 
to identify sent messages within the stream. The size is the total attempt message 
bytes without encryption, not including grpc or transport framing bytes.
* When an inbound message has been received from the transport, add message 
events to capture seq no., type(Received), wire message size, and uncompressed
message size if any decompression. The seq no. is a sequence of integer numbers 
starting from 0 to identify received messages within the stream.
* When the stream is closed, end the attempt span with RPC status.

At the server:
* When an outbound message has been sent, add message events to capture seq no.,
type(SENT) and uncompressed message size, and compressed message size if any compression.
* When an inbound message has been read from the transport, add message events
to capture seq no., type(Received), wire message size, and uncompressed message size,
if any decompression.
* When the stream is closed, end the span with RPC status.

Note that C++ is missing the seq no. information. And Java has issue of reporting 
decompressed message size upon receiving messages, 
as a workaround, on the client parent span and server span:
* When the uncompressed size of some outbound data is revealed, annotate seq no.,
type(Received) and uncompressed message size.

### gRPC Census API
The APIs to enable gRPC tracing are different between languages, e.g. in 
grpc-java it is zero-configuration: as long as the grpc-census dependency exists in
the classpath, the traces are automatically generated. In C++, there is an API 
for users to call to enable tracing. In Go, it is exposed via stream tracers.

### Related Proposals and Documents: 
* [gRFC L29: C++ API Changes for OpenCensus Integration][L29]
* [gRFC A45: Exposing OpenCensus Metrics and Tracing for gRPC retry][A45]
* [Microservices observability overview][grpc-observability-public-doc]
* [gRFC A66: OpenTelemetry Metrics][A66]

## Proposal
gRPC users depending on the grpc-census plugin have non-trivial migration paths 
to OpenTelemetry. Consider the following use cases:
1. Migrate an application binary where both OpenCensus and OpenTelemetry maybe exist
in the dependency tree. This can be the application’s own tracing code, or gRPC 
OpenCensus, or other dependencies that involve OpenCensus and/or OpenTelemetry.
2. Compatibility between a gRPC client and server as two distributed components, 
where during migration one will use OpenCensus and the other will use OpenTelemetry.

All related APIs here are experimental until OpenTelemetry metrics [gRPC A66][A66] 
is design and implementation complete.

### Tracing Function In OpenTelemetry Plugin
We will add tracing functions in grpc-open-telemetry plugin, along with OpenTelemetry 
metrics [gRPC A66][A66].
Languages will keep using the same gRPC infrastructures, e.g. interceptor and 
stream tracer to implement the feature.
We keep the grpc-census plugin to allow users who already depend on grpc-census to
continue using it for newer grpc versions.
In the new tracing function we will produce the same tracing information as we produce 
for Census. But due to API differences between OpenCensus and OpenTelemetry, the
trace information is represented slightly differently.
In the new tracing function, the client will add `Event`s (name:
`Outbound message sent` and `Inbound message read`) with corresponding attributes,
mapped from OpenCensus `MessageEvent` fields:

| OpenCensus Trace Message Event Fields | OpenTelemetry Trace Event Attribute Key |
|---------------------------------------|-----------------------------------------|
| Type                                  | `message.event.type`                    |
| Message Id                            | `message.message.id`                    |
| Uncompressed message size             | `message.event.size.uncompressed`       |
| Compressed message size               | `message.event.size.compressed`         |

OpenCensus span annotation description maps to OpenTelemetry event name, and 
annotation attributes keys are mapped to event attributes keys:

| OpenCensus Trace Annotation Attribute Key | OpenTelemetry Trace Event Attribute Key |
|-------------------------------------------|-----------------------------------------|
| `type`                                    | `message.event.type`                    |
| `id`                                      | `message.message.id`                    |

And OpenTelemetry no longer has span end options as OpenCensus does.

### Propagator Wire Format
While gRPC OpenCensus directly interacts with the metadata API, gRPC OpenTelemetry 
will use the standardized propagators API for context propagation, for the 
following benefits:
1. Full integration with OpenTelemetry APIs that is easier for users to reason about.
2. Make it possible to plugin other propagators that the community supports.
3. Flexible API that allows clean and simple migration paths to a different propagator.

As of today, OpenTelemetry propagator API only supports `TextMapPropagator`, 
that is to send string key/value pairs between the client and server, which is 
different from the binary header that gRPC currently uses. The future roadmap 
to support binary propagators at OpenTelemetry is unclear. So, gRPC will use 
propagator API in TextMap format with an optimization path (Go and Java) to work 
around the lack of binary propagator API to support `grpc-trace-bin`. In fact,
TextMap propagator does not show visible performance impact for C++, which is 
the most sensitive language to performance, based on internal micro benchmarking. 
Therefore, gRPC will only support propagating `grpc-trace-bin` in TextMap propagator.

gRPC will expose a custom `GrpcTraceBinPropagator` that implements `TextMapPropagator`.
This grpc-provided propagator still uses the `grpc-trace-bin` header for context 
propagation. When using `grpc-trace-bin` the OpenCensus spanContext and 
OpenTelemetry spanContext are identical, therefore a gRPC OpenCensus client can 
speak with a gRPC OpenTelemetry server and vice versa. Users can provide a 
single composite propagator that combines one or multiple `TextMapPropagator` 
for their client and server separately. This way, users can define their own 
migration path for context propagators in distributed components, see detailed
discussion in the later session. Configuring gRPC OpenTelemetry with this 
propagator when dealing with cross-process concerns during migration is
straightforward and recommended. In the long term, community
standardized propagators, e.g. W3C is more encouraged than `GrpcTraceBinPropagator`. 

####  Propagator API in Java/Go
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
      // Slow path for C++. gRPC C++ does not have runtime type inspection, so we 
      // encode bytes to String to comply with the TextMapSetter API. This code 
      // path is also used in the situation where GrpcTraceBinPropagator 
      // is used with a TextMapSetter externally.
      setter.set(carrier, "grpc-trace-bin", Base64.getEncoder().encodeToString(value));
    }
  }

  @Override
  public <C> Context extract(Context context, @Nullable C c, TextMapGetter<C> textMapGetter) {
    byte[] bytes;
    if (textMapGetter instanceof GrpcCommonGetter) { //Fast path for Java/Go
      bytes = ((GrpcCommonGetter) textMapGetter).getBinary((Metadata) c, "grpc-trace-bin");
    } else {
      // Slow path for C++. gRPC C++ does not have runtime type inspection, so 
      // we decode String from TextMapGetter API to bytes. This code path applies 
      // to the situation where GrpcTraceBinPropagator is used with a TextMapGetter
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
      // Slower path in C++. It shows the decoding part of the just encoded 
      // String at GrpcTraceBinPropagator.inject(). This can also be used to 
      // propagate any other binary header.
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
      // Slow path for C++: return string encoded from bytes. Later we decode to 
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

#### Context Propagation APIs in C++
C++ will also support propagator APIs to provides API uniformity among
languages. Since gRPC C++ avoids RTTI, it can not use the same optimization path
as Java/Go. This will result in an extra base64 encoding/decoding step to
satisfy `TextMapPropagator` requirement that the key/value pair be a valid HTTP
field. There are possible optimizations C++ might pursue in the future, for
example, providing an explicit knob on `GrpcTraceBinTextMapPropagator` that
assumes that this propagator is being used with gRPC and can hence skirt `TextMapPropagator` compatibility requirements.

```C++
std::unique_ptr<opentelemetry::context::TextMapCarrier>
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

std::unique_ptr<opentelemetry::context::TextMapCarrier>
MakeGrpcTraceBinTextMapPropagator() {
  return std::make_unique<internal::GrpcTraceBinTextMapPropagator>();
}

}  // namespace grpc

```

### gRPC OpenTelemetry Tracing API
This section talks about enabling and configuring OpenTelemetry tracing.
The OpenTelemetry API will coexist with the OpenCensus API. Only one 
`grpc-trace-bin` header will be sent for a single RPC as long as only one of 
OpenTelemetry or OpenCensus is enabled for the channel.

The APIs are different among languages due to different underlying infrastructures.

#### Java
In Java, it will be part of global interceptors, so that the interceptors are 
managed in a more sustainable way and user-friendly. As a prerequisite, the stream
tracer factory API will be stabilized. OpenTelemetryModule will be created with 
an OpenTelemetryAPI instance passing in for necessary configurations. 
Users can also rely on SDK autoconfig extension that configures the sdk object 
through environmental variables or Java system properties, then obtain the sdk 
object passed in to gRPC.

```Java
// Construct OpenTelemetry to be passed to gRPC OpenTelemetry module for 
// trace and metrics configurations.
SdkTracerProvider sdkTracerProvider = SdkTracerProvider
  .builder()
  .addSpanProcessor(
     BatchSpanProcessor.builder(exporter).build())
  .build();

OpenTelemetry openTelemetry = OpenTelemetrySdk.builder()
  .setTracerProvider(sdkTracerProvider)
  .setMeterProvider(...)
  .setPropagators(
     ContextPropagators.create(GrpcTraceBinPropagator.getInstance()))
  .build();

// Alternatively, use auto configuration:
// OpenTelemetry openTelemetry = 
// AutoConfiguredOpenTelemetrySdk.getOpenTelemetrySdk().

// Create a module that hosts tracing infrastructures. Should document that 
// the module implementation may change as OpenTelemetry evolves.
OpenTelemetryModule otModule = OpenTelemetryModule.getInstance(openTelemetry);

GlobalInterceptors.setInterceptors(
    Arrays.asList(otModule.getClientTracingInterceptor()),
    Arrays.asList(otModule.getServerTracerFactory()));

```

#### C++
The following new methods will be added in `OpenTelemetryPluginBuilder`.

```C++
class OpenTelemetryPluginBuilder {
 public:
  // If `SetTracerProvider()` is not called, no traces are collected.
  OpenTelemetryPluginBuilder& SetTracerProvider(
      std::shared_ptr<opentelemetry::metrics::TracerProvider> tracer_provider);
  // Set one or multiple text map propagators for span context propagation, e.g.
  // GrpcTraceBinTextMapPropagator or community standard ones like W3C, etc.
  OpenTelemetryPluginBuilder& SetTextMapPropagator(
      std::unique_ptr<opentelemetry::context::propagation::TextMapPropagator>
          text_map_propagator);
};
```

#### Go
In Go, the OpenTelemetry stream tracers and interceptors will be provided for users to install.

TODO: add Go API.

### Migrate to OpenTelemetry: Cross-process Networking Concerns
When users first introduce gRPC OpenTelemetry, for the time window when the
gRPC client and server have mixed plugins of OpenTelemetry and OpenCensus, 
it is encouraged to use `GrpcTraceBinPropagator` that propagates
`grpc-trace-bin` header for the migration. Using the same header greatly simplifies rollout.

A `grpc-trace-bin` formatter implementation for OpenTelemetry is needed in each language, 
which can be similar to the OpenCensus implementation. Go already has community 
support for that.

After migration period, users have the flexibility to switch to other propagators.
OpenTelemetry and its extension packages support multiple text map propagators. 
The gRPC OpenTelemetry API allows specifying 
multiple propagators: either public standard ones or custom propagators that 
implement the OpenTelemetry propagator API interface. The API composites the 
propagators and gRPC puts all the propagator data into the wire through metadata. 
This allows users to easily migrate a group of applications with an old propagator to 
a new propagator. An example migration path can be:
1. Configure server to accept both old and new propagators.
2. Configure the client with the desired new propagators and to drop the old propagator.
3. Make the server only accept the new propagators and complete the migration.

### Migrate to OpenTelemetry: In Binary
The OpenCensus [shim](https://github.com/open-telemetry/opentelemetry-java/tree/main/opencensus-shim)
(currently available in Java, Go, Python) allows binaries that have a mix of 
OpenTelemetry and OpenCensus dependencies to export trace spans from both 
frameworks, and keep the correct parent-child relationship. This is the 
recommended approach to migrate to OpenTelemetry in one binary gradually. 
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
[grpc-observability-public-doc]: https://cloud.google.com/stackdriver/docs/solutions/grpc
[A66]: A66-otel-stats.md