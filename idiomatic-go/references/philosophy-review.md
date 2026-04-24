# Philosophy: Design Review & Red Flags

Use this reference when reviewing structural changes, not just local code style.

## Review Order

1. boundary clarity
2. ownership of invariants and state
3. interface depth and information hiding
4. complexity and change amplification
5. tests and verification seams
6. naming and comments

## Small-Change Bias

Prefer small, self-contained changes.

| Signal | Guidance |
| --- | --- |
| around 100 lines | usually reviewable |
| around 1000 lines | usually too large |
| one change per CL/PR | preferred |

If a change cannot be explained as one coherent design move, split it.

## Red Flags

| Red flag | Why it matters |
| --- | --- |
| shallow module | interface cost is close to implementation cost |
| information leakage | the same decision is known in many places |
| time-based decomposition | layers exist only because A happens before B |
| pass-through methods | abstraction adds names but not meaning |
| DTO churn | data is copied across layers without gaining semantics |
| hard-to-name abstraction | responsibility is still mixed |
| long explanation required | the design is not obvious enough |
| tests need half the system | boundaries are too entangled |

## Structural Review Checklist

- Can I point to one owner for each important rule
- Does each public interface hide more than it exposes
- Is there a smaller change than this redesign
- Are the new packages organized by capability, not by ritual layering
- Would the same feature now require fewer edits than before
- Can the main behavior be tested without booting unrelated infrastructure

## Comment Standard

Design comments should explain one of these:
- why this boundary exists
- what invariant is protected here
- what tradeoff was chosen
- what the caller must not assume

Do not spend comments repeating code that is already obvious.

## Output Shape for Reviews

When reporting a structural review, use this order:

1. Finding
2. Why it matters
3. Smallest boundary change
4. Validation plan
