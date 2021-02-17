Title
----
* Author(s): Chris St. Pierre
* Approver: a11r
* Status: Draft
* Implemented in: python
* Last updated: 2021-02-17
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

Expose context status code, details, and trailing metadata on the
server side.

## Background

Currently, on the server-side `ServicerContext` object, status code,
details, and trailing metadata are private. As a result, an
interceptor cannot take action based on any of those properties.

### Related Proposals

N/A

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

https://github.com/grpc/grpc/issues/24605 sketches out an earlier
iteration of this.

## Implementation

1. Add the new API calls to the synchronous and AIO server
implementations.  https://github.com/grpc/grpc/pull/25457 holds an
incomplete draft.
