Service Config Error Handling
----
* Author: yashkt
* Approver: markdroth
* Status: Draft
* Implemented in:
* Last updated: 02-19-2019
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

1. A gRPC config can have multiple service configs. When multiple service
configs are available, clients choose the service config to use based on the
proposal
[A2 Service Config in DNS](A2-service-configs-in-dns.md#canarying-changes).
Only the chosen service config affects the validation of the gRPC config, i.e.,
if the chosen service config is found invalid while the other service configs
were invalid, the gRPC config would still be considered invalid as a whole.
2. The client validates the choice based on the criteria outlined in the
section below.
3. If the service config is valid, the client uses the configuration provided.
4. If the service config is invalid, the clients should reject the received
config in its entirety and do different things depending on its state and
configuration.
	1. If it previously had selected a valid service config or no service
	config, it should continue using that previous configuration from that
	config. An empty service config is valid.
	2. If the client has not previously selected a valid service config or no
	service config, for example a new client, then:
		1. It uses the default service config, if configured and valid. Note
		that an empty default service config is a valid service config.
		2. If it does not have a default service config, it should treat the
		resolution attempt as having failed (e.g., for a polling-based
		resolver, it should retry the query after appropriate backoff). In
		other words,the channel will remain in state TRANSIENT_FAILURE until a
		valid service config is received.

### Criteria to determine the validity of a service config

A valid service config needs to be well formatted, i.e., the JSON representation
of the service config needs to be in a valid JSON format. If the service config
is well formatted, the client validates each field that it recognizes while
ignoring the unknown fields, i.e., unknown fields do not affect validation. If
any of the known fields fails in validation, the entire service config is deemed
invalid.

Future gRFCs that add new service config fields must define the validation
criteria for those fields. In a case where different clients may support
different values for a field, a gRFC may define a field to be a list of values,
where each client will use the first value in the list that it understands.

## Rationale

By discarding invalid service configs instead of ignoring fields that have
invalid values, we prevent clients from resorting to using default values which
might be unwanted for certain systems. Existing clients continuing to use older
service configs prevent an invalid service config from bringing down the
entire system, while newer clients which do not have a previously received
valid service config fail to start, avoiding new traffic for the duration that
the service is set with an invalid config.

Examples of invalid service configs:

* Badly Formatted service config
```
"serviceConfig" : {
  "MethodConfig" : {
    "Service // bad format
  }
```

* Fields with invalid values are not allowed
```
"serviceConfig" : {
  "loadBalancingPolicy" : “UnknownPolicy”
}
```

Note that loadBalancingPolicy is deprecated in favor of loadBalancingConfig,
which provides a list of LB policies with the client selecting the first policy
it supports. If it doesn’t understand any of the policies, it is treated as
invalid, and the entire service config is rejected.

```
"serviceConfig" : {
  "loadBalancingConfig" : [
    "UnknownPolicy1" : {},
    "UnknownPolicy2" : {}
  ]
}
```

Examples of valid service configs :
* Unknown fields are allowed
```
"serviceConfig" : {
  "UnknownField" : “value”, // Field is unknown and hence ignored
  "methodConfig" : {}
}
```


### Service Config Channel Arguments

gRPC allows two channel arguments that affect the behavior of service configs.
One argument provides a service config to use if the resolver does not return a
service config, and the other provides an option to disable service config
resolution.

If a default service config is provided, the client should fallback on the
default service config (in case of an invalid config with no valid service
config received earlier). On the other hand, if there is no default service
config provided as a channel argument, the client should fail to start and keep
waiting for a valid service config. Note that the default service config
being set to an empty service config would be a valid fallback.

## Implementation

To implement the above spec -
1. The resolver needs to be provided a parsing method(s) that it can run for a
service config. This function will be provided to the resolver at instantiation
time. This method will return a parsed form of the service config, or an error
indicating that parsing failed.
2. When a resolver gets an update, it runs the parser on this service config.
	1. If parsing succeeds, it returns the parsed result to the channel to
	apply.
	2. If parsing fails, it returns the parse error.
Irrespective of whether the parsing on the service config failed or not, the
resolver always returns the list of addresses from the update.

The load balancing policy API will also need to provide a way to parse the load
balancing part of the service config, where the first recognized load balancing
policy is parsed according to the requirements specified by that load balancing
policy. This method will return a parsed form of the first recognized load
balancing policy if parsing succeeds, or an error if parsing fails, as described
above. The parsed form will be what the load balancing policy API would see
again if the parsing of the rest of the service config was also successful.

### Avoiding Parsing the Service Config Twice
In a naive implementation, the parsing method and the method to apply the new
service config would both need to parse the service config. To avoid having to
perform this parsing twice, the parse method will return the result of the
parsing in some kind of object that when provided back to individual
sub-parts/filters can be reused. Once parsing is done, this parsed result (as
opposed to the original JSON representation) is provided back to the client.

This also avoids potential inconsistencies between the code that validates the
service config and the code that applies the service config. (We do not want to
fail at parsing the service config when we are trying to apply, since this would
require us to support partial roll-backs.)

C -

JAVA -

Go -

## Open issues (if applicable)

N/A
