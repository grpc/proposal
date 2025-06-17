## A96: OTel Metrics for Retries

*   Author: Yash Tibrewal (@yashykt)
*   Approver: Mark Roth (@markdroth)
*   Status: In Review
*   Implemented in:
*   Last updated: 2025-05-16
*   Discussion at: https://groups.google.com/g/grpc-io/c/bFUHkcBA9cw

## Abstract

Propose OpenTelemetry metrics for gRPC retries.

## Background

Since OpenCensus has been sunsetted in favor of OpenTelemetry, gRPC has been
developing its own OpenTelemetry plugin that is meant to replace the OpenCensus
plugin ([A66]). This document proposes the OpenTelemetry version of the retry
metrics originally proposed in [A45].

### Related Proposals:

*   [A45]: Exposing OpenCensus Metrics and Tracing for gRPC retry
*   [A66]: OpenTelemetry Metrics
*   [A79]: Non-per-call Metrics Architecture

[A45]: A45-retry-stats.md
[A66]: A66-otel-stats.md
[A79]: A79-non-per-call-metrics-architecture.md

## Proposal

Metric Name                          | Type      | Unit                | Labels                                         | Description
------------------------------------ | --------- | ------------------- | ---------------------------------------------- | -----------
grpc.client.call.retries             | Histogram | {retry}             | grpc.method (required), grpc.target (required) | Number of retries during the client call. If there were no retries, 0 is not reported. Recommended histogram bucket boundaries are [1,2,3,4,5].
grpc.client.call.transparent_retries | Histogram | {transparent_retry} | grpc.method (required), grpc.target (required) | Number of transparent retries during the client call. If there were no transparent retries, 0 is not reported. Recommended histogram bucket boundaries are [1,2,3,4,5,10].
grpc.client.call.hedges              | Histogram | {hedge}             | grpc.method (required), grpc.target (required) | Number of hedges during the client call. If there were no hedges, 0 is not reported. Recommended histogram bucket boundaries are [1,2,3,4,5].
grpc.client.call.retry_delay         | Histogram | s                   | grpc.method (required), grpc.target (required) | Total time of delay while there is no active attempt during the client call. Recommended to use exponential histograms if available, otherwise, use latency buckets defined in [A66].

The labels `grpc.method` and `grpc.target` have been defined in [A66].

These metrics are recorded at the end of the call utilizing the `CallTracer`
approach (also defined in [A66]).

### Stability

As recommended by [A79], these metrics will start off as experimental, and hence
off-by-default. The decision on whether these metrics will be on-by-default or
off-by-default on de-experimentalization will be made at the same time as the
de-experimentalization.

## Rationale

OpenCensus Metric                           | Equivalent OpenTelemetry Metric
------------------------------------------- | -------------------------------
grpc.io/client/retries_per_call             | grpc.client.call.retries
grpc.io/client/retries                      | Sum of grpc.client.call.retries
grpc.io/client/transparent_retries_per_call | grpc.client.call.transparent_retries
grpc.io/client/transparent_retries          | Sum of grpc.client.call.retries
grpc.io/client/retry_delay_per_call         | grpc.client.call.retry_delay

The names of the metrics proposed for the OpenTelemetry version follows the
general OpenTelemetry
[semantic conventions](https://opentelemetry.io/docs/specs/semconv/general/metrics/),
[naming guidelines](https://opentelemetry.io/docs/specs/semconv/general/naming/),
the existing per-call gRPC OpenTelemetry metrics (proposed in [A66]) and the
gRPC OpenTelemetry metric instrument naming conventions (proposed in [A79]).

### Separate metrics for retries and hedges

The OpenCensus version of the retry metrics combined the number of retry and
hedging attempts under a single OpenCensus measure
`grpc.io/client/retries_per_call` and had a separate measure for transparent
retries `grpc.io/client/transparent_retries_per_call`. Since different
channels/clients can be configured differently, for example, some with a retry
policy while others with a hedging policy, it is useful to differentiate between
retry attempts and hedging attempts as well. This would also help in the future
if we allow both retry and hedging policies to be configured on a client at the
same time.

### Label addition - grpc.target

The OpenCensus version did not have an equivalent `grpc.target` label on the
retry metrics. This label has been added to the OpenTelemetry version keeping in
line with the other per-call metrics defined in [A66].

## Implementation

*   C++ - Will be implemented by @yashykt
*   Java - Will be implemented by @agravator
*   Go - TBD
*   Python - TBD
