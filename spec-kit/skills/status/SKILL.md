---
name: status
description: "Report the state of every spec-kit artifact set in the repo (root and features/*/): which artifacts exist, whether each machine-readable one validates, and which are STALE — older than an upstream artifact they were derived from. Read-only; changes nothing. Triggers: 'spec status', 'what's stale', 'where did the spec run leave off', '/spec-kit:status'."
---

# Spec-kit status

Read-only report over the repo's spec-kit artifacts. Answers three questions per artifact set:
what exists, what validates, and what is stale. Upstream edits do not auto-ripple — this skill is
how drift gets seen.

## The dependency order (staleness is defined against it)

```
REPO_MAP.md (survey)                      # informs everything below (brownfield)
  └─ PRODUCT_SPEC.md                      # defines PR-*
       ├─ TECHNICAL_SPEC.md + constraints.yaml
       │    └─ build-plan.yaml + BUILD_PLAN.md        (also consumes design + CI artifacts)
       ├─ STYLE_TILE.md → UI_STYLE_GUIDE.md + design-tokens.yaml   (UI features)
       ├─ acceptance-plan.yaml + ACCEPTANCE_SPEC.md
       └─ CI_SPEC.md                       (when the pipeline was designed)
TEST_AUDIT.md / CI_AUDIT.md               # audits: stale against the repo, not the chain
```

An artifact is **stale** when any artifact upstream of it has a newer modification time. (mtime is
a heuristic — flag, don't conclude; the fix is regenerating or explicitly confirming the downstream
artifact.)

## Procedure

1. **Locate artifact sets.** The repo root and every `features/<slug>/` directory that contains any
   of the artifacts above. Report each set separately.
2. **Existence.** For each set, list which artifacts exist and which are absent. Absence is only a
   gap when something downstream of it exists (a build plan with no technical spec and no light-path
   note is worth flagging; a repo with only a product spec is just early).
3. **Validation.** Run each validator that applies (from inside the set's directory so siblings
   resolve):
   ```sh
   ${CLAUDE_PLUGIN_ROOT}/bin/validate-constraints      constraints.yaml
   ${CLAUDE_PLUGIN_ROOT}/bin/validate-build-plan       build-plan.yaml
   ${CLAUDE_PLUGIN_ROOT}/bin/validate-acceptance-plan  acceptance-plan.yaml
   ${CLAUDE_PLUGIN_ROOT}/bin/validate-design-tokens    design-tokens.yaml
   ```
   Record ok / invalid (with the errors) / not applicable.
4. **Staleness.** Compare mtimes along the dependency order above (e.g.
   `stat -f '%m %N' <files>` on macOS, `stat -c '%Y %n'` on Linux). Flag every artifact older than
   any of its upstreams, and say which upstream moved.
5. **Coverage snapshot** (cheap, high-signal): every `PR-*` in the set's product spec appears in at
   least one build-plan ticket's `tracesTo` and one acceptance journey's `tracesTo`; list the
   uncovered.
6. **Report** one table per artifact set — artifact | exists | validates | stale-against — followed
   by the uncovered `PR-*` list and a short "suggested next step" per set (the per-phase skill that
   regenerates the stalest artifact, or `/spec-kit:run` to resume the chain). **Change nothing;
   suggest, don't launch.**
