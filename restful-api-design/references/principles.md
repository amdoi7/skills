# Principles (Zalando-Inspired, Language-Agnostic)

## Core Principles

1. API First
Define API contract and behavior before implementation.

2. API as Product
Optimize for consumer experience: predictability, clear errors, stable evolution.

3. Consistency Over Local Preference
Keep naming, payload shape, and status semantics consistent across teams/services.

4. Conservative Evolution
Prefer backward-compatible change; version only when compatibility cannot be preserved.

5. Security by Default
Assume hostile traffic and enforce auth, validation, and least privilege everywhere.

## Decision Heuristics

- Contract conflicts with framework default: keep contract, adapt framework.
- Multiple valid designs: choose the option with lower long-term cognitive load.
- Breaking change candidate: try additive extension first.
- Ambiguous behavior: define explicit status codes and error types.

## Non-Goals

- Do not optimize for one language/framework at the cost of contract consistency.
- Do not expose internal storage/domain implementation details as API surface.
