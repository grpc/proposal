Add Typing to gRPC Python Sync API
----
* Author(s): XuanWnag-Amos
* Approver: gnossen
* Status: Draft
* Implemented in: Python
* Last updated: 11/10/2022
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

Add type annotations to the remainder of the gRPC Python API.

## Background

[Type Hints](https://peps.python.org/pep-0484/#union-types) was introduced for a while now, we already added type annotation to python Async APIs since it has a high enough version floor. Now it's time to do the same for Sync APIs.

This proposal plans to add type annotation to python Sync public APIs which allows users to build safer code by running static type analysis tools.


### Related Proposals: 
* Loosely related to [Async API proposal](https://github.com/lidizheng/proposal/blob/grpc-python-async-api/L58-python-async-api.md#introduce-typing-to-generated-code).
  * The proposal includes the part to introduce typing to gRPC Python Async APIs.


## Proposal

Add type annotation to the following:
* All public Python APIs listed in [`__all__` variable](https://github.com/grpc/grpc/blob/54dd7563c2d563bff74e4b558f2e985db4a01f2d/src/python/grpcio/grpc/__init__.py#L2100) (the actuall list is too long, please refer to the following PR for changes):
  * [PR link holder]
* gRPC Python generated Code.
  * [PR Link holder]
* Python ancillary packages.
  * [PR link holder]

## Rationale

The alternative is keep APIs in Sync stack unchanged and without type annotation, this way users won't be able to know if their implementation is correct or not without actually running the code.

After add type annotation, instead of original un-typed API (`UnaryUnaryMultiCallable.__call__`):
```python
@abc.abstractmethod
def __call__(self,
             request,
             timeout=None,
             metadata=None,
             credentials=None,
             wait_for_ready=None,
             compression=None):
```

We'll have typed API which will looks like this:
```python
@abc.abstractmethod
def __call__(self,
             request: RequestType,
             timeout: Optional[float] = None,
             metadata: Optional[MetadataType] = None,
             credentials: Optional[CallCredentials] = None,
             wait_for_ready: Optional[bool] = None,
             compression: Optional[Compression] = None) -> ResponseType:
```

## Implementation

* Add typing to internal APIs.
  * [PR1: Add typing for some internal python files.](https://github.com/grpc/grpc/pull/31514)
* Add typing to public API and test though pre-release.
  * [PR link holder]
* Release typed public APIs.

## Open issues (if applicable)

N/A
