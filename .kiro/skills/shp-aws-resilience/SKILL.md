---
name: shp-aws-resilience
description: AWS-specific resilience for Go - AWS SDK for Go v2 retry modes and timeouts, DynamoDB conditional-write idempotency, at-least-once Lambda event sources, SQS partial batch responses and DLQ redrive, transactional outbox with DynamoDB Streams, Step Functions sagas, API Gateway and event contracts. Use when making AWS calls safe to retry, handling SQS/SNS/EventBridge events in Lambda, designing failure handling on AWS, or when the user asks about DLQs, redrive or partial batch failures.
license: MIT
metadata:
  kit: shuppatsu-go
  version: "1.3.0"
---

# shp-aws-resilience

AWS implementation of the generic patterns in `shp-patterns-resilience` (timeouts, retries, idempotency, outbox, saga, DLQ). Read that skill first for the pattern; this one maps it to AWS services and the AWS SDK for Go v2. For SDK setup see `shp-aws-sdk`; for Lambda packaging and handlers see `shp-aws-lambda`.

## Pattern to AWS mapping

| Pattern | AWS building block |
|---|---|
| Timeout | `context.WithTimeout`, client timeouts; stay below the API Gateway and Lambda limits |
| Retry with jitter | AWS SDK v2 retryer (standard or adaptive mode) |
| Idempotency key | DynamoDB conditional write with TTL |
| Poison messages | SQS dead-letter queue with redrive policy + CloudWatch alarm |
| One bad record in a batch | SQS + Lambda partial batch response (`ReportBatchItemFailures`) |
| Transactional outbox | DynamoDB `TransactWriteItems` + DynamoDB Streams + Lambda relay to SNS/EventBridge/SQS |
| Saga | AWS Step Functions with `Retry` and `Catch` and compensating states |

## Timeouts

```go
ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
defer cancel()
out, err := client.GetItem(ctx, in)
```

Set the timeout below the caller's own deadline (API Gateway integration timeout, Lambda function timeout; read the remaining time from `ctx.Deadline()` inside a handler) so you can still return a meaningful error.

## Retries with the AWS SDK for Go v2

The SDK already retries throttling and transient errors with exponential backoff and jitter (standard mode). Tune it instead of writing your own loop:

```go
cfg, err := config.LoadDefaultConfig(ctx,
	config.WithRetryMaxAttempts(5),
	config.WithRetryMode(aws.RetryModeAdaptive), // adds client-side rate limiting under throttling
)
```

For non-AWS calls in the same service, use the generic `Retry` helper from `shp-patterns-resilience`. Do not wrap an SDK call in your own retry loop on top of the SDK's retryer: attempts multiply.

## Idempotency with DynamoDB

Record the key with a conditional write so a duplicate is detected atomically:

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

- Enable DynamoDB TTL on the attribute so idempotency records expire.
- Lambda event sources (SQS, EventBridge, SNS) deliver **at least once**: handlers must be idempotent. Derive the key from the SQS `MessageId`, the EventBridge event `id`, or a business key.

## Transactional outbox on DynamoDB

1. In the same transaction as the state change, write an outbox record with DynamoDB `TransactWriteItems` (up to 100 items), or in the same SQL transaction if you use a relational database.
2. A relay publishes outbox records to SNS, EventBridge or SQS. With DynamoDB, DynamoDB Streams + Lambda is the relay.
3. Consumers are idempotent because the relay can publish twice.

## Saga with Step Functions

Model long-running, multi-service workflows in AWS Step Functions: each step is a Lambda or service integration with `Retry` and `Catch`, and failures trigger compensating steps (refund, release stock). Keep compensations idempotent too.

## SQS, DLQs and partial batch failures

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

- Enable `ReportBatchItemFailures` on the event source mapping.
- Configure a DLQ (redrive policy with `maxReceiveCount`) and alarm on `ApproximateNumberOfMessagesVisible` of the DLQ.
- Set the queue visibility timeout to at least six times the function timeout.
- Use the DLQ redrive feature to move messages back once the bug is fixed.

## API Gateway and event contracts

The API Gateway status mapping, error envelope and event-source contract rules live in the "API Gateway and event sources" section of `shp-aws-lambda`. For delivery guarantees, dedupe event IDs with the DynamoDB idempotency pattern above.

## Anti-patterns

- Wrapping SDK calls in a second retry loop on top of the SDK's retryer.
- A Lambda timeout longer than the API Gateway timeout that fronts it (the client gives up while work continues).
- Failing the whole SQS batch because one record is bad.
- A DLQ with no alarm, or no process to inspect it.
- Idempotency records without TTL (unbounded table growth).

## Verification gate

- SDK retry mode and max attempts are set explicitly and documented; no duplicate retry loops.
- A test shows that a duplicate event produces one side effect (conditional write path covered, including `ConditionalCheckFailedException`).
- SQS partial batch response tested with a mix of good and bad records; `ReportBatchItemFailures` is enabled in IaC.
- DLQ exists, has a redrive policy and an alarm.
- AWS claims were checked against the documentation (`shp-go-verify` for the Go side, AWS docs for quotas and limits).

## Sources

- Amazon Builders' Library, "Timeouts, retries, and backoff with jitter": https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/
- Amazon Builders' Library, "Making retries safe with idempotent APIs": https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/
- AWS Prescriptive Guidance, cloud design patterns (outbox, saga, circuit breaker, retry with backoff): https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/introduction.html
- AWS SDK for Go v2, retries and timeouts: https://docs.aws.amazon.com/sdk-for-go/v2/developer-guide/configure-gosdk.html
- Lambda + SQS partial batch responses: https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-errorhandling.html
- SQS dead-letter queues and redrive: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html
- DynamoDB transactions: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/transaction-apis.html
- DynamoDB Streams: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Streams.html
- AWS Step Functions: https://docs.aws.amazon.com/step-functions/latest/dg/welcome.html
- Generic patterns: `shp-patterns-resilience`
