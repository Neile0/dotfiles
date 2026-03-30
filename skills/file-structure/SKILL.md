---
name: file-structure
description: >
  File and directory structure conventions. Language-agnostic.
  Covers modular monolith, Vertical Slice Architecture, Clean Architecture within slices,
  DDD naming, common layers, presentation separation, and test structure.
  Use when creating new files, moving files, or refactoring structure.
  When uncertain where something belongs — ask before creating.
---

# File Structure

> **These are personal structural preferences — not universal mandates.**
> When working in an existing codebase, always follow its established conventions first.
> Apply this structure when starting fresh, or when there is genuine freedom to set the pattern.
> Where constraints exist, honour them — and replicate the *intent* of these preferences
> within whatever structure is already in place. Be pragmatic. Make trade-offs when the
> context demands it, and be explicit about why.

Structure communicates intent. A reader should understand the shape of the system from
the directory tree alone, and the shape of a domain concept from the files within its slice.

When in doubt about where something belongs — ask. A wrong structural decision compounds
over time and is worth a moment of clarification.

---

## Default Pattern: Modular Monolith

When starting a new codebase or personal project, use the **modular monolith** pattern.
Modules map to **bounded contexts** — named after the domain capability they represent,
never after a technical role.

```
// Bad — modules named after technical roles
src/
  services/
  handlers/
  utils/

// Good — modules named after bounded contexts
src/
  automation/
  observability/
  reporting/
```

Each module is self-contained: it owns its features, its cross-cutting concerns, and its
presentation entry points. Module boundaries are the unit of potential future extraction
into independent services.

### Full Structure

```
src/
  common/                          // cross-module shared concerns (see common/)
  moduleA/                         // bounded context A
    public                         // module-level public interface for inter-module use
    features/
      featureA/
      featureB/
    common/                        // concerns shared within this module only
    presentation/
  moduleB/                         // bounded context B
    public
    features/
      featureC/
    common/
    presentation/
  main                             // composition root — wires everything together

tests/
  common/                          // shared test utilities, factories, builders
  moduleA/
    features/
      featureA/
      featureB/
  moduleB/
    features/
      featureC/
```

**Tests mirror `src/` exactly.** The location of a test is always predictable from the
location of the code it covers. A reader should never have to search.

In a monorepo, this rule applies *per application* — each application's `tests/` mirrors
its own `src/`. There is no single top-level `tests/` spanning all applications.

The composition root (`main`) lives at the top of `src/`. It is the only place where
concrete implementations are wired to abstractions. Nothing else constructs a dependency directly.

---

### Monorepo with Multiple Applications

If the modular monolith spans multiple applications or platforms, group by
**application** first. The language is an implementation detail inside each application.

```
mobile/            // e.g. Flutter
  src/
  tests/
web/               // e.g. Angular
  src/
  tests/
backend/           // e.g. Python
  src/
  tests/
```

Each application sits at the repo root as a peer — not nested inside a shared `src/`.
Each has its own independent `src/` and `tests/`.
Cross-application communication happens at the network boundary (HTTP, message queue,
shared API contract) — never via direct imports across application directories.

---

## Working in an Existing Codebase

When working in someone else's codebase, **follow the existing structure**.
Do not impose this pattern onto a codebase not built with it.

The *meaning* behind these preferences still applies everywhere, even when the structure differs:
- Domain logic should not leak into the presentation layer
- Cross-cutting concerns should be abstracted, not scattered
- Related concepts should live together, not separated by technical role
- Tests should be predictable to find

Replicate the intent within the conventions already in place. If the codebase uses
`services/`, work within `services/`. If it uses `controllers/`, keep them thin.
Do not create competing structural patterns in the same codebase.

**Trade-offs are expected and reasonable.** A large, established codebase may make
a full structural approach impractical. In those cases, apply the principles at the
smallest scope where they add value — a single feature, a new module, a refactor
being done anyway. Incremental improvement within existing conventions is always
better than a half-finished structural migration.

When a structural decision will have lasting impact, raise it explicitly rather than
silently diverging or silently conforming. Name the trade-off and let the team decide.

---

## Module Structure

```
src/moduleA/
  public           // module-level public interface — what other modules may import
  features/        // domain — one directory per bounded concept
  common/          // cross-cutting concerns within this module
  presentation/    // entry points — API, CLI, workers
```

These four concerns have different rules, different dependencies, and different reasons
to change. Keep them strictly separate.

---

## features/ — Vertical Slice Architecture

Organise by **domain concept**, not by technical layer. Each slice owns everything
needed to implement one bounded domain concept.

```
// Wrong — organised by technical layer
models/
  job
  pipeline
services/
  job_service
  pipeline_service

// Right — organised by domain concept
features/
  job/
  pipeline/
```

