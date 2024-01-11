# The wire format of BinderTransport.

All communication occurs via one-way binder
[transactions](https://developer.android.com/reference/android/os/Binder#transact\(int,%20android.os.Parcel,%20android.os.Parcel,%20int\))
which are made up of an integer “code”, and a [Parcel] containing arbitrary
data.

Three types of binder are involved in our implementation.

*   The _endpoint binder_, owned by the BinderServer or OnDeviceServerEndpoint
    instance, and returned from the host Android service's onBind method.
*   The _client binder_, owned by the ClientTransport implementation within the
    Channel.
*   The _server binder_, owned by the ServerTransport implementation.

A single endpoint binder exists for each BinderServer or OnDeviceServerEndpoint.
A client binder / server binder pair is created for each transport pair.

## Transaction Types

There are two types of transactions, control and stream.

### Transport-level Control Transactions

For general transport lifecycle, using one of 1000 reserved transaction codes.

*   Setup Transport
    *   Transaction code: BinderTransport.SETUP_TRANSPORT (1)
    *   From ClientTransport to host binder, and from ServerTransport to client
        binder.
*   Shutdown Transport
    *   Transaction code: BinderTransport.SHUTDOWN_TRANSPORT (2)
    *   From ClientTransport to server binder, or ServerTransport to client
        binder.
*   Acknowledge bytes.
    *   Transaction code: BinderTransport.ACKNOWLEDGE_BYTES (3)
    *   From either transport to its peer's binder. Reports reception of
        transaction data for flow control. The "num bytes" field should contain
        the sum of "[data size]" of the parcels received until now.
*   Ping
    *   Transaction code: BinderTransport.PING (4)
    *   Currently only sent from ClientTransport to server binder.
*   Ping Response:
    *   Transaction code: BinderTransport.PING_RESPONSE (5)
    *   From SeverTransport to client binder or ClientTransport to server binder,
        in response to a ping message.

If either side of a transport receives an unrecognized control transacton,
it will shutdown the transport gracefully.

#### Transaction Data

```
version = int;
protocol extension flags = int;
shutdown flags = int;
num bytes = long;
ping id = int;
initial stream receive window size = int; (* The initial receive window size for
                                             new streams created on this
                                             transport, in bytes, before any
                                             window updates are sent. *)

setup transport transaction =
    version,
    binder,
    [protocol extension flags]
    [initial stream receive window size] (* if FLAG_STREAM_FLOW_CONTROL is set *)
    ;
shutdown transport transaction = [shutdown flags];
acknowledge bytes transaction = num bytes;
ping transaction = ping id;
ping response transaction = ping id;
```

`setup transport transaction` is guaranteed to be backward-compatible and so
is safe to process even if the version number is unknown to an implementation.
During transport setup, the version sent by the client should be the largest
version it supports. The server should respond with the actual version to be
used, which will be equal to or less than the client-advertised version. The
server must not fail if the client-advertised version is higher than supported.
The server may respond with SHUTDOWN_TRANSPORT if the client-advertised version
is too low to be supported. The client may respond with SHUTDOWN_TRANSPORT if
the server-selected version is too low to be supported.

Both client and server transport may also include protocol extension flags at
the end of their setup transport transaction. See the next section for currently
defined flags. Unrecognized flags must be ignored.

`shutdown flags` is a bit field reserved for future extensions to the shutdown
transaction. Receivers must ignore flags they do not understand and current
senders must set this field to zero (since no flags have been defined yet).
This field is optional. If missing, no flags have been set.

#### Protocol Extension Flags

*   FLAG_STREAM_FLOW_CONTROL (0x1) - Indicates that a peer supports the optional
stream flow control aspect of this wire format. That is, it will respect its
peer's stream flow control window as a sender and will produce stream window
update messages as a receiver. Stream flow control will be enabled for the
transport if and only if both client and server set this flag.

### Stream Transactions

Aside from the reserved 1000 transaction codes, all others are used for streams.
When the ClientTransport creates a new Stream, it assigns an ID to use for all
transactions relating to that stream (in both directions).

IDs are allocated sequentially from FIRST_CALL_ID to LAST_CALL_ID, wrapping
around at the end. Since there are roughly 2^24 available IDs, there’s no chance
of a collision except in pathological cases. If we _do_ detect a collision
however, we will fail the new call with UNAVAILABLE, and shut down the
transport gracefully.

Each normally-completed stream transmits (in both directions) a prefix, 0 or
more blocks of message data, and a suffix. The contents of prefix and suffix
data are direction dependenet, and all three can be sent in a single
transaction.

#### Transaction Data

```
flags = int; (* A bitset of flags, and an optional status code. *)
sequence no = int; (* A transaction sequence no for the RPC *)
method ref = string; (* The full method name *)
status = [string] (* The status code is in the flags. The description is only
                     included if FLAG_STATUS_DESCRIPTION is set.  *);
count = int;
bytes data = count, [byte array]; (* byte array included iff count > 0 *)
metadata = count, { metadata key, metadata val }; (* repeated count times *)
metadata sentinel = int (* -1 = Parcelable. *)
metadata key = bytes data;
metadata value = (count | metadata sentinel), [parcelable], [byte array];

client prefix = method ref, metadata
client suffix = (* currently empty *)

server prefix = metadata (* headers *)
server suffix = status, metadata (* trailers *)

stream receive window update = int; (* A positive number of additional bytes the
                                       peer can send on this stream. )

message data = parcelable | bytes data (* parcelable iff
                                          FLAG_MESSAGE_DATA_IS_PARCELABLE
                                          set, bytes otherwise *)
client rpc data = [client prefix] (* if FLAG_PREFIX is set *)
                  [message data]  (* if FLAG_MESSAGE_DATA is set *)
                  [client suffix] (* if FLAG_SUFFIX is set *)
                  [stream receive window update] (* if FLAG_WINDOW_UPDATE set *)
server rpc data = [server prefix] (* if FLAG_PREFIX is set *)
                  [message data]  (* if FLAG_MESSAGE_DATA is set *)
                  [server suffix] (* if FLAG_SUFFIX is set *)
                  [stream receive window update] (* if FLAG_WINDOW_UPDATE set *)
out of band close data = status;
rpc data = client rpc data | server rpc data | out of band close data;

rpc transaction = flags, sequence no, rpc data;
```

#### Flag fields

*   FLAG_PREFIX (0x1) - Indicates the transaction contains the streams prefix.
*   FLAG_MESSAGE_DATA (0x2) - Indicates the transaction contains message data.
*   FLAG_SUFFIX (0x4) - Indicate the transaction contains streams suffix.
*   FLAG_OUT_OF_BAND_CLOSE (0x8) - Indicates this transaction contains out of band
    close data. (This type is only ever sent in a transaction by itself).
*   FLAG_EXPECT_SINGLE_MESSAGE (0x10) - When set with client-prefix data, provides a hint
    to the server-side transport that the client expects only a single message in response.
    (I.e. The call is UNARY or CLIENT_STREAMING). This allows the server-side transport to
    combine the response message and status in a single transaction. Should the server
    responds will multiple messages _anyway_, the hint will be ignored, and messages will be
    delivered normally.
*   FLAG_STATUS_DESCRIPTION (0x20)- Indicates the status description string is
    included.
*   FLAG_MESSAGE_DATA_IS_PARCELABLE (0x40) - Indicates the message is a parcelable
    object.
*   FLAG_MESSAGE_DATA_IS_PARTIAL (0x80) - Indicates the message is partial and
    the receiver should continue reading. When a message doesn't fit into a
    single transaction, the message will be sent in multiple transactions with
    this bit set on all but the final transaction of the message.
*   FLAG_WINDOW_UPDATE (0x100) - Indicates the transaction contains an increment
    to the sender's receive window size. Can only be set if
    FLAG_STREAM_FLOW_CONTROL was negotiated for this transport.
*   status - If a status is included in the data, the status code will be
    present in the top 16 bits of the flags value.

To allow for potential additions to the protocol, any unrecognized flags must be
ignored. However, since a set flag can indicate the presence of following data
in the parcel, flags are implicitly ordered by increasing bit value, and an
implementation may never ignore a bit with a lower numerical value than a bit
it does not ignore.

#### Sequence number

The sequence number is used to detect dropped, or out of sequence transactions.
It’s always 0 on the first transaction of a stream, and increments with each
transaction sent. The sequence number will wrap around to 0 if more than 2^31
messages are sent. The sequence numbers of client-to-server and server-to-client
transactions are independent of each other.

## Transport Setup

The setup of a new transport occurs in the following steps.

1.  The ClientTransport first binds to the endpoint Android Service, to obtain
    an endpoint binder. When binding, the client will use the intent action
    "grpc.io.action.BIND".
2.  The ClientTransport sends a _setup transport_ transaction to the endpoint
    binder, passing the client binder in the parcel, then discarding the
    reference to the endpoint binder.
3.  The server receives the _setup transport_ transaction from the client and
    creates a ServerTransport instance, passing it a reference to the client
    binder.
4.  The newly-created ServerTransport instance sends a _setup transport_
    transaction back to the client, passing the server binder in the parcel.
5.  The ClientTransport receives the server binder.
6.  The transport pair is now ready, and rpc transactions can be sent.

## Transport Lifecycle & Shutdown

Each transport implementation only connects once, with any failure causing it to
shutdown for good. Reconnection attempts and retries are left up to the
ManagedChannel which creates transports when needed. A transport can be shut
down either gracefully, or ungracefully. A graceful shutdown causes all new
calls to be rejected, but allows ongoing calls to complete normally.

By comparison, an ungraceful shutdown closes all ongoing calls, sends a
shutdown-transport transaction to the remote binder, and shuts down all binders
immediately. If the remote binder dies, or any internal failure occurs, an
ungraceful shutdown will occur. If the Android Service hosting the server is
destroyed, a graceful shutdown will occur, though if the host process also dies,
that will lead to a binder death in the client, shutting the client-side down
ungracefully. If the Android component hosting the Channel is destroyed, an
ungraceful shutdown will occur, with the status code CANCELLED. The remote
transport will be notified, cancelling all ongoing remote calls on the server
side.

## Large Messages

We limit the amount of message data sent in a single transaction to 16KB.
Messages larger than that will be sent in multiple sequential transactions, with
all but the last transaction having FLAG_MESSAGE_DATA_IS_PARTIAL set.

## Transport Flow Control

Due to Android's limited per-process transaction buffer of 1MB, BinderTransport
supports transport-level flow control, aiming to keep use of the process
transaction buffer to at most 128KB. Each stream transaction we send adds to an
internal count of unacknowledged outbound data (here the count refers to the
"data size" of the parcels). If that exceeds 128KB, we’ll stop sending
stream transactions until we receive an acknowledgement message from the transport
peer. Each transport sends an acknowledgement for each 16KB of stream transaction
received, which should avoid blocking the transport most of the time.

## Stream Flow Control

While transport flow control limits consumption of scarce buffer resources at
the Android/binder layer, stream flow control limits the amount of buffering at
the gRPC layer. The resulting back pressure lets an application adjust its rate
of message production to match the remote reader's rate of stream consumption.

### Sender Responsibilities
Each stream sender (both client and server) must keep track of the amount of
space remaining in its peer's receive window, decrementing this counter by the
`bytes data` or `parcelable` size of each `rpc transaction` it sends and
incrementing it upon receipt of a `stream receive window update`. A stream
transaction must not be sent unless there's room for its data in the peer's
window. Availability of peer receive window space should be exposed to the
sending application using the language-specific stream readiness API. 

The initial receive window size for new streams is specified independently by
each receiver in its `setup transport transaction`. While a receiver may change
the window size in its direction for an established stream (described below),
the initial window size for new streams cannot change for the life of the
transport.

### Receiver Responsibilities
A stream receiver (either client or server) must send window updates as
its application layer requests messages and frees up space in the incoming
transaction buffer. Window updates may be sent in their own stream transaction
or piggybacked on message data in the opposite direction on the same stream.
Window updates may be delayed to increase the opportunity for such piggybacking,
or simply to combine multiple window updates into a single larger one.

A receiver may increase the size of the incoming transaction buffer of an existing
stream simply by sending a window update message advertising the additional
capacity. A receiver can also reduce the size of this buffer by suppressing
window updates as incoming messages are consumed by the app. However, negative
window updates are not allowed.

Receivers must detect incoming transactions that exceed their advertised receive
window and respond by "out of band" closing the stream with canonical code
`INTERNAL`. Receivers that reduce their incoming transaction buffer size must
take care to account for window update messages they sent before the reduction.

### Backwards Compatibility
Although stream flow control is a core part of the gRPC abstraction, the
earliest revisions of this wire format unfortunately did not support it. For
compatibility with old implementations, the behavior described in this section
is optional. Support is negotiated in the setup transaction and is only enabled
for a transport if both sides advertise the FLAG_STREAM_FLOW_CONTROL protocol
extension.

[Binder]: https://developer.android.com/reference/android/os/Binder
[Parcel]: https://developer.android.com/reference/android/os/Parcel
[onBind]: https://developer.android.com/reference/android/app/Service#onBind\(android.content.Intent\)
[data size]: https://developer.android.com/reference/android/os/Parcel#dataSize()