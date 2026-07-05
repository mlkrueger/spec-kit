# CI/CD Standards (Reference)

> Reference document for the **ci-architect** agent (both its design and audit modes), and the
> source of the CI discipline the technical spec and build plan carry. Covers the pipeline: gates,
> jobs, environments, promotion, secrets, deploy/rollback. The *content* of the test suites the
> pipeline runs is owned by the testing standards; this document owns **whether and how those suites
> actually gate anything**. When a project's own conventions conflict with these, the project wins,
> but call out the divergence.

## The one discipline everything else serves

**The pipeline is the executable form of the done-gate.** Every quality bar the specs define — the
build plan's inner suite, the acceptance plan's journeys, the coverage floor, lint/type-check, the
constraint checks, the brownfield regression clauses — either runs in CI on every merge or it does
not exist. A gate that runs "usually," "locally," or "before release" is an aspiration. Conversely,
CI enforces bars the specs actually set — a pipeline step tracing to no spec'd requirement is
ceremony; justify it or cut it.

## The two-job shape (the load-bearing structure)

Mirrors the two TDD loops (see `acceptance-standards.md` §CI):

- **The merge gate** — fast, blocking, on every PR: lint, type-check, unit + integration suites,
  the logic-core coverage floor. Target **< 10 minutes**; a slow merge gate breeds bypass culture,
  which is a security posture, not an inconvenience.
- **The acceptance job** — the E2E suite against the built app + mocked backend, as its **own job**
  (not inside the merge gate), uploading trace/video/screenshots on failure.
- **The done-gate** — the feature is done when **both** are green (brownfield: plus the
  must-stay-green clause; change mode: plus the must-be-migrated clause). The build plan's `ci`
  ticket wires exactly this; the pipeline is where it becomes real.

Post-merge / scheduled work (slow scans, dependency audits, nightly full matrices) runs off the
merge path — never as an excuse to move a real gate off it.

## Required checks, enforced

The gates are **enforced by the forge**, not by convention: branch protection / required status
checks on the default branch, no direct pushes, no merge on red. A green suite nobody is forced to
run is the same as no suite. Red main is an incident: revert or fix forward immediately — a team
that tolerates red main has no gate at all.

## Determinism & the flake policy in CI

The suites' determinism is owned by the testing standards; CI adds the policy teeth: a flaky test is
**quarantined or fixed immediately** — tracked, owned, time-boxed. Blanket auto-retry
(`retry: 2` on the whole suite) is a smell that converts real failures into noise; per-test
quarantine with an owner is the honest form. Pin toolchain and dependency versions in CI (lockfiles,
pinned runners/images) so a pipeline run is reproducible.

## Speed is a feature

Cache dependencies keyed on lockfiles; parallelize suites when runtime warrants; fail fast (lint
before tests, cheap before expensive); use concurrency groups to cancel superseded runs on the same
ref. Budget the merge gate and treat busting the budget as a regression to fix, not a new normal.

## Artifacts, environments & promotion

- **Build once, promote the same artifact.** The artifact that passed the gates is the artifact
  that ships — rebuilding per environment invalidates everything the pipeline proved.
- **Name the environments** (e.g. ci → staging → production) and the promotion rule between each
  pair (automatic on green, manual approval, canary). Keep them boring and few.
- **Environment gating is proven, not assumed** — dev-only conveniences (auth bypass, seed routes,
  debug endpoints) are provably absent from the production build, enforced by a test in the gate
  (see the failure-mode catalog in `unit-integration-standards.md`).

## Secrets

- The merge gate's unit/integration suites need **no secrets at all** (everything external is
  mocked — the testing standards already require this); a unit job asking for a secret is a finding.
- Secrets are scoped to the narrowest job/environment that needs them; short-lived identity
  (e.g. OIDC to the cloud) over long-lived stored keys where the forge supports it; no secrets in
  logs, no secrets in fork-triggered runs.

## Deploy & rollback

Deploys happen **from the pipeline only** — a laptop deploy bypasses every gate the pipeline proved.
Every deploy has a stated rollback path (previous artifact redeploy, flag flip), and migrations are
forward-compatible so rollback doesn't corrupt (ties to the technical spec's rollout/migration
section and change mode's migration ADRs). Rollback is exercised, not theoretical.

