# Spec-Phase Agents — Design

> A design (not a build) for two new agents that extend your existing `test-spec` plugin into a
> complete **spec phase** for an AI SDLC: a **Product Spec** agent (outside looking in) and a
> **Technical Spec** agent (inside looking out). Written to mirror the conventions already
> established in `test-spec` (agent body in `agents/*.md`, standards-as-data in `reference/`, a
> machine-validatable side-artifact with a `validate-*` tool, project-scoped agent memory).

---

## 1. Where these fit

Your `test-spec-architect` already sits at the *end* of the spec phase and consumes a technical
spec. These two agents sit upstream and feed it, turning the spec phase into one autonomous chain:

```
idea / brief ──▶ product-spec-architect ──▶ PRODUCT_SPEC.md      (outside-in: what & why)
                                                   │
                                                   ▼
                       technical-spec-architect ──▶ TECHNICAL_SPEC.md   (inside-out: how)
                                                   │   (+ optional constraints.yaml)
                                                   ▼
                           test-spec-architect ──▶ TEST_SPEC.md + ticket-plan.yaml
```

The **artifact is the contract** between phases — exactly as `ticket-plan.yaml` is the contract
between the architect and the publishers today. Each agent ends by handing off, never by doing
the next phase's job.

### Design principles carried over from `test-spec`

| Pattern in `test-spec` | Applied to the new agents |
|---|---|
| Agent persona + methodology in `agents/*.md`, rich `description` with `<example>` blocks | Same frontmatter shape and example-driven invocation |
| Domain judgment lives in `reference/*-standards.md` (standards-as-data) | `product-spec-standards.md`, `technical-spec-standards.md` |
| Structured side-artifact + JSON Schema + `bin/validate-*` | Optional `constraints.yaml` + schema + `validate-constraints` |
| "Pick the lowest layer that proves it" core discipline | Each agent gets one equivalently sharp core rule (below) |
| Project-scoped agent memory, durable/non-obvious only | Identical memory discipline per agent |
| Produce the artifact, then **stop** and point to the next step | Same hand-off discipline between phases |

---

## 2. Packaging recommendation

Promote the existing `test-spec` plugin into a single **spec-phase plugin** (working name
`spec-kit`) that holds all three agents and one shared `reference/` tree. Rationale:

- Your stated goal is "a whole plugin for the spec phase," and the three agents share standards,
  the traceability convention, and (optionally) a publisher.
- One `reference/` tree avoids drift between the three agents' notions of quality.
- It's a rename + reorg, not a rewrite — the test-spec internals are untouched.

Alternative (lower effort, looser cohesion): keep `test-spec` as-is and add `product-spec` and
`technical-spec` as sibling plugins. Everything below works either way; only the paths change.

Proposed layout under one plugin:

```
spec-kit/
├── .claude-plugin/plugin.json
├── agents/
│   ├── product-spec-architect.md
│   ├── technical-spec-architect.md
│   └── test-spec-architect.md         # existing, unchanged
├── reference/
│   ├── product-spec-standards.md      # NEW
│   ├── technical-spec-standards.md    # NEW
│   ├── testing-standards.md           # existing
│   ├── traceability.md                # NEW — the cross-phase ID spine
│   ├── constraints.schema.json        # NEW (optional artifact)
│   ├── ticket-plan.schema.json        # existing
│   └── examples/
│       ├── PRODUCT_SPEC.example.md
│       ├── TECHNICAL_SPEC.example.md
│       └── constraints.example.yaml
├── bin/
│   ├── validate-plan                  # existing
│   └── validate-constraints           # NEW (optional)
└── skills/
    └── publish-linear/                # existing; could later publish epics from specs
```

---

## 3. Agent 1 — `product-spec-architect` (outside looking in)

**Role.** An elite staff PM who turns a rough idea, brief, or transcript into a rigorous,
implementation-free product specification that design and engineering can debate without ever
touching architecture.

**Core discipline (its "narrow waist" rule).**
> **Describe behavior and outcomes, never mechanism.** If a sentence couldn't survive a complete
> rewrite of the tech stack, it doesn't belong in this document.

That single rule is what operationalizes "outside looking in." It is enforced two ways: a
**banned-vocabulary** check in the reference standards (database, endpoint, table, queue, service,
cache, index…), and a self-review pass before finishing.

**Input.** A brief / problem statement / PRD stub / meeting transcript. May ask focused
clarifying questions first when the input is thin (matching `test-spec`'s "ask before proceeding"
behavior).

**Output.** `PRODUCT_SPEC.md`.

### Proposed `PRODUCT_SPEC.md` structure

