# Challenger Agent — Design

> A design (not a build) for one new agent that adds an **adversarial review step** to every
> `spec-kit` phase: a **`spec-challenger`** that attacks an architect's artifact *before* the human
> checkpoint, so the draft that reaches the human gate is already contested, hardened, and annotated
> with what remains genuinely disputed. It produces one artifact per challenged phase — a
> **disposition report** (`reviews/CHALLENGE_<PHASE>.md`) — and **never edits the spec it reviews**.
> Written to mirror the conventions already established in `spec-kit` (agent body in `agents/*.md`,
> judgment-as-data in `reference/*-standards.md`, the `PR-*` traceability spine, project-scoped agent
> memory, produce-then-stop hand-off).

---

## 1. Where it fits

Every phase of the chain today has two quality layers: the architect's self-validation plus a
mechanical `bin/validate-*` check, and the human checkpoint. The gap is between them — nothing
*semantic* and *independent* examines the artifact before a human does. The validators check
structure (IDs resolve, schema holds, `tracesTo` links exist); the human checks judgment. The
challenger fills the middle: it checks whether the judgment *survives attack* — are the assumptions
stated, are the rejected alternatives real, does the journey actually verify the requirement it
cites — so the human gate becomes "adjudicate the three contested points" instead of "read a
400-line spec cold."

It slots into the existing seam in every phase, **after the validator passes and before the ⏸
checkpoint**:

```
architect ─▶ artifact ─▶ validate-* ─▶ spec-challenger ─▶ findings
                                            │
                     blockers/majors ───────▶ architect revises OR rebuts
                                            │
                     re-challenge (pass 2: deltas + prior blockers ONLY)
                                            │
                              ⏸ checkpoint (artifact + disposition report)
```

The chain itself is unchanged — same phases, same artifacts, same human gates. The challenger is a
quality amplifier inside each phase, not a new phase.

### Design principles carried over from `spec-kit`

| Pattern in `spec-kit` | Applied to `spec-challenger` |
|---|---|
| Agent persona + methodology in `agents/*.md`, rich `description` with `<example>` blocks | Same frontmatter shape and example-driven invocation |
| Domain judgment lives in `reference/*-standards.md` (standards-as-data) | `challenge-standards.md` — one rubric section per phase |
| One sharp core discipline per agent | "File findings, never fixes" (below) |
| `PR-*` traceability spine across phases | Findings cite the `PR-*` / ticket / journey they attack |
| Produce the artifact, then **stop** and point to the next step | Writes the disposition report; never edits the spec; never decides the dispute |
| Project-scoped agent memory, durable/non-obvious only | Remembers *classes* of finding that recur in this project (e.g. "NFR translations here habitually drop the p99") |
| Brownfield: ground in repo reality | Brownfield findings must cite the file/convention the spec contradicts |

### One agent, not one per phase

The adversarial skill — surface implicit assumptions, attack the weakest claim, refuse to
rubber-stamp — is the same regardless of artifact. What varies per phase is the rubric, and
`spec-kit` already keeps per-phase judgment in `reference/*-standards.md`. So: **one agent**, scoped
per invocation by (a) the artifact under challenge and (b) the matching rubric section of
`challenge-standards.md`. This is one job — adversarial review — scoped by artifact, consistent
with the "dedicated scopable agents over modes" principle: the alternative (a challenger baked into
each architect) is the *mode* version, and it also forfeits independence (§2). N architects do not
become 2N agents to maintain.

### Relationship to the user-level `assumption-challenger`

