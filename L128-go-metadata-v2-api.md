L128: Go Metadata V2 API
----

- Author(s): ktalg
- Approver: a11r
- Status: Draft
- Implemented in:
- Last updated: 2026/03/18
- Discussion at: TBD

## Abstract

This proposal redesigns the public metadata API for grpc-go as a true **v2** API.

The new API stops exposing `metadata.MD` as the primary public abstraction and instead provides a **context-bound, function-only API** whose surface matches gRPC protocol semantics directly:

- request metadata
- response headers
- response trailers

The proposal has four main goals:

1. make the API match the protocol model more closely;
2. eliminate the need for grpc-go to manage a public `map[string][]string` carrier for users;
3. remove the main source of avoidable copying and materialization overhead in the current API;
4. unify request/response metadata operations so interceptors and middleware can reason about them consistently.

This proposal intentionally treats the work as a **v2** redesign rather than an incremental tweak of the existing API surface.

## Background

gRPC metadata is a side channel attached to an RPC. At the protocol level it is not an abstract “bag” detached from a call; it is carried as HTTP/2 headers and trailers.

The protocol defines the stream shape as:

- request = request headers + messages + EOS
- response = response headers + messages + trailers, or trailers-only for immediate failure

Custom metadata is part of request headers, response headers, or response trailers. Keys are ASCII, are case-insensitive, must not use the reserved `grpc-` prefix, and binary metadata is identified with the `-bin` suffix. Binary values are encoded at the HTTP/2 boundary. Header order is generally not preserved except for duplicate-name semantics, and implementations must accept both padded and unpadded base64 forms for binary metadata. Servers and clients may also enforce header size limits, with 8 KiB suggested as a default. Status is always carried in trailers.

In grpc-go today, however, the public API is centered around `metadata.MD`, which is exposed as `type MD map[string][]string`, plus a set of helpers such as `New`, `Pairs`, `Join`, `Copy`, `AppendToOutgoingContext`, `FromIncomingContext`, `FromOutgoingContext`, and separate top-level response APIs such as `grpc.SetHeader`, `grpc.SendHeader`, and `grpc.SetTrailer`.

This split has historically grown around the publicly exposed `MD` type instead of around the protocol model.

## Motivation and Problems with the Current Design

### 1. `MD` makes grpc-go manage a public carrier it does not need to own

The current API does not only expose metadata at the RPC boundary. It also exposes a concrete carrier type, `MD`, and then has to provide more API surface around it (`New`, `Pairs`, `Join`, `Copy`, `Get`, `Set`, `Delete`, etc.).

This mixes two responsibilities:

1. metadata operations at an RPC boundary; and
2. general-purpose user management of a metadata map.

For a gRPC library, only (1) is essential. If users need an offline carrier, they can manage their own `map[string][]string` or another structure and inject it at the boundary.

### 2. The current public read path forces materialization and copying

