Surface 'wait-for-ready' setting in Python client
----
* Author(s): lidizheng
* Approver: srini100
* Status: In Review
* Implemented in: Python
* Last updated: October 16, 2018
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/Q6fgbn0RZLs

## Abstract

Surface 'wait-for-ready' configuration in gRPC Python client that will use the mechanism provided by C-Core via the initial metadata flags. The implementation should allow adding future such configurations in gRPC Python easily.

## Background

### Definition of 'wait-for-ready' semantics
> If an RPC is issued but the channel is in TRANSIENT_FAILURE or SHUTDOWN states, the RPC is unable to be transmitted promptly. By default, gRPC implementations SHOULD fail such RPCs immediately. This is known as "fail fast," but the usage of the term is historical. RPCs SHOULD NOT fail as a result of the channel being in other states (CONNECTING, READY, or IDLE).
> 
> gRPC implementations MAY provide a per-RPC option to not fail RPCs as a result of the channel being in TRANSIENT_FAILURE state. Instead, the implementation queues the RPCs until the channel is READY. This is known as "wait for ready." The RPCs SHOULD still fail before READY if there are unrelated reasons, such as the channel is SHUTDOWN or the RPC's deadline is reached.
> 
> From https://github.com/grpc/grpc/blob/master/doc/wait-for-ready.md 

### Use cases for 'wait-for-ready'

When developers spin up gRPC clients and servers at the same time, it is very like to fail first couple RPC calls due to unavailability of the server. If developers failed to prepare for this situation, the result can be catastrophic. But with 'wait-for-ready' semantics, developers can initialize the client and server in any order, especially useful in testing.

Also, developers may ensure the server is up before starting client. But in some cases like transient network failure may result in a temporary unavailability of the server. With 'wait-for-ready' semantics, those RPC calls will automatically wait until the server is ready to accept incoming requests.

### Current 'wait-for-ready' behavior

By default, the client RPC call will fail immediately if the server is unavailable.

But there is a Python-only function to simulate the 'wait-for-ready' mechanism. It is a class named `_ChannelReadyFuture` in `_utilities.py`. It will create a `Future` object to enable the client to block till the channel is ready. It works similarly to the 'wait-for-ready' mechanism with Pythonic handling approach, but it has three drawbacks:

1. Usage is different from other languages like C++, Golang, Java. It may create inconsistent usage across language and additional cognitive cost for migrating from/to Python.
2. Considered technical debt. More than 100 lines of code implementing the same function which C-Core already provided.
3. It only account for the first connection attempt. Instead, C-Core 'wait-for-ready' mechanism allows every RPC call to be blocked if server is unavailable.

Strictly speaking, the `_ChannelReadyFuture` works in most cases. Though it is fungible to 'wait-for-ready', this interface has to be maintained in the near future.

### Desired 'wait-for-ready' behavior

The desired behavior will be able to config each RPC call with different settings. Currently, Python is using keyword argument for call level configuration, so this feature will use keyword argument as well. The keyword argument mechanism also enables developers to configure certain calls with one setting in many ways (like expand a dictionary-like object every time).

In other languages, there is a few difference in implementation. They prefer to have a separate class for call configuration ('ClientContext' in C++, 'CallOptions' in Java & Go), then pass it to every RPC invocation. The new design can be equivalent in behavior.

### The 3-states switch of 'wait-for-ready' in C-Core

The implementation of 'wait-for-ready' flags is not as simple as a one bit switch. It contains two bits one is `GRPC_INITIAL_METADATA_WAIT_FOR_READY` another one is `GRPC_INITIAL_METADATA_WAIT_FOR_READY_EXPLICITLY_SET`. Although two bits can represent four states, three of them are valid.

Assume `flag` is the flag variable we are manipulating.

If the developer wants to use default value, then `flag = flag & ~WAIT_FOR_READY & ~WAIT_FOR_READY_EXPLICITLY_SET` (both 0).

If the developer doesn't want to 'wait-for-ready', then `flag = flag & ~WAIT_FOR_READY & WAIT_FOR_READY_EXPLICITLY_SET` (0, 1).

If the developer wants to 'wait-for-ready', then `flag = flag & WAIT_FOR_READY & WAIT_FOR_READY_EXPLICITLY_SET` (1, 1).

### Related Proposals: 

N/A

## Proposal

* Add an optional `wait_for_ready` variable to `MultiCallable` classes initialization methods. Default `None`, accept `bool`.
* Import initial metadata flags constants from `grpc_types.h` to `grpc.pxi`.

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
# Per RPC level with configuration for subsets
stub = ...Stub(...)

important_call_options = {'wait_for_ready': True, 'timeout': 4000}
failfast_call_options = {'wait_for_ready': False}

stub.initial_rpc_call(..., **important_call_options)
stub.important_transaction_1(..., **important_call_options)
stub.unimportant_transaction_2(..., **failfast_call_options)
stub.important_transaction_3(..., **important_call_options)
stub.unimportant_transaction_4(..., **failfast_call_options)
```

## Rationale

### Default `None`, accept `bool` value
* The `None` state defined that this value haven't been touched
* It might be useful in some scenarios that interceptors wants to read its value and make changes

### Whether to add channel level variable or not
* The channel level variable is convenient and Pythonic
* But it may create extra burden for future API evolution
* Comments supporting the decision not to support channel-level configuration:
    * > Users "pay" for the complexity even if they don't use the feature.  --Eric Anderson
    * > We should be conservative when adding APIs, as we can easily add but cannot remove APIs. --Mehrdad Afshari

### Why not integrate with `_channel_managed_call_management`?
* Currently, we have a `managed_call` mechanism which works similar to call context in other languages.
* It doesn’t work properly with flags.
* It just pass a hard-coded `0` every time
* It only be called for asynchronous calls. 
* I prefer to implement it separately, with another private member variable.

### Put the new variable in middle or tail
* If this optional separate variable is right after mandatory variables, then developers can set it right away without typing extra variable name
* But this will break if the existing program is using lazy position argument
* So, put it in the tail sacrifices a little bit convenience to exchange the stability of API

## Implementation

Please refer to PR #16919

### Testing Plan
The testing will be similar to the unit test of _ChannelReadyFuture. It will contain three parts:

1. Validate the options are set properly, and make sure the C-Core receives correct flags.
2. Validate the behavior of client failed connect to an non-existing server without 'wait-for-ready' option.
3. Validate the behavior of client successfully connect to a later-spawned server with 'wait-for-ready' option.

### 'wait-for-ready' doesn’t cover all UNAVAILABLE status.
As mentioned in the definition, during my experiments, the 'wait-for-ready' works to handle "Connect Failed" but not "Socket Closed", which means if the server crashes while handling a request, the request will still be failed without retry or wait.

This behavior makes sense if the request is not idempotent, then it is possible that the server starts to handle the request and conduct some write operation to databases already. In this special cases, developers need to solve failed requests themselves.

## Open issues

* Issue #6770
