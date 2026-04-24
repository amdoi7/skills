# Design Philosophy

> This is the secondary reference loaded when the problem is structural rather than local.
> Start here, then load only the same-directory tertiary reference that matches the actual symptom.

## When to Use

- Package boundaries feel wrong
- Change amplification is high
- DTO churn or pass-through layers dominate the code path
- Ownership of state, invariants, or decisions is unclear
- A design review needs a sharper frame than ordinary code-style advice

## When Not to Use

- The issue is still local to one function, file, or type
- The real problem is error semantics, testing, context, or synchronization
- The user only wants a small implementation fix

## Routing Table

| If the structural problem is mainly about… | Load |
| --- | --- |
| boundary parsing, strong types, DTO vs domain, trusted internal data | [philosophy-boundaries.md](philosophy-boundaries.md) |
| special cases, deep nesting, awkward control flow, ugly local shape | [philosophy-good-taste.md](philosophy-good-taste.md) |
| deep modules, information hiding, change amplification, pass-through layers | [philosophy-complexity.md](philosophy-complexity.md) |
| design review order, red flags, small CLs, review checklist | [philosophy-review.md](philosophy-review.md) |

## Escalation Test

Before changing architecture, answer these in order:

1. Is there a concrete structural symptom, not just aesthetic discomfort
2. Does the symptom cross a function or file boundary
3. Would a local type, naming, or test fix be insufficient
4. Can the smallest boundary change remove the symptom

If the answer to 1 or 2 is no, do not escalate yet.

## Default Structural Posture

- Prefer capability-first decomposition over layer-first decomposition
- Prefer deep modules over thin pass-through layers
- Keep boundary translation at the boundary
- Keep invariants inside domain types and constructors
- Make common paths simple even if implementation gets harder internally
- Redesign only when the symptom is persistent and observable

## Capability-First Architecture

Default posture: capability-first and implementation-first, not layer-first.

```text
                 ┌───────────────────┐
                 │   cmd/<app>/      │
                 │ thin entry, wire  │
                 └─────────┬─────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────┐
│ internal/                                            │
│                                                      │
│  ┌──────────────────┐      ┌──────────────────┐      │
│  │ auth/            │      │ billing/         │      │
│  │ same capability  │      │ same capability  │      │
│  │ keeps boundary   │      │ keeps boundary   │      │
│  │ + rules together │      │ + rules together │      │
│  └──────────────────┘      └──────────────────┘      │
│                                                      │
│  inside one capability, keep related code together   │
│  instead of forcing empty handler/service/repo tiers │
└──────────────────────────────────────────────────────┘
                           │
                           ▼
                 ┌───────────────────┐
                 │ pkg/              │
                 │ only if exported  │
                 │ outside the repo  │
                 └───────────────────┘
```

- Split by business capability before splitting into `handler` / `service` / `repository` shells.
- Keep `cmd/` thin; put reusable application code under `internal/`.
- Use `pkg/` only when code is intentionally for external consumers.
- Prefer manual wiring first; introduce DI frameworks only when construction and lifecycle pain are real.
- Prefer deep modules that hide real decisions over pass-through layers that rename calls.
- If the user asks for architecture or layout work explicitly, summarize the current shape, name the structural symptom, and propose the smallest boundary change that fixes it.
- For blank repo scaffolding rather than structural review, use [project-bootstrap.md](project-bootstrap.md).

## Review Order

1. Boundary clarity
2. Ownership of invariants and state
3. Interface depth and information hiding
4. Change amplification risk
5. Test seam quality

## Quick Smell List

- `handler/service/repository` becomes the main package shape by default
- multiple packages know the same low-level rule
- a type is hard to name because its responsibility is fuzzy
- one feature change requires edits in many layers with the same abstraction
- tests must boot half the system to exercise one behavior

Load the child reference that matches the smell. Do not read all four by reflex.
