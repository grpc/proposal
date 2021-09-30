# Native cross-process communication on Android using BinderChannel

*   Author(s): [Ming-Chuan Lin](https://github.com/sifmelcara),
    [Ta-Wei Tu](https://github.com/taweitu),
    [Nanping Jiang](https://github.com/napathome)
*   Approver: TBD
*   Status: Draft
*   Implemented in: gRPC-core, C++
*   Last updated: September 30, 2021
*   Discussion at: https://groups.google.com/g/grpc-io/c/n8Qhlfwd8tE

## Abstract

Implements [BinderChannel protocol](proposal L73) in gRPC-core, and provide C++
interface for Android native code.

## Background

Secure cross-process gRPC communication on Android was made possible by
[BinderChannel protocol]. However, it is only supported in gRPC Java.

For various reasons (performance, code reuse, etc.) people sometimes have to
write their code in C++. It would be nice if BinderChannel servers and clients
are also supported in C++, as this would increase the interoperability between
Java and native code on Android.

Although it is possible to use [JNI] to utilize gRPC Java's implementation of
[BinderChannel protocol], it incurs significant overhead when JNI call is
involved in every RPC call. Also [JNI] could make the codebase harder to
maintain.

## Proposal

We propose to implement [BinderChannel protocol] as a new transport in
gRPC-Core, and expose C++ APIs along with some Java helper classes to support
use scenario in Android native code.

[NdkBinder] API will be used to implement the [wireformat] specified by the
protocol.

[JNI] and Java will be used by the transport implementation to set up the
connection (since APIs for setting up connections are only available in Java).
After the connection is established, the rest can be done in pure C++.

### Client side API

Since we need more than a simple URI string to create the channel, we will have
separate APIs for creating binder transport channels.

```cpp
std::shared_ptr<grpc::Channel> CreateBinderChannel(
    void* jni_env, jobject context, const std::string& package_name,
    const std::string& class_name,
    std::shared_ptr<grpc::binder::SecurityPolicy> security_policy);

std::shared_ptr<grpc::Channel> CreateCustomBinderChannel(
    void* jni_env, jobject context, const std::string& package_name,
    const std::string& class_name,
    std::shared_ptr<grpc::binder::SecurityPolicy> security_policy,
    const grpc::ChannelArguments& args);

class SecurityPolicy {
 public:
  // returns true if the UID is authorized to connect
  virtual bool IsAuthorized(int uid) = 0;
};
```

`jni_env`: Pointer to [JNIEnv]. It is required because the transport call into
Java code to establish the connection.

`context`: The service will only be considered required by this client for as
long as the `context` exists (See [bindService] for details). This object should
be a `android.content.Context`.

`package_name` and `class_name`: These 2 arguments will be used to create a
[ComponentName], which identifies a specific service to connect to.

`security_policy`: Instance of class that can check if a given uid is authorized
to connect. We will provide some possible implementations of this class. Some
examples are (a) Allows all connection. (b) Allows the connection if signature
is the same. (c) Allows the connection if the other end's APK is signed by
Google.

`args`: `CreateCustomBinderChannel` accepts an extra argument `args`, used as
additional options for channel creation.

The API is implemented using [bindService] in Java. Upon [onServiceConnected]
called by Android, the channel's [connectivity state] will become `READY`.

Because the API's signature contains [JNI] specific type `jobject`, the API will
only be declared if the header is compiled with Android tool chain. (that is,
when `GPR_ANDROID` is defined)

### Server side API

Service implementation will be an Android [bound service] in Java, and it should
return a binder object created by our transport implementation to the client.

We will reuse existing `grpc::ServerBuilder` class and introduce a new API that
returns credentials for binder server:

```cpp
std::shared_ptr<ServerCredentials> BinderServerCredentials(
    std::shared_ptr<grpc::binder::SecurityPolicy> security_policy);
```

The following snippet sets up a gRPC server listening for incoming binder
transactions, and only allows clients that signed by the same key to connect:

```cpp
grpc::ServerBuilder server_builder;

// `jvm` is passed into SameSignatureSecurityPolicy so it can call Java
server_builder.AddListeningPort(
    "binder://example", grpc::BinderServerCredentials(
                            grpc::binder::SameSignatureSecurityPolicy(jvm)));
```

The first argument specifies a URI with a customized URI scheme "binder". The
string will later be used in Java to locate the endpoint binder object created
by gRPC server.

After `server_builder.BuildAndStart()` is called, an endpoint binder will be
created internally in the transport.

To let the client connect to the server, the service needs to return the
endpoint binder. To let the bound service implementation get the endpoint binder
easily, we provide a Java class `GrpcCppServerBuilder` will be provided.

```java
GrpcCppServerBuilder::GetEndpointBinder("binder://example")
public class GrpcCppServerBuilder {
  public static IBinder GetEndpointBinder(String uri) {
    // Call C++ to get the endpoint binder and return
    // ...
  }
}
```

The static Java method that can be used like the following to get the IBinder

```java
GrpcCppServerBuilder::GetEndpointBinder("binder://example");
```

Note that the string parameter corresponds to the first argument passed in
`grpc::ServerBuilder::AddListeningPort`.

The native library containing our transport implementation must have been loaded
by user before using this helper class. This typically will not be an issue
because loading the single native library after the application starts is
generally considered as best practice.

Under the hood we will use a shared static variable to store the mapping between
service name and endpoint binder.

## Notes

### Caveat: Connection might not be established if the thread is blocked

Due to Android's design, we will need to wait for Android to notify us when the
connection become ready. However, Android will never be able to call us if the
thread is blocked. This means the gRPC channel will never become `READY` state
if the main thread is blocked.

For example, making a synchronous RPC call on main thread while channel is in
disconnected states will cause deadlock: the RPC call will block until success,
and the connection cannot be established because the RPC call blocks the thread.

### Parcelable objects

The protocol and Java implementation allows passing Parcelable objects. We
currently don't have a plan to support it in C++. When we see a parcelable
object in `Parcel`, we will print an error message and fail the RPC call.

### Error handling

When the process hosting the service has crashed, been killed, been updated, or
recovered, we will receive a notification in our [ServiceConnection] class. We
will update [connectivity state] of the channel correspondingly. User will not
need to handle connection recovery.

Note that the API will be experimental for now and we need to experiment it in
real world use cases.

### Android API level requirement

For Java interoperability, we will require the device to have Android native API
that is currently only available in AOSP master branch.

The code will not require latest NDK to build, but the shared library
(`libbinder_ndk.so`) on device must have the API available for the
implementation to behave properly.

[BinderChannel protocol]: https://github.com/grpc/proposal/blob/master/L73-java-binderchannel.md
[JNI]: https://en.wikipedia.org/wiki/Java_Native_Interface
[NdkBinder]: https://developer.android.com/ndk/reference/group/ndk-binder
[wireformat]: https://github.com/grpc/proposal/blob/master/L73-java-binderchannel/wireformat.md
[bound service]: https://developer.android.com/guide/components/bound-services.html
[JNIEnv]: https://developer.android.com/training/articles/perf-jni
[ComponentName]: https://developer.android.com/reference/android/content/ComponentName
[bindService]: https://developer.android.com/reference/android/content/Context#bindService(android.content.Intent,%20android.content.ServiceConnection,%20int)
[onServiceConnected]: https://developer.android.com/reference/android/content/ServiceConnection#onServiceConnected(android.content.ComponentName,%20android.os.IBinder)
[connectivity state]: https://grpc.github.io/grpc/core/md_doc_connectivity-semantics-and-api.html
[ServiceConnection]: https://developer.android.com/reference/android/content/ServiceConnection
