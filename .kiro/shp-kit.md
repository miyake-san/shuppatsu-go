# shuppatsu-go Kiro kit (v1.3.0)

Cloud-agnostic Go expert kit for Kiro, with cloud-provider packs. Source: https://github.com/miyake-san/shuppatsu-go

Every kit asset is prefixed with `shp-` so it never overwrites, shadows or gets shadowed by skills, agents or steering from other sources. The only exceptions are the project-level files `steering/product.md`, `steering/tech.md`, `steering/structure.md` and `settings/mcp.json`, which ship only in the project zip.

## Structure: Go set plus provider packs

| Part | Names | Cloud-specific? |
|---|---|---|
| **Go set** | `shp-go-*`, `shp-patterns-*`, `shp-router`, `shp-tdd`, `shp-incremental-build`, `shp-debugging`, `shp-doubt-check`, `shp-simplify`, `shp-adr`, `shp-api-design`, `shp-observability`, `shp-teach`, agents `shp-go-expert`, `shp-go-mentor`, `shp-go-reviewer` | No. Provider-neutral wording and examples |
| **AWS pack** | `shp-aws-sdk`, `shp-aws-lambda`, `shp-aws-resilience`, `shp-aws-observability`, steering `shp-aws-conventions`, agent `shp-aws-builder`, MCP servers in `providers/aws/mcp.aws.json` | Yes |
| **Future packs** | `shp-gcp-*`, `shp-azure-*`, ... | One pack per provider; adding one never changes the Go set |

Kiro skills live in one flat folder, so packs are separated by name prefix and by agent, not by directory.

## Which zip goes where

| Zip | Unzip into | Contains |
|---|---|---|
| `shuppatsu-go-kiro-kit-project.zip` | `<your-project>\.kiro\` | everything: steering, skills, agents, hooks, settings/mcp.json, providers/ |
| `shuppatsu-go-kiro-kit-global.zip` | `%USERPROFILE%\.kiro\` (`~/.kiro/`) | `shp-*` skills, agents and generic steering only; no hooks, no mcp.json |

PowerShell:

```powershell
Expand-Archive .\shuppatsu-go-kiro-kit-project.zip -DestinationPath .\.kiro
Expand-Archive .\shuppatsu-go-kiro-kit-global.zip  -DestinationPath $HOME\.kiro
```

Add `-Force` only if you want to overwrite files with the same names. Do not install the AWS pack in projects that do not use AWS: delete its `shp-aws-*` skills, `shp-aws-builder` agent and `shp-aws-conventions` steering.

## Contents

- **Skills (23):**
  - Entry point: `shp-router`
  - Documentation validation: `shp-go-verify`
  - Process: `shp-tdd`, `shp-incremental-build`, `shp-debugging`, `shp-doubt-check`, `shp-simplify`, `shp-adr`
  - Design: `shp-patterns-select`, `shp-api-design`, `shp-observability`
  - Patterns: `shp-patterns-creational`, `shp-patterns-structural`, `shp-patterns-behavioral`, `shp-patterns-concurrency`, `shp-patterns-architecture`, `shp-patterns-resilience`
  - Review and teaching: `shp-go-review`, `shp-teach`
  - AWS pack: `shp-aws-sdk`, `shp-aws-lambda`, `shp-aws-resilience`, `shp-aws-observability`
- **Agents:**
  - Go expert: `shp-go-expert` (cloud-agnostic; validates against go.dev and trusted sites)
  - Read-only, shared: `shp-go-mentor`, `shp-go-reviewer` (validate claims against trusted docs before answering)
  - AWS pack: `shp-aws-builder` (validates against docs.aws.amazon.com)
- **Steering:** `shp-go-style`, `shp-go-testing`, `shp-security` (+ project-only `product`, `tech`, `structure`); AWS pack: `shp-aws-conventions`
- **Hooks (project only):** `hooks/shp-go-quality.json`
- **MCP (project only):** `settings/mcp.json` with gopls (disabled by default); AWS servers in `providers/aws/mcp.aws.json` to merge when needed

Third-party inspirations (MIT) are credited in `THIRD_PARTY_NOTICES.md` at the repository root and in each skill's `metadata.inspired_by`.

Nothing in this kit needs `npx`, `uvx` or internet access to work. Optional tools (gopls, golangci-lint, govulncheck, uv) only unlock extra hooks and MCP servers.
