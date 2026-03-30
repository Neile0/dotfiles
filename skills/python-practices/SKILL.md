---
name: python-practices
description: >
  Python-specific coding standards and idioms for Python 3.12+.
  Covers type hints, Pydantic v2, guards, exceptions, async, tooling, and style.
  Complements coding-practices — engineering principles there apply here too.
  Use whenever writing or reviewing Python code.
---

# Python Practices

Python 3.12+ minimum. These are Python-specific standards only — the broader engineering
principles in `coding-practices` apply here too and are not repeated.

---

## Tooling

| Tool | Purpose |
|---|---|
| `uv` | Package management and virtual environments — always. Never `pip` |
| `pytest` | Testing — TDD first. See `tdd-python` skill |
| `ruff` | Linting and formatting — replaces `flake8`, `isort`, `black` |
| `mypy` | Static type checking |

```bash
uv add <package>        # never: pip install
uv run pytest           # run tests
uv run ruff check .     # lint
uv run mypy .           # type check
```

---

## Type Hints

All code is fully typed. No exceptions.

- Every function signature has typed parameters and a return type, including `-> None`
- Use `X | None` over `Optional[X]`
- Use `X | Y` over `Union[X, Y]`
- Use `collections.abc` for abstractions: `Callable`, `Sequence`, `Mapping`, `Iterator`
- Use `typing.Protocol` for structural interfaces — always prefer over `ABC`
- Use the 3.12+ **type parameter syntax** for generic functions and classes — not the legacy `TypeVar`
- Never use `Any` without a comment explaining why it is genuinely unavoidable

```python
# Good
async def find_job(job_id: JobId) -> Job | None: ...
def process_all(jobs: Sequence[Job]) -> list[JobResult]: ...

# Good — 3.12+ generic syntax
def first[T](items: Sequence[T]) -> T | None: ...

class Repository[T]:
    def get(self, id: str) -> T | None: ...

# 3.12+ type alias
type Vector = list[float]

# Bad
def process(data):               # untyped
def handle(x: Any) -> dict: ...  # Any without justification
T = TypeVar("T")                 # legacy — use 3.12 syntax instead
```

Do not suppress `mypy` errors without understanding them.

---

## Pydantic v2

Pydantic is the default for **all structured data** — domain entities, value objects,
DTOs, configuration, and command/query objects. Do not reach for `dataclass` for anything
that carries domain meaning or has validation requirements.

**Model defaults — always start here:**

```python
from pydantic import BaseModel, ConfigDict

class MyModel(BaseModel):
    model_config = ConfigDict(
        frozen=True,       # immutable by default — prevents accidental mutation
        extra="forbid",    # fail loudly on unexpected input fields
    )
```

Use `extra="ignore"` only on models that consume external API responses you do not control.

**Value Objects via `Annotated`:**

```python
from typing import Annotated
from pydantic import StringConstraints

JobName = Annotated[str, StringConstraints(min_length=1, max_length=100, strip_whitespace=True)]
```

**Rules:**
- Validate in the model with `@field_validator` / `@model_validator` — never in service code
- Use `pydantic-settings` `BaseSettings` for all config — never `os.environ` directly
- Use `SecretStr` for any sensitive value — never plain `str`
- Use `model_dump()` — never `.dict()` (deprecated in v2)
- Use `model_dump(mode="json")` when serialising for JSON output
- Mutable field defaults require `default_factory` — never a bare mutable default

**Configuration pattern:**

```python
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import SecretStr

class AppConfig(BaseSettings):
    model_config = SettingsConfigDict(
        env_prefix="APP_",
        env_file=".env",
        case_sensitive=False,
    )

    database_url: str
    api_key: SecretStr
    worker_concurrency: int = 4
```

Instantiate once at the composition root. Never call `AppConfig()` inside a feature or slice.

---

## Enums and Constants

Always use `Enum` or module-level constants over hardcoded literals. Raw strings and
integers in business logic are a primitive obsession smell.

```python
from enum import StrEnum, auto

class JobStatus(StrEnum):
    PENDING = auto()
    RUNNING = auto()
    COMPLETED = auto()
    FAILED = auto()
```

