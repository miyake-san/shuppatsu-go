# shuppatsu-go

A spec-driven Golang starter template pre-configured for Kiro, featuring opinionated project layout, steering rules, and automated workflows.

The `.kiro/` folder is a self-contained, **cloud-agnostic Go expert kit** (steering, skills including a teaching skill and design-pattern skills, custom agents, quality hooks and an MCP configuration) plus **cloud-provider packs**, one per provider (`shp-aws-*` today; `shp-gcp-*`, `shp-azure-*` and others can be added without touching the Go set). It works offline: nothing needs `npx`, `uvx` or internet access, which makes it usable in restricted corporate environments.

## Contents

```text
.kiro/
├── shp-kit.md                      # kit summary (also shipped inside the zips)
├── steering/
│   ├── product.md                  # project: product context (fill in)
│   ├── tech.md                     # project: stack, toolchain, commands
│   ├── structure.md                # project: layout and dependency rules
│   ├── shp-go-style.md             # Go idioms, errors, concurrency, version-gated features
│   ├── shp-go-testing.md           # testing conventions
│   ├── shp-security.md             # security baseline
│   └── shp-aws-conventions.md      # AWS pack: SDK v2, Lambda, IAM, ops
├── skills/
│   ├── shp-router/                 # entry point: which skill and agent for which phase
│   ├── shp-go-verify/              # validate work against go doc and trusted sites
│   ├── shp-teach/                  # explains WHY code is written that way, with sources
│   ├── shp-tdd/  shp-incremental-build/  shp-debugging/  shp-doubt-check/  shp-simplify/  shp-adr/
│   ├── shp-patterns-select/  shp-api-design/  shp-observability/
│   ├── shp-go-review/
│   ├── shp-patterns-{creational,structural,behavioral,concurrency,architecture,resilience}/
│   └── shp-aws-{sdk,lambda,resilience,observability}/   # AWS pack
├── agents/
│   ├── shp-go-expert.md            # cloud-agnostic Go expert builder, validates against docs
│   ├── shp-go-mentor.md            # read-only teacher (uses shp-teach), validates claims
│   ├── shp-go-reviewer.md          # read-only reviewer (uses shp-go-review), validates findings
│   └── shp-aws-builder.md          # AWS pack: AWS specialist with AWS guardrails
├── hooks/
│   └── shp-go-quality.json         # gofmt on save; go vet + go test on Stop; optional extras
├── providers/
│   └── aws/mcp.aws.json            # AWS pack: MCP servers to merge into settings/mcp.json
└── settings/
    └── mcp.json                    # gopls, disabled by default
dist/
├── shuppatsu-go-kiro-kit-project.zip
└── shuppatsu-go-kiro-kit-global.zip
docs/
└── references.md                   # research, links, complementary materials, v1.1 changelog
THIRD_PARTY_NOTICES.md              # credits and MIT notices for the projects that inspired v1.1 skills
```

## Quick start

### Option A: use this repository as a template

Clone it (or use it as a GitHub template), open the folder in Kiro, then:

1. Fill in `.kiro/steering/product.md` and pick your IaC tool in `.kiro/steering/tech.md`.
2. Run `go mod init <module-path>`: the Stop hooks run `go vet` and `go test` and expect a `go.mod` at the workspace root.
3. Optionally enable MCP servers in `.kiro/settings/mcp.json` (see below).

### Option B: restricted environment (unzip the kit)

Download the zips from [`dist/`](dist/) (for example `https://github.com/miyake-san/shuppatsu-go/raw/main/dist/shuppatsu-go-kiro-kit-project.zip`). Each zip's root maps directly onto a `.kiro` folder.

| Zip | Unzip into | What you get |
|---|---|---|
| `shuppatsu-go-kiro-kit-project.zip` | `<your-project>\.kiro\` | Everything: steering, skills, agents, hooks, `settings/mcp.json` |
| `shuppatsu-go-kiro-kit-global.zip` | `%USERPROFILE%\.kiro\` | Only `shp-*` skills, agents and generic steering. No hooks, no `mcp.json`, no product-specific steering |

```powershell
# Project level, from the project root
Expand-Archive .\shuppatsu-go-kiro-kit-project.zip -DestinationPath .\.kiro

