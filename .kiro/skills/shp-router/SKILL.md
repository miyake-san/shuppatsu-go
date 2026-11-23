---
name: shp-router
description: Entry point that tells the agent which shp-* skill to load for the current phase of Go work - define, build, verify, review, ship - and in what order. Use at the start of a Go or AWS task, when unsure which kit skill applies, or when two skills seem to overlap.
license: MIT
metadata:
  kit: shuppatsu-go
  version: "1.1.0"
  inspired_by: "https://github.com/addyosmani/agent-skills (using-agent-skills, MIT)"
---

# shp-router

Load only the skills the current phase needs (progressive disclosure). Do not load all of them.

## Phase map

| Phase | Situation | Load |
|---|---|---|
| Define | New feature, unclear requirements | Kiro specs (`requirements.md`, `design.md`, `tasks.md`); record decisions with `shp-adr` |
| Design | Choosing structure or a pattern | `shp-patterns-select` first, then the matching `shp-patterns-*` |
| Design | Exported API, HTTP endpoint, event schema | `shp-api-design` |
| Build | Implementing any behaviour | `shp-incremental-build` + `shp-tdd` |
| Build | Cloud-provider specifics (only when the project uses that provider) | The provider pack, e.g. AWS: `shp-aws-sdk`, `shp-aws-lambda`, `shp-aws-resilience`, `shp-aws-observability` |
| Build | Remote calls, queues, retries | `shp-patterns-resilience` |
| Build | Goroutines, pipelines | `shp-patterns-concurrency` |
| Verify | Test failing, build broken, odd behaviour | `shp-debugging` |
| Verify | High-stakes or irreversible change | `shp-doubt-check` |
| Review | Reviewing a diff or PR | `shp-go-review` (agent `shp-go-reviewer`) |
| Review | Code works but is hard to read | `shp-simplify` |
| Ship | Adding logs, metrics, traces, alerts | `shp-observability` |
| Verify | Check work against official docs (API, version, idiom) | `shp-go-verify` (trusted Go sites; provider claims are checked against the provider's official docs by its pack) |
| Learn | "Why is it written this way?" | `shp-teach` (agent `shp-go-mentor`) |

## Which agent

| Work | Agent |
|---|---|
| Cloud-agnostic Go: services, libraries, CLIs, tests, refactors, patterns, performance | `shp-go-expert` |
| Cloud provider work (pack per provider) | AWS: `shp-aws-builder` (SDK, Lambda, event sources, IAM, IaC). Future packs: `shp-gcp-*`, `shp-azure-*` |
| Explain why | `shp-go-mentor` (read-only) |
| Review a change | `shp-go-reviewer` (read-only) |

The Go set (`shp-go-*`, `shp-patterns-*`, process and design skills) is cloud-agnostic. Each cloud provider is a separate pack named `shp-<provider>-*`, so adding a provider never changes the Go set. All agents validate their technical claims against trusted documentation before reporting.

## Typical flows

- **New feature:** spec → `shp-patterns-select` (if structure is in question) → `shp-api-design` (if a contract is involved) → `shp-incremental-build` + `shp-tdd` per slice → `shp-observability` → `shp-go-review`.
- **Bug fix:** `shp-debugging` (reproduce) → `shp-tdd` (failing test, fix) → `shp-go-review`.
- **Refactor:** `shp-tdd` (characterization tests) → `shp-simplify` → `shp-go-review`.
- **Risky change:** add `shp-doubt-check` before merging, and `shp-adr` if the decision is lasting.

## Rules

1. One methodology: these skills are the process backbone. If an external skill set is also installed (for example `golang-*`, `go-*`, `aws-*`), pick one skill per topic and say which.
2. Language and AWS specifics live in steering and the `shp-go-*`/`shp-aws-*` skills; process lives in the process skills.
3. Every skill ends with a verification gate; do not report "done" without its evidence.
4. If the user's request is trivial (one-line fix), do it; do not run the whole flow.

## Sources

- Agent Skills specification: https://agentskills.io
- Kiro skills and steering: https://kiro.dev/docs/skills/ , https://kiro.dev/docs/steering/

## Inspired by

Routing idea inspired by `using-agent-skills` in addyosmani/agent-skills (MIT). Phase map is original to this kit. See `THIRD_PARTY_NOTICES.md`.
