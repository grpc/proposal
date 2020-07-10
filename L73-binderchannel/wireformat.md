# The wire format of BinderTransport.

All communication occurs via one-way binder
[transactions](https://developer.android.com/reference/android/os/Binder#transact\(int,%20android.os.Parcel,%20android.os.Parcel,%20int\))
which are made up of an integer “code”, and a [Parcel] containing arbitrary
data.

Three types of binder are involved in our implementation.

*   The _endpoint binder_, owned by the OnDeviceServerEndpoint instance, and
    returned from concrete services onBind method.
*   The _client binder_, owned by the ClientTransport implementation within the
    Channel.
*   The _server binder_, owned by the ServerTransport implementation.

A single endpoint binder exists for each endpoint. A client binder / server
binder pair is created for each transport pair.

## Transaction Types

There are two types of transactions, control and stream.

### Transport-level Control Transactions

For general transport lifecycle, using one of 1000 reserved transaction codes.

*   Setup Transport
    *   Transaction code: BinderTransport.SETUP_TRANSPORT
    *   From ClientTransport to host binder, and from ServerTransport to client
        binder.
*   Shutdown Transport
    *   Transaction code: BinderTransport.SHUTDOWN_TRANSPORT
    *   From ClientTransport to server binder, or ServerTransport to client
        binder.
*   Acknowledge bytes.
    *   Transaction code: BinderTransport.ACKNOWLEDGE_BYTES
    *   From either transport to its peer's binder. Reports reception of
        transaction data for flow control.
*   Ping
    *   Transaction code: BinderTransport.PING
    *   From ClientTransport to server binder.
*   Ping Response:
    *   Transaction code: BinderTransport.PING_RESPONSE
    *   From SeverTransport to client binder, in response to a ping.

#### Transaction Data

```
version = int; (* Currently always 1 *)
num bytes = int;
ping id = int;

setup transport transaction = version, binder;
shutdown transport transaction = nothing;
acknowledge bytes transaction = num bytes;
ping transaction = ping id;
ping response transaction = ping id;
```

### Stream Transactions

Aside from the reserved 1000 transaction codes, all others are used for streams.
When the ClientTransport creates a new Stream, it assigns an ID to use for all
transactions relating to that stream (in both directions).

IDs are allocated sequentially from FIRST_CALL_ID to LAST_CALL_ID, wrapping
around at the end. Since there are roughly 2^24 available IDs, there’s no chance
of a collision except in pathological cases. If we _do_ detect a collision
however, we immediately shut down the transport with an INTERNAL error code.

Each normally-completed stream transmits (in both directions) a prefix, 0 or
more blocks of message data, and a suffix. The contents of prefix and suffix
data are direction dependenet, and all three may can be sent in a single
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
metadata sentinel = int (* -1 = Parcelable, -2 = bad metadata. *)
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

*   FLAG_PREFIX - Indicates the transaction contains the streams prefix.
*   FLAG_MESSAGE_DATA - Indicates the transaction contains message data.
*   FLAG_SUFFIX - Indicate the transaction contains streams suffix.
*   FLAG_OUT_OF_BAND_CLOSE - Indicates this transaction contains out of band
    close data. (This type is only ever sent in a transaction by itself).
*   FLAG_MESSAGE_DATA_IS_PARCELABLE - Indicates the message is a parcelable
    object.
*   FLAG_STATUS_DESCRIPTION - Indicates the status description string is
    included.
*   status - If a status is included in the data, the status code will be
    present in the top 16 bits of the flags value.

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
2.  The ClientTransport sends a _setup transport_ transaction to the host
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
ungraceful shutdown will occur, with the status code ABORTED. The remote
transport will be notified, cancelling all ongoing remote calls on the server
side.

[Binder]: https://developer.android.com/reference/android/os/Binder
[Parcel]: https://developer.android.com/reference/android/os/Parcel
[onBind]: https://developer.android.com/reference/android/app/Service#onBind\(android.content.Intent\))

