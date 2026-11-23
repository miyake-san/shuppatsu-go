---
name: shp-patterns-resilience
description: Cloud-agnostic resilience and distributed-systems patterns for Go services - timeouts, retries with exponential backoff and jitter, idempotency keys, circuit breaker, bulkhead, transactional outbox, saga with compensations, dead-letter queues and partial batch failures. Use when calling remote services, processing events or queue messages, designing failure handling in design.md, or when the user asks how to make code safe to retry.
license: MIT
metadata:
  kit: shuppatsu-go
  version: "1.3.0"
---

# shp-patterns-resilience

Assume every remote call can be slow, fail, or succeed without you knowing. Resilience is layered: **timeout → retry (only if safe) → idempotency → isolation → compensation**.

## Decision guide

| Failure mode | Pattern |
|---|---|
| Call hangs | Timeout via context deadline and client timeouts |
| Transient error or throttling | Retry with exponential backoff and jitter |
| Retry may duplicate side effects | Idempotency key + conditional write |
| Dependency is down; retries pile up | Circuit breaker, fail fast |
| One slow dependency starves others | Bulkhead: separate pools/limits per dependency |
| Database write and event publish must both happen | Transactional outbox |
| Multi-step business transaction across services | Saga (orchestrated or choreographed) with compensations |
| Poison messages | Dead-letter queue + alarm |
| One bad record in a batch | Partial batch response (report only the failed items) |

## Timeouts

```go
httpClient := &http.Client{Timeout: 5 * time.Second}

ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
defer cancel()
out, err := store.Get(ctx, key)
```

Set the timeout below the caller's own deadline (an upstream gateway, a serverless function limit, a request deadline carried in the context) so you can still return a meaningful error instead of being killed mid-flight.

## Retries with backoff and jitter

Retry only idempotent operations, cap the attempts, use full jitter, and stop when the context is done. If a client library already retries (most cloud SDKs do), configure it instead of stacking a second loop on top.

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

Retry classification: network timeouts, throttling (`429`), and `5xx` are usually retryable; validation and authorisation errors are not. Honour a `Retry-After` header when present.

## Idempotency

- Accept an idempotency key from the client (header or message attribute) or derive it from a stable event ID.
- Record it with an **atomic conditional write** so a duplicate is detected without a race:

```go
// IdempotencyStore is any key-value store that supports "insert if absent".
type IdempotencyStore interface {
	// PutIfAbsent returns ErrExists when the key is already present.
	PutIfAbsent(ctx context.Context, key string, result []byte, ttl time.Duration) error
	Get(ctx context.Context, key string) ([]byte, error)
}

err := store.PutIfAbsent(ctx, "idem#"+key, nil, 24*time.Hour)
if errors.Is(err, ErrExists) {
	return store.Get(ctx, "idem#"+key) // duplicate: return the stored outcome
}
```

- Store a TTL so idempotency records expire.
- Message brokers and event sources usually deliver **at least once**: consumers must be idempotent.
- With SQL, a unique constraint plus `INSERT ... ON CONFLICT DO NOTHING` gives the same guarantee.

## Circuit breaker and bulkhead

- A breaker counts recent failures per dependency and rejects calls for a cool-down period once a threshold is passed. Use a small, well-tested library (for example `github.com/sony/gobreaker`) rather than a hand-rolled one.
- Bulkhead: give each dependency its own concurrency limit (`errgroup.SetLimit`, semaphores, separate HTTP clients), so one slow dependency cannot exhaust all goroutines or connections.

## Transactional outbox

1. In the same transaction as the state change, write an outbox record (the same SQL transaction, or the store's multi-item transaction).
2. A relay (a poller or a change-data-capture stream) publishes outbox records to the message broker and marks them sent.
3. Consumers are idempotent because the relay can publish twice.

## Saga

Model long-running, multi-service workflows as a sequence of local transactions, each with a **compensating action** (refund, release stock) that runs when a later step fails. Orchestrate with a workflow engine or a coordinator service, or choreograph through events. Keep compensations idempotent too, and record saga state durably so it can resume after a crash.

## Queues, DLQs and partial batch failures

```go
// Process a batch; report only the records that failed so successes are not redelivered.
func handleBatch(ctx context.Context, records []Record) []string {
	var failed []string
	for _, rec := range records {
		if err := process(ctx, rec); err != nil {
			failed = append(failed, rec.ID)
		}
	}
	return failed
}
```

- Configure a dead-letter queue (with a maximum receive count) for messages that keep failing, and alarm on its depth.
- Prefer partial-batch responses over failing the whole batch; failing it redelivers messages that already succeeded.
- Keep a way to inspect and redrive dead-lettered messages once the bug is fixed.

## Anti-patterns

- Retrying non-idempotent operations (payments, emails) without an idempotency key.
- Retrying at every layer (client, service, queue) and multiplying load.
- Infinite retries or retries that ignore context cancellation.
- Swallowing errors to "keep going" without a DLQ or metric.

## Verification gate

- Every outbound call has a timeout that is shorter than its caller's deadline (show the value).
- Retries are bounded, jittered, context-aware and only applied to idempotent operations; a test covers the give-up path.
- A duplicate delivery test shows that the handler produces one side effect.
- Poison messages end up in a DLQ with an alarm, not in an infinite loop.

## Sources

- Release It!, stability patterns (timeouts, circuit breaker, bulkhead): Michael T. Nygard, https://pragprog.com/titles/mnee2/release-it-second-edition/
- Microservices patterns (transactional outbox, saga, idempotent consumer): https://microservices.io/patterns/index.html
- Martin Fowler, "Circuit Breaker": https://martinfowler.com/bliki/CircuitBreaker.html
- Google SRE book, "Handling Overload" and "Addressing Cascading Failures": https://sre.google/sre-book/handling-overload/ , https://sre.google/sre-book/addressing-cascading-failures/
- Go `context`, `errgroup`, `math/rand/v2`: `go doc context`, `go doc golang.org/x/sync/errgroup`, `go doc math/rand/v2`
- `net/http` client timeouts: https://go.dev/blog/context and `go doc net/http.Client`

Provider-specific details live in the cloud-provider skill packs (for example the `shp-aws-*` skills).
