---
name: publish-jira
description: "Publish a tracker-neutral spec-kit plan (build-plan.yaml or acceptance-plan.yaml) to Jira. Creates an epic and issues, maps neutral fields to Jira via per-team config, and is idempotent via key stamping so re-running updates instead of duplicating. Use when the user asks to create/publish/sync build or acceptance tickets to Jira from a spec-kit plan. Triggers: 'publish to Jira', 'create Jira issues from the build plan', '/spec-kit:publish-jira'."
---

# Publish a spec-kit plan to Jira

Maps a tracker-neutral `build-plan.yaml` or `acceptance-plan.yaml` onto Jira. The portable contract —
plan-kind detection, body rendering, idempotency stamping, preview/confirm — lives in
`${CLAUDE_PLUGIN_ROOT}/reference/publishing.md`; **read it first**. This file adds only the Jira mapping
and MCP calls. It is the `publish-linear` skill with the tracker bindings swapped — the neutral plan and
the architects do not change.

## Prerequisites

- **The Jira (Atlassian) MCP server must be configured** in the user's environment. This plugin does
  not bundle it. If the Jira MCP tools are unavailable, stop and tell the user to connect the Atlassian
  MCP, then retry. If those tools are deferred, load them first with `ToolSearch` (e.g. `select:mcp__claude_ai_Atlassian_Rovo__getAccessibleAtlassianResources,mcp__claude_ai_Atlassian_Rovo__getVisibleJiraProjects,mcp__claude_ai_Atlassian_Rovo__getJiraProjectIssueTypesMetadata,mcp__claude_ai_Atlassian_Rovo__searchJiraIssuesUsingJql,mcp__claude_ai_Atlassian_Rovo__createJiraIssue,mcp__claude_ai_Atlassian_Rovo__editJiraIssue,mcp__claude_ai_Atlassian_Rovo__getJiraIssue,mcp__claude_ai_Atlassian_Rovo__createIssueLink,mcp__claude_ai_Atlassian_Rovo__getIssueLinkTypes,mcp__claude_ai_Atlassian_Rovo__lookupJiraAccountId`).
- A **validated** plan. Per `publishing.md` Step 0, validate with the matching tool before publishing
  and refuse a plan that fails.

## Inputs

- **Plan:** path argument, else `build-plan.yaml` or `acceptance-plan.yaml` in the working directory.
- **Config:** `${CLAUDE_PROJECT_DIR}/.spec-kit/publisher.yaml` (Jira section). If missing, walk the user
  through it (see *Config*) — do not guess project/issue types.
- **Flags:** `--update` (overwrite managed fields on existing issues; default skips if exists),
  `--dry-run` (preview only; the first pass always previews and confirms before writing).

## Procedure

Follow `publishing.md` Steps 0–5. The Jira bindings for each step:

1. **Detect & validate** (Step 0) — pick `validate-build-plan` or `validate-acceptance-plan` by plan
   kind; refuse if it fails.
2. **Config & cloud** (Step 1) — read the Jira routing config; resolve the `cloudId` via
   `getAccessibleAtlassianResources` and confirm the project with `getVisibleJiraProjects`. Read the
   project's issue-type + field metadata with `getJiraProjectIssueTypesMetadata` (for the epic/story
   type ids and the story-points field).
3. **Milestone → Epic** (Step 2) — search via JQL for an Epic stamped with the milestone marker
   (`searchJiraIssuesUsingJql`, e.g. `project = X AND issuetype = Epic AND text ~ "<markerPrefix>-key: <milestone.key>"`);
   create with `createJiraIssue` (issuetype Epic) if absent, stamping the key.
4. **Preview & confirm** (Step 3) — summarize create/update/skip; get explicit confirmation; honor
   `--dry-run`.
5. **Create/update issues** (Step 4), in dependency order:
   - **Search** via JQL on the body marker `text ~ "<markerPrefix>-key: <key>"` (and/or the
     `<markerPrefix>-<key>` label).
   - **Skip / `--update` / create** per the idempotency rule; create with `createJiraIssue` (issuetype
     from `issueTypeMap`, default Story), setting the epic parent.
   - **Render the description** per `publishing.md` *Body rendering* for this plan kind, ending with the
     hidden marker line.
   - **Stamp** the `<markerPrefix>-<key>` label on create.
6. **Wire relationships** (Step 5) once issues exist: `parent` → a sub-task or the issue's parent field
   per `parentAs`; each `blockedBy` → an "is blocked by" link via `createIssueLink` (resolve the link
   type with `getIssueLinkTypes`).
7. **Report** created/updated/skipped issues with their Jira keys/URLs.

## Mapping (neutral → Jira)

| neutral | Jira |
|---|---|
| `milestone` | Epic |
| `ticket` | issue of `issueTypeMap[layer]` (default Story) |
| `title` | summary |
| rendered body (`description` + the plan-kind sections from `publishing.md`) | description |
| `layer`, `stack`, `labels` | labels (config may prefix/translate; no spaces — use hyphens) |
| `tier` (build-plan) | label `tier-<value>` |
| each `tracesTo` `PR-*` | label `pr-<PR-id>` |
| `priority` | Jira priority name via `config.priorityMap` |
| `estimate` | story-points field (from project metadata; dropped if unconfigured) |
| `parent` | sub-task, or the parent field, per `parentAs` |
| `blockedBy` | "is blocked by" issue link |
| `key` | **idempotency marker**: label `<markerPrefix>-<key>` + body comment `<!-- <markerPrefix>-key: <key> -->` |

Jira labels disallow spaces, so the markers/labels use hyphens (`<markerPrefix>-<key>`) rather than the
colon form Linear uses; the hidden body marker is identical across trackers and is the authoritative
fallback found via JQL `text ~`.

Routing values (project, issue types, priority scale, story-points field, assignee) come from config,
never the plan.

## Config

`${CLAUDE_PROJECT_DIR}/.spec-kit/publisher.yaml`:

```yaml
tracker: jira
markerPrefix: "skp"           # idempotency label/marker prefix — keep stable forever
labelPrefix: ""               # optional prefix applied to neutral labels
priorityMap:                  # neutral -> Jira priority NAME
  urgent: "Highest"
  high: "High"
  medium: "Medium"
  low: "Low"
jira:
  project: "GC"               # Jira project key
  issueTypeMap:               # neutral layer -> Jira issue type (default Story)
    harness: "Task"
    ci: "Task"
    docs: "Task"
    refactor: "Task"
    # feature / interface / migration / e2e -> Story (default)
  parentAs: subtask           # subtask | parent-field
  storyPointsField: null      # e.g. "customfield_10016"; null drops estimates
  defaultAssignee: null       # Jira accountId or email (resolved via lookupJiraAccountId), or null
```

If the file is absent, create it with the user (project key + the epic/story issue types at minimum),
then proceed. Confirm the `priorityMap` names match the project's priority scheme and the
`storyPointsField` id from the project metadata.
