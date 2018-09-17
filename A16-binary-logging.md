# gRPC Binary Logging

*   Author(s): Spencer Fang, Yuchen Zeng
*   Approver: markdroth
*   Status: Final
*   Implemented in:
*   Last updated: 2018-09-17
*   Discussion at: https://groups.google.com/forum/#!topic/grpc-io/URLb-bTEi1k

## Abstract

Provide binary logging functionality in gRPC.

## Background

gRPC binary logging logs RPCs in binary format. RPC logs generated from
production traffic will be useful in the following cases:

-   Troubleshooting services, finding exceptions
-   Loadtesting
-   Replaying RPCs from production

This functionality should be added in each of our supported languages. The ease
of adoption should also be considered.

## Proposal

The proposed design is adding binary logging as a built-in functionality in
gRPC. Users can turn it on/off by setting the `GRPC_BINARY_LOG_FILTER`
environment variable or a similarly named flag or option string.
Turning on binary logging should be a configuration setting and require no
code changes. The control interface should allow users to change the amount
and contents of logs.

The logged events correspond to events that may be delivered to the
application. In the cases of [hedging and
retries](./A6-client-retries.md), events from uncommitted streams are not logged.

### Log Format

gRPC logs each RPC in the format defined by proto GrpcLogEntry. Initial/trailing
metadata and messages are logged separately to match the on-the-wire
representation.

The proto of GrpcLogEntry:

