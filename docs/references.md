# References and complementary materials

Research behind this kit (November 2026): Kiro capabilities, AWS assets for agents, and Go-specific skills, MCP servers and examples. **Nothing listed here is vendored into the kit.** The kit's own skills are original, self-contained and namespaced with `shp-`; these links are for going further when your environment allows installing extra assets.

> Third-party assets can run shell commands through hooks and MCP servers. Review them before installing, and prefer official sources in regulated environments.

## Key findings

- **No official Go power exists** in the Kiro Registry or in `kirodotdev/powers` at the time of writing. Go coverage comes from community skills, the official `gopls` MCP server and Kiro's built-in code intelligence (tree-sitter + gopls).
- **AWS coverage is official and broad:** Agent Toolkit for AWS (plugins `aws-core`, `aws-agents`, `aws-data-analytics`, `aws-agents-for-devsecops`), Kiro powers maintained by AWS, and the `awslabs` MCP servers.
- Kiro supports **Agent Skills** (`SKILL.md`, agentskills.io), **powers** following the **Agent Plugins** spec, **custom agents/subagents**, **steering** (including `AGENTS.md`) and **hooks v1** (`.kiro/hooks/*.json`, PascalCase triggers).

## 1. Kiro documentation

| Topic | Link |
|---|---|
| Powers (overview, create, install) | https://kiro.dev/docs/powers/ · https://kiro.dev/docs/powers/create/ · https://kiro.dev/docs/powers/installation/ |
| Powers catalogue and Registry (powers + MCP servers) | https://kiro.dev/powers/ · https://kiro.dev/registry/ |
| Agent Skills | https://kiro.dev/docs/skills/ |
| Steering (inclusion modes, `AGENTS.md`, file references) | https://kiro.dev/docs/steering/ |
| Hooks v1 (file format, triggers, examples, 0.x → 1.0 mapping) | https://kiro.dev/docs/hooks/ · https://kiro.dev/docs/hooks/types/ · https://kiro.dev/docs/hooks/examples/ · https://kiro.dev/docs/ide/whats-new-v1/hooks/ |
| Custom agents and subagents (IDE) | https://kiro.dev/docs/custom-agents/creating/ · https://kiro.dev/docs/custom-agents/configuration-reference/ · https://kiro.dev/docs/custom-agents/examples/ · https://kiro.dev/docs/custom-agents/subagents/ |
| Custom agents (CLI) | https://kiro.dev/docs/cli/custom-agents/creating/ · https://kiro.dev/docs/cli/custom-agents/configuration-reference |
| MCP configuration (`mcp.json`, precedence, env vars) | https://kiro.dev/docs/mcp/configuration/ |
| Code intelligence (Go via tree-sitter and gopls) | https://kiro.dev/docs/tools/code-intelligence/ |
| Specs | https://kiro.dev/docs/specs/ |
| Kiro CLI v3 | https://kiro.dev/docs/cli/v3/ · https://kiro.dev/blog/cli-3/ |
| Release posts: subagents + skills (0.9), powers support Agent Plugins | https://kiro.dev/blog/custom-subagents-skills-and-enterprise-controls/ · https://kiro.dev/blog/powers-supports-plugins/ |
| Open specs | Agent Skills: https://agentskills.io · Agent Plugins: https://agent-plugins.org |

## 2. AWS assets for agents (official)

| Asset | What it provides | Link |
|---|---|---|
| Agent Toolkit for AWS | AWS MCP Server + AWS skills + plugins (`aws-core` first); rules file `rules/aws-agent-rules.md` | https://github.com/aws/agent-toolkit-for-aws · https://docs.aws.amazon.com/agent-toolkit/latest/userguide/quick-start.html |
| `aws-core` plugin (also listed as a Kiro power) | MCP config + core skills: infra, serverless, containers, security, cost, SDK usage | https://github.com/aws/agent-toolkit-for-aws/tree/main/plugins/aws-core |
| Kiro powers by Kiro/AWS | `aws-sam`, `aws-infrastructure-as-code` (CDK/CloudFormation), `aws-observability`, `aws-step-functions`, `aws-mcp`, `terraform`, `localstack`, `saas-builder`, `cloud-architect` | https://github.com/kirodotdev/powers |
| Lambda powers (one-click install) | SAM, Lambda Managed Instances, Lambda Durable Functions | https://kiro.dev/launch/powers/add/?name=aws-sam · https://kiro.dev/launch/powers/add/?name=aws-lambda-managed-instances · https://kiro.dev/launch/powers/add/?name=aws-lambda-durable-functions |
| Lambda agent setup guide (has a Kiro section) | `aws-serverless` skill + Serverless MCP server | https://docs.aws.amazon.com/lambda/latest/dg/agent-setup-guide.html |
| `awslabs/agent-plugins` | `deploy-on-aws`, `aws-serverless`, `databases-on-aws`... (Kiro install is experimental; hooks are dropped on conversion; successor is the Agent Toolkit) | https://github.com/awslabs/agent-plugins |
| AI-DLC workflows | AWS AI-Driven Development Life Cycle: skills, 15 agents, steering and gate hooks for Kiro | https://github.com/awslabs/aidlc-workflows · https://awslabs.github.io/aidlc-workflows/guide/harnesses/kiro/ |
| Kiro SecOps baseline (AWS sample, Go CLI) | Guard hooks, permissions, security-review agent | https://github.com/aws-samples/sample-kiro-secops-baseline |
| Kiro Steering Studio (AWS sample) | Generates `product.md`, `tech.md`, `structure.md` by voice | https://github.com/aws-samples/sample-kiro-steering-studio |

