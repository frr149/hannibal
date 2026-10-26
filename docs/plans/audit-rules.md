# Audit rules

The rules hannibal must verify on a PRD and its task plan. Each rule has an id so it maps to tests
and to fixtures that violate it. Draft agreed with Fernando on 2026-10-26; refine while importing
the first real PRD.

## Levels

| Level        | Meaning                                                                           |
| ------------ | --------------------------------------------------------------------------------- |
| **error**    | Rejected at write time. Never stored.                                             |
| **pending**  | Allowed while drafting, listed after every write; `check --strict` requires zero. |
| **warning**  | Reported, never blocks.                                                           |
| **judgment** | Needs a model. Never the model that wrote the PRD (opus never reviews).           |

## ORD — Order

- **ORD-1** (pending): every `depends_on` points to an existing task.
- **ORD-2** (error): no dependency cycles.
- **ORD-3** (error): a task never depends on a task of a later milestone. *This is the real error
  that motivated hannibal.*
- **ORD-4** (error): milestones are totally ordered; dates, when given, strictly increase.
- **ORD-5** (judgment): implicit dependencies: a task uses something another task produces (a
  file, a command, a table) without declaring it.
- **ORD-6**: the rendered and planned order is topological; ties break by id.

## SIZE — Task size (single responsibility, without fanaticism)

- **SIZE-1** (pending): every task has at least one test of its own that closes it. If not, it is
  too small: merge it with a neighbour.
- **SIZE-2** (warning): more than ~8 acceptance criteria, or `touches` mixes more than two layers
  (domain, infrastructure, UI).
- **SIZE-3** (judgment): its tests could fail for unrelated reasons: too big, split it. Never split
  only for the sake of it.

## OWN — Owners and the human's time

- **OWN-1** (pending): every task has an owner: `agent` or `human:<name>` (the PRD owner or a third
  party, e.g. a colleague who must answer a question).
- **OWN-2**: separate *asking* from *needing*. Each human task gets a derived `needed_by` (right
  before the first task that depends on it). Tasks that wait on a third party are flagged **ask
  now**: other people's latency is unpredictable.
- **OWN-3**: the plan groups each person's human tasks into **sessions**: tasks whose windows
  overlap merge into one session, placed just before the earliest need. Target: at most one
  session per person per milestone (warning if more).
- **OWN-4** (pending): a task with irreversible or outward-facing effects (writes to another repo,
  creates accounts or infrastructure, spends money, touches production data, publishes) depends on
  a human approval task.
- **OWN-5**: a human task never interrupts a chain of agent tasks unless it blocks that chain.

## REV — Reviews: when and when not

Proportional, following the Adversarial Programming framework: trivial → nothing; normal →
`/penny`; large or risky → panel. Reviews are **derived** from `touches` through one table; tasks
never list reviews by hand.

Default table (one place, data file in the repo):

| Surface touched                                                           | Review                       | Placement                             |
| ------------------------------------------------------------------------- | ---------------------------- | ------------------------------------- |
| Domain model, state machines                                              | `/types --design` · `/types` | before the first task · end of stage  |
| Any production code                                                       | `/improve`                   | end of stage                          |
| Email, notifications, payments, external APIs, deletions, production data | `/safety-preflight`          | before the first task                 |
| Claims about external facts (APIs, versions, laws, prices)                | `/groundedness`              | before the first task                 |
| Auth, personal data, public HTTP surface                                  | `/owasp`                     | once per milestone, before production |
| Servers, provisioning, secrets                                            | `/infra-sec`                 | once per milestone, before production |
| Shared facts across modules (single source of truth)                      | AP patterns audit            | end of stage                          |
| UI forms and flows                                                        | `/design-review`             | stage that builds them                |
| The PRD itself                                                            | `/sheldon` + `/penny`        | before freezing the PRD               |

- **REV-1**: every review run is derived from the table; the plan shows each run and the tasks it
  covers.
- **REV-2** (when not): no expensive review on tasks that only touch docs, fixtures or tests, or
  config without new surface; never rerun a review over an unchanged surface; drop a review that a
  later consolidated run covers.
- **REV-3** (consolidation): each expensive review runs once per surface and scope: *before*
  reviews at the first task that touches the surface, *after* reviews at the last task of the
  stage or milestone.
- **REV-4** (pending): a task that changes code has a non-empty `touches`. (judgment): detect
  missing surfaces (a task that sends email without declaring `email`).
- **REV-5**: the reviewer model differs from the implementer model.

## TEST — Tests and acceptance criteria

- **TEST-1** (pending): every task has at least one acceptance criterion and one test.
- **TEST-2** (pending): every acceptance criterion is covered by at least one named test
  (explicit link criterion → test).
- **TEST-3** (warning + judgment): criteria are verifiable. Lint vague words ("fast", "appropriate",
  "correct", "user-friendly", "etc.", "rápido", "adecuado", "correctamente"…); a model judges the
  rest.
- **TEST-4** (pending): every user story has at least one e2e test in some task.
- **TEST-5** (pending): every incident has a guarantee: `impossible` (citing the type or constraint
  that makes it unrepresentable) or `tests` (naming the battery). Never neither.
- **TEST-6** (error): test names are unique in the PRD and declare a kind: unit, property,
  integration, e2e, smoke.
- **TEST-7**: a stage closes only with every test, linter and type check green. The PRD declares
  its gate commands; the plan prints them per stage.
- **TEST-8** (judgment): invariants (state machines, time zones, money, concurrency) have property
  tests, not only examples.

## TRACE — Traceability and single source

- **TRACE-1** (pending): every reference in prose (`T11`, `HU-A3`, `I7`) resolves.
- **TRACE-2** (warning): orphans: a rule, story or incident no task serves; a task that serves
  nothing.
- **TRACE-3** (error): prose never restates a field (`Depends on:`, `Milestone:` outside fields).

## DEC — Open decisions

- **DEC-1** (pending): every open decision has an owner and the tasks it blocks; its `ask_by` is
  derived (before the earliest blocked task).
- **DEC-2**: a task blocked by an open decision is not ready; the plan lists the decisions to ask
  now.

## VAL — Validation by a human

- **VAL-1** (pending): stories, rules and decision answers are validated by a named human before
  `check --strict` passes.
- **VAL-2**: changing a validated item returns it to draft, and the answer says so.

## Possibly missing (to decide)

- Deliverable overlap: two tasks owning the same file (matters for parallel work in stage 2).
- Milestone feasibility: a milestone date before the `needed_by` of its human tasks.
- Gate drift: a stage whose declared gates differ from the project's actual commands.
