---
name: async-worker
description: >
  Asyncio-based service worker patterns for Python 3.12+.
  Covers worker lifecycle, structured concurrency, cancellation, timeouts,
  blocking I/O, retry, and signal handling.
  Use when writing or reviewing any async background processing code.
---

# Async Worker Patterns

The worker is `asyncio`-based throughout. These patterns ensure correct lifecycle
management, clean shutdown, and resilient error handling under concurrent load.

---

## Worker Lifecycle

Every worker has four explicit phases: **startup → run → shutdown → cleanup**.
Never rely on garbage collection for resource teardown — be explicit.

```python
import asyncio
import signal

class ServiceWorker:
    def __init__(self, config: AppConfig) -> None:
        self._config = config
        self._shutdown_event = asyncio.Event()

    async def startup(self) -> None:
        """Initialise connections, caches, subscriptions."""
        ...

    async def run(self) -> None:
        """Main loop — runs until shutdown is signalled."""
        async with asyncio.TaskGroup() as tg:
            tg.create_task(self._process_loop())
            tg.create_task(self._health_check_loop())

    async def shutdown(self) -> None:
        """Signal the worker to stop after current work completes."""
        self._shutdown_event.set()

    async def cleanup(self) -> None:
        """Release resources — close connections, flush buffers."""
        ...

    async def _process_loop(self) -> None:
        while not self._shutdown_event.is_set():
            try:
                await self._process_next()
            except asyncio.CancelledError:
                raise  # always re-raise — never swallow
            except Exception:
                # log and continue — one bad message must not kill the worker
                ...

    async def _process_next(self) -> None:
        ...
```

---

## Entrypoint and Signal Handling

The entrypoint is the composition root for the worker. Wire concrete dependencies here.
`SIGTERM` and `SIGINT` both trigger graceful shutdown.

```python
async def main() -> None:
    config = AppConfig()
    worker = ServiceWorker(config=config)
    loop = asyncio.get_running_loop()

    def _handle_signal() -> None:
        asyncio.create_task(worker.shutdown())

    for sig in (signal.SIGTERM, signal.SIGINT):
        loop.add_signal_handler(sig, _handle_signal)

    await worker.startup()
    try:
        await worker.run()
    finally:
        await worker.cleanup()


# Only in an explicit entrypoint script — never in library or feature code
if __name__ == "__main__":
    asyncio.run(main())
```

---

## Structured Concurrency

Use `asyncio.TaskGroup` for concurrent tasks where correct error propagation matters.
If one task fails, sibling tasks are cancelled automatically.

```python
# Good — TaskGroup: correct cancellation and error propagation
async with asyncio.TaskGroup() as tg:
    tg.create_task(fetch_metrics())
    tg.create_task(poll_queue())
```

Use `asyncio.gather` when you need the **return values** from a bounded set of
coroutines. Pair with `asyncio.Semaphore` to control concurrency:

```python
# gather + Semaphore — collecting results with concurrency control
async def process_all(items: list[Item], limit: int = 10) -> list[Result]:
    semaphore = asyncio.Semaphore(limit)

    async def process_one(item: Item) -> Result:
        async with semaphore:
            return await process(item)

    return await asyncio.gather(*[process_one(i) for i in items])
```

---

## Timeouts

Use `asyncio.timeout()` context manager — never `asyncio.wait_for`:

```python
async def process_with_timeout(message: Message) -> Result:
    try:
        async with asyncio.timeout(30.0):
            return await _process(message)
    except TimeoutError:
        raise ProcessingTimeoutError(
            f"Processing exceeded 30s for message {message.id}"
        )
```

`asyncio.timeout()` is composable — it stacks cleanly with `TaskGroup` and
other context managers. `wait_for` does not compose as well.

---

## Cancellation

**Always re-raise `CancelledError`.** Swallowing it breaks structured concurrency
and prevents clean shutdown.

```python
async def do_work() -> None:
    try:
        await long_running_operation()
    except asyncio.CancelledError:
        await cleanup_partial_work()  # cleanup is fine
        raise                          # re-raise is mandatory
    finally:
        await release_lock()           # finally always runs — safe for teardown
```

Use `asyncio.shield` only to protect a critical coroutine from external cancellation
— for example, a final checkpoint write during shutdown:

```python
await asyncio.shield(persist_checkpoint(state))
```

Treat `shield` as an exception that needs a comment explaining why it is needed.

---

## Blocking I/O

Never block the event loop. Offload every synchronous blocking call to a thread:

```python
from pathlib import Path

async def read_config(path: Path) -> str:
    return await asyncio.to_thread(path.read_text, encoding="utf-8")
```

Operations that always need `to_thread`:
- File I/O (unless using `aiofiles`)
- Synchronous DB drivers — prefer async drivers (`asyncpg`, `aiosqlite`)
- CPU-bound computation — consider `ProcessPoolExecutor` for heavy work
- Any third-party library without an async API

---

## Retry with Exponential Backoff

Do not reach for a library dependency just for retries. A small helper is sufficient
and keeps the behaviour explicit and auditable.

```python
from collections.abc import Callable, Awaitable

async def with_retry[T](
    fn: Callable[[], Awaitable[T]],
    *,
    attempts: int = 3,
    base_delay: float = 1.0,
    max_delay: float = 30.0,
) -> T:
    """Retry fn with exponential backoff. Never retries on CancelledError."""
    last_exc: Exception | None = None
    for attempt in range(attempts):
        try:
            return await fn()
        except asyncio.CancelledError:
            raise
        except Exception as exc:
            last_exc = exc
            delay = min(base_delay * 2 ** attempt, max_delay)
            await asyncio.sleep(delay)
    raise last_exc  # type: ignore[misc]
```

Uses 3.12+ generic function syntax (`[T]`). `CancelledError` is never retried —
it propagates immediately.

---

## Task References

Always keep a reference to created tasks. Tasks without a reference can be silently
garbage collected before they complete.

```python
# Bad — task can be GC'd
asyncio.create_task(background_job())

# Good — reference kept
self._tasks: set[asyncio.Task] = set()
task = asyncio.create_task(background_job())
self._tasks.add(task)
task.add_done_callback(self._tasks.discard)
```

---

## Testing Workers

Test worker behaviour through its **public interface** — not private methods.
Use `asyncio.Event` to coordinate timing. Never `asyncio.sleep`.

```python
async def test_worker_processes_message_and_emits_metric(
    worker: ServiceWorker,
    fake_queue: FakeQueue,
    fake_metrics: FakeMetricEmitter,
) -> None:
    # Arrange
    message = MessageBuilder().build()
    await fake_queue.enqueue(message)

    processed = asyncio.Event()
    fake_metrics.on_emit(lambda _: processed.set())

    # Act
    await worker.startup()
    await asyncio.wait_for(processed.wait(), timeout=2.0)
    await worker.shutdown()
    await worker.cleanup()

    # Assert
    assert fake_metrics.emitted_count("message.processed") == 1
```

Configure `asyncio_mode = "auto"` in `pyproject.toml` — see `python-testing` skill.

---

## What NOT to Do

- Never swallow `CancelledError` — always re-raise
- Never block the event loop — use `to_thread` for any synchronous I/O
- Never create tasks without keeping a reference
- Never use `asyncio.sleep(0)` as a yield point in production — it is a test smell
- Never use `loop.run_until_complete` inside an already-running event loop
- Never mix sync and async in the same call path without `to_thread`
