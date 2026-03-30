---
name: workflow
description: >
  Engineering workflow for approaching any non-trivial task.
  Enforces a plan-first, clarify-first, code-last discipline.
  Load at the start of any new feature, task, design problem, or refactor.
  Do not write implementation code until this workflow is complete.
---

# Engineering Workflow

Code is the last thing that gets written — not the first.

Writing code before the problem is understood produces work that solves the wrong thing
with confidence. Every assumption that goes unchecked in planning becomes a bug or a
rework in production.

This workflow exists to prevent that. Follow it in order. Do not skip phases.

---

## The Phases

```
1. Understand     — what is actually being asked?
2. Clarify        — what is unclear, assumed, or missing?
3. Decompose      — break the problem into discrete units of work
4. Diagram        — visualise the design before writing anything
5. Validate       — confirm the plan is correct before coding
6. Implement      — write code, test-first, slice by slice
7. Review         — does the result match the plan?
```

---

## Phase 1 — Understand

Before anything else, read the full brief or requirement and form a clear mental model.

Ask:
- What is the **outcome** being requested — not the output?
- Who consumes this? What do they need from it?
- What are the **boundaries** — what is in scope and explicitly out of scope?
- Does this touch existing code? If so, what is the current behaviour?

Do not proceed if the outcome is unclear. A vague requirement produces vague code.

---

## Phase 2 — Clarify

Surface every assumption and ambiguity before designing anything.

List assumptions explicitly:

```
Assumptions:
- The job queue already exists and exposes an async consumer interface
- Jobs are idempotent — processing twice is safe
- Failure handling is out of scope for this iteration

Open questions:
- What is the expected throughput? (affects concurrency model)
- Should failed jobs be retried automatically, or flagged for manual review?
- Is there an existing error reporting mechanism to integrate with?
```

**Ask Aidan to resolve open questions before proceeding.** Do not make silent
assumptions on non-trivial decisions. One focused question at a time if there are many.

The cost of a clarifying question is minutes. The cost of building on a wrong assumption
is hours.

---

## Phase 3 — Decompose

Break the work into discrete, independently deliverable slices.

Each slice should:
- Deliver a testable unit of behaviour
- Map to a single vertical slice in the codebase
- Be completable without depending on another unfinished slice

```
Example decomposition for "add job scheduling":

Slice 1: Job domain model + validation (no I/O)
Slice 2: Schedule job use case + in-memory repository
Slice 3: Postgres repository adapter (integration)
Slice 4: CLI command to schedule a job
Slice 5: Observability — metrics and structured logging
```

Order slices so the domain layer is built first, infrastructure last.
Never start with infrastructure and build inward — the domain is what matters.

Identify **dependencies between slices** and sequence accordingly.
Flag any slice that blocks others — those are the critical path.

---

## Phase 4 — Diagram

Visualise the design before writing implementation. See the `diagramming` skill.

Diagrams to produce depending on the task:

| Task type | Diagrams to create |
|---|---|
| New feature / slice | Domain model + sequence diagram of the happy path |
| New integration / adapter | System context + data flow |
| Refactor | Before and after architecture diagram |
| API design | Sequence diagram of request/response flow |
| Complex domain logic | State machine or flow diagram |

A diagram that takes five minutes to draw can prevent a day of wrong implementation.

Do not skip diagramming for "simple" tasks — simplicity is often an assumption that
collapses under implementation. The diagram will confirm or challenge it.

---

## Phase 5 — Validate

Before writing a single line of implementation, confirm the plan is correct.

Present the following to Aidan for sign-off:

```
1. Problem statement — one or two sentences: what is being solved and why
2. Assumptions made — everything that was not explicitly stated in the brief
3. Open questions resolved — decisions made on each ambiguity
4. Slice breakdown — the work units and their order
5. Key diagrams — the design visualised
```

**Do not proceed to Phase 6 without explicit confirmation.**

If any part of the plan is wrong, it is far cheaper to correct it here than in code.

---

## Phase 6 — Implement

Only now is it time to write code.

Follow the implementation order:
1. Write the test first (TDD — see `python-testing` skill)
2. Implement the minimum to make it pass
3. Refactor
4. Move to the next slice

Work slice by slice. Do not jump between slices or implement ahead of the current test.

Apply all standards from the other skills:
- `coding-practices` — engineering principles
- `python-practices` — Python standards
- `file-structure` — where files live
- `python-testing` — TDD and test design

If a new ambiguity surfaces during implementation — **stop and clarify**.
Do not make a silent decision and keep going. Raise it.

---

## Phase 7 — Review

When a slice is complete, pause and review before moving to the next:

- Does the implementation match the diagram from Phase 4?
- Are all behaviours from the slice covered by tests?
- Does the code follow the standards in the other skills?
- Has anything changed that invalidates assumptions in remaining slices?

If the plan has drifted — update the plan before continuing.
The plan is a living document, not a contract. Keep it honest.

---

## When to Shortcut This Workflow

Not every task warrants all seven phases. Use judgment:

| Task | Minimum workflow |
|---|---|
| Small bug fix | Understand → Clarify if unclear → Implement |
| Minor refactor | Understand → Decompose → Implement |
| New feature | All phases |
| New system or service | All phases — diagram heavily |
| Exploratory / spike | Understand → Implement freely → Discard or formalise |

A spike is intentionally throwaway. If something built during a spike is worth keeping,
run it through the full workflow before treating it as production code.

---

## Non-Negotiables

- Never write implementation code before the problem is understood and clarified
- Never proceed past Phase 5 without Aidan's sign-off
- Never make a silent assumption on a non-trivial decision — ask
- Never skip diagramming for anything involving system interactions or domain design
- Always implement domain logic before infrastructure
- Always raise new ambiguities discovered during implementation — do not silently resolve them
