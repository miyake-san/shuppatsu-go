---
name: shp-patterns-behavioral
description: Behavioral design patterns adapted to idiomatic Go - strategy with function types or interfaces, observer and pub/sub, command, chain of responsibility, state machines, template method via composition, iterator with iter.Seq, visitor via type switches, and mediator. Use when choosing how objects collaborate, how to swap algorithms at runtime, how to model workflows and state transitions, or when the user asks which behavioral pattern applies.
license: MIT
metadata:
  kit: shuppatsu-go
  version: "1.0.0"
---

# shp-patterns-behavioral

In Go, many behavioral patterns collapse into **first-class functions**, **small interfaces** and **channels**. Choose the lightest representation that keeps the code readable.

## Decision guide

| Need | Pattern | Go shape |
|---|---|---|
| Swap an algorithm at runtime | Strategy | Function type, or one-method interface |
| Notify many listeners of events | Observer / pub-sub | Callbacks, or channels; across services: SNS/EventBridge |
| Encapsulate a request as a value (queue, undo, audit) | Command | Struct with `Execute(ctx) error`, or a closure |
| Pass a request through ordered handlers | Chain of responsibility | Middleware chain |
| Behaviour depends on current state | State | State as type + transition table, or state functions |
| Fixed algorithm with pluggable steps | Template method | Function taking step functions or an interface |
| Traverse a collection | Iterator | `iter.Seq[T]` / `iter.Seq2[K,V]` (Go 1.23+) |
| Operate on a closed set of types | Visitor | Type switch |
| Coordinate many peers | Mediator | Central coordinator, or a channel-owning goroutine |

## Strategy

```go
type PricingFunc func(base Money, c Customer) Money

func Standard(base Money, _ Customer) Money { return base }
func Loyalty(base Money, c Customer) Money  { return base.Discount(c.LoyaltyPercent()) }

type Checkout struct{ price PricingFunc }
```

Use a function type when the strategy is stateless and has one operation; use an interface when it has state or several methods. Standard-library example: `sort.Slice(x, less)` and `slices.SortFunc(x, cmp)`.

## Observer

```go
type Listener func(ctx context.Context, e OrderPlaced)

type Bus struct {
	mu        sync.RWMutex
	listeners []Listener
}

func (b *Bus) Subscribe(l Listener) { b.mu.Lock(); b.listeners = append(b.listeners, l); b.mu.Unlock() }

func (b *Bus) Publish(ctx context.Context, e OrderPlaced) {
	b.mu.RLock()
	ls := slices.Clone(b.listeners)
	b.mu.RUnlock()
	for _, l := range ls {
		l(ctx, e)
	}
}
```

In-process only. Across services, publish to Amazon SNS, EventBridge or SQS instead (see `shp-patterns-resilience` for delivery guarantees).

## Command

```go
type Command interface{ Execute(ctx context.Context) error }

type CancelOrder struct{ ID string; Reason string; repo OrderRepository }

func (c CancelOrder) Execute(ctx context.Context) error { /* ... */ return nil }
```

Commands are a natural fit for SQS messages: serialise the command struct, consume it in a Lambda, execute idempotently.

## Chain of responsibility

HTTP middleware is the canonical Go form: `handler := Auth(RateLimit(Logging(mux)))`. Each link decides to handle, short-circuit, or call `next`. Keep the chain order explicit in one place (usually `main`).

## State machine

```go
type State string

const (
	Pending  State = "PENDING"
	Paid     State = "PAID"
	Shipped  State = "SHIPPED"
	Canceled State = "CANCELED"
)

var transitions = map[State][]State{
	Pending: {Paid, Canceled},
	Paid:    {Shipped, Canceled},
}

func (o *Order) Transition(to State) error {
	if !slices.Contains(transitions[o.State], to) {
		return fmt.Errorf("invalid transition %s -> %s", o.State, to)
	}
	o.State = to
	return nil
}
```

For long-running, multi-step workflows across services, model the state machine in AWS Step Functions instead of in code.

## Template method without inheritance

```go
func Import(ctx context.Context, read func(context.Context) ([]Row, error), validate func(Row) error, save func(context.Context, []Row) error) error {
	rows, err := read(ctx)
	if err != nil {
		return fmt.Errorf("read: %w", err)
	}
	for _, r := range rows {
		if err := validate(r); err != nil {
			return fmt.Errorf("validate row %v: %w", r.ID, err)
		}
	}
	return save(ctx, rows)
}
```

## Iterator (Go 1.23+)

```go
func (s *Store) All(ctx context.Context) iter.Seq2[Order, error] {
	return func(yield func(Order, error) bool) {
		p := dynamodb.NewScanPaginator(s.client, &dynamodb.ScanInput{TableName: aws.String(s.table)})
		for p.HasMorePages() {
			page, err := p.NextPage(ctx)
			if err != nil {
				yield(Order{}, err)
				return
			}
			for _, item := range page.Items {
				var o Order
				if err := attributevalue.UnmarshalMap(item, &o); err != nil {
					if !yield(Order{}, err) {
						return
					}
					continue
				}
				if !yield(o, nil) {
					return
				}
			}
		}
	}
}
```

Respect the `yield` return value: stop as soon as it returns `false`. Check the `go` directive before using iterators.

## Visitor

Go has no double dispatch. For a closed set of types, a type switch is clearer:

```go
switch e := event.(type) {
case OrderPlaced:
	return handlePlaced(ctx, e)
case OrderCanceled:
	return handleCanceled(ctx, e)
default:
	return fmt.Errorf("unhandled event %T", e)
}
```

Standard-library example of a real visitor: `ast.Walk(v ast.Visitor, node)`.

## Anti-patterns

- Interfaces with one implementation and no test fake: use the concrete type until a second need appears.
- Observer callbacks that block the publisher or run without a context.
- State spread across booleans (`isPaid`, `isShipped`) instead of an explicit state value.

## Sources

- Go blog, "Range Over Function Types": https://go.dev/blog/range-functions
- `iter` package: `go doc iter`
- Refactoring.Guru behavioral patterns in Go: https://refactoring.guru/design-patterns/behavioral-patterns
- AWS Step Functions: https://docs.aws.amazon.com/step-functions/latest/dg/welcome.html
