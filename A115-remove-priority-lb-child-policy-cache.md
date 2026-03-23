A115: disable Priority LB policy child policy retention cache
----
* Author(s): @apolcyn, @markdroth
* Approvers: @markdroth, @ejona86, @dfawley, @easwars
* Status: {Draft}
* Implemented in: C-core, Java, Go, Node
* Last updated: 2026-03-17
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

[A56](A56-priority-lb-policy.md) describes
[mechanisms](A56-priority-lb-policy.md#child-lifetime-management)
whereby priority LB child policies are cached. There are two cases:

1) When a higher priority child becomes reachable, we deactive
the lower-priority children, and remove them only after an expiry.

2) When a child is removed from the LB policy config.

This proposal removes the usage of a cache for case 2.

## Background

The priority LB child policy retention cache consumes excessive memory under the
right circumstances (depending on the rate and pattern of locality updates).

This is especially the case when a locality is flapping between failover and primary
priorities. For example, notice how priority LB child names increase in the following
sequence of locality updates. On each child name update, previous policies are added
to the retention cache.

```
[[AA, BB], [CC, DD]] => [priority-0-0 priority-0-1]
[[CC], [DD, EE]] => [priority-0-1 priority-0-2]
[[AA, BB], [CC, DD]] => [priority-0-3 priority-0-2]
```

Additionally, priority LB child names are generated with strictly increasing numbers
(once a priority LB child name is unconfigured, it will never be configured again). As such,
the cache is not providing us value.

## Proposal

Priority LB should disable the child policy retention cache, when a
child is removed from its config (i.e., case 2 only).

Note this should be done for Java and Go only.

For C-core, we are actually potentially getting benefit from this
behavior due to subchannel pooling, so we're not planning to drop
it there until we have the longer-term solution ready.

### Temporary environment variable protection

Implementations should provide an environment variable to revert
to the previous behavior (child policy cache enabled with 15-minute timer).

This should be kept around for a few releases, and then removed.

Env var name: `GRPC_EXPERIMENTAL_ENABLE_PRIORITY_LB_CHILD_POLICY_CACHE`.

## Rationale

- Caching the child when it gets removed from the config does not actually
  accomplish anything useful in the case of choosing a priority within an
  xDS cluster (which is the primary case where this policy is used), because
  the hueristic in the cds policy that assigns the child names will never
  reuse a child name once it has been removed from the config.

- We have seen cases where retaining the children has used up a lot of memory
  and file descriptors, which has caused problems for users.

- In the long run, we want a better solution involving a separate layer for
  caching subchannels rather than LB policies, but that will be a separate
  project to be undertaken later.

## Implementation

N/A

## Open issues (if applicable)

N/A