### AWS MCP servers (awslabs)

| Server | Link |
|---|---|
| Installation guide (Kiro section) | https://awslabs.github.io/mcp/installation |
| AWS Serverless | https://awslabs.github.io/mcp/servers/aws-serverless-mcp-server |
| AWS IaC (CDK / CloudFormation) | https://awslabs.github.io/mcp/servers/aws-iac-mcp-server |
| Amazon ECS | https://awslabs.github.io/mcp/servers/ecs-mcp-server |
| Amazon EKS | https://awslabs.github.io/mcp/servers/eks-mcp-server |
| Amazon DynamoDB | https://awslabs.github.io/mcp/servers/dynamodb-mcp-server |
| AWS Documentation | https://awslabs.github.io/mcp/servers/aws-documentation-mcp-server |
| AWS Pricing | https://awslabs.github.io/mcp/servers/aws-pricing-mcp-server |
| Amazon CloudWatch | https://awslabs.github.io/mcp/servers/cloudwatch-mcp-server |

### Community AWS assets

| Asset | Link |
|---|---|
| `itsmostafa/aws-agent-skills` (18 skills: IAM, Lambda, DynamoDB, ECS, EKS, ...) | https://github.com/itsmostafa/aws-agent-skills |
| AWS Security Analyzer power | https://github.com/aquavis12/power-aws-security-analyzer |
| MiniStack local AWS emulator power | https://github.com/ministackorg/ministack-power |

## 3. Go assets

| Asset | What it provides | Link |
|---|---|---|
| `samber/cc-skills-golang` | 46 Go skills (testing, concurrency, errors, gRPC, lint, observability, performance, security, layout) | https://github.com/samber/cc-skills-golang |
| `cxuu/golang-skills` | 20 Go skills distilled from Google and Uber style guides | https://github.com/cxuu/golang-skills |
| `saisudhir14/golang-agent-skill` | Single Go best-practices skill | https://github.com/saisudhir14/golang-agent-skill |
| ECC `.kiro/` | Kiro agents (`go-reviewer`, `go-build-resolver`), skills (`golang-patterns`, `golang-testing`), steering, hooks | https://github.com/affaan-m/ECC/tree/main/.kiro |
| gopls MCP server (official) | `gopls mcp` (detached) or `gopls serve -mcp.listen=...` (attached) | https://go.dev/gopls/features/mcp |
| Go MCP SDK (official) | Build your own MCP servers in Go | https://github.com/modelcontextprotocol/go-sdk |
| Example: Go + Kiro setup | Steering and hooks for a Go service | https://www.santekno.com/en/aws-kiro-golang/ |

### Name map: kit vs. complementary skills

The kit uses the `shp-` prefix so none of its names collide with these sets. Some purposes overlap; if you install both, pick one per topic to avoid two skills competing for the same request.

| Source | Name pattern | Overlapping kit skill |
|---|---|---|
| `samber/cc-skills-golang` | `golang-*` (e.g. `golang-design-patterns`, `golang-concurrency`, `golang-testing`) | `shp-patterns-*`, `shp-go-review` |
| `cxuu/golang-skills` | `go-*` (e.g. `go-code-review`, `go-concurrency`, `go-functional-options`) | `shp-go-review`, `shp-patterns-concurrency`, `shp-patterns-creational` |
| ECC | `golang-patterns`, `golang-testing` | `shp-patterns-*` |
| Agent Toolkit for AWS | `aws-*` (e.g. `aws-serverless`, `aws-lambda-durable-functions`) | `shp-aws-lambda-go`, `shp-aws-sdk-go` |
| AI-DLC | `aidlc*` | none |

## 4. Primary documentation used to write the kit

| Topic | Link |
|---|---|
| Go 1.27 / 1.26 release notes | https://go.dev/doc/go1.27 · https://go.dev/doc/go1.26 |
| Effective Go · Code Review Comments · Test Comments | https://go.dev/doc/effective_go · https://go.dev/wiki/CodeReviewComments · https://go.dev/wiki/TestComments |
| Organizing a Go module | https://go.dev/doc/modules/layout |
| AWS SDK for Go v2: using clients, errors, configuration | https://docs.aws.amazon.com/sdk-for-go/v2/developer-guide/using.html · https://docs.aws.amazon.com/sdk-for-go/v2/developer-guide/handle-errors.html · https://docs.aws.amazon.com/sdk-for-go/v2/developer-guide/configure-gosdk.html |
| Lambda with Go (.zip packaging, provided.al2023, SAM) | https://docs.aws.amazon.com/lambda/latest/dg/golang-package.html |
| Amazon Builders' Library: retries/backoff/jitter, idempotency | https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/ · https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/ |
| AWS Prescriptive Guidance: cloud design patterns | https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/introduction.html |
| Design patterns with Go examples | https://refactoring.guru/design-patterns/go |

The full per-topic source map used by the `shp-teach` skill is in `.kiro/skills/shp-teach/references/go-sources.md`.
