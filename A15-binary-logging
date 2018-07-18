# gRPC Binary Logging

*   Author(s): Yuchen Zeng, Spencer Fang
*   Approver: a11r
*   Status: In Review
*   Implemented in:
*   Last updated: 07/18/2018
*   Discussion at:

## Abstract

Provide binary logging functionality in gRPC.

## Background

gRPC binary logging logs RPCs in binary format. RPC logs generated from
production traffic will be useful in the following cases:

-   Troubleshooting services, find exceptions
-   Loadtesting
-   Replay RPCs from production

This functionality should be added in each of our supported languages. The ease
of adoption should also be considered.

## Proposal

The proposed design is adding binary logging as a built-in functionality in
gRPC. Users can turn it on/off by setting the `GRPC_BINARY_LOG_FILTER`
environment variable in C. Other language implementations are free to use a
similarly named flag or option string. Turning on binary logging should be a
configuration setting and require no code changes. The control interface should
allow users to change the amount and contents of logs.

The logged events correspond to what is visible at the application
level, and as a result they exclude things like [hedging and
retries](./A6-client-retries.md).

### Log Format

gRPC logs each RPC in the format defined by proto GrpcLogEntry. Initial/trailing
metadata and messages are logged separately to match the on-the-wire
representation.

The format is intentionally flat so that it is possible to craft the
message without code generation. This allows the possibility of hand
optimized serialization if needed.

The proto of GrpcLogEntry:

```
// Log entry we store in binary logs
message GrpcLogEntry {
  // Enumerates the type of event
  enum EventType {
    EVENT_TYPE_UNKNOWN = 0;
    // Headers sent from client to server
    EVENT_TYPE_CLIENT_HEADER = 1;
    // Headers sent from server to client
    EVENT_TYPE_SERVER_HEADER = 2;
    // Message sent from client to server
    EVENT_TYPE_CLIENT_MESSAGE = 3;
    // message sent from server to client
    EVENT_TYPE_SERVER_MESSAGE = 4;
    // Signal that client is done sending
    EVENT_TYPE_CLIENT_HALF_CLOSE = 5;
    // Trailers sent from server to client.
    // On client side, this is the last network event.
    // On server side, a EVENT_TYPE_CANCEL can still happen after this.
    EVENT_TYPE_SERVER_TRAILER = 6;
    // Signal that the RPC should be cancelled
    EVENT_TYPE_CANCEL = 7;
    // A special marker to indicate that the RPC is done.
    EVENT_TYPE_END_MARKER = 8;
  }

  // Enumerates the entity that generates the log entry
  enum Logger {
    UNKNOWN_LOGGER = 0;
    CLIENT = 1;
    SERVER = 2;
  }

  EventType type = 1;
  Logger logger = 2;  // One of the above Logger enum

  // Uniquely identifies a call. Each call may have several log entries, they
  // will share the same call_id. 128 bits split into 2 64-bit parts.
  Uint128 call_id = 3;

  // The logger uses one of the following fields to record the payload,
  // according to the type of the log entry.
  oneof payload {
    // Used by EVENT_TYPE_CLIENT_HEADER, EVENT_TYPE_SERVER_HEADER and
    // EVENT_TYPE_SERVER_TRAILER. This contains only the metadata
    // from the application.
    Metadata metadata = 4;
    // Used by EVENT_TYPE_CLIENT_MESSAGE, EVENT_TYPE_SERVER_MESSAGE
    Message message = 5;
  }

  // Peer address information, will only be recorded in
  // EVENT_TYPE_CLIENT_HEADER or EVENT_TYPE_SERVER_HEADER
  Peer peer = 6;

  // true if payload does not represent the full message or metadata.
  bool truncated = 7;

  // The method name. Logged for EVENT_TYPE_CLIENT_HEADER
  string method_name = 8;

  // status_code and status_message:
  // Only present for EVENT_TYPE_SERVER_TRAILER.
  uint32 status_code = 9;

  // An original status message before any transport specific
  // encoding.
  string status_message = 10;

  // The value of the 'grpc-status-details-bin' metadata key. If
  // present, this is always an encoded 'google.rpc.Status' message.
  bytes status_details = 11;

  // the RPC timeout
  google.protobuf.Duration timeout = 12;

  // The entry sequence id for this call. The first GrpcLogEntry has a
  // value of 1, to disambiguate from an unset value. The purpose of
  // this field is to detect missing entries in environments where
  // durability or ordering is not guaranteed.
  uint32 sequence_id_within_call = 13;
};

// Message payload, used by CLIENT_MESSAGE and SERVER_MESSAGE
message Message {
  // This flag is currently used to indicate whether the payload is compressed,
  // it may contain other semantics in the future. Value of 1 indicates that the
  // binary octet sequence of Message is compressed using the mechanism declared
  // by the Message-Encoding header. A value of 0 indicates that no encoding of
  // Message bytes has occurred.
  uint32 flags = 1;
  // Length of the message. It may not be the same as the length of the
  // data field, as the logging payload can be truncated or omitted.
  uint32 length = 2;
  // May be truncated or omitted.
  bytes data = 3;
}

// A list of metadata pairs, used in the payload of CLIENT_HEADER,
// SERVER_HEADER and SERVER_TRAILER.
// Implementations may omit some entries to honor the header limits
// of GRPC_BINARY_LOG_CONFIG.
// 
// Implementations will not log the following entries, and this is
// not to be treated as a truncation:
// - entries handled by grpc that are not user visible, such as those
//   that begin with 'grpc-' or keys like 'lb-token'
// - transport specific entries, including but not limited to:
//   ':path', ':authority', 'content-encoding', 'user-agent', 'te', etc
// - entries added for call credentials
message Metadata {
  repeated MetadataEntry entry = 1;
}

// A metadata key value pair
message MetadataEntry {
  bytes key = 1;
  bytes value = 2;
}

// Peer information
message Peer {
  enum PeerType {
    UNKNOWN_PEERTYPE = 0;
    // address is the address in 1.2.3.4 form
    PEER_IPV4 = 1;
    // address the address in canonical form (RFC5952 section 4)
    // The scope is NOT included in the peer string.
    PEER_IPV6 = 2;
    // address is UDS string
    PEER_UNIX = 3;
  };
  PeerType peer_type = 1;
  string address = 3;
  // only for PEER_IPV4 and PEER_IPV6
  uint32 ip_port = 4;
}

// Used to record call_id.
message Uint128 {
  fixed64 high = 1;
  fixed64 low = 2;
};
```

