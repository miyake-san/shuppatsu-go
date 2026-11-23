---
name: shp-aws-observability
description: AWS observability for Go - CloudWatch Logs and Logs Insights, CloudWatch Embedded Metric Format, X-Ray and ADOT tracing, otelaws for the AWS SDK v2, Lambda request IDs and built-in Lambda and SQS metrics, CloudWatch alarms. Use when instrumenting Go on Lambda, ECS or EKS with AWS tooling, building dashboards or alarms, or debugging an AWS-hosted Go service from logs and traces.
license: MIT
metadata:
  kit: shuppatsu-go
  version: "1.3.0"
  inspired_by: "https://github.com/addyosmani/agent-skills (observability-and-instrumentation, MIT)"
---

# shp-aws-observability

AWS implementation of `shp-observability` (logs, metrics, traces, alerts). Read that skill first for the generic rules (`slog`, RED/USE, cardinality, SLOs); this one maps them to AWS services. For SDK setup see `shp-aws-sdk`; for Lambda handlers see `shp-aws-lambda`.

## Signal to AWS mapping

| Signal | AWS tooling |
|---|---|
| Logs | `log/slog` JSON handler → stdout → CloudWatch Logs; query with Logs Insights |
| Metrics | CloudWatch metrics via Embedded Metric Format (EMF), or OpenTelemetry metrics through ADOT |
| Traces | OpenTelemetry Go SDK + ADOT Collector → AWS X-Ray (`otelhttp`, `otelaws` for SDK v2); Lambda active tracing |
| Alerts | CloudWatch alarms (metric and composite), routed to SNS |

## Logs: CloudWatch Logs

- Lambda captures stdout/stderr; use the JSON handler so fields are queryable. Configure the function's log format and log level (JSON, `INFO`) in the function settings.
- Use the Lambda request ID as the correlation ID:

```go
import "github.com/aws/aws-lambda-go/lambdacontext"

if lc, ok := lambdacontext.FromContext(ctx); ok {
	logger = logger.With("request_id", lc.AwsRequestID)
}
```

- Set a **retention period** on every log group; the default is never expire.
- Logs Insights example: `fields @timestamp, @message | filter level = "ERROR" | stats count() by bin(5m)`.

## Metrics: Embedded Metric Format

On Lambda, prefer EMF: a JSON log line that CloudWatch converts to metrics with no API call and no added latency.

```go
type emf struct {
	AWS struct {
		Timestamp         int64 `json:"Timestamp"`
		CloudWatchMetrics []struct {
			Namespace  string              `json:"Namespace"`
			Dimensions [][]string          `json:"Dimensions"`
			Metrics    []map[string]string `json:"Metrics"`
		} `json:"CloudWatchMetrics"`
	} `json:"_aws"`
	// metric values and dimension values are top-level fields
}
```

- Keep dimensions low-cardinality (service, operation, outcome); put IDs in log fields, not dimensions.
- Also watch the built-in metrics: Lambda `Duration`, `Errors`, `Throttles`, `ConcurrentExecutions`, `IteratorAge`; SQS `ApproximateAgeOfOldestMessage` and `ApproximateNumberOfMessagesVisible` (including the DLQ).
- For services on ECS or EKS, use ADOT or the CloudWatch agent to ship OpenTelemetry or Prometheus metrics.

## Traces: X-Ray and ADOT

- Wrap inbound handlers (`otelhttp.NewHandler`, Lambda instrumentation) and instrument the AWS SDK v2 clients with `otelaws.AppendMiddlewares`:

```go
cfg, _ := config.LoadDefaultConfig(ctx)
otelaws.AppendMiddlewares(&cfg.APIOptions)
```

- On Lambda, enable active tracing or use the ADOT Lambda layer; on containers, run the ADOT Collector as a sidecar.
- Pass `ctx` everywhere so spans connect; the X-Ray trace header propagates across supported AWS services.
- Sample in production; keep error traces.

## Alarms

- Alarm on symptoms: Lambda error rate, p99 duration near the timeout, throttles, DLQ depth > 0, oldest SQS message age, API Gateway 5XX rate and latency.
- Use metric math for ratios (errors / invocations) and composite alarms to cut noise.
- Every alarm has an owner and a runbook link; route to SNS (and from there to chat or paging).
- Alarm on missing data for heartbeat-style metrics only (treat missing data as breaching deliberately).

## Anti-patterns

- Calling `PutMetricData` from every Lambda invocation (cost, latency, throttling); use EMF.
- Log groups with infinite retention and DEBUG logging in production.
- High-cardinality EMF dimensions (user ID, request ID).
- Alarms without an owner or action.

## Verification gate (evidence required)

- A Logs Insights query returns the request's log lines by request ID.
- The failure path emits an error log and an EMF (or OTel) error metric; an alarm is defined for it in the IaC.
- Log group retention is set; no secrets or personal data in sampled logs.
- A trace shows the handler span and the downstream AWS calls (`otelaws`) as children.
- `go vet ./...` and `go test ./...` pass.

## Sources

- Lambda logging for Go: https://docs.aws.amazon.com/lambda/latest/dg/golang-logging.html
- Lambda monitoring and X-Ray: https://docs.aws.amazon.com/lambda/latest/dg/monitoring-functions.html
- CloudWatch Embedded Metric Format: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Embedded_Metric_Format_Specification.html
- CloudWatch Logs Insights query syntax: https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax.html
- CloudWatch alarms: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/AlarmThatSendsEmail.html
- AWS Distro for OpenTelemetry (Go): https://aws-otel.github.io/docs/getting-started/go-sdk
- `otelaws` (AWS SDK v2 instrumentation): https://pkg.go.dev/go.opentelemetry.io/contrib/instrumentation/github.com/aws/aws-sdk-go-v2/otelaws
- `lambdacontext`: `go doc github.com/aws/aws-lambda-go/lambdacontext`
- Generic rules: `shp-observability`

## Inspired by

Three-signal structure, RED/USE and symptom-based alerting inspired by `observability-and-instrumentation` in addyosmani/agent-skills (MIT) and the Google SRE book. AWS mapping is original. See `THIRD_PARTY_NOTICES.md`.
