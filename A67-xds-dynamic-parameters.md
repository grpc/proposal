## xDS Dynamic Parameters

*   Author(s): @allanrbo
*   Approver: @markdroth
*   Status: Draft
*   Implemented in:
*   Last updated: 2023-07-21
*   Discussion at:

## Abstract

Dynamic parameters is a mechanism for a resource identified by a single name,
e.g. `xdstp://xds.example.com/envoy.config.listener.v3.Listener/foo`, to have
dynamically generated content, while still preserving cacheability. An example
of a dynamic parameter could be `{key: "env", value: "prod"}`.

We will add support for dynamic parameters in gRPC in all languages. For this,
we will extended the bootstrap JSON, and modify how ADS requests are constructed
and how ADS responses are parsed.

## Background

[TP2](https://github.com/cncf/xds/blob/main/proposals/TP2-dynamically-generated-cacheable-xds-resources.md)
describes in detail how dynamic parameters will function. While TP2 discusses
the functionality in terms of xDS generally, this document instead goes into
details about what more specifically in gRPC will need to be modified.

This document assumes gRPC's `XdsClient` is only used by leaf clients. If at
some point gRPC's `XdsClient` gets reused in an xDS proxy, additional
functionality for processing dynamic parameter constraints will need to be
added.

TP2 suggests that:

-   Subscriptions to xdstp named resources allow for a separate set of dynamic
    parameters per authority.
-   Subscriptions to non-xdstp named resources use a set of dynamic parameters
    configured globally for the client.

We will follow this suggestion.

### Request

Currently a request that basically look like this:

```text
resource_names: "xdstp://xds.example.com/envoy.config.listener.v3.Listener/foo"
type_url: "type.googleapis.com/envoy.config.listener.v3.Listener"
```

With dynamic parameters it will look like this:

```text
resource_locators {
  name: "xdstp://xds.example.com/envoy.config.listener.v3.Listener/foo"
  dynamic_parameters: [ {key: "env", value: "prod"} ]
}
type_url: "type.googleapis.com/envoy.config.listener.v3.Listener"
```

### Response

TP2 specifies that xDS servers responding to requests containing dynamic
parameters should wrap returned resources in Resource messages, and use the
`resource_name` field instead of the name field.

When we make a request with `resource_names` as we currently do, we expect to
get a response like this:

```text
type_url: "type.googleapis.com/envoy.config.listener.v3.Listener"
resources {
  [type.googleapis.com/envoy.config.listener.v3.Listener] {
    name: "xdstp://xds.example.com/envoy.config.listener.v3.Listener/foo"
    ...
  }
}
```

With dynamic parameters, we should instead be getting a response with a
Resource-wrapper like this:

```text
type_url: "type.googleapis.com/envoy.config.listener.v3.Listener"
resources {
  [type.googleapis.com/envoy.service.discovery.v3.Resource] {
    resource_name {
      name: "xdstp://xds.example.com/envoy.config.listener.v3.Listener/foo"
      dynamic_parameter_constraints {
        constraint { key: "env", value: "prod" }
      }
    }
    resource: {
      [type.googleapis.com/envoy.config.listener.v3.Listener] {
        name: "xdstp://xds.example.com/envoy.config.listener.v3.Listener/foo"
        ...
      }
    }
  }
}
```

### Related Proposals:

*   CNCF xDS proposal
    [TP2: Dynamically Generated Cacheable xDS Resources](https://github.com/cncf/xds/blob/main/proposals/TP2-dynamically-generated-cacheable-xds-resources.md).

## Proposal

We can extended the format of the bootstrap JSON file like highlighted with ">"
below:

```text
 {
   "authorities": {
     "xds.example.com": {
       "xds_servers": [
         {
           "server_uri": "xds.example.com:443",
           "channel_creds": [ { "type": "google_default" } ]
         }
       ],
>      "dynamic_parameters": {
>        "env": "prod"
>      }
     }
   },
   "node": {
     "id": "projects/123456789123/networks/default/nodes/4d5f8bd2-3e91-4940-855a-fdabca4c2f70"
   }
 }
```

To support non-xdstp use cases, we can add a map at the root level too:

```text
 {
   "xds_servers": [
     {
       "server_uri": "xds.example.com:443",
       "channel_creds": [ { "type": "google_default" } ]
       ],
       "server_features": [
         "xds_v3"
       ]
     }
   ],
   "node": {
     "id": "projects/123456789123/networks/default/nodes/4d5f8bd2-3e91-4940-855a-fdabca4c2f70"
   },
>  "dynamic_parameters": {
>    "env": "prod"
>  }
 }
```

The `XdsClient` is currently structured as follows in all languages:

-   There is a bootstrap config known by the `XdsClient` object. The bootstrap
    config contains settings for each authority.
-   We have a shared pool of xDS channel objects, so if multiple authorities use
    the same xDS server, they will share the same xDS channel object.
-   The code that constructs the ADS request already knows the set of resource
    names to send, and that list may include resources from multiple authorities
    that are sharing the same xDS server.

The changes this design proposes are:

-   The bootstrap config will contain the dynamic parameters to use for each
    authority.
-   The code to construct the ADS request will be changed such that for each
    authority for which there are resource names in the request, it looks up the
    dynamic parameters for that authority in the bootstrap config. If the
    dynamic parameters are not empty, that will trigger using the
    `resource_locators` field instead of the `resource_names` field in the
    request
-   The code to parse the ADS response will be changed to look in the
    `resource_name` field and use this instead of the name field if it exists.
    As this `XdsClient` is only a leaf client, and not a caching proxy, we can
    ignore the dynamic_parameter_constraints field of the ResourceName message.

The updated code to construct the ADS request would look something like this
(pseudo code):

```text
// Example authority_resource_names:
[
  // Resources for authority "foo.example.com"
  { resource_names: ["bar", "baz"], dynamic_parameters: {"env": "prod"} },

  // Resources for authority "zzz.example.com"
  { resource_names: ["yyy"], dynamic_parameters: {"env": "prod", "version": "v2"} },
]

function ConstructADSRequest(authority_resource_names)
  ...
  foreach resource_names, dynamic_parameters in authority_resource_names
    if dynamic_parameters is empty
      request.resource_names = resource_names
    else
      foreach name in resource_names
        request.resource_locators.add(name, dynamic_parameters)
  ...
```

### Temporary environment variable protection

An environment variable to enable/disable this feature should not be necessary,
as this feature is only activated if the new `dynamic_parameters` field is
present in the bootstrap JSON.

## Rationale

The following sub-sections describe tradeoffs that were made in this design.

### No changas to dynamic parameters after startup

For our initial support of dynamic parameters, we can let the parameter
key-value pairs be only read at startup. A client restart will be required when
changing the parameters.

### No metadata reuse

An idea in TP2 is to reuse the node metadata field as dynamic parameters. I
propose we do not do this, as there is no way of knowing whether a given control
plane supports dynamic parameters.

### No merging of top-level and per-authority dynamic parmeters

The `dynamic_parameters` at root level only apply to the non-xdstp case. There
will be no "inheritance" or merging of the top level `dynamic_parameters` to the
`dynamic_parameters` inside the authority objects. If a user wants the same
parameters sent in the non-xdstp and xdstp case, they will need to specify the
parameters both places. The reasoning behind this is that it provides the
flexibility of when a user wants a parameter for the non-xdstp case, but doesn't
want this parameter for the xdstp case.

## Implementation

Allan Boll (allanrbo@google.com) plans to implement this in C++ in 2023 Q3.
