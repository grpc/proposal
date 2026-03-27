# Axx: xDS Request Mirroring

- Author(s): @shadialtarsha
- Approver: TBD
- Status: Draft
- Implemented in: N/A
- Last updated: 2026-03-07

## Abstract
This proposal defines request mirroring for gRPC xDS routing using static mirror
cluster targets. Mirrored RPCs are best-effort shadow traffic and MUST NOT
affect primary RPC correctness, latency guarantees, or availability. This gRFC
specifies policy precedence, runtime semantics, non-blocking dependency
behavior, observability requirements, and conformance expectations across gRPC
implementations.

## Background
Request mirroring is a notable gap in gRPC's xDS feature set. Implementations
like Envoy already support mirroring and rely on it for production-safe
validation and migrations. Without equivalent behavior, gRPC users cannot apply
the same xDS traffic experimentation patterns consistently across their stack.

## Motivation
Operators need to:
1. Validate behavior against a shadow backend.
2. Compare production requests across multiple backend versions.
3. Run controlled experiments with low blast radius.

## Goals
1. Define cross-language mirroring semantics for xDS route processing.
2. Ensure the primary RPC path is unaffected by mirroring failures.
3. Define policy precedence across Route, VirtualHost, and RouteConfiguration.
4. Require non-blocking mirror target dependency handling.
5. Define minimum observability and conformance expectations.

## Non-Goals
1. Supporting dynamic mirror target lookup from headers in this proposal.
2. Defining mirrored response handling semantics (responses are ignored).

## Design

### Policy Input
A mirror policy contains:
- `cluster` (static mirror target), and
- optional fractional sampling.

Policies using non-static target selection (like a dynamic header-based cluster
name) are out of scope for this gRFC.

### Policy Precedence
For a matched route, effective mirroring policy set MUST be selected as:
1. Route-level non-empty policy set.
2. Else VirtualHost-level non-empty policy set.
3. Else RouteConfiguration-level policy set.

Policies are not merged across levels.

### Fraction Semantics
Implementations MAY accept source-specific fractional representations, but MUST
normalize to a single internal probability representation before runtime
evaluation. Sampling MUST be applied independently per mirror policy.

### Runtime Semantics
For each matched primary RPC:
1. Evaluate effective mirror policy set.
2. Apply fraction gate for each policy.
3. For each selected mirror policy, issue a mirror RPC to its target cluster.
4. Mirror RPCs are best-effort and fire-and-forget.
5. Mirror responses MUST be ignored for primary path decisions.

### Primary Path Isolation
Implementations MUST satisfy all of the following:
1. Mirror dispatch failure MUST NOT fail primary RPC.
2. Mirror target unavailability MUST NOT delay primary RPC selection/dispatch.
3. Mirror retries (if any) MUST NOT consume primary RPC retry budget/state.
4. Mirror cancellation/timeout MUST NOT mutate primary RPC outcome.
5. Primary cluster resolution must remain usable when mirror dependencies are
   unresolved.

### Dependency/Discovery Behavior
Mirror target discovery/resolution MUST be non-blocking relative to primary
traffic:
- Unresolved mirror dependencies MUST cause mirror skips, not primary gating.
- If implementation has aggregate dependency managers, mirror-only dependencies
  MUST NOT block emission of otherwise valid primary config.

### Mirrored RPC Marking
Mirrored RPCs SHOULD carry an internal marker header for downstream
identification (analysis, policy, logging). Exact header key/value is
implementation-defined for initial rollout, but implementations SHOULD avoid
collision with application metadata namespaces.

## Behavioral Matrix

| Condition | Required Behavior |
|---|---|
| Mirror target cluster missing/unresolved | Skip mirror, continue primary |
| Mirror stream creation/send fails | Record telemetry, continue primary |
| Fraction gate fails | No mirror attempt |
| Multiple mirror policies configured | Apply independently; primary unaffected |
| Primary routing fails (no matched route, etc.) | Existing primary semantics unchanged |
| Mirror config absent | No behavior change |

## Security Considerations
1. Mirroring duplicates request payloads; operators MUST explicitly opt in via
   xDS config.
2. Mirror marker headers must avoid exposing sensitive info and avoid
   user-header collisions.
3. Implementations should document interaction with authn/authz filters and
   policy engines on mirror targets.

## Observability Requirements
Implementations SHOULD expose at minimum:

1. `grpc.xds.mirror.attempted`
   Number of mirror policies evaluated for matched RPCs.

2. `grpc.xds.mirror.sent`
   Number of mirror RPCs successfully dispatched.

3. `grpc.xds.mirror.skipped` (reason label REQUIRED)
   Reasons include at least:
   - `fraction`
   - `target_unavailable`
   - `unsupported_policy`

4. `grpc.xds.mirror.dispatch_failures`
   Local mirror dispatch failures.

Recommended dimensions:
- route identity
- virtual host
- mirror cluster
- authority
- method

Implementations SHOULD include debug logs for:
- effective policy selection source (route/vhost/route-config),
- mirror skip reasons,
- unresolved mirror dependency transitions.

## Implementation Notes (grpc-go Example)
A possible implementation strategy:
1. Parse and normalize mirror policies at xDS decode boundary.
2. Precompute effective policy set per route at config-selector build time.
3. Execute mirroring in per-RPC interceptor path.
4. Dispatch mirror RPCs asynchronously and isolate failures.

## Alternatives Considered
1. Blocking mirror dependency resolution
   Non-appealing: risks primary-path availability.

2. Merge policy sets across route/vhost/route-config
   Non-appealing: contradicts xDS specificity model and increases ambiguity.

3. Omit mirror marker header
   Semi-appealing: however, this will lead to poor operational visibility and
   attribution.

## Considerations Not Included
1. Dynamic target selection via header-based policy (`cluster_header`) is
   intentionally excluded from this gRFC due to unclear operational value vs.
   complexity in cross-language standardization.
2. Standardizing a cross-language mirror marker header key/value is deferred.

## Open Questions
1. Should this capability be enabled by default or only via an explicit feature
   gate?
2. Should we consider adding dynamic target selection via headers to achieve
   parity with other xDS implementations like Envoy?
