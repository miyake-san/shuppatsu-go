---
inclusion: auto
name: shp-aws-go-conventions
description: AWS conventions for Go code - AWS SDK for Go v2, Lambda handlers, IAM, configuration, retries and observability. Use when code imports aws-sdk-go-v2 or aws-lambda-go, or when writing IaC (SAM, CDK, Terraform) for a Go service.
---

# AWS + Go conventions (shuppatsu-go kit)

Details and examples live in the skills `shp-aws-sdk-go` and `shp-aws-lambda-go`.

## SDK

- Use AWS SDK for Go v2 only. Load config once with `config.LoadDefaultConfig(ctx)` and build clients with `<service>.NewFromConfig(cfg)`. Clients are safe for concurrent use: create once, reuse.
- Pass the caller's `context.Context` to every operation; set deadlines at the edge (handler, request).
- Never hard-code credentials, account IDs or Regions. Credentials come from the default chain (env, profile, IAM role). Region comes from config/env.
- Inspect errors with `errors.As` against `smithy.APIError` or modeled types from the service `types` package.
- Always `Close()` streaming bodies (`GetObjectOutput.Body`).
- Use paginators (`New<Op>Paginator`) and waiters (`New<Resource><State>Waiter`) instead of hand-written loops.

## Lambda

- Runtime `provided.al2023`, executable named `bootstrap`, `arm64` unless a dependency needs x86_64. Build with `-tags lambda.norpc` and `CGO_ENABLED=0`.
- Initialise SDK clients and config outside the handler (cold start), keep the handler thin, and call into `internal/<domain>`.
- Handlers must be idempotent: events can be delivered more than once.

## Security and operations

- Least-privilege IAM per function/service; no `*` actions or resources unless justified in the design doc.
- Secrets in AWS Secrets Manager or SSM Parameter Store (SecureString), fetched at init and cached.
- Structured JSON logs via `log/slog`; include request IDs; never log secrets or personal data.
- Infrastructure changes go through IaC in `infra/`, never manual console changes.
- Destructive commands (`delete-*`, `rm`, `destroy`) require explicit user approval.
