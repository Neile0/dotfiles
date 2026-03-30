---
name: shared-lib-context
description: >
  Living document: context about the shared library in the current codebase.
  What it contains, what it exports, how to consume it, and what belongs in it.
  Use whenever writing code that may need shared models, utilities, or interfaces.
  Update this file as the shared lib evolves — it should always reflect reality.
---

# Shared Library Context

> ⚠️ **This is a living document.** Keep it current — an outdated skill is worse than
> no skill. Update it whenever the shared lib changes: new models, new utilities,
> new constraints, breaking changes.

The shared library is the **single source of truth** for domain models, utilities,
and abstractions used across multiple services or repos in this codebase.

---

## What the Shared Lib Is For

The shared lib owns:
- Domain models consumed by more than one service or repo
- Shared Pydantic schemas and value objects
- Abstract ports (Protocol interfaces) implemented by multiple consumers
- Cross-cutting utilities (logging setup, config base classes, common guards)

It must **never** import from any of its consumers. Dependency flows one way:
consumers depend on the shared lib, not the reverse.

---

## Adding It as a Dependency

**uv workspace (monorepo):**
```toml
# consuming repo's pyproject.toml
[project]
dependencies = ["shared-lib"]
```

**git dependency (separate repo):**
```toml
[project]
dependencies = [
    "shared-lib @ git+https://github.com/your-org/shared-lib.git@main",
]
```

```bash
uv add shared-lib
```

Pin to a specific tag or commit in production — never track a mutable branch directly.

---

## What It Currently Exports

> Fill in as you explore and add to the lib. Be specific — module paths, types, purpose.

```
shared_lib/
  models/
    # document each model as you discover it
  config/
    # base configuration classes
  ports/
    # shared abstract interfaces
  common/
    # guards, exceptions, logging interface
```

### Known Models
<!-- Example entry format:
- `shared_lib.models.Job` — core job entity; fields: id, name, status, created_at
- `shared_lib.models.JobId` — value object; validated string identifier
-->
_Fill in as the lib is explored._

### Known Ports / Interfaces
<!-- Example:
- `shared_lib.ports.EventEmitter` — abstract interface for emitting domain events
-->
_Fill in as the lib is explored._

### Known Utilities
<!-- Example:
- `shared_lib.common.logging.configure_logger()` — sets up structured logging
- `shared_lib.common.guards.ensure_not_none()` — generic guard utility
-->
_Fill in as the lib is explored._

---

## What Belongs in the Shared Lib

| Add to shared lib | Keep in the consuming repo |
|---|---|
| Domain models used by 2+ repos | Models specific to one service |
| Shared Pydantic settings base classes | Service-specific configuration |
| Abstract ports implemented by 2+ repos | Concrete adapter implementations |
| Cross-cutting utilities used widely | Feature-specific logic |
| Common exceptions (AppError, EntityNotFoundException) | Service-specific exceptions |

**The rule:** if you find yourself copying a model or utility between repos,
it belongs in the shared lib. Do not pre-empt this — move it when the second
real consumer exists, not before.

When uncertain whether something belongs in the shared lib, ask before adding it.
Shared lib additions affect all consumers and warrant deliberate decisions.

---

## Before Writing a New Model or Utility

Before defining anything new in a consuming repo, check here:

1. **Is it already in the shared lib?** Check the Known Models / Utilities sections above.
2. **Does a similar concept exist under a different name?** If yes — use the existing one and update this document with the alias.
3. **Does this concept belong to more than one consumer?** If yes — add it to the shared lib and document it here.
4. **Is it genuinely repo-specific?** Then define it locally — do not add to the shared lib speculatively.

---

## Versioning and Breaking Changes

- Treat the shared lib as a **public API** — backwards compatibility matters
- Prefer **additive changes**: new optional fields, new models
- Breaking changes require: version bump + coordinated update across all consumers
- Never merge a breaking shared lib change without updating all consumers in the same release
- Tag releases — consumers should pin to a version, not track `main`

---

## Known Constraints and Gotchas

> Document anything non-obvious discovered while working with the lib.
> Examples: known bugs, fields that behave unexpectedly, deprecated APIs still in use.

_Fill in as the lib is explored._