## Forge notes

The structure above is forge-neutral. Per-forge, the same ideas are spelled:

| Concern | GitHub Actions | GitLab CI |
|---|---|---|
| Required checks | branch protection required status checks | protected branches + merge checks |
| Cancel superseded runs | `concurrency` groups | `interruptible: true` |
| Dependency cache | `actions/cache` keyed on lockfile | `cache:` keyed on lockfile |
| Short-lived cloud identity | OIDC federation | OIDC / JWT integration |
| Failure artifacts | `actions/upload-artifact` | `artifacts: when: on_failure` |

Stack-specific commands (runners, coverage tooling) come from `profiles/<stack>.md`.

## Design mode: `CI_SPEC.md`

The design-mode artifact — the pipeline as a contract, before (or instead of) writing workflow YAML:

- **Gates & jobs** — a table: job, trigger, contents, budget (runtime), blocking or not. The two-job
  shape above, adapted to the repo's stacks.
- **Required checks** — exactly which jobs guard the default branch.
- **Environments & promotion** — the environment list, the promotion rule per hop, the
  build-once artifact flow.
- **Secrets model** — which jobs get which secrets, and the identity mechanism.
- **Deploy & rollback** — how a deploy runs, how it rolls back, migration compatibility.
- **Flake & red-main policy** — quarantine mechanics, ownership, time-box.
- **Traceability** — which constraint (`constraints.yaml` key) or spec'd bar each gate enforces;
  a gate tracing to nothing is flagged. Compliance constraints (audit trails, provenance) land here.

`CI_SPEC.md` is consumed by the **build-plan-architect**: the pipeline work becomes ordinary
`ci`-layer tickets (merge-gate job, acceptance job, coverage-floor wiring, environment/promotion
setup), dependency-ordered like everything else. The spec is the contract; the tickets are the work.

## Audit mode: dimensions

Audit an existing pipeline the way `test-audit-standards.md` audits suites — **evidence-first**:
every finding cites the workflow file/run and the standard violated; every gap names what merges
unguarded. Score each dimension **sound / weak / absent**:

- **Gate completeness** — does every quality bar the repo's specs/plans define actually run and
  block? (The single highest-leverage check: diff the done-gate against the workflows.)
- **Enforcement** — required checks on the default branch, or convention-only.
- **The two-job shape** — E2E separated from the merge gate; failure artifacts uploaded.
- **Speed** — merge-gate runtime vs. budget; caching; superseded-run cancellation.
- **Determinism & flake policy** — retry-blankets, quarantine practice, pinned toolchains.
- **Secrets hygiene** — secrets in unit jobs, over-scoped secrets, long-lived keys, fork exposure.
- **Environments & promotion** — build-once vs. rebuild-per-env; env gating proven; promotion rules
  explicit.
- **Deploy & rollback** — pipeline-only deploys, stated and exercised rollback.

Findings use the shared format: evidence → standard violated → consequence → remediation → severity
(`high`: unguarded merge path, secrets exposure, no rollback; `medium`: slow gate, blanket retries,
rebuild-per-env; `low`: hygiene). Remediations become an ordinary **`build-plan.yaml`** of
`ci`-layer tickets (keys prefixed `ci-audit-`), validated and publishable like any plan.

## Failure-mode catalog (audit every pipeline against this)

- **The unwired bar** — a coverage floor, lint rule, or regression clause that exists in the specs
  but no job runs. The most common and most damaging.
- **E2E inside the merge gate** — a 40-minute gate everyone bypasses.
- **Retry-until-green** — blanket retries converting real failures to noise.
- **Convention-only gates** — green checks that aren't required checks.
- **Secrets in the fast suite** — unit/integration jobs with cloud credentials (either the tests
  aren't mocked, or the secrets are gratuitous — both findings).
- **Rebuild-per-environment** — the artifact that ships is not the artifact that passed.
- **Laptop deploys** — a deploy path that bypasses the pipeline.
- **No rollback story** — or one that has never been exercised.
- **Red main tolerated** — the gate exists but nobody believes it.