Use `StrEnum` (3.11+) for string-valued enums — it serialises cleanly with Pydantic and
is comparable directly to strings at boundaries without manual `.value` calls.
Use `IntEnum` for integer-valued enums. Use `Enum` for everything else.

---

## Guard Clauses

Guards are always **raise-based assertions** — they enforce a precondition and raise if
it is not met. They never return a boolean for the caller to check.

Create a `guards.py` per vertical slice for domain-specific guards. Shared, generic
guards live in `common/guards.py`.

```python
# features/job/guards.py
from features.job.models import Job
from features.job.exceptions import InvalidJobStateError

def ensure_job_is_schedulable(job: Job) -> None:
    if job.status != JobStatus.PENDING:
        raise InvalidJobStateError(
            f"Job '{job.id}' cannot be scheduled from status '{job.status}'"
        )

# common/guards.py
def ensure_not_none[T](value: T | None, label: str) -> T:
    if value is None:
        raise ValueError(f"{label} must not be None")
    return value
```

Name all guards with `ensure_` prefix. Call them at the top of a function before any logic.

---

## Exception Hierarchy

Define a shared base in `common/exceptions.py`. Each slice extends it locally in its own
`exceptions.py`. This gives a catchable base for the whole package while keeping
slice-specific errors close to the code that raises them.

```python
# common/exceptions.py
class AppError(Exception):
    """Base for all application errors."""

class EntityNotFoundException(AppError):
    def __init__(self, entity_type: str, entity_id: str) -> None:
        super().__init__(f"{entity_type} '{entity_id}' was not found")
        self.entity_type = entity_type
        self.entity_id = entity_id

class PolicyViolationError(AppError): ...
class InvalidStateTransitionError(AppError): ...

# features/job/exceptions.py
from common.exceptions import AppError

class InvalidJobStateError(AppError): ...
class JobAlreadyExistsError(AppError): ...
```

Never raise a bare `Exception`. The exception type is the documentation.

---

## Imports

Always use **absolute imports**. Never relative imports.

```python
# Good
from features.job.models import Job
from common.guards import ensure_not_none

# Bad
from .models import Job
from ..common.guards import ensure_not_none
```

Import order (enforced by `ruff`): stdlib → third-party → internal.
Do not manually sort — let `ruff` handle it.

---

## Style

`ruff` enforces formatting. These are conventions it does not handle:

- Private methods and attributes: single `_` prefix. Never `__` unless name mangling is
  explicitly required (rare)
- Constants at module level: `UPPER_SNAKE_CASE`
- No type-encoding in names: `job_name` not `str_job_name`, `jobs` not `job_list`
- No single-letter names except loop counters (`i`) or well-known math notation
- No `**kwargs` in domain code — be explicit about what a function accepts
- Do not manually write `__repr__` or `__eq__` on Pydantic models — they are generated
- Do not add `if __name__ == "__main__"` blocks in library or feature code — only in
  explicit entrypoint scripts
- Do not use `@staticmethod` unless the method genuinely has no relationship to the
  instance or class — a module-level function is almost always clearer
- Do not use `@classmethod` as a workaround for complex construction — use a Factory or
  a named constructor function instead

---

## Functional Style

Prefer comprehensions over imperative loops where the intent is clear and the expression
stays readable. Do not compress complex logic into a single comprehension to avoid a loop.

```python
# Good — clear and concise
active_jobs = [job for job in jobs if job.is_active]

# Bad — comprehension obscuring logic that should be a loop or named function
result = [transform(x) for x in items if predicate(x) and other_condition(x) or fallback(x)]
```

Use `functools` where it reduces noise:
- `functools.cache` / `functools.lru_cache` for pure, expensive computations
- `functools.partial` for fixing arguments to produce focused callables
- Avoid `functools.reduce` unless the operation is genuinely a fold — a loop is usually clearer

---

## Async

- `async def` throughout — do not mix sync and async casually
- Use `asyncio.TaskGroup` for structured concurrency — correct cancellation and error propagation
- Use `asyncio.gather` when you need return values from a bounded set of coroutines; pair with
  `asyncio.Semaphore` for concurrency control
- Use `asyncio.timeout()` context manager — never `asyncio.wait_for`
- Always re-raise `asyncio.CancelledError` — never swallow it
- Wrap all blocking I/O in `asyncio.to_thread` — never block the event loop
- Use `contextlib.asynccontextmanager` for async resource management

