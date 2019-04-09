gRPC Objective-C Flow Control
----
* Author(s): mxyan
* Approver: psrini
* Status: In Review
* Implemented in: Objective-C
* Last updated: 2019-03-14
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/xSOtIAKSESk

## Abstract

This proposal describes design of gRPC Objective-C library's flow control feature.

## Background

As gRPC allows streaming call, flow control is an essential part of the system
which could prevent excessive amount of messages from being generated and
buffered in the system, causing memory issues. Current gRPC API (proposed and
implemented in gRFC L38) does not offer this feature to users.

### Related Proposals: 
* [L38: Proposal to A New gRPC Objective-C
  API](https://github.com/grpc/proposal/blob/master/L38-objc-api-upgrade.md)

## Proposal

### Enable/Disable flow control
The new flow control mechanism should allow new users who want to use flow
control feature to turn it on, while existing users not using flow control
should not need to worry about it. We thus provides an enable/disable switch
for a call which is defaulted at disabling flow control.

```
@interface GRPCMutableCallOptions

...

/** Enable flow control feature */
@property BOOL enableFlowControl;

...

@end
```

Both read and write flow control (described below) are enabled if the boolean's
value is YES. Note that this variable is independent of
[`GRPC\_EXPERIMENTAL\_DISABLE\_FLOW\_CONTROL`](https://github.com/grpc/grpc/blob/master/doc/environment_variables.md)
environment variable that controls behavior of gRPC core. If
`GRPC\_EXPERIMENTAL\_DISABLE\_FLOW\_CONTROL` is set, flow control may not work
properly even if `enableFlowControl` is YES.

### Write flow control
Write flow control intend to slow down user's message writes such that there's
only one pending message for each call in gRPC Objective-C layer. This is
achieved by an additional callback which informs a user that the previous
message write is complete (i.e. forwarded to gRPC core) and a new message can
be accepted.

```
@protocol GRPCResponseHandler<NSCopying>

...

/** Notify that the call has sent the message to gRPC core */
- (void)didWriteData;

...

@end

@interface GRPCCall2

...

- (void)writeData:(NSData *)data;

...

@end
```
A user that enabled flow control should wait for issuance of `didWriteData`
callback before making another `writeData` call. However, if `writeData` are
called multiple times, e.g. due to a race to write messages, gRPC will still
send all of the writes in order and issue `didWriteData` for each invoke of
`writeData`. However in this case users assume their own responsibility of
memory issue, e.g. it may be best to keep track of the number of pending
`writeData` calls without `didWriteData` callback. There is no API to get these
values from gRPC in such case. When flow control is not enabled, the callback
`didWriteData` is not issued to the user.

### Read flow control
For read flow control, a method `receiveNextMessage` is provided to the users
to indicate that they are ready to receive another message. It will trigger the
gRPC Objective-C layer to retrieve a message with gRPC core with RECV_MESSAGE
operation. If more than one `receiveNextMessage` is called before a message is
forwarded to user, they stack up and allows more consecutive reads of messages.
If flow control is not enabled, messages will be automatically read and
forwarded to the users.

```
@interface GRPCCall2

...

- (void)receiveNextMessage;

...

@end
```

## Rationale

### Write flow control performance
The write flow control implies only one message be buffered in gRPC Objective-C
layer, and the `didWriteData` callback is issued to another thread. So a
context switch happens for each write of message. On mobile platform this may
not be a big issue, but in case some users do send a large amount of small
messages and find the performance poorer than their expectation, the issue may
be resolved in either of the following ways:
* since gRPC does not put a hard restriction on how many writes can be
  pending for each call, users may have more than 1 pending writes for each
  call in gRPC Objective-C layer and write their own flow control logic based
  on `didWriteData` callbacks;
* gRPC Objective-C layer may detect the size of message at `writeData` function
  and immediately issue back a `didWriteData` callback if the message size is
  small.

### Write flow control API
We considered putting a hard limit on the write flow control such that when
more than one message is pending in gRPC Objective-C layer, an error is
returned to the user.  However, the benefit of doing so is not very clear, and
it eliminates the possibilities of providing users flexibility have more
flexible flow control policy themselves as described in the previous
subsection. Therefore we decide to not put such a hard limit.

### Read flow control API
An alternative read flow control API could be:
```

- (void)receiveNextMessages:(UInteger)numberOfMessages;

```
The effect would be the same as calling `receiveNextMessage:` multiple times.

## Implementation
The feature involves minor changes in `GRPCStreamingProtoCall` and `GRPCCall2`
interfaces, and medium size change in the implementation of `GRPCCall2`. The
major implementation change is 1) to propagate SEND_MESSAGE operation
completion back to the `GRPCCall` interface, and 2) to not automatically issue
READ_MESSAGE operation to gRPC core but wait for it be triggered by a call to
`receiveNextMessage:` method.

The feature will be implemented by @muxi.
