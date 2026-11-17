---
name: shp-simplify
description: Behaviour-preserving simplification of working Go code - flatten nesting, remove dead code and needless abstraction, shorten functions - guarded by tests and Chesterton's Fence. Use when code works but is harder to read than it should be, after a feature lands, when a review flags complexity, or when asked to clean up, simplify or reduce nesting.
license: MIT
metadata:
  kit: shuppatsu-go
  version: "1.1.0"
  inspired_by: "https://github.com/addyosmani/agent-skills (code-simplification, MIT)"
---

# shp-simplify

Simplify to make the next change cheaper, never to show off. **Behaviour must not change.**

## Preconditions

1. Tests exist and are green for the code in scope. If not, write characterization tests first (`shp-tdd`).
2. Scope is explicit: one function, file or package. Do not simplify and change behaviour in the same commit.
3. **Chesterton's Fence:** before removing something odd (a check, a retry, a lock, a comment), find out why it exists: `git log -S`, `git blame`, the spec, an ADR. Remove it only when you can explain why it is no longer needed.

## Simplifications that usually pay off in Go

| Smell | Simplification |
|---|---|
| Deep `if`/`else` nesting | Guard clauses with early `return`; happy path left-aligned |
| Long function doing several jobs | Extract named functions at the same level of abstraction |
| `else` after `return` | Drop the `else` |
| Interface with one implementation and no test seam | Use the concrete type; reintroduce the interface when needed (`shp-patterns-select`) |
| Pass-through layers (`service` that only calls `repo`) | Remove the layer or give it a real responsibility |
| Hand-written loops | `slices`, `maps`, `cmp`, `min`/`max`, `for range n` (check the `go` directive) |
| Manual error string matching | `errors.Is/As` with wrapped errors |
| Boolean flag parameters | Two functions or an options value |
| Repeated struct setup | A small constructor or test helper (but keep tests DAMP) |
| Dead code, unused params, stale comments | Delete (`go vet`, `staticcheck`/`golangci-lint unused`) |
| Clever one-liners, named returns with naked `return` in long funcs | Plain, explicit code |

`go fix ./...` (Go 1.26+) applies safe modernizers; review its diff like any other.

## Procedure

1. Run the tests; record green.
2. Make **one** simplification.
3. Run `gofmt`, `go vet ./...`, `go test ./...`; confirm still green.
4. Repeat; stop when the next change would add risk without clear readability gain.
5. If a test must change, ask: is it testing behaviour (keep) or an implementation detail (rewrite)? Never loosen an assertion to get green.

## Anti-rationalization

| Excuse | Rebuttal |
|---|---|
| "While simplifying I'll fix this bug" | Separate commit; otherwise a regression hides inside a refactor. |
| "Fewer lines is simpler" | Optimise for reading time, not line count. |
| "This check looks pointless, remove it" | Find out why it is there first (Chesterton's Fence). |
| "No tests, but it is a trivial refactor" | Trivial refactors break things quietly; write characterization tests. |

## Verification gate (evidence required)

- Tests green before and after (show both runs); no assertion removed or weakened.
- `git diff --stat` shows a net reduction in complexity (fewer branches, shorter functions, less code), and the diff contains no behaviour change.
- `go vet ./...` clean; benchmarks unchanged if the code is on a hot path.

## Sources

- Effective Go, control structures and "Least surprise": https://go.dev/doc/effective_go#control-structures
- Go Code Review Comments, indent error flow: https://go.dev/wiki/CodeReviewComments#indent-error-flow
- `go fix` and modernizers: https://go.dev/blog/gofix
- Chesterton's Fence: https://fs.blog/chestertons-fence/
- Refactoring catalogue: https://refactoring.guru/refactoring/techniques

## Inspired by

Behaviour-preserving rule and Chesterton's Fence inspired by `code-simplification` in addyosmani/agent-skills (MIT). Go smell table is original. See `THIRD_PARTY_NOTICES.md`.
