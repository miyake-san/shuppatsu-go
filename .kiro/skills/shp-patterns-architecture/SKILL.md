---
name: shp-patterns-architecture
description: Application architecture patterns for Go services - hexagonal (ports and adapters), clean architecture layering mapped to Go packages, repository, service layer, dependency injection by constructor, unit of work, CQRS-lite and domain events. Use when designing a new service or feature, writing design.md in a Kiro spec, deciding package boundaries, or when the user asks where code should live and why.
license: MIT
metadata:
  kit: shuppatsu-go
  version: "1.3.0"
---

# shp-patterns-architecture

Go favours simple, flat structures. Apply architecture patterns to keep **business rules independent of delivery (HTTP, serverless functions, CLIs) and infrastructure (cloud services, databases)**, not to create layers for their own sake.

## Ports and adapters (hexagonal) mapped to Go

```text
cmd/api/main.go              -> composition root: build config, clients, adapters; inject them
internal/order/              -> domain + application service (the hexagon)
    order.go                 -> entities, value objects, invariants
    service.go               -> use cases (PlaceOrder, CancelOrder)
    ports.go                 -> interfaces the service needs (Repository, Publisher, Clock)
internal/platform/store/     -> adapter implementing order.Repository (SQL or key-value store)
internal/platform/queue/     -> adapter implementing order.Publisher (message queue or topic)
internal/transport/http/     -> driving adapter: HTTP handlers calling order.Service
internal/transport/func/     -> driving adapter: serverless function handlers calling order.Service
```

- **Ports** are interfaces declared in the domain package that needs them (consumer-side interfaces).
- **Adapters** live under `internal/platform` or `internal/transport` and import the domain, never the reverse.
- `go list -deps ./internal/order` should show no cloud SDK, database driver or `net/http` packages: that is a quick architecture test.

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
- Return domain types and domain errors (`ErrNotFound`); translate SDK and driver errors inside the adapter.
- Storage layout (single-table design, partition keys, indexes, sharding) is an adapter detail; the domain should not know about it.

## Dependency injection

- Constructor injection, wired by hand in `main`. It is explicit, type-checked and needs no framework.
- Consider a DI library (Wire, Fx) only when the wiring code itself becomes hard to maintain.
- Inject time (`func() time.Time`), randomness and ID generation to keep tests deterministic.

## Unit of work

When several writes must succeed together:

- Relational DB: pass a transaction-scoped repository into a function: `store.WithTx(ctx, func(r Repository) error { ... })`.
- Key-value or document stores: the store's own transaction primitive (check its item and size limits), built inside the adapter.

## CQRS-lite

Separate read models when query needs differ strongly from write models (for example, a transactional table for writes and a search index or secondary-index-based read model). Do not split by default; start with one model.

## Domain events

Represent meaningful state changes as types (`OrderPlaced`, `OrderCanceled`). Publish them via an outbox to a message queue or topic for other services.

## Deep modules and a shared glossary

- Prefer **deep modules**: a small exported surface hiding substantial behaviour (`os.Open`, `http.Client`). A package whose exports mirror its internals adds cost without hiding anything.
- Keep a short domain glossary (in `product.md` or `docs/glossary.md`) and name types, packages and events with those exact words, so code, specs and conversations agree.

## Reference implementation

ThreeDotsLabs **Wild Workouts** is a complete Go example of domain, application (commands and queries), ports and adapters, repositories and CQRS, with the reasoning in the "DDD, CQRS and Clean Architecture combined" article. Study it for structure; do not copy its full layering into a small service (decide with `shp-patterns-select`, record the decision with `shp-adr`).

## Mapping to Kiro specs

- `requirements.md`: user stories and acceptance criteria in domain language.
- `design.md`: name the ports, adapters, data model, events and failure modes; reference these patterns by name.
- `tasks.md`: one task per port/adapter/use case so each can be implemented and tested in isolation.

## Anti-patterns

- `models`, `controllers`, `services`, `utils` packages grouping by technical kind instead of by domain.
- Domain structs with storage (`db`, `bson`, attribute-mapping) or `json` tags leaking storage and transport concerns; map at the adapter when they diverge.
- Generic repositories (`Repository[T]`) that erase domain meaning.
- Interfaces for everything up front; add a port when there is a second implementation or a test fake.

## Sources

- Organizing a Go module: https://go.dev/doc/modules/layout
- Alistair Cockburn, "Hexagonal architecture": https://alistair.cockburn.us/hexagonal-architecture/
- Martin Fowler, "Repository" and "Unit of Work": https://martinfowler.com/eaaCatalog/repository.html , https://martinfowler.com/eaaCatalog/unitOfWork.html
- Chris Richardson, microservices patterns (transactional outbox, CQRS): https://microservices.io/patterns/
- ThreeDotsLabs, Wild Workouts (Go DDD/CQRS example): https://github.com/ThreeDotsLabs/wild-workouts-go-ddd-example
- ThreeDotsLabs, "DDD, CQRS and Clean Architecture combined": https://threedots.tech/post/ddd-cqrs-clean-architecture-combined/
- Deep modules and glossary ideas: *A Philosophy of Software Design* (J. Ousterhout); mattpocock/skills `codebase-design`, `domain-modeling` (MIT, read as reference only): https://github.com/mattpocock/skills

Provider-specific details live in the cloud-provider skill packs (for example the `shp-aws-*` skills).
