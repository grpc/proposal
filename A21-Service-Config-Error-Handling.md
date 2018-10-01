Service Config Error Handling
----
* Author: yashkt
* Approver: markdroth
* Status: Draft
* Implemented in:
* Last updated: 09-17-2018
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/bAxyse6c2eI

## Abstract

This document proposes a methodology to handle errors in a resolved service
config.

## Background

The
[service config](https://github.com/grpc/grpc/blob/master/doc/service_config.md)
is a mechanism that allows service owners to publish
parameters that clients could use. If the received service config is malformed
or has unknown fields or has invalid values, the client could either ignore the
invalid service config or partially accept the service config while ignoring
certain parts.


### Related Proposals:
* [A2-Service Config via DNS](A2-service-configs-in-dns.md)

## Proposal

If a resolved service config is badly formatted or contains invalid values for
known fields, the client should ignore that service config in its entirety.
Unknown fields in the service config should be ignored. If there was a service
config that the client was previously using, it should continue using those
parameters when communicating with the service. If there was no service config
being used, then the client should continue behaving as if it never received a
service config update from the resolver. On the other hand, if it is a new
client, then the client should fail to start, i.e., not act on the resolution
result till a valid service config is received.

If a default service config is provided (as a channel argument), service config
resolution is enabled and the received service config is invalid, the client
should NOT fall back on the default service config. It should as before, fail to
start if it has not received a valid service config. On the other hand, if
service config resolution is disabled, then the client should use the default
service config.

Based on this proposal, if new load balancing policies are to be introduced and
the rollout is not controlled, then older clients will start discarding entire
service configs due to the load balancing policy being unknown. To make this
rollout process easier, load balancing policy could be changed into a list of
policies. The clients would iterate through the list and use the first policy
that it knows. If none of the policies are known, the client would treat the
service config as invalid.

For example,

“loadBalancingPolicy” : [
    “NewUnknownPolicy”,
    “RoundRobin”
  ]

In this case, since the client does not understand the first policy, it would
use RoundRobin.

## Rationale

By discarding invalid service configs instead of ignoring fields that have
invalid values, we prevent clients from resorting to using default values which
might be unwanted for certain systems. The clause that existing clients continue
using older service configs prevents an invalid service config from bringing
down the entire system, while newer clients which do not have a previously
received valid service config fail to start, avoiding new traffic for the
duration that the service is set with an invalid config.

Examples of invalid service configs :

* Badly Formatted service config
“serviceConfig” : {
  “MethodConfig” :
   {
   “Service // bad format
  }

* Fields with invalid values are not allowed
“serviceConfig” : {
      “loadBalancingPolicy” : “UnknownPolicy”
}

## Implementation

C -
JAVA -
Go -

## Open issues (if applicable)

N/A
