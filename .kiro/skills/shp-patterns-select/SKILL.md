---
name: shp-patterns-select
description: Decide whether a Go design needs a pattern at all - reuse what exists, refactor first, introduce a pattern, or do nothing - using the Rule of Three and a verdict scale for over-engineering. Routes to the shp-patterns-* skills. Use before introducing an interface, factory, builder, decorator, strategy or layer, when reviewing for over-engineered abstractions, or when the user asks which pattern to use.
license: MIT
metadata:
  kit: shuppatsu-go
  version: "1.3.0"
  inspired_by: "https://github.com/fidelisakilan/design-pattern-skills (/pattern, MIT); https://github.com/jaeleeps/gof-patterns (verdict scale, MIT); https://github.com/samber/cc-skills-golang (golang-design-patterns, MIT)"
---

# shp-patterns-select

Patterns solve recurring pain. No pain, no pattern. In Go the default answer to "which pattern?" is often "a function and a struct".

## Step 1: name the pain

Write one sentence: *"Today X is hard because Y."* Examples: "adding a payment provider means editing five files", "tests need a real database", "the constructor has nine parameters". If you cannot write it, **stop: no pattern**.

## Step 2: choose one of four outcomes

| Outcome | When | Next |
|---|---|---|
| **Do nothing** | No concrete pain, or one use case | Keep the plain code |
| **Reuse** | The standard library or an existing helper already solves it (`http.Handler` middleware, `sort.Interface`, `io.Reader`, `errgroup`, `sync.Once`) | Use it; do not wrap it |
| **Refactor first** | The pain is a smell, not a missing pattern: long function, duplicated branches, hidden global, mixed concerns | Extract function, split package, remove duplication; re-evaluate afterwards |
| **Introduce a pattern** | Rule of Three holds: three real variants or call sites exist (or a test seam is required) | Pick from the table below |

**Rule of Three:** do not abstract on the first or second occurrence. The third shows what actually varies.

## Step 3: pick the pattern

| Pain | Pattern | Skill |
|---|---|---|
| Hard to construct, many optional settings | Functional options, constructor | `shp-patterns-creational` |
| Incompatible interfaces, cross-cutting wrapper (logging, retry, auth) | Adapter, decorator, middleware | `shp-patterns-structural` |
| Behaviour that varies at runtime | Strategy, state, command, chain | `shp-patterns-behavioral` |
| Parallel work, pipelines, shutdown | errgroup, worker pool, fan-out | `shp-patterns-concurrency` |
| Domain logic tangled with cloud SDKs, a database or HTTP | Ports and adapters, repository, service layer | `shp-patterns-architecture` |
| Remote calls fail, duplicates, partial failure | Timeouts, retry with jitter, idempotency, outbox, saga | `shp-patterns-resilience` |

Go-specific idioms to check before reaching for a GoF pattern: consumer-side interfaces, small interfaces, embedding, closures as strategies, `iota` enums with a validating `String`/`Parse`, `init()` only for registration with no side effects, panic only for programmer errors.

## Step 4: audit verdicts (use when reviewing existing code)

For each pattern found, cite `file:line` evidence and rate it:

| Verdict | Meaning |
|---|---|
| **Appropriate** | Solves a present, named pain with the least machinery |
| **Over-engineered** | Abstraction with one implementation, speculative extension point, layers that only forward calls |
| **Misapplied** | Wrong pattern for the pain (e.g. inheritance-style hierarchy, service locator instead of injection) |
| **Degraded** | Was appropriate, now eroded by special cases or leaks; refactor or simplify (`shp-simplify`) |

## Anti-rationalization

| Excuse | Rebuttal |
|---|---|
| "We will need another implementation later" | Add the interface when the second one arrives; moving it is cheap in Go. |
| "It is the textbook pattern" | Textbooks assume class hierarchies. State the pain it solves here. |
| "More layers are cleaner" | Every layer is code to read, test and keep in sync. |
| "Everyone uses a factory" | Show the three call sites that need one. |

## Verification gate

Report: the pain sentence, the outcome chosen, the pattern (or "none") and why, and for audits the verdict table with evidence. A recommendation without a pain sentence is incomplete.

## Sources

- Effective Go, interfaces and embedding: https://go.dev/doc/effective_go
- Go Code Review Comments, interfaces: https://go.dev/wiki/CodeReviewComments#interfaces
- Rule of Three (refactoring): https://martinfowler.com/bliki/BeckDesignRules.html and *Refactoring*, M. Fowler
- Refactoring.Guru, patterns in Go: https://refactoring.guru/design-patterns/go
- Go proverbs: https://go-proverbs.github.io/

## Inspired by

The four-outcome decision (reuse, introduce, refactor first, no pattern) is inspired by the `/pattern` command in fidelisakilan/design-pattern-skills (MIT); the verdict scale by jaeleeps/gof-patterns (MIT); the Go idiom topics by samber/cc-skills-golang `golang-design-patterns` (MIT). Text is original and Go-specific. See `THIRD_PARTY_NOTICES.md`.
