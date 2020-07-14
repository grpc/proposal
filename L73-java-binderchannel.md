# BinderChannel for native cross-process communication on Android

*   Author(s): markb74
*   Approver:
*   Status: In Review
*   Implemented in: Java
*   Last updated: July 14, 2020
*   Discussion at: https://groups.google.com/g/grpc-io/c/RFmjrxdtwzE

## Abstract

Proposes a gRPC channel and server for cross-process communication on Android,
with an underlying transport using native Android [Binder] transactions.

## Background

While the majority of Android Apps have no need for cross-process communication,
those that do are often large and complex, with dozens or even hundreds of
contributors.

The Android [bound service] API is relatively low-level, and leaves problems
like threading, flow-control, and error-handling to the application developer.

Just as the use of gRPC for network communication significantly reduces the
burden on application developers vs. lower-level APIs (e.g. posix sockets), we
expect the same to hold for cross-process communication on Android.

Protocol buffers are also more resilient to version skew than traditional
Android [Parcelable] instances, an important consideration when calls are
cross-application.

Despite this, the use of [bound services] is necessary for the Android platform
to be aware of inter-process dependencies, and there's occasionally a need to
pass platform-defined parcelable objects (e.g. `PendingIntent`), some of which
can't simply be serialized into a byte stream.

## Proposal

We propose the creation of client and server transports that support
communication via [Binder] transactions with an Android [bound service], and
corresponding channel and server builders. This transport will support the
inclusion of Android Parcelable objects in Metadata, to allow Parcelables when
necessary, while discouraging Parcelables as a general message format.

In addition we propose the creation of an "on-device" gRPC server concept, which
is simultaneously a "binder" server and an in-process server, with a
NameResolver to choose the most efficient transport mechanism for a given client
context.

In both cases, we're unable to support channel creation via [ManagedChannelBuilder]
since a [bound service] connection requires an Android [Context] object to bind
_from_.

### A note on lifecycle

To help prevent accidental memory leaks, we allow Channels to be attached to the
lifecycle of an Android component via a [Lifecycle] instance. When the
component is destroyed, the bound Channel will be shutdown, preventing remote
processes from being anchored for longer than necessary.

Similarly, each [bound service] that exposes a gRPC Server endpoint should be a
[LifecycleService], to ensure any active transports passing through it can be
shutdown if it's destroyed by the platform.

### AndroidComponentAddress

In Android, a [ComponentName] identifies a [bound service] to which a client can
connect. We will create an `AndroidComponentAddress` that extends
`SocketAddress` to wrap the ComponentName and allow it to be returned by a
`NameResolver`.

### BinderTransport

A new transport implementation that communicates via Android binder
transactions.

Client side, a single service binding is used for each transport instance, with
the server creating a new transport for each incoming connection.

Since binding to an Android Service requires an Android [Context] object, a
channel builder which may use BinderTransport requires a [Context] to create.

Since the Android transaction buffer is a fixed-size, per-process buffer,
BinderTransport must manually apply flow control to control messages, in
addition to data messages, to limit the amount of in-flight data, holding back
messages where necessary.

To allow passing Parcelable objects between processes, a new
`ParcelableInputStream` class will be created. It will behave similarly to
[ProtoInputStream] and lazily serialize the message.

```java
class ParcelableInputStream extends InputStream {
  public static <P extends Parcelable> ParcelableInputStream<P> readFromParcel(
      Parcel parcel, ClassLoader classLoader) {...}
  public static <P extends Parcelable> ParcelableInputStream<P> forInstance(
      P value, Parcelable.Creator<P> creator) {...}
  public static <P extends Parcelable> ParcelableInputStream<P> forImmutableInstance(
      P value, Parcelable.Creator<P> creator) {...}

  // Will copy mutable instances
  public P getParcelable() {...}
  // Allows serializing without copy
  public int writeToParcel(Parcel parcel) {...}
}
```

The transport will do `instanceof` checks to notice `ParcelableInputStream`
returned by a [BinaryStreamMarshaller][] (via `Metadata.serializePartial()`).
This gRFC does not introduce any Parcelable-based service code generation and
Parcelables are hard for users to maintain with backward-compatibility, so while
it would be easy to support [MethodDescriptor.Marshaller] as well, we are
consciously deciding not to. The metadata marshallers will be created similarly
to protobuf, using:

