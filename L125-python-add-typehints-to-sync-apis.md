Add Typing to gRPC Python Sync API
----
* Author(s): asheshvidyut
* Approver: sergiitk
* Status: In Review
* Implemented in: Python
* Last updated: 2026/03/25
* Discussion at:

## Abstract

This proposal aims to add inline Type Annotations (or Type Hints) to untyped gRPC Python API. 
Inline Type Annotations are added considering [PEP-484](https://peps.python.org/pep-0484/) and [PEP-526](https://peps.python.org/pep-0526/).
This proposal also add [Pyright](https://github.com/microsoft/pyright) (a static type checker) which will run in CI to check for correctness of the Type Hints
added in the gRPC Python Source Code.

## Background

We already have Type Annotations added in gRPC Python Async IO APIs. This proposal is to add Type Hints to the Sync APIs.
Type Hints would benefit users and developers in following ways
* Better IDE Support with Intelligent Autocomplete
* Catching bugs before they happen
* Code as documentation

Additionally, this proposal includes addition of `Pyright` which is static type checker for Python.
This will verify the correctness of Type Hints.

### Related Proposals:
* Related to [Sync API Type Hints](https://github.com/grpc/proposal/pull/459)

## Proposal

* This proposal majorly targets adding inline Type Annotations to file - `src/python/grpcio/grpc/__init__.py`
* Fixes all the Type Annotations added in folder `src/python/grpcio` using static type checking tool - `Pyright`
* Adds missing inline Type Annotations in the `.py` files inside `src/python/grpcio/grpc`

## Rationale

Adding type annotations to the gRPC Python Sync API would provide several benefits:

* **Improved Developer Productivity:** Type annotations would help developers catch errors early in the development process, reducing the time spent on debugging.
* **Enhanced Code Maintainability:** Type annotations would ensure type consistency across the codebase, making it easier to maintain and update the code.
* **Enabled Static Analysis:** Type annotations would allow static analysis tools like MyPy to effectively analyze gRPC code, leading to earlier detection of potential issues.


## Implementation

* PR corresponding to this proposal - https://github.com/grpc/grpc/pull/41210


