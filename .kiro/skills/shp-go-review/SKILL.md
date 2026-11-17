---
name: shp-go-review
description: Structured code review checklist for Go changes - correctness, error handling, context and concurrency, API design, tests, security, performance, AWS SDK usage and spec compliance - with severity-ranked findings. Use when asked to review a diff, a PR, a file, or the output of a Kiro spec task, or before committing Go code. Reports findings; it does not rewrite code unless asked.
license: MIT
metadata:
  kit: shuppatsu-go
  version: "1.0.0"
---

# shp-go-review

## Workflow

1. **Scope.** Identify the change: `git diff` (or `git diff --staged`), the files the user named, or the files touched by the last spec task. Read the related spec (`.kiro/specs/<feature>/`) to know the intended behaviour.
2. **Run the cheap checks first** (skip any tool that is not installed and say so):
   - `gofmt -l .` (must print nothing)
   - `go vet ./...`
   - `go build ./...`
   - `go test -short ./...`
   - optional: `golangci-lint run`, `govulncheck ./...`, `go test -race ./...` (needs cgo)
3. **Read the code against the checklist** below. Look at callers and tests, not only the diff.
4. **Report** findings ranked by severity using the output format. Only report issues you can point to; label anything uncertain as a question.

## Checklist

### Correctness
- Logic matches the acceptance criteria in the spec; edge cases (empty, nil, zero, max, duplicates) handled.
- No ignored errors (`_ = f()` needs a comment explaining why).
- Nil maps are not written to; slices aliasing after `append` is understood; no out-of-range indexing.
- Integer overflow and unit mistakes (ms vs s, bytes vs KiB) checked.

### Errors
- Wrapped with context using `%w`; checked with `errors.Is/As/AsType`, never by string.
- Handled once (not logged and returned).
- Not wrapping a pointer to an error type with `%w` when callers use `errors.Is` (Go 1.27 vet flags this).

### Context and concurrency
- `ctx` is the first parameter for I/O; not stored in structs; cancellation respected in loops and selects.
- Every goroutine has an exit path; channels closed by the sender; bounded parallelism.
- Shared state protected; no data races (run `-race` where possible).

### API and design
- Accept interfaces, return concrete types; interfaces small and declared by the consumer.
- Package names and exported identifiers follow Go naming; no stutter; doc comments on exported symbols.
- No premature abstraction; patterns used where they reduce complexity (`shp-patterns-*`).
- Respects the `go` directive (no newer APIs) and the project structure steering.

### Tests
- New behaviour has table-driven tests covering error paths; tests are deterministic (no sleeps, injected clock).
- Fakes over mocks; helpers call `t.Helper()`; `-short` skips slow tests.

### Security
- No secrets, tokens or credentials in code, tests, logs or fixtures.
- Input validated at boundaries; SQL parameterised; no `InsecureSkipVerify`; HTTP clients have timeouts.
- Least-privilege IAM in any IaC change.

### AWS SDK and Lambda
- One `LoadDefaultConfig` and long-lived clients; context passed to every call.
- Paginators and waiters instead of manual loops; response bodies closed.
- Lambda handlers idempotent; init outside the handler; partial batch failures for SQS.

### Performance (only when relevant)
- No allocation or I/O inside hot loops without need; preallocate when size is known.
- Claims about performance backed by a benchmark (`b.Loop()`).

## Output format

```markdown
## Review summary
<one paragraph: overall assessment and merge recommendation>

## Findings
| # | Severity | Location | Issue | Suggested fix |
|---|---|---|---|---|
| 1 | blocker / major / minor / nit | path/file.go:42 | ... | ... |

## Checks run
- gofmt: ok | go vet: ok | go test -short: 2 failures | golangci-lint: not installed

## Questions
- <anything that needs the author's intent>
```

Severity: **blocker** (bug, data loss, security issue, broken build), **major** (likely bug or maintainability problem), **minor** (idiom or clarity), **nit** (style preference). If the user wants to understand a finding, hand off to `shp-teach`.

## Sources

- Go Code Review Comments: https://go.dev/wiki/CodeReviewComments
- Go Test Comments: https://go.dev/wiki/TestComments
- Effective Go: https://go.dev/doc/effective_go
