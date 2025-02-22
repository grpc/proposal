# A90: Add List Method to gRPC Health Service
----
* **Author(s):** @marcoshuck
* **Approver:** @markdroth
* **Status:** Draft  
* **Implemented in:** TBD
* **Last updated:** 2024-12-17  
* **Discussion at:** TBD


## Abstract

This proposal introduces a new `List` RPC method for the Health service, allowing clients to retrieve the statuses of all monitored services or a filtered list of specific services. This feature simplifies integration with status-reporting dashboards and enhances observability for microservices.

## Background

The existing Health service provides basic health check functionality but lacks a mechanism to retrieve a comprehensive list of all monitored services and their statuses.

This limitation makes it challenging for clients to aggregate health information across multiple services, particularly for use cases like publishing service statuses to dashboards such as [Cachet](https://cachethq.io/) or [Statuspage](https://www.atlassian.com/software/statuspage).

It's important to keep in mind that the list of health services exposed by an application can change over the lifetime of the process.

Kubernetes provides a similar capability with its `/readyz?verbose` endpoint, which lists the status of all components. This proposal aims to bring analogous functionality to the Health service, providing a unified view of service health.

### Related Proposals

N/A

## Proposal

Introduce a new `List` RPC method in the Health service with the following features:

- Retrieve the health statuses of all monitored services or a filtered list of specified services.
- Ensure the implementation is idempotent and side-effect free.
- Provide a clear schema for the request and response to facilitate integration with external tools.

### Proposed API Changes

```protobuf
message HealthListRequest {
  repeated string services = 1; // Contains the name of the services that should be obtained. If empty, must return all.
}

message HealthListResponse {
  map<string, HealthCheckResponse> statuses = 1; // Contains all the services and their respective status.
}


service Health {
  // List gets the health of all the available services. A filtered list can
  // be obtained by sending the list of the respective service names. If the
  // list of service names is empty, all the available services must be returned.
  //
  // Clients should set a deadline when calling List, and can declare the
  // server unhealthy if they do not receive a timely response.
  //
  //
  // List implementations should be idempotent and side effect free.
  rpc List(HealthListRequest) returns (HealthListResponse);
}
```

### Temporary environment variable protection

N/A

## Rationale

Adding the List RPC method to the Health service strikes a balance between functionality and simplicity.

### Alternative approaches
- Separate Service: Extract the new functionality into a standalone service. This avoids altering the existing Health service API but increases complexity by introducing another service for clients to interact with.

## Implementation

1. Add the `List` RPC method and the associated request/response messages in the Health service `.proto` file.
2. Update client libraries to support the `List` method. Provide examples demonstrating how to call the method and handle its response.
3. Implement `List` RPC method in the respective gRPC languages.
4. Unit tests to validate the behavior of the new method.
5. Update API documentation to describe the new method, its use cases, and sample requests/responses.

## Open issues (if applicable)

N/A
