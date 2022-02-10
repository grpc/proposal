A44: xDS Custom Backend Metrics Support
----
* Author(s): [Yifei Zhuang](), [Eric Anderson](https://github.com/ejona86)
* Approver: Eric Anderson
* Status: Ready for Implementation
* Implemented in: <language, ...>
* Last updated: 2022-
* Discussion at: https://groups.google.com/g/grpc-io/c/??

## Abstract
Support custom metrics injection at a gRPC server, and consumption at a gRPC client. 

Two metrics reporting mechanisms are supported:

* Per-query metrics reporting: the backend server attaches the injected custom metrics in the trailing metadata 
* when the corresponding RPC finishes. This is typically useful for short RPCs like unary calls.
* Out-of-band metrics reporting: the backend server periodically pushes metrics data, 
* e.g. cpu and memory utilization, to the client. This is useful for all situations: unary calls, 
* long RPCs in streaming calls, or no RPCs. The metrics emitting frequency is user-configurable, 
* and this configuration resides in the custom load balancing policy.


There are two types
of metrics involved: 
* Request cost metrics show the amount of resources it takes to handle a request, specific for each RPC. 
gRPC support reporting request cost via per-query metrics reporting mechanism only.
* Utilization metrics show the server status machine wide, independent of any RPC. gRPC support reporting
utilization metrics via both per-query (only suitable in high QPS systems) and OOB mechanisms.

## Background
### Related Proposals:

* [A36: xDS-Enabled Servers][A36]

[A36]: A36-xds-for-servers.md
[A39]: A39-xds-http-filters.md

## Proposal

## Rationale
## Implementation

This will be implemented simultaneously in Java by @YifeiZhuang, in
Go by @, and C++ by @.
