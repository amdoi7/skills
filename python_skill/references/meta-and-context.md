# Meta Programming & Context Management

> Unified guide for decorators + contextlib patterns.

## Goals

- Keep cross-cutting logic reusable and explicit
- Guarantee resource cleanup in success/failure/cancellation paths
- Avoid hidden control-flow and decorator overuse

## Quick Selector

| Scenario | Prefer |
|---|---|
| Add cross-cutting behavior (timing/retry/validation) | Decorator |
| Manage open/close/commit/rollback lifecycle | Context manager (`with`) |
| Dynamic number of resources | `ExitStack` / `AsyncExitStack` |
| Optional context | `nullcontext` |
| Ignore expected narrow exception | `suppress` |
| Adapt legacy object with `close()` | `closing` |

## Decorator Patterns

### 1) Baseline typed decorator

```python
import functools
from typing import Callable, ParamSpec, TypeVar

P = ParamSpec("P")
R = TypeVar("R")

def traced(func: Callable[P, R]) -> Callable[P, R]:
    @functools.wraps(func)
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        # pre
        result = func(*args, **kwargs)
        # post
        return result
    return wrapper
```

### 2) Parameterized decorator

```python
def retry(max_attempts: int = 3):
    def decorator(func: Callable[P, R]) -> Callable[P, R]:
        @functools.wraps(func)
        async def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
            last_error: Exception | None = None
            for i in range(max_attempts):
                try:
                    return await func(*args, **kwargs)
                except Exception as e:
                    last_error = e
                    if i < max_attempts - 1:
                        await asyncio.sleep(2 ** i)
            raise last_error
        return wrapper
    return decorator
```

### 3) Composition order

```python
@timed
@retry(3)
@validate_input
async def process(data: dict) -> Result:
    ...

# Equivalent: process = timed(retry(3)(validate_input(process)))
```

## Contextlib Patterns

### 1) `@contextmanager`

```python
from contextlib import contextmanager

@contextmanager
def db_transaction(conn):
    tx = conn.begin()
    try:
        yield tx
        tx.commit()
    except Exception:
        tx.rollback()
        raise
```

### 2) `ExitStack` for dynamic resources

```python
from contextlib import ExitStack

with ExitStack() as stack:
    files = [stack.enter_context(open(p)) for p in paths]
    for f in files:
        process(f)
```

### 3) Utility helpers

```python
from contextlib import suppress, closing, nullcontext

with suppress(FileNotFoundError):
    os.remove("temp.tmp")

with closing(urlopen(url)) as resp:
    data = resp.read()

cm = open(path) if path else nullcontext(default_data)
with cm as source:
    handle(source)
```

### 4) Async resource stack

```python
from contextlib import AsyncExitStack

async with AsyncExitStack() as stack:
    conn = await stack.enter_async_context(get_db_connection())
    stack.callback(sync_cleanup)
    stack.push_async_callback(async_cleanup)
    await do_work(conn)
```

## Common Pitfalls

- Missing `@functools.wraps` (metadata lost)
- Using sync decorator on async function (or opposite)
- Swallowing broad exceptions in wrappers
- Heavy business logic hidden in decorators
- Overusing `suppress` (masking real bugs)
- Forgetting cancellation semantics in async wrappers

## Review Checklist

- [ ] Decorator intent is obvious from name
- [ ] Wrapper preserves type and metadata
- [ ] Exception handling is narrow and explicit
- [ ] Cleanup always happens (`finally` / context manager)
- [ ] Async code does not block event loop
- [ ] Resource ownership is clear (who closes what)

## Migration Note

This file consolidates guidance previously split across:
- `references/decorators.md`
- `references/contextlib.md`
