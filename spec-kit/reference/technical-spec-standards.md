# Technical Spec Standards (Reference)

> Reference document for the **technical-spec-architect** agent. These are the standing standards for
> what a good engineering specification looks like, independent of any one system or stack. The
> technical spec describes **how** a system satisfies an approved product spec: architecture,
> component boundaries, dataflow, and the **hard/soft constraint envelope** that bounds everything
> downstream. When a project's own conventions conflict with these, the project wins, but call out
> the divergence.

## The one discipline everything else serves

**Decide the load-bearing things; deliberately defer the rest.** Set the architecture and the
constraint envelope, then explicitly grant per-module implementation freedom. This is "inside
looking out": you are the engineer inside the system, choosing the seams that downstream builders
must respect and consciously leaving everything else open so they can move fast.

The technical spec is the bridge that makes autonomous downstream execution possible. Its constraint
envelope is the **only** place platform, language, and compliance decisions are made; the build and
acceptance agents read those as givens and never re-litigate them. Every *load-bearing* decision
records its rejected alternative, so the choice can be audited later without re-running the analysis.

## Constraints: the hard/soft taxonomy (the highest-leverage idea)

Constraints bound the solution space. Split every constraint into exactly two kinds, each recorded
with **rationale + owner** (and, optionally, the `PR-*` it serves):

- **Hard constraints** — non-negotiable, externally imposed givens. Downstream agents may **not**
  violate them. Examples: "deployed on AWS (us-east-1 + us-west-2)," "Python 3.12 for all services,"
  "must satisfy SOC2," "data resides in-region (Cloud Act)," "p99 < 200 ms." A hard constraint with
  no real external force behind it is probably a soft constraint in disguise — downgrade it.
- **Soft constraints** — strong defaults, **overridable with documented justification**. Each one
  carries an explicit **escape hatch** describing what a valid override looks like. Examples: "prefer
  Postgres unless a workload demands otherwise," "favor boring, well-understood tech," "monorepo by
  default." A soft constraint with no escape hatch is malformed — either it's actually hard, or you
  haven't said how to override it.

Why this split is the centerpiece: making hard/soft explicit, attributed, and machine-readable is
exactly what lets a downstream coding/IaC agent build autonomously — it can consume the hard
constraints programmatically and lint generated code/Terraform against them, while treating soft
constraints as defaults it may override on the record.

### Eliciting constraints

Pull constraints from three sources, in order:

1. **The product spec** — every user-facing NFR ("feels instant," "works offline," "accessible")
   becomes one or more numeric constraints here (the NFR handoff, below).
2. **The org / environment** — existing cloud accounts, compliance boundaries, language/runtime
   standards, team expertise, budget ceilings, data-residency law.
3. **The existing codebase** (when extending one) — the stack, datastore, and conventions already in
   place are de-facto constraints; respect them or record the decision to diverge as an ADR.

## The NFR handoff (the seam with the product spec)

The product spec states NFRs **as the user experiences them** and deliberately leaves the number
open. The technical spec **translates each into a numeric, testable target** — this is the *only*
place those numbers are set:

| Product spec said | Technical spec sets (a constraint + a cross-cutting target) |
|---|---|
| "feels instant" | p99 interaction latency < 200 ms |
| "never loses my work" | RPO = 0; autosave + durable write before ack |
| "always available" | 99.95% monthly uptime; graceful degradation path defined |
| "works on slow connections" | usable at 3G / 400 kbps; payload budget per view |
| "accessible to everyone" | WCAG 2.2 AA conformance, verified in CI |

Leaving an NFR vague here is a failure mode: every user-facing NFR from the product spec must land as
a concrete target or an explicitly recorded deferral.

## ADR-lite: every load-bearing decision names its rejected alternative

Record each significant decision in a compact form. The rejected alternative is mandatory — a
decision with no alternative considered is either trivial (don't record it) or unexamined (examine
it):

