---
inclusion: fileMatch
fileMatchPattern: ["**/*.go", "**/go.mod"]
---

# Go style (shuppatsu-go kit)

Sources: Effective Go, Go Code Review Comments, the Go spec, Google and Uber Go style guides. When the user asks *why* something is written a certain way, use the `shp-teach` skill.

## Formatting and naming

- Code is always `gofmt`-clean. Imports grouped: standard library, then third-party, then local.
- `MixedCaps`, never underscores. Initialisms keep one case: `userID`, `HTTPClient`, `parseURL`.
- Short receiver names (1-2 letters), consistent across methods. No `self`/`this`.
- No stutter: `order.New`, not `order.NewOrder`; `http.Client`, not `http.HTTPClient`.
- Getters have no `Get` prefix: `u.Name()`, setter `u.SetName()`.

## Errors

- Return errors, do not panic. `panic` only for programmer errors that cannot happen at runtime.
- Add context when propagating: `fmt.Errorf("load order %s: %w", id, err)`. Lower-case messages, no trailing punctuation.
- Inspect with `errors.Is` / `errors.As` (or `errors.AsType[T]` when `go.mod` is 1.26+), never by comparing strings.
- Sentinel errors (`var ErrNotFound = errors.New("not found")`) for conditions callers branch on; custom types when callers need data.
- Handle an error once: either log it or return it, not both.

## APIs and types

- `context.Context` is the first parameter of anything that does I/O or may block. Never store it in a struct.
- Accept interfaces, return concrete types. Declare small interfaces in the consumer package.
- Make the zero value useful; use constructors (`NewX`) only when invariants must be established.
- Use functional options for optional configuration with more than ~3 knobs (see `shp-patterns-creational`).
- Pointer receivers when the method mutates or the struct is large; be consistent per type.
- Generics only when they remove real duplication across types; prefer interfaces for behaviour.

## Concurrency

- Never start a goroutine without knowing how it stops (context cancellation, closed channel, `errgroup`).
- Prefer `errgroup.Group` / `sync.WaitGroup.Go` (1.25+) over hand-rolled counters.
- Protect shared state with a mutex placed next to the fields it guards; or share by communicating.
- See `shp-patterns-concurrency` for pipelines, worker pools and fan-out/fan-in.

## Modern Go, gated by the `go` directive

| Since | Use |
|---|---|
| 1.21 | `min`, `max`, `clear`; packages `slices`, `maps`, `cmp`, `log/slog` |
| 1.22 | per-iteration loop variables, `for i := range n`, `ServeMux` method/wildcard patterns, `math/rand/v2` |
| 1.23 | range-over-func iterators (`iter.Seq`, `slices.Collect`, `maps.Keys`) |
| 1.24 | `t.Context()`, `b.Loop()`, `omitzero` JSON tag, `os.Root`, `tool` directives in `go.mod` |
| 1.25 | `sync.WaitGroup.Go`, `testing/synctest` |
| 1.26 | `new(expr)`, `errors.AsType`, `slog.NewMultiHandler`, `go fix` modernizers |
| 1.27 | generic methods, `uuid` package, `strings.CutLast`, `encoding/json` backed by v2 |

## Logging

- `log/slog` with key-value attributes; JSON handler in deployed code. Never log secrets or full payloads with personal data.