- **Title & one-paragraph summary** — what this is, in plain language.
- **Problem & context** — who hurts, why now, evidence (links, data, quotes).
- **Goals & non-goals** — explicit non-goals are first-class; out-of-scope is as load-bearing as
  scope.
- **Personas / segments** — who this is for, and who it is explicitly *not* for.
- **User stories / jobs-to-be-done** — each with a stable requirement ID (see §5).
- **Functional requirements as observable behaviors** — the spec's "test cases": trigger / input
  → user-visible outcome. No mechanism. This is the heart of the doc.
- **UX flows & states** — happy path, empty, loading, error, and edge states *described* (not
  designed). Wireframes optional and clearly non-binding.
- **Acceptance criteria** — Given/When/Then, tracker-neutral, falsifiable.
- **Success metrics & guardrails** — the metric that proves it worked, plus the guardrail metric
  that must *not* regress.
- **User-facing non-functional requirements** — "feels instant," "works offline," "accessible" —
  deliberately *deferring numeric targets to the tech spec*. (See the NFR handoff rule, §6.)
- **Open questions & assumptions.**
- **Out of scope for this document: implementation** — an explicit boundary line.

### `reference/product-spec-standards.md` (its standards-as-data)

- The outside-in writing discipline + the banned-vocabulary list.
- An INVEST-style quality bar for requirements (Independent, Negotiable, Valuable, Estimable,
  Small, Testable).
- A **failure-mode catalog** to audit each spec against: vague/unfalsifiable requirements,
  solutioning-in-disguise, missing non-goals, personas with no real job, success metrics that
  can't move, acceptance criteria that aren't checkable.
- The rule for separating user-facing NFRs from engineering targets (the handoff to the tech
  spec).

---

## 4. Agent 2 — `technical-spec-architect` (inside looking out)

**Role.** A principal engineer/architect who turns an approved `PRODUCT_SPEC.md` into the
engineering spec: architecture, component boundaries, interactions, dataflow, and — the part you
specifically called out — the **hard and soft constraints** that bound everything downstream.

**Core discipline.**
> **Decide the load-bearing things; deliberately defer the rest.** Set the architecture and the
> constraint envelope, then explicitly grant per-module implementation freedom. Every load-bearing
> decision records its rejected alternative.

This is the bridge that makes the test-spec agent's job possible — it already expects a technical
spec as input.

**Input.** `PRODUCT_SPEC.md` (+ optionally an existing codebase to respect, the same dual-input
shape the test-spec agent already supports).

**Output.** `TECHNICAL_SPEC.md`, plus an optional machine-readable `constraints.yaml`.

### The constraints model — your explicit ask, made first-class

Constraints are split into two kinds, each recorded with **rationale + owner**:

- **Hard constraints** — non-negotiable, externally imposed givens that bound the solution space.
  Examples: "deployed on AWS," "Python 3.12," "must satisfy SOC2," "Cloud Act / data-residency
  in-region," "p99 < 200ms." Downstream agents may **not** violate these.
- **Soft constraints** — strong defaults, overridable **with documented justification**. Examples:
  "prefer Postgres unless a workload demands otherwise," "favor boring, well-understood tech,"
  "monorepo by default." Each carries an explicit escape hatch.

Making the hard/soft split explicit, attributed, and (optionally) machine-readable is the
highest-leverage idea in this design: it's precisely what lets a downstream coding agent build
autonomously without re-litigating settled decisions or silently breaking a real boundary.

### Proposed `TECHNICAL_SPEC.md` structure

- **Summary & traced product requirements** — which `PR-*` IDs this design satisfies.
- **Architecture overview** — components, responsibilities, a described diagram.
- **Component / module boundaries & interfaces** — contracts, not code.
- **Data model & dataflow** — entities, lifecycle, where data lives and how it moves.
- **External integrations & dependencies.**
- **Constraints** — hard vs. soft, each with rationale + owner. (Mirrors `constraints.yaml` if
  used.)
