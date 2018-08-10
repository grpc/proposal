OpenCensus C++ Span API changes
----
* Author(s): @g-easy
* Approver: a11r
* Status: Draft
* Implemented in: C++
* Last updated: 2018-07-17
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/ZpXeFinzl5A

## Abstract

Add a public API for getting access to the OpenCensus Span for the current RPC.

## Background

The [OpenCensus](https://opencensus.io/)
[filter](https://github.com/grpc/grpc/tree/master/src/cpp/ext/filters/census)
creates a Span per RPC, which can be accessed through the `ServerContext`
object.

Currently, the public API for the filter comes from:
```
#include <grpcpp/opencensus.h>
```

And `GetSpanFromServerContext()` comes from:
```
#include "src/cpp/ext/filters/census/grpc_plugin.h"
```

### Related Proposals:

[L29](https://github.com/grpc/proposal/blob/master/L29-opencensus-filter.md) was
the initial gRFC for OpenCensus integration.

## Proposal

[PR15984](https://github.com/grpc/grpc/pull/15984) proposes moving
`GetSpanFromServerContext()` into `grpcpp/opencensus.h`

## Rationale

Pros:
* Only one file needs to be `#include`d for OpenCensus tracing, instead of two.

Cons:
* `grpcpp/opencensus.h` has to depend on `opencensus/trace/span.h` which, in
  turn, depends on `absl/strings/string_view.h`

OpenCensus depends on Abseil, so at some point the consumer inevitably has to
build it and link with it.

## Implementation

See PR15984.

## Open issues (if applicable)

N/A