A global `assumption-challenger` agent may exist at the user level. The plugin ships its own
`spec-challenger` regardless (the plugin must be self-contained for other installers), and the two
divide cleanly: the global agent challenges **problem framing before work starts** ("why is this the
problem?"); `spec-challenger` challenges **artifacts against their upstream contracts** ("does this
spec honor the approved product spec / repo reality?"). The agent descriptions keep that distinction
explicit so the two are never confused or double-invoked.

---

## 2. The agent — `spec-challenger`

**Role.** A skeptical senior reviewer — part red-team, part principal engineer doing a cold read —
whose only job is to find where an artifact would mislead the phases downstream of it or the humans
approving it. Not a co-author, not an editor, not a second architect.

**Core discipline (its "narrow waist" rule).**
> **File findings, never fixes.** The challenger's only write is its own disposition report. It
> never edits the artifact under review, never proposes replacement text beyond the minimum needed
> to make a finding concrete, and never resolves a dispute — the architect owns the artifact, and
> standing disagreements go to the human. The moment the challenger edits the spec, authorship
> blurs, the architect's accountability dissolves, and the adversary becomes a second author whose
> output nobody challenges.

This is a **hard constraint**, enforced three ways: the norm above in the agent body, the rubric's
"findings only" output contract, and the tool list — the agent gets `Write` (for its report) but
**not `Edit`**, so it mechanically cannot modify an existing artifact in place.

**Independence is the load-bearing property.** The challenger is invoked fresh (its own Agent-tool
context) with the same *inputs* the architect had — the brief, the upstream artifacts,
`REPO_MAP.md`/the repo in feature mode — plus the artifact under challenge. It must **never** see
the architect's conversation, reasoning, or prior drafts. Same-context self-critique inherits the
same blind spots; the entire value of this agent is a genuinely cold read. (This is also why
"add a self-review section to each architect" was rejected.)

**Anti-rubber-stamp norm.** The challenger must either raise substantive findings **or explicitly
certify** the artifact with stated reasons ("I attacked X, Y, Z and they held because …"). "LGTM"
is not an output. Symmetrically, the **anti-noise norm**: every finding must name a concrete failure
scenario or the downstream consumer it would mislead; style preferences and restatements of the
standards docs are not findings; findings are severity-capped (§5) so ten nitpicks cannot drown two
blockers.

**Input.** The artifact under challenge, its upstream inputs (per phase), the rubric section, and —
on a pass 2 — its own prior report plus the architect's per-finding responses.

**Output.** `reviews/CHALLENGE_<PHASE>.md` (§4). Nothing else.

**Suggested frontmatter** (mirrors the other agents; note the deliberately absent `Edit`):

```yaml
# agents/spec-challenger.md
name: spec-challenger
description: "Use this agent to adversarially challenge a spec-kit artifact (product spec, technical
  spec + constraints, design spec, acceptance plan, build plan) BEFORE the human checkpoint: it
  attacks assumptions, traceability semantics, and constraint gaps against the phase rubric in
  challenge-standards.md, and files a severity-ranked disposition report
  (reviews/CHALLENGE_<PHASE>.md). It never edits the artifact it reviews. <example>…</example>"
tools: Read, Write, Bash, Glob, Grep, WebFetch, WebSearch   # no Edit — findings, never fixes
model: opus
color: red
memory: project
```

---

## 3. The challenge loop

Run by the orchestrating skill (`run`, or a per-phase skill via `single-phase.md`), not by the
agents themselves — the challenger and architect never converse directly.

1. **Pass 1.** After the phase validator passes, launch `spec-challenger` with the artifact + its
   upstream inputs. It writes the disposition report with every finding `open`.
2. **Triage.** `blocker` and `major` findings go back to the architect (re-invoke with the report).
   `question`s skip the loop entirely — they ride the report straight to the human.
3. **Architect responds** per finding: **revise** the artifact, or **rebut** in writing (the
   rebuttal is recorded verbatim in the report). The architect is never obligated to comply — see
   rebuttal rights below.
4. **Pass 2 (final).** Re-launch the challenger with the revised artifact, its own prior report,
   and the architect's responses. **Scope-locked:** it may only re-examine its prior
   blockers/majors and the revised sections — no new findings. It marks each finding `resolved`,
   `rebutted — stands` (it accepts the rebuttal), or `open — contested` (it doesn't).
5. **Re-validate** the artifact if it changed (`bin/validate-*` again — the checkpoint discipline
   that a failed validator blocks presentation is unchanged).
6. **⏸ Checkpoint.** Present the artifact *and* the disposition report, leading with anything
   `open — contested`.

**Termination is structural, not behavioral:** at most two challenger passes, pass 2 scope-locked,
and no direct agent-to-agent channel. The loop cannot churn.

**Rebuttal rights are what keep authorship singular.** Without them the challenger becomes the
de-facto author — every finding a command — and you get spec churn plus an unchallenged second
author. With them, disagreement is *information*: a finding that survives rebuttal contested is
exactly what the human most needs to see. The orchestrator never breaks ties; **the human
adjudicates** at the checkpoint, and their ruling loops through the normal "changes loop, don't
skip" checkpoint discipline if it requires edits.

---

## 4. The artifact — `reviews/CHALLENGE_<PHASE>.md`

One report per challenged phase, living beside the specs (`features/<slug>/reviews/` in feature
mode, `reviews/` at root for greenfield) so `/spec-kit:status` and resume can see which phases were
challenged and whether contested findings were ever adjudicated.

Proposed structure:

- **Header** — artifact + version challenged (content hash or mtime), pass count, verdict line:
  `certified | revised-and-certified | contested (N open)`.
- **Findings**, severity-ordered, each:
  - `id` — `CH-<phase>-<n>`, stable across passes.
  - `severity` — `blocker` (downstream phases would build on something wrong) / `major` (a
    downstream agent or the human would likely be misled) / `question` (genuine ambiguity only the
    human can settle).
  - `target` — the specific `PR-*` / constraint key / ticket id / journey / section attacked.
  - `claim` — one sentence, falsifiable.
  - `failure scenario` — the concrete way this bites: which downstream consumer, misled how.
  - `disposition` — `open` → `resolved` / `rebutted — stands` / `open — contested`, with the
    architect's rebuttal verbatim where one was made.
- **Certification note** — what was attacked and held (required even when there are findings, so a
  clean area is distinguishable from an unexamined one).

The report is **presentation-layer for the gate and audit trail after it** — it is never an input
to downstream architects. Downstream phases consume the approved artifact only, same as today;
otherwise the challenger's opinions would leak into the contract lane it exists to test.

---

## 5. `reference/challenge-standards.md` (its standards-as-data)

The durable judgment the agent body points at:

- **The "file findings, never fixes" discipline** and the output contract (report only, no spec
  edits, no replacement prose beyond making a finding concrete).
- **The anti-rubber-stamp and anti-noise norms** (§2), plus the severity taxonomy and caps: at most
  ~3 blockers + ~5 majors surfaced per pass — if there are more, the top ones by blast radius, with
  a count of what was withheld. A wall of findings is as useless to the human as none.
- **Pass-2 scope-lock rules** (prior blockers/majors + revised sections only).
- **Per-phase rubrics** — what to attack, keyed by artifact:
  - **Product spec:** requirements that are solutions in disguise; untestable acceptance criteria;
    missing non-goals (the absent non-goal is the classic scope leak); conflicting or overlapping
    `PR-*`s; personas/journeys the requirement set silently excludes; user-facing NFRs with no
    observable form.
  - **Technical spec + constraints:** ADR rejected-alternatives that are strawmen; soft constraints
    that should be hard (and vice versa); missing constraints the product spec implies; NFR→numeric
    translations that don't follow from the product NFR; failure-mode and scaling blind spots;
    brownfield — claims the repo contradicts (must cite the file).
  - **Design spec:** UX states in the product spec with no component to render them; accessibility
    deviations inherited silently from a reference; tokens/components tracing to no requirement.
  - **Acceptance plan:** *semantic* traceability — does the journey actually verify the `PR-*` it
    cites, or merely touch the same screen; missing failure-path journeys; feature mode — regression
    journeys that don't cover the must-stay-green surface.
  - **Build plan:** dependency edges that are wrong or missing; `tddCases` that test implementation
    rather than behavior; `simple`-tier tickets that aren't (the model-routing hint is a promise);
    tickets too vague for an execution agent to build without re-deriving design; feature mode —
    `modulesInScope` that smell greenfield.

  The rubrics complement, never duplicate, the validators: everything mechanically checkable stays
  in `bin/validate-*`; the rubric holds only what requires judgment.

---

## 6. Threading into the skills

- **`run/SKILL.md`** — each phase gains the challenge step between "validate" and "⏸":
  - **Default ON, full loop (2 passes)** for the **product spec** and **technical spec** — errors
    there propagate furthest, and the principal-eng review is the load-bearing gate the disposition
    report most improves.
  - **Default ON, single pass** (challenge → architect responds → straight to checkpoint, no
    pass 2) for the **design spec** and **phase-3 plans** — already backstopped by validators, the
    coverage cross-check, and (design) the tile-first loop.
  - **OFF on the light path** — a challenger on a bugfix-sized change is ceremony.
  - The user can widen or narrow this per run; the checkpoint presentation leads with
    `open — contested` findings.
- **`single-phase.md`** — the shared ceremony gains the same optional step ("invoke, validate,
  **challenge**, present, stop"), with the same per-phase defaults, so incremental runs get the
  same hardening.
- **`/spec-kit:challenge <artifact>`** — a thin standalone skill (same shape as the other per-phase
  wrappers): resolve the artifact + upstream inputs, invoke the challenger once, present the
  report, stop. Useful for adversarially reviewing a spec that predates spec-kit or re-challenging
  after manual edits. No loop — the loop belongs to the orchestrating skills.

---

## 7. Deliberately deferred: the panel (and other rejected shapes)

- **Challenger panel — deferred, and the one dial worth keeping warm.** The stronger version of
  this design runs **N challengers in parallel with distinct lenses** (e.g. correctness /
  security / operability for a technical spec), then merges deduplicated findings into one report.
  Perspective diversity catches failure modes that one skeptic — however good — reliably misses.
  It is deferred because it multiplies per-phase cost 3–5× in a pipeline that already ends every
  phase at a human gate, and because the merge/dedup step adds orchestration the single-challenger
  design doesn't need. **If the panel is ever enabled, the technical-spec gate is where it earns
  its cost first.** The design accommodates it additively: same rubric file (lenses become
  sections), same report format (findings already carry ids and targets; a merge step dedups by
  target), same loop — only the fan-out in the orchestrating skill changes. Recorded here so the
  upgrade path is a dial, not a redesign.
- **Free-form architect↔challenger debate — rejected.** Plumbing-heavy in this harness (agents
  don't share a session), unbounded by construction, and the bounded loop with written rebuttals
  captures most of the value while leaving an audit trail the human can actually read.
- **Self-critique inside each architect — rejected.** Same context ⇒ same blind spots; forfeits
  the independence that is the point (§2).
- **Challenger as artifact editor — rejected as a hard constraint, not a preference.** See §2.
  Findings, never fixes.

---

## 8. Build order (when going from this design → files)

> **Status: BUILT** (2026-07-05). All six steps below are done; this document is retained as the
> design record. The panel variant (§7) remains deferred.

1. Write `reference/challenge-standards.md` first — norms, severity taxonomy + caps, loop/scope-lock
   rules, the per-phase rubrics (standards-as-data, same as every other agent).
2. Write `agents/spec-challenger.md` — persona, independence rules, the no-edit tool list (`Write`
   but not `Edit`), report format, memory discipline.
3. Thread the loop into `skills/run/SKILL.md` (per-phase defaults above) and
   `reference/single-phase.md` (the shared ceremony step).
4. Add `skills/challenge/SKILL.md` — the thin standalone wrapper.
5. Add a worked example: `reference/examples/CHALLENGE_PRODUCT_SPEC.example.md` against the
   existing example feature, showing all three severities, a rebuttal that stands, and a
   certification note.
6. Update `OPEN_ITEMS.md`, `README.md` (agent list), and the plugin manifest version.

---

## 9. Open questions

- **Does the challenger see the disposition history of *upstream* phases?** Leaning no for v1
  (each phase challenged against its contract, fresh), but a technical-spec challenger who knows
  the product-spec round left `PR-4` contested might attack the right place sooner. Revisit after
  first use.
- **Severity caps as hard numbers vs. guidance.** Caps (~3 blockers / ~5 majors) are in the rubric
  as guidance for v1; if reports bloat in practice, make them hard limits.
- **Should `/spec-kit:status` surface unadjudicated `open — contested` findings** as a staleness
  analog ("phase approved but contested findings never ruled on")? Cheap to add once reports exist;
  decide when `status` next gets touched.