- **Cross-cutting concerns** — security/auth, observability, error handling, performance targets
  (the *numeric* version of the PM's "feels instant"), scaling.
- **Key decisions & alternatives** — ADR-lite: decision, why, and the rejected option(s).
- **Rollout, migration & operational concerns.**
- **Risks & open technical questions.**
- **Requirements-traceability matrix** — every `PR-*` → where the design addresses it (flag any
  unaddressed).

### `reference/technical-spec-standards.md`

- The hard/soft constraint taxonomy and how to elicit constraints from the product spec, the org,
  and the existing codebase.
- An **ADR-lite** decision-recording format (every load-bearing decision names a rejected
  alternative).
- A **cross-cutting checklist** — security, authz, observability, error handling, performance,
  scaling, cost, migration — each must be *addressed or explicitly deferred*.
- A **failure-mode catalog**: unstated assumptions, missing failure/scaling paths, decisions with
  no rejected alternative, constraints with no owner, "magic" components with undefined contracts.

### Optional `constraints.yaml` + schema + `validate-constraints`

The structural twist worth taking, mirroring your `ticket-plan` pattern exactly:

```yaml
version: 1
spec: TECHNICAL_SPEC.md
constraints:
  - key: deploy-aws
    kind: hard
    category: platform
    statement: "Deployed on AWS (us-east-1 + us-west-2)."
    rationale: "Existing org account, SOC2 boundary, data-residency."
    owner: platform-team
  - key: lang-python
    kind: hard
    category: language
    statement: "Python 3.12 for all services."
    rationale: "Team expertise; shared internal libraries."
    owner: eng-leadership
  - key: db-postgres
    kind: soft
    category: datastore
    statement: "Prefer Postgres unless a workload demands otherwise."
    rationale: "Operational familiarity."
    owner: data-platform
    escapeHatch: "Document the workload need and the chosen alternative in an ADR."
```

- **Schema** (`constraints.schema.json`, draft 2020-12): `key` unique/kebab-case, `kind` ∈
  {hard, soft}, required `statement`/`rationale`/`owner`, `escapeHatch` required when
  `kind: soft`.
- **`bin/validate-constraints`**: schema check + the non-schema rules (key uniqueness, every soft
  constraint has an escape hatch, optional referential link from each constraint to a `PR-*` or
  decision).
- **Payoff**: a downstream coding/IaC agent can *consume* the hard constraints programmatically,
  and you can lint generated Terraform/code against them.

Fallback if you'd rather stay lightweight: keep constraints as a well-structured section inside
`TECHNICAL_SPEC.md` and skip the artifact entirely — the agent body would simply omit the
emit-and-validate step.

---

## 5. Cross-phase traceability spine (recommended, optional)

Give every product requirement a **stable, kebab-case ID** (e.g. `PR-checkout-guest`). Then:

- The **tech spec** references `PR-*` IDs in its traceability matrix.
- The **ticket plan** can reference them too (a small additive field, no breaking change).

This turns three standalone documents into a *linked chain* — idea → requirement → design
decision → test ticket — which is exactly what makes autonomous agent hand-offs auditable and lets
you answer "is every requirement designed and tested?" mechanically. It reuses the same
"reference-by-stable-key" idea your `ticket-plan` already relies on, extended across phases.
Captured in `reference/traceability.md` so all three agents share one convention.

---

## 6. Two seams that keep the phases clean

1. **The NFR handoff.** The product spec states NFRs *as the user experiences them* ("feels
   instant," "never lose my work"); the technical spec **translates each into a numeric target**
   ("p99 < 200ms," "RPO = 0"). Encoding this rule in both reference files prevents the PM from
   solutioning and the tech spec from leaving NFRs vague.

2. **The constraint envelope.** The tech spec's hard/soft constraints are the *only* place
   platform/language/compliance decisions are made. Downstream agents (test-spec today; a future
   coding agent) read them as givens. This is what lets builders act autonomously without
   re-deciding settled questions.

---

## 7. Suggested agent frontmatter (mirrors `test-spec-architect`)

```yaml
# agents/product-spec-architect.md
name: product-spec-architect
description: "Use this agent to turn a brief, problem statement, or transcript into a rigorous,
  implementation-free product spec (PRODUCT_SPEC.md) that design and eng can debate without
  touching architecture. <example>…</example>"
tools: Read, Write, Edit, Bash, Glob, Grep, WebFetch, WebSearch
model: opus
color: green
memory: project
```

```yaml
# agents/technical-spec-architect.md
name: technical-spec-architect
description: "Use this agent to turn an approved product spec into an engineering spec
  (TECHNICAL_SPEC.md): architecture, boundaries, dataflow, and hard/soft constraints, optionally
  emitting a machine-readable constraints.yaml. <example>…</example>"
tools: Read, Write, Edit, Bash, Glob, Grep, WebFetch, WebSearch
model: opus
color: blue
memory: project
```

---

## 8. Build order (when you're ready to go from design → files)

1. Decide packaging (one `spec-kit` plugin vs. siblings) and confirm the constraints-artifact
   choice.
2. Write the two `reference/*-standards.md` files first — they're the durable judgment, and the
   agent bodies just point at them (same as `test-spec`).
3. Write the two `agents/*.md` bodies.
4. (If structured constraints) add `constraints.schema.json` + `bin/validate-constraints` +
   an example.
5. Add `reference/traceability.md` and thread `PR-*` IDs through the examples.
6. Add worked examples (`*.example.md`) so each agent has a gold-standard target.