# User level (all projects)
Expand-Archive .\shuppatsu-go-kiro-kit-global.zip -DestinationPath $HOME\.kiro
```

Every file in the global zip is named `shp-*`, so it never overwrites your existing global files. The project zip also contains `product.md`, `tech.md`, `structure.md` and `settings/mcp.json`; avoid `-Force` if your project already has them. Restart Kiro or open a new chat after unzipping.

## What the kit provides

### Skills

| Skill | Use it when |
|---|---|
| `shp-teach` | You want to understand **why** code is written a certain way. Classifies each decision (language rule, idiom, pattern, platform guidance, project convention, trade-off), researches `go doc` and official Go docs, cites sources, and ends with a comprehension check. |
| `shp-router` | Start of a task, or unsure which skill applies. Maps phases (define, design, build, verify, review, ship) to skills and typical flows. |
| `shp-tdd` | Implementing behaviour or fixing a bug test-first: red-green-refactor, table-driven tests, fakes, fuzz and benchmarks, `go test` evidence gate. |
| `shp-incremental-build` | Delivering a spec task as thin, always-green slices (~100 lines) with scope discipline. |
| `shp-debugging` | A test fails, a goroutine hangs, a deployed function errors: reproduce, localize, reduce, fix, guard, with `-race`, Delve, pprof and logs. |
| `shp-doubt-check` | High-stakes or irreversible changes: state the claim, attack the assumptions, test the cheapest, record a verdict. |
| `shp-simplify` | Behaviour-preserving cleanup of working code, guarded by tests and Chesterton's Fence. |
| `shp-adr` | Recording a lasting architecture decision (`docs/adr`) with options, consequences and a revisit trigger. |
| `shp-patterns-select` | Before adding an interface, layer or pattern: reuse, refactor first, introduce a pattern, or do nothing (Rule of Three); verdicts for over-engineering. |
| `shp-api-design` | Designing exported Go APIs, HTTP/JSON endpoints and event schemas; error semantics, versioning, compatibility checks. |
| `shp-observability` | Adding `log/slog` logging, RED/USE metrics, OpenTelemetry traces, health checks and SLO alerts (provider backends live in the provider packs). |
| `shp-go-review` | Reviewing a diff, PR or spec task output with a severity-ranked checklist, five axes, change-size rules and an optional pattern-verdict audit. |
| `shp-patterns-creational` | Constructors, functional options, builder, factory, `sync.Once`, pools, `iota` enums, `init()` and global state, panic vs error. |
| `shp-patterns-structural` | Adapter, decorator/middleware, facade, proxy, composite, embedding. |
| `shp-patterns-behavioral` | Strategy, observer, command, chain of responsibility, state, template method, `iter.Seq` iterators, visitor. |
| `shp-patterns-concurrency` | errgroup, worker pools, pipelines, fan-out/fan-in, semaphores, timeouts, shutdown. |
| `shp-patterns-architecture` | Hexagonal/ports and adapters, service layer, repository, DI, unit of work, CQRS-lite, mapping to Kiro specs. |
| `shp-patterns-resilience` | Timeouts, retries with jitter, idempotency, circuit breaker, bulkhead, outbox, saga, dead-letter queues, partial batch failures. |

**AWS pack** (use only in projects on AWS):

| Skill | Use it when |
|---|---|
| `shp-aws-sdk` | Writing or explaining code that uses the AWS SDK for Go v2 (config, clients, paginators, waiters, errors, testing, v1 migration). |
| `shp-aws-lambda` | Building, packaging (including Windows) and deploying Go Lambda functions on `provided.al2023`; API Gateway and event-source contracts. |
| `shp-aws-resilience` | AWS SDK retry modes, conditional-write idempotency, SQS partial batch responses, DLQ redrive, outbox with DynamoDB Streams, Step Functions sagas. |
| `shp-aws-observability` | CloudWatch Logs and Insights, Embedded Metric Format, X-Ray/ADOT, `otelaws`, Lambda and SQS metrics. |

The process, design and quality skills (`shp-router`, `shp-tdd`, `shp-incremental-build`, `shp-debugging`, `shp-doubt-check`, `shp-simplify`, `shp-adr`, `shp-patterns-select`, `shp-api-design`, `shp-observability`) were added in v1.1.0. They are original texts inspired by MIT-licensed projects, credited in [`THIRD_PARTY_NOTICES.md`](THIRD_PARTY_NOTICES.md) and in each skill's `metadata.inspired_by`; the full list of changes is in [`docs/references.md`](docs/references.md#what-changed-in-v110).

Try it: *"Use shp-teach to explain why `internal/order/service.go` is written this way"* or select code and ask *"why is this idiomatic?"*.

`shp-go-verify` (added in v1.2.0) validates work and explanations against `go doc` and an allow-list of trusted Go sites before anything is called done.

**Go set vs provider packs.** Everything not named `shp-<provider>-*` is cloud-agnostic Go and belongs to `shp-go-expert`. Each cloud provider is its own pack, named `shp-<provider>-*` with its own agent (AWS: `shp-aws-sdk`, `shp-aws-lambda`, `shp-aws-resilience`, `shp-aws-observability`, steering `shp-aws-conventions`, agent `shp-aws-builder`). Kiro skills live in one flat folder, so packs are separated by name prefix. A future `shp-gcp-*` or `shp-azure-*` pack adds files without changing the Go set. `shp-go-mentor` and `shp-go-reviewer` are cloud-agnostic and load a pack's skills only when the code uses that provider.

### Agents

| Agent | Can edit? | Purpose |
|---|---|---|
| `shp-go-expert` | Yes | **Cloud-agnostic Go expert.** Builds test-first in thin slices, then validates its work against `go doc` and trusted sites (go.dev, pkg.go.dev, golang GitHub, Google/Uber style guides) with `shp-go-verify`. Loads no provider skills; destructive commands are denied. |
| `shp-go-mentor` | No | Teaches with `shp-teach`; validates every claim against trusted docs before stating it; may run `go doc`, `git log/blame` and use the web. |
| `shp-go-reviewer` | No | Reviews with `shp-go-review` (and `shp-doubt-check` on risky changes); validates each finding against trusted docs (unverifiable findings are marked as such); may run `go vet/test/build`, `gofmt -l`, linters, `govulncheck`, `gosec`, `git diff`. |
| `shp-aws-builder` | Yes | **AWS pack agent**: SDK v2, Lambda, event sources, IAM, IaC. Validates against `docs.aws.amazon.com`. Delegates general Go quality to the Go expert's rules. Go tooling allowed; `aws`, `sam`, `cdk`, `terraform`, `git commit/push` ask first; destructive commands denied. |

All agents deny reading credential files (`.env`, `*.pem`, `*.key`, `credentials*`, ...).

### Steering

| File | Inclusion | Scope |
|---|---|---|
| `product.md`, `tech.md`, `structure.md` | always | project only |
| `shp-security.md` | always | project and global |
| `shp-go-style.md` | `**/*.go`, `**/go.mod` | project and global |
| `shp-go-testing.md` | `**/*_test.go` | project and global |
| `shp-aws-conventions.md` (AWS pack) | auto (AWS-related requests) | project and global |

### Hooks (`.kiro/hooks/shp-go-quality.json`, Kiro hooks v1 format)

| Hook | Trigger | Default |
|---|---|---|
| `shp-gofmt-on-save` | `PostFileSave` on `*.go` | on |
| `shp-go-vet-on-stop` | `Stop` | on |
| `shp-go-test-on-stop` (`-short`) | `Stop` | on |
| `shp-golangci-lint-on-stop` | `Stop` | off (needs golangci-lint v2) |
| `shp-go-test-race-on-stop` | `Stop` | off (needs cgo / C toolchain) |
| `shp-govulncheck-on-stop` | `Stop` | off (needs govulncheck + network) |
| `shp-review-after-spec-task` | `PostTaskExec` (agent action) | off (uses extra credits) |

Turn hooks on or off by editing `"enabled"` in the JSON file.

### MCP servers (`.kiro/settings/mcp.json`)

All are **disabled by default** so nothing fails in a locked-down environment. Set `"disabled": false` for the ones you can run.

| Server | Requires | Notes |
|---|---|---|
| `gopls` | `go install golang.org/x/tools/gopls@latest` (v0.20+) | Official Go language server in MCP mode (`gopls mcp`) |

Provider servers are kept out of this file. For AWS, merge the entries of `.kiro/providers/aws/mcp.aws.json` into `mcpServers` (also all disabled by default):

| Server | Requires | Notes |
|---|---|---|
| `aws-mcp` | `uv`, AWS credentials | AWS MCP Server via `mcp-proxy-for-aws-cli`; change `AWS_REGION` to your Region |
| `aws-serverless` | `uv`, AWS profile | awslabs Serverless MCP server |
| `aws-documentation` | `uv` | Search AWS documentation |

## Naming convention

Every kit asset (skill, agent, generic steering file, hook) starts with **`shp-`** (short for shuppatsu). This guarantees that:

- no two kit skills share a name, and every skill's `name` matches its folder;
- kit skills never collide with popular complementary sets (`golang-*` from samber, `go-*` from cxuu, `aws-*` from the Agent Toolkit for AWS, `aidlc*`, and the unprefixed names used by Addy Osmani's, Superpowers' and Matt Pocock's sets such as `test-driven-development`), so Kiro's "workspace overrides global" rule never silently shadows one with the other.

Cloud-provider assets add the provider after the prefix: `shp-<provider>-<topic>` (for example `shp-aws-lambda`, a future `shp-gcp-functions`). Names in the Go set never contain a provider name. When you add a skill, keep the prefix, use lower-case words separated by hyphens (max 64 characters), make the folder name equal to `name:`, and check that the name is not already used. See [`docs/references.md`](docs/references.md#name-map-kit-vs-complementary-skills) for overlaps in purpose with external skills.

## Requirements

- Kiro IDE 1.x or Kiro CLI v3 (hooks use the v1 format with PascalCase triggers).
- Go toolchain. The kit targets Go 1.27; the `go` directive in your `go.mod` decides which features the agents may use.
- Optional: `gopls`, `golangci-lint` v2, `govulncheck`. AWS pack only: `uv` (AWS MCP servers) and AWS credentials.

## Rebuilding the zips after changes

PowerShell 7, from the repository root:

```powershell
Compress-Archive -Path .kiro\* -DestinationPath dist\shuppatsu-go-kiro-kit-project.zip -Force

$stage = Join-Path $env:TEMP 'shp-global'
Remove-Item $stage -Recurse -Force -ErrorAction SilentlyContinue
New-Item -ItemType Directory "$stage\steering" | Out-Null
Copy-Item .kiro\shp-kit.md, .kiro\skills, .kiro\agents $stage -Recurse
Copy-Item .kiro\steering\shp-*.md "$stage\steering"
Compress-Archive -Path "$stage\*" -DestinationPath dist\shuppatsu-go-kiro-kit-global.zip -Force
```

Bash, from `.kiro/`:

```bash
zip -r -X ../dist/shuppatsu-go-kiro-kit-project.zip shp-kit.md steering skills agents hooks settings
zip -r -X ../dist/shuppatsu-go-kiro-kit-global.zip shp-kit.md steering/shp-*.md skills agents
```

## References and complementary materials

[`docs/references.md`](docs/references.md) lists the research behind this kit: Kiro documentation (powers, skills, steering, hooks, custom agents, MCP), official AWS assets (Agent Toolkit for AWS, Kiro powers, awslabs MCP servers, AI-DLC), Go skill collections, the gopls MCP server, and the primary Go and AWS documentation used to write the skills.

## License

[MIT](LICENSE)
