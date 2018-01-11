C++ synchronous server should catch exceptions from method handlers
----
* Author(s): vpai
* Approver: a11r
* Status: Approved
* Implemented in: https://github.com/grpc/grpc/pull/13815
* Last updated: December 19, 2017
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/u1XmQPAi3fA

## Abstract

The C++ synchronous server should catch exceptions thrown by user method
handlers and fail the RPC rather than crashing the server.

## Background

The C++ sync server is the one place in the C++ API that calls out from the gRPC
library to user code. Although it is a [Google standard not to throw
exceptions](https://google.github.io/styleguide/cppguide.html#Exceptions), we
cannot be sure that user code follows those guidelines. At the current time, the
gRPC sync server will not catch any exceptions thrown in a method handler, and
the process will terminate with an uncaught exception.

### Related Proposals:

N/A

## Proposal

The C++ sync server should wrap its invocation of user method handler functions
in a `try/catch` block.
- If the method handler throws any kind of exception, the sync server will
  treat it as though it returned an `UNKNOWN` `Status` and will fill in some
  error message.

**NOTE**: An earlier version of this proposal suggested using the
  `what` result of a `std::exception` as the error message, but this
  should not be done since it might leak sensitive information from
  the server implementation.

Additionally, this work will have the following constraints:
1. No `throw`s will be introduced into the gRPC C++ source code base
1. gRPC will continue to build and run correctly if the `-fno-exceptions`
compiler flag is used to prevent the use of C++ exceptions. In that
case, the pre-existing behavior of the library will be maintained.
1. A new portability build configuration will be added to guarantee
that the library continues to build without the use of exceptions.
1. If there is no exception thrown by a method handler invocation,
there will be no observable performance impact for common compiler
and runtime configurations.

## Rationale

Although we can push back and say that the service implementer should be
responsible for making sure to not call functions that cause exceptions or catch
any exceptions that their code may generate, this is unreliable and
error-prone. The user may not realize that 20 levels down the abstraction stack,
some function can trigger an exception. So, the method handler will end up
generating an uncaught exception and terminating the server.

Another alternative is that the user unsure about exception semantics of the
method handler implementation can wrap every method handler in a `try/catch`
block. However, this leads to excessive boilerplate code.

In contrast to both of the existing options, `catch`ing in the library doesn't
blow up the user's code and helps to maintain server robustness. Using an
`UNKNOWN` status code is literally reasonable since gRPC by definition does not
know the details of the user's method handler. Modern compiler and runtime
implementations also do not take a performance hit from using `try` if there is
no exception to catch.

Users may want a different status code or error message choice on
exceptions. Such users should create their own `try/catch` blocks in
their method handlers and `return` the `Status` of their choice.

## Implementation

Wrap all method handler invocations in a `try/catch` block that is itself
protected by preprocessor macros to allow for compilation without exception
support (e.g., with the use of `-fno-exceptions` in gcc or clang).

## Open issues (if applicable)

N/A
