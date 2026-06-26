A99: xDS Aggregate Cluster Resources
----
* Author(s): @markdroth
* Approver: @ejona86, @dfawley
* Status: {Draft, In Review, Ready for Implementation, Implemented}
* Implemented in: <language, ...>
* Last updated: 2025-06-18
* Discussion at: https://groups.google.com/g/grpc-io/c/NkAv5WGdxnk

## Abstract

We will add the ability to get the ordered list of underlying clusters
for an aggregate cluster from a separate resource, rather than being
inlined into the CDS resource.

## Background

As described in [A37], an xDS aggregate cluster is a cluster that
redirects to a prioritized list of underlying clusters, directing
traffic to the highest priority underlying cluster that is actually
reachable.

The list of underlying clusters for an aggregate cluster is encoded in a
[`ClusterConfig`
proto](https://github.com/envoyproxy/envoy/blob/82ccdbc0f4a78441e776a9fbb793b59831077f77/api/envoy/extensions/clusters/aggregate/v3/cluster.proto#L22).
Today, that proto is inlined into the CDS resource in the [`cluster_type`
field](https://github.com/envoyproxy/envoy/blob/82ccdbc0f4a78441e776a9fbb793b59831077f77/api/envoy/config/cluster/v3/cluster.proto#L833)
(embedded in the [`typed_config`
field](https://github.com/envoyproxy/envoy/blob/82ccdbc0f4a78441e776a9fbb793b59831077f77/api/envoy/config/cluster/v3/cluster.proto#L194)).
This causes problems in cases where the order of the underlying clusters
needs to be changed dynamically based on the health of the underlying
clusters, because many control planes restrict dynamic content
generation to EDS resources, whereas CDS resources are generally created
solely from static configuration.

### Related Proposals: 
* [A37: xDS Aggregate and Logical DNS Clusters][A37]
* [A75: xDS Aggregate Cluster Behavior Fixes][A75]
* [A74: xDS Config Tears][A74]

[A37]: A37-xds-aggregate-and-logical-dns-clusters.md
[A75]: A75-xds-aggregate-cluster-behavior-fixes.md
[A74]: A74-xds-config-tears.md

## Proposal

We will allow control planes to provide the list of underlying clusters
for an aggregate cluster in a separate resource, rather than having it
inlined into the CDS resource.

The necessary xDS API change has been made in
https://github.com/envoyproxy/envoy/pull/39669.  The idea is that
instead of having the CDS resource contain the `ClusterConfig` proto
directly, we will fetch the `ClusterConfig` proto as a separate xDS
resource.  The CDS proto will instead contain the
[`AggregateClusterResource`
proto](https://github.com/envoyproxy/envoy/blob/82ccdbc0f4a78441e776a9fbb793b59831077f77/api/envoy/extensions/clusters/aggregate/v3/cluster.proto#L36),
which will tell it the name of the `ClusterConfig` resource to fetch.

gRPC will require two main changes to support this:
- changes to the xDS resource types
- changes in `XdsDependencyManager`

### Changes to xDS Resource Types

First, we will move the code for parsing the `ClusterConfig` proto out
of the CDS resource type and into its own resource type.  The parsed
form of this resource type will be a list of strings indicating the
underlying cluster names.  In the parsed CDS resource, the aggregate
cluster will now be represented as a parsed `ClusterConfig` resource
rather than directly specifying the list of underlying cluster names.

Second, we will change the parsed representation of the CDS resource to
support the new `AggregateClusterResource` proto.  Today, the parsed
CDS resource supports three types of clusters: EDS, Logical DNS,
and Aggregate.  We will add a new type of cluster called an Aggregate
Resource cluster, whose contents will be the name of the `ClusterConfig`
resource to fetch.  This new type of cluster will be used when the CDS
resource cluster type is an `AggregateClusterResource` proto.

#### Parsed Resource Representations

As an example to clarify the intended parsed representation changes,
the current representation for the parsed CDS resource in C-core looks
like this:

```c++
// Parsed form of CDS resource.
struct XdsClusterResource {
  struct Eds { /*...*/ };

  struct LogicalDns { /*...*/ };

  struct Aggregate {  // Parsed form of ClusterConfig proto.
    std::vector<std::string> prioritized_cluster_names;
  };

  std::variant<Eds, LogicalDns, Aggregate> type;

  // ...other fields...
};
```

We will add a new xDS resource type for the `ClusterConfig` proto, which
will look like this:

```c++
// Parsed form of ClusterConfig proto.
struct XdsAggregateClusterConfigResource {
  std::vector<std::string> prioritized_cluster_names;
};
```

Then we will change the parsed representation of the CDS resource to
look like this:

```c++
// Parsed form of CDS resource.
struct XdsClusterResource {
  struct Eds { /*...*/ };

  struct LogicalDns { /*...*/ };

  struct Aggregate {
    // Note: This contains the parsed `ClusterConfig` resource, rather
    // than directly listing the underlying clusters.
    std::shared_ptr<XdsAggregateClusterConfigResource> cluster_config;
  };

  struct AggregateResource {
    std::string resource_name;
  };

  std::variant<Eds, LogicalDns, Aggregate, AggregateResource> type;

  // ...other fields...
};
```

Note that the code to parse the `ClusterConfig` proto can be called
either when that proto comes in its own resource or when it is inlined
into CDS.  This is similar to how the `RouteConfig` proto can either be
obtained via RDS or can be inlined into LDS.

### `XdsDependencyManager` Changes

As per [A74], the `XdsDependencyManager` is responsible for recursively
expanding the dependency tree of xDS resources into a cohesive
representation of the complete xDS configuration for the gRPC client.
As part of this proposal, when the `XdsDependencyManager` sees a parsed
CDS resource that represents an Aggregate Resource cluster, it will
start a watch for the specified `ClusterConfig` resource.  When it
receives the `ClusterConfig` resource, it will then continue processing
the same way that it would have if the parsed CDS resource had
represented a normal Aggregate cluster.

Note that the resulting `XdsConfig` struct generated by the
`XdsDependencyManager` should not require any changes, since it already
includes the list of leaf clusters for each aggregate cluster.

### Temporary environment variable protection

Support for the new `AggregateClusterResource` cluster type in CDS will
be guarded by the `GRPC_EXPERIMENTAL_XDS_AGGREGATE_CLUSTER_RESOURCE`
env var.  The env var guard will be removed when this feature passes
interop testing.

## Rationale

We considered a few options involving shoehoring Logical DNS
functionality into EDS to leverage the existing prioritization mechanism
in EDS.  However, that seemed like a poor fit, since EDS is really
intended to be a name service itself, not to forward to DNS.

## Implementation

Will be implemented in C-core, Java, Go, and Node.
