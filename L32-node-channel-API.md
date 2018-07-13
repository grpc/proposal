Node.js Channel API
----
* Author(s): mlumish
* Approver: wenbozhu
* Status: Draft
* Implemented in: Node.js
* Last updated: 
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/VR0C7DiG7bs

## Abstract

Expose the `Channel` class in the Node API, along with APIs for overriding the channel created in the `Client` constructor.

## Background

In the past, some users have requested the ability to explicitly create channels and share them among clients. In addition, another library needs to be able to intercept some channel functionality.

## Proposal

We will add to the API a `Channel` class and two `Client` construction options.

### New `Channel` Class API

```ts
enum ConnectivityState {
    IDLE = 0,
    CONNECTING = 1,
    READY = 2,
    TRANSIENT_FAILURE = 3,
    SHUTDOWN = 4
}

class Channel {
    /**
     * This constructor API is almost identical to the Client constructor,
     * except that some of the options for the Client constructor are not valid
     * here.
     * @param target The address of the server to connect to
     * @param credentials Channel credentials to use when connecting
     * @param options A map of channel options that will be passed to the core
     */
    constructor(target: string, credentials: ChannelCredentials, options: ([key:string]: string|number));
    /**
     * Close the channel. This has the same functionality as the existing grpc.Client.prototype.close
     */
    close(): void;
    /**
     * Return the target that this channel connects to
     */
    getTarget(): string;
    /**
     * Get the channel's current connectivity state. This method is here mainly
     * because it is in the existing internal Channel class, and there isn't
     * another good place to put it.
     * @param tryToConnect If true, the channel will start connecting if it is
     *     idle. Otherwise, idle channels will only start connecting when a
     *     call starts.
     */
    getConnectivityState(tryToConnect: boolean): ConnectivityState;
    /**
     * Watch for connectivity state changes. This is also here mainly because
     * it is in the existing external Channel class.
     * @param currentState The state to watch for transitions from. This should
     *     always be populated by calling getConnectivityState immediately
     *     before.
     * @param deadline A deadline for waiting for a state change
     * @param callback Called with no error when a state change, or with an
     *     error if the deadline passes without a state change.
     */
    watchConnectivityState(currentState: ConnectivityState, deadline: Date|number, callback: (error?: Error) => void);
    /**
     * Create a call object. Call is an opaque type that is used by the Client
     * class. This function is called by the gRPC library when starting a
     * request. Implementers should return an instance of Call that is returned
     * from calling createCall on an instance of the provided Channel class.
     * @param method The full method string to request.
     * @param deadline The call deadline
     * @param host A host string override for making the request
     * @param parentCall A server call to propagate some information from
     * @param propagateFlags A bitwise combination of elements of grpc.propagate
     *     that indicates what information to propagate from parentCall.
     */
    createCall(method: string, deadline: Date|number, host: string|null, parentCall: Call|null, propagateFlags: number|null): Call;
}
```

### New `Client` Construction Options

We will add the following options to the `Client` constructor's options map:

 - `channelOverride`: a `Channel` instance. The `Client` will use this channel for communicating, and will ignore all other channel construction options.
 - `channelFactoryOverride`: A function that takes the same arguments as the `Channel` constructor and returns a `Channel` or an object that implements the `Channel` API. This uses the channel construction arguments passed to the client constructor.

## Rationale

The proposed `Channel` API closely matches the existing internal `Channel` class, with the addition of the existing internal `Call` constructor as `createCall`, which is necessary if we want to allow `channelFactoryOverride` to return wrappers or other `Channel` API implementations. The `Channel` API is also very similar in the pure JavaScript implementation, so this minimizes the difficulty of porting this new API to that library. On the other hand, the internal `Call` API is very different in the two libraries, which is why the `Call` class should be opaque.


## Implementation

I (@murgatroid99) will implement this in the Node.js library after this proposal is accepted.
