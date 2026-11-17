---
name: shp-aws-sdk-go
description: How to use the AWS SDK for Go v2 correctly - loading config, client lifetime, context and timeouts, retries, paginators, waiters, error handling with smithy.APIError and modeled errors, DynamoDB attributevalue, S3 streaming bodies, and testing SDK code with consumer-side interfaces and fakes. Use when writing, reviewing or explaining Go code that imports github.com/aws/aws-sdk-go-v2, or when migrating from aws-sdk-go v1.
license: MIT
metadata:
  kit: shuppatsu-go
  version: "1.0.0"
---

# shp-aws-sdk-go

AWS SDK for Go **v1 is end-of-support**. All new code uses v2 modules: `github.com/aws/aws-sdk-go-v2/config`, `.../aws`, `.../service/<name>`.

## Config and clients

```go
cfg, err := config.LoadDefaultConfig(ctx) // env vars, shared config/profile, SSO, IAM role (Lambda/ECS/EC2)
if err != nil {
	return fmt.Errorf("load aws config: %w", err)
}
ddb := dynamodb.NewFromConfig(cfg)
s3c := s3.NewFromConfig(cfg, func(o *s3.Options) { o.UsePathStyle = true }) // per-client override
```

- Load config **once** (in `main` or Lambda init). Clients are safe for concurrent use: create once, reuse.
- Region and credentials come from the environment; never hard-code them. `config.WithRegion`, `config.WithSharedConfigProfile` exist for explicit cases.
- Per-call overrides: pass `func(o *dynamodb.Options)` as the last argument of an operation.

## Context, timeouts, retries

- Every operation takes `ctx` first; the SDK honours deadlines and cancellation.
- Default retryer: standard mode, exponential backoff with jitter. Tune with `config.WithRetryMaxAttempts(n)` and `config.WithRetryMode(aws.RetryModeAdaptive)`.
- Do not wrap SDK calls in another retry loop; you will multiply attempts (see `shp-patterns-resilience`).

## Pointers and helpers

Input fields are often pointers. Use `aws.String`, `aws.Int32`, `aws.Bool` to build inputs and `aws.ToString`, `aws.ToInt32` to read outputs safely (nil-safe).

## Paginators and waiters

```go
p := s3.NewListObjectsV2Paginator(s3c, &s3.ListObjectsV2Input{Bucket: aws.String(bucket)})
for p.HasMorePages() {
	page, err := p.NextPage(ctx)
	if err != nil {
		return fmt.Errorf("list objects: %w", err)
	}
	for _, obj := range page.Contents {
		_ = aws.ToString(obj.Key)
	}
}

w := dynamodb.NewTableExistsWaiter(ddb)
err = w.Wait(ctx, &dynamodb.DescribeTableInput{TableName: aws.String(table)}, 5*time.Minute)
```

## Errors

```go
out, err := ddb.GetItem(ctx, in)
if err != nil {
	var nf *types.ResourceNotFoundException // modeled error from service/dynamodb/types
	if errors.As(err, &nf) {
		return ErrTableMissing
	}
	var apiErr smithy.APIError // any service error: code, message, fault
	if errors.As(err, &apiErr) && apiErr.ErrorCode() == "ThrottlingException" {
		// usually already retried by the SDK; surface as retryable to the caller
	}
	return fmt.Errorf("get item: %w", err)
}
```

- `smithy.OperationError` gives service and operation names; `awshttp.ResponseError` (`github.com/aws/aws-sdk-go-v2/aws/transport/http`) gives `ServiceRequestID()` for support tickets.
- With Go 1.26+, `errors.AsType[*types.ResourceNotFoundException](err)` is the typed alternative to `errors.As`.

## DynamoDB items

```go
item, err := attributevalue.MarshalMap(order)     // struct -> map[string]types.AttributeValue
err = attributevalue.UnmarshalMap(out.Item, &order) // and back; use `dynamodbav:"name"` tags
```

Keep `dynamodbav` tags on adapter-level structs if the domain type should stay storage-agnostic.

## S3 bodies

`GetObjectOutput.Body` is an `io.ReadCloser`: always `defer out.Body.Close()`, even if you do not read it. For large uploads use the transfer manager (check the current package with `go doc` / pkg.go.dev before choosing between `feature/s3/manager` and its successor).

## Testing

```go
// Consumer-side interface with only the methods this package calls.
type itemGetter interface {
	GetItem(ctx context.Context, in *dynamodb.GetItemInput, optFns ...func(*dynamodb.Options)) (*dynamodb.GetItemOutput, error)
}

type fakeGetter struct{ out *dynamodb.GetItemOutput; err error }

func (f fakeGetter) GetItem(context.Context, *dynamodb.GetItemInput, ...func(*dynamodb.Options)) (*dynamodb.GetItemOutput, error) {
	return f.out, f.err
}
```

`*dynamodb.Client` satisfies `itemGetter` automatically. For integration tests, use LocalStack or a dedicated test account, never production.

## Migrating from v1

| v1 | v2 |
|---|---|
| `session.NewSession()` | `config.LoadDefaultConfig(ctx)` |
| `dynamodb.New(sess)` | `dynamodb.NewFromConfig(cfg)` |
| `GetItemWithContext(ctx, in)` | `GetItem(ctx, in)` |
| `awserr.Error` + `Code()` | `errors.As` with `smithy.APIError` or modeled types |
| `ListObjectsPages` | `NewListObjectsV2Paginator` |
| `dynamodbattribute` | `feature/dynamodb/attributevalue` |

## Sources

- Developer guide: https://docs.aws.amazon.com/sdk-for-go/v2/developer-guide/welcome.html
- Using clients, paginators, waiters: https://docs.aws.amazon.com/sdk-for-go/v2/developer-guide/using.html
- Error handling: https://docs.aws.amazon.com/sdk-for-go/v2/developer-guide/handle-errors.html
- Configuration and retries: https://docs.aws.amazon.com/sdk-for-go/v2/developer-guide/configure-gosdk.html
- API reference: https://pkg.go.dev/github.com/aws/aws-sdk-go-v2
