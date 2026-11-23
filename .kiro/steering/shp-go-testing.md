---
inclusion: fileMatch
fileMatchPattern: "**/*_test.go"
---

# Go testing (shuppatsu-go kit)

Sources: Go Test Comments wiki, Table Driven Tests wiki, `testing` package docs.

## Shape of a test

- Table-driven tests with named cases and `t.Run(tc.name, ...)`. Name fields `name`, `give`/`input`, `want`.
- Failure messages state input, got and want: `t.Errorf("Parse(%q) = %v, want %v", in, got, want)`.
- Use `t.Fatal` only when the test cannot continue; otherwise `t.Error` to report all failures.
- Mark helpers with `t.Helper()`; release resources with `t.Cleanup`.
- Use `t.Context()` (Go 1.24+) instead of `context.Background()` in tests.
- Call `t.Parallel()` for independent tests; never share mutable fixtures between parallel cases.

## Test doubles

- Depend on small consumer-side interfaces so tests can pass hand-written fakes. Prefer fakes and stubs over mocking frameworks.
- For any cloud or vendor SDK, wrap only the client methods you call in an interface (for example `Get`, `Put`) and fake that interface.
- For HTTP, use `net/http/httptest` (`httptest.NewServer`, `httptest.NewRecorder`).

## Time and concurrency

- Never `time.Sleep` to wait for goroutines. Use channels, `sync.WaitGroup`, or `testing/synctest` (Go 1.25+) for code that uses timers.
- Run `go test -race ./...` in CI (Linux). It needs cgo, which may be unavailable on Windows workstations.

## Other techniques

- Fuzz parsers and decoders: `func FuzzX(f *testing.F)`.
- Benchmarks use `for b.Loop() { ... }` (Go 1.24+).
- Golden files live in `testdata/`; update them behind a `-update` flag.
- `-short` must skip slow tests (`if testing.Short() { t.Skip(...) }`): the Stop hook runs `go test -short ./...`.
