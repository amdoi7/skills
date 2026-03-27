# Design Philosophy

## Table of Contents

- [Central Thesis](#central-thesis)
- [Symptoms of Complexity](#symptoms-of-complexity)
- [Core Design Principles](#core-design-principles)
- [Python Design Moves](#python-design-moves)
- [Error Semantics](#error-semantics)
- [Comments and Naming](#comments-and-naming)
- [Review Red Flags](#review-red-flags)
- [Design It Twice](#design-it-twice)

## Central Thesis

The main job of software design is to manage complexity.

In Python, complexity often shows up as code that is easy to write once but hard to understand,
change, or trust later. The goal is not fewer lines; the goal is lower cognitive load.

Prefer designs that:

- localize change
- make the common path simple
- hide policy and mechanism behind small interfaces
- make behavior obvious without reading five files

## Symptoms of Complexity

Watch for these three symptoms during reviews:

### Change Amplification

A small behavior change requires edits in many files, layers, or branches.

Typical Python examples:

- adding a new status requires router, service, repository, serializer, and template edits
- introducing a new source means extending multiple `if/elif` chains
- validation rules are duplicated in handlers, services, and models

### Cognitive Load

The reader must keep too much in their head to make a safe change.

Typical Python examples:

- a function needs many unrelated parameters or `**kwargs`
- a class has several mode flags that change behavior
- service methods depend on hidden conventions instead of explicit types

### Unknown Unknowns

It is unclear where to look, what invariants hold, or what code paths are affected.

Typical Python examples:

- broad exceptions hide the real failure boundary
- vague names like `data`, `info`, `helper`, `manager`, or `process`
- "magic" decorators or metaprogramming that alter control flow invisibly

## Core Design Principles

### Prefer Deep Modules

A deep module has a small interface and hides meaningful implementation complexity.

Good:

- `UserService.get_user(user_id)` hides caching, repository access, and not-found semantics
- `Settings()` hides env parsing, defaults, and normalization

Risky:

- wrappers that only forward arguments
- helper classes with many configuration switches but little behavior

### Hide Design Decisions, Not Just Code

Encapsulation is about hiding knowledge.

If multiple modules must know the same validation rule, retry policy, naming convention, or status
mapping, that decision has leaked and future changes will amplify.

Prefer one owner for each important decision:

- schemas own input shape
- domain types own invariants
- repositories own persistence mechanics
- services own business workflow

### Separate Layers by Abstraction

Different layers should do different work.

Warning signs:

- a service method that just forwards to a repository
- a router that contains business rules
- a repository that returns half-processed domain decisions

If two adjacent layers expose nearly the same interface, the split may be temporal rather than
architectural.

### Pull Complexity Downward

When complexity is unavoidable, absorb it inside the module instead of pushing it to every caller.

Prefer:

- one parsing function that returns a trusted type
- one helper that normalizes optional inputs
- one boundary adapter that converts third-party errors into domain errors

Avoid APIs that force every caller to remember subtle preconditions.

## Python Design Moves

### Parse at the Boundary

Convert untrusted input into strong, trustworthy shapes early.

```python
from enum import StrEnum


class Status(StrEnum):
    ACTIVE = "active"
    INACTIVE = "inactive"


def parse_status(value: str) -> Status:
    return Status(value)
```

After parsing, internal code can trust `Status` instead of repeatedly checking raw strings.

Useful boundary tools:

- `pydantic` models for request parsing
- `StrEnum` for closed sets of values
- `pathlib.Path` instead of raw path strings
- small value objects when invariants matter

### Replace Mode Switches with Explicit Types

This is often shallow design:

```python
class Loader:
    def __init__(self, source: str, **kwargs): ...
```

Prefer separate implementations behind a small protocol or factory when behaviors diverge.

### Flatten Control Flow

Prefer guard clauses and extracted helpers over nested "happy path" pyramids.

```python
def process(order: Order | None) -> Receipt:
    if order is None:
        raise ValueError("order is required")
    if not order.is_ready():
        raise OrderNotReadyError(order.id)
    return charge(order)
```

Flat control flow makes invariants and exit points obvious.

### Avoid Pass-Through Abstractions

Be suspicious of methods like:

```python
async def get_user(self, user_id: int) -> User:
    return await self.repo.get_user(user_id)
```

If a layer adds no policy, validation, translation, or simplification, it may not deserve to exist.

## Error Semantics

Error handling is a design problem, not just a syntax problem.

Prefer to reduce the number of exceptions callers must think about.

Good strategies:

- redefine operations so common edge cases are not errors
- catch and translate low-level exceptions at boundaries
- aggregate related failures in one place

Examples:

- `discard(key)` semantics are often simpler than `remove(key)` if missing keys are normal
- parsing a request once is simpler than repeating `if value is None` checks everywhere
- wrapping third-party exceptions in domain errors reduces leakage across modules

Do not hide real failures. The goal is to remove accidental complexity, not observability.

## Comments and Naming

### Comments

Comments should explain information the code does not show:

- why this approach exists
- what invariant must hold
- what side effect or boundary behavior matters

Bad comments restate code. Good comments reduce the need to inspect another file.

### Naming

Names should be precise, stable, and consistent.

Prefer:

- `user_repository` over `repo`
- `normalized_email` over `value`
- `fetch_with_retry` over `handle_request`

If something is hard to name, the design may still be blurry.

## Review Red Flags

Use these signals in reviews:

| Signal | Why it Matters |
|--------|----------------|
| Growing `if/elif` chains | New cases require edits in many places |
| Constructor mode switches | One type is hiding several unrelated behaviors |
| Pass-through services | Layers do not provide distinct abstractions |
| Cross-module validation duplication | Information has leaked |
| Broad `except Exception` | Failure boundaries are obscure |
| Vague names | Important intent is hidden |
| Comment repeats code | Documentation adds no information |
| Stringly-typed domain data | Invariants are not carried by the type |
| Deep nesting | Important cases and exits are hard to see |

## Design It Twice

For important interfaces, sketch at least two shapes before coding.

Compare them on:

- simplicity of the common call path
- amount of hidden knowledge required by the caller
- likelihood of change amplification
- whether the abstraction can serve nearby use cases too

The best design is usually the one that makes future readers feel least surprised.

## Related References

- Use [principles.md](principles.md) for general coding guidance.
- Use [architecture.md](architecture.md) for layering and module layout.
- Use [error-handling.md](error-handling.md) for exception tactics.
- Use [review-checklist.md](review-checklist.md) during code review.
