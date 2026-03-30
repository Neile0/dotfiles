---
name: diagramming
description: >
  Visual diagramming standards using Mermaid.
  Covers flow diagrams, sequence diagrams, architecture diagrams,
  entity-relationship models, and state machines.
  Always apply Aidan's brand colours. Use when visualising design,
  system interactions, data models, or domain flows.
---

# Diagramming

Diagrams exist to surface problems that prose hides. A sequence diagram that does not
make sense is a design that does not make sense — the diagram found it cheaply.

Always use **Mermaid** for diagrams. It is text-based, version-controllable, and
renders natively in GitHub, VS Code, and most documentation tools.

---

## Brand Colours

Always apply Aidan's brand palette to diagrams.

| Role | Name | Hex |
|---|---|---|
| Primary / accent | Marsala | `#964F4C` |
| Background / light | Cloud Dancer | `#F0EEE9` |
| Text / dark | Black Beauty | `#27272A` |

Apply via Mermaid's `%%{init}%%` directive and `style` / `classDef` declarations.

---

## Flow Diagrams

Use for: process flows, decision trees, guard clause logic, pipeline steps.

```mermaid
%%{init: {
  "theme": "base",
  "themeVariables": {
    "primaryColor": "#964F4C",
    "primaryTextColor": "#F0EEE9",
    "primaryBorderColor": "#27272A",
    "lineColor": "#27272A",
    "background": "#F0EEE9",
    "nodeBorder": "#27272A",
    "clusterBkg": "#F0EEE9",
    "titleColor": "#27272A",
    "edgeLabelBackground": "#F0EEE9",
    "fontFamily": "monospace"
  }
}}%%
flowchart TD
    A([Start]) --> B{Job status\nPENDING?}
    B -- Yes --> C[Validate scheduling window]
    B -- No --> D([Raise InvalidJobStateError])
    C --> E{Within\nwindow?}
    E -- Yes --> F[Enqueue job]
    E -- No --> G([Raise PolicyViolationError])
    F --> H([Return ScheduledJob])
```

---

## Sequence Diagrams

Use for: request/response flows, inter-service communication, use case walkthroughs.

```mermaid
%%{init: {
  "theme": "base",
  "themeVariables": {
    "primaryColor": "#964F4C",
    "primaryTextColor": "#F0EEE9",
    "primaryBorderColor": "#964F4C",
    "lineColor": "#27272A",
    "background": "#F0EEE9",
    "actorBkg": "#964F4C",
    "actorTextColor": "#F0EEE9",
    "actorBorder": "#27272A",
    "activationBkgColor": "#F0EEE9",
    "fontFamily": "monospace"
  }
}}%%
sequenceDiagram
    actor CLI
    participant JobService
    participant JobRepository
    participant MetricEmitter

    CLI->>JobService: schedule(command)
    JobService->>JobService: ensureJobIsSchedulable(job)
    JobService->>JobRepository: save(job)
    JobRepository-->>JobService: ok
    JobService->>MetricEmitter: emit(job_scheduled)
    MetricEmitter-->>JobService: ok
    JobService-->>CLI: ScheduledJob
```

---

## Architecture / System Diagrams

Use for: module relationships, system context, clean architecture layers,
bounded context maps.

```mermaid
%%{init: {
  "theme": "base",
  "themeVariables": {
    "primaryColor": "#964F4C",
    "primaryTextColor": "#F0EEE9",
    "primaryBorderColor": "#27272A",
    "lineColor": "#27272A",
    "background": "#F0EEE9",
    "clusterBkg": "#F0EEE9",
    "fontFamily": "monospace"
  }
}}%%
flowchart LR
    subgraph presentation["Presentation Layer"]
        CLI["CLI"]
        API["REST API"]
    end

    subgraph automation["automation module"]
        direction TB
        subgraph job["job slice"]
            M["model"]
            S["service"]
            P["ports"]
            A["adapters"]
        end
    end

    subgraph common["common"]
        LOG["logger"]
        ERR["exceptions"]
    end

    CLI --> S
    API --> S
    S --> M
    S --> P
    A --> P
    S --> LOG
```

---

## Entity Relationship Diagrams

Use for: data models, aggregate boundaries, Pydantic model relationships.

```mermaid
%%{init: {
  "theme": "base",
  "themeVariables": {
    "primaryColor": "#964F4C",
    "primaryTextColor": "#F0EEE9",
    "lineColor": "#27272A",
    "background": "#F0EEE9",
    "fontFamily": "monospace"
  }
}}%%
erDiagram
    JOB {
        JobId id PK
        JobName name
        JobStatus status
        RetryCount retries
        datetime created_at
    }
    SCHEDULE {
        ScheduleId id PK
        JobId job_id FK
        datetime start_at
        datetime end_at
    }
    POLICY {
        PolicyId id PK
        string name
        int max_retries
        int timeout_seconds
    }

    JOB ||--o| SCHEDULE : "has"
    JOB }o--|| POLICY : "governed by"
```

---

## State Machine Diagrams

Use for: entity lifecycle, job status transitions, workflow states.

```mermaid
%%{init: {
  "theme": "base",
  "themeVariables": {
    "primaryColor": "#964F4C",
    "primaryTextColor": "#F0EEE9",
    "primaryBorderColor": "#27272A",
    "lineColor": "#27272A",
    "background": "#F0EEE9",
    "fontFamily": "monospace"
  }
}}%%
stateDiagram-v2
    [*] --> PENDING : created

    PENDING --> RUNNING : scheduled
    PENDING --> CANCELLED : cancelled

    RUNNING --> COMPLETED : success
    RUNNING --> FAILED : error
    RUNNING --> CANCELLED : cancelled

    FAILED --> PENDING : retried
    COMPLETED --> [*]
    CANCELLED --> [*]
```

---

## When to Use Each Diagram Type

| Situation | Diagram type |
|---|---|
| How does this process flow? | Flowchart |
| How do these components talk to each other? | Sequence |
| How is the system structured? | Architecture flowchart |
| What does the data model look like? | ERD |
| What states can this entity be in? | State machine |
| What are the module/bounded context relationships? | Architecture flowchart |

When in doubt, a sequence diagram of the happy path is almost always the most
useful first diagram for any new feature.

---

## Diagramming Rules

- Always apply brand colours via `%%{init}%%` — never use default Mermaid theming
- Use `monospace` as the font family for consistency with a technical context
- Keep diagrams focused — one concern per diagram
- If a diagram is getting large, split it — complexity in a diagram reflects complexity in the design
- Label edges (arrows) with the action or data being passed, not just a direction
- Use subgraphs to group related components — they communicate ownership and boundaries
- Diagrams live in documentation (`docs/`) or inline in ADRs — not scattered in comments
