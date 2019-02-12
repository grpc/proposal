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

In brief, if a service config (JSON formatted) received is badly formatted or
contains invalid values for known fields, the client should ignore that service
config in its entirety. This section enumerates the exact steps that clients
need to take on receiving a new config.

### Behavior on receiving a new gRPC Config

1. A gRPC config can have multiple service configs. When multiple service configs
are available, clients choose the service config to use based on the proposal
[here](A2-service-configs-in-dns.md#canarying-changes).
2. On choosing a service config, the client validates the service config based on
the criteria outlined in the section below.
3. If the service config is valid, the client uses the configuration provided.
4. If the service config is invalid, the clients should reject the received
config in its entirety and do different things depending on its state and
configuration.
	1. If it previously received an update with a valid service config, it should
	continue using the configuration from that config.
	2. If this is a new client or a client which has not yet received a valid
	service config -
		1. It uses the default service config, if configured. Note that, an empty 
		default service config is a valid service config.
		2. If it does not have a default service config, it should treat the 
		resolution attempt as having failed (e.g., for a polling-based resolver, 
		it should retry the query after appropriate backoff). In other words, 
		the channel will remain in state TRANSIENT_FAILURE until a valid service 
		config is received.

### Criteria to determine the validity of a service config

A valid service config needs to be well formatted, i.e., the JSON representation
of the service config needs to be in a valid JSON format. If the service config
is well formatted, the client validates each field that it recognizes while
ignoring the unknown fields, i.e., unknown fields do not affect validation. For
known fields, valid values are defined by the field itself. If any of the known
fields fails in validation, the entire service config is deemed invalid.

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

Note that loadBalancingPolicy is deprecated in favor of loadBalancingConfig,
which provides a list of LB policies with the client selecting the first policy
it supports. If it doesn’t understand any of the policies, it is treated as
invalid, and the entire service config is rejected.

“serviceConfig” : {
  “loadBalancingConfig” : [
    “UnknownPolicy1” : {},
    “UnknownPolicy2” : {}
  ]
}


## Implementation

C -
JAVA -
Go -

## Open issues (if applicable)

N/A