```java
public class ParcelableUtils {
  public static <P extends Parcelable> Metadata.Key<P> metadataKey(
      String name, Parcelable.Creator<P> creator) {...}
  public static <P extends Parcelable> Metadata.Key<P> metadataKeyForImmutableType(
      String name, Parcelable.Creator<P> creator) {...}
}
```

### BinderChannelBuilder

BinderChannelBuilder is used to create a channel to a BinderServer, and takes an
AndroidComponentAddress as target. Channels can be created either globally for
the entire application, or tied to the lifecycle of one component via an
optional [Lifecycle] instance.

```java
BinderChannelBuilder.create(
      applicationContext,
      AndroidComponentAddress.forRemoteComponent("pkg", "pkg.ServiceClass"))
    .build();

BinderChannelBuilder.create(
      activity,
      activity.getLifecycle(),
      AndroidComponentAddress.forRemoteComponent("pkg", "pkg.ServiceClass"))
    .build();
```

### BinderServer

BinderServer is an implementation of [InternalServer], normally created via a
corresponding BinderServerBuilder class. Each BinderServer is intended to be
hosted within a concrete Android [LifecycleService], and creates instances of
_BinderTransport_ in response to incoming transactions to that service.

Building the server returns a supplier of IBinder, which the host Android
service should return from its [onBind] method.

```java
Supplier<IBinder> binderSupplier =
    BinderServerBuilder.forService(lifecycleService)
      .addService(myService)
      .buildAndAttachToServiceLifecycle();
```

### OnDeviceServer

Like BinderServer, OnDeviceSever is an implementation of [InternalServer],
created via a corresponding OnDeviceServerBuilder class. However, an
OnDeviceServer instance wraps two other InternalServer instances, BinderServer
(described above) and gRPC's existing InProcessServer.

This allows rpcs to use an in-process channel if the client happens to be 
in the same process.

### OnDeviceChannelBuilder

OnDeviceChannelBuilder is used to create a channel to an on-device server. This
typically deals with logical server uris of the form “ondevice://server_name”,
and relies on name resolution to produce either an InProcessSocketAddress or an
AndroidComponentAddress.

Like BinderChannelBuilder, each channel can be created either globally for the
entire application, or tied to the lifecycle of one component via a [Lifecycle]
instance.

~~~java
OnDeviceChannelBuilder.create(
        application,
        "ondevice://example")
    .build();

OnDeviceChannelBuilder.create(
        activity,
        activity.getLifecycle(),
        "ondevice://example")
    .build();
```

Creating a channel directly to an external AndroidComponentAddress is also
supported.

```java
OnDeviceChannelBuilder.create(
        activity,
        activity.getLifecycle(),
        AndroidComponentAddress.forRemoteComponent("pkg", "pkg.ServiceClass"))
    .build();
~~~

### OnDeviceNameResolverProvider

OnDeviceNameResolverProvider handles name resolution from logical names to
either in-process (InProcessSocketAddress) or cross-process
(AndroidComponentAddress) addresses. An instance is created using a Builder API
with server names being either mapped to ComponentName instances (for
out-of-process servers), or to `Supplier<Server>` for in-process servers.

The use of `Supplier<Server>` for in-process name resolution, means server
creation can happen lazily. This is particularly important since we're running
in a mobile operating system, where processes are often started from cold in
response to user invocation.

### OnDeviceServerEndpoint

OnDeviceServerEndpoint enables a concrete [bound service] to expose a
cross-process endpoint to an internal `OnDeviceServer`. An instance of
OnDeviceEndpoint holds an instance of `Binder` to be returned from the concrete
service's [onBind] method. We'll require the concrete service to be an instance
of [LifecycleService], in order to shutdown transports if that service is
destroyed by the platform.

Each OnDeviceEndpoint instance is created, via a corresponding builder, with the
the logical name of the server it's exposing. An instance of
OnDeviceNameResolverProvider is used to resolve (and potentially create) the
real Server instance in response to incoming connections, with transactions
being passed to the internal _BinderServer_.

