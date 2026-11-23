# Authoritative sources by topic

Use the most specific page. Prefer official Go sources over blogs. Offline fallback: `go doc <pkg>.<Symbol>`.

## Language and core references

| Topic | Source |
|---|---|
| Language rules | Go spec: https://go.dev/ref/spec |
| Memory model, data races | https://go.dev/ref/mem |
| Idioms (overall) | Effective Go: https://go.dev/doc/effective_go |
| Review conventions | Go Code Review Comments: https://go.dev/wiki/CodeReviewComments |
| Test conventions | Go Test Comments: https://go.dev/wiki/TestComments |
| Table-driven tests | https://go.dev/wiki/TableDrivenTests |
| FAQ (design rationale: no inheritance, no exceptions...) | https://go.dev/doc/faq |
| Modules | https://go.dev/ref/mod |
| Project layout | https://go.dev/doc/modules/layout |
| Release notes (feature availability) | https://go.dev/doc/devel/release, https://go.dev/doc/go1.27, https://go.dev/doc/go1.26 |
| Standard library API | https://pkg.go.dev/std (offline: `go doc`) |
| Go proverbs | https://go-proverbs.github.io/ |
| Google Go style guide | https://google.github.io/styleguide/go/ |
| Uber Go style guide | https://github.com/uber-go/guide/blob/master/style.md |

## Go blog articles by theme

| Theme | Article |
|---|---|
| Error wrapping, `errors.Is/As` | https://go.dev/blog/go1.13-errors |
| Errors as values | https://go.dev/blog/errors-are-values |
| defer / panic / recover | https://go.dev/blog/defer-panic-and-recover |
| Context | https://go.dev/blog/context |
| Pipelines and cancellation | https://go.dev/blog/pipelines |
| Advanced concurrency patterns | https://go.dev/blog/io2013-talk-concurrency |
| Share memory by communicating | https://go.dev/blog/codelab-share |
| Generics introduction | https://go.dev/blog/intro-generics |
| When to use generics | https://go.dev/blog/when-generics |
| Range-over-func iterators | https://go.dev/blog/range-functions |
| Structured logging | https://go.dev/blog/slog |
| Package names | https://go.dev/blog/package-names |
| Fuzzing | https://go.dev/doc/security/fuzz/ |
| Vulnerability management | https://go.dev/doc/security/vuln/ |
| Diagnostics (pprof, trace) | https://go.dev/doc/diagnostics |

## Design patterns

| Topic | Source |
|---|---|
| GoF patterns with Go examples | https://refactoring.guru/design-patterns/go |
| Community Go pattern catalogue | https://github.com/tmrts/go-patterns |
| Microservice and distributed-systems patterns (outbox, saga, CQRS) | https://microservices.io/patterns/ |
| Enterprise application patterns (repository, unit of work) | https://martinfowler.com/eaaCatalog/ |
| Kit pattern skills | `shp-patterns-creational`, `shp-patterns-structural`, `shp-patterns-behavioral`, `shp-patterns-concurrency`, `shp-patterns-architecture`, `shp-patterns-resilience` |

## Cloud-provider documentation

Provider documentation (SDKs, serverless runtimes, managed services, well-architected guidance) belongs to the cloud-provider packs, each with its own source map. This file covers the provider-neutral Go sources only.
