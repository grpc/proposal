Java: Channel and Server Credentials
----
* Author(s): [Eric Anderson](https://github.com/ejona86)
* Approver: sanjaypujare
* Status: In Review {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2020-06-17
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

Develop an API that separates transport credential construction from
channel/server construction.

This is to allow:

*   implementing new transport credentials without creating a new
    channel/server type, which allows users of the API to separate concerns
*   providing stable APIs for advanced credential configuration like mutual
    TLS, without polluting the general-purpose ManagedChannelBuilder API with
    many overloads that don't apply to all transports
*   composition of credentials, like necessary for xDS where a fallback
    credential needs to be provided by the user
*   aligning with C core and Go implementations

_The design focuses on channel credentials, since they are the most complex and
most discussed._ Server credentials will mostly just mirror the channel
credentials; they suffer from most of the same problems but at a smaller
scale.

## Background

Prior to gRPC v0.12.0, C core had a "unified" `Credential` type that was
simultaneously a call and channel credential. This was the design proposed for
all languages. However, the Java implementation could not determine how such an
API could be implemented for Java with its multiple transports. The designer of
the API had too little bandwidth (it was a _very_ busy time), so Java (and Go)
developed its own API. Java used interceptors (which no other language had) and
credential-specific configuration on the ManagedChannelBuilder.

In gRPC v0.12.0, C core swapped to a “split” CallCredential and
ChannelCredential. ChannelCredentials could contain CallCredentials. At this
point it was not a priority for Java to investigate the new API; Java was happy
they didn’t use the previous API, such that they didn’t have an API migration.

gRPC Java 0.15.0 added CallCredentials to replace the interceptor-based call
credentials. At this point Java had looked into the full “split” credential
design, but decided it was too hard to implement at that moment. Java worked to
stabilize the `ManagedChannelBuilder.usePlaintext()` and
`useTransportSecurity()` APIs.

Go has split call credentials (`credentials.PerRPCCredentials`) and channel
credentials (`credentials.TransportCredentials`), but with a separate type to
combine them (`credentials.Bundle`). The split credentials significantly
predate `Bundle`.

### Related Proposals: 

N/A

## Proposal

`ChannelCredentials` will be introduced, which can be provided when
constructing a `ManagedChannelBuilder':

```java
// instead of:
ManagedChannelBuilder.forTarget("example.com")
                     .build();
ManagedChannelBuilder.forTarget("example.com").usePlaintext()
                     .build();
// the user would now:
ManagedChannel.newBuilderForTarget("example.com", new TlsChannelCredentials())
                     .build();
ManagedChannel.newBuilderForTarget("example.com", new InsecureChannelCredentials())
                     .build();
```

`newBuilderForTarget()` will iterate through the `ManagedChannelProviders`
trying to create a builder for each in turn. The first provider that succeeds
will be used. If no provider is able to handle the credentials, then
`newBuilderForTarget()` will throw.

The new ChannelCredentials API cannot be mixed with the pre-existing
`useTransportSecurity()` and `usePlaintext()` builder APIs. The old-style
methods will throw if called on a builder with a ChannelCredential provided.

Note that this new API does not allow: 1) creating a channel without a
credential and supplying the credential later and 2) changing the credential
after creating the builder. This is not expected to be much of an issue as
users typically have the security information available before creating the
builder and reusing a builder is rare, especially to change the security. 

```java
package io.grpc;

/**
 * Represents a security configuration to be used for channels. There is no
 * generic mechanism for processing arbitrary ChannelCredentials; the consumer
 * of the credential (the channel) must support each implementation explicitly
 * and separately. Consumers are not required to support all types or even all
 * possible configurations for types that are partially supported, but they
 * <em>must</em> at least fully support ChoiceChannelCredentials.
 *
 * <p>A Credential provides client identity and authenticates the server. This
 * is different from CallCredentials, which only provides client identity. They
 * can also influence types of encryption used and similar security
 * configuration.
 */
public abstract class ChannelCredentials {}

public final class TlsChannelCredentials extends ChannelCredentials {
  /* Complicated. Described separately below. */
}

/** No client identity, authentication, or encryption is to be used. */
public final class InsecureChannelCredentials extends ChannelCredentials {}

/**
  * Provides a list of ChoiceChannelCredentials, where any one may be used. The
  * credentials are in preference order.
  */
public final class ChoiceChannelCredentials extends ChannelCredentials {
  public ChoiceChannelCredentials(ChannelCredentials... creds) {...}

  public List<ChannelCredentials> getCredentialsList() { return creds; }
}
```

`ChoiceChannelCredentials` is intended to be used in cases like
`GoogleDefaultCredentials`, where TLS may be satisfactory, but there may also
be more optimal credentials if the transport is a certain type.

To allow choosing `ChannelCredentials` and `CallCredentials` simultaneously,
there will be a composite credential. If `CallCredentials` are specified on the
Channel and per-RPC, then _both_ `CallCredentials` will be used. This matches
the behavior in other languages.

```java
package io.grpc;

public final class CompositeChannelCredentials extends ChannelCredentials {
  public CompositeChannelCredentials(
    ChannelCredentials chanCreds, CallCredentials callCreds) {...}
  public ChannelCredentials getChannelCredentials() { return channelCredentials; }
  public CallCredentials getCallCredentials() { return callCredentials; }
}
```

Unlike most of the credentials, TLS may have a considerable amount of
configuration. And as time goes on, more configuration options may be provided.
It could be a severe bug if certain TLS restrictions were ignored by a channel.
To detect cases where the channel does not understand enough of the TLS
configuration, there will be a method to check if there are required features
that are not understood. If some are not understood, enough information will be
returned to produce a useful error message.

`TlsChannelCredentials` will support a no-arg constructor for the "all
defaults" configuration. For any more advanced configuration, a builder would
be used.

```java
package io.grpc;

public final class TlsChannelCredentials extends ChannelCredentials {
  /**
   * Returns an empty set if this credential can be adequately understood via
   * the features listed, otherwise returns a hint of features that are lacking
   * to understand the configuration to be used for manual debugging.
   *
   * <p>An "understood" feature does not imply the caller is able to fully
   * handle the feature. It simply means the caller understands the feature
   * enough to use the appropriate APIs to read the configuration. The caller
   * may support just a subset of a feature, in which case the caller would
   * need to look at the configuration to determine if only the supported
   * subset is used.
   *
   * <p>This method may not be as simple as a set difference. There may be
   * multiple features that can independently satisfy a piece of configuration.
   * If the configuration is incomprehensible, all such features would be
   * returned, even though only one may be necessary.
   *
   * <p>An empty set does not imply that the credentials are fully understood.
   * There may be optional configuration that can be ignored if not understood.
   */
  public Set<Feature> incomprehensible(EnumSet<Feature> understoodFeatures) {...}

  /* Other methods can be added to support various Features */

  public enum Feature {}

  public final static class Builder {
    public TlsChannelCredentials build() {...}
  }
}
```

To support very advanced use cases, Netty will provide a credential that wraps
a `ProtocolNegotiator`. This allows implementations like ALTS and XDS to use
internal APIs without forcing their users to use experimental or internal APIs,
as their users would just interact with `ChannelCredentials`.

```java
package io.grpc.netty;

final class NettyChannelCredentials extends ChannelCredentials {
  public NettyChannelCredentials(ProtocolNegotiator.Factory negotiator) {...}
  public ProtocolNegotiator.Factory getNegotiator() { return negotiator; }
}

@Internal
public final class InternalNettyChannelCredentials {
  private InternalNettyChannelCredentials() {}

  public ChannelCredentials create(InternalProtocolNegotiator.Factory negotiator)
    {...}

  /**
   * Converts credentials to a negotiator, in similar fashion as for a new
   * channel.
   *
   * @throws IllegalArgumentException if unable to convert
   */
  public InternalProtocolNegotiator.Factory toNegotiator(ChannelCredentials creds)
    {...}
}
```

### GoogleDefault ChannelCredentials Example

The API provided here is not part of the design. It is meant as a
demonstration.

```java
public final class GoogleDefaultChannelCredentials {
  private GoogleDefaultChannelCredentials() {}

  public static ChannelCredentials create() {
    return new ChoiceChannelCredentials(
        InternalNettyChannelCredentials.create(
          new AltsProtocolNegotiatorFactory()),
        new CompositeChannelCredentials(
          new TlsChannelCredentials(),
          MoreCallCredentials.from(GoogleCredentials.getApplicationDefault())));
  }
}
```

### xDS ChannelCredentials Example

The API provided here is not part of the design. It is meant as a
demonstration.

```java
public final class XdsChannelCredentials {
  private XdsChannelCredentials() {}

  /**
   * Creates credentials to be configured by xDS, falling back to other
   * credentials if no configuration is provided.
   *
   * @throws IllegalArgumentException if fallback is unable to be used
   */
  public static ChannelCredentials createWithFallback(
      ChannelCredentials fallback) {
    return new InternalNettyChannelCredentials.create(
        new XdsProtocolNegotiatorFactory(
          InternalNettyChannelCredentials.toNegotiator(fallback)));
  }
}
```

### Netty ChannelCredentials Processing Example

The API provided here is not part of the design. It is meant as a
demonstration.

```java
final class ProtocolNegotiators {
  private static final EnumSet<> understoodTlsFeatures =
      EnumSet.noneOf(TlsChannelCredentials.Feature.class);

  public static Result from(ChannelCredentials creds) {
    if (creds instanceof TlsChannelCredentials) {
      TlsChannelCredentials tlsCreds = (TlsChannelCredentials) creds;
      Set<TlsChannelCredentials.Feature> incomprehensible =
          tlsCreds.incomprehensible(understoodTlsFeatures);
      if (!incomprehensible.isEmpty()) {
        return Result.error("TLS features not understood: " + incomprehensible);
      }
      return Result.negotiator(tls());

    } else if (creds instanceof InsecureChannelCredentials) {
      return Result.negotiator(plaintext());

    } else if (creds instanceof CompositeChannelCredentials) {
      CompositeChannelCredentials compCreds = (CompositeChannelCredentials) creds;
      return from(compCreds.getChannelCredentials())
          .withCallCredentials(compCreds.getCallCredentials());

    } else if (creds instanceof NettyChannelCredentials) {
      NettyChannelCredentials nettyCreds = (NettyChannelCredentials) creds;
      return Result.negotiator(nettyCreds.getNegotiator());

    } else if (creds instanceof ChoiceChannelCredentials) {
      ChoiceChannelCredentials choiceCreds = (ChoiceChannelCredentials) creds;
      StringBuilder error = new StringBuilder();
      for (ChannelCredentials innerCreds : choiceCreds.getCredentialsList()) {
        Result result = from(innerCreds);
        if (!result.isError()) {
          return result;
        }
        error.append("; ");
        error.append(result.getError());
      }
      return Result.error(error.substring(2));

    } else {
      return Result.error(
          "Unsupported credential type: " + creds.getClass().getName());
    }
  }

  public final class Result {
    public static Result error(String error) {...}
    public static Result negotiator(ProtocolNegotiator.Factory factory) {...}
    public Result withCallCredentials(CallCredentials callCreds) {...}
  }
}
```

### ServerCredentials

Server-side will be a clone of client-side, but using the `Server` instead of
`Channel`. `ServerCredentials` cannot contain `CallCredentials`, so
`CompositeServerCredentials` will be omitted. There is also currently no known
use-case for `ChoiceServerCredentials`, so it will be omitted.


```java
package io.grpc;

public abstract class ServerCredentials {}
```

## Rationale

A channel credential API is substantially harder in Java than the other
languages as Java has multiple transports that behave radically differently.
OkHttp uses blocking I/O with a byte[]-backed Buffer whereas Netty uses
nonblocking I/O with a reference-counted, ByteBuffer-backed ByteBuf with an
event loop. Go and C use a code-based API where credentials implement an
interface and use a single I/O model. It is not practical to have one
code-based API to support both OkHttp and Netty, as it would have a
considerable amount of complexity in the implementations, be hard to optimize,
and have a wide public API surface. Instead, Java needs a data-based API, where
configuration values are communicated and each transport can have its own
implementation. A data-based API can also be used by InProcess and Binder
transports.

There needs to be a `TlsChannelCredential` which is able to work with both
OkHttp and Netty transports. This credential needs to live in io.grpc and so
cannot depend on OkHttp nor Netty. Since it is not feasible for the credential
to provide the TLS implementation, it will provide configuration. Since new
configuration will be added over time, there needs to be a way for a transport
to know that it understands the important parts of the configuration.

To support transport credentials like ALTS and TLS-offloading, there still
needs to be a way to provide implementations. However, these implementations
_can_ depend on the specific transport being used. Just the users of those
implementations should avoid depending _explicitly_ on a particular transport.

The API surface should handle generic and transport-specific credentials in the
same manner, so the user’s code should not care about the implementation
details.

It would be possible to use a versioning scheme for `TlsChannelCredentials`.
Version 1 could be all defaults. Version 2 could have file-based client
certificates and ca certificates for trust. Version 3 could include ciphers,
KeyManagers, TrustManagers, and session cache configuration. While versions
safely allow adding features in the future and is simpler than the Features
enum approach, it means a consumer that only supports a subset of features at a
version must check _all_ the features at a particular version to see if they
are set. Keeping track of which features were available at each version can be
error-prone. It's also hard to produce a user-meaningful error message, so they
are aware of which specific configuration is causing the incompatibility.

## Implementation

ejona86 will implement. TLS will initially be bare-bones, with just the
compatibility checker. Over the long-term TLS will be fleshed out with
additional configuration. ServerCredentials will be implemented shortly after
ServerCredentials.

## Open issues (if applicable)

Do we put the constructor in ManagedChannel instead of ManagedChannelBuilder?
On io.grpc.Grpc? The static methods on ManagedChannelBuilder have been a
problem, as they are exposed on all subclasses, but don’t operate within the
scope of the subclass, unless they do.

It would be very convenient to be able to share the TlsChannelCredential API on
server-side, as it is quite a wide API and _almost_ the same. The consumer
would need to be slightly different, but much of the configuration can
potentially be shared. Should we make TlsChannelCredential and
TlsServerCredential just a holder for a TlsConfiguration class? For validation,
it would need to know whether it was for server-side or client-side. It would
also have the union of all client- and server-specific configuration. See
[Netty's SslContextBuilder](
https://netty.io/4.1/api/io/netty/handler/ssl/SslContextBuilder.html)
for a view into how client and server differ.