Issue [#8860](https://github.com/grpc/grpc-go/issues/8860) demonstrates the current cost clearly. After `FromOutgoingContextRaw` became internal-only, the public way to inspect outgoing metadata is `FromOutgoingContext`, which materializes a new `MD` and copies values. In the issue benchmark, `FromOutgoingContext` takes about `1172 ns/op`, `496 B/op`, and `8 allocs/op`, while the old raw path was around `3.989 ns/op`, `0 B/op`, and `0 allocs/op`.

This is not merely an implementation bug. It is a symptom of the public API shape: once the public contract is “return a mutable `MD`”, grpc-go must build that object.

### 3. The outgoing append model is expensive in deep append chains

The current implementation stores appended outgoing kv pairs in a `[][]string` slice and copies append history on every append. The experimental work in [#8942](https://github.com/grpc/grpc-go/issues/8942) shows that replacing this with a persistent linked structure makes append effectively O(1) per call instead of O(n) with respect to append depth, reducing total append cost from O(N²) to O(N) for a depth-N append chain.

This again suggests that the current public abstraction is fighting the desired performance profile.

### 4. The current API surface does not map cleanly to protocol semantics

gRPC protocol semantics are request headers, response headers, and response trailers.

grpc-go public APIs today instead expose:

- incoming vs outgoing context metadata
- plus separate response operations in `grpc.SetHeader`, `grpc.SendHeader`, `grpc.SetTrailer`
- plus client-side header/trailer retrieval through call options

This produces a surface that is harder to explain and reason about than the protocol itself.

### 5. Validation and safety are still fragmented

Issue [#5913](https://github.com/grpc/grpc-go/issues/5913) shows validation inconsistencies:

- invalid keys could panic or bypass validation depending on the entry path;
- metadata appended through `AppendToOutgoingContext` could avoid validation that occurred on other paths.

### 6. Response metadata is not modeled as a first-class coherent surface

Issue [#4317](https://github.com/grpc/grpc-go/issues/4317) shows server interceptors resorting to reflection to inspect header/trailer data after business logic.

At the same time, grpc-go exposes response metadata operations in a separate top-level `grpc` API (`SetHeader`, `SendHeader`, `SetTrailer`) and in stream methods, instead of unifying them with the rest of metadata handling.

This makes response metadata harder to reason about than request metadata and creates additional asymmetry for interceptors.

## Related Issues

This proposal is directly motivated by the following issues and prior design history:

- [#8860](https://github.com/grpc/grpc-go/issues/8860) — public outgoing read path is expensive
- [#8942](https://github.com/grpc/grpc-go/issues/8942) — prototype showing better internal append representation
- [#5913](https://github.com/grpc/grpc-go/issues/5913) — validation inconsistencies across entry points
- [#4317](https://github.com/grpc/grpc-go/issues/4317) — server interceptors cannot easily access response header/trailer
- [L7 Go metadata API change](L7-go-metadata-api.md) — incoming/outgoing split was needed because the old model was unsafe

## Goals

1. Replace the public `MD`-centric API with a context-bound API that models metadata at the RPC boundary.
2. Make the public naming reflect protocol semantics: **request**, **response header**, **response trailer**.
3. Avoid materializing `MD` merely to read metadata.
4. Permit an internal representation optimized for append-heavy request metadata paths.
5. Centralize normalization and validation at metadata entry points.
6. Bring response header/trailer operations under the same metadata API family.
7. Preserve room for future binary metadata cleanup without forcing a large type-system redesign in this proposal.

## Non-Goals

1. This proposal does not attempt to preserve source compatibility with the current metadata API surface.
2. This proposal does not redesign HTTP/2 or gRPC wire semantics.
3. This proposal does not introduce a new public metadata carrier type.
4. This proposal does not require users to stop using their own map-based carriers in application code.
5. This proposal does not fully solve all future binary-metadata API questions in the first cut.

## Proposal

### Overview

Introduce a new metadata v2 API for grpc-go whose public surface consists only of package-level functions operating on `context.Context`.

The API is organized around the protocol model:

- **request metadata**: client writes, server reads
- **response headers**: server writes, client reads
- **response trailers**: server writes, client reads

The API deliberately does **not** expose a public metadata carrier type such as `MD`.

If users need offline construction, merging, copying, synchronization, or storage, they may use their own `map[string][]string` or other carrier types and inject them through `Replace*` APIs.

### Public API Sketch

```go
package metadatav2

import "iter"

// Request metadata.
func AppendRequest(ctx context.Context, kv ...string) context.Context
func SetRequest(ctx context.Context, kv ...string) context.Context
func ReplaceRequest(ctx context.Context, md map[string][]string) context.Context
func DeleteRequest(ctx context.Context, keys ...string) context.Context
func GetRequest(ctx context.Context, key string) []string
func IterRequest(ctx context.Context) iter.Seq2[string, []string]

// Response headers.
func AppendHeader(ctx context.Context, kv ...string) error
func SetHeader(ctx context.Context, kv ...string) error
func ReplaceHeader(ctx context.Context, md map[string][]string) error
func DeleteHeader(ctx context.Context, keys ...string) error
func GetHeader(ctx context.Context, key string) []string
func IterHeader(ctx context.Context) iter.Seq2[string, []string]

// Response trailers.
func AppendTrailer(ctx context.Context, kv ...string) error
func SetTrailer(ctx context.Context, kv ...string) error
func ReplaceTrailer(ctx context.Context, md map[string][]string) error
func DeleteTrailer(ctx context.Context, keys ...string) error
func GetTrailer(ctx context.Context, key string) []string
func IterTrailer(ctx context.Context) iter.Seq2[string, []string]
```

### Semantics

#### Request metadata

Request metadata is attached to the request context before the RPC is dispatched.

- On the client, `AppendRequest`, `SetRequest`, `ReplaceRequest`, and `DeleteRequest` return a derived context that carries the modified request metadata state.
- On the server, `GetRequest` and `IterRequest` read the request metadata that arrived with the RPC.
- Interceptors and middleware may read request metadata with `GetRequest`/`IterRequest` and may derive a new context using `AppendRequest`/`SetRequest`/`ReplaceRequest` for downstream calls.

#### Response headers and trailers

Response headers and trailers are per-call response state.

- On the server, `AppendHeader`/`SetHeader`/`ReplaceHeader` and `AppendTrailer`/`SetTrailer`/`ReplaceTrailer` mutate response metadata associated with the RPC referenced by `ctx`.
- On the client, `GetHeader`/`IterHeader` and `GetTrailer`/`IterTrailer` read the metadata received for the completed or partially completed call.

This unifies response metadata under the metadata API itself instead of splitting it across `grpc.SetHeader`, `grpc.SendHeader`, `grpc.SetTrailer`, stream methods, and opaque call options.

#### Iterator-based reads

The v2 read API uses Go's iterator style for metadata traversal.

That keeps the surface aligned with newer Go iteration patterns, lets callers use normal `for ... range` syntax, and still allows grpc-go to iterate directly over internal state without materializing `MD`.

### Why use request / header / trailer instead of incoming / outgoing?

The protocol itself is defined in terms of request headers, response headers, and response trailers.

`request`, `header`, and `trailer` match the protocol directly, work symmetrically across client and server roles, and avoid overloading “outgoing” on the server side, where the real operation is on response headers or trailers.

### Why remove public dependence on `MD`?

The key design claim of this proposal is that grpc-go should stop designing the public metadata API around `MD`.

`MD` is an implementation carrier, not a protocol concept. Once it is the center of the public API, grpc-go has to support construction, copying, merging, and mutation of that carrier, and read APIs tend to materialize it even when callers only need lookup or iteration. Users that need an offline carrier can manage their own `map[string][]string` and inject it at the RPC boundary.

Accordingly, this proposal keeps `map[string][]string` only as an **injection format** for `Replace*`, not as the core public abstraction.

### Internal Representation

The public API does not mandate a specific internal layout. That is intentional.

However, the proposal is designed to enable an internal model such as:

- normalized base metadata plus append log / persistent linked chain for request metadata;
- lazy key indexing for point lookup;
- direct read iteration without full `MD` materialization;
- response header/trailer stores directly attached to per-call state.

The [#8942](https://github.com/grpc/grpc-go/issues/8942) prototype is strong evidence that this internal freedom matters.

### Validation and Normalization

All public entry points must perform common normalization and validation.

At minimum:

- normalize keys to lowercase;
- reject reserved `grpc-` keys for custom metadata;
- validate allowed key characters according to protocol expectations;
- preserve duplicate value semantics;
- keep `-bin` handling at the transport boundary rather than making user-visible state depend on wire-encoded base64 forms.

A v2 API should provide exactly one validation model for all request/header/trailer mutation paths, instead of different behavior depending on whether metadata was created by `New`, `Pairs`, `AppendToOutgoingContext`, or top-level response helpers.

## Performance

### Current Measured Problems

#### Outgoing read cost in current public API

From [#8860](https://github.com/grpc/grpc-go/issues/8860):

| Benchmark | ns/op | B/op | allocs/op |
|---|---:|---:|---:|
| `FromOutgoingContext` | 1172 | 496 | 8 |
| `FromOutgoingContextRaw` | 3.989 | 0 | 0 |

This shows the cost of the current public contract that returns a materialized `MD`.

#### Outgoing append prototype results

From the prototype in [#8942](https://github.com/grpc/grpc-go/issues/8942):

| Benchmark | ns/op | B/op | allocs/op |
|---|---:|---:|---:|
| append depth 1, old | 126.1 | 168 | 4 |
| append depth 1, v2 | 131.4 | 168 | 4 |
| append depth 4, old | 593.2 | 824 | 16 |
| append depth 4, v2 | 531.9 | 672 | 16 |
| from depth 1, old | 315.0 | 464 | 5 |
| from depth 1, v2 | 302.0 | 464 | 5 |
| from depth 4, old | 730.7 | 1048 | 11 |
| from depth 4, v2 | 672.7 | 1048 | 11 |

These results match the intended improvement: append history should not be recopied on every append.

### Expected API-Level Performance Characteristics

The proposed public API does not promise a single exact implementation, but it is specifically designed to allow the following characteristics:

| Operation | Current MD-centric API | Proposed API target |
|---|---|---|
| Append request metadata | O(n) per append with append-depth copying in current implementation | O(1) amortized append with internal persistent append state |
| Read one request key | Often requires full metadata materialization or copy for public access | O(1) or near O(1) point lookup without exporting carrier |
| Iterate request metadata | Return a copied `MD`, then iterate | `IterRequest` directly over internal representation |
| Replace all request metadata | `NewOutgoingContext` + public `MD` creation | `ReplaceRequest(ctx, map[string][]string)` |
| Set response header/trailer | Separate grpc-level APIs and stream APIs | unified metadata API surface |

In other words, v2 aims to make the *common operations* (`append`, `set`, `get`, `range`) cheap without requiring grpc-go to expose its internal storage layout.

## Benefits

### 1. Better protocol fit

The API names and grouping directly match the protocol model:

- request metadata
- response header metadata
- response trailer metadata

This makes the API easier to teach, easier to document, and easier to reason about than today’s mix of `MD`, incoming/outgoing helpers, top-level grpc functions, and call options.

### 2. Better ergonomics

The most common operations become explicit and small:

- append values
- set values
- replace from caller-owned map
- delete keys
- read by key
- iterate with a Go iterator

Users that need more than that can build it themselves on ordinary Go maps without requiring grpc-go to expose and maintain a public carrier type.

### 3. Better interceptor story

A single metadata API family for request/header/trailer makes middleware easier to reason about. Today request metadata is largely context-based, while response metadata is split across separate APIs and opaque call options. Issue [#4317](https://github.com/grpc/grpc-go/issues/4317) is one example of the resulting awkwardness. Iterator-based reads also make traversal fit more naturally into interceptor code.

### 4. Better validation and safety

A v2 API can apply identical normalization and validation rules across all entry paths and avoid the fragmented behavior described in [#5913](https://github.com/grpc/grpc-go/issues/5913).

By removing `MD` as the central public type, grpc-go also reduces its obligation to preserve risky observable behaviors such as permissive stringification or map-like mutation patterns.

### 5. Better performance headroom

This proposal does not depend on any single internal layout, which is exactly the point. Once grpc-go no longer needs to return a public `MD`, it can optimize for append-heavy and point-read-heavy workloads more aggressively.

## Response Header/Trailer Consolidation

This proposal specifically recommends moving header/trailer operations into the new metadata API family.

### Current problems

- request metadata lives primarily in `metadata`
- response header/trailer operations live in `grpc`
- some response observation relies on client call options
- stream APIs also have parallel methods

This fragmentation makes it harder to express “metadata for one RPC” as a coherent concept.

### Benefits of consolidation

1. one package and one naming system for request/header/trailer metadata;
2. one normalization and validation model;
3. clearer interceptor support;
4. easier documentation;
5. easier future optimization because all metadata flows are modeled together.

The proposed design does **not** eliminate the protocol distinction between response headers and trailers. On the contrary, it gives them explicit first-class status inside the metadata API.

## Alternatives Considered

### Keep `MD`, but optimize internals only

This would improve append performance but would not solve the core API problem:

- grpc-go would still publicly revolve around `MD`;
- public read APIs would still tend to materialize `MD`;
- response header/trailer APIs would remain split;
- validation and representation concerns would remain fragmented.

This is therefore useful as an implementation step, but insufficient as the final API design.

### Introduce a new public metadata object type

A dedicated opaque metadata type could also solve some representation problems, similar to Java.

This proposal rejects that path for now because:

- it still makes grpc-go responsible for a general-purpose public carrier;
- it expands the public object model more than needed;
- the common operations can be expressed directly on `context.Context`;
- users that need an offline carrier already have ordinary Go maps.

### Use incoming/outgoing naming instead of request/header/trailer

This would be better than the old API but still less direct than protocol naming.

`incoming` and `outgoing` are library-oriented terms; `request`, `header`, and `trailer` are the protocol concepts users already ultimately interact with.

## Rationale

The current grpc-go metadata API has accumulated around a public carrier type. That design has led to a broad API surface, fragmented semantics, response-metadata asymmetry, and visible performance costs.

A v2 design should correct the abstraction boundary itself rather than continuing to optimize around it.

The best boundary is not “metadata as a public Go map-like object.”
The best boundary is “metadata operations bound to one RPC context, named after the protocol roles they represent.”

## Implementation

A possible implementation strategy is:

1. request metadata stored in context as normalized immutable base state plus append chain;
2. `GetRequest` and `IterRequest` read directly without building `MD`;
3. response header/trailer state stored in per-call runtime objects referenced from context;
4. all mutation entry points share one validation/normalization layer;
5. binary metadata continues to be identified by `-bin`, with encoding/decoding handled at transport edges.

This keeps the external API simple while leaving the runtime free to optimize.

## Open issues

1. Whether binary metadata should receive explicit byte-oriented APIs in a follow-up proposal.
2. Whether client-side response header availability before full RPC completion needs additional lifecycle wording for streaming APIs.
3. Whether `SendHeader`-style explicit flush semantics should survive as a separate transport-level API or be represented differently in v2.
4. Whether `Iter*` should guarantee stable duplicate-key visitation order or only protocol-equivalent semantics.

## Conclusion

grpc-go metadata v2 should stop exposing `MD` as the center of the public design.

Instead, it should expose a small, context-bound API named after the protocol itself:

- request metadata
- response headers
- response trailers

This gives grpc-go a cleaner abstraction boundary, a more coherent interceptor story, a simpler public surface, and much better room for internal performance optimization.
