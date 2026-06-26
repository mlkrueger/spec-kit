---
name: publish-linear
description: "Publish a tracker-neutral spec-kit plan (build-plan.yaml or acceptance-plan.yaml) to Linear. Creates a project/milestone and issues, maps neutral fields to Linear via per-team config, and is idempotent via key stamping so re-running updates instead of duplicating. Use when the user asks to create/publish/sync build or acceptance tickets to Linear from a spec-kit plan. Triggers: 'publish to Linear', 'create Linear issues from the build plan', '/spec-kit:publish-linear'."
---

# Publish a spec-kit plan to Linear

Maps a tracker-neutral `build-plan.yaml` or `acceptance-plan.yaml` onto Linear. The portable contract —
plan-kind detection, body rendering, idempotency stamping, preview/confirm — lives in
`${CLAUDE_PLUGIN_ROOT}/reference/publishing.md`; **read it first**. This file adds only the Linear
mapping and MCP calls.

## Prerequisites

- **The Linear MCP server must be configured** in the user's environment. This plugin does not bundle
  it (MCP bundling is all-or-nothing, so trackers stay opt-in). If the Linear MCP tools are
  unavailable, stop and tell the user to connect the Linear MCP, then retry. If those tools are
  deferred, load them first with `ToolSearch` (e.g. `select:mcp__plugin_linear_linear__list_teams,mcp__plugin_linear_linear__list_projects,mcp__plugin_linear_linear__save_project,mcp__plugin_linear_linear__list_issues,mcp__plugin_linear_linear__save_issue,mcp__plugin_linear_linear__list_issue_labels,mcp__plugin_linear_linear__create_issue_label`).
- A **validated** plan. Per `publishing.md` Step 0, validate with the matching tool before publishing
  and refuse a plan that fails.

## Inputs

- **Plan:** path argument, else `build-plan.yaml` or `acceptance-plan.yaml` in the working directory.
- **Config:** `${CLAUDE_PROJECT_DIR}/.spec-kit/publisher.yaml` (Linear section). If missing, walk the
  user through it (see *Config*) — do not guess team/project.
- **Flags:** `--update` (overwrite managed fields on existing issues; default skips if exists),
  `--dry-run` (preview only; the first pass always previews and confirms before writing).

## Procedure

Follow `publishing.md` Steps 0–5. The Linear bindings for each step:

1. **Detect & validate** (Step 0) — pick `validate-build-plan` or `validate-acceptance-plan` by plan
   kind; refuse if it fails.
2. **Config** (Step 1) — read the Linear routing config; resolve the team and target project via
   `list_teams` / `list_projects`.
3. **Milestone** (Step 2) — search for a Linear Project (or Milestone, per `milestoneAs`) stamped with
   `<markerPrefix>:<milestone.key>`; create with `save_project` if absent, stamping the key.
4. **Preview & confirm** (Step 3) — summarize create/update/skip; get explicit confirmation; honor
   `--dry-run`.
5. **Create/update issues** (Step 4), in dependency order:
   - **Search** via `list_issues` filtered by the `<markerPrefix>:<key>` label; fall back to the body
     marker.
   - **Skip / `--update` / create** per the idempotency rule; create with `save_issue`.
   - **Render the body** per `publishing.md` *Body rendering* for this plan kind, ending with the hidden
     marker line.
   - **Stamp** the `<markerPrefix>:<key>` label on create (create labels via `create_issue_label` as
     needed).
6. **Wire relationships** (Step 5) once issues exist: `parent` → Linear parent (sub-issue); each
   `blockedBy` → a "blocked by" issue relation.
7. **Report** created/updated/skipped issues with their Linear identifiers/URLs.

## Mapping (neutral → Linear)

| neutral | Linear |
|---|---|
| `milestone` | Project (default) or Milestone, per `milestoneAs` |
| `ticket` | Issue (in the configured team/project) |
| `title` | issue title |
| rendered body (`description` + the plan-kind sections from `publishing.md`) | issue description |
| `layer`, `stack`, `labels` | labels (config may prefix/translate) |
| `tier` (build-plan) | label `tier:<value>` |
| each `tracesTo` `PR-*` | label `pr:<PR-id>` |
| `priority` | Linear priority via `config.priorityMap` |
| `estimate` | issue estimate (dropped if the team has estimates disabled) |
| `parent` | parent issue (sub-issue) |
| `blockedBy` | "blocked by" issue relation |
| `key` | **idempotency marker**: label `<markerPrefix>:<key>` + body comment `<!-- <markerPrefix>-key: <key> -->` |

Routing values (team, project, priority scale, label prefix, assignee) come from config, never the
plan.

## Config

`${CLAUDE_PROJECT_DIR}/.spec-kit/publisher.yaml`:

```yaml
tracker: linear
markerPrefix: "skp"           # idempotency label/marker prefix — keep stable forever
labelPrefix: ""               # optional prefix applied to neutral labels
priorityMap:                  # neutral -> Linear's scale: 0 none, 1 urgent, 2 high, 3 normal, 4 low
  urgent: 1
  high: 2
  medium: 3
  low: 4
linear:
  team: "ENG"                 # team key or name
  project: "Guest Checkout"   # project to hold the issues; created if missing
  milestoneAs: project        # project | milestone
  defaultAssignee: null       # Linear user id/email, or null
```

> **Confirm the `priorityMap`.** Linear's scale is `0=none, 1=urgent, 2=high, 3=normal, 4=low`. The
> values above are a sensible default; adjust to match how your team actually uses priority.

If the file is absent, create it with the user (team + project at minimum), then proceed. This one-time
setup is how a team pins Linear specifics without polluting the portable plan.
