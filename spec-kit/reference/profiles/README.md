# Stack profiles

Per-stack reference cards consumed by the test-layer agents — **`build-plan-architect`** (for the
unit/integration test-file naming infix, runner, and the gotchas to encode as `tddCases`) and
**`acceptance-spec-architect`** (for the E2E runner and spec naming infix). Each card is one stack: a
language/runtime + its test tooling.

## What a profile provides

- **Runner & commands** — the test runner, the whole-suite command, and the **single-test** command
  (so a ticket can say exactly how to run its one test).
- **Layers** — which of unit / integration / component / e2e apply to this stack, and what each holds.
- **File naming & location** — the convention that routes a test to the correct runner/environment.
  This is load-bearing: a misnamed test silently runs in the wrong environment.
- **Mocking seam & determinism** — where to mock, how to inject the clock, how to reset state.
- **Gotchas to encode as regression tests** — stack-specific failure modes for the
  `unit-integration-standards.md` catalog.

## Using a profile

The agents detect the stack(s) from manifest/lockfiles and source layout, then load the matching
`<stack>.md`. If no profile matches, infer these facts from the codebase and proceed (note the gap).
When a project's own conventions conflict with a profile, the project wins.

## Current profiles

| Stack key | Covers |
|---|---|
| `python` | Python services/libraries (pytest) |
| `js-frontend` | TypeScript/JavaScript browser apps — component + E2E (Vitest + jsdom, Playwright) |
| `js-node` | TypeScript/JavaScript Node backends — unit + integration, black-box HTTP e2e (Vitest, node env) |
| `go` | Go services/CLIs/libraries (`go test`, table-driven, `-race`) |

`js-frontend` and `js-node` are split deliberately: the browser layer (DOM, component tests, Playwright)
and the Node backend layer (no DOM, API/handler integration, black-box HTTP) have genuinely different
runners, environments, and gotchas. A repo with both is polyglot across these two keys.

Add a stack by dropping a `<stack>.md` here following the others. Nothing else changes.
