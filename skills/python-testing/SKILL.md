---
name: python-testing
description: >
  Python testing standards and practices using pytest.
  Covers TDD workflow, test types, structure, fixtures, fakes vs mocks,
  async testing, parametrize, coverage, and test design principles.
  Replaces tdd-python. Use whenever writing, reviewing, or running tests.
---

# Python Testing

Testing is a design activity, not a verification activity. Tests written after the fact
verify behaviour. Tests written first *shape* the design — they are the first consumer
of the code, and they reveal structural problems before production does.

**If a test is hard to write, the design is wrong — not the test.**

---

## TDD: The Default Workflow

Always follow the TDD cycle. No implementation exists before a failing test does.

```
1. Red    — write a failing test describing the behaviour you want
2. Green  — write the minimum implementation to make it pass. No more.
3. Refactor — clean up both test and implementation without changing behaviour
```

Never skip Red. A test written after the implementation was never meaningfully failing —
it gives false confidence.

**The discipline:** resist writing more implementation than the current failing test
demands. The next test will pull out the next behaviour. Trust the cycle.

---

## Test Structure

Tests live in `tests/`, mirroring the `src/` directory exactly:

```
src/
  moduleA/
    features/
      job/
        service.py
        model.py

tests/
  moduleA/
    features/
      job/
        test_service.py
        test_model.py
```

Shared test utilities, fakes, and fixtures live in `tests/common/` — not scattered
across individual test files.

```
tests/
  common/
    fakes/            // in-memory implementations of ports
    builders/         // object builders / test data factories
    fixtures/         // shared pytest fixtures
```

---

## pytest Configuration

```toml
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"           # all async tests run without a decorator
testpaths = ["tests"]
markers = [
    "integration: requires real infrastructure (DB, HTTP, queue)",
    "e2e: full system tests — slow, run in CI only",
]
```

Run commands:

```bash
uv run pytest                          # all tests
uv run pytest -m "not integration"     # unit tests only (fast)
uv run pytest -m "not integration and not e2e"  # exclude all slow tests
uv run pytest --cov=src --cov-report=term-missing  # with coverage
uv run pytest tests/moduleA/features/job/  # single feature
```

---

## Test Types and Markers

### Unit Tests
Fast, isolated, no real I/O. The default. No marker needed.

Test one unit of behaviour in isolation — a service method, a domain rule, a guard.
Infrastructure is replaced with fakes. Tests run in milliseconds.

### Integration Tests
Mark with `@pytest.mark.integration`. Require real infrastructure (DB, HTTP, queue).
Excluded from fast local runs. Run in CI on every push.

```python
@pytest.mark.integration
async def test_job_persisted_to_postgres(db_session: AsyncSession) -> None:
    ...
```

### End-to-End Tests
Mark with `@pytest.mark.e2e`. Full system under test. Slow. Run in CI only.

---

## Naming Tests

Test names describe **behaviour and outcome**, not implementation:

```python
# Good — tells you what the system does and when
def test_job_fails_when_deadline_exceeded() -> None: ...
def test_metric_emitted_on_successful_job_completion() -> None: ...
def test_schedule_rejected_when_priority_is_invalid() -> None: ...

# Bad — tells you nothing about behaviour
def test_run() -> None: ...
def test_job_1() -> None: ...
def test_service() -> None: ...
```

Pattern: `test_<subject>_<expected_outcome>_when_<condition>`

A failing test name should read as a complete bug report.

---

## Arrange, Act, Assert

Structure every test with three explicit phases. Blank lines between phases.

```python
async def test_job_completes_when_processed_successfully(
    service: JobService,
    fake_repo: FakeJobRepository,
) -> None:
    # Arrange
    job = JobBuilder().with_status(JobStatus.PENDING).build()
    await fake_repo.save(job)

    # Act
    result = await service.process(job.id)

    # Assert
    assert result.status == JobStatus.COMPLETED
```

