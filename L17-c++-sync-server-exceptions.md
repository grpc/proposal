C++ synchronous server should catch exceptions from method handlers
----
* Author(s): vpai
* Approver: a11r
* Status: Proposed
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
- If the method handler throws a `std::exception`, the sync server will treat it
  as though it returned an `UNKNOWN` `Status` marked with the `what` result of
  the exception as its error message.
- If the method handler throws any other kind of exception, the sync server will
  treat it as though it returned an `UNKNOWN` `Status` and will fill in some
  error message.

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

Users may want a different status code choice on exceptions. Such users should
create their own `try/catch` blocks in their method handlers and `return` the
`Status` of their choice.

## Implementation

Wrap all method handler invocations in a `try/catch` block that is itself
protected by preprocessor macros to allow for compilation without exception
support (e.g., with the use of `-fno-exceptions` in gcc or clang).

## Open issues (if applicable)

N/A
