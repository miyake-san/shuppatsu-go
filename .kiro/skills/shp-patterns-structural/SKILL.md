---
name: shp-patterns-structural
description: Structural design patterns adapted to idiomatic Go - adapter, decorator and HTTP middleware, facade, proxy, composite, bridge, and struct embedding as composition instead of inheritance. Use when wrapping a third-party or AWS client behind your own interface, adding cross-cutting behaviour (logging, metrics, retries, auth), simplifying a complex subsystem, or when the user asks which structural pattern applies.
license: MIT
metadata:
  kit: shuppatsu-go
  version: "1.0.0"
---

# shp-patterns-structural

Go composes behaviour with small interfaces, embedding and function types. Most structural patterns reduce to "a type that satisfies an interface by delegating to another value".

## Decision guide

| Need | Pattern | Go shape |
|---|---|---|
| Use a type whose API does not match what your code expects | Adapter | Struct wrapping the foreign type, implementing your interface |
| Add behaviour without changing the wrapped type | Decorator | Same interface in and out (`func(http.Handler) http.Handler`) |
| Hide a complex subsystem behind a simple API | Facade | Struct with a few task-level methods |
| Control access (lazy init, cache, auth, rate limit) | Proxy | Same interface, guards the call |
| Treat a tree of objects uniformly | Composite | Interface implemented by both leaf and container |
| Vary abstraction and implementation independently | Bridge | Struct holding an interface field |
| Reuse fields and methods | Embedding | Composition, not inheritance |

## Adapter: keep AWS types out of the domain

```go
// Declared by the consumer (domain), not by the AWS package.
type OrderRepository interface {
	Get(ctx context.Context, id string) (Order, error)
}

// Adapter in internal/platform/dynamo.
type DynamoOrders struct {
	client *dynamodb.Client
	table  string
}

func (r *DynamoOrders) Get(ctx context.Context, id string) (Order, error) {
	out, err := r.client.GetItem(ctx, &dynamodb.GetItemInput{
		TableName: aws.String(r.table),
		Key:       map[string]types.AttributeValue{"pk": &types.AttributeValueMemberS{Value: id}},
	})
	if err != nil {
		return Order{}, fmt.Errorf("get order %s: %w", id, err)
	}
	if out.Item == nil {
		return Order{}, ErrNotFound
	}
	var o Order
	if err := attributevalue.UnmarshalMap(out.Item, &o); err != nil {
		return Order{}, fmt.Errorf("decode order %s: %w", id, err)
	}
	return o, nil
}
```

Why: "Accept interfaces, return structs" and "define interfaces where they are used" (Code Review Comments, "Interfaces"). The domain stays testable with a fake and independent of the SDK.

## Decorator and middleware

```go
func Logging(log *slog.Logger) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			start := time.Now()
			next.ServeHTTP(w, r)
			log.InfoContext(r.Context(), "request", "method", r.Method, "path", r.URL.Path, "dur", time.Since(start))
		})
	}
}
```

Decorators for interfaces work the same way:

```go
type timedRepo struct{ next OrderRepository; hist Histogram }

func (t timedRepo) Get(ctx context.Context, id string) (Order, error) {
	defer t.hist.Since(time.Now())
	return t.next.Get(ctx, id)
}
```

Standard-library examples: `bufio.NewReader(io.Reader)`, `io.LimitReader`, `http.TimeoutHandler`, `gzip.NewWriter`.

## Facade

A package-level API such as `billing.Charge(ctx, customer, amount)` that orchestrates several clients (DynamoDB, SQS, payment provider). Keep the facade thin; business rules live in the domain.

## Proxy

Same interface as the real subject, adding caching, lazy creation, authorisation or rate limiting. Example: a caching `OrderRepository` that checks an in-memory LRU before DynamoDB. Make cache invalidation explicit and bounded.

## Composite

```go
type Validator interface{ Validate(Order) error }

type All []Validator

func (a All) Validate(o Order) error {
	var errs []error
	for _, v := range a {
		errs = append(errs, v.Validate(o))
	}
	return errors.Join(errs...) // errors.Join drops nil errors
}
```

## Embedding is not inheritance

- Embedding promotes methods, but the embedded value does not know about the outer type: there is no virtual dispatch back to the outer struct.
- Embed to reuse behaviour or satisfy an interface; do not embed to build "is-a" hierarchies.
- Embedding an interface in a struct (for partial fakes in tests) panics on unimplemented methods. Use it only in tests.

## Anti-patterns

- Wrapping every external type "just in case": adapt only at architecture boundaries.
- Huge interfaces mirroring a whole AWS client. Declare only the methods you call.
- Decorators that change semantics (swallowing errors) instead of adding behaviour.

## Sources

- Go Code Review Comments, "Interfaces": https://go.dev/wiki/CodeReviewComments#interfaces
- Effective Go, "Embedding": https://go.dev/doc/effective_go#embedding
- Refactoring.Guru structural patterns in Go: https://refactoring.guru/design-patterns/structural-patterns
- `errors.Join`: `go doc errors.Join`; DynamoDB `attributevalue`: `go doc github.com/aws/aws-sdk-go-v2/feature/dynamodb/attributevalue`