Never mix arrangement and assertion. Never put multiple acts in one test.
One test, one behaviour, one reason to fail.

---

## Fixtures

All shared setup lives in `conftest.py`. Never use `setUp` / `tearDown`.

Fixtures compose — build complex setups from simple ones:

```python
# tests/moduleA/features/job/conftest.py
import pytest
from tests.common.fakes import FakeJobRepository, FakeMetricEmitter
from src.moduleA.features.job.service import JobService

@pytest.fixture
def fake_repo() -> FakeJobRepository:
    return FakeJobRepository()

@pytest.fixture
def fake_metrics() -> FakeMetricEmitter:
    return FakeMetricEmitter()

@pytest.fixture
def service(fake_repo: FakeJobRepository, fake_metrics: FakeMetricEmitter) -> JobService:
    return JobService(repo=fake_repo, metrics=fake_metrics)
```

Use fixture scope deliberately:
- `scope="function"` — default, fresh per test (safest)
- `scope="module"` — shared within a test file
- `scope="session"` — shared across the entire run (use only for expensive setup like DB connections)

---

## Fakes Over Mocks

Prefer **in-memory fakes** over `MagicMock`. Fakes implement the real interface contract.
Mocks assert on calls — which ties tests to implementation, not behaviour.

```python
# tests/common/fakes/fake_job_repository.py
from src.moduleA.features.job.ports import JobRepository
from src.moduleA.features.job.model import Job, JobId

class FakeJobRepository:
    """In-memory implementation of JobRepository for testing."""

    def __init__(self) -> None:
        self._store: dict[JobId, Job] = {}

    async def save(self, job: Job) -> None:
        self._store[job.id] = job

    async def get(self, job_id: JobId) -> Job | None:
        return self._store.get(job_id)

    async def list_pending(self) -> list[Job]:
        return [j for j in self._store.values() if j.status == JobStatus.PENDING]
```

Assert on **state**, not on calls:

```python
# Good — asserts on observable outcome
assert fake_repo.get(job.id) is not None

# Bad — asserts on implementation detail
mock_repo.save.assert_called_once_with(job)
```

Use `MagicMock` / `AsyncMock` only when a fake would require disproportionate effort —
typically third-party libraries or interfaces with many methods where only one is relevant.

---

## Object Builders

Use builders to construct test data. Avoid raw constructors in test bodies — they
couple tests to model signatures and make the intent harder to read.

```python
# tests/common/builders/job_builder.py
from src.moduleA.features.job.model import Job, JobId, JobStatus, JobName

class JobBuilder:
    def __init__(self) -> None:
        self._id = JobId("test-job-id")
        self._name = JobName("test-job")
        self._status = JobStatus.PENDING

    def with_status(self, status: JobStatus) -> "JobBuilder":
        self._status = status
        return self

    def with_id(self, job_id: str) -> "JobBuilder":
        self._id = JobId(job_id)
        return self

    def build(self) -> Job:
        return Job(id=self._id, name=self._name, status=self._status)
```

```python
# Clean, intention-revealing test setup
job = JobBuilder().with_status(JobStatus.FAILED).build()
```

---

## Parametrize

Use `@pytest.mark.parametrize` to test multiple inputs for the same behaviour.
Do not write near-identical tests for each case.

```python
import pytest
from src.moduleA.features.job.guards import ensure_job_is_schedulable
from src.moduleA.features.job.model import JobStatus
from src.moduleA.features.job.exceptions import InvalidJobStateError

@pytest.mark.parametrize("status", [
    JobStatus.RUNNING,
    JobStatus.COMPLETED,
    JobStatus.FAILED,
])
def test_scheduling_rejected_for_non_pending_status(status: JobStatus) -> None:
    job = JobBuilder().with_status(status).build()
    with pytest.raises(InvalidJobStateError):
        ensure_job_is_schedulable(job)
```

Name parametrize IDs when the default representation is unclear:

