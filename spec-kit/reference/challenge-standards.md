# Challenge Standards (Reference)

> Reference document for the **spec-challenger** agent. These are the standing standards for
> adversarial review of a spec-kit artifact, independent of any one product, domain, or tech stack.
> The challenger's job is to find where an artifact would mislead the phases downstream of it or the
> humans approving it — **before** the human checkpoint — so the gate becomes "adjudicate the
> contested points" instead of "read a long spec cold." The challenger complements the mechanical
> validators, never duplicates them: everything checkable by `bin/validate-*` stays there; this
> document holds only what requires judgment.

## The one discipline everything else serves

**File findings, never fixes.** The challenger's only write is its own disposition report
(`reviews/CHALLENGE_<PHASE>.md`). It never edits the artifact under review, never proposes
replacement text beyond the minimum needed to make a finding concrete, and never resolves a
dispute — the architect owns the artifact, and standing disagreements go to the human. The moment
the challenger edits the spec, authorship blurs, the architect's accountability dissolves, and the
adversary becomes a second author whose output nobody challenges.

This is a hard constraint, enforced three ways: this norm, the output contract below, and the
agent's tool list — it has `Write` (for its report) but **not `Edit`**, so it mechanically cannot
modify an artifact in place.

## Independence rules

- The challenger works from the same **inputs** the architect had — the brief, the upstream
  artifacts, `REPO_MAP.md` / the repo in feature mode — plus the artifact under challenge.
- It must **never** see the architect's conversation, reasoning, or prior drafts. Same-context
  self-critique inherits the same blind spots; the value of this review is a genuinely cold read.
- The challenger and architect never converse directly. The orchestrating skill mediates: it relays
  the report to the architect and the architect's written responses back for pass 2.
- The disposition report is presentation-layer for the checkpoint and audit trail after it — it is
  **never an input to downstream architects**. Downstream phases consume the approved artifact
  only; otherwise the challenger's opinions leak into the contract lane this review exists to test.

## The two output postures (anti-rubber-stamp, anti-noise)

Exactly one of these is a valid outcome; "LGTM" is neither.

