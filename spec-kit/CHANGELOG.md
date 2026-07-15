# Changelog

All notable changes to the spec-kit plugin. Format follows
[Keep a Changelog](https://keepachangelog.com/); versions follow semver
(`.claude-plugin/plugin.json` is the version of record).

Each version's section is what gets surfaced: in-session by the update-notify
hook the first time Claude starts after an update, and on GitHub as the
release body for that version's tag.

## [Unreleased]

## [0.6.1] — 2026-07-15

### Added
- **Orchestration-hint labels: `mod:<area>` and `resource:<name>`.** Build-plan tickets now carry
  the execution hints the companion dev-orchestrator plugin requires: `mod:<area>` for the ticket's
  component seam (orchestrators parallelize by it) and `resource:<name>` for shared mutable
  resources needing exclusive use (`resource:db` on migrations; orchestrators never run two tickets
  sharing one in parallel). Emitted by build-plan-architect in the free-form `labels` field — no
  schema change — and passed through **verbatim** by publishers, same rule as `tier:*`. The
  spec-challenger's build-plan rubric now flags code-bearing tickets missing a `mod:*` hint and
  schema-touching tickets missing `resource:db`. Field-driven: overheard's first orchestrated run
  needed a manual ticket-smith grooming pass solely to add these labels; published plans are now
  orchestration-ready as-is.

### Changed
- **Idempotency identity is the hidden body marker; per-ticket marker labels are gone.** Publishers
  previously stamped every issue with a `<markerPrefix>:<key>` label — one label per ticket, flooding
  the team's label namespace with identities nobody filters by. The hidden body comment
  (`<!-- <markerPrefix>-key: <key> -->`) is now the sole identity: `publish-linear` searches it with
  one bulk description-contains query, still *matches* legacy marker labels (pre-0.6.0 issues are
  found, and `--update` sheds their marker labels), but never creates new ones. The queryable labels
  (`tier:*`, `pr:*`, layer/stack) are unchanged. Contract updated in `publishing.md`; the Jira skill
  follows the same rule.

### Added
- **`publish-linear --list-teams` / `--list-projects [--team X]`** — discovery for the config
  walkthrough (team key, project names) and an API-key smoke test, so first-run setup needs no MCP.

### Fixed
- **`publish-linear`: workspace-level label collisions.** `issueLabelCreate` fails with "duplicate
  label name" when the name already exists as a workspace-level label (e.g. Linear's built-in
  `Feature`), which the team-scoped label fetch can't see. On that error the script now looks the
  label up workspace-wide and reuses it instead of failing mid-publish. Field-hit on the first real
  run (overheard).
- **Archived issues are matched on re-publish** (`includeArchived` in the stamped-issue search), so a
  published-then-archived ticket is reported as skipped instead of silently recreated; archived
  matches are never updated.
- **`--update` no longer strips a human-added label** that happens to share a name with *another*
  ticket's managed label (preservation now checks the current ticket's label set, not the plan-wide
  set).
- **Silent drops now warn on stderr**: a `defaultAssignee` that resolves to no Linear user, a
  `priority` value missing from `priorityMap`, and an ambiguous team match (multiple teams matching
  the configured name).

## [0.5.0] — 2026-07-15

### Added
- **`bin/publish-linear`**: the Linear publish write path is now a bundled script (PEP-723 `uv` script,
  Linear GraphQL API) instead of agent-driven MCP calls — batched reads (one stamped-issue query, one
  label fetch), deterministic body rendering and label mapping per `publishing.md`, idempotent
  skip/`--update`/create by marker label + body marker, dependency-ordered creation, `parent`/`blockedBy`
  relation wiring with duplicate detection, `--dry-run` preview (works offline without `LINEAR_API_KEY`),
  `--yes` write gate, and `--report` JSON output. The publish-linear skill now drives the script; the
  MCP flow remains as a documented fallback for environments without an API key. Cuts a 40-ticket
  publish from ~150 model-mediated tool calls to one command.

### Changed
- **Context economy for phase-3 agents.** brownfield principle 5 now distinguishes existence claims
  (verify **in bulk** — one Glob per directory or a single `ls`/`test -f` loop, never one Read per file)
  from content claims (Read only the files the artifact makes interface/seam claims about), and states
  that grounding is not re-surveying — `REPO_MAP.md` is the authoritative inventory unless reality
  contradicts it. build-plan-architect and acceptance-spec-architect carry the same discipline inline,
  plus a "context economy" note on their standards lists (read each standard once; load only the
  matching stack profile; `brownfield.md`/`ci-standards.md` only when applicable; skip
  `reference/examples/` unless the schema doc leaves a question open). Cuts the early token spike in
  phase-3 subagents.