### Anatomy of a Slice

```
features/
  <concept>/
    public                          // narrow public interface — the only export surface
    model                           // entity, value objects, enums
    service                         // application service — orchestrates via ports
    ports                           // abstract interfaces this slice depends on
    adapters                        // concrete implementations of ports
    guards                          // ensure_ guard functions for this concept
    exceptions                      // domain exceptions for this concept
    <concept>_created_domain_event  // one file per domain event
    <concept>_updated_domain_event
    <concept>_deleted_domain_event
```

**Domain events get their own file**, named after the event itself.
This makes events discoverable, independently versioned, and easy to find:

```
features/
  job/
    job_created_domain_event
    job_completed_domain_event
    job_failed_domain_event
```

Only create files that are needed. A simple slice may only need `model`, `service`,
`exceptions`, and one or two event files. **Do not create empty files to satisfy a template.**

### The Public Interface

Every slice exposes a **narrow, deliberate public interface**. Nothing inside a slice
is accessible to the outside except through it.

```
// features/job/public

export Job
export JobId
export JobNotFoundError
export JobCreatedDomainEvent
export JobCompletedDomainEvent

// Not exported: adapters, ports, guards, internal helpers
```

If another slice needs something not in the public interface, the question is not
"should I export this?" — it is "does this concept belong here, or somewhere shared?"

### Growing a Slice

When a single file becomes large, promote it to a sub-directory. The public interface
does not change — internal restructuring is invisible to consumers.

```
// Before
features/
  job/
    model             // growing — Job, JobStatus, RetryPolicy, Schedule all here

// After — promoted, public interface unchanged
features/
  job/
    model/
      job             // primary entity
      status          // JobStatus enum
      retry           // RetryPolicy value object
      schedule        // Schedule value object
    public            // callers see no difference
```

---

## Clean Architecture Within a Slice

Each slice enforces a strict **dependency direction**. Outer layers depend on inner ones.
Inner layers never import outer ones.

```
┌─────────────┐
│   adapters  │  ← infrastructure detail (DB, HTTP, queue)
└──────┬──────┘
       │ implements
┌──────▼──────┐
│    ports    │  ← abstract interfaces (inbound + outbound)
└──────┬──────┘
       │ used by
┌──────▼──────┐
│   service   │  ← application logic, orchestration
└──────┬──────┘
       │ operates on
┌──────▼──────┐
│    model    │  ← domain entities, value objects, events
└─────────────┘
```

**Rules — not guidelines:**
- `model` imports nothing from within the project — pure inner core
- `service` imports `model` and `ports` — never `adapters` directly
- `adapters` imports `ports` and `model` — implements the interfaces
- Nothing imports `adapters` except the composition root (`main`)

Violation of this direction is always a design problem, never a naming problem.

---

## common/ — Cross-Cutting Concerns

There are two levels of `common/`. Both follow the same rule: **do not add speculatively**.
Move something to `common/` when a second real consumer exists — not before.

### Root common/ — Cross-Module Concerns

Concerns that span multiple modules with no single owner live here. This is the highest-trust
layer — changes here affect the whole system.

```
src/
  common/
    exceptions       // AppError and widely-shared base types
    guards           // generic guard utilities (ensureNotNone, ensurePositive...)
    events           // event bus abstraction, base domain event type
    logging          // abstract logger interface
    ports            // interfaces used by multiple modules
    config           // base configuration abstractions
```

### Module common/ — Within-Module Concerns

Concerns shared by multiple slices within one module, but not needed outside it.
If a concern grows to be needed by another module, promote it to root `common/`.

```
src/moduleA/
  common/
    exceptions       // module-specific base exceptions
    guards           // module-specific guards used by multiple slices
```

The module-level `common/` may be empty or absent in simple modules. That is fine —
do not create it until there is a real need.

### What Belongs Where

| Concern | Location |
|---|---|
| Shared across modules | `src/common/` |
| Shared within one module | `src/moduleA/common/` |
| Owned by one slice | `src/moduleA/features/<concept>/` |

---

## Inter-Module Communication

Modules communicate through **module-level public interfaces** — not by importing directly
into another module's slice internals.

Each module exposes a top-level public interface that re-exports what other modules may use:

```
// src/automation/public

export Job
export JobId
export JobScheduledDomainEvent
export ScheduleJobCommand

// Internal slices, adapters, guards — not exported
```

Another module imports from this surface only:

```
// Good — moduleB depends on moduleA's public interface
import { Job } from "automation/public"

// Bad — moduleB reaches into moduleA's internals
import { Job } from "automation/features/job/model"  // never
```

