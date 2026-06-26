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

## Future (deferred by design)

- [ ] **Constraint linting** — make hard constraints mechanically enforced on generated code/IaC via an
  optional `check` predicate + a `lint-constraints` tool (design note in `constraints-schema.md`).
- [ ] **Build-plan fan-out** — for very large technical specs, parallelize one sub-plan per component
  (the agent already documents the merge seam).
- [ ] **More profiles** — add `rust`, `java`, `ruby`, … as needed; one file each.