## [0.4.0] — 2026-07-14

### Added
- **Tier-mix bar in `validate-build-plan`** (hard failure, not guidance): `complex` tickets capped at max(1, 15% of the plan) — override via `--complex-max-share` — and bundle triggers on code-bearing tickets (feature/interface/migration): >5 `tddCases` or >6 `modulesInScope` entries fails. Infra/ci/harness tickets are exempt (scaffold tickets legitimately list many files). Field data (console-v1 run): 40% of tickets shipped as `complex`, putting ~84% of weighted run cost on the top-tier model — and 8 of those 14 tickets still needed retries there, because bundles fail at execution regardless of model tier. Error messages say split, don't re-label. Test-backfill and CI-remediation plans reuse this validator, so the bar applies to them too.
- **Changelog surfacing**: this file, plus a `SessionStart` hook (`scripts/notify_update.py`) that announces what changed the first session after a plugin update, and `scripts/check_changelog.py` as a release gate that also emits the GitHub release body.

### Changed
- **build-plan-architect: tier prices residual design ambiguity, not domain riskiness.** A ticket whose `interface` + `tddCases` fully pin the design is `standard` even in security/RLS/migration territory — the implementer executes, it doesn't architect. Complex budget ~1 in 10 with a required can't-split justification; mechanical split triggers (>5 tddCases, >6 files in scope, multi-behavior titles); a new complex-tail decomposition pass ("what standard tickets is this hiding?") before validation; tier-mix item in the QC checklist. The `tier` description in `build-plan.schema.json` carries the same framing.
- **challenge-standards: `complex`-tier inflation** added to the build-plan rubric — the mirror of the existing "simple tickets that aren't" check, so the spec-challenger contests over-tiered plans before the human checkpoint.

## [0.3.0] — 2026-07-05

### Added
- **Adversarial challenge phase**: spec-challenger agent attacks every artifact (product spec, technical spec + constraints, design spec, acceptance plan, build plan) between validator and human checkpoint — findings filed as `reviews/CHALLENGE_<PHASE>.md`, never edits. Scope-locked pass-2 re-challenge (dispositions only, no new findings) so the loop terminates. Per-phase rubric in `reference/challenge-standards.md`; `/spec-kit:challenge` for one-off cold reads of existing artifacts.
- **First-class dev-orchestrator hand-off**: plans are the contract; execution stays a separate plugin, no coupling.

## [0.2.0] — 2026-07-05

### Added
- **Design phase**: design-spec-architect producing STYLE_TILE.md → UI_STYLE_GUIDE.md + validated `design-tokens.yaml`; reference-controlled mode (brand guide / existing system / reference repo wins, provenance on every token) alongside invention mode.
- **CI as a first-class phase**: ci-architect agent (DESIGN → CI_SPEC.md; AUDIT → CI_AUDIT.md + optional remediation build-plan.yaml) with `reference/ci-standards.md`.
- **Test audit**: test-audit-architect scoring existing suites per layer against the testing standards; TEST_AUDIT.md + optional test-backfill build-plan.yaml reusing the standard plan schema/validator/publishers.
- **Brownfield support**: change mode, tiered Phase-0 repo survey (REPO_MAP.md), repo grounding, and the light path (mini spec, no tech spec) for small changes.
- **Per-phase skills**: run any spec phase as a one-off (`/spec-kit:product-spec`, `:technical-spec`, `:design-spec`, `:acceptance-plan`, `:build-plan`, `:survey`, `:status`, `:test-audit`, `:ci`).

## [0.1.0] — 2026-06-25

### Added
- Initial release: the spec phase of an AI SDLC as one plugin. Architect-agent chain (product-spec → technical-spec + hard/soft constraint envelope → acceptance plan + test-first build plan) linked by a PR-* traceability spine, with human checkpoints between phases; `/spec-kit:run` drives the full chain.
- Machine-readable artifacts with validators: `constraints.yaml`, `build-plan.yaml`, `acceptance-plan.yaml` (`bin/validate-*`).
- Tracker-neutral plan publishing to Linear and Jira (idempotent via key stamping).
