---
name: shp-patterns-creational
description: Creational design patterns adapted to idiomatic Go - constructor functions, functional options, builder, factory and abstract factory via interfaces, singleton via sync.Once, prototype, and object pools. Use when deciding how to construct values or services, when a constructor has many optional parameters, when choosing between zero values and NewX, or when the user asks which creational pattern applies.
license: MIT
metadata:
  kit: shuppatsu-go
  version: "1.0.0"
---

# shp-patterns-creational

Go has no classes, constructors or inheritance. Creational patterns become plain functions, interfaces and closures. Prefer the simplest form that keeps invariants.

## Decision guide

| Situation | Use |
|---|---|
| The zero value is already valid (`sync.Mutex`, `bytes.Buffer`) | No constructor at all |
| Required fields or invariants must be set | Constructor `NewX(required...) (*X, error)` |
| Many optional settings with defaults | Functional options |
| Step-by-step assembly with validation at the end | Builder (rare in Go; options usually win) |
| Caller picks an implementation at runtime | Factory function returning an interface |
| Families of related implementations (e.g. AWS vs in-memory) | Abstract factory: a struct of constructors or an interface |
| Exactly one lazily-initialised instance | `sync.Once` / `sync.OnceValue` (avoid global mutable state when possible) |
| Expensive, short-lived buffers | `sync.Pool` |

## Constructor function

```go
// New validates invariants once, so methods can assume them.
func New(table string, db DynamoAPI) (*Store, error) {
	if table == "" {
		return nil, errors.New("store: table name is required")
	}
	return &Store{table: table, db: db}, nil
}
```

Why: Effective Go ("Constructors and composite literals") and Code Review Comments prefer explicit constructors only when needed. Return the concrete type so callers keep access to all methods.

## Functional options

```go
type Server struct {
	addr    string
	timeout time.Duration
	logger  *slog.Logger
}

type Option func(*Server)

func WithTimeout(d time.Duration) Option   { return func(s *Server) { s.timeout = d } }
func WithLogger(l *slog.Logger) Option     { return func(s *Server) { s.logger = l } }

func NewServer(addr string, opts ...Option) *Server {
	s := &Server{addr: addr, timeout: 10 * time.Second, logger: slog.Default()}
	for _, opt := range opts {
		opt(s)
	}
	return s
}
```

- Use when there are several optional knobs and you want backwards-compatible growth.
- Required parameters stay positional; only optional ones become options.
- If options can fail, use `type Option func(*Server) error`.
- Real-world example: AWS SDK v2 `s3.NewFromConfig(cfg, func(o *s3.Options) { o.UsePathStyle = true })`.
- Alternative: a `Config` struct literal. Simpler and fine for internal code; options shine in public APIs.

## Builder

```go
type QueryBuilder struct {
	table string
	conds []string
	limit int
	err   error
}

func (b *QueryBuilder) Where(c string) *QueryBuilder { b.conds = append(b.conds, c); return b }
func (b *QueryBuilder) Limit(n int) *QueryBuilder {
	if n <= 0 {
		b.err = fmt.Errorf("limit must be positive, got %d", n)
	}
	b.limit = n
	return b
}
func (b *QueryBuilder) Build() (Query, error) {
	if b.err != nil {
		return Query{}, b.err
	}
	if b.table == "" {
		return Query{}, errors.New("table is required")
	}
	return Query{table: b.table, conds: slices.Clone(b.conds), limit: b.limit}, nil
}
```

Use only when construction is genuinely multi-step. Record the first error and return it from `Build`, since chained methods cannot return errors. `strings.Builder` is a builder for strings.

## Factory and abstract factory

```go
type Notifier interface{ Notify(ctx context.Context, msg string) error }

func NewNotifier(kind string, cfg aws.Config) (Notifier, error) {
	switch kind {
	case "sns":
		return &snsNotifier{client: sns.NewFromConfig(cfg)}, nil
	case "log":
		return &logNotifier{}, nil
	default:
		return nil, fmt.Errorf("unknown notifier %q", kind)
	}
}
```

An abstract factory in Go is usually a small struct that groups constructors for one environment (for example `AWSAdapters` vs `InMemoryAdapters`) wired in `main`. Do not build factory hierarchies.

## Singleton (use sparingly)

```go
var loadConfig = sync.OnceValues(func() (aws.Config, error) {
	return config.LoadDefaultConfig(context.Background())
})
```

Prefer creating dependencies in `main` and passing them down (dependency injection by constructor). Package-level singletons hide dependencies and make tests harder. In Lambda, initialise at package level or in `main` before `lambda.Start` so warm invocations reuse it.

## Prototype

Go copies structs by value. Provide a `Clone()` method only when the struct holds slices, maps or pointers that must be deep-copied (`slices.Clone`, `maps.Clone`).

## Object pool

`sync.Pool` for temporary objects that are expensive to allocate (buffers in hot paths). Do not pool connections with `sync.Pool`; clients such as `http.Client` and AWS SDK clients already manage their own connection pools.

## Anti-patterns

- Java-style `NewXFactoryFactory`, getters/setters for every field, or interfaces declared before a second implementation exists.
- Constructors that start goroutines or perform I/O without a context.
- Global mutable singletons used as service locators.

## Sources

- Effective Go, "Constructors and composite literals": https://go.dev/doc/effective_go#composite_literals
- Dave Cheney, "Functional options for friendly APIs": https://dave.cheney.net/2014/10/17/functional-options-for-friendly-apis
- Refactoring.Guru creational patterns in Go: https://refactoring.guru/design-patterns/creational-patterns
- `sync.OnceValues`, `sync.Pool`: `go doc sync`
