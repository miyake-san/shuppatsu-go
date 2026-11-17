---
inclusion: always
---

# Security baseline (shuppatsu-go kit)

- Never read, print or copy files that hold credentials: `.env`, `.env.*`, `*.pem`, `*.key`, `*.p12`, `*.pfx`, `secrets.*`, `credentials*`, `~/.aws/credentials`. Reference values through environment variables or a secrets manager instead.
- Never write secrets, tokens or passwords into code, tests, logs, specs, commit messages or chat output. Use placeholders such as `<YOUR_API_KEY>`.
- Validate all external input at the boundary (HTTP, events, files). Use parameterised SQL; never build queries with string concatenation.
- Use the standard library `crypto/*` packages; never set `InsecureSkipVerify: true` outside tests.
- Set timeouts on every network call (`http.Client{Timeout: ...}`, context deadlines).
- Keep dependencies minimal and pinned in `go.mod`/`go.sum`; run `govulncheck ./...` when network access allows.
- Destructive or irreversible commands (deleting cloud resources, force-pushing, `rm -rf`) need explicit user confirmation.
