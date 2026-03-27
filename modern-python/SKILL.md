---
name: modern-python
description: Modern Python 3.13+ guidance for modeling domain logic clearly, isolating framework concerns, and writing maintainable sync or async code. Use when writing or refactoring Python, reviewing design, shaping module boundaries, building FastAPI services, or when code is drifting into framework-first layers, DTO churn, or over-abstracted services.
---

# Modern Python

Use this skill to keep Python code readable, domain-first, and cheap to evolve.

## Decision Order

1. Model the domain and boundaries clearly
2. Preserve correctness and explicit behavior
3. Localize change and isolate technical dependencies
4. Optimize readability and maintainability
5. Optimize performance after measurement

## Core Principles

- **Domain language first**: prefer names that match business concepts over framework jargon.
- **Contexts before layers**: split by capability or bounded context before introducing `api`, `service`, `repository`, or `models`.
- **Invariants inside, policies outside**: keep state validity in constructors, models, and value objects; keep variable business rules in policy objects or orchestration.
- **Boundary objects stay at boundaries**: DTOs, Pydantic schemas, ORM rows, and HTTP payloads belong at integration edges; do not mirror domain objects without a reason.
- **Prefer explicit code**: avoid decorator magic, implicit side effects, and broad exception swallowing.
- **Default to `loguru`**: use `loguru` for application logging unless the repository already standardizes another logger.
- **One obvious way per codebase**: choose a consistent pattern and apply it everywhere.

## Workflow

1. Determine the primary task:
   - **Modeling, package split, refactor, API shape, or design review**: read [references/philosophy.md](references/philosophy.md), [references/architecture.md](references/architecture.md), and [references/principles.md](references/principles.md).
   - **Exceptions, retries, or boundary translation**: read [references/error-handling.md](references/error-handling.md).
   - **Decorators, wrappers, timing, retry, or FastAPI decorator usage**: read [references/decorators.md](references/decorators.md).
   - **Context managers, cleanup, `ExitStack`, or `AsyncExitStack`**: read [references/contextlib.md](references/contextlib.md).
   - **Async execution or cancellation**: read [references/async-concurrency.md](references/async-concurrency.md).
   - **Tests or review**: read [references/testing.md](references/testing.md) or [references/review-checklist.md](references/review-checklist.md).
2. Load only the reference files needed for the task.
3. Prefer the simplest design that keeps business rules obvious and change localized.

## Quick Reference

| Topic | Use When | Reference |
| --- | --- | --- |
| Design Philosophy | Complexity management, abstraction, design review | [references/philosophy.md](references/philosophy.md) |
| Principles | Naming, decomposition, code review | [references/principles.md](references/principles.md) |
| Architecture | Context boundaries, layering, DI | [references/architecture.md](references/architecture.md) |
| Error Handling | Exception taxonomy, retries, API translation | [references/error-handling.md](references/error-handling.md) |
| Decorators | Typed wrappers, retries, timing, composition order | [references/decorators.md](references/decorators.md) |
| Contextlib | `suppress`, `closing`, `ExitStack`, async cleanup | [references/contextlib.md](references/contextlib.md) |
| Async & Concurrency | `asyncio`, task groups, backpressure | [references/async-concurrency.md](references/async-concurrency.md) |
| Testing | pytest strategy, fakes, coverage | [references/testing.md](references/testing.md) |
| DevOps | `uv`, Docker, deployment workflow | [references/devops.md](references/devops.md) |
| Review | Focused review checklist | [references/review-checklist.md](references/review-checklist.md) |

## Essential Patterns

### Type Hints and Value-Rich Types

```python
from enum import StrEnum

class OrderStatus(StrEnum):
    PENDING = "pending"
    PAID = "paid"

def load_orders(ids: list[int]) -> list["Order"]: ...
```

### Preserve Error Context

```python
try:
    result = risky_operation()
except ValueError as e:
    raise BusinessError("Operation failed") from e
```

### Logging and Cleanup

```python
from loguru import logger
from contextlib import AsyncExitStack

logger.info("Processing order {}", order_id)

async with AsyncExitStack() as stack:
    conn = await stack.enter_async_context(get_connection())
    await handle(conn)
```

## Change Preview Mode

Use this mode when the user asks for `dryrun`, `预演`, `先看方案`, `先不要改文件`, or when the blast radius is unclear.

- Do not modify files.
- Return `Intent`, `Files`, `Patch Preview`, `Validation Plan`, `Risk Notes`, and `Next Action`.
- Prefer minimal, reversible previews and call out assumptions explicitly.

## Tooling

```toml
# pyproject.toml
[tool.ruff]
line-length = 88
select = ["E", "F", "I", "UP", "B", "SIM"]

[tool.ruff.format]
quote-style = "double"
```

```bash
# Format and lint
ruff format .
ruff check --fix .
```