```
### Decision: <the choice made>
- **Context:** what forces this decision (which PR-*, which constraint, what scale).
- **Decision:** what we will do.
- **Rejected:** the alternative(s) considered, and the specific reason each lost.
- **Consequences:** what this commits us to, and what it leaves open downstream.
```

"Load-bearing" means: it constrains downstream work, is expensive to reverse, or a reasonable
engineer would have chosen differently. Datastore choice, sync vs. async boundaries, the
consistency model, the auth model, a build-vs-buy call — yes. Variable names and file layout — no.

## Cross-cutting checklist (each item: addressed or explicitly deferred)

Every technical spec must walk this list and, for each item, either address it or record an explicit,
reasoned deferral. Silent omission is the failure mode:

- **Security & authn/authz** — trust boundaries, identity, the authorization model, secrets handling.
- **Observability** — logs, metrics, traces; how a failure in production is detected and diagnosed.
- **Error handling & failure paths** — what happens when each dependency is slow, down, or returns
  garbage. Fail-closed where security depends on it.
- **Performance targets** — the numeric versions of the product NFRs (see the handoff).
- **Scaling** — the expected load, the growth axis, and what breaks first when it 10×s.
- **Data lifecycle** — retention, deletion, migration, backup/restore, RPO/RTO.
- **Cost** — the rough cost envelope and what dominates it.
- **Rollout & migration** — how this ships without downtime; how it rolls back.
- **CI/CD & environments** — the pipeline gates this design requires (the merge gate + the separate
  acceptance job), the environment list and promotion path, and any compliance constraint that must
  be enforced in CI. Name the gates and environments as constraints here; the full pipeline design
  is the **ci-architect**'s job (`CI_SPEC.md`, per `ci-standards.md`) — don't inline workflow
  detail, and don't leave the gates unstated either.

## The design boundary (frontend features)

The technical spec owns the frontend *architecture* — component framework, state management,
rendering strategy — recorded as constraints like everything else. It **defers all visual
specifics** (palette, type, spacing, component look/states) to the design phase
(`design-spec-architect` → `UI_STYLE_GUIDE.md` + `design-tokens.yaml`), the same membrane that
keeps the product spec out of mechanism. A hex code in a technical spec is the same smell as a
database table in a product spec.

## Requirements traceability (the spine)

Maintain a **requirements-traceability matrix**: every `PR-*` from the product spec → where in this
design it is addressed. Flag any `PR-*` left unaddressed (a coverage gap upstream-of-build) and any
design element that traces to *no* requirement (possible scope creep — justify or cut). See
`traceability.md` for the shared ID convention. This matrix is what lets the downstream validators
confirm every requirement is designed before any ticket is built.

## Failure-mode catalog (audit every spec against this)

- **Unstated assumptions** — load, scale, latency, or environment taken for granted. Surface and
  record them.
- **Missing failure / scaling paths** — only the happy path is designed. Add the dependency-down and
  the 10×-load paths.
- **Decisions with no rejected alternative** — an ADR that considered nothing. Add the alternative or
  drop the ADR.
- **Constraints with no owner** — a constraint nobody is accountable for is unenforceable. Assign an
  owner.
- **Soft constraints with no escape hatch** — malformed; make it hard or add the override condition.
- **"Magic" components** — a box in the architecture with an undefined contract ("the orchestrator
  handles this"). Define its interface or decompose it.
- **NFRs left vague** — a user-facing NFR from the product spec with no numeric target here. Set the
  number or record the deferral.
- **Premature mechanism inside a boundary** — over-specifying a module's internals when the contract
  is what matters. Specify the seam; grant implementation freedom inside it.

## Output discipline

- Specify **contracts, not code** — interfaces, request/response shapes, and dataflow, not
  implementations. Downstream build tickets own the implementation.
- Address the constraint envelope as a first-class section, mirroring `constraints.yaml` when the
  machine-readable artifact is emitted.
- Produce the artifact(s), then **stop** — the next step is decomposition (build plan) and acceptance
  tests, each owned by its own agent. Do not write tickets or tests here.
