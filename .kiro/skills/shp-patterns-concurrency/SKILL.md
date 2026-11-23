---
name: shp-patterns-concurrency
description: Go concurrency patterns - goroutine lifecycle and cancellation with context, errgroup, worker pools, pipelines, fan-out/fan-in, bounded parallelism with semaphores, rate limiting, timeouts, and safe shutdown. Use when writing or explaining code with goroutines, channels, sync primitives, parallel remote calls, or batch processing, or when debugging leaks, deadlocks and data races.
license: MIT
metadata:
  kit: shuppatsu-go
  version: "1.3.0"
---

# shp-patterns-concurrency

Rules that apply to every pattern below:

1. Every goroutine has an owner and a defined way to stop (context cancellation, channel close, or completion).
2. The sender closes a channel, never the receiver. Closing is a broadcast, not a requirement.
3. Bound parallelism: unbounded `go f()` in a loop is a bug waiting for load.
4. Return errors to the owner; do not log-and-forget inside goroutines.
5. Verify with `go test -race` and, for leaks, the `goroutineleak` pprof profile (Go 1.27).

## Decision guide

| Need | Pattern |
|---|---|
| Run N independent tasks, fail fast on first error | `errgroup.WithContext` |
| Same, but cap concurrency | `errgroup` + `g.SetLimit(n)` |
| Long-lived consumers of a queue | Worker pool reading from a channel |
| Multi-stage processing | Pipeline of stages connected by channels |
| Spread work then merge results | Fan-out / fan-in |
| Cap concurrent use of a resource | Semaphore (buffered channel or `golang.org/x/sync/semaphore`) |
| Limit request rate | `golang.org/x/time/rate` |
| Wait for many goroutines without errors | `sync.WaitGroup` with `wg.Go(f)` (Go 1.25+) |
| One-time init shared by goroutines | `sync.Once` / `sync.OnceValue` |

## errgroup with bounded parallelism

```go
func FetchAll(ctx context.Context, ids []string, get func(context.Context, string) (Item, error)) ([]Item, error) {
	g, ctx := errgroup.WithContext(ctx)
	g.SetLimit(8) // bounded parallelism

	items := make([]Item, len(ids)) // each goroutine writes its own index: no mutex needed
	for i, id := range ids {
		g.Go(func() error {
			it, err := get(ctx, id)
			if err != nil {
				return fmt.Errorf("fetch %s: %w", id, err)
			}
			items[i] = it
			return nil
		})
	}
	if err := g.Wait(); err != nil {
		return nil, err
	}
	return items, nil
}
```

Since Go 1.22 the loop variables `i` and `id` are per-iteration, so capturing them in the closure is safe.

## Worker pool

```go
func Run(ctx context.Context, jobs <-chan Job, workers int, handle func(context.Context, Job) error) error {
	g, ctx := errgroup.WithContext(ctx)
	for range workers {
		g.Go(func() error {
			for {
				select {
				case <-ctx.Done():
					return ctx.Err()
				case j, ok := <-jobs:
					if !ok {
						return nil // producer closed the channel: done
					}
					if err := handle(ctx, j); err != nil {
						return err
					}
				}
			}
		})
	}
	return g.Wait()
}
```

## Pipeline and fan-in

```go
func gen(ctx context.Context, nums ...int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for _, n := range nums {
			select {
			case out <- n:
			case <-ctx.Done():
				return
			}
		}
	}()
	return out
}

func merge[T any](ctx context.Context, cs ...<-chan T) <-chan T {
	out := make(chan T)
	var wg sync.WaitGroup
	for _, c := range cs {
		wg.Go(func() { // Go 1.25+; use wg.Add(1) + defer wg.Done() on older versions
			for v := range c {
				select {
				case out <- v:
				case <-ctx.Done():
					return
				}
			}
		})
	}
	go func() { wg.Wait(); close(out) }()
	return out
}
```

Every send is inside a `select` with `ctx.Done()`, so cancelling the context unblocks all stages and nothing leaks.

## Timeouts

```go
ctx, cancel := context.WithTimeout(ctx, 2*time.Second)
defer cancel() // always release resources
```

Prefer deadlines on the context over `time.After` in loops. In a serverless function, derive from the invocation context, which already carries the function deadline.

## Graceful shutdown

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()
// start server/workers with ctx; on ctx.Done(), call srv.Shutdown with a fresh timeout context.
```

## Mutex vs channel

- Mutex: protecting state (a cache, a counter). Keep the critical section small; document which fields it guards.
- Channel: transferring ownership of data or signalling events between goroutines.
- `sync/atomic` typed values (`atomic.Int64`, `atomic.Pointer[T]`) for single counters and flags.

## Serverless and managed-queue notes

- Check each SDK's documentation for goroutine-safety; most vendor clients are safe for concurrent use and should be shared across goroutines rather than created per call.
- Respect service quotas and rate limits: bound parallelism and rely on the client's retry and backoff for throttling (see `shp-patterns-resilience`).
- For batch handlers fed by a managed queue, process records concurrently with a limit and report which records failed instead of failing the whole batch.

## Anti-patterns

- `go func() { ... }()` with no way to stop or report errors.
- `time.Sleep` for synchronisation, in code or tests.
- Closing a channel from the receiver side, or from more than one sender.
- Holding a mutex while doing network I/O.

## Sources

- Go blog, "Go Concurrency Patterns: Pipelines and cancellation": https://go.dev/blog/pipelines
- Go blog, "Go Concurrency Patterns: Context": https://go.dev/blog/context
- Go blog, "Advanced Go Concurrency Patterns": https://go.dev/blog/io2013-talk-concurrency
- Go memory model: https://go.dev/ref/mem
- `errgroup`: https://pkg.go.dev/golang.org/x/sync/errgroup

Provider-specific details live in the cloud-provider skill packs (for example the `shp-aws-*` skills).