If a module needs something from another module that is not on the public interface,
the options in order of preference are:
1. Add it to the module's public interface if it genuinely belongs there
2. Move it to `src/common/` if it is shared by nature
3. Reconsider whether the dependency direction is correct

Inter-module dependencies should flow in one direction where possible. Circular
inter-module dependencies are always a design problem.

---

## presentation/ — Presentation Layer

The presentation layer contains all **entry points**: REST APIs, GraphQL, gRPC, CLIs,
frontends, async workers, and scheduled runners.

```
presentation/
  api/          // HTTP routes, request/response schemas
  cli/          // commands and argument parsing
  web/          // frontend, if co-located
  workers/      // async worker entrypoints, message consumers
```

### The Fundamental Rule

**Presentation imports from features. Features never import from presentation.**

```
// Good — presentation depends on the slice's public interface
src/moduleA/presentation/api/job_routes  →  imports  →  src/moduleA/features/job/public

// Never — domain coupled to presentation
src/moduleA/features/job/service  →  imports  →  src/moduleA/presentation/api/schemas
```

The presentation layer is entirely replaceable. Adding a CLI to an existing API, or
replacing REST with GraphQL, requires zero changes to domain logic.

### Presentation is Thin

Presentation code does three things only:
1. Parse and validate input into a domain command or query object
2. Call the appropriate domain service
3. Map the result to a response format

```
// Good — thin handler
handle POST /jobs:
  command = ScheduleJobCommand.from(request.body)
  job = jobService.schedule(command)
  return JobResponse.from(job)

// Bad — domain logic has leaked into the presentation layer
handle POST /jobs:
  if request.body.priority > 5 and user.role == "admin":
    ...  // this is a policy — it belongs in the domain
```

Define request/response schemas inside `presentation/` — never in `features/`.
Map between them at the boundary, in the handler.

---

## tests/ — Test Structure

Tests mirror `src/` exactly. Shared test utilities live in `tests/common/`.

```
tests/
  common/               // shared factories, builders, fakes, fixtures
  moduleA/
    features/
      featureA/         // mirrors src/moduleA/features/featureA/
  moduleB/
    features/
      featureC/
```

`tests/common/` is for test infrastructure only — fakes, in-memory implementations,
object builders, shared fixtures. It is not for shared assertions or test helpers that
should be co-located with the tests that use them.

Integration and end-to-end tests live in the mirrored path of the slice they cover,
clearly marked so they can be excluded from fast local runs.

---

## DDD Naming in Structure

Directory and file names reflect the **ubiquitous language** of the domain.

```
// Bad — technical naming hides the domain
src/
  processors/
  handlers/
  managers/

// Good — domain concepts visible at the correct depth
src/
  moduleA/
    features/
      job/
      pipeline/
      policy/
```

### DDD Concepts Map to Files

| DDD Concept | Location |
|---|---|
| Entity | `features/<concept>/model` |
| Value Object | `features/<concept>/model` |
| Aggregate Root | `features/<concept>/model` — primary exported type |
| Domain Service | `features/<concept>/service` |
| Application Service | `features/<concept>/service` |
| Repository (interface) | `features/<concept>/ports` |
| Repository (implementation) | `features/<concept>/adapters` |
| Domain Event | `features/<concept>/<concept>_<verb>_domain_event` |
| Bounded Context | Top-level module directory |

Do not suffix file names when the directory already provides context:
- `features/job/service` — not `features/job/job_service`
- `features/job/model` — not `features/job/job_model`

---

## Within-File Ordering

The most important thing goes at the top.

**Model:** primary entity first, value objects and enums below, private helpers at the bottom.

**Service:** constructor first (states dependencies), public methods next, private methods last.

**Ports:** primary interface first, secondary interfaces below, type aliases at the bottom.

```
// Service file — public contract always above implementation detail
JobService:
  constructor(repo: JobRepository, events: EventEmitter)

  schedule(command: ScheduleJobCommand) -> Job     // public
  cancel(jobId: JobId) -> void                     // public

  private validateWindow(command) -> void          // private below
  private buildJob(command) -> Job                 // private below
```

---

## Preferred Defaults

These are the defaults to reach for. Apply them with judgment — context always wins.

- Modules are named after bounded contexts — never after technical roles
- No top-level `models/`, `services/`, `repositories/` — layer-oriented, not concept-oriented
- No `utils/` or `helpers/` — every utility has a domain home or belongs in `common/`
- No business logic in `presentation/` — parse, delegate, respond
- No imports from `presentation/` inside `features/` — ever
- No cross-slice or cross-module imports that bypass a public interface
- No logic in any public interface file — re-exports only
- No additions to `common/` without a second real consumer to justify them
- No empty files created to satisfy a structural template
- One file per domain event — named after the event
