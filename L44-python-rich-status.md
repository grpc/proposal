New RPC Status API for Python
----
* Author(s): lidizheng
* Approver: a11r
* Status: Draft
* Implemented in: Python
* Last updated: 12-10-2018
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/8Ys6ba8gpLc

## Abstract

The current design of status API doesn't support rich status. Also the concept of status "details" doesn't comply with the gRPC spec. So, after discussion with @gnossen and @ericgribkoff, we think we should add a new set of status APIs to correct them.


## Background

The gRPC [Spec](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md#responses) defined two trailing data to indicate the status of an RPC call `Status` and `Status-Message`. However, status code and a text message themselves are insufficient to express more complicated situation like the RPC call failed because of quota check of specific policy, or the stack trace from the crashed server, or obtain retry delay from the server (see [error details](https://github.com/googleapis/api-common-protos/blob/master/google/rpc/error_details.proto)).

The one primary use case of this feature is Google Cloud API. The Google Cloud API needs it to transport rich status from the server to the client. Also, they are the owner of those proto files.

So, the gRPC team introduced a new internal trailing metadata entry `grpc-status-details-bin` to serve this purpose. It takes in serialized proto `google.rpc.status.Status` message binary and transmits as binary metadata. However, since gRPC is a ProtoBuf-agnostic framework, it can't enforce either the content of the binary data be set in the trailing metadata entry, nor the consistency of status code/status message with the additional status detail proto message. The result of this unsolvable problem is that for three major implementations of gRPC we have, each of them is slightly different about this behavior.

### Behavior Different Across Implementations

C++ implementation tolerates anything to be put in the `grpc-status-details-bin` entry; Java checks if the code of the proto status is matching the status code of the RPC call; Golang enforces both code and message of the proto status, and the RPC call be the same.

So, which paradigm should Python follow?

### The Proto of Status

The proto of status is well-defined and used in many frameworks. Here is the definition of `Status`, for full version see [status.proto](https://github.com/googleapis/api-common-protos/blob/master/google/rpc/status.proto).
```proto
message Status {
  int32 code = 1;
  string message = 2;
  repeated google.protobuf.Any details = 3;
}
```

### Current Python API

```Python
# Client side
stub = ...Stub(channel)
try:
    resp = stub.ARpcCall(...)
except grpc.RpcError as rpc_error:
    code = rpc_error.code()
    message = rpc_error.details()
    # Unable to get rich status
```

```Python
# Server side
def ...Servicer(...):
    ...
    def ARPCCall(request, context):
        ...
        context.set_code(...)
        context.set_details(...)
        ...

```

## Proposal

### A New Public Interface: Status

This class is used to describe the status of an RPC.

The name of the class is `Status`, because it is shorter and cleaner than `RpcStatus` or `ServerRpcStatus`. In the future, we might want to utilize it at client side as well.

The metadata field of `Status` is `trailing_metadata` instead of `metadata`. The `Status` should be the final status of an RPC, so it should assist developers to get information that only accessible at the end of the RPC. Also, if we want to support it at client side, we need to separate `initial_metadata` and `trailing_metadata`.

```Python
class Status(six.with_metaclass(abc.ABCMeta)):
    """Describes the status of an RPC.

    This is an EXPERIMENTAL API.

    Attributes:
      code: A StatusCode object to be sent to the client.
      details: An ASCII-encodable string to be sent to the client upon
        termination of the RPC.
      trailing_metadata: The trailing :term:`metadata` in the RPC.
    """
```

### Server side API

gRPC Python team encountered lots of disagreements about how this part should be implemented. Available options are listed in the `Rationale` section. Here is our final consensus as a team, but feel free to comment about other options.

During the discussion we come up with several criteria for server-side API:
1. Shouldn't pass `ServicerContext` around and let extension package to mutate it.
2. Should minimize the changes for our main package.
3. Should allow the status code/message validation at some point.
4. Should reserve the extensibility for future updates.
5. Should not confuse developers about the priority between the new API and old APIs.
6. Should be Pythonic.

As a result, we finally decided to add a new API named `abort_with_status` that accepts the new interface `grpc.Status`. Unlike `set_code`/`set_details` can be called multiple times, the abortion mechanism ensured the code, details, and metadata of the would be set as the final status of `grpc.Call`.

```Python
def abort_with_status(self, status):
    """Raises an exception to terminate the RPC with a non-OK status.

    The status passed as argument will supercede any existing status code,
    status message and trailing metadata.

    This is an EXPERIMENTAL API.

    Args:
      status: A grpc.Status object. The status code in it must not be
        StatusCode.OK.

    Raises:
      Exception: An exception is always raised to signal the abortion the
        RPC to the gRPC runtime.
    """
```

### Rich Status Usage

Besides the change to status API, there should be a new gRPC Python extension package that depends on ProtoBuf named `grpcio-status`. The new package should provide convenient functions to help developers transform from ProtoBuf instance to gRPC status. The usage should look like:

```Python
# Client side
from grpc_status import rpc_status

stub = ...Stub(channel)

try:
    resp = stub.AMethodHandler(...)
except grpc.RpcError as rpc_error:
    rich_status = rpc_status.from_rpc_error(rpc_error)
    # `rich_status` here is a ProtoBuf instance of `google.rpc.status.Status` proto message
```
```Python
# Server side
from grpc_status import rpc_status
from google.protobuf import any_pb2

def ...Servicer(...):
    def ARPCCall(request, context):
        ...
        detail = any_pb2.Any()
        detail.Pack(
            rpc_status.error_details_pb2.DebugInfo(
                stack_entries=traceback.format_stack(),
                detail="Can't recognize this argument",
            )
        )
        rich_status = grpc_status.status_pb2.Status(
            code=grpc_status.code_pb2.INVALID_ARGUMENT,
            message='API call quota depleted',
            details=[detail]
        )
        context.abort_with_status(rpc_status.to_status(rich_status))
        # The method handler will abort
```

### Optional: Client side API

Currently, the documentation about getting status resides in [grpc.Call](https://grpc.io/grpc/python/grpc.html#client-side-context) section. We propose to add a new member function named `status()`, it will return a `grpc.Status`.

The usage snippet should look like:

```Python
stub = ...Stub(channel)

try:
    resp = stub.AMethodHandler(...)
except grpc.RpcError as rpc_error:
    code = rpc_error.status().code
    details = rpc_error.status().details
    trailing_metadata = rpc_error.status().trailing_metadata
```


## Rationale

There are six more alternative options for implementing this feature.

### Option 1: A New Exception Handling Mechanism

We expose a new public exception interface that developer can raise within their servicer method handler. The exception itself contains information like status code, status message, and most importantly the rich status details. As for the extension package, we shall expose a function to help developer assemble that exception.

Current server side abortion mechanism relies on exception as well. When a developer called `context.abort(...)`, a designated exception raises. Then upstream function should catch that exception and perform an abortion for the RPC call.

PS: The custom error handler mechanism is needed for gRPC Python but unsupported yet. The correct way is to do it through fully-featured server-side interceptor in the proposal [L13](https://github.com/grpc/proposal/pull/39). However, it is never got implemented.

```Python
# Usage Snippet
import grpc_status
from google.protobuf import any_pb2

def ...Servicer(...):
    def AMethodHandler(request, context):
        ...
        detail = any_pb2.Any()
        detail.Pack(
            error_details_pb2.DebugInfo(
                stack_entries=traceback.format_stack(),
                detail="Can't recognize this argument",
            )
        )
        rich_status = grpc_status.status_pb2.Status(
            code=grpc_status.code_pb2.INVALID_ARGUMENT,
            message='API call quota depleted',
            details=[detail]
        )

        raise grpc_status.to_exception(rich_status)
```

#### Pros
* Follows the existing implementation mechanism (abort)
* Allows raising the exception from any layer of the stack, also prevents servicer context be passed around


#### Cons
* Adds a new public interface
* Adds a new exception handling mechanism (@gnossen: it's magic!)
* No rich status for the successful call


### Option 2: Server Context Status Class

In C++/Java/Golang, the status is implemented in a cohesive class that handles status-related information. Unfortunately, in Python, current design doesn't support this. And as @ericgribkoff mentioned, Python API name mixed the concept of the status message and status details. There is a `set_details` method for servicer context, but its used to set a status message. So, we presumably can reorganize that information as a server-side status class an alternative method of existing methods, and deprecate those single-value setters in future.

Also, the new interface allows gRPC Python to validate the code, message, and details provided by the developer are matched. Even with the validation, developers still able to abuse this API by providing arbitrary status details.

```Python
# Add a new interface for ExtraDetails
class ExtraDetails(abc.ABCMeta):
    @abc.abstractmethod
    def code(self): pass

    @abc.abstractmethod
    def message(self): pass

    @abc.abstractmethod
    def details(self): pass

# Usage Snippet
import grpc_status
from google.protobuf import any_pb2

def ...Servicer(...):
    def AMethodHandler(request, context):
        ...
        detail = any_pb2.Any()
        detail.Pack(
            error_details_pb2.DebugInfo(
                stack_entries=traceback.format_stack(),
                detail="Can't recognize this argument",
            )
        )
        rich_status = grpc_status.assemble_status(
            code=grpc_status.code_pb2.INVALID_ARGUMENT,
            message='API call quota depleted',
            details=[detail],
        )

        context.set_extra_status(grpc_status.convert_to_extra_details(rich_status))
        ...
```

#### Pros
* Expose the setting of status to developers like Java/Go
* Status-related information managed in one place
* Clean, plain design if `set_code`/`set_details` removed

#### Cons
* Have to add a brand new `Status` class
* Ambiguity of responsibility of the new status class
* Asymmetric with client side channel design
* Needs to educate developer about priority


### Option 3: Call Server Context Method in New Extension Package

In this design, the new extension package provides a single function `abort_with_status` that accepts the server context and status proto instance. It will set code, message and trailing metadata inside the function. The downside is developers have to pass a mutable instance to another package to change it. Also, the left semantic ambiguity about the abortion behavior during exception whether to continue abortion or not.

```Python
# Usage Snippet
import grpc_status
from google.protobuf import any_pb2

def ...Servicer(...):
    def AMethodHandler(request, context):
        ...
        detail = any_pb2.Any()
        detail.Pack(
            error_details_pb2.DebugInfo(
                stack_entries=traceback.format_stack(),
                detail="Can't recognize this argument",
            )
        )
        rich_status = grpc_status.status_pb2.Status(
            code=grpc_status.code_pb2.INVALID_ARGUMENT,
            message='API call quota depleted',
            details=[detail]
        )
        grpc_status.abort_with_status(context, rich_status)
        # An exception will be raised. RPC call abort.
```

#### Pros
* Zero code change to the main package

#### Cons
* Passing server context around (@ericgribkoff: ugly!)
* Ambiguous abortion behavior


### Option 4: Convert to Metadata

The new package handles the conversion from ProtoBuf message instance to gRPC Python metadata, which is a double-value `tuple`. It is the most hands-off option. Our main framework promised nothing to developers about the usage of the `grpc-status-details-bin`.

```Python
# Usage Snippet
import grpc_status
from google.protobuf import any_pb2

def ...Servicer(...):
    def AMethodHandler(request, context):
        ...
        detail = any_pb2.Any()
        detail.Pack(
            error_details_pb2.DebugInfo(
                stack_entries=traceback.format_stack(),
                detail="Can't recognize this argument",
            )
        )
        rich_status = grpc_status.status_pb2.Status(
            code=grpc_status.code_pb2.INVALID_ARGUMENT,
            message='API call quota depleted',
            details=[detail]
        )

        context.set_code(rich_status.code)
        context.set_details(rich_status.message)
        context.set_trailing_metadata(
            grpc_status.to_metadata(rich_status)
        )
```

#### Pros
* Zero code change to the main package

#### Cons
* Verbosity
* No code/message validation at all
* Confusing definition of message/details


### Option 5: Add Status Details Setter

Similar to option 4, this option is one-step further that the framework automatically set the string to the metadata entry `grpc-status-details-bin`.

```Python
### Client side ###
stub = ...Stub(channel)
try:
    resp = stub.AMethodHandler(...)
except grpc.RpcError as rpc_error:
    binary_status = rpc_error.binary_status()
    status = grpc_status.parse(binary_status)
    # Do stuff with the status

### Server side ###
def ...Servicer(...):
    def AMethodHandler(request, context):
        ...
        detail = any_pb2.Any()
        detail.Pack(
            error_details_pb2.DebugInfo(
                stack_entries=traceback.format_stack(),
                detail="Can't recognize this argument",
            )
        )
        rich_status = grpc_status.status_pb2.Status(
            code=grpc_status.code_pb2.INVALID_ARGUMENT,
            message='API call quota depleted',
            details=[detail]
        )

        context.set_code(rich_status.code)
        context.set_details(rich_status.message)
        context.set_binary_status(rich_status.serializeToString())
        ...
```

#### Pros
* Similar to C++'s strictness: code/message/details are independent
* Promised developers the usage of metadata entry `grpc-status-details-bin`

#### Cons
* Verbosity
* No code/message validation at all
* Confusing definition of message/details


### Option 6: Pack as Tuple & Splat if Needed

The new API will be added to [ServicerContext](https://grpc.io/grpc/python/grpc.html#service-side-context), and named `set_status`. It accepts two positional arguments and a keyword argument, so setting the status code and status message is mandatory but not details. At the same time, current API `set_code`/`set_details` will be labeled as "deprecated".

The function signature should be:

```Python
def set_status(code, message, details=""): pass
```

The usage should look like:

```Python
# Server side
from grpc_status import rpc_status
from google.protobuf import any_pb2

def ...Servicer(...):
    def ARPCCall(request, context):
        ...
        detail = any_pb2.Any()
        detail.Pack(
            rpc_status.error_details_pb2.DebugInfo(
                stack_entries=traceback.format_stack(),
                detail="Can't recognize this argument",
            )
        )
        rich_status = grpc_status.status_pb2.Status(
            code=grpc_status.code_pb2.INVALID_ARGUMENT,
            message='API call quota depleted',
            details=[detail]
        )

        context.set_status(*rpc_status.convert(rich_status))
```

#### Pros
* Concise API call
* Avoid introducing new classes

#### Cons
* Recommend using splat is not Pythonic
* Need to educate developers about the priority of those three APIs


### Why we need a new package `grpcio-status`?

Although we can't enforce the consistency of status code, status message, and status details, there is still value for the new package to help developers validate that. It demonstrates our recommendation through the design.

One more reason is that in the future, the `google.rpc.status.Status` may not be the only status proto we accept. This package provides a straightforward mapping between different status proto message.


### Not Swapping the Two Concepts

Ideally speaking, the **status message** and **status details** are misused in Python implementation, and this proposal is our best chance to get them right. According to spec, the correct definition should be as followed:

* **Status message** should be interpreted as a text (Unicode allowed) field that indicates the status of the RPC call, and it is intended to be read by the developers.
* **Status details** should be understood as a encoded `google.rpc.status.Status` proto message that serves as rich status error details about the RPC Call.

However, the **status details** is deeply bound with gRPC Python since its very first version. The concept swapping may be correct, but it presumably increases developers' recognition burden. Moreover, even if we can cleanly update our document and API interface to swap those two concepts, there are tons of tutorial/articles/examples about gRPC Python on the internet we can't ever correct them. So, we think maintaining the status quo is our current optimal solution.


## Implementation

Pull Requests:
* Part of new status API is implemented in https://github.com/grpc/grpc/pull/17481
* The new extension package is implemented in https://github.com/grpc/grpc/pull/17490

## Open issues (if applicable)

Issue: https://github.com/grpc/grpc/issues/16366
