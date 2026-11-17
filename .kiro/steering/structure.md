---
inclusion: always
---

# Project structure

Follows the official guidance in "Organizing a Go module" (https://go.dev/doc/modules/layout). Start small; add directories only when they are needed.

```text
.
├── cmd/
│   └── <app>/main.go          # one directory per binary; main only wires things together
├── internal/
│   ├── <domain>/              # business logic, no AWS/HTTP types in signatures
│   ├── platform/<adapter>/    # adapters: aws (dynamodb, s3, sqs...), http, db
│   └── config/                # env/config loading
├── infra/                     # IaC (SAM template, CDK app or Terraform) if this repo owns infra
├── .kiro/
│   ├── specs/                 # requirements.md, design.md, tasks.md per feature
│   ├── steering/              # this file and friends
│   ├── skills/  agents/  hooks/  settings/
├── go.mod
└── README.md
```

## Rules

- Code lives in `internal/` unless it is intentionally consumed by other modules. Do not create `pkg/` by default.
- Package names are short, lower-case, singular nouns that describe what the package provides (`order`, not `orders` or `orderutils`). No `util`, `common`, `helpers`.
- Dependencies point inward: `cmd` → `internal/platform` → `internal/<domain>`. The domain never imports adapters.
- Interfaces are declared in the package that **uses** them, next to the consumer.
- One Lambda function = one `cmd/<function>/main.go` producing a `bootstrap` binary.
- Tests sit next to the code (`x_test.go`). Shared test helpers go in `internal/testutil` only if reused by 3+ packages.
