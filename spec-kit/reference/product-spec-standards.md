# Product Spec Standards (Reference)

> Reference document for the **product-spec-architect** agent. These are the standing standards for
> what a good product specification looks like, independent of any one product, domain, or tech
> stack. Apply them to every `PRODUCT_SPEC.md` you produce. The product spec describes **what** a
> system must do and **why** — never **how**. When a project's own conventions conflict with these,
> the project wins, but call out the divergence.

## The one discipline everything else serves

**Describe behavior and outcomes, never mechanism.** If a sentence couldn't survive a complete
rewrite of the tech stack — a different language, datastore, cloud, or framework — it does not
belong in this document. This is what "outside looking in" means operationally: you are the user
and the business looking *at* the system, never the engineer looking *out from inside* it.

The product spec is the source of truth for **acceptance** (what the user can observe) and for the
**why** (the problem, the goals, the metric that proves it worked). The technical spec downstream
owns every *how*. Keeping the membrane between them clean is the single highest-value thing this
document does — it lets design and engineering debate the product without prematurely arguing
architecture, and it lets the architecture change later without reopening the product.

## Banned vocabulary (the mechanism filter)

Before finishing, scan the document for mechanism leaking in. The presence of any of these words is
a **smell**, not an automatic error — but each occurrence must justify itself or be rewritten as an
observable behavior. If you cannot restate it as something a user or the business can observe, it is
solutioning and belongs in the technical spec.

- **Storage / data shape:** database, table, schema, column, row, index, query, cache, bucket, blob,
  partition, shard, migration.
- **Compute / topology:** endpoint, API, route, service, microservice, server, queue, worker, job,
  cron, lambda, container, pod, cluster, load balancer.
- **Implementation:** class, function, method, module, library, framework, SDK, repository (code),
  thread, mutex, polling, webhook, websocket.
- **Specific tech names:** Postgres, Redis, Kafka, S3, React, Kubernetes, GraphQL, REST, gRPC, etc.

Allowed and encouraged instead: "the user sees…", "the system remembers…", "within a moment…",
"across sessions…", "is notified when…", "can no longer…". Describe the *experienced effect*, and
let the technical spec choose the machinery.

Two deliberate exceptions: (1) **hard external givens** that genuinely constrain the product
experience (e.g. "must work on a 5-year-old Android phone," "must function with no network") — state
these as user-facing constraints, not as architecture. (2) **Named third-party products the user
directly interacts with** (e.g. "signs in with their existing Google account") — that is observable
behavior, not internal mechanism.

## INVEST quality bar for every requirement

Each functional requirement and user story must clear the INVEST bar:

- **Independent** — stands on its own; minimal entanglement with other requirements' ordering.
- **Negotiable** — captures intent and outcome, leaving room for design; it is not a contract of
  implementation detail.
- **Valuable** — a human or the business can name who benefits and how. If you can't, cut it.
- **Estimable** — concrete enough that engineering could ballpark the effort. Vague requirements
  ("make it fast," "handle errors gracefully") fail here until made observable and bounded.
- **Small** — a coherent single behavior, not a bundle. Split anything with an "and" doing real work.
- **Testable** — falsifiable. There must be an observation that would prove it unmet. "The system is
  intuitive" is not testable; "a first-time user completes checkout without help in under 2 minutes"
  is.

## Requirements as observable behaviors (the heart of the doc)

The functional-requirements section is the spec's equivalent of test cases. Each requirement is a
**trigger / input → user-visible outcome**, with no mechanism between the arrow's two sides:

> **PR-checkout-guest** — *When* a shopper with items in their cart chooses to check out *without*
> an account, *then* they can complete payment and receive an order confirmation, and their order is
> retrievable later via the email they provided.

Every requirement carries a stable `PR-*` ID (see the traceability convention). These IDs are the
spine the entire downstream chain hangs on — the technical spec traces each design decision to them,
and the build and acceptance plans trace each ticket to them.

## Acceptance criteria: Given/When/Then, tracker-neutral, falsifiable

For each requirement (or cluster), write acceptance criteria in **Given/When/Then** form. They must
be falsifiable and free of tracker- or framework-specific language. These are what the downstream
**acceptance-spec-architect** turns directly into executable end-to-end tests — they are the
outer-loop contract, so write them as real user-observable journeys, not UI-click scripts.

> *Given* a guest shopper with one item in their cart, *when* they submit valid payment details,
> *then* they see an order confirmation with an order number, and a confirmation is sent to their
> email.

## The NFR handoff (the seam with the technical spec)

State non-functional requirements **as the user experiences them**, and deliberately **defer the
numeric target to the technical spec**. This is a load-bearing seam encoded in both reference files:

| Product spec says (user-facing) | Technical spec will set (numeric) |
|---|---|
| "feels instant" | p99 latency < 200 ms |
| "never loses my work" | RPO = 0, autosave on every change |
| "always available" | 99.95% monthly uptime |
| "works on slow connections" | usable at 3G / 400 kbps |
| "accessible to everyone" | WCAG 2.2 AA conformance |

You name the experience and the *direction* of what matters; you do **not** invent the number. If
you find yourself writing a threshold, you have crossed into the technical spec's job.

## Goals, non-goals, and personas are first-class

- **Non-goals are as load-bearing as goals.** An explicit "this does **not** do X" prevents scope
  creep and tells the technical spec what it may safely ignore. A spec with no non-goals is
  under-specified.
- **Every persona names a real job.** A persona without a concrete job-to-be-done they are trying to
  accomplish is decoration — cut it. Name who the product is explicitly *not* for, too.
- **Success metrics must be able to move.** State the one metric that proves the feature worked, and
  the **guardrail** metric that must not regress while you move it. A metric no plausible version of
  the feature could change is not a success metric.

## Failure-mode catalog (audit every spec against this)

Before finishing, audit the document for each of these recurring failure modes and fix what you find:

- **Solutioning in disguise** — a requirement that names or implies mechanism (see banned vocab).
  Rewrite as observable behavior.
- **Vague / unfalsifiable requirements** — "intuitive," "fast," "robust," "seamless" with no
  observable test. Replace with a concrete observation and bound.
- **Missing non-goals** — scope with no stated boundary. Add the explicit out-of-scope line.
- **Personas with no job** — a named user type with no concrete task they accomplish. Cut or give
  them a real job.
- **Metrics that can't move** — a success metric no version of the feature could plausibly change,
  or one with no guardrail. Fix the metric or add the guardrail.
- **Acceptance criteria that aren't checkable** — Given/When/Then that no observation could falsify.
  Make the "then" observable.
- **States not described** — only the happy path is specified. Every flow needs empty, loading,
  error, and the meaningful edge states *described* (not designed).
- **NFRs with numbers** — a latency/uptime/throughput threshold in the product spec. Move the number
  to the technical spec; keep the user-facing phrasing here.

## Output discipline

- Write the *specification* — prose, requirements, criteria — never implementation or code.
- Wireframes, if any, are **non-binding** and clearly labeled as illustration, not design.
- End with an explicit boundary line: **"Out of scope for this document: implementation."** The next
  step is the technical spec. Produce the artifact, then **stop** — do not begin designing the
  *how*.
