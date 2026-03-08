---
name: modern-python
description: Modern Python 3.13+ development guidance for writing clean, maintainable, and performant code. Use when writing new Python code, refactoring, reviewing, designing APIs, working with async/concurrency, or setting up Python projects. Triggers on .py files, pyproject.toml, Python-related questions, FastAPI, asyncio, pytest, uv, ruff discussions.
---

# Modern Python

Concise guidance for writing modern Python 3.13+ code that is clean, maintainable, and performant.

## Decision Order

1. Correctness and clear boundaries
2. Readability and maintainability
3. Extensibility and evolution cost
4. Performance (profile first, optimize last)

## Core Principles

- **KISS & YAGNI**: Simple solutions, no speculative features
- **SOLID**: Single responsibility, open/closed, dependency inversion
- **Prefer stdlib**: Use `pathlib`, `dataclasses`, `zoneinfo` before external deps
- **One obvious way**: Avoid multiple styles for the same task

## Quick Reference

| Topic               | When to Use                              | Reference                                                          |
| ------------------- | ---------------------------------------- | ------------------------------------------------------------------ |
| Principles          | Design decisions, code review            | [references/principles.md](references/principles.md)               |
| Architecture        | Project structure, layering, DI          | [references/architecture.md](references/architecture.md)           |
| Error Handling      | Exceptions, EAFP, error context          | [references/error-handling.md](references/error-handling.md)       |
| Meta & Context      | Decorators + contextlib + resource scope | [references/meta-and-context.md](references/meta-and-context.md)   |
| Async & Concurrency | asyncio, threading, pitfalls             | [references/async-concurrency.md](references/async-concurrency.md) |
| Testing             | pytest, mocking, coverage                | [references/testing.md](references/testing.md)                     |
| DevOps              | uv, Docker, deployment                   | [references/devops.md](references/devops.md)                       |
| Review              | Code review checklist                    | [references/review-checklist.md](references/review-checklist.md)   |

## Dryrun Update Mode (inspired by piglet)

Use this mode when user asks for `dryrun`, `预演`, `先看方案`, `先不要改文件`, or when change impact is unclear.

### Dryrun workflow

1. **Scope first**: Identify target files and expected behavior changes.
2. **Apply decision order**:
   1. Correctness and explicit behavior
   2. Readability and maintainability
   3. Extension cost and change isolation
   4. Performance and micro-optimizations
3. **Simulate edits only**: Provide proposed diffs/snippets without writing files.
4. **Run check plan only**: Show commands/tests that would be run after real edit.
5. **Request confirmation**: Ask for explicit go-ahead before any real file modification.

### Dryrun output contract

Return results in this structure:

- **Intent**: what will be changed and why
- **Files**: impacted files list
- **Patch Preview**: before/after snippets or unified diff
- **Validation Plan**: lint/test/type-check commands
- **Risk Notes**: edge cases, compatibility, rollback notes
- **Next Action**: `Apply`, `Revise`, or `Cancel`

### Safety rules

- Do not modify files in dryrun mode.
- Prefer minimal, reversible changes in preview.
- Highlight uncertainty and assumptions explicitly.
- If requirements are ambiguous, ask clarifying questions before proposing patch.

## Essential Patterns

### Type Hints (3.13+)

```python
# Use builtin generics, not typing module
def process(items: list[str]) -> dict[str, int]: ...

# New generic syntax
def first[T](items: list[T]) -> T | None:
    return items[0] if items else None

# Override decorator for clarity
from typing import override

class Child(Parent):
    @override
    def method(self) -> None: ...
```

### Error Handling

```python
# Catch specific exceptions only
try:
    result = risky_operation()
except ValueError as e:
    logger.error("Invalid value: %s", e)  # NOT f-string in logs
    raise BusinessError("Operation failed") from e

# Early return pattern
def process(data: Data | None) -> Result:
    if data is None:
        return Result.empty()
    if not data.is_valid():
        raise ValidationError("Invalid data")
    return do_processing(data)
```

### Logging (loguru)

```python
from loguru import logger

# Use lazy formatting, NOT f-strings
logger.info("Processing user {}", user_id)

# Expensive operations: use lazy evaluation
logger.opt(lazy=True).debug("Data: {}", lambda: expensive_serialize(data))

# Structured context
logger.bind(request_id=req_id).info("Request started")
```

### Configuration

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str
    debug: bool = False

    model_config = {"env_prefix": "APP_"}

settings = Settings()
```

## Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|----------|
| `def f(x=[])` | Mutable default shared | `def f(x=None): x = x or []` |
| `is` vs `==` | Identity vs equality | Use `==` for values, `is` for `None` |
| `[[]] * n` | Shared references | `[[] for _ in range(n)]` |
| Loop lambda | Late binding | `lambda x=x: ...` or `functools.partial` |
| f-string in logs | Eager evaluation | Use `%s` or `{}` placeholders |

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
