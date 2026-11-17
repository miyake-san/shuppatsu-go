---
inclusion: always
---

# Tech stack

## Language and toolchain

- **Go.** The `go` directive in `go.mod` is the source of truth for which language features and standard library APIs are allowed. This template targets **Go 1.27**; do not use features newer than the version declared in `go.mod`.
- Format with `gofmt` (enforced by a hook). Vet with `go vet ./...`.
- Run `go fix ./...` after a toolchain upgrade to apply modernizers, then review the diff.
- Prefer the standard library. Add a third-party module only when it removes significant code or risk, and say why in the PR/spec.

## Approved building blocks

| Concern | Default choice |
|---|---|
| Logging | `log/slog` (JSON handler in production) |
| HTTP | `net/http` (Go 1.22+ routing patterns on `http.ServeMux`) |
| JSON | `encoding/json` (backed by `encoding/json/v2` since Go 1.27) |
| Testing | `testing`, `testing/synctest`, `net/http/httptest`, fuzzing |
| AWS | **AWS SDK for Go v2** only (`github.com/aws/aws-sdk-go-v2/...`). v1 is end-of-support. |
| Lambda | `github.com/aws/aws-lambda-go`, runtime `provided.al2023`, binary named `bootstrap`, `arm64` by default |
| IaC | _choose one and delete the others:_ AWS SAM / AWS CDK / Terraform |

## Commands

```text
go build ./...
go vet ./...
go test ./...
go test -race ./...        # needs cgo; run in CI on Linux if not available locally
golangci-lint run          # optional, if installed
govulncheck ./...          # optional, needs network access to the vuln DB
```

## Tooling the agent can rely on

- `go doc <pkg>[.<symbol>]` works offline and is the first place to confirm an API.
- `gopls` (if installed) powers code intelligence and the optional `gopls` MCP server.
