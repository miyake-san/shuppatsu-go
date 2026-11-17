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

### Five axes (cross-check before reporting)
Every finding belongs to one axis; make sure each axis was actually looked at:
1. **Correctness:** does it do what the spec says, including edge cases and failure paths?
2. **Readability:** could a new teammate follow it? Names, structure, comments that explain why.
3. **Architecture:** package boundaries, dependency direction, fit with `structure.md` and existing patterns.
4. **Security:** input handling, secrets, IAM, dependencies (`shp-security`).
5. **Performance:** only where it matters, backed by a benchmark or profile.

### Change size
- Prefer reviews of about 100 changed lines; above ~300 lines ask the author to split (`shp-incremental-build`), or review in slices and say so.
- Flag unrelated changes mixed into the diff (renames, reformatting, drive-by fixes) as **minor** and ask for a separate change.
- Approve when the change definitely improves overall code health, even if it is not how you would write it; mark preferences as **nit**.

### Dead and unverified claims
- "Fixed" or "done" without evidence (test output, command run) is a **major** finding.
- Tests deleted, skipped (`t.Skip`) or assertions loosened in the same diff are a **blocker** until explained.

## Audit mode (optional): pattern verdicts
When the user asks to audit design patterns or over-engineering, list each pattern found with `file:line` evidence and a verdict: **Appropriate**, **Over-engineered**, **Misapplied** or **Degraded** (scale and decision process in `shp-patterns-select`). For risky changes, also run `shp-doubt-check`.

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
- Google Engineering Practices, code review standard and small changes: https://google.github.io/eng-practices/review/reviewer/standard.html , https://google.github.io/eng-practices/review/developer/small-cls.html

## Inspired by

The five-axis pass, ~100-line change sizing and "evidence for done" rule are inspired by `code-review-and-quality` in addyosmani/agent-skills (MIT) and Google's Engineering Practices; the pattern verdict scale by jaeleeps/gof-patterns (MIT). See `THIRD_PARTY_NOTICES.md`.
