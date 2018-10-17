Title
----
* Author(s): lidizheng
* Approver: a11r
* Status: Draft
* Implemented in: Python
* Last updated: October 16, 2018
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

Enable gRPC Python client to utilize the “Wait For Ready” mechanism provided by C-Core, which underlying is to utilize the initial metadata flags. This mechanism can later be used to support future metadata flags.

## Background

### Definition of “Wait For Ready” semantics
> If an RPC is issued but the channel is in TRANSIENT_FAILURE or SHUTDOWN states, the RPC is unable to be transmitted promptly. By default, gRPC implementations SHOULD fail such RPCs immediately. This is known as "fail fast," but the usage of the term is historical. RPCs SHOULD NOT fail as a result of the channel being in other states (CONNECTING, READY, or IDLE).
> 
> gRPC implementations MAY provide a per-RPC option to not fail RPCs as a result of the channel being in TRANSIENT_FAILURE state. Instead, the implementation queues the RPCs until the channel is READY. This is known as "wait for ready." The RPCs SHOULD still fail before READY if there are unrelated reasons, such as the channel is SHUTDOWN or the RPC's deadline is reached.
> 
> From https://github.com/grpc/grpc/blob/master/doc/wait-for-ready.md 

### Use cases for “Wait For Ready”

When developers spin up gRPC clients and servers in the same time, it is very like to fail first couple RPC calls due to unavailability of the server. If developers failed to prepare for this situation, the result can be catastrophic. But with “Wait For Ready” semantics, developers can initialize the client and server in any order, especially useful in testing.

Also, developers may ensure the server is up before starting client. But in some cases like transient network failure may result in a temporary unavailability of the server. With “Wait For Ready” semantics, those RPC calls will automatically wait until the server is ready to accept incoming requests.

### Current "Wait For Ready" behavior

After the channel instance is instantiated, it will attempt to connect to the server. Even if the connection attempt is failed, the gRPC client won’t be blocked. If the first connection attempt failed, any gRPC requests immediately after will fail with StatusCode.UNAVAILABLE.

But there is a Python-only function to simulate the “Wait For Ready” mechanism. It is a class named `_ChannelReadyFuture` in `_utilities.py`. It will create a `Future` object to enable the client to block till the channel is ready. It works just like the “Wait For Ready” mechanism with Pythonic handling approach, but it has three drawbacks:

1. Usage is different from other languages like C++, Golang, Java. It may create inconsistency usage across language and additional cognitive cost for migrating from/to Python.
1. Considered technical debt. More than 100 lines of code implementing the same function which C-Core already provided.
1. It only account for the first connection attempt. Instead, C-Core “Wait For Ready” mechanism allows every RPC call to be blocked if server is unavailable.

Strictly speaking, the `_ChannelReadyFuture` works in most cases. And it is already used in many Google internal projects. So this interface has to be maintained in the near future.

### Desired “Wait For Ready” behavior
After setting “Wait For Ready” option to the channel construction method, the channel will automatically enable this mechanism for all the outgoing RPC calls. NOT only the first one. This usage will be consistent with other language’s implementation.

Also, by adding per RPC variable, the client can customize each RPC call to leverage the “Wait For Ready” mechanism. This can give Python a little advantage over other languages.

### The 3-states switch of “Wait For Ready” in C-Core
The implementation of “Wait For Ready” flags is not as simple as a one bit switch. It contains two bits one is `GRPC_INITIAL_METADATA_WAIT_FOR_READY` another one is `GRPC_INITIAL_METADATA_WAIT_FOR_READY_EXPLICITLY_SET`. Although two bits can represent four states, three of them are valid.

Assume `flag` is the flag variable we are manipulating.

If the developer wants to use default value, then `flag = flag & ~WAIT_FOR_READY & ~WAIT_FOR_READY_EXPLICITLY_SET` (both 0).

If the developer doesn't want to "wait for ready", then `flag = flag & ~WAIT_FOR_READY & WAIT_FOR_READY_EXPLICITLY_SET` (0, 1).

If the developer wants to "wait for ready", then `flag = flag & WAIT_FOR_READY & WAIT_FOR_READY_EXPLICITLY_SET` (1, 1).

### Related Proposals: 

N/A

## Proposal

* Add an optional `wait_for_ready` variable to `MultiCallable` classes initialization methods. Default `None`, accept `bool`.
* Per RPC level `wait_for_ready` variable can override upper level.
* Import initial metadata flags constants from `grpc_types.h` to `grpc.pxi`.
* (Suggesting) Add an optional `wait_for_ready` variable to `Channel` class initialization method. Default `None`, accept `bool`.

### DEMO snippets 

```Python
# Per RPC level
stub = ...Stub(...)

stub.initial_rpc_call(..., wait_for_ready=True)
stub.important_transaction_1(..., wait_for_ready=True)
stub.unimportant_transaction_2(...)
stub.important_transaction_3(..., wait_for_ready=True)
stub.unimportant_transaction_4(...)
# The unimportant transactions can be status report, or health check, etc.
```

```Python
# Channel level
channel = grpc.insecure_channel(..., wait_for_ready=True)
stub = ...Stub(channel)

stub.important_transaction_1(...)
stub.important_transaction_2(...)
stub.unimportant_transaction_3(...) # This will wait as well
stub.important_transaction_4(...)
```

## Rationale

### Default `None`, accept `bool` value
* The `None` state defined that this value haven't been touched
* It might be useful in some scenarios that interceptors wants to read its value and make changes

### Whether to add channel level variable or not
* The channel level variable is convenient and Pythonic
* But it may create extra burden for future API evolution
* **Please** comment about whether you feel like this design works for you or not

### Separate variable vs. Add an option
* Separate variable provides easy-to-use interface; just add an option can be done quickly and remove easily if needed.
* Another problem is currently `Channel Args` or `options` are in the form of `tuple(<key>, <value>)` which is not exactly suit for flags-like variable
* Also, the explosion of variables won't be a concern because other high level APIs won't be ready to expose in near future. Besides, optional API in Python can be ignored to keep program simple.
* That's why using a separate variable is better.

### Why not integrate with `_channel_managed_call_management`?
* Currently, we have a `managed_call` mechanism which works similar to call context in other languages.
* It doesn’t work properly with flags.
* It just pass a hardcoded `0` every time
* It only be called for asynchronous calls. 
* I prefer to implement it separately, with another private member variable.

### Put the new variable in middle or tail
* If this optional separate variable is right after mandatory variables, then developers can set it right away without typing extra variable name
* But this will break if the existing program is using lazy position argument
* So, put it in the tail sacrifices a little bit convenience to exchange the stability of API

## Implementation

Please refer to PR #16915

### Testing Plan
The testing will be similar to the unit test of _ChannelReadyFuture. It will contain three parts:

1. Validate the options are set properly, and make sure the C-Core receives correct flags.
2. Validate the behavior of client failed connect to an unexisting server without “Wait For Ready” option.
3. Validate the behavior of client successfully connect to a later-spawned server with “Wait For Ready” option.

### “Wait For Ready” doesn’t cover all UNAVAILABLE status.
As mentioned in the definition, during my experiments, the “Wait For Ready” works to handle “Connect Failed” but not “Socket Closed”, which means if the server crashes while handling a request, the request will still be failed without retry or wait.

This behavior makes sense if the request is not idempotent, then it is possible that the server starts to handle the request and conduct some write operation to databases already. In this special cases, developers need to solve failed requests themselves.

## Open issues

* Issue #6770
