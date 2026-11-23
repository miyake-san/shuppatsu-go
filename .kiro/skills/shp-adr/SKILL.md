---
name: shp-adr
description: Record architecture decisions as short ADRs in docs/adr, with context, options, decision and consequences, so shp-teach and future maintainers can cite the why. Use when choosing between technologies or patterns, changing a public API, accepting a trade-off, deviating from steering, or when a spec design.md contains a decision worth keeping.
license: MIT
metadata:
  kit: shuppatsu-go
  version: "1.3.0"
  inspired_by: "https://github.com/addyosmani/agent-skills (documentation-and-adrs, MIT); https://adr.github.io/"
---

# shp-adr

Code shows **what**; an ADR records **why** and what was rejected. Write one when the decision is hard to reverse, surprising, or likely to be re-litigated.

## When to write one

Yes: new framework or cloud service, data store choice, sync vs async, module or package boundary, error and retry policy, public API shape, deviation from kit steering, security trade-off.
No: renames, formatting, bug fixes, anything obvious from the code or a one-line comment.

## Where and how

- Files: `docs/adr/NNNN-short-title.md`, numbered, never renumbered. Keep a one-line index in `docs/adr/README.md`.
- Immutable once accepted: to change a decision, write a new ADR and mark the old one `Superseded by NNNN`.
- Keep it to one page. Link specs (`.kiro/specs/<feature>/design.md`), issues and PRs.

## Template (MADR-style)

```markdown
# NNNN. <Decision in a short phrase>

- Status: proposed | accepted | superseded by NNNN | deprecated
- Date: YYYY-MM-DD
- Deciders: <names or roles>

## Context
The forces at play: requirements, constraints, what hurts today. Facts, not advocacy.

## Options considered
1. <Option A>: pros, cons
2. <Option B>: pros, cons
3. <Do nothing>: pros, cons

## Decision
We will <option>, because <the deciding reasons>.

## Consequences
- Positive: ...
- Negative / costs: ...
- Follow-ups and how we will know it is wrong (revisit trigger): ...

## References
Docs, benchmarks, spec links.
```

## Writing rules

- Always include **"do nothing"** as an option and at least one real alternative.
- State the **revisit trigger** (e.g. "revisit if p99 exceeds 300 ms or traffic passes 500 rps").
- Cite evidence: benchmarks (`go test -bench`), docs, quotas, prices; mark guesses as guesses.
- Record decisions at the moment you make them, in the same PR as the code.

## Link with the rest of the kit

- `shp-teach` cites ADRs when asked "why is it this way?".
- `shp-patterns-select` outcomes that introduce a pattern for a non-obvious reason deserve an ADR.
- Agents add a one-line rationale in the PR or spec; promote it to an ADR when it meets the criteria above.

## Anti-rationalization

| Excuse | Rebuttal |
|---|---|
| "The code is self-documenting" | It shows what, never what was rejected or why. |
| "We will remember" | Not in six months, not for the next person. |
| "ADRs are bureaucracy" | One page, written once, saves repeated debates. |
| "Edit the old ADR" | History matters; supersede instead. |

## Verification gate

The ADR has status, date, at least two options including "do nothing", a decision with reasons, consequences with a revisit trigger, and is indexed in `docs/adr/README.md`.

## Sources

- Michael Nygard, "Documenting Architecture Decisions": https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions
- ADR templates and tooling: https://adr.github.io/ , MADR: https://adr.github.io/madr/
- ThoughtWorks Technology Radar, lightweight ADRs: https://www.thoughtworks.com/radar/techniques/lightweight-architecture-decision-records
- Go doc comments (for in-code documentation): https://go.dev/doc/comment

## Inspired by

ADR habit and documentation rules inspired by `documentation-and-adrs` in addyosmani/agent-skills (MIT), Nygard's original article and MADR. Template text is original. See `THIRD_PARTY_NOTICES.md`.
