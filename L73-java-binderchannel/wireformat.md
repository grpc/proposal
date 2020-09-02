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
        transaction data for flow control.
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
major version = int;
minor version = int;
protocol extension flags = int;
version extras = minor version, protocol extension flags
num bytes = int;
ping id = int;
version info = major version, [version extras] (* If major version is at least 2. *)


setup transport transaction = version info, binder;
shutdown transport transaction = nothing;
acknowledge bytes transaction = num bytes;
ping transaction = ping id;
ping response transaction = ping id;
```

The setup transaction sends version information as a major version followed by
optional extras, which are only includede if the major version is at
least 2.

The extras are a minor version number, and a set of protocol extension flags.

During transport setup, the client transport should provide the set of
extension flags it supports, and the server should respond with the intersecion
of those extensions _it_ supports. From that point those protocol extensions
may be used.

Currenly none of these flags are in use, so they're always zero.

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

message data = parcelable | bytes data (* parcelable iff
                                          FLAG_MESSAGE_DATA_IS_PARCELABLE
                                          set, bytes otherwise *)
client rpc data = [client prefix] (* if FLAG_PREFIX is set *)
                  [message data]  (* if FLAG_MESSAGE_DATA is set *)
                  [client suffix] (* if FLAG_SUFFIX is set *)
server rpc data = [server prefix] (* if FLAG_PREFIX is set *)
                  [message data]  (* if FLAG_MESSAGE_DATA is set *)
                  [server suffix] (* if FLAG_SUFFIX is set *)
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
*   status - If a status is included in the data, the status code will be
    present in the top 16 bits of the flags value.

To allow for potential additions to the protocol, any any unrecognized flags may
be ignored. However, since a set flag can indicate the presence of following data
in the parcel, flags are implicitly ordered by increasing bit value, and an implementation
may never ignore a bit with a lower numerical value than a bit it does not ignore.

#### Sequence number

The sequence number is used to detect dropped, or out of sequence transactions.
It’s always 0 on the first transaction of a stream, and increments with each
transaction sent. The sequence number will wrap around to 0 if more than 2^31
messages are sent. The sequence numbers of client-to-server and server-to-client
transactions are independent of each other.

## Transport Setup

The setup of a new transport occurs in the following steps.

1.  The ClientTransport first binds to the endpoint Android Service, to obtain
    an endpoint binder.
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

[Binder]: https://developer.android.com/reference/android/os/Binder
[Parcel]: https://developer.android.com/reference/android/os/Parcel
[onBind]: https://developer.android.com/reference/android/app/Service#onBind\(android.content.Intent\))

