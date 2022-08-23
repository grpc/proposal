A56: `priority_experimental` LB policy
----
* Author(s): @markdroth
* Approver: @ejona86
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: C-core, Java, Go, Node
* Last updated: 2022-08-23
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

When gRPC's xDS support was first implemented (as described in [gRFC
A27][A27]), all of the EDS logic was in a single monolithic LB policy
called `xds_experimental`.  We subsequently refactored that monolithic
LB policy into several different independent LB policies that could be
composed together to provide the same functionality.  At the time, no
gRFC was written for that change, since it was considered a refactoring
rather than the introduction of a new feature (although the refactoring
was referred to in [gRFC A37][A37]).

Of the new LB policies created by that refactoring, the only policy
that was never documented anywhere and that still exists today is the
`priority_experimental` policy.  Since this policy was created, we
have found several edge cases that have needed to be carefully handled
in its logic, and keeping the various gRPC implementations in sync in
terms of their exact behavior has proven challenging without a written
specification to refer to.  To address that problem, this gRFC serves
as a written specification for the `priority_experimental` LB policy.

## Background

The original monolithic `xds_experimental` LB policy was responsible for
all of the following:

- Processing EDS updates.
- Managing priority failover.
- Managing WRR picking for localities.
- Delegating to a configurable child policy for endpoint picking within a
  locality.
- Generating data for load reporting.

As part of the refactoring, that functionality was split into four new LB
policies.  Two of those policies no longer exist:

1. `eds_experimental`: This policy processed EDS updates from the
   `XdsClient`.  It also handled drops and generated LRS load reporting stats
   for drops.  (This policy was subsequently split up into the
   `xds_cluster_resolver` and `xds_cluster_impl` policies as part of
   [gRFC A37][A37].)
2. `lrs_experimental`: This policy generated LRS load reports for data sent
   to a specific locality.  (This policy's functionality was moved into the
   `xds_cluster_impl` policy as part of [gRFC A42][A42].)

However, the other two LB policies still exist and are actively used:

1. `priority_experimental`: This policy is given a prioritized list of
   child policies.  It routes picks to the highest prioritized child that
   is reporting `READY`.  (This LB policy is used both for picking
   the priority within an xDS cluster and for handling xDS aggregate
   clusters, as described in [gRFC A37][A37].)
2. `weighted_target_experimental`: This policy is given a list of child
   policies, each of which has a weight.  It routes picks to those children
   based on the configured weights.  (This LB policy is used for picking
   the xDS locality within a cluster.  It is described in [gRFC A28][A28].)

This gRFC describes the `priority_experimental` LB policy as it exists
today, which has evolved somewhat since it was originally introduced.

### Related Proposals: 
* [gRFC A27: xDS-Based Global Load Balancing][A27]
* [gRFC A28: xDS Traffic Splitting and Routing][A28]
* [gRFC A37: xDS Aggregate and Logical DNS Clusters][A37]
* [gRFC A42: xDS Ring Hash LB Policy][A42]

[A27]: A27-xds-global-load-balancing.md
[A28]: A28-xds-traffic-splitting-and-routing.md
[A37]: A37-xds-aggregate-and-logical-dns-clusters.md
[A42]: A42-xds-ring-hash-lb-policy.md

## Proposal

The `priority_experimental` LB policy will be given a prioritized list of
child policies.  It will route picks to the highest prioritized child that
is reporting connectivity state `READY`.

Note that this policy is not xDS-specific.  It does not interact with
the `XdsClient` and contains no xDS-specific code.  It is intended to be a
generic policy that could be used in any non-xDS context.

### LB Policy Configuration

The config for this policy is as follows:

```proto
// Configuration for priority LB policy.
message PriorityLoadBalancingPolicyConfig {
  // A map of name to child policy configuration.
  // The names are used to allow the priority policy to update
  // existing child policies instead of creating new ones every
  // time it receives a config update.
  message Child {
    repeated LoadBalancingConfig config = 1;

    // If true, will ignore reresolution requests from this child.
    bool ignore_reresolution_requests = 2;
  }
  map<string, Child> children = 1;

  // A list of child names in decreasing priority order
  // (i.e., first element is the highest priority).
  repeated string priorities = 2;
}
```

