---
name: coding-practices
description: >
  Core engineering principles and practices. Language-agnostic.
  Covers error handling, SOLID, DDD, guard clauses, testability, naming, and design philosophy.
  Use whenever writing, reviewing, or refactoring any code.
---

# Coding Practices

Engineering principles, not style rules. The goal is code that is reliable, resilient,
readable, and honest about what it does and what can go wrong.

---

## Core Disposition

Write the **simplest thing that correctly solves the problem**. Complexity must be justified.

Before implementing, ask:
- What is the actual requirement, right now?
- What is the simplest structure that handles this correctly?
- Am I building for a problem I have, or one I'm imagining?

---

## YAGNI and KISS

**YAGNI** — build for the requirement you have. Speculative abstractions are liabilities.

**KISS** — the simplest correct solution is the right one. Complexity must earn its place.

**Abstract on the second use, not the first.** The first time a pattern appears, implement it
directly. The second time, extract the abstraction — now you have evidence it is genuinely reusable.

Exception: when an engineer has clear domain knowledge a second use is imminent, an early
abstraction is valid — but make that judgment explicit with a comment stating why.

---

## Negative Space Programming

Design code around what **must not happen**, not just what should.

The negative path is not an edge case — it is half the program:
- Make invalid states **unrepresentable** at the type level
- Make the failure path **explicit** and immediately visible
- Validate and reject at the **earliest possible boundary**

```
// Bad — invalid state is representable; failure surfaces late
Job(name: string, retries: int)  // nothing prevents retries = -1

// Good — invariant is owned by the type; illegal state cannot be constructed
RetryCount(value: int):
    if value < 0: reject("RetryCount cannot be negative")

Job(name: JobName, retries: RetryCount)
```

If a type can hold an invalid value, it will eventually hold one.

Use **guard clauses** to enforce preconditions at the entry point of a function — assert
what must be true before proceeding, and exit immediately if not. Name guards descriptively:
`ensureJobIsSchedulable(job)`. A guard owns the decision; the caller just proceeds.

**Railway-Oriented Flow** is a complementary technique for chaining operations that return
recoverable errors. Each step either continues the chain or exits with an explicit error —
no nesting, no hidden failure paths.

---

## Primitive Obsession

Never represent domain concepts with raw primitives. A `string` is not a `JobName`.
An `int` is not a `RetryCount`.

Wrap primitives in typed Value Objects that enforce their own invariants at construction.
This makes it impossible to pass arguments in the wrong order, and moves validation out of
call sites and into the type itself.

Always use **enums or named constants** over hardcoded literals. A raw string or integer
in business logic is always a primitive obsession smell:

```
// Bad
if job.status == "pending"
if retryCount > 3

// Good
if job.status == JobStatus.PENDING
if retryCount > RetryPolicy.MAX_ATTEMPTS
```

---

## Error Handling

### Distinguish Error Categories

**Programmer errors** — violated precondition, unexpected state, invalid argument.
Fail immediately and loudly. These are defects, not domain conditions. Do not recover.

**Expected domain failures** — validation failure, entity not found, policy violation,
external call failure. These are recoverable, anticipated outcomes.
Return them explicitly so the caller is forced to handle them — do not rely on the caller
catching an exception they may not know to expect.

### Custom Error Types

Never raise or throw a generic base error. Define a meaningful error hierarchy:

```
DomainError (base)
  ├── EntityNotFoundException(entityType, entityId)
  ├── PolicyViolationError(policy, reason)
  └── InvalidStateTransitionError(from, to)
```

The error type is the documentation. `EntityNotFoundException` tells the caller everything.
A generic error with a message string tells them nothing and forces string matching.

### Never Silently Swallow Errors

Every caught error must be intentional. If you catch and do not re-raise, you must have a
concrete reason — and that reason belongs in a WHY comment.

---

## Idempotency

When designing an operation that may be retried, replayed, or called more than once,
ask: **what happens if this runs twice?**

Idempotency is a design-time concern, not an infrastructure one. If an operation is not
naturally idempotent, make it so explicitly — or document clearly why it is intentionally not.

**When to consider it:** any operation triggered by a queue, event, scheduled job, webhook,
or retry mechanism. If the context is unclear, ask for clarification before implementing.

---

## Self-Documenting Code

Code should read like prose. If a reader needs a comment to understand *what* or *how*,
rewrite the code — not add a comment.

**Comments explain WHY. Never WHAT or HOW.**

```
// Bad — states the obvious
// loop through jobs and check if expired
for job in jobs:
    if job.expiry < now: ...

// Good — explains a non-obvious decision
// Skip rather than reject — downstream idempotency guarantees make double-processing
// safe but slow. Rejecting here would stall the queue on a non-critical condition.
if job.id in queuedIds: continue
```

If you feel the urge to write a comment, first try to make the code say it.

---

## Naming

Names are the primary documentation. They must be honest, precise, and domain-aware.

- Name after **domain concept**, not technical role — `JobScheduler` not `Manager` or `Helper`
- Methods: **verb + noun** — `scheduleJob`, `emitMetric`, `cancelPolicy`
- Booleans: `is`, `has`, `can`, `should` prefixes only
- Guard functions: `ensure` prefix — they enforce, not query
- No type-encoding in names — `jobName` not `strJobName`
- If a name feels vague (`process`, `handle`, `do`) — the concept is not understood yet.
  Name the concept before naming the function.

---

## Single Responsibility

Every unit of code — function, method, class, module — should **do one thing**.

If a function needs to do something else, extract a named method or function that describes
that thing. The name of the extracted unit becomes part of the documentation.

If you find yourself writing "and" to describe what a class or function does, that is the
signal to split it.

---

## Abstraction Levels

A function must operate at **one level of abstraction**. Do not mix orchestration and detail.