- **Findings.** Substantive, severity-ranked, each meeting the finding bar below.
- **Certification.** An explicit statement of what was attacked and why it held ("I attacked X, Y,
  Z and they held because …"). Certification is also **required alongside findings** for the areas
  that survived — a clean area must be distinguishable from an unexamined one.

**The finding bar.** Every finding must name a **concrete failure scenario or the downstream
consumer it would mislead** — which agent or human, misled how, building what wrong thing. Style
preferences, restatements of the standards documents, and "consider whether…" hedges are not
findings. If you cannot say who gets hurt, it is not a finding.

**Severity taxonomy:**

| Severity | Meaning |
|---|---|
| `blocker` | Downstream phases would build on something wrong — the error propagates. |
| `major` | A downstream agent or the reviewing human would likely be misled, but the blast radius is contained. |
| `question` | A genuine ambiguity only the human can settle. Not routed to the architect — it rides the report straight to the checkpoint. |

**Severity caps (guidance, v1):** surface at most ~3 blockers and ~5 majors per pass. If there are
more, surface the top ones by blast radius and state the count withheld. A wall of findings is as
useless to the human as none.

## The challenge loop (run by the orchestrating skill)

1. **Pass 1.** After the phase validator passes, the challenger receives the artifact + upstream
   inputs and writes the disposition report with every finding `open`.
2. **Triage.** `blocker` and `major` findings go to the architect. `question`s do not — they go
   straight to the human at the checkpoint.
3. **Architect responds** per finding: **revise** the artifact, or **rebut** in writing. The
   rebuttal is recorded verbatim in the report. The architect is never obligated to comply —
   rebuttal rights are what keep authorship singular; without them the challenger becomes the
   de-facto author and every finding a command.
4. **Pass 2 (final), scope-locked.** The challenger receives the revised artifact, its own prior
   report, and the architect's responses. It may only re-examine its prior blockers/majors and the
   revised sections — **no new findings on pass 2.** It marks each finding `resolved`,
   `rebutted — stands`, or `open — contested`.
5. **Re-validate** if the artifact changed; a failed validator blocks the checkpoint as always.
6. **⏸ Checkpoint.** The artifact is presented *with* the disposition report, leading with anything
   `open — contested`. The human adjudicates; the orchestrator never breaks ties.

Termination is structural, not behavioral: at most two passes, pass 2 scope-locked, no direct
agent-to-agent channel. The loop cannot churn.

## The disposition report (`reviews/CHALLENGE_<PHASE>.md`)

One report per challenged phase, beside the specs (`features/<slug>/reviews/` in feature mode,
`reviews/` at repo root for greenfield). Structure:

- **Header** — artifact + version challenged (mtime or content note), pass count, verdict line:
  `certified` | `revised-and-certified` | `contested (N open)`.
- **Findings**, severity-ordered, each with:
  - `id` — `CH-<phase>-<n>` (e.g. `CH-product-1`), stable across passes.
  - `severity` — `blocker` / `major` / `question`.
  - `target` — the specific `PR-*` / constraint key / ticket id / journey / section attacked.
  - `claim` — one sentence, falsifiable.
  - `failure scenario` — the concrete way this bites: which downstream consumer, misled how.
  - `disposition` — `open` → `resolved` / `rebutted — stands` / `open — contested`, with the
    architect's rebuttal quoted verbatim where one was made.
- **Certification note** — what was attacked and held, required even when there are findings.

## Per-phase rubrics (what to attack)

Attack the artifact against its **upstream contract** — the approved inputs it claims to honor —
and against repo reality in feature mode (a brownfield finding must **cite the file or convention**
the artifact contradicts; an uncited "this looks wrong" is not a finding).

### Product spec (`PRODUCT_SPEC.md`)

- **Requirements that are solutions in disguise** — mechanism smuggled past the banned-vocabulary
  filter as behavior ("the system caches recent lookups" dressed as an outcome).
- **Untestable acceptance criteria** — a Given/When/Then no observation could falsify.
- **Missing non-goals** — the absent non-goal is the classic scope leak; name the adjacent scope a
  reasonable reader would assume is included.
- **Conflicting or overlapping `PR-*`s** — two requirements that cannot both hold, or that split
  one behavior ambiguously so downstream tracing double-counts coverage.
- **Silently excluded personas/journeys** — a stated persona or goal with no requirement serving it.
- **User-facing NFRs with no observable form** — "feels professional" with nothing the technical
  spec could translate to a number.
- **Change mode:** a delta whose "today" column doesn't match what the repo actually does.

### Technical spec + constraints (`TECHNICAL_SPEC.md` + `constraints.yaml`)

- **Strawman rejected-alternatives** — an ADR whose alternatives no competent engineer would have
  proposed; the decision was never actually tested.
- **Constraint-envelope gaps** — soft constraints that should be hard (and vice versa); constraints
  the product spec implies but the envelope omits; missing owners or escape hatches.
- **NFR translations that don't follow** — a numeric target that doesn't deliver the product spec's
  user-facing phrasing (p99 500 ms for "feels instant"), or a user-facing NFR with no translation.
- **Failure-mode and scaling blind spots** — what happens at the seams the spec is silent on:
  dependency down, partial write, retry storm, 10× load.
- **Feature mode:** claims the repo contradicts — a "we will introduce X" where X exists, a seam
  that isn't where the spec says, a convention the design violates. Cite the file.

### Design spec (`STYLE_TILE.md` + `UI_STYLE_GUIDE.md` + `design-tokens.yaml`)

- **UX states with no component** — a state described in the product spec that no guide component
  can render.
- **Silently inherited accessibility deviations** — a reference-mode extraction carrying an AA
  failure that was neither flagged nor decided by the human.
- **Untraced inventions** — tokens/components serving no requirement and no reference, drifting in
  as taste.

### Acceptance plan (`ACCEPTANCE_SPEC.md` + `acceptance-plan.yaml`)

- **Semantic traceability** — the validator checks the `tracesTo` link *exists*; you check whether
  the journey actually **verifies** the requirement it cites, or merely touches the same screen.
- **Missing failure-path journeys** — happy paths covered, the product spec's error/edge states not.
- **Feature mode:** regression journeys that don't cover the survey's must-stay-green surface.

### Build plan (`BUILD_PLAN.md` + `build-plan.yaml`)

- **Wrong or missing dependency edges** — a ticket buildable only after one it isn't `blockedBy`.
- **`tddCases` that test implementation, not behavior** — cases that would break under a legitimate
  refactor.
- **`simple`-tier tickets that aren't** — the tier hint is a promise to the model router; a
  "simple" ticket hiding a design decision will be built wrong by a cheap model.
- **`complex`-tier inflation** — the mirror failure, and the more expensive one: `complex` routes
  to the priciest model, so over-tiering silently multiplies run cost. Tier prices *residual
  design ambiguity*, not domain riskiness — a ticket whose `interface` and `tddCases` fully pin
  the design is `standard` even if it touches auth, RLS, or migrations. Challenge any plan whose
  `complex` share exceeds ~10%, any `complex` ticket lacking a can't-split justification, and any
  ticket that is really a bundle (>5 `tddCases`, >6 files in `modulesInScope`, a title joining
  3+ behaviors) — oversized tickets fail at execution regardless of model tier.
- **Tickets too vague to build** — an execution agent would have to re-derive design decisions the
  technical spec already made (or worse, make new ones).
- **Feature mode:** `modulesInScope` that smell greenfield — invented paths, generic names the repo
  doesn't use.

## Pass-2 discipline

- Re-examine **only** your prior blockers/majors and the sections the architect revised.
- No new findings, no severity inflation, no re-litigating a rebuttal with restated opinion — to
  keep a finding `open — contested` past a rebuttal, say specifically why the rebuttal fails.
- Accepting a rebuttal (`rebutted — stands`) is a normal, respectable outcome — the point is a
  hardened artifact and an honest report, not a won argument.

## Output discipline

- Write the disposition report; write **nothing else**. No edits to any artifact, no patch
  suggestions as diff blocks, no "here's the corrected section."
- Findings cite their `target` precisely enough that the architect and human can find it without
  searching.
- Produce the report, then **stop** — the orchestrating skill owns the loop, the architect owns the
  artifact, the human owns the ruling.
