# shuppatsu-go Kiro kit (v1.0.0)

Go + AWS starter kit for Kiro. Source: https://github.com/miyake-san/shuppatsu-go

Every kit asset is prefixed with `shp-` so it never overwrites, shadows or gets shadowed by skills, agents or steering from other sources. The only exceptions are the project-level files `steering/product.md`, `steering/tech.md`, `steering/structure.md` and `settings/mcp.json`, which ship only in the project zip.

## Which zip goes where

| Zip | Unzip into | Contains |
|---|---|---|
| `shuppatsu-go-kiro-kit-project.zip` | `<your-project>\.kiro\` | everything: steering, skills, agents, hooks, settings/mcp.json |
| `shuppatsu-go-kiro-kit-global.zip` | `%USERPROFILE%\.kiro\` (`~/.kiro/`) | `shp-*` skills, agents and generic steering only; no hooks, no mcp.json |

PowerShell:

```powershell
Expand-Archive .\shuppatsu-go-kiro-kit-project.zip -DestinationPath .\.kiro
Expand-Archive .\shuppatsu-go-kiro-kit-global.zip  -DestinationPath $HOME\.kiro
```

Add `-Force` only if you want to overwrite files with the same names.

## Contents

- **Skills:** `shp-teach`, `shp-go-review`, `shp-aws-sdk-go`, `shp-aws-lambda-go`, `shp-patterns-creational`, `shp-patterns-structural`, `shp-patterns-behavioral`, `shp-patterns-concurrency`, `shp-patterns-architecture`, `shp-patterns-resilience`
- **Agents:** `shp-go-mentor`, `shp-go-reviewer`, `shp-aws-go-builder`
- **Steering:** `shp-go-style`, `shp-go-testing`, `shp-aws-go-conventions`, `shp-security` (+ project-only `product`, `tech`, `structure`)
- **Hooks (project only):** `hooks/shp-go-quality.json`
- **MCP (project only):** `settings/mcp.json` with gopls and AWS servers, all disabled by default

Nothing in this kit needs `npx`, `uvx` or internet access to work. Optional tools (gopls, golangci-lint, govulncheck, uv) only unlock extra hooks and MCP servers.
