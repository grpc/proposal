# Add List endpoint to gRPC Health service

* **Author(s):** Marcos Huck.
* **Approver:** TBD
* **Status:** Draft  
* **Implemented in:** TBD
* **Last updated:** 2024-12-17  
* **Discussion at:** TBD


## Abstract

This proposal introduces a new `List` RPC endpoint for the Health service, allowing clients to retrieve the statuses of all monitored services or a filtered list of specific services. This feature simplifies integration with status-reporting dashboards and enhances observability for microservices.

## Background

The existing Health service provides basic health check functionality but lacks a mechanism to retrieve a comprehensive list of all monitored servicesand their statuses. 

This limitation makes it challenging for clients to aggregate health information across multiple services, particularly for use cases like publishing service statuses to dashboards such as [Cachet](https://cachethq.io/) or [Statuspage](https://www.atlassian.com/software/statuspage).

Kubernetes provides a similar capability with its `/readyz?verbose` endpoint, which lists the status of all components.This proposal aims to bring analogous functionality to the Health service, providing a unified view of service health.

### Related Proposals

N/A

## Proposal

Introduce a new `List` RPC endpoint in the Health service with the following features:

- Retrieve the health statuses of all monitored services or a filtered list of specified services.
- Ensure the implementation is idempotent and side-effect free.
- Provide a clear schema for the request and response to facilitate integration with external tools.

### Proposed API Changes

https://github.com/grpc/grpc-proto/pull/143

### Temporary environment variable protection

N/A

## Rationale

Adding the List RPC endpoint to the Health service strikes a balance between functionality and simplicity.

### Alternative approaches
- Separate Service: Extract the new functionality into a standalone service. This avoids altering the existing Health service API but increases complexity by introducing another service for clients to interact with.

## Implementation

1. Add the `List` RPC endpoint and the associated request/response messages in the Health service .proto file.
2. Update client libraries to support the `List` endpoint. Provide examples demonstrating how to call the endpoint and handle its response.
3. Unit tests to validate the behavior of the new endpoint.
4. Update API documentation to describe the new endpoint, its use cases, and sample requests/responses.

## Open issues (if applicable)

N/A
