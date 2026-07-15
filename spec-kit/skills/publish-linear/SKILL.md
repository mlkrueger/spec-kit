---
name: publish-linear
description: "Publish a tracker-neutral spec-kit plan (build-plan.yaml or acceptance-plan.yaml) to Linear. Creates a project/milestone and issues, maps neutral fields to Linear via per-team config, and is idempotent via key stamping so re-running updates instead of duplicating. Use when the user asks to create/publish/sync build or acceptance tickets to Linear from a spec-kit plan. Triggers: 'publish to Linear', 'create Linear issues from the build plan', '/spec-kit:publish-linear'."
---

# Publish a spec-kit plan to Linear

Maps a tracker-neutral `build-plan.yaml` or `acceptance-plan.yaml` onto Linear. The portable contract —
plan-kind detection, body rendering, idempotency stamping, preview/confirm — lives in
`${CLAUDE_PLUGIN_ROOT}/reference/publishing.md`. The whole write path is implemented by the bundled
script **`${CLAUDE_PLUGIN_ROOT}/bin/publish-linear`** (Linear GraphQL API, batched calls, deterministic
mapping) — **drive the script; do not publish ticket-by-ticket yourself.** An MCP fallback exists for
environments without an API key (see *Fallback*).

## Prerequisites

- **`LINEAR_API_KEY`** in the environment (a Linear personal API key: Linear → Settings → Security &
  access → Personal API keys). If unset, ask the user to export it (suggest they run
  `! export LINEAR_API_KEY=...` so it lands in this session), or fall back to MCP.
- A **validated** plan. Per `publishing.md` Step 0, validate with the matching tool before publishing
  and refuse a plan that fails.
- **Config** at `${CLAUDE_PROJECT_DIR}/.spec-kit/publisher.yaml` (Linear section). If missing, walk the
  user through it (see *Config*) — do not guess team/project.

## Procedure

1. **Validate** (Step 0): run `validate-build-plan` / `validate-acceptance-plan` per plan kind; refuse
   a failing plan.
2. **Preview** — always first, no writes:
   ```sh
   ${CLAUDE_PLUGIN_ROOT}/bin/publish-linear <plan.yaml> --config .spec-kit/publisher.yaml --dry-run
   ```
   It prints the container action and the per-ticket create/update/skip set (add `--update` to the
   preview if the user wants re-stamped issues overwritten). Show the user this summary and **get
   explicit confirmation**.
3. **Publish** — same command with `--yes` (plus `--update` if confirmed), and `--report` for a
   machine-readable result:
   ```sh
   ${CLAUDE_PLUGIN_ROOT}/bin/publish-linear <plan.yaml> --config .spec-kit/publisher.yaml --yes --report publish-report.json
   ```
4. **Report** created/updated/skipped issues with their Linear identifiers/URLs (from the script's
   output / the JSON report).

The script implements the full contract: idempotency by `<markerPrefix>:<key>` label + hidden body
marker (skip by default; `--update` overwrites managed fields only, preserving human-added labels),
dependency-ordered creation, `parent` → sub-issue, `blockedBy` → "blocked by" relations (existing
relations detected, never duplicated), labels created as needed, `estimate` dropped automatically if
the team has estimates disabled.

## Mapping (neutral → Linear)

| neutral | Linear |
|---|---|
| `milestone` | Project (default) or Project Milestone, per `milestoneAs` |
| `ticket` | Issue (in the configured team/project) |
| `title` | issue title |
| rendered body (`description` + the plan-kind sections from `publishing.md`) | issue description |
| `layer`, `stack`, `labels` | labels (prefixed by `labelPrefix`) |
| `tier` (build-plan) | label `tier:<value>` (verbatim) |
| each `tracesTo` `PR-*` | label `pr:<PR-id>` (verbatim) |
| `priority` | Linear priority via `config.priorityMap` |
| `estimate` | issue estimate (dropped if the team has estimates disabled) |
| `parent` | parent issue (sub-issue) |
| `blockedBy` | "blocked by" issue relation |
| `key` | **idempotency marker**: hidden body comment `<!-- <markerPrefix>-key: <key> -->` (sole identity; legacy `<markerPrefix>:<key>` labels are matched but never created) |

Routing values (team, project, priority scale, label prefix, assignee) come from config, never the
plan.

## Config

`${CLAUDE_PROJECT_DIR}/.spec-kit/publisher.yaml`:

```yaml
tracker: linear
markerPrefix: "skp"           # idempotency body-marker prefix — keep stable forever
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
setup is how a team pins Linear specifics without polluting the portable plan. To discover the values
(and smoke-test the API key), use the script itself — no MCP needed:

```sh
${CLAUDE_PLUGIN_ROOT}/bin/publish-linear --list-teams
${CLAUDE_PLUGIN_ROOT}/bin/publish-linear --list-projects --team ENG
```

## Fallback: Linear MCP (no API key available)

When the user cannot mint an API key but has the Linear MCP connected, execute `publishing.md` Steps
0–5 manually via the MCP tools (load them first with `ToolSearch`, e.g.
`select:mcp__plugin_linear_linear__list_teams,mcp__plugin_linear_linear__list_projects,mcp__plugin_linear_linear__save_project,mcp__plugin_linear_linear__list_issues,mcp__plugin_linear_linear__save_issue,mcp__plugin_linear_linear__list_issue_labels,mcp__plugin_linear_linear__create_issue_label`):
search by the hidden body marker (`<markerPrefix>-key:` in the description; legacy `<markerPrefix>:<key>`
labels also count as a match but are never created), skip/update/create per the idempotency rule, render
bodies per `publishing.md`, stamp the body marker on create, then wire `parent`/`blockedBy` relations.
This path is slow and token-heavy for large plans — prefer the script whenever possible.
