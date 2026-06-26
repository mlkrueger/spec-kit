# Profile: Go (`go test`)

Default test stack for Go services, CLIs, and libraries. Use the standard toolchain; reach for `testify` only if the repo already does.

## Runner & commands

- **Runner:** `go test` with the standard `testing` package; table-driven tests are idiomatic.
- **Whole suite:** `go test ./...`
- **One package:** `go test ./pkg/foo`
- **One test:** `go test ./pkg/foo -run TestName` (subtests: `-run TestName/case`)
- **Race detector:** `go test -race ./...` — run in CI; Go's concurrency makes this load-bearing.
- **Coverage:** `go test -cover ./...` (profile: `-coverprofile=cover.out`)

## Layers

- **Unit** — pure functions, business logic, parsers, validation. Table-driven (`tests := []struct{...}` with `t.Run(tc.name, ...)`).
- **Integration** — handlers and services with dependencies behind interfaces, substituting fakes; HTTP handlers via `net/http/httptest`. Real DB only behind a build tag / `testing.Short()` gate.
- **Component** — N/A (no UI layer).
- **End-to-end** — for a service: `httptest.Server` or black-box HTTP against a built binary with mocked externals; for a CLI: build and exec the binary, assert stdout/exit code, or test the command layer directly.

## File naming & location

- Tests live beside source as `*_test.go` in the same package (or `package foo_test` for black-box API tests).
- Name tests `TestXxx`; use subtests for cases. Put shared helpers in `export_test.go` or a `testdata/`-backed helper; fixtures go in `testdata/`.
- Gate slow/external tests with `//go:build integration` tags or `if testing.Short() { t.Skip() }`.

## Mocking seam & determinism

- **Depend on interfaces**, inject fakes — Go has no monkeypatching; design the seam in the code. Recommend extracting an interface when logic can't otherwise be faked, and file it as a prerequisite ticket.
- **Inject a clock** — pass a `Clock` interface or a `now func() time.Time`; never call `time.Now()` in logic under test. Lock `TZ=UTC` in CI; add DST/boundary cases.
- Use `t.Cleanup()` for teardown; `t.Parallel()` only when tests share no mutable state.

## Gotchas to encode as regression tests

- **Goroutine/race bugs** — anything concurrent must be covered under `-race`; test cancellation via `context.Context` deadlines.
- **`context` propagation** — cancellation/timeout must thread through and be honored; test the cancelled path.
- **`error` handling** — wrapped errors (`errors.Is`/`errors.As`); the error branch must be tested, and authz must fail-closed on dependency error.
- **`nil` map/slice/pointer** — writing to a nil map panics; nil-slice vs empty-slice JSON differences.
- **`defer` evaluation** — arguments evaluated at `defer` time, not at return; loop-variable capture in closures (pre-1.22 semantics).
- **JSON (de)serialization** — unexported fields dropped; `omitempty` surprises; numeric types decoding into `interface{}` as `float64`.
- **Time** — `time.Time` monotonic-clock comparisons; truncation/rounding; timezone in formatting.