```
// Bad — high-level orchestration mixed with low-level implementation
runJob(jobId):
    row = db.execute("SELECT * FROM jobs WHERE id = ?", jobId)
    if not row: raise EntityNotFoundException(...)
    job = Job(id=row.id, name=row.name)
    if job.status not in ["pending", "retrying"]: raise InvalidStateTransitionError(...)

// Good — orchestration reads at one level; detail lives in named functions
runJob(jobId):
    job, err = loadJob(jobId)
    if err: return err
    ensureJobIsRunnable(job)
    result = execute(job)
    recordOutcome(job, result)
```

If you shift mental gears mid-function — from "what is happening" to "how it works" —
the abstraction levels are mixed. Extract the detail into a named function.

---

## Encapsulation

Hide *how* something works. Expose *what* it does.

- Expose behaviour, not data. Internal state is private.
- If external code reaches into an object's internals, the responsibility is in the wrong place.
- Internal helpers are implementation detail, not public contract.

---

## SOLID

**Single Responsibility** — do one thing. See above.

**Open/Closed** — new behaviour is additive, not surgical. If adding a feature requires
editing existing logic, a missing abstraction is the problem.

**Liskov Substitution** — any implementation of an interface must be fully substitutable.
Needing to check the concrete type of a dependency in business logic is a violation signal.

**Interface Segregation** — narrow, focused interfaces. Callers should not depend on
methods they do not use. Prefer two small interfaces over one large one.

**Dependency Inversion** — depend on abstractions. High-level policy never imports
low-level detail. Wire concrete dependencies at the composition root only.

---

## Dependency Injection and Testability

**If code is hard to test, the design is wrong — not the test.**

Inject all dependencies explicitly. Make time, randomness, and I/O injectable:

```
// Bad — time is a hidden dependency; cannot be controlled in tests
isWithinWindow(job):
    return SystemClock.now() < job.deadline

// Good — clock is injected; fully deterministic in tests
JobScheduler(clock: TimeProvider)
isWithinWindow(job):
    return clock.now() < job.deadline
```

`TimeProvider`, `IdGenerator`, `EventEmitter` are canonical examples of abstractions that
exist to make domain logic testable. Define them as interfaces alongside your domain.

Prefer **pure functions** for computation that does not require I/O or state.
Push side effects to the boundary. The more domain logic is pure, the less test
infrastructure you need.

---

## Inheritance

Inherit for **extension**, not for reuse of implementation.

Use it when a subtype is a genuine specialisation of the parent and extends without
breaking the parent's contract. Default to composition. Reach for inheritance when the
domain relationship is genuinely hierarchical.

It becomes a problem when a subclass overrides behaviour to *change* it, or when the
hierarchy becomes deep enough that you must navigate it to understand a subtype.

---

## Cross-Cutting Concerns

Logging, retry, metrics, and auth must not be scattered through business logic.
Abstract them behind injected components that the domain depends on — not the library itself.

The concrete implementation can use any library. The domain never knows or cares.

**Always use structured logging.** Log events with key-value context, not interpolated strings:

```
// Bad — unstructured, unsearchable
log("Job " + jobId + " failed after " + retries + " retries")

// Good — structured, queryable; event name is a stable identifier
log("job_processing_failed", jobId=jobId, retries=retries, reason=error)
```

---

## Design Patterns

Use patterns when they solve a structural problem you already have.

- **Strategy** — behaviour that varies; replaces conditionals that dispatch on type
- **Factory** — complex or context-dependent construction
- **Observer / Domain Events** — decoupled communication between slices
- **Decorator** — cross-cutting concerns without modifying the target
- **Null Object** — eliminates null checks at call sites with a safe no-op default

A pattern is wrong when it adds more code than the problem it removes, or when a new
engineer needs to learn the pattern to understand the feature.

---

## Domain-Driven Language

Use the language of the domain. If the business calls something a "Policy", the code
has a `Policy` — not a `Rule`, `Config`, or `Setting`.

- **Entities** — have identity and lifecycle
- **Value Objects** — defined by their value, immutable, validated at construction
- **Domain Events** — something meaningful happened
- **Domain Services** — operations that don't belong on a single entity
- **Application Services** — orchestrate domain objects and ports; contain no domain knowledge

Domain logic belongs on domain objects. Services orchestrate.
Infrastructure language must never leak into the domain (`row`, `record`, `cursor`).

---

## Modernisation

When working on a feature or integration, actively ask whether a modern approach would
improve what already exists — not speculatively, but when a real opportunity is visible.

Examples of opportunities worth surfacing:
- An external integration where an **MCP server** already exists — before writing a custom adapter
- Adding observability — **OpenTelemetry** as the vendor-neutral default over platform-specific instrumentation
- Async communication — event-driven or message-queue patterns over polling or tight synchronous chains
- Internal tooling that could be exposed as an **MCP-compatible tool** for AI composability

**When an opportunity arises, raise it explicitly.** Do not silently adopt or silently ignore it.
State what problem the modern approach solves, assess the cost, and ask the engineer to decide.

YAGNI applies here too — modernisation is justified by a concrete, present problem. Novelty alone is not justification.

---

## Code Review Mindset

1. Does this name describe the domain concept accurately?
2. Is there a path where an error is silently ignored?
3. Can an invalid value reach this function — and if so, why isn't it rejected earlier?
4. Is this function operating at a single level of abstraction?
5. Is this function doing more than one thing?
6. If this operation runs twice, is the outcome safe?
7. Is this abstraction solving a problem I have, or one I'm anticipating?
8. Would a new engineer understand this without asking for context?
9. Is this comment explaining WHY — or apologising for unclear code?
10. If I needed to test this in isolation, could I? If not, what is the design problem?
