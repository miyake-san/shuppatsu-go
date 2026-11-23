---
name: shp-aws-lambda
description: Build, package, test and deploy AWS Lambda functions in Go - provided.al2023 runtime, bootstrap binary, arm64, lambda.norpc, handler signatures and event types from aws-lambda-go, init vs handler work, structured logging, idempotency, SQS partial batch failures, and SAM, CDK or CLI deployment including Windows builds. Use when creating or changing a Go Lambda, its build scripts or its IaC.
license: MIT
metadata:
  kit: shuppatsu-go
  version: "1.0.0"
---

# shp-aws-lambda

The managed `go1.x` runtime is deprecated. Go functions run on an **OS-only runtime**: `provided.al2023`, with an executable named `bootstrap`.

## Function skeleton

```go
package main

import (
	"context"
	"log/slog"
	"os"

	"github.com/aws/aws-lambda-go/events"
	"github.com/aws/aws-lambda-go/lambda"
	"github.com/aws/aws-sdk-go-v2/config"
	"github.com/aws/aws-sdk-go-v2/service/dynamodb"
)

type handler struct {
	log   *slog.Logger
	db    *dynamodb.Client
	table string
}

func (h *handler) handle(ctx context.Context, ev events.APIGatewayV2HTTPRequest) (events.APIGatewayV2HTTPResponse, error) {
	// thin: parse input, call internal/<domain>, map the result to a response
	return events.APIGatewayV2HTTPResponse{StatusCode: 200, Body: `{"ok":true}`}, nil
}

func main() {
	// Init phase: runs once per execution environment and is reused by warm invocations.
	ctx := context.Background()
	cfg, err := config.LoadDefaultConfig(ctx)
	if err != nil {
		slog.Error("load aws config", "err", err)
		os.Exit(1)
	}
	h := &handler{
		log:   slog.New(slog.NewJSONHandler(os.Stdout, nil)),
		db:    dynamodb.NewFromConfig(cfg),
		table: os.Getenv("TABLE_NAME"),
	}
	lambda.Start(h.handle)
}
```

- Valid handler signatures take an optional `context.Context` and one input, and return an optional output plus `error`.
- Event structs live in `github.com/aws/aws-lambda-go/events` (`SQSEvent`, `APIGatewayV2HTTPRequest`, `EventBridgeEvent`, `S3Event`, ...).
- The invocation `ctx` carries the function deadline; pass it to every SDK call.
- `lambdacontext.FromContext(ctx)` exposes the request ID for logging.

## Build and package

Linux/macOS:

```bash
GOOS=linux GOARCH=arm64 CGO_ENABLED=0 go build -tags lambda.norpc -o bootstrap ./cmd/<function>
zip function.zip bootstrap
```

Windows PowerShell (zip with `build-lambda-zip` so `bootstrap` keeps executable permissions):

```powershell
go install github.com/aws/aws-lambda-go/cmd/build-lambda-zip@latest
$env:GOOS = "linux"; $env:GOARCH = "arm64"; $env:CGO_ENABLED = "0"
go build -tags lambda.norpc -o bootstrap ./cmd/<function>
~\Go\Bin\build-lambda-zip.exe -o function.zip bootstrap
```

- `bootstrap` must be at the root of the zip.
- `-tags lambda.norpc` drops the RPC component only needed by the deprecated `go1.x` runtime.
- Prefer `arm64` (Graviton) unless a dependency requires `x86_64`.

## AWS SAM

```yaml
Resources:
  OrdersFunction:
    Type: AWS::Serverless::Function
    Metadata:
      BuildMethod: go1.x          # SAM's Go builder; still correct with provided.al2023
    Properties:
      CodeUri: cmd/orders/
      Handler: bootstrap
      Runtime: provided.al2023
      Architectures: [arm64]
      Environment:
        Variables:
          TABLE_NAME: !Ref OrdersTable
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref OrdersTable
```

`sam build` then `sam deploy --guided`. Use SAM policy templates or explicit least-privilege statements.

## AWS CDK

Use the Go Lambda construct (`GoFunction`, currently in the alpha module for each CDK language) or build the `bootstrap` yourself and use `lambda.Function` with `Code.fromAsset` and `Runtime.PROVIDED_AL2023`. Confirm the construct's current module name in the CDK API reference before adding it.

## Patterns that matter in Lambda

- **Idempotency:** events are delivered at least once. Use an idempotency key with a DynamoDB conditional write (`shp-patterns-resilience`).
- **SQS:** return `events.SQSEventResponse{BatchItemFailures: ...}` and enable `ReportBatchItemFailures`; configure a DLQ.
- **Timeouts:** the function timeout must exceed the sum of downstream timeouts; set SDK and HTTP timeouts below it.
- **Logging:** JSON `slog` to stdout; include the request ID; never log secrets or full payloads with personal data.
- **Config:** environment variables for non-secret config; Secrets Manager or SSM Parameter Store for secrets, fetched at init and cached.
- **Cold start:** keep dependencies small, avoid reflection-heavy init, and reuse clients across invocations.

## Testing

- Unit-test the domain and the handler struct with fakes; call `h.handle(t.Context(), event)` directly.
- Local invocation: `sam local invoke -e event.json` (needs Docker), or LocalStack.

## API Gateway and event sources

Contracts for HTTP APIs and events follow `shp-api-design`; this section adds the AWS specifics.

API Gateway (REST and HTTP APIs):

| Topic | Rule |
|---|---|
| Timeouts | The integration has a hard timeout; keep the function timeout and downstream timeouts below it |
| Status mapping | 400 invalid input, 401/403 authn/authz, 404 missing, 409 conflict or idempotency clash, 429 throttled with `Retry-After`, 5xx server fault |
| Error envelope | One stable envelope, e.g. `{"error":{"code":"order_not_found","message":"...","requestId":"..."}}`; use the Lambda request ID as `requestId` |
| Throttling | Usage plans and stage throttling; clients must back off on 429 |
| Routing | Prefer HTTP APIs with payload format 2.0 and typed `events.APIGatewayV2HTTPRequest` structs |

Event sources (SQS, SNS, EventBridge):

- Use the generic envelope from `shp-api-design` (`id`, `type`, `version`, `occurredAt`, `source`, `data`); consumers ignore unknown fields.
- Delivery is at-least-once: carry a stable event ID and make handlers idempotent (`shp-aws-resilience`).
- Handlers use the typed structs in `github.com/aws/aws-lambda-go/events` and, for SQS, return partial batch failures (`events.SQSEventResponse`).
- Evolve schemas by adding optional fields; the EventBridge schema registry can document them.
- Source: https://docs.aws.amazon.com/apigateway/latest/developerguide/welcome.html

## Related kit skills and external powers

- `shp-aws-sdk` for SDK usage, `shp-aws-resilience` for AWS retries, idempotency and batch failures, `shp-patterns-resilience` for the generic patterns.
- When available in your environment, the Kiro powers `aws-sam`, `aws-lambda-durable-functions` and `aws-lambda-managed-instances`, and the AWS Serverless MCP server, add live tooling (see `docs/references.md`).

## Sources

- Building Lambda functions with Go: https://docs.aws.amazon.com/lambda/latest/dg/lambda-golang.html
- Deploy Go Lambda functions with .zip archives: https://docs.aws.amazon.com/lambda/latest/dg/golang-package.html
- `aws-lambda-go`: https://github.com/aws/aws-lambda-go
- SQS partial batch responses: https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-errorhandling.html
