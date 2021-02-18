Title
----
* Author(s): stpierre
* Approver: a11r, gnossen
* Status: Draft
* Implemented in: [python](https://github.com/grpc/grpc/pull/25457)
* Last updated: 2021-02-17
* Discussion at: https://groups.google.com/g/grpc-io/c/ctZJkd4MfHg

## Abstract

Expose context status code, details, and trailing metadata on the
server side.

## Background

Currently, on the server-side `ServicerContext` object, status code,
details, and trailing metadata are private. As a result, an
interceptor cannot take action based on any of those properties.

## Proposal

We will add three methods to the servicer context objects:

* `code()` returns the status code.
* `details()` returns the state details.
* `trailing_metadata()` returns the trailing metadata.

All three are functions (not properties) for symmetry with the stub
context class.

## Rationale

One use case for that is a logging interceptor that logs the status of
every response; ideally, with something like `grpc-interceptor`, you'd
want to do something like:

```python
def intercept(
            self,
            method: Callable,
            request: Any,
            context: grpc.ServicerContext,
            method_name: str,
    ) -> Any:
        LOG.info('[REQUEST] %s', method_name)

        try:
            retval = method(request, context)
        except Exception:
            LOG.info('[RESPONSE] %s (%s): %s', method_name, context.code(), context.details())
            raise
        else:
            LOG.info('[RESPONSE] %s: OK', method_name)
        return retval
```

But, of course, `context.code()` and `context.details()` don't exist, and
are instead the private attributes `context._state.code` and
`context._state.details`.

Why add getter methods to the ServicerContext when the author of the
service handler is the one that calls the corresponding setter
methods? They could keep track of what they set
themselves. Interceptor authors, however, don't have this
option. They're not necessarily in control of the service handler. So
it makes sense to make these getters available to them.

But once we add the getters to the interceptors, we want them added to
the non-interceptor `ServicerContext` to maintain uniformity between
the two interfaces.

https://github.com/grpc/grpc/issues/24605 sketches out an earlier
iteration of this.

## Implementation

1. Add the new API calls to the interface (`grpc.ServicerContext`),
and synchronous (`grpc._server._Context`) and AIO server
(`grpc.aio.ServicerContext`)
implementations. https://github.com/grpc/grpc/pull/25457 holds an
incomplete draft.
