New RPC Status API for Python
----
* Author(s): lidizheng
* Approver: a11r
* Status: Draft
* Implemented in: Python
* Last updated: 12-06-2018
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

Current design of status API doesn't support rich status, also the concept of status "details" doesn't comply with gRPC spec. So, after discussion with @gnossen and @ericgribkoff, we think we should add a new set of status APIs to correct them.


## Background

The gRPC [Spec](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md#responses) defined two trailing data to indicate the status of a RPC call `Status` and `Status-Message`. But status code and a text message themselves are insufficient to express more complicate situation like the RPC call failed because of quota check of certain policy, or the stack trace from the crashed server, or obtain retry delay from the server (see [error details](https://github.com/googleapis/api-common-protos/blob/master/google/rpc/error_details.proto)).

The one major use case of this feature is Google Cloud API. The Google Cloud API needs it to transport rich status from server to client, also they are the owner of those proto files.

So, the gRPC team introduced a new internal trailing metadata entry `grpc-status-details-bin` to serve this purpose. It takes in serialized proto `google.rpc.status.Status` message binary and transmits as a binary metadata. However, since gRPC is a ProtoBuf-agnostic framework, it can't enforce either the content of the binary data be set in the trailing metadata entry, nor the consistency of status code/status message with the additional status detail proto message. The result of this unsolvable problem is that for three major implementation of gRPC we have, each of them are slightly different about this behavior.

### Behavior Different Across Implementations

C++ implementation tolerates anything to be put in the `grpc-status-details-bin` entry; Java will check if the code of the proto status is matching the status code of the RPC call; Golang will enforce the both code and message of the proto status and the RPC call be the same.

So, which paradigm should Python follow?

### The Proto of Status

The proto of status is well-defined and used in numerous framework. Here is the definition of `Status`, for full version see [status.proto](https://github.com/googleapis/api-common-protos/blob/master/google/rpc/status.proto).
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

### Concept Change

**Status message** should be interpreted as a text (unicode allowed) field that indicates the status of the RPC call and it is intended to be read by the developers.

**Status details** should be understood as a encoded `google.rpc.status.Status` proto message that serves as rich status error details about the RPC Call.

### Client side API

Currently, the documentation about getting status resides in [grpc.Call](https://grpc.io/grpc/python/grpc.html#client-side-context) section. We propose to add a new member function named `status()`, it will return a `namedtuple('Status', ['code', 'message', 'details'])`.

The usage snippet will look like:

```Python
stub = ...Stub(channel)

try:
    resp = stub.AMethodHandler(...)
except grpc.RpcError as rpc_error:
    code = rpc_error.status().code
    message = rpc_error.status().message
    details = rpc_error.status().details
    # or
    code, message, details = rpc_error.status()
```

### Server side API

gRPC Python team encountered lots of disagreements about how this part should be implemented. Available options will be listed in the `Rationale` section. Here I will post the final consensus for the team, but feel free to comment about other options.

The new API will be added to [ServicerContext](https://grpc.io/grpc/python/grpc.html#service-side-context), and named `set_status`. It accept two positional arguments and a keyword argument, so setting the status code and status message are mandatory but not details. At the same time, current API `set_code`/`set_details` will be labeled as "deprecated".

The function signature will be:

```Python
def set_status(code, message, details=""): pass
```

The usage will look like:

```Python
def ...Servicer(...):
    def ARPCCall(request, context):
        ...
        context.set_status(
            grpc.StatusCode.INVALID_ARGUMENT,
            "Failed to recognize input variables",
        )
```

### Rich Status Usage

Besides this proposal, there will be a new gRPC Python extension package that depends on ProtoBuf named `grpcio-status`. The new package will provide convenient functions to help developers transform from ProtoBuf instance to gRPC status. The usage will look like:

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

        context.set_status(*rpc_status.convert(rich_status))
```


## Rationale

There are five more alternative options of implementing this feature.

### Option 1: A New Exception Handling Mechanism

We expose a new public exception interface that developer can raise within their servicer method handler. The exception itself will contain information like status code, status message, and most importantly the rich status details. As for the extension package, we shall expose a function to help developer assemble that exception.

Current server side abortion mechanism is rely on exception as well. When user called `context.abort(...)`, an designated exception will be raised. Then upstream function will catch that exception and perform abortion for the RPC call.

PS: The custom error handler mechanism is needed for gRPC Python but unsupported yet. The correct way is to do it through fully-featured server-side interceptor in proposal [L13](https://github.com/grpc/proposal/pull/39). However, it is never got implemented.

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
* Allows to raise exception from any layer of stack, also prevents servicer context be passed around


#### Cons
* Adds a new public interface
* Adds a new exception handling mechanism (@gnossen: it's magic!)
* No rich status for successful call


### Option 2: Server Context Status Class

In C++/Java/Golang, the status is implemented in a cohesive class that handles status-related information. Unfortunately, in Python, current design doesn't support this. And as @ericgribkoff mentioned, Python API name mixed the concept of status message and status details. There is a `set_details` method for servicer context, but its used to set status message. So, we presumably can reorganize those information as a server-side status class as alternative method of existing methods, and deprecate those single-value setter in future.

Also, the new interface will allow gRPC Python to validate the code, message and details provided by developer are matched. Even with the validation, developers still have the ability to abuse this API by providing arbitrary status details.

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

In this design, the new extension package provides a single function `abort_with_status` that accepts the server context and status proto instance. It will set code, message and trailing metadata inside the function. The downside is developers have to pass a mutable instance to another package to change it. Also, the semantic left ambiguity about the abortion behavior during exception whether to continue abortion or not.

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

The new package handles the conversion from ProtoBuf message instance to gRPC Python metadata, which is a double-value `tuple`. It is the most hands-off option, our main framework promised nothing to developers about the usage of the `grpc-status-details-bin`.

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


### Why we need a new package `grpcio-status`?

Although we can't enforce the consistency of status code, status message and status details, there is still value for the new package to help developers validate that. It demonstrates our recommendation through the design.

One more point is that in the future, the `google.rpc.status.Status` may not be the only status proto we accept. This package will provide easy mapping between different status proto message.

## Implementation

PR: https://github.com/grpc/grpc/pull/17384

## Open issues (if applicable)

Issue: https://github.com/grpc/grpc/issues/16366
