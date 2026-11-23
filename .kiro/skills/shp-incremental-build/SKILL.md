---
name: shp-incremental-build
description: Deliver Go changes as thin vertical slices - each slice compiles, passes tests and is committed before the next starts - with scope discipline and a change-size budget. Use when implementing a Kiro spec task, any change touching more than one file, a large refactor, a feature behind a flag, or whenever a task feels too big to land in one step.
license: MIT
metadata:
  kit: shuppatsu-go
  version: "1.3.0"
  inspired_by: "https://github.com/addyosmani/agent-skills (incremental-implementation, MIT)"
---

# shp-incremental-build

Big-bang changes hide bugs and make review impossible. Build in slices that each leave the repository green.

## Slice rules

1. **Vertical, not horizontal.** A slice cuts through the layers for one behaviour (handler → service → adapter → test), not "all the models first".
2. **Small.** Aim for about 100 changed lines (excluding generated code and tests' table rows). Above ~300, split.
3. **Always green.** At the end of a slice: `gofmt -l .` prints nothing, `go build ./...`, `go vet ./...` and `go test -short ./...` pass.
4. **Test first** inside the slice (`shp-tdd`).
5. **One commit per slice** with a message that says what behaviour now works. Commit only when the user asked for commits; otherwise leave the tree green and report.
6. **Scope discipline.** Touch only what the slice needs. Spot a nearby smell or bug? Write it down in the final report; do not fix it in this slice. Do not rename, reformat or reorganise unrelated code.

## Order of slices

| Slice | Content |
|---|---|
| 0 | Walking skeleton: the thinnest path that compiles and is called end to end (even returning a constant) |
| 1..n | One acceptance criterion each, in risk order (riskiest or most uncertain first) |
| n+1 | Hardening: timeouts, error paths, idempotency, observability (`shp-observability`, `shp-patterns-resilience`) |
| last | Docs: README or ADR if a decision was made (`shp-adr`) |

When working from Kiro specs, map one `tasks.md` task to one or more slices; never start a task whose predecessor is red.

## Rolling out risky changes

- Behind a flag or config value that defaults to the old behaviour.
- Expand then contract for schema or API changes: add the new shape, migrate readers and writers, then remove the old one in a later change (see `shp-api-design`).
- Keep changes reversible: no destructive cloud, infrastructure or database command without an explicit user OK.

## Anti-rationalization

| Excuse | Rebuttal |
|---|---|
| "It is faster to do it all at once" | Until the first red build with 800 changed lines and no idea which one broke it. |
| "I'll fix this unrelated thing while I'm here" | It widens the diff and the review. Note it; do it in its own slice. |
| "Tests can wait until the end" | Then slices are not independently verifiable. |
| "I'll commit when everything works" | You will have no safe point to return to. |

## Verification gate (evidence required)

For each slice, show:

- the commands run and their results (`go build`, `go vet`, `go test`);
- the diff size (`git diff --stat`) and that it matches the slice's stated scope;
- the list of things noticed but deliberately left alone.

## Sources

- Trunk-based development and small changes: https://trunkbaseddevelopment.com/ , Google Engineering Practices, "Small CLs": https://google.github.io/eng-practices/review/developer/small-cls.html
- Go commit and change conventions: https://go.dev/doc/contribute#commit_messages
- Expand and contract (parallel change): https://martinfowler.com/bliki/ParallelChange.html
- Kiro specs and tasks: https://kiro.dev/docs/specs/

## Inspired by

Slice sizing, always-green rule and scope discipline inspired by `incremental-implementation` in addyosmani/agent-skills (MIT) and Google's Engineering Practices. Text and Go/Kiro mapping are original. See `THIRD_PARTY_NOTICES.md`.
