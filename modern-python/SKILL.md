---
name: modern-python
description: >
  Use for Python application code work: implementing, debugging, refactoring, or reviewing modules, FastAPI/service boundaries, public SDK entry paths, async/sync behavior, exception semantics, tests, packaging checks, and Python-facing docs tied to code behavior.
---

# Modern Python

Keep Python code explicit, boundary-aware, and cheap to evolve.

## Routing Guardrails

- Route out when the task is mainly about CPython internals, bytecode, GIL, GC, or VM implementation.
- Follow repository conventions first when they conflict with generic examples in this skill.
- Prefer the smallest guidance that makes the next change safer and clearer.

## Task Modes

- **implementation** — make the smallest clear code change
- **review** — return concrete findings and minimal fixes
- **debug** — locate the failure boundary and preserve context
- **design** — simplify boundaries, public entry paths, and change locality

## Decision Order

1. Clarify the boundary and the user-visible behavior
2. Keep the common path obvious
3. Preserve correctness and explicit failure semantics
4. Localize change and isolate technical dependencies
5. Keep docs, tests, and public entry paths aligned with real behavior
6. Optimize performance only after measurement

## Core Rules

- Prefer domain language over framework jargon.
- Split by capability before adding generic `service`, `repository`, or `models` layers.
- Keep invariants in constructors, models, and value-rich types.
- Keep policy choices in orchestration or policy objects.
- Keep transport schemas, ORM rows, and DTOs at the boundary unless they truly are the model.
- Keep public entry paths short for the common case, and expose deeper control explicitly.
- Prefer explicit code over hidden side effects, broad exception swallowing, and magic decorators.
- Keep examples instructional and tests assertive. Do not let one replace the other.
- Keep sync and async surfaces behaviorally aligned when both are public.
- Keep release checks small, strong, and tied to user-visible promises.
- Prefer `loguru` for application logging unless the repository has already standardized another logger.
- Follow repository conventions for tooling and test style.

## Anti-Patterns

- Do not wrap business flow in broad `except Exception` blocks that return `None`, `False`, or empty containers.
- Do not hide required input behind `or` defaults, broad `.get()` fallbacks, or excessive optional chaining.
- Do not clone `api -> service -> repository` pass-through layers unless each layer hides a real decision.
- Do not mirror every Pydantic schema, ORM row, DTO, and domain type when the contracts do not actually differ.
- Do not hide I/O, retries, transactions, or validation in decorators that make entry behavior surprising.
- Do not claim async/sync parity, cancellation safety, or release readiness without tests that exercise those paths.

## Workflow

1. Identify the primary need.
   - **Modeling, refactor, package split, API shape, public entry design, or design review**: read [references/philosophy.md](references/philosophy.md), [references/architecture.md](references/architecture.md), and [references/principles.md](references/principles.md).
   - **Exceptions, retries, boundary translation, or failure semantics**: read [references/error-handling.md](references/error-handling.md).
   - **Decorators, wrappers, or public entry ergonomics**: read [references/decorators.md](references/decorators.md) and [references/principles.md](references/principles.md).
   - **Cleanup, `ExitStack`, `AsyncExitStack`, or resource lifetimes**: read [references/contextlib.md](references/contextlib.md).
   - **Async execution, cancellation, backpressure, or sync and async parity**: read [references/async-concurrency.md](references/async-concurrency.md).
   - **Tests, examples, smoke checks, release gates, or review**: read [references/testing.md](references/testing.md) and [references/review-checklist.md](references/review-checklist.md).
   - **CI, release gates, or compatibility matrices**: read [references/devops.md](references/devops.md).
2. Load only the reference files needed for the task.
3. Prefer the smallest design that keeps behavior obvious and change localized.

## Quick Reference

| Topic | Use When | Reference |
| --- | --- | --- |
| Design Philosophy | Complexity management, abstraction, design review | [references/philosophy.md](references/philosophy.md) |
| Principles | Naming, public entry paths, decomposition, docs guidance | [references/principles.md](references/principles.md) |
| Architecture | Context boundaries, layering, DI, public facade vs deeper control | [references/architecture.md](references/architecture.md) |
| Error Handling | Exception taxonomy, retries, API translation | [references/error-handling.md](references/error-handling.md) |
| Decorators | Typed wrappers, retries, timing, composition order | [references/decorators.md](references/decorators.md) |
| Contextlib | `suppress`, `closing`, `ExitStack`, async cleanup | [references/contextlib.md](references/contextlib.md) |
| Async & Concurrency | `asyncio`, task groups, backpressure, sync and async parity | [references/async-concurrency.md](references/async-concurrency.md) |
| Testing | pytest strategy, examples vs tests, smoke-feature-integration-release layers | [references/testing.md](references/testing.md) |
| DevOps | `uv`, CI, release gate, compatibility matrix | [references/devops.md](references/devops.md) |
| Review | Focused review checklist | [references/review-checklist.md](references/review-checklist.md) |

## Essential Patterns

### Keep the Common Entry Short, Keep Deeper Control Explicit

```python
client = connect()

config = Config(timeout=5, retries=2)
client = Client(config)
```

### Preserve Error Context

```python
try:
    result = risky_operation()
except ValueError as e:
    raise BusinessError("operation failed") from e
```

### Keep Dual Surfaces Aligned

```python
data = fetch_user(user_id)
data = await fetch_user_async(user_id)
```

## Output Contract

Use this shape for review and design tasks.

- **Finding** — identify the concrete problem
- **Why it matters** — explain the behavioral or maintenance cost
- **Minimal change** — propose the smallest fix worth making
- **Validation** — state how to verify the fix

## Change Preview Mode

Use this mode when the user asks for `dryrun`, `预演`, `先看方案`, `先不要改文件`, or when the blast radius is unclear.

- Do not modify files.
- Return `Intent`, `Files`, `Patch Preview`, `Validation Plan`, `Risk Notes`, and `Next Action`.
- Prefer minimal, reversible previews.
- State assumptions explicitly.

## Trigger Eval Prompts

Use these prompts when tuning routing or checking neighboring skill competition.

Should trigger:

- Fix this Python async task cancellation bug; shutdown hangs.
- Review this FastAPI module boundary; the service and repository layers feel like pass-through code.
- Refactor this Python SDK quickstart so sync and async clients expose the same behavior.
- Tighten Python exception handling so DB errors preserve context.

Should stay quiet:

- Rewrite this README to sound less AI.
- Design OpenAPI endpoints and Problem JSON for orders.
- Split this domain into bounded contexts before choosing language.
- Commit these changes as two Conventional Commits.
- Explain CPython GIL internals.
- Draw an ASCII architecture diagram.

## Tooling

```toml
[tool.ruff]
line-length = 88
select = ["E", "F", "I", "UP", "B", "SIM"]

[tool.ruff.format]
quote-style = "double"
```

```bash
ruff format .
ruff check --fix .
```
