# Open Items

Tracking loose ends for the `spec-kit` plugin (the `test-spec` plugin has been retired and folded in).

## To confirm before sharing

- [ ] **Push to GitHub & confirm install flow.** Confirm the real remote slug, then:
  ```
  /plugin marketplace add <your-gh-username>/agent-specs
  /plugin install spec-kit@agent-specs
  ```
  The `marketplace.json` currently assumes `mkrueger/agent-specs`; this is not yet a git repo locally.
- [ ] **Publishers' MCP round-trip is unverified.** `publish-linear` and `publish-jira` are written
  against the Linear / Atlassian MCP tool names but have not been run against a live tracker (doing so
  creates real issues). Do a `--dry-run` against your Linear/Jira before relying on them. Confirm the
  Linear `priorityMap` (Linear's scale is `0=none,1=urgent,2=high,3=normal,4=low`) and the Jira
  `priorityMap` names + `storyPointsField` match your projects.
- [ ] **Memory scope.** Each architect writes project-scoped memory to
  `${CLAUDE_PROJECT_DIR}/.claude/agent-memory/<agent>/`, committed for team sharing. Confirm that scope
  is right for shared/team use.

## test-spec retirement — done

- [x] Testing judgment split into `testing-standards-shared.md` + `unit-integration-standards.md` +
  `acceptance-standards.md`.
- [x] Profiles migrated (`python`, `js-frontend`, `js-node`, `go`).
- [x] `publish-linear` generalized into spec-kit; `publish-jira` added.
- [x] `ticket-plan` schema + `validate-ticket-plan` retained as legacy publish-only support (no agent
  produces a ticket-plan; build-plan + acceptance-plan supersede it).
- [x] Standalone `test-spec` plugin and `test-spec-architect` agent removed.

## v0.2.0 round (2026-07) — done

- [x] **Per-phase skills** — `/spec-kit:survey|product-spec|technical-spec|design-spec|
  acceptance-plan|build-plan|test-audit|ci|status`; shared ceremony in `reference/single-phase.md`.
- [x] **Brownfield deepened** — change mode (behavior deltas, `Modifies:` links, test migration,
  regression-surface split), tiered survey (scoped default / full), repo-grounding principle, light
  path for small changes.
- [x] **test-audit-architect** — scopable per layer; `TEST_AUDIT.md` + optional backfill build plan
  via the standard machinery; `reference/test-audit-standards.md`.
- [x] **ci-architect** — design (`CI_SPEC.md`) / audit (`CI_AUDIT.md` + remediation plan);
  `reference/ci-standards.md`; technical spec gained the CI/CD & environments item; build plan emits
  explicit `ci` tickets.
- [x] **design-spec-architect** — built per `DESIGN_PHASE_AGENT_DESIGN.md` including the
  reference-controlled mode (brand guide / existing system / reference repo controls; provenance;
  AA floor mechanically enforced by `bin/validate-design-tokens`); threaded into run skill + build
  plan; worked examples added.

## Future (deferred by design)

- [ ] **Constraint linting** — make hard constraints mechanically enforced on generated code/IaC via an
  optional `check` predicate + a `lint-constraints` tool (design note in `constraints-schema.md`).
- [ ] **Build-plan fan-out** — for very large technical specs, parallelize one sub-plan per component
  (the agent already documents the merge seam).
- [ ] **More profiles** — add `rust`, `java`, `ruby`, … as needed; one file each.
- [ ] **Execution agent** — every hand-off names "an execution agent" that walks the build plan
  red→green→refactor with tier routing; nothing in the plugin implements it yet. Decide: own agent
  here, or explicitly out of scope (a separate plugin).
- [ ] **`designRefs` ticket field** — optional build-plan field mirroring `constraintRefs` for design
  contracts (additive; design doc §6). Skipped for v1 — guide references live in ticket prose.
- [ ] **Validator tests + repo CI** — `bin/validate-*` have no test suite and this repo has no
  pipeline; fitting, given `ci-standards.md` now exists. Eat the dog food.
- [ ] **`/spec-kit:status` as a `bin/` tool** — the skill computes staleness/coverage by hand; a
  `bin/spec-status` would make it deterministic.
