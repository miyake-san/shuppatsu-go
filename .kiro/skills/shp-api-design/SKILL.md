---
name: shp-api-design
description: Cloud-agnostic contract-first design of Go package APIs, HTTP APIs and event schemas - small stable surfaces, error semantics, versioning, backward compatibility, event envelopes. Use when designing or changing an exported Go API, REST or JSON endpoint, a message or event schema, module boundary, or when asked whether a change is breaking.
license: MIT
metadata:
  kit: shuppatsu-go
  version: "1.3.0"
  inspired_by: "https://github.com/addyosmani/agent-skills (api-and-interface-design, MIT)"
---

# shp-api-design

An interface is a promise. Design the contract first, keep it small, and assume that anything observable will be depended on (**Hyrum's Law**).

## Workflow

1. **Write the contract before the code:** Go signatures and doc comments, or an OpenAPI/JSON Schema/event example. Include errors and edge cases.
2. **Review it as a caller:** write the calling code (or a test) first and see whether it reads naturally.
3. **Minimise the surface.** Unexport by default; put implementation detail in `internal/`. Every exported name is a long-term commitment.
4. **Implement, then check compatibility** (see below).

## Go package APIs

- Accept interfaces, return concrete types; define the interface in the consumer (`shp-patterns-architecture`).
- `context.Context` first for anything that blocks; no stored contexts.
- Options: functional options or a config struct with a useful zero value; adding a field or option is compatible, changing a signature is not.
- Return `error` with documented, testable conditions: sentinel errors or typed errors that callers match with `errors.Is/As`. Document which errors a function can return.
- Do not expose third-party types (a cloud SDK's structs, a database driver's types) in your public API unless wrapping them is worse; they tie you to that module's versioning.
- Doc comment on every exported symbol, starting with its name; add runnable `Example` functions for non-obvious use.
- Compatibility: adding a method to an exported interface is **breaking**; adding a function, a struct field (with keyed literals) or an option is not. Check with `apidiff` or `gorelease` (`golang.org/x/exp/cmd/apidiff`, `golang.org/x/exp/cmd/gorelease`) when available.
- Breaking change: bump the major version (`/v2` module path), or deprecate first with `// Deprecated:` and a migration note.

## HTTP and JSON APIs (`net/http`)

| Topic | Rule |
|---|---|
| Routing | `http.ServeMux` method and wildcard patterns (`GET /orders/{id}`, Go 1.22+) |
| Status codes | 400 invalid input, 401/403 authn/authz, 404 missing, 409 conflict or idempotency clash, 422 semantic validation, 429 throttled with `Retry-After`, 5xx server fault |
| Errors | One stable envelope, e.g. `{"error":{"code":"order_not_found","message":"...","requestId":"..."}}`; machine-readable `code`, never leak internals or stack traces |
| Validation | At the boundary; reject unknown fields with `json.Decoder.DisallowUnknownFields()` for strict APIs; limit body size with `http.MaxBytesReader` |
| Idempotency | `Idempotency-Key` header for non-idempotent POSTs (`shp-patterns-resilience`) |
| Pagination | Opaque cursor tokens over offsets for large or changing sets |
| Versioning | Additive changes only inside a version; breaking changes in `/v2` or a new media type; never change meaning of an existing field |
| Time and money | RFC 3339 UTC timestamps; money as integer minor units plus currency code |
| Timeouts | Server `ReadHeaderTimeout`, `WriteTimeout`; client timeouts on every call |

A reverse proxy or edge layer in front of the service adds its own timeout and throttling: keep the service's timeouts below the proxy's, and return `429` with `Retry-After` when throttled.

## Events and messages

- Envelope with `id`, `type`, `version`, `occurredAt`, `source`, and a `data` payload; consumers ignore unknown fields (tolerant reader).
- Delivery is usually at-least-once: consumers must be idempotent; carry a stable event ID.
- Evolve by adding optional fields; never repurpose or remove a field without a new `version`.
- Describe schemas with JSON Schema, AsyncAPI or protobuf and keep them in the repository next to the code.
- For queue consumers, report failures per message so one bad record does not redeliver the whole batch (`shp-patterns-resilience`).

## Anti-rationalization

| Excuse | Rebuttal |
|---|---|
| "Export it, someone might need it" | You cannot take it back. Export when a real caller needs it. |
| "It is only internal" | Internal consumers still break; put it under `internal/` or version it. |
| "Callers should not depend on that behaviour" | They will (Hyrum's Law). Document it or change it deliberately. |
| "We can change the response later" | Not without a version; clients parse what you send today. |

## Verification gate

- Contract (signatures or schema) reviewed before implementation.
- Tests cover documented errors and boundary values.
- Compatibility check run (`apidiff`/`gorelease`) or a manual note that the change is additive.
- `go vet ./...` and `go doc` render the public docs sensibly.

## Sources

- Go API design: https://go.dev/doc/modules/release-workflow , https://go.dev/blog/module-compatibility
- Go doc comments: https://go.dev/doc/comment
- Effective Go and Code Review Comments (interfaces, naming): https://go.dev/doc/effective_go , https://go.dev/wiki/CodeReviewComments
- Routing enhancements (Go 1.22): https://go.dev/blog/routing-enhancements
- Hyrum's Law: https://www.hyrumslaw.com/
- OpenAPI: https://spec.openapis.org/oas/latest.html ; AsyncAPI: https://www.asyncapi.com/docs
- Tolerant Reader: https://martinfowler.com/bliki/TolerantReader.html
- Idempotent consumers: https://microservices.io/patterns/communication-style/idempotent-consumer.html

## Inspired by

Contract-first approach, Hyrum's Law and minimal-surface rules inspired by `api-and-interface-design` in addyosmani/agent-skills (MIT). Go, HTTP and event guidance is original. See `THIRD_PARTY_NOTICES.md`.

Provider-specific details live in the cloud-provider skill packs (for example the `shp-aws-*` skills).
