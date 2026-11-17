---
name: shp-patterns-resilience
description: Resilience and distributed-systems patterns for Go services on AWS - timeouts, retries with exponential backoff and jitter, idempotency keys, circuit breaker, bulkhead, transactional outbox, saga with Step Functions, dead-letter queues, and partial batch failures for SQS. Use when calling remote services or AWS APIs, processing events or queues, designing failure handling in design.md, or when the user asks how to make code safe to retry.
license: MIT
metadata:
  kit: shuppatsu-go
  version: "1.0.0"
---

# shp-patterns-resilience

Assume every remote call can be slow, fail, or succeed without you knowing. Resilience is layered: **timeout → retry (only if safe) → idempotency → isolation → compensation**.

## Decision guide

| Failure mode | Pattern |
|---|---|
| Call hangs | Timeout via context deadline and client timeouts |
| Transient error or throttling | Retry with exponential backoff and jitter (SDK retryer) |
| Retry may duplicate side effects | Idempotency key + conditional write |
| Dependency is down; retries pile up | Circuit breaker, fail fast |
| One slow dependency starves others | Bulkhead: separate pools/limits per dependency |
| DB write and event publish must both happen | Transactional outbox |
| Multi-step business transaction across services | Saga (Step Functions) with compensations |
| Poison messages | Dead-letter queue + alarm |
| One bad record in a batch | Partial batch response (SQS + Lambda) |

## Timeouts

```go
httpClient := &http.Client{Timeout: 5 * time.Second}

ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
defer cancel()
out, err := client.GetItem(ctx, in)
```

Set the timeout below the caller's own deadline (API Gateway, Lambda timeout) so you can still return a meaningful error.

## Retries with the AWS SDK for Go v2

The SDK already retries throttling and transient errors with exponential backoff and jitter (standard mode). Tune it instead of writing your own loop:

```go
cfg, err := config.LoadDefaultConfig(ctx,
	config.WithRetryMaxAttempts(5),
	config.WithRetryMode(aws.RetryModeAdaptive), // adds client-side rate limiting under throttling
)
```

For non-AWS calls, retry only idempotent operations, cap attempts, use full jitter, and stop when the context is done:

```go
func Retry(ctx context.Context, attempts int, base time.Duration, op func(context.Context) error) error {
	var err error
	for i := range attempts {
		if err = op(ctx); err == nil || !isRetryable(err) {
			return err
		}
		backoff := base << i
		sleep := time.Duration(rand.Int64N(int64(backoff))) // full jitter, math/rand/v2
		select {
		case <-time.After(sleep):
		case <-ctx.Done():
			return errors.Join(err, ctx.Err())
		}
	}
	return err
}
```

## Idempotency

- Accept an idempotency key from the client (header or message attribute) or derive it from the event ID.
- Record it with a conditional write so a duplicate is detected atomically:

```go
_, err := db.PutItem(ctx, &dynamodb.PutItemInput{
	TableName:           aws.String(table),
	Item:                item, // includes pk = "IDEMPOTENCY#<key>", ttl
	ConditionExpression: aws.String("attribute_not_exists(pk)"),
})
var ccf *types.ConditionalCheckFailedException
if errors.As(err, &ccf) {
	return previousResult(ctx, key) // duplicate: return the stored outcome
}
```

- Store a TTL so idempotency records expire.
- Lambda event sources (SQS, EventBridge, SNS) deliver **at least once**: handlers must be idempotent.

## Circuit breaker and bulkhead

- A breaker counts recent failures per dependency and rejects calls for a cool-down period once a threshold is passed. Use a small, well-tested library (for example `github.com/sony/gobreaker`) rather than a hand-rolled one.
- Bulkhead: give each dependency its own concurrency limit (`errgroup.SetLimit`, semaphores, separate HTTP clients), so one slow dependency cannot exhaust all goroutines or connections.

## Transactional outbox

1. In the same transaction as the state change, write an outbox record (DynamoDB `TransactWriteItems`, or the same SQL transaction).
2. A relay publishes outbox records to SNS/EventBridge/SQS. With DynamoDB, DynamoDB Streams + Lambda is the relay.
3. Consumers are idempotent because the relay can publish twice.

## Saga

Model long-running, multi-service workflows in AWS Step Functions: each step is a Lambda/integration with `Retry` and `Catch`, and failures trigger compensating steps (refund, release stock). Keep compensations idempotent too.

## Queues, DLQs and partial batch failures

```go
func handler(ctx context.Context, ev events.SQSEvent) (events.SQSEventResponse, error) {
	var resp events.SQSEventResponse
	for _, rec := range ev.Records {
		if err := process(ctx, rec); err != nil {
			resp.BatchItemFailures = append(resp.BatchItemFailures,
				events.SQSBatchItemFailure{ItemIdentifier: rec.MessageId})
		}
	}
	return resp, nil
}
```

Enable `ReportBatchItemFailures` on the event source mapping, configure a DLQ (redrive policy) and alarm on its depth.

## Anti-patterns

- Retrying non-idempotent operations (payments, emails) without an idempotency key.
- Retrying at every layer (client, service, queue) and multiplying load.
- Infinite retries or retries that ignore context cancellation.
- Swallowing errors to "keep going" without a DLQ or metric.

## Sources

- Amazon Builders' Library, "Timeouts, retries, and backoff with jitter": https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/
- Amazon Builders' Library, "Making retries safe with idempotent APIs": https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/
- AWS Prescriptive Guidance, cloud design patterns (outbox, saga, circuit breaker, retry with backoff): https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/introduction.html
- AWS SDK for Go v2, retries and timeouts: https://docs.aws.amazon.com/sdk-for-go/v2/developer-guide/configure-gosdk.html
- Lambda + SQS partial batch responses: https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-errorhandling.html
