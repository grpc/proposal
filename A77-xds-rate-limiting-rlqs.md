A77: xDS Server-Side Rate Limiting
======

* Author(s): Sergii Tkachenko (sergiitk@)
* Approver: Mark Roth (@markdroth)
* Status: Draft
* Implemented in:
* Last updated: 2024-01-24
* Discussion at: `TODO(sergiitk): <google group thread>`

## Abstract

This proposal introduces support for the
[quota-based global rate limit](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/rate_limit_quota_filter)
feature to [xDS-Enabled gRPC servers][A36].\
Main characteristics of the feature:

* Only covers server-side rate limiting.
* Can only be applied to HTTP-level (L7) requests.
* Rate limit decisions are offloaded to a remote Rate Limit Quota Service (RLQS)
  asynchronously.
* RLQS maintains a global view of the shared rate limit resource, and fairly
  distributes it among multiple servers (best-effort).

## Background

[Global rate limiting](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/other_features/global_rate_limiting)
allows mesh users to manage fair consumption of their services and prevent
misbehaving clients from overloading the services. *Global* rate limiting
implies that there must be a single source of truth for the state of a finite
shared resource; a Rate Limiting Service. As of 2024-01-24, Envoy describes two
types of such service:

1. [Rate Limiting Service (RLS)](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/other_features/global_rate_limiting#per-connection-or-per-http-request-rate-limiting):
   synchronous (blocking) per-HTTP-request rate limit check. Best suited for
   precise cases, where even a single request over the limit must be throttled.
2. [Rate Limiting Quota Service (RLQS)](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/other_features/global_rate_limiting#quota-based-rate-limiting):
   asynchronous (non-blocking) quota-based check, with periodic load reports of
   bucketed HTTP requests. Best suited for high-request-per-second applications,
   where a certain margin of error is acceptable as long as expected average QPS
   is achieved.

gRPC only intends to support quota-based global rate limiting, Rate Limiting
Quota Service (RLQS), which is described in this document.

This proposal details how gRPC will add support
for [Rate Limit Quota xDS HTTP filter][rate_limit_quota_filter]
configured via [xDS HTTP Filters][A39] to [xDS-Enabled gRPC servers][A36]. It
will cover four major parts needed for language-specific gRPC implementations:

1. Support missing xDS types needed to parse the Rate Limit Quota Filter config.
2. Implement the client side of RLQS protocol (RLQS
   Client): `StreamRateLimitQuotas.StreamRateLimitQuotas`. It will establish
   bidirectional gRPC stream to the remote [Rate Limit Quota Service][rlqs].
3. Implement a Server Interceptor (filter in C-core, later called
   "the Interceptor" for simplicity) that requests.
4. Send and receive updates.

### Related Proposals:

* [A36: xDS-Enabled Servers][A36]
* [A39: xDS HTTP Filter Support][A39]

[A36]: A36-xds-for-servers.md

[A39]: A39-xds-http-filters.md

## Proposal

### xDS types support
#### Unified Matchers
#### io.envoyproxy.envoy.config.core.v3.GrpcService.GoogleGrpc
#### Canonical CEL

TODO(sergiitk): A precise statement of the proposed change.

#### Simplified RLQS flow happy-path:

// TODO(sergiitk): Insert graphics

1. [Rate Limit Quota xDS HTTP filter][rate_limit_quota_filter]
   is configured on a gRPC Server via an LDS update in the HTTP connection
   manager filter chain.
2. gRPC Server establishes a gRPC channel to the RLQS as specified
   in [RateLimitQuotaFilterConfig.rlqs_server].
3. gRPC Server parses [RateLimitQuotaFilterConfig.bucket_matchers] tree and
   caches it in the filter state.
4. gRPC Server installs a Server Interceptor (filter in C-core, later called
   "the Interceptor" for simplicity)
5. Once a request is intercepted by the Interceptor:
    - The request is matched into a Bucket by evaluating the `bucket_matchers`
      tree against the request attributes.
    - If the Bucket doesn't exist, gRPC server immediately sends the
      initial `BucketQuotaUsage` report to the RLQS server. The request is
      throttled according
      to [RateLimitQuotaBucketSettings.no_assignment_behavior].
    - If the Bucket exists, the request is throttled according to Bucket's quota
      assignment. Bucket's `num_requests_allowed`
      or `num_requests_denied` request counter is increased by one.
6. For all existing buckets, a `BucketQuotaUsage` report is sent
   every [RateLimitQuotaBucketSettings.reporting_interval]
   to `RateLimitQuotaService.StreamRateLimitQuotas`.
7. RLQS may send Bucket quota assignments via `RateLimitQuotaResponse` at any
   time. Once received, it must the quota must be applied to a Bucket with
   matching [bucket_id].

### Temporary environment variable protection

During initial development, this feature will be enabled via the
`GRPC_EXPERIMENTAL_XDS_ENABLE_RLQS` environment variable. This environment
variable protection will be removed once the feature has proven stable.

## Rationale

TODO(sergiitk): A discussion of alternate approaches and the trade offs,
advantages, and disadvantages of the specified approach.

## Implementation

The initial implementation will be in Java.\
C-core, Python, and Go will follow after that.

## Open issues (if applicable)

TODO(sergiitk): A discussion of issues relating to this proposal for which the
author does not know the solution. This section may be omitted if there are
none.

<!-- Reference links -->

[rate_limit_quota_filter]: https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/rate_limit_quota_filter

[RateLimitQuotaBucketSettings.domain]: https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/http/rate_limit_quota/v3/rate_limit_quota.proto.html#envoy-v3-api-field-extensions-filters-http-rate-limit-quota-v3-ratelimitquotafilterconfig-domain

[RateLimitQuotaBucketSettings.no_assignment_behavior]: https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/http/rate_limit_quota/v3/rate_limit_quota.proto#envoy-v3-api-field-extensions-filters-http-rate-limit-quota-v3-ratelimitquotabucketsettings-no-assignment-behavior

[RateLimitQuotaBucketSettings.reporting_interval]: https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/http/rate_limit_quota/v3/rate_limit_quota.proto#envoy-v3-api-field-extensions-filters-http-rate-limit-quota-v3-ratelimitquotabucketsettings-reporting-interval

[RateLimitQuotaFilterConfig.rlqs_server]: https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/http/rate_limit_quota/v3/rate_limit_quota.proto#envoy-v3-api-field-extensions-filters-http-rate-limit-quota-v3-ratelimitquotafilterconfig-rlqs-server

[RateLimitQuotaFilterConfig.bucket_matchers]: https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/http/rate_limit_quota/v3/rate_limit_quota.proto#envoy-v3-api-field-extensions-filters-http-rate-limit-quota-v3-ratelimitquotafilterconfig-bucket-matchers

[bucket_id]: https://www.envoyproxy.io/docs/envoy/latest/api-v3/service/rate_limit_quota/v3/rlqs.proto#envoy-v3-api-field-service-rate-limit-quota-v3-ratelimitquotaresponse-bucketaction-bucket-id

[rlqs]: https://www.envoyproxy.io/docs/envoy/latest/api-v3/service/rate_limit_quota/v3/rlqs.proto
