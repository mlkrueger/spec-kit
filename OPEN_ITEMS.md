# Open Items

Tracking loose ends for the `spec-kit` plugin (the `test-spec` plugin has been retired and folded in).

## To confirm before sharing

- [ ] **Align the GitHub remote with the documented slug.** The docs are canonical
  (2026-07-05): `mkrueger/agent-specs` in README/`marketplace.json`, `mkrueger/dev-orchestrator`
  for the companion link. The actual remote is currently `mlkrueger/spec-kit` — rename/move the
  GitHub repo to match (then `git remote set-url origin git@github.com:mkrueger/agent-specs.git`),
  or update the docs if the plan changes. Until they match, the README install flow
  (`/plugin marketplace add mkrueger/agent-specs`) won't resolve.
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

## v0.3.0 round (2026-07) — done

- [x] **`spec-challenger` — built** (`CHALLENGER_AGENT_DESIGN.md`, 2026-07-05). Adversarial review
  of every artifact between validator and ⏸ checkpoint: bounded 2-pass loop, rebuttal rights,
  disposition report (`reviews/CHALLENGE_<PHASE>.md`), hard "findings, never fixes" constraint (no
  `Edit` tool). Shipped: `agents/spec-challenger.md`, `reference/challenge-standards.md`,
  `skills/challenge/` (`/spec-kit:challenge`), threading into `run` + `single-phase.md` (full loop
  for product/technical spec, single pass for design/plans, off on the light path), worked example
  `CHALLENGE_PRODUCT_SPEC.example.md`, manifests to 0.3.0. **Panel variant deliberately deferred**
  as a dial (design doc §7); challenge report is presentation/audit only, never downstream input.

## Future (deferred by design)

- [ ] **Challenger panel** — N parallel challengers with distinct lenses, merged by target;
  enable at the technical-spec gate first if ever (design doc §7 — a dial, not a redesign).

- [ ] **Constraint linting** — make hard constraints mechanically enforced on generated code/IaC via an
  optional `check` predicate + a `lint-constraints` tool (design note in `constraints-schema.md`).
- [ ] **Build-plan fan-out** — for very large technical specs, parallelize one sub-plan per component
  (the agent already documents the merge seam).
- [ ] **More profiles** — add `rust`, `java`, `ruby`, … as needed; one file each.
- [x] **Execution agent — decided: out of scope; first-class companion is `dev-orchestrator`.**
  Planning and execution stay two focused plugins, deliberately not bundled. The handshake is via
  the tracker and is already aligned: publish the plans → `tier:<value>` labels + criteria-bearing
  descriptions are exactly what `/dev-orchestrator:orchestrate` routes and gates on (spec-kit
  tickets arrive pre-groomed for its readiness scan). Named in the README's Execution section, the
  run skill's hand-off, and `publishing.md` (tier label format pinned verbatim); everything else
  stays executor-agnostic — no coupling in either direction.
- [ ] **`designRefs` ticket field** — optional build-plan field mirroring `constraintRefs` for design
  contracts (additive; design doc §6). Skipped for v1 — guide references live in ticket prose.
- [ ] **Validator tests + repo CI** — `bin/validate-*` have no test suite and this repo has no
  pipeline; fitting, given `ci-standards.md` now exists. Eat the dog food.
- [ ] **`/spec-kit:status` as a `bin/` tool** — the skill computes staleness/coverage by hand; a
  `bin/spec-status` would make it deterministic.
