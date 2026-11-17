---
name: shp-observability
description: Make Go services and Lambda functions diagnosable in production - structured logging with log/slog, RED/USE metrics, OpenTelemetry tracing, CloudWatch Embedded Metric Format, health checks, SLO-based alerts. Use when adding logging, metrics, tracing or alerts, when shipping a feature to production, or when an incident cannot be explained from available data.
license: MIT
metadata:
  kit: shuppatsu-go
  version: "1.1.0"
  inspired_by: "https://github.com/addyosmani/agent-skills (observability-and-instrumentation, MIT)"
---

# shp-observability

If you cannot tell from telemetry whether a feature works, it is not finished. Instrument as part of the change, not after the first incident.

## The three signals

| Signal | Answers | Go and AWS tooling |
|---|---|---|
| **Logs** | What happened in this request? | `log/slog` JSON handler → stdout → CloudWatch Logs |
| **Metrics** | Is it healthy over time? | CloudWatch metrics via Embedded Metric Format (EMF), or OpenTelemetry metrics |
| **Traces** | Where did the time go across services? | OpenTelemetry Go SDK + ADOT/X-Ray (`otelhttp`, `otelaws` for SDK v2) |

## Logging with `log/slog`

```go
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelInfo}))
slog.SetDefault(logger)

// Per request: attach stable identifiers once.
l := logger.With("request_id", reqID, "order_id", orderID)
l.InfoContext(ctx, "order accepted", "items", len(items), "duration_ms", time.Since(start).Milliseconds())
l.ErrorContext(ctx, "charge failed", "err", err) // log at the boundary that handles the error, once
```

Rules:
- Key-value attributes, never string-built messages; constant message text so logs can be grouped.
- Levels: `Error` = needs action, `Warn` = degraded but handled, `Info` = business events, `Debug` = off in production.
- **Never log secrets, tokens, full payloads or personal data.** Redact with `slog.LogValuer` on sensitive types.
- Handle an error once: log it **or** return it, not both (`shp-go-style`).
- Propagate a request/correlation ID through `context.Context`; in Lambda use `lambdacontext.FromContext(ctx).AwsRequestID`.

## Metrics: RED for services, USE for resources

- **RED:** Rate (requests/s), Errors (failed/s), Duration (latency percentiles p50/p95/p99).
- **USE:** Utilisation, Saturation, Errors for queues, pools, CPU, memory.
- Business counters (orders created, payments failed) beside technical ones.
- Keep label cardinality low: no user IDs, request IDs or free text as dimensions.
- On Lambda, prefer EMF (a JSON log line CloudWatch converts to metrics): no API call, no latency. Use the built-in `Duration`, `Errors`, `Throttles`, `ConcurrentExecutions`, and SQS `ApproximateAgeOfOldestMessage` too.

## Tracing

- Wrap inbound handlers (`otelhttp.NewHandler`, Lambda instrumentation) and outbound clients (`otelhttp.NewTransport`, `otelaws.AppendMiddlewares`).
- Pass `ctx` everywhere so spans connect; add span attributes for business keys, not payloads.
- Sample in production (parent-based, ratio); keep 100% of errors if the backend allows.

## Health and readiness

- `/healthz` (process alive) and `/readyz` (dependencies reachable) for long-running services; do not make liveness depend on downstreams.
- Graceful shutdown on `SIGTERM`: stop accepting, drain in-flight work with a deadline (`shp-patterns-concurrency`).

## Alerts

- Alert on **symptoms** users feel (error ratio, latency SLO burn, DLQ depth > 0, oldest message age), not on every cause.
- Every alert has an owner, a runbook link and a clear action; delete alerts nobody acts on.
- Define SLOs (e.g. 99.9% of requests succeed in < 500 ms over 30 days) and alert on burn rate.

## Anti-rationalization

| Excuse | Rebuttal |
|---|---|
| "We can add logging when something breaks" | The first failure will have no data. |
| "Log everything to be safe" | Cost, noise and privacy risk; log events with context, not dumps. |
| "Metrics with the user ID are more useful" | They explode cost and cardinality; put IDs in logs and traces. |
| "Alert on every error" | Pager fatigue; alert on symptoms and SLO burn. |

## Verification gate (evidence required)

- A test or run shows a log line with the request ID and the expected attributes.
- The failure path emits an error log and increments an error metric.
- No secret or personal data in the sampled log output.
- A named dashboard or query (CloudWatch Logs Insights) answers "is it healthy" for this feature.
- `go vet ./...` and `go test ./...` pass.

## Sources

- `log/slog`: https://go.dev/blog/slog , `go doc log/slog`
- OpenTelemetry Go: https://opentelemetry.io/docs/languages/go/
- `otelaws` (AWS SDK v2 instrumentation): https://pkg.go.dev/go.opentelemetry.io/contrib/instrumentation/github.com/aws/aws-sdk-go-v2/otelaws
- CloudWatch Embedded Metric Format: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Embedded_Metric_Format_Specification.html
- Lambda logging for Go: https://docs.aws.amazon.com/lambda/latest/dg/golang-logging.html
- Lambda monitoring and X-Ray: https://docs.aws.amazon.com/lambda/latest/dg/monitoring-functions.html
- Google SRE book, monitoring and SLOs: https://sre.google/sre-book/monitoring-distributed-systems/
- RED method: https://grafana.com/blog/the-red-method-how-to-instrument-your-services/

## Inspired by

Three-signal structure, RED/USE and symptom-based alerting inspired by `observability-and-instrumentation` in addyosmani/agent-skills (MIT) and the Google SRE book. Go, `slog` and AWS mapping are original. See `THIRD_PARTY_NOTICES.md`.
