---
name: shp-patterns-architecture
description: Application architecture patterns for Go services - hexagonal (ports and adapters), clean architecture layering mapped to Go packages, repository, service layer, dependency injection by constructor, unit of work, CQRS-lite and domain events. Use when designing a new service or feature, writing design.md in a Kiro spec, deciding package boundaries, or when the user asks where code should live and why.
license: MIT
metadata:
  kit: shuppatsu-go
  version: "1.0.0"
---

# shp-patterns-architecture

Go favours simple, flat structures. Apply architecture patterns to keep **business rules independent of delivery (HTTP, Lambda) and infrastructure (AWS, databases)**, not to create layers for their own sake.

## Ports and adapters (hexagonal) mapped to Go

```text
cmd/api/main.go              -> composition root: build config, clients, adapters; inject them
internal/order/              -> domain + application service (the hexagon)
    order.go                 -> entities, value objects, invariants
    service.go               -> use cases (PlaceOrder, CancelOrder)
    ports.go                 -> interfaces the service needs (Repository, Publisher, Clock)
internal/platform/dynamo/    -> adapter implementing order.Repository
internal/platform/sns/       -> adapter implementing order.Publisher
internal/transport/http/     -> driving adapter: HTTP handlers calling order.Service
internal/transport/lambda/   -> driving adapter: Lambda handlers calling order.Service
```

- **Ports** are interfaces declared in the domain package that needs them (consumer-side interfaces).
- **Adapters** live under `internal/platform` or `internal/transport` and import the domain, never the reverse.
- `go list -deps ./internal/order` should show no AWS SDK or `net/http` packages: that is a quick architecture test.

## Service layer

```go
type Service struct {
	repo  Repository
	pub   Publisher
	clock func() time.Time
}

func NewService(repo Repository, pub Publisher, clock func() time.Time) *Service {
	return &Service{repo: repo, pub: pub, clock: clock}
}

func (s *Service) Place(ctx context.Context, cmd PlaceOrder) (Order, error) {
	o, err := NewOrder(cmd.CustomerID, cmd.Items, s.clock())
	if err != nil {
		return Order{}, err // validation error from the domain
	}
	if err := s.repo.Save(ctx, o); err != nil {
		return Order{}, fmt.Errorf("save order: %w", err)
	}
	if err := s.pub.Publish(ctx, OrderPlaced{ID: o.ID}); err != nil {
		return Order{}, fmt.Errorf("publish order placed: %w", err)
	}
	return o, nil
}
```

Note: saving then publishing is not atomic. If both must succeed together, use the transactional outbox (see `shp-patterns-resilience`).

## Repository

- One repository per aggregate, expressed in domain language (`Save`, `Get`, `ListByCustomer`), not table operations.
- Return domain types and domain errors (`ErrNotFound`); translate SDK errors inside the adapter.
- DynamoDB single-table design is an adapter detail; the domain should not know about partition keys.

## Dependency injection

- Constructor injection, wired by hand in `main`. It is explicit, type-checked and needs no framework.
- Consider a DI library (Wire, Fx) only when the wiring code itself becomes hard to maintain.
- Inject time (`func() time.Time`), randomness and ID generation to keep tests deterministic.

## Unit of work

When several writes must succeed together:

- Relational DB: pass a transaction-scoped repository into a function: `store.WithTx(ctx, func(r Repository) error { ... })`.
- DynamoDB: `TransactWriteItems` (up to 100 items), built inside the adapter.

## CQRS-lite

Separate read models when query needs differ strongly from write models (for example, a DynamoDB table for writes and an OpenSearch or GSI-based read model). Do not split by default; start with one model.

## Domain events

Represent meaningful state changes as types (`OrderPlaced`, `OrderCanceled`). Publish them via an outbox to EventBridge, SNS or SQS for other services.

## Mapping to Kiro specs

- `requirements.md`: user stories and acceptance criteria in domain language.
- `design.md`: name the ports, adapters, data model, events and failure modes; reference these patterns by name.
- `tasks.md`: one task per port/adapter/use case so each can be implemented and tested in isolation.

## Anti-patterns

- `models`, `controllers`, `services`, `utils` packages grouping by technical kind instead of by domain.
- Domain structs with `dynamodbav` or `json` tags leaking storage and transport concerns; map at the adapter when they diverge.
- Generic repositories (`Repository[T]`) that erase domain meaning.
- Interfaces for everything up front; add a port when there is a second implementation or a test fake.

## Sources

- Organizing a Go module: https://go.dev/doc/modules/layout
- Alistair Cockburn, "Hexagonal architecture": https://alistair.cockburn.us/hexagonal-architecture/
- AWS Prescriptive Guidance, hexagonal architecture pattern: https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/hexagonal-architecture.html
- DynamoDB transactions: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/transaction-apis.html
