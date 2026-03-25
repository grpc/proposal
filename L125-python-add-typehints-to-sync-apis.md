Add Typing to gRPC Python Sync API
----
* Author(s): asheshvidyut
* Approver: sergiitk
* Status: In Review
* Implemented in: Python
* Last updated: 2026/03/25
* Discussion at:

## Abstract

This proposal aims to add inline type annotations (or type hints) to the untyped gRPC Python API. 
These annotations are added in accordance with [PEP-484](https://peps.python.org/pep-0484/) and [PEP-526](https://peps.python.org/pep-0526/).
The proposal also introduces [Pyright](https://github.com/microsoft/pyright), a static type checker that will run in CI to verify the correctness of the type hints added to the gRPC Python source code.

## Background

Type annotations are already present in the gRPC Python AsyncIO APIs. This proposal extends type hints to the synchronous APIs.
Type hints benefit users and developers in the following ways:
* Better IDE support with intelligent autocomplete
* Catching bugs before they happen
* Code as documentation

Additionally, this proposal includes the addition of `Pyright`, a static type checker for Python, to verify the correctness of these hints.

### Related Proposals:
* Related to [Sync API Type Hints](https://github.com/grpc/proposal/pull/459)

## Proposal

* This proposal primarily targets adding inline type annotations to the file `src/python/grpcio/grpc/__init__.py`.
* It ensures all type annotations in the folder `src/python/grpcio` pass validation using `Pyright`.
* It adds missing inline type annotations to `.py` files inside `src/python/grpcio/grpc`.

## Rationale

Adding type annotations to the gRPC Python Sync API would provide several benefits:

* **Improved Developer Productivity:** Type annotations would help developers catch errors early in the development process, reducing the time spent on debugging.
* **Enhanced Code Maintainability:** Type annotations would ensure type consistency across the codebase, making it easier to maintain and update the code.
* **Enabled Static Analysis:** Type annotations would allow static analysis tools like MyPy to effectively analyze gRPC code, leading to earlier detection of potential issues.


## Implementation

* PR corresponding to this proposal - https://github.com/grpc/grpc/pull/41210


