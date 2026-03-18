A115: disable Priority LB policy child policy retention cache
----
* Author(s): @apolcyn
* Approvers: @markdroth, @ejona86, @dfawley, @easwars
* Status: {Draft}
* Implemented in: C-core, Java, Go, Node
* Last updated: 2026-03-17
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

[A56](https://github.com/grpc/proposal/blob/master/A56-priority-lb-policy.md) describes
a [caching mechanism](https://github.com/grpc/proposal/blob/master/A56-priority-lb-policy.md#child-lifetime-management) whereby priority LB child policies are kept around for an additional 15 minutes
before being fully torn down.

This proposal removes that cache.

## Background

The priority LB child policy retention cache consumes excessive memory under the
right circumstances (depending on the rate and pattern of locality updates).

This is especially the case when a locality is flapping between failover and primary
priorities. For example, notice how priority LB child names increase in the following
sequence of locality updates. On each child name update, previous policies are added
to the retention cache.

```
[[AA,BB],[CC, DD]] => [priority-0-0 priority-0-1]
[[AA, BB, CC], [DD]] => [priority-0-1 priority-0-2]
[[AA, BB], [CC, DD]] => [priority-0-1 priority-0-2]
[[AA, BB, CC], [DD]] => [priority-0-2 priority-0-3]
[[AA, BB], [CC, DD]] => [priority-0-2 priority-0-3]
[[AA, BB, CC], [DD]] => [priority-0-3 priority-0-4]
```

Additionally, priority LB child names are generated with strictly increasing numbers
(once a priority LB child name is unconfigured, it will never be configured again). As such,
the cache is not providing us value.

## Proposal

Priority LB should disable the child policy retention cache.

### Temporary environment variable protection

Implementations should provide an environment variable to revert
to the previous behavior (child policy cache enabled with 15-minute timer).

Env var name: `GRPC_EXPERIMENTAL_ENABLE_PRIORITY_LB_CHILD_POLICY_CACHE`.

## Rationale

N/A

## Implementation

N/A

## Open issues (if applicable)

N/A