```proto
syntax = "proto3";

package grpc.binarylog.v1;

import "google/protobuf/duration.proto";
import "google/protobuf/timestamp.proto";

option java_multiple_files = true;
option java_package = "io.grpc.binarylog.v1";
option java_outer_classname = "BinaryLogProto";

// Log entry we store in binary logs
message GrpcLogEntry {
  // Enumerates the type of event
  // Note the terminology is different from the RPC semantics
  // definition, but the same meaning is expressed here.
  enum EventType {
    EVENT_TYPE_UNKNOWN = 0;
    // Header sent from client to server
    EVENT_TYPE_CLIENT_HEADER = 1;
    // Header sent from server to client
    EVENT_TYPE_SERVER_HEADER = 2;
    // Message sent from client to server
    EVENT_TYPE_CLIENT_MESSAGE = 3;
    // Message sent from server to client
    EVENT_TYPE_SERVER_MESSAGE = 4;
    // A signal that client is done sending
    EVENT_TYPE_CLIENT_HALF_CLOSE = 5;
    // Trailer indicates the end of the RPC.
    // On client side, this event means a trailer was either received
    // from the network or the gRPC library locally generated a status
    // to inform the application about a failure.
    // On server side, this event means the server application requested
    // to send a trailer. Note: EVENT_TYPE_CANCEL may still arrive after
    // this due to races on server side.
    EVENT_TYPE_SERVER_TRAILER = 6;
    // A signal that the RPC is cancelled. On client side, this
    // indicates the client application requests a cancellation.
    // On server side, this indicates that cancellation was detected.
    // Note: This marks the end of the RPC. Events may arrive after
    // this due to races. For example, on client side a trailer
    // may arrive even though the application requested to cancel the RPC.
    EVENT_TYPE_CANCEL = 7;
  }

  // Enumerates the entity that generates the log entry
  enum Logger {
    LOGGER_UNKNOWN = 0;
    LOGGER_CLIENT = 1;
    LOGGER_SERVER = 2;
  }

  // The timestamp of the binary log message
  google.protobuf.Timestamp timestamp = 1;

  // Uniquely identifies a call. The value must not be 0 in order to disambiguate
  // from an unset value.
  // Each call may have several log entries, they will all have the same call_id.
  // Nothing is guaranteed about their value other than they are unique across
  // different RPCs in the same gRPC process.
  uint64 call_id = 2;

  // The entry sequence id for this call. The first GrpcLogEntry has a
  // value of 1, to disambiguate from an unset value. The purpose of
  // this field is to detect missing entries in environments where
  // durability or ordering is not guaranteed.
  uint64 sequence_id_within_call = 3;

  EventType type = 4;
  Logger logger = 5;  // One of the above Logger enum

  // The logger uses one of the following fields to record the payload,
  // according to the type of the log entry.
  oneof payload {
    ClientHeader client_header = 6;
    ServerHeader server_header = 7;
    // Used by EVENT_TYPE_CLIENT_MESSAGE, EVENT_TYPE_SERVER_MESSAGE
    Message message = 8;
    Trailer trailer = 9;
  }

  // true if payload does not represent the full message or metadata.
  bool payload_truncated = 10;

  // Peer address information, will only be recorded on the first
  // incoming event. On client side, peer is logged on
  // EVENT_TYPE_SERVER_HEADER normally or EVENT_TYPE_SERVER_TRAILER in
  // the case of trailers-only. On server side, peer is always
  // logged on EVENT_TYPE_CLIENT_HEADER.
  Address peer = 11;
};

message ClientHeader {
  // This contains only the metadata from the application.
  Metadata metadata = 1;

  // The name of the RPC method, which looks something like:
  // /<service>/<method>
  // Note the leading "/" character.
  string method_name = 2;

  // A single process may be used to run multiple virtual
  // servers with different identities.
  // The authority is the name of such a server identitiy.
  // It is typically a portion of the URI in the form of
  // <host> or <host>:<port> .
  string authority = 3;

  // the RPC timeout
  google.protobuf.Duration timeout = 4;

}

message ServerHeader {
  // This contains only the metadata from the application.
  Metadata metadata = 1;
}

message Trailer {
  // This contains only the metadata from the application.
  Metadata metadata = 1;

  // The gRPC status code.
  uint32 status_code = 2;

  // An original status message before any transport specific
  // encoding.
  string status_message = 3;

  // The value of the 'grpc-status-details-bin' metadata key. If
  // present, this is always an encoded 'google.rpc.Status' message.
  bytes status_details = 4;
}

// Message payload, used by CLIENT_MESSAGE and SERVER_MESSAGE
message Message {
  // Length of the message. It may not be the same as the length of the
  // data field, as the logging payload can be truncated or omitted.
  uint32 length = 1;
  // May be truncated or omitted.
  bytes data = 2;
}

// A list of metadata pairs, used in the payload of client header,
// server header, and server trailer.
// Implementations may omit some entries to honor the header limits
// of GRPC_BINARY_LOG_CONFIG.
//
// Header keys added by gRPC are omitted. To be more specific,
// implementations will not log the following entries, and this is
// not to be treated as a truncation:
// - entries handled by grpc that are not user visible, such as those
//   that begin with 'grpc-' (with exception of grpc-trace-bin)
//   or keys like 'lb-token'
// - transport specific entries, including but not limited to:
//   ':path', ':authority', 'content-encoding', 'user-agent', 'te', etc
// - entries added for call credentials
//
// Implementations must always log grpc-trace-bin if it is present.
// Practically speaking it will only be visible on server side because
// grpc-trace-bin is managed by low level client side mechanisms
// inaccessible from the application level. On server side, the
// header is just a normal metadata key.
// The pair will not count towards the size limit.
message Metadata {
  repeated MetadataEntry entry = 1;
}

// A metadata key value pair
message MetadataEntry {
  string key = 1;
  bytes value = 2;
}

// Address information
message Address {
  enum Type {
    TYPE_UNKNOWN = 0;
    // address is in 1.2.3.4 form
    TYPE_IPV4 = 1;
    // address is in IPv6 canonical form (RFC5952 section 4)
    // The scope is NOT included in the address string.
    TYPE_IPV6 = 2;
    // address is UDS string
    TYPE_UNIX = 3;
  };
  Type type = 1;
  string address = 2;
  // only for TYPE_IPV4 and TYPE_IPV6
  uint32 ip_port = 3;
}
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
`GRPC_BINARY_LOG_FILTER` env var or flag to a filter string can turn on
binary logging for methods whose full names (in the form of
`/<service>/<method>`) match the filter.

The format of a filter is a ','-separated list of method patterns. A pattern is
in the form of `<service>/<method>` or just a character "*". It can be
optionally followed by a "`{[h:<header_length>];[m:<message_length>]}`" string.
By default, the full header or message will be logged. If a header or message
length is given and the entry size is larger than the given length, a truncated
entry will be logged. However gRPC may choose to ignore the header limit in
order to log certain required headers.

*   If the pattern is "*", it specifies the defaults for all the services.
*   If the pattern is `<service>/*`, it specifies the defaults for all
    methods in the specified service `<service>`
*   If the pattern is `-<service>/<method>`, then the specified method
    must not be logged. The "*" wildcard must not be used for negations,
    i.e. `-<service>/*` is not supported.
*   Specifying a method while wildcarding the service is not supported, i.e.
    `*/<method>` is not supported.
*   If a pattern is malformed, the gRPC process must refuse to start up

ABNF definition of the filter pattern:

```
Full-Method-Name -> {IDL-specific service name} "/" {method name}
Service-Wildcard -> {IDL-specific service name} "/*"

Header-Config -> "h" [ ":"  *DIGIT]    ; Max length of each header that should be logged
Message-Config -> "m" [":"  *DIGIT]    ; Max length of each message that should be logged
Pattern-Config -> "{" Message-Config / ( Header-Config [";" Message-Config] ) "}"

Global-Default-Pattern -> "*"  [Pattern-Config]
Per-Service-Pattern -> Service-Wildcard [Pattern-Config]
Per-Method-Pattern -> Full-Method-Name [Pattern-Config]
Excluded-Method -> "-" Full-Method-Name

Config-String -> *1( Global-Default-Pattern ) *( "," ( Per-Service-Pattern / Per-Method-Pattern / Excluded-Method ) )
```

If the full name of a method can match several patterns in the filter string,
the most exact match will be used. For example, let's say we have a filter
string with two patterns:

```
    GRPC_BINARY_LOG_FILTER=Foo/*{h};Foo/Bar{m:256}
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
should not depend on this library, so that it could be removed without any
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