```python
@pytest.mark.parametrize("input,expected", [
    pytest.param("", False, id="empty_string"),
    pytest.param("  ", False, id="whitespace_only"),
    pytest.param("valid-name", True, id="valid"),
])
def test_job_name_validation(input: str, expected: bool) -> None:
    ...
```

---

## Async Tests

`asyncio_mode = "auto"` is configured — all async test functions run automatically.
No decorator needed.

```python
async def test_worker_processes_queued_job(
    service: JobService,
    fake_repo: FakeJobRepository,
) -> None:
    job = JobBuilder().build()
    await fake_repo.save(job)

    result = await service.process(job.id)

    assert result.status == JobStatus.COMPLETED
```

For async fixtures, mark them `async` and they compose cleanly:

```python
@pytest.fixture
async def seeded_repo(fake_repo: FakeJobRepository) -> FakeJobRepository:
    await fake_repo.save(JobBuilder().build())
    return fake_repo
```

Use `asyncio.Event` to coordinate timing in async tests — never `asyncio.sleep`.

---

## Testing Exceptions

Use `pytest.raises` as a context manager. Assert on the exception type and, where
meaningful, on its attributes — not on the string message.

```python
def test_entity_not_found_error_carries_context() -> None:
    with pytest.raises(EntityNotFoundException) as exc_info:
        raise EntityNotFoundException("Job", "missing-id")

    assert exc_info.value.entity_type == "Job"
    assert exc_info.value.entity_id == "missing-id"
```

Do not assert on exception message strings — they are not part of the public contract
and will cause brittle tests that break on message wording changes.

---

## Testing Pydantic Models

Test validation behaviour directly on the model — not through a service:

```python
import pytest
from pydantic import ValidationError
from src.moduleA.features.job.model import Job, RetryCount

def test_retry_count_rejects_negative_values() -> None:
    with pytest.raises(ValidationError):
        Job(name="test", retries=RetryCount(-1))

def test_job_is_immutable() -> None:
    job = JobBuilder().build()
    with pytest.raises(ValidationError):
        job.name = "changed"  # frozen=True should prevent this
```

---

## Testing Guards

Guards are pure functions — test them directly without a service or fixture:

```python
from src.moduleA.features.job.guards import ensure_job_is_schedulable
from src.moduleA.features.job.exceptions import InvalidJobStateError

def test_ensure_job_is_schedulable_raises_for_running_job() -> None:
    job = JobBuilder().with_status(JobStatus.RUNNING).build()
    with pytest.raises(InvalidJobStateError):
        ensure_job_is_schedulable(job)

def test_ensure_job_is_schedulable_passes_for_pending_job() -> None:
    job = JobBuilder().with_status(JobStatus.PENDING).build()
    ensure_job_is_schedulable(job)  # no exception = pass
```

---

## Coverage

Coverage is a **floor**, not a goal. A test that exists only to hit a line is worthless.

```toml
[tool.coverage.run]
source = ["src"]
omit = ["src/*/adapters.py"]   # integration tested separately

[tool.coverage.report]
fail_under = 80
show_missing = true
```

Targets by layer:
- **Domain logic** (`model`, `service`, `guards`) — ≥ 80% line coverage
- **Adapters** — happy path + at least one failure path, via integration tests
- **Presentation** — thin handlers; coverage less critical than integration tests

Do not chase 100%. Untestable code at the boundary (framework wiring, entrypoints)
is acceptable to exclude. Everything domain-owned should be covered.

---

## What Makes a Good Test

A good test is:
- **Fast** — milliseconds, not seconds. Slow tests do not get run.
- **Isolated** — does not depend on test execution order or shared mutable state
- **Deterministic** — same result every run, on every machine
- **Readable** — a new engineer can understand what it is testing without context
- **Focused** — one behaviour, one reason to fail

A bad test is one that:
- Passes when the code is broken
- Fails when the code is correct
- Requires understanding the implementation to understand the test
- Breaks when unrelated code changes
- Tests multiple behaviours in one function
