Add Typing to gRPC Python Sync API
----
* Author(s): Sourabh Singh
* Approver: gnossen
* Status: Draft
* Implemented in: Python
* Last updated: 2024/08/23
* Discussion at: [discussion thread](https://groups.google.com/g/grpc-io/c/_zVpELQnUQU)

## Abstract

This proposal aims to add type annotations to the remaining untyped parts of the gRPC Python API. This would allow developers to use static type analysis tools like MyPy to catch potential errors early in the development process.

## Background

[Type Hints](https://peps.python.org/pep-0484/#union-types) was introduced for a while now, we already added type annotation to python Async APIs since it has a high enough version floor. Now it's time to do the same for Sync APIs.

This proposal plans to add type annotation to python Sync public APIs which allows users to build safer code by running static type analysis tools.


### Related Proposals: 
* Loosely related to [Async API proposal](https://github.com/lidizheng/proposal/blob/grpc-python-async-api/L58-python-async-api.md#introduce-typing-to-generated-code).
  * The proposal includes the part to introduce typing to gRPC Python Async APIs.


## Proposal
  
We propose to add type annotations to the following:

* **All public Python APIs listed in the [`__all__` variable](https://github.com/grpc/grpc/blob/54dd7563c2d563bff74e4b558f2e985db4a01f2d/src/python/grpcio/grpc/__init__.py#L2100) :** This includes all the core gRPC functionality exposed in the `grpcio.grpc` module.  
    * **Example:** The `UnaryUnaryMultiCallable.__call__` method would look like this with type annotations:
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
    * **Implementation:** [python sync api updates](https://github.com/grpc/grpc/pull/37967)
* **gRPC Python Async Code:**
    * **Implementation:** [python async api updates](https://github.com/grpc/grpc/pull/37921)
* **Python ancillary packages:** This includes packages like `grpcio-admin`, `grpcio-channelz`, etc., which provide additional functionality for gRPC.
    * **Implementation:** [ancillary packages updates](https://github.com/grpc/grpc/pull/37854)

## Rationale

Adding type annotations to the gRPC Python Sync API would provide several benefits:

* **Improved Developer Productivity:** Type annotations would help developers catch errors early in the development process, reducing the time spent on debugging.
* **Enhanced Code Maintainability:** Type annotations would ensure type consistency across the codebase, making it easier to maintain and update the code.
* **Enabled Static Analysis:** Type annotations would allow static analysis tools like MyPy to effectively analyze gRPC code, leading to earlier detection of potential issues.


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

We propose a phased approach to implementing this proposal:

1. **Add typing to internal APIs:** This would involve adding type annotations to the internal functions and classes used by the gRPC Python Sync API.
    * **Implementation:** [PR1: Add typing for some internal python files.](https://github.com/grpc/grpc/pull/31514)
2. **Add typing to public API and mark it as experimental for at least one release:** This would involve adding type annotations to the public API and marking it as experimental to allow for feedback and potential adjustments.
    * **Implementation:** [python sync api updates](https://github.com/grpc/grpc/pull/37967)
3. **Release typed public APIs:** Once the experimental phase is complete and any necessary adjustments are made, the typed public APIs would be released as stable.
4. **Integrate MyPy and typeguard into CI:** To ensure type consistency and catch potential errors early, we will integrate MyPy (for static type checking) and typeguard (for runtime type checking) into our continuous integration pipeline. 

## Open issues (if applicable)

N/A

## Conclusion

This proposal presents a comprehensive approach to providing type hints for gRPC Python applications. By addressing the challenges of type inference, code maintainability, and static analysis, we aim to significantly enhance the developer experience and improve the overall quality of gRPC Python code.