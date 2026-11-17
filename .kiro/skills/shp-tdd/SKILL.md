---
name: shp-tdd
description: Test-driven development loop for Go - red, green, refactor with table-driven tests, subtests, fakes at consumer-defined interfaces, fuzz and benchmark hooks, and a verification gate based on go test. Use when implementing any new behaviour, fixing a bug (write the failing test first), changing existing logic, or when asked to add tests to Go code.
license: MIT
metadata:
  kit: shuppatsu-go
  version: "1.1.0"
  inspired_by: "https://github.com/addyosmani/agent-skills (test-driven-development, MIT); https://quii.gitbook.io/learn-go-with-tests"
---

# shp-tdd

Write the test that describes the behaviour before the code that provides it. In Go the loop is fast (`go test` compiles and runs in seconds), so there is no excuse to skip it.

## The loop

1. **Red.** Write one failing test for the next smallest behaviour. Run it and **read the failure**: it must fail for the expected reason (assertion mismatch), not because of a compile error or a typo.
2. **Green.** Write the least code that passes. Hard-coding the answer is allowed; the next test will force generalisation.
3. **Refactor.** With the bar green, remove duplication and improve names. Run `go test ./...` after every edit. Never refactor on red.
4. Commit (see `shp-incremental-build`), then pick the next behaviour.

Bug fixes: first write a test that reproduces the bug and fails (**prove-it pattern**), then fix, then keep the test as the regression guard.

## Go test shapes

| Need | Use |
|---|---|
| Many input/output cases | Table-driven test with `t.Run(tc.name, ...)` and named fields |
| Parallel-safe cases | `t.Parallel()` in the test and in subtests (Go 1.22+ loop variables are per-iteration) |
| Context in a test | `t.Context()` (Go 1.24+), cancelled when the test ends |
| Temp files, cleanup | `t.TempDir()`, `t.Cleanup()`, `t.Setenv()` |
| Time-dependent or concurrent code | `testing/synctest` (Go 1.25+) instead of `time.Sleep` |
| Dependencies (DynamoDB, S3, HTTP) | A small interface declared in the consumer package plus a hand-written fake; see `shp-patterns-architecture` and `shp-aws-sdk-go` |
| Parsers, encoders, validators | `go test -fuzz` with a seed corpus in `testdata/fuzz` |
| Hot paths | `b.Loop()` benchmark (Go 1.24+) and `benchstat`; measure before optimising |

```go
func TestParseAmount(t *testing.T) {
	tests := []struct {
		name    string
		in      string
		want    int64
		wantErr bool
	}{
		{name: "whole", in: "10", want: 1000},
		{name: "cents", in: "10.05", want: 1005},
		{name: "empty", in: "", wantErr: true},
	}
	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) {
			t.Parallel()
			got, err := ParseAmount(tc.in)
			if (err != nil) != tc.wantErr {
				t.Fatalf("ParseAmount(%q) error = %v, wantErr %v", tc.in, err, tc.wantErr)
			}
			if got != tc.want {
				t.Errorf("ParseAmount(%q) = %d, want %d", tc.in, got, tc.want)
			}
		})
	}
}
```

Failure messages follow Go Test Comments: include the function, the input, the got and the want, in that order.

## Test quality rules

- **Test behaviour through the public API**, not private helpers; internals can then be refactored freely.
- **DAMP over DRY** in tests: a little repetition that keeps each case readable beats a clever shared helper.
- **Prefer fakes over mocks.** Assert on outcomes (state returned, records written), not on the exact sequence of calls.
- **Test pyramid:** many fast unit tests, fewer integration tests (build tag or `-run Integration`, local emulators), very few end-to-end checks.
- One reason to fail per test. Use `t.Fatalf` when later lines would panic, `t.Errorf` otherwise.
- Never delete or skip a failing test to get green (`t.Skip`, commented-out cases). Fix the code or the test, and say which.

## Anti-rationalization

| Excuse | Rebuttal |
|---|---|
| "I'll add tests after it works" | Tests written after only confirm what the code already does. Write the failing one first. |
| "It is too simple to test" | Simple code is cheap to test, and the test documents the intent. |
| "Mocks make this hard" | A fake behind a consumer-side interface is a few lines. If it is hard, the design is too coupled; see `shp-patterns-architecture`. |
| "The test is flaky, rerun it" | Flakiness is a bug (shared state, time, ordering). Use `-race`, `-shuffle=on`, `synctest`. |
| "Coverage is 80%, done" | Coverage says what ran, not what was asserted. Check that a mutation of the code would fail a test. |

## Verification gate (evidence required)

Do not say "done" until you can show:

- the new test failed first, for the right reason (quote the failing line);
- `go test ./...` passes (or `go test -short ./...` when integration tests are tagged);
- `go vet ./...` is clean;
- `go test -race ./...` passes for code that touches goroutines (needs cgo; say so if unavailable);
- no test was deleted, skipped or weakened to get there.

## Sources

- Learn Go with Tests (TDD in Go): https://quii.gitbook.io/learn-go-with-tests
- Go Test Comments: https://go.dev/wiki/TestComments
- `testing` package, fuzzing and `synctest`: `go doc testing`, https://go.dev/doc/security/fuzz/ , `go doc testing/synctest`
- Table-driven tests: https://go.dev/wiki/TableDrivenTests
- Software Engineering at Google, ch. 11-14 (testing, test pyramid, DAMP): https://abseil.io/resources/swe-book

## Inspired by

Process (red-green-refactor, prove-it pattern, test pyramid, anti-rationalization table, verification gate) inspired by `test-driven-development` in addyosmani/agent-skills (MIT) and by Learn Go with Tests. Text and examples are original and Go-specific. See `THIRD_PARTY_NOTICES.md`.