A call id must be enclosed in each log record, so that the analyzer can use it
to group records generated by the same call. The generation of call id should be
separated from the log generation code, and provided as an interface. Call ids
should be unique across the calls of a log instance. A simple implementation
may be an atomically incremented integer.

### Control Interface

The control interface of gRPC binary logging should express the amount and
contents of logs clearly and directly.

gRPC binary logging is turned off by default. Setting the
`GRPC_BINARY_LOG_FILTER` environment variable to a filter string can turn on
binary logging for methods whose full names (in the form of
`/<package>.<service>/<method>`) match the filter.

The format of a filter is a ','-separated list of method patterns. A pattern is
in the form of `<package>.<service>/<method>` or just a character "*". It can be
optionally followed by a "`{[h:<header_length>],[m:<message_length>]}`" string.
By default, the full header or message will be logged. If a header or message
length is given and the entry size is larger than the given length, a truncated
entry will be logged.

*   If the pattern is "*", it specifies the defaults for all the services.
*   If the pattern is `<package>.<service>/*`, it specifies the defaults for all
    methods in the specified service `<package>.<service>`
*   If the pattern is `-<package>.<service>/<method>`, then the specified method
    must not be logged.
*   Specifying a method while wildcarding the service is not supported, i.e.
    `*/<method>` is not supported.

If the full name of a method can match several patterns in the filter string,
the most exact match will be used. For example, let's say we have a filter
string with two patterns:

```
    GRPC_BINARY_LOG_FILTER=Foo/*{h},Foo/Bar{m:256}
```

For an RPC for Foo/Bar, we will use the second pattern, because it exactly
matches the service and method name.

For an RPC for Foo/Baz, we will use the first pattern, because it provides the
default for all methods in service Foo.

Example filter strings:

*   `GRPC_BINARY_LOG_FILTER=` Nothing will be logged

*   `GRPC_BINARY_LOG_FILTER=*` All headers and messages will be fully logged.

*   `GRPC_BINARY_LOG_FILTER=*{h}` Only headers will be logged.

*   `GRPC_BINARY_LOG_FILTER=*{m:256}` Only the first 256 bytes of each message
    will be logged.

*   `GRPC_BINARY_LOG_FILTER=Foo/*` Logs every method in service Foo

*   `GRPC_BINARY_LOG_FILTER=Foo/*,-Foo/Bar` Logs every method in service Foo
    except method /Foo/Bar

*   `GRPC_BINARY_LOG_FILTER=Foo/*,Foo/Bar{m:256}` Logs the first 256 bytes of
    each message in method /Foo/Bar, logs all headers and messages in every
    other method in service Foo.

### Log generation

In the binary logging library, a filter/interceptor should be introduced in our
supported languages. This filter monitors the operations of sending and
receiving metadata/messages. At each of these operation, this newly added filter
checks the `GRPC_BINARY_LOG_FILTER` environment variable, and decides whether
the log should be generated. If the flag has been turned on, the filter should
wrap the metadata/message with some system and peer information, and then send
them to the logging interface (remote logging service on diskless servers, or
file based logging in the open source version). 

### Pluggability

gRPC binary logging should serve as a separate library, making it convenient to
be changed or even replaced. Configuration knobs should be exposed directly via
this binary logging library. The production target can remove this library
according to its need to minimize the target size.  Other parts of gRPC code
hould not depend on this library, so that it could be removed without any
trouble.

## Future Work

### Content Filter

As the RPC content may have sensitive data (e.g. encryption keys) that must not
be logged for any period of time, gRPC Binary logging should provide content
filtering APIs to help filter out these data. With the content filter, users can
enable gRPC binary logging on methods that may contain sensitive data.

Two types of filter APIs should be provided, one for headers, one for messages.
Pseudo interfaces:

```
void SetHeaderFilter(HeaderKey key, HeaderFilter filter)
void SetMessageFilter(MessageType message, MessageFilter filter)
```

HeaderFilter should accept an RPC header. It may return another header (with the
same key as input) that should be used as the payload, or return an empty object
if the original header is to be used as is.

MessageFilter should accept an proto message. It may return another proto
message (of the same type as input) that should be used as the payload, or
return an empty object if the original message is to be used as is.

The content filter may improve the usability of gRPC binary logging in
troubleshooting production services. But as service owners can fully disable
gRPC binary logging on methods that have sensitive data, the content filter is
not required at the first stage, and may be delivered later as a separate
feature.

## Appendix

### Alternatives

We have another potential way of adding the binary logging functionality in
gRPC.

*   Providing binary logging as a standalone library that is independent from
    gRPC. To enable the log generation, users will need to link in this library.
    An RPCLogger class will be provided in this library. In the application
    code, users have to call the corresponding methods in this class to log
    request/response.

The first approach is preferred, since it does not require any change in the
application code.