### Security

During transport setup, both client and server transport implementations will lookup the
UID of their peer (via
[Binder.getCallingUid](https://developer.android.com/reference/android/os/Binder#getCallingUid\(\))).

An instance of the `SecurityPolicy` class decides whether any given UID can
be communicated with. The default policy is to only allow comunnication with the
same UID.

Both OnDeviceChannelBuilder and BinderChannelBuilder take a SecurityPolicy in
order to validate the connected-to server's UID.

OnDeviceServerEndpointBuilder and BinderServerBuilder take a ServerSecurityPolicy to validate
the UID of each client. ServerSecurityPolicy allows for a separate SecurityPolicy to be
set for each service name.

## Rationale

### Benefits of gRPC over traditional binder & AIDL.

*   Stateless programming model.
*   Standardized error codes.
*   Support for deadlines, retries, cancellation, streaming calls.
*   Powerful interceptor APIs.
*   Standardized telemetry collection.
*   Flow control to avoid filling the platform transaction buffer.

### Some alternatives considered.

#### Regular [bound services] with protocol buffers.

A common alternative is the use of regular bound services with AIDL, but sending
protocol buffers instead of parcelables. While this does address the problem of
version skew in the message data, it’s just one problem of many. None of the
lifecycle or connection management issues are addressed by this approach, so
they remain the application developers problem.

#### A hand-rolled RPC mechanism, using proto service definitions, but not gRPC

This approach has been prototyped, but not being an existing standard, we don’t
expect it to be as compelling as gRPC with the intended audience. Many large
applications already use gRPC to communicate with servers.

#### A higher-level gRPC channel implementation (at ClientCall/ServerCall level)

This was also prototyped, and while the direct nature of the implementation was
slightly more performant, it meant the loss of standard gRPC features. E.g.
Automatic retries, metrics collection. The conclusion was that this was a
premature optimization, and using ManagedChannel is a better choice.

### Reasons not to use gRPC on Android

#### APK Size

gRPC relies on guava and code generation for protocol buffers & stubs, and
without the use of something like proguard, this can lead to significant apk
size costs.

#### Performance

A gRPC call will always come with more overhead than a hand-rolled binder call,
though testing shows this overhead is small enough for most use cases.

## Implementation

I will implement this myself. Much of this is already working and being
productionized but some internal users.

[androidx.lifecycle]: https://developer.android.com/reference/androidx/lifecycle/package-summary
[bound service]: https://developer.android.com/guide/components/bound-services.html
[bound services]: https://developer.android.com/guide/components/bound-services.html
[Service]: https://developer.android.com/reference/android/app/Service
[Lifecycle]: https://developer.android.com/reference/android/arch/lifecycle/Lifecycle
[LifecycleService]: https://developer.android.com/reference/android/arch/lifecycle/LifecycleService
[Context]: https://developer.android.com/reference/android/content/Context
[Activity]: https://developer.android.com/reference/android/app/Activity
[Parcelable]: https://developer.android.com/reference/android/os/Parcelable
[ComponentName]: https://developer.android.com/reference/android/content/ComponentName
[InProcessTransport]: https://github.com/grpc/grpc-java/blob/master/core/src/main/java/io/grpc/inprocess/InProcessTransport.java
[Binder]: https://developer.android.com/reference/android/os/Binder
[onBind]: https://developer.android.com/reference/android/app/Service#onBind\(android.content.Intent\)
[InternalServer]: https://github.com/grpc/grpc-java/blob/v1.29.0/core/src/main/java/io/grpc/internal/InternalServer.java
[BinaryStreamMarshaller]: https://grpc.github.io/grpc-java/javadoc/io/grpc/Metadata.BinaryStreamMarshaller.html
[ProtoInputStream]: https://github.com/grpc/grpc-java/blob/v1.29.0/protobuf-lite/src/main/java/io/grpc/protobuf/lite/ProtoInputStream.java
[MethodDescriptor.Marshaller]: https://grpc.github.io/grpc-java/javadoc/io/grpc/MethodDescriptor.Marshaller.html
[ManagedChannelBuilder]: https://grpc.github.io/grpc-java/javadoc/io/grpc/ManagedChannelBuilder.html