Note that, just like the `weighted_target_experimental` policy as
described in [gRFC A28][A28], the config for this policy decouples the
names of the children from their priorities.  This gives us the ability to
move existing children from one priority to another without having to
recreate the child policy (which in Java and Go would require us to recreate
all of the subchannel connections).  For example, in xDS, if the client
receives an EDS update that removes all of the localities from priority
N and replaces them with one or more of the localities from priority N+1,
we want to move the existing child name from priority N+1 to priority N
and then create a new name for the child for priority N+1.

The `ignore_reresolution_requests` field was added as part of [gRFC A37][A37];
see [this section](A37-xds-aggregate-and-logical-dns-clusters.md#priority-policy-reresolution-handling)
for details.

### Hierarchical Addresses

The endpoint addresses are passed to the LB policy API as a flat list.
However, when a tree of LB policies is in use, each leaf of the tree
will need a different subset of those addresses.  This comes into play
in multi-child LB policies like `priority_experimental` and
`weighted_target_experimental`, which need to know which child to send
each address to.

To support this, each address must have an associated attribute that
indicates the path to the child to which it should be sent to at each
level of the hierarchy such that it will wind up at the right leaf policy.
Each multi-child LB policy will look at the first element of the path of
each address to determine which child to send the address to.  It will then
remove that first element when passing the address down to its child.

For example, let's say that we have the following LB policy hierarchy:
- `priority_experimental`:
  - child0 (`weighted_target_experimental`)
    - localityA (`round_robin`)
    - localityB (`round_robin`)
  - child1 (`weighted_target_experimental`)
    - localityC (`round_robin`)
    - localityD (`round_robin`)

We can generate the following addresses with attached paths:
- 10.0.0.1:80 path=["child0", "localityA"]
- 10.0.0.2:80 path=["child0", "localityB"]
- 10.0.0.3:80 path=["child1", "localityC"]
- 10.0.0.4:80 path=["child1", "localityD"]

The `priority_experimental` policy will split this up into two lists, one
for each of its children:
- child0:
  - 10.0.0.1:80 path=["localityA"]
  - 10.0.0.2:80 path=["localityB"]
- child1:
  - 10.0.0.3:80 path=["localityC"]
  - 10.0.0.4:80 path=["localityD"]

The `weighted_target_experimental` policy for child0 will split its list
up into two lists, one for each of its children:
- localityA:
  - 10.0.0.1:80 path=[]
- localityB:
  - 10.0.0.2:80 path=[]

Similarly, the `weighted_target_experimental` policy for child1 will split
its list up into two lists, one for each of its children:
- localityC:
  - 10.0.0.3:80 path=[]
- localityD:
  - 10.0.0.4:80 path=[]

The gRPC implementation provides a utility library that implements a
mechanism for determining which address is passed to which leaf policy.

### Child Lifetime Management

The `priority_experimental` policy will create child policies lazily.
In other words, it will not proactively create the child for every
priority as soon as it gets its config; instead, it will create each
child only at the moment when it needs to attempt to use that priority.
If it chooses to use the child for a given priority, it will not proactively
create the child for any lower priority (higher numeric value).

To avoid unnecessary churn when priorities are flapping up and down, once
the `priority_experimental` policy creates a child for a given priority, if
it later switches to a higher priority child (lower numeric value), it will
not immediately destroy the lower priority (higher numeric value) child that
it had previously been using.  Instead, it will *deactivate* the lower priority
child, which will cause the child to start a 15-minute timer, after which
the child will be destroyed.  However, if the higher priority (lower numeric
value) fails again before the lower priority (higher numeric value) child is
destroyed, then the `priority_experimental` policy will switch back to
the lower priority child and *reactivate* it, which will cause the child
to cancel its timer to prevent it from being destroyed.

The same deactivation timer will be used when the `priority_experimental`
policy receives a config update from its parent that no longer includes the
child (by name).  If the `priority_experimental` policy subsequently receives
another update that re-adds the child (by name), the child will be given
its updated config, but it will not be reactivated unless it is actually
chosen as the current priority.

### Child Connectivity State Tracking

For each child policy, the `priority_experimental` policy will save the
most recent connectivity state, status, and picker, since it may need to
reuse that data at any time.

Each child will also have a 10-second failover timer that indicates
that it is currently trying to connect.  While this timer is running,
the algorithm used to choose a priority (see below) will wait for
this child to finish attempting to connect before it moves on to the
next priority.  The timer will be cancelled when the child reports its
state as `READY`, `IDLE`, or `TRANSIENT_FAILURE`.  If the timer fires
without being cancelled, the child will be treated as if it reported
`TRANSIENT_FAILURE`.  The timer will be started when the child is first
created.  It will also be started if the child reports `CONNECTING`
and it has previously reported `READY` or `IDLE` more recently than
`TRANSIENT_FAILURE`.

### Configuration Updates

When the `priority_experimental` policy receives an updated config from
its parent, it will iterate over its existing children.  If a child does
not exist in the new config, that child will be deactivated (see "Child
Lifetime Management" above).  Otherwise, the child will be sent an
update containing its updated config and the appropriate subset of
addresses (see "Hierarchical Addresses" above).

Note that when the `priority_experimental` policy sends updates to its
child policies, that may cause those child policies to immediately send an
updated picker back up to the `priority_experimental` policy.  However,
if there are multiple children, the report of the updated picker from a
child may result in the `priority_experimental` policy attempting to
attempt to choose a new priority while its state has been only partially
updated (i.e., some of the children have been updated but not others).
To avoid this, the `priority_experimental` policy must not attempt to
choose a new priority as a result of a child's picker update if it is
still in the middle of applying a config update from its parent.
Instead, it should explicitly choose a new priority once it has finished
applying the update from its parent to all of its children.

### Algorithm for Choosing a Priority

The `priority_experimental` policy will use an idempotent algorithm
for choosing a priority to use -- in other words, if the algorithm is
run multiple times with the same underlying state of all of its
children, it will always yield the same result.

This algorithm will be run in two events:
- when processing a child connectivity state update
- when processing an update from the parent

The algorithm will be as follows (Python-style pseudo-code):

```
def SetCurrentPriority(self, priority, deactivate_lower_priorities):
  self.current_priority = priority
  # Deactivate lower priorities if needed.
  if deactivate_lower_priorities:
    for p in range(priority + 1, self.config.priorities.size()):
      child_name = self.config.priorities[p]
      if child_name in self.children:
        self.children[child_name].Deactivate()
  # Use this child's picker.
  child = self.children[self.config.priorities[priority]]
  self.helper.UpdateState(child.connectivity_state(), child.status(),
                          child.GetPicker())

def ChoosePriority(self):
  # If priority list is empty, report TRANSIENT_FAILURE.
  if self.config.priorities.empty():
    status = Status(UNAVAILABLE, "priority policy has empty priority list")
    self.helper.UpdateState(TRANSIENT_FAILURE, status,
                            TransientFailurePicker(status))
    return
  # Iterate through priorities, searching for one in READY or IDLE,
  # creating new children as needed.
  self.current_priority = None
  for priority in range(0, self.config.priorities.size()):
    child_name = self.config.priorities[priority]
    # If the child does not exist, create it.
    if child_name not in self.children:
      child = self.CreateChild(child_name)
      self.children[child_name] = child
      child.Update(self.config.children[child_name])
    else:
      # The child exists.
      child = self.children[child_name]
      # Reactivate if needed.
      child.MaybeReactivate()
    # If the child is READY or IDLE, use it.
    if child.connectivity_state() in (READY, IDLE):
      self.SetCurrentPriority(priority, True)
      return
    # If the child's failover timer is pending, use it.
    if child.FailoverTimerPending():
      self.SetCurrentPriority(priority, False)
      return
  # We did not find a priority in READY or IDLE or whose failover timer
  # was pending, so check for one in CONNECTING.
  for priority in range(0, self.config.priorities.size()):
    child_name = self.config.priorities[priority]
    child = self.children[child_name]
    if child.connectivity_state() == CONNECTING:
      self.SetCurrentPriority(priority, False)
      return
  # We didn't find a child in CONNECTING, so delegate to the last child.
  self.SetCurrentPriority(self.config.priorities.size() - 1, False)
```

### Temporary environment variable protection

N/A

## Rationale

N/A

## Implementation

N/A

## Open issues (if applicable)

N/A
