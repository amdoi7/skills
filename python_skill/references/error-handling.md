# Error Handling

## Goals

- Make failure paths explicit and understandable.
- Preserve domain meaning when a request is rejected.
- Translate infrastructure failures at boundaries without losing debugging context.

## Failure Taxonomy

Use one vocabulary across the skill:

| Error Kind | Meaning | Typical Handling |
| --- | --- | --- |
| `InvariantViolation` | The model would become invalid or an impossible state was observed | Prevent with parsing and constructors; if it still happens in trusted code, treat it as a bug or corrupted input |
| `PolicyViolation` | Expected business rejection from a changeable rule | Return a clear business error to the caller |
| `NotFoundError` | Required entity or relation is absent | Translate at the boundary, often to `404` or a domain result |
| `ConcurrencyConflict` | Optimistic lock or stale write conflict | Retry or surface a conflict response |
| `InfrastructureError` | DB, network, file, queue, or SDK failure | Wrap with context, retry selectively, or translate to a safe public message |

## Core Principles

| Principle | Description |
| --- | --- |
| Catch specific | Never bare `except:`; catch only exceptions you can handle |
| Preserve context | Use `raise ... from e` when translating errors |
| Separate invariant from policy | Invalid state and business rejection are not the same failure |
| Catch at boundaries | Handle exceptions where you can recover, retry, or translate |
| Use EAFP with care | Prefer direct operations, but not when failure is expected and expensive |

## Shared Exception Hierarchy

```python
class AppError(Exception):
    """Base class for application-visible failures."""


class DomainError(AppError):
    """Base class for domain-level failures."""


class InvariantViolation(DomainError):
    """State or input makes the model invalid."""


class PolicyViolation(DomainError):
    """Business rule rejection that callers may handle."""


class NotFoundError(DomainError):
    """Expected absence of a required entity or relation."""


class ConcurrencyConflict(DomainError):
    """Write failed because another actor updated the same record."""


class InfrastructureError(AppError):
    """External system failure."""
```

## Preserve Context When Crossing Boundaries

```python
try:
    row = await session.execute(stmt)
except sqlalchemy.exc.SQLAlchemyError as e:
    raise InfrastructureError("load order failed") from e
```

Use `from None` only when the original exception adds no value to the reader.

## Invariants vs Policies

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class Money:
    cents: int

    def __post_init__(self) -> None:
        if self.cents < 0:
            raise InvariantViolation("money cannot be negative")


class CreditPolicy:
    def check(self, total: Money, limit: Money) -> None:
        if total.cents > limit.cents:
            raise PolicyViolation("credit limit exceeded")
```

- `Money(-1)` is an invariant breach.
- "This VIP can exceed the limit during promotion week" is a policy choice.

## Translate at the Boundary

```python
@app.exception_handler(PolicyViolation)
async def policy_violation_handler(request: Request, exc: PolicyViolation) -> JSONResponse:
    return JSONResponse(
        status_code=409,
        content={"error": "policy_violation", "message": str(exc)},
    )


@app.exception_handler(InfrastructureError)
async def infrastructure_handler(request: Request, exc: InfrastructureError) -> JSONResponse:
    logger.exception("Infrastructure failure", exc_info=exc)
    return JSONResponse(
        status_code=503,
        content={"error": "service_unavailable"},
    )
```

## Batch vs Pipeline

Use the failure mode that matches the use case.

```python
def pipeline(data: RawData) -> FinalResult:
    validated = validate(data)
    transformed = transform(validated)
    enriched = enrich(transformed)
    return finalize(enriched)
```

```python
results: list[Result] = []
errors: list[tuple[str, Exception]] = []

for item in items:
    try:
        results.append(process(item))
    except PolicyViolation as e:
        errors.append((item.id, e))
```

## Context Managers and Cleanup

```python
from contextlib import contextmanager


@contextmanager
def managed_resource(name: str):
    resource = acquire_resource(name)
    try:
        yield resource
    except SpecificError as e:
        raise InfrastructureError(f"{name} failed") from e
    finally:
        release_resource(resource)
```

## Async Retries

Retry only exceptions that are transient and well understood.

```python
async def fetch_with_retry(url: str, retries: int = 3) -> dict:
    last_error: Exception | None = None

    for attempt in range(retries):
        try:
            async with asyncio.timeout(10):
                return await fetch(url)
        except TimeoutError as e:
            last_error = e
        except aiohttp.ClientError as e:
            last_error = e

        if attempt < retries - 1:
            await asyncio.sleep(2 ** attempt)

    raise InfrastructureError(f"fetch failed after {retries} attempts") from last_error
```

Do not retry `PolicyViolation`, `InvariantViolation`, or programmer bugs.

## Anti-Patterns

```python
# ❌ Bare except
try:
    do_something()
except:
    pass

# ❌ Hide bugs behind broad recovery
try:
    value = int(user_input)
except Exception:
    value = 0

# ❌ Lose causal chain
try:
    do_something()
except SomeError:
    raise OtherError("failed")

# ✅ Translate specific failures
try:
    value = int(user_input)
except ValueError as e:
    raise InvariantViolation("invalid integer input") from e
```