```python
# TaskGroup — structured background tasks, correct cancellation
async with asyncio.TaskGroup() as tg:
    tg.create_task(fetch_metrics())
    tg.create_task(poll_queue())

# gather + Semaphore — collecting return values with concurrency control
async def fetch_all(urls: list[str], limit: int = 10) -> list[Response]:
    semaphore = asyncio.Semaphore(limit)

    async def fetch_one(url: str) -> Response:
        async with semaphore:
            return await fetch(url)

    return await asyncio.gather(*[fetch_one(u) for u in urls])

# Good — injectable clock for testable async code
async def is_within_window(self, job: Job, clock: TimeProvider) -> bool:
    return clock.now() < job.deadline
```

Configure `asyncio_mode = "auto"` in `pyproject.toml` so async tests need no decorator:

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
```

---

## Testing

Tests are written **before** implementation. See `tdd-python` skill for the full workflow.

Quick reference:
- Tests live **alongside the code** they cover, inside the same feature directory
- Test names describe behaviour: `test_job_fails_when_deadline_exceeded`
- Fixtures in `conftest.py` — never `setUp` / `tearDown`
- Mock at the boundary (adapters, infrastructure) — never domain logic
- Prefer in-memory fakes over `MagicMock` — fakes are honest about the contract
- Mark integration tests: `@pytest.mark.integration`

---

## Paths and File I/O

Always use `pathlib.Path` — never string concatenation or `os.path`:

```python
from pathlib import Path

# Good
config_path = Path("data") / "config" / "settings.json"
content = config_path.read_text(encoding="utf-8")
Path("output.txt").write_text(result, encoding="utf-8")

# Bad
path = os.path.join("data", "config", "settings.json")  # never
```

`Path` is cross-platform, composable, and expressive. Treat it as the default for
any file or directory operation.

---

## Resource Management

Always use context managers for anything that requires cleanup — files, connections,
locks, HTTP clients:

```python
# File — prefer Path methods for simple reads/writes (no explicit open needed)
content = Path("data.txt").read_text(encoding="utf-8")

# When a handle is needed
with open(path, encoding="utf-8") as f:
    for line in f:
        process(line)

# Async resources
async with asynccontextmanager_resource() as resource:
    await resource.do_work()
```

Never rely on garbage collection for resource teardown.
Define async context managers with `contextlib.asynccontextmanager`.

---

## Docstrings

Public functions, methods, and classes that are not self-evident from their signature
and name get a docstring. Internal helpers and anything obvious do not.

Keep docstrings **short and to the point** — one line if possible, a brief paragraph if not.
Do not restate the function name. Do not describe the obvious. Explain what is not clear
from the signature alone.

```python
# Good — one line, adds information the signature doesn't give
def schedule_job(job: Job, clock: TimeProvider) -> ScheduledJob:
    """Schedule a job, rejecting it if outside the allowed processing window."""

# Good — brief, explains the non-obvious constraint
class RetryPolicy:
    """Encapsulates retry behaviour for a job class.

    Exponential backoff is applied between attempts. Max delay is capped
    regardless of attempt count to prevent unbounded wait times.
    """

# Bad — restates the obvious, adds no value
def get_job(job_id: JobId) -> Job:
    """Gets a job by its ID.

    Args:
        job_id: The ID of the job.

    Returns:
        The job.
    """
```

Use Google-style for the rare case where args or exceptions need documenting —
but prefer a clearer signature over a long docstring.

---

## Key Principles

1. **Typed by default** — untyped code is unfinished code
2. **Validate at the boundary** — never deep inside logic
3. **Explicit over implicit** — clear intent over cleverness
4. **Errors should never pass silently** — handle or propagate, always
5. **Flat is better than nested** — guard clauses over conditional pyramids
6. **Hard to test means wrong design** — fix the code, not the test

---



- `global` state — pass dependencies explicitly
- Mutable module-level state — it makes testing impossible
- `try/except Exception` — always catch the specific type
- Bare `raise Exception("message")` — define meaningful exception types
- `isinstance` checks in business logic — use polymorphism or `Protocol`
- Silently catching and ignoring exceptions — always at minimum log them
- Importing from a slice's internals directly — use its `__init__.py` public interface
